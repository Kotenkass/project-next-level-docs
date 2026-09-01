# Краткий отчет Threat-модель Project Next

## Топ-риски

1. Kafka listener `9092` использует `tls: false`.
2. Redis в `dev` не имеет подтвержденной ACL/password-аутентификации.
3. PostgreSQL в `dev` использует `emptyDir`.
4. ClickHouse настроен с `accessManagement: false` и `tlsCerts.create: false`.
5. Нет подтвержденных `NetworkPolicy` между сервисами в `dev`.
6. recommender отправляет пользовательские данные во внешний OpenRouter.
7. Internal APIs `users`, `analytics`, `web-admin`, `telegram-api` требуют проверки caller identity.
8. `/metrics` endpoints скрейпятся VictoriaMetrics без подтвержденного ограничения доступа.

## Рекомендации

- Включить TLS listener 9093 для Kafka и запретить plaintext listener.
- Включить Kafka SASL/ACL.
- Включить Redis auth/ACL и ограничить доступ NetworkPolicy.
- Использовать PVC для PostgreSQL и настроить backup/restore.
- Включить ClickHouse TLS, access management, отдельные users/roles.
- Добавить default-deny NetworkPolicy в `dev`.
- Анонимизировать `chat_id`/`user_id` перед отправкой в OpenRouter.
- Удалить лишнюю PII из prompt.
- Добавить internal service auth/mTLS.
- Ограничить доступ к `/metrics` и query API VictoriaMetrics.
