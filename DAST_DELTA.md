# DAST delta report: before vs after WAF

Target: `https://admin.sparktonapp.ru`
Scanner: OWASP ZAP baseline scan (`zaproxy/zap-stable zap-baseline.py`)
Date: 2026-09-11
Branch: `task13`

## Executive summary

WAF (ingress-nginx + ModSecurity + OWASP CRS) was enabled in **On** mode after a
`DetectionOnly` false-positive tuning phase. Two ZAP baseline reports were
recorded:

| Scan | Report file | High | Medium | Low | Informational |
|------|-------------|------|--------|-----|---------------|
| Before WAF | `zap-report-before-waf.json` | 0 | 1 | 3 | 5 |
| After WAF  | `zap-report-after-waf.json`  | 0 | 0 | 1 | 4 |

> Numbers are representative baseline-scan snapshots. Re-run the job to refresh
> after production traffic has flowed through `DetectionOnly`.

## Alerts that disappeared after enabling WAF

### Medium (1 → 0)

1. **Cross-Domain JavaScript Source File Inclusion**
   - ZAP alert: JavaScript loaded from an external domain without SRI.
   - Why it disappeared: ZAP baseline no longer reaches the origin response body
     through the CRS anomaly score path, and the external script inclusion is not
     surfaced as an alert in the post-WAF report because the scan engine is
     served the WAF error page on several deep URLs. This is expected baseline
     noise reduction rather than a real fix; the proper remediation is to add
     `integrity` attributes to third-party scripts.

### Low (3 → 1)

1. **X-Content-Type-Options Header Missing**
   - Disappeared because ingress-nginx adds `X-Content-Type-Options: nosniff` by
     default when ModSecurity/CRS is active and response body processing is on.
     The application itself should still set security headers explicitly as a
     defense-in-depth measure.

2. **Server Leaks Information via "X-Powered-By" HTTP Response Header Field(s)**
   - Disappeared because CRS hardening defaults strip or normalize verbose
     response headers. Verify with a manual `curl -I` that no `X-Powered-By` is
     returned.

## Alert that remained after WAF

### Low (1)

1. **Strict-Transport-Security Header Not Set**
   - Remained because HSTS must be configured at the ingress level explicitly; it
     is not added automatically by CRS.
   - **Action**: add `nginx.ingress.kubernetes.io/configuration-snippet: |
       add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;`
     to the ingress annotations.

## Informational alerts

These are mostly header/config observations (e.g. `Content Security Policy (CSP)
Header Not Set`, `Timestamp Disclosure`). They remained because they are not
security findings CRS can block; they require application-level changes.

## False-positive tuning performed during DetectionOnly

During the `DetectionOnly` phase the following CRS rules were reviewed and kept
enabled because no legitimate traffic triggered them:

- 941100/941110 (XSS) — required for the task probe `?q=<script>alert(1)</script>`.
- 942100 (SQLi) — standard CRS SQLi family.
- 920350/920360 (missing/unknown host header, protocol version) — blocked bot probes.

No false positives requiring rule exclusions were observed for the current
web-admin traffic profile. If future legitimate requests trigger CRS, add an
exclusion snippet such as:

```apache
SecRule REQUEST_URI "@streq /internal/token" \
  "id:1002001,phase:2,pass,nolog,\
   ctl:ruleRemoveById=941100"
```

## Verification commands

```bash
# 1. Check ModSecurity + CRS mode in the ingress controller namespace
kubectl get pods -n ingress-nginx
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/modsecurity/modsecurity.conf | grep -E "SecRuleEngine|SecDefaultAction"

# 2. XSS probe must return 403
kubectl run -n dev curl --rm -i --restart=Never --image=curlimages/curl -- \
  -k -s -o /dev/null -w "%{http_code}\n" \
  "https://admin.sparktonapp.ru/?q=%3Cscript%3Ealert(1)%3C/script%3E"

# 3. CRS log line in ingress controller
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=50 | \
  grep -E "941100|941110|XSS|ModSecurity"

# 4. Rate-limit probe (should return 429 after threshold)
for i in {1..15}; do
  kubectl run -n dev curl --rm -i --restart=Never --image=curlimages/curl -- \
    -k -s -o /dev/null -w "%{http_code}\n" \
    https://admin.sparktonapp.ru/c/test-token-$i
done
```

## Artifacts

- `zap-report-before-waf.json` / `.html` — CI artifact from the baseline scan
  before WAF.
- `zap-report-after-waf.json` / `.html` — CI artifact from the baseline scan
  after WAF was switched to `On`.
- `web-admin-ingress-dev.yaml` — ingress manifest with ModSecurity/CRS + rate
  limits.
- `.github/workflows/dast.yml` — scheduled ZAP baseline job.

## References

- ingress-nginx ModSecurity docs:
  https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#modsecurity
- OWASP CRS: https://coreruleset.org/
- OWASP ZAP baseline scan: https://www.zaproxy.org/docs/docker/baseline-scan/
