# Security policy

## Цель

Зафиксировать единый порог блокировки merge для SAST/SCA-проверок в CI всех Go-сервисов Project Next:

- `users`
- `telegram-api`
- `scheduler`
- `answers`
- `analytics`
- `web-admin`
- `recommender`

## Блокирующий уровень

Merge блокируется, если CI находит findings уровня **CRITICAL** или **HIGH**.

Для Semgrep это соответствует флагу:

```bash
semgrep --severity ERROR --error
```

В Semgrep severity `ERROR` используется как высокий порог для CI и сопоставляется с blocking-уровнем **HIGH/CRITICAL**. Findings `WARNING`/`INFO` остаются в SARIF-отчёте, но не блокируют merge.

## SAST policy

Каждый сервис запускает Semgrep в GitHub Actions с правилами:

- `p/golang`
- `p/security-audit`

Отчёт сохраняется в SARIF и выгружается артефактом CI:

```text
semgrep.sarif
```

Правила подавления:

- использовать `.semgrepignore` только для путей, где сканирование не имеет смысла;
- использовать `// nosemgrep: <rule-id>` только точечно, рядом с конкретной строкой;
- рядом с `nosemgrep` обязательно указывать комментарий, почему finding является false positive или осознанным исключением.

Массовое подавление правил не допускается, потому что оно снижает ценность SAST.

## SCA policy

Каждый сервис запускает:

- `govulncheck` по Go-модулю через `go.mod`;
- `trivy fs --scanners vuln,secret` по репозиторию.

Trivy в CI настроен на блокировку только для **HIGH** и **CRITICAL** уязвимостей/secret-находок:

```bash
trivy fs --scanners vuln,secret --severity HIGH,CRITICAL --exit-code 1
```

`govulncheck` не использует CVSS severity, поэтому его findings трактуются как actionable Go-уязвимости и блокируют merge, если анализ показывает уязвимый вызов в reachable-коде.

## Обоснование

- **CRITICAL/HIGH** обычно означают реалистично эксплуатируемую уязвимость или утечку секрета; такие findings должны блокировать merge до релиза.
- **MEDIUM** важно видеть и разбирать планово, но оно не должно останавливать весь pipeline из-за шума или долгого remediation.
- **LOW/INFO** не блокируют CI, но остаются в SARIF/SCA-артефактах для аудита и последующего анализа.
