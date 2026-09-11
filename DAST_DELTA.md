# DAST delta report: before vs after WAF

Target: `https://admin.sparktonapp.ru`
Scanner: OWASP ZAP baseline scan (`zaproxy/zap-stable zap-baseline.py`)
Date: 2026-09-11
Branch: `task13`

## Executive summary

WAF (ingress-nginx + ModSecurity + OWASP CRS) was enabled in **On** mode.
Two ZAP baseline reports were recorded:

| Scan | Run ID | High | Medium | Low | Informational |
|------|--------|------|--------|-----|---------------|
| Before WAF (DetectionOnly) | 34654070961 | 0 | 0 | 0 | 2 |
| After WAF (On) | 34654230118 | 0 | 0 | 0 | 2 |

> **Note:** The application requires authentication (returns 401 without valid session),
> so ZAP baseline scan can only access public endpoints (/, /robots.txt, /sitemap.xml).
> No application-level vulnerabilities were found in either scan.

## What changed

### Manual XSS probe test

| Test | Command | Result |
|------|---------|--------|
| Without WAF | `curl https://admin.sparktonapp.ru/?q=%3Cscript%3Ealert(1)%3C/script%3E` | 401 (passed through) |
| With WAF | `curl https://admin.sparktonapp.ru/?q=%3Cscript%3Ealert(1)%3C/script%3E` | **403 blocked** |

### Ingress-nginx logs with WAF enabled

```
2026/09/11 22:11:40 [error] 942#942: *663 [client 31.76.38.78] ModSecurity: Access denied with code 403 (phase 2). detected XSS using libinjection. [id "1099"] [msg "XSS blocked"] [tag "task13"] ...
```

## Alerts found (informational)

Both scans found only cache-related informational alerts:

1. **Non-Storable Content** - Response has `WWW-Authenticate` header requiring auth
2. **Storable and Cacheable Content** - /robots.txt and /sitemap.xml are cacheable

These are expected for a protected endpoint and not security issues.

## WAF configuration

- **Repository:** `project-next-charts`
- **File:** `manifests/web-admin/web-admin-ingress-dev.yaml`
- **Config:**
  - `SecRuleEngine On`
  - OWASP CRS enabled via `nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"`
  - Virtual patch for XSS: `SecRule ARGS:q "@detectXSS" "id:1099,phase:2,deny,status:403"`

## Verification

```bash
# Test XSS probe - should return 403 with WAF On
curl -s -o /dev/null -w "%{http_code}" "https://admin.sparktonapp.ru/?q=%3Cscript%3Ealert(1)%3C/script%3E"

# Check WAF logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep -E "ModSecurity|1099|XSS"
```

## Artifacts

- DAST scan (DetectionOnly): https://github.com/Kotenkass/web-admin/actions/runs/34654070961
- DAST scan (WAF On): https://github.com/Kotenkass/web-admin/actions/runs/34654230118
- Ingress manifest: `project-next-charts/manifests/web-admin/web-admin-ingress-dev.yaml`
- DAST workflow: `web-admin/.github/workflows/dast.yml`

## Conclusion

WAF is working correctly:
- XSS probes are blocked with HTTP 403
- ModSecurity logs show the block with rule ID 1099
- ZAP baseline scan cannot find vulnerabilities because the app is protected by auth
- No false positives were observed during DetectionOnly phase
