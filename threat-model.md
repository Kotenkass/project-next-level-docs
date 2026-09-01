# Threat-модель Project Next

## 1. Метаданные

- **Название модели:** Project Next — Threat Model
- **Проект:** Project Next
- **Архитектура:** Telegram Bot + LLM + Analytics + Web Admin
- **Развертывание:** Kubernetes namespaces `dev`, `kafka`, `metrics`
- **Методология:** STRIDE
- **Инструмент:** OWASP Threat Dragon
- **Источник требований:** `~/todo_threatdragon.txt`, репозиторий `~/Project next`, примеры OWASP Threat Model Cookbook

## 2. Описание системы

Project Next — Telegram-бот и веб-админка, развернутые в Kubernetes namespace `dev`. Пользователи взаимодействуют с ботом через Telegram Bot API. `telegram-api` обрабатывает входящие сообщения, публикует события в Kafka topic `answers.received`, взаимодействует с Redis и `users`, а также передает данные в `web-admin`.

`analytics` читает события из Kafka, пишет аналитику в ClickHouse и отдает метрики/агрегаты через HTTP. `recommender` работает с Redis pub/sub, `users`, `analytics` и внешним LLM-провайдером OpenRouter для генерации рекомендаций. `scheduler` работает с Redis и `users`.

Веб-админка доступна через Ingress `admin.sparktonapp.ru` и общается с `users` и `analytics`. В кластере также находятся PostgreSQL, Redis, Kafka в namespace `kafka`, ClickHouse и VictoriaMetrics/vmagent для сбора метрик.

## 3. Trust Boundaries

### 3.1 Internet

Внутри:

- Telegram User
- User Browser / Admin
- Telegram Bot API
- OpenRouter API

### 3.2 K8s namespace dev

Внутри:

- `telegram-api`
- `scheduler`
- `recommender`
- `web-admin`
- `users`
- `analytics`
- PostgreSQL
- Redis
- ClickHouse

### 3.3 K8s namespace kafka

Внутри:

- Kafka Cluster
- Kafka Topic `answers.received`

Факт по манифестам:

- `project-next-charts/manifests/kafka/kafka-dev.yaml`
- namespace: `kafka`

Важно:

- Kafka имеет listener:
  - `plain:9092 tls: false`
  - `tls:9093 tls: true`
- plaintext listener включен.

### 3.4 K8s namespace metrics

Внутри:

- VictoriaMetrics / vmagent

Факт по манифестам:

- `project-next-charts/manifests/victoriametrics/`
- namespace: `metrics`

## 4. Элементы модели

### Internet

- Actor: Telegram User
- Actor: User Browser / Admin
- External: Telegram Bot API
- External: OpenRouter API

### K8s namespace dev

- Process: `telegram-api :8080`
- Process: `scheduler`
- Process: `recommender :8080`
- Process: `web-admin :8080`
- Process: `users :8080`
- Process: `analytics :8080`
- Store: PostgreSQL `:5432`
- Store: Redis `:6379`
- Store: ClickHouse `:9000`

### K8s namespace kafka

- Process/Store: Kafka Cluster
- Store: Kafka Topic `answers.received`

### K8s namespace metrics

- Process: VictoriaMetrics / vmagent

## 5. Data Flows

1. Telegram User → Telegram Bot API
2. Telegram Bot API ↔ `telegram-api`
   - polling loop: `telegram-api` опрашивает Telegram Bot API
   - пересекает Internet → K8s `dev`
3. `telegram-api` → Redis
4. `telegram-api` → `users`
5. `telegram-api` → `web-admin`
6. `telegram-api` → Kafka topic `answers.received`
   - пересекает K8s `dev` → K8s `kafka`
7. Kafka topic `answers.received` → `analytics`
   - пересекает K8s `kafka` → K8s `dev`
8. `analytics` → ClickHouse
9. `scheduler` → Redis
10. `scheduler` → `users`
11. `recommender` → Redis
12. `recommender` → `users`
13. `recommender` → `analytics`
14. `recommender` → OpenRouter API
    - пересекает K8s `dev` → Internet/External
15. User Browser → `web-admin`
    - пересекает Internet → K8s `dev`
16. `web-admin` → `users`
17. `web-admin` → `analytics`
18. VictoriaMetrics/vmagent → metrics endpoints сервисов
    - пересекает K8s `metrics` → K8s `dev`/`kafka`

## 6. STRIDE-анализ элементов

### 6.1 telegram-api

**Spoofing**

- Threat: Компрометация `TELEGRAM_TOKEN` позволяет взаимодействовать с Telegram Bot API от имени бота.
- Mitigation: Хранить `TELEGRAM_TOKEN` в Kubernetes Secret; ограничить доступ к Secret; не логировать env; включить ротацию токена.
- Risk: High

**Information Disclosure**

- Threat: Внутренние HTTP-вызовы `telegram-api` → `users`/`web-admin`/Redis/Kafka могут идти без mTLS и без аутентификации между сервисами.
- Mitigation: Добавить mTLS/service mesh или internal service auth; ограничить NetworkPolicy.
- Risk: Medium/High

**Denial of Service**

- Threat: Отсутствие rate limiting на входящие Telegram-сообщения может привести к росту нагрузки на `telegram-api`, Redis, Kafka и `users`.
- Mitigation: Rate limiting по `chat_id`/`user_id`; backpressure; resource limits; autoscaling.
- Risk: Medium

### 6.2 scheduler

**Tampering**

- Threat: `scheduler` публикует/читает Redis pub/sub без подтвержденной аутентификации.
- Mitigation: Redis ACL/password; ограничить доступ NetworkPolicy; подписывать критичные сообщения.
- Risk: High

**Information Disclosure**

- Threat: `scheduler` имеет `TELEGRAM_TOKEN` в Secret, но любой pod с доступом к Secret/env может его получить.
- Mitigation: Ограничить доступ к Secret; не логировать env; rotation.
- Risk: Medium

### 6.3 recommender

**Spoofing**

- Threat: Компрометация `OPENROUTER_API_KEY` позволяет использовать лимит или средства проекта у LLM-провайдера.
- Mitigation: Хранить ключ в Secret; ограничить доступ; включить ротацию; настроить бюджет/quotas у провайдера.
- Risk: High

**Information Disclosure**

- Threat: `recommender` отправляет данные пользователя во внешний OpenRouter API.
- Mitigation: Анонимизировать `chat_id`/`user_id`; удалять лишнюю PII из prompt; настроить data retention с LLM-провайдером.
- Risk: Medium/High

**Denial of Service**

- Threat: OpenRouter или LLM latency может блокировать обработку рекомендаций.
- Mitigation: Rate limiter, max concurrency, circuit breaker, timeout, queue/backpressure.
- Risk: Medium

### 6.4 web-admin

**Spoofing**

- Threat: Админ-доступ зависит от одноразовых токенов/session cookie. При компрометации `SESSION_KEY` возможны поддельные сессии.
- Mitigation: `SESSION_KEY` в Secret; rotation; HTTPOnly, Secure, SameSite cookie; TTL сессий.
- Risk: Medium

**Information Disclosure**

- Threat: Ingress `admin.sparktonapp.ru` открывает веб-админку в Internet.
- Mitigation: TLS уже настроен; добавить HSTS; rate limiting; auth перед доступом; audit log.
- Risk: Medium

**Elevation of Privilege**

- Threat: Если `web-admin` может читать `analytics`/`users` без проверки прав, пользователь может получить доступ к чужим данным.
- Mitigation: Проверять авторизацию по `chat_id`/`user_id`; не доверять только client-side.
- Risk: High

### 6.5 users

**Spoofing**

- Threat: `users` API доступен внутри кластера без подтвержденной аутентификации вызывающего сервиса.
- Mitigation: Добавить internal service authentication или mTLS; ограничить callers через NetworkPolicy.
- Risk: High

**Tampering**

- Threat: Любой сервис в `dev` namespace потенциально может вызвать CRUD-эндпоинты `users`.
- Mitigation: Авторизация по caller identity; RBAC/NetworkPolicy; запретить прямой доступ из лишних сервисов.
- Risk: High

**Denial of Service**

- Threat: `users` зависит от PostgreSQL на emptyDir и может потерять данные при пересоздании pod.
- Mitigation: Использовать PersistentVolumeClaim; backup/restore; readiness/liveness probes.
- Risk: Medium/High

### 6.6 analytics

**Spoofing**

- Threat: Kafka consumer может читать/обрабатывать неподтвержденные события из `answers.received`.
- Mitigation: Использовать Kafka TLS listener 9093, ACLs/SASL, проверить producer identity.
- Risk: High

**Information Disclosure**

- Threat: `/metrics` доступен без аутентификации и может раскрывать внутреннюю информацию.
- Mitigation: Ограничить доступ к `/metrics` через NetworkPolicy; добавить auth для метрик; не выводить PII.
- Risk: Low/Medium

**Elevation of Privilege**

- Threat: `analytics` может читать или агрегировать данные пользователей, к которым не должен иметь доступ.
- Mitigation: Проверять авторизацию на уровне API; минимизировать объем PII в Kafka/ClickHouse.
- Risk: Medium

### 6.7 PostgreSQL

**Information Disclosure**

- Threat: `POSTGRES_PASSWORD` хранится в Kubernetes Secret, но Secret не является полноценным шифрованием и может быть прочитан при доступе к namespace/RBAC.
- Mitigation: Ограничить RBAC на secrets; external secret manager; rotation.
- Risk: Medium

**Denial of Service**

- Threat: PostgreSQL использует emptyDir, данные теряются при пересоздании pod.
- Mitigation: PersistentVolumeClaim; backup/restore; мониторинг disk.
- Risk: High

### 6.8 Redis

**Tampering**

- Threat: Redis pub/sub может быть доступен любому pod внутри namespace без ACL/password-аутентификации.
- Mitigation: Redis ACL/password; отдельные users/channels; NetworkPolicy.
- Risk: High

**Information Disclosure**

- Threat: Redis может хранить временные пользовательские данные, сессии, последние рекомендации.
- Mitigation: Не хранить PII дольше необходимого; включить auth; ограничить доступ.
- Risk: Medium

### 6.9 Kafka

**Information Disclosure**

- Threat: Kafka имеет plaintext listener `9092 tls=false`.
- Mitigation: Использовать TLS listener 9093; запретить plaintext listener; включить SASL/ACL.
- Risk: High

**Tampering**

- Threat: Без ACL любой authorized consumer/producer может писать/читать topic `answers.received`.
- Mitigation: Kafka ACLs; отдельные service accounts; ограничить dev access.
- Risk: High

**Denial of Service**

- Threat: Kafka replication factor = 1, min.insync.replicas = 1.
- Mitigation: Для prod увеличить replication factor и min ISR; monitoring lag.
- Risk: Medium/High

### 6.10 ClickHouse

**Information Disclosure**

- Threat: ClickHouse TLS certs не создаются в манифесте, `accessManagement=false`.
- Mitigation: Включить TLS; включить access management; отдельные users/roles; ограничить NetworkPolicy.
- Risk: High

**Tampering**

- Threat: Аналитические данные могут быть изменены пользователем с доступом к ClickHouse.
- Mitigation: RBAC/roles; audit; backup; readonly roles для аналитики.
- Risk: Medium

### 6.11 VictoriaMetrics / vmagent

**Information Disclosure**

- Threat: vmagent скрейпит `/metrics` сервисов в `dev`/`kafka` и может собирать внутреннюю информацию.
- Mitigation: Ограничить NetworkPolicy; не выводить секреты/PII в метрики; ограничить доступ к query API.
- Risk: Medium

**Denial of Service**

- Threat: Частый scrape interval 10s может добавить нагрузку на сервисы.
- Mitigation: Настроить разумный scrape interval; resource limits; мониторинг scrape errors.
- Risk: Low/Medium

## 7. STRIDE-анализ потоков через границы доверия

### F2: Telegram Bot API ↔ telegram-api

- Threat: Компрометация `TELEGRAM_TOKEN` позволяет читать/отправлять сообщения от имени бота.
- Mitigation: Secret, RBAC на Secret, rotation, минимизация логов.
- Risk: High

- Threat: В polling-запросах используется `TELEGRAM_TOKEN`; при компрометации канала/лога токен может быть раскрыт.
- Mitigation: Использовать только HTTPS; не логировать токены; ротация при подозрении.
- Risk: Low/Medium

### F6/F7: telegram-api/analytics ↔ Kafka namespace kafka

- Threat: Данные событий пользователей передаются между namespace `dev` и `kafka`; plaintext listener `9092 tls=false` допускает перехват внутри кластера.
- Mitigation: Использовать TLS listener 9093; запретить plaintext listener; Kafka ACLs/SASL.
- Risk: High

- Threat: Без ACL producer/consumer может подменить или прочитать события в `answers.received`.
- Mitigation: Kafka ACLs; отдельные service accounts; producer identity.
- Risk: High

### F14: recommender → OpenRouter API

- Threat: OpenRouter получает chatID и аналитику пользователя.
- Mitigation: Анонимизировать данные перед отправкой; удалить PII из prompt; настроить retention у провайдера.
- Risk: Medium/High

- Threat: `OPENROUTER_API_KEY` в Bearer header — при компрометации pod можно использовать LLM за счет проекта.
- Mitigation: HTTPS; Secret; rotation; budget/quotas у провайдера.
- Risk: High

### F15: User Browser → web-admin

- Threat: Одноразовый токен + HMAC-session защищают админ-доступ, но при компрометации `SESSION_KEY` возможны поддельные сессии.
- Mitigation: token TTL=15min, session TTL=1h, HTTPOnly, Secure, SameSite, rotation `SESSION_KEY`.
- Risk: Medium

- Threat: Ingress `admin.sparktonapp.ru` открывает веб-админку в Internet.
- Mitigation: TLS включен через cert-manager; добавить HSTS; rate limiting; audit log.
- Risk: Medium

### F18: VictoriaMetrics/vmagent → metrics endpoints

- Threat: VictoriaMetrics скрейпит `/metrics` сервисов в `dev`/`kafka`; метрики могут раскрывать внутреннюю информацию.
- Mitigation: Не выводить PII/секреты в метрики; ограничить доступ к query API; NetworkPolicy.
- Risk: Medium

- Threat: Частый scrape interval 10s может добавить нагрузку на сервисы.
- Mitigation: Настроить разумный interval; resource limits; мониторинг ошибок scrape.
- Risk: Low/Medium

## 8. Kubernetes-level угрозы

**Elevation of Privilege**

- Threat: GitHub runner manifests монтируют service account token и имеют RBAC-права на namespace.
- Mitigation: Проверить минимальность Role/RoleBinding; `automountServiceAccountToken` только там, где нужно.
- Risk: Medium/High

**Elevation of Privilege**

- Threat: В namespace `dev` отсутствует подтвержденный NetworkPolicy, значит pod-to-pod трафик может быть не ограничен.
- Mitigation: Добавить default-deny NetworkPolicy и разрешить только нужные сервисы.
- Risk: High

**Information Disclosure**

- Threat: Kubernetes Secrets используются для токенов/паролей, но нет подтверждения external secret manager/encryption at rest.
- Mitigation: Проверить encryption at rest для etcd; ограничить RBAC; использовать external-secrets/Sealed Secrets/SOPS.
- Risk: Medium

**Tampering**

- Threat: Docker images используются с тегом `dev`/`latest` без явной защиты от подмены.
- Mitigation: Pin images by digest; private registry; image pull policy; image scanning.
- Risk: Medium

## 9. Top risks и рекомендации

1. **Kafka namespace `kafka` использует plaintext listener `9092 tls=false`.**
   - Action: Использовать TLS listener 9093, запретить plaintext listener, включить SASL/ACL.

2. **Redis в `dev` не имеет подтвержденной ACL/password-аутентификации.**
   - Action: Включить Redis auth/ACL; ограничить доступ NetworkPolicy.

3. **PostgreSQL в `dev` использует emptyDir.**
   - Action: Использовать PersistentVolumeClaim; настроить backup/restore.

4. **ClickHouse configured with `accessManagement=false` and `tlsCerts.create=false`.**
   - Action: Включить TLS, access management, отдельные users/roles.

5. **Нет подтвержденных NetworkPolicy между сервисами в namespace `dev`.**
   - Action: Добавить default-deny и разрешить только нужные сервисы.

6. **OpenRouter получает пользовательские данные.**
   - Action: Анонимизировать `chat_id`/`user_id` и удалять лишнюю PII из prompt.

7. **Internal APIs `users`/`analytics`/`web-admin`/`telegram-api` требуют проверки caller identity.**
   - Action: Добавить internal service auth/mTLS и авторизацию по caller.

8. **Metrics endpoints доступны внутри кластера и скрейпятся VictoriaMetrics.**
   - Action: Ограничить доступ, не выводить PII/секреты в метрики, ограничить доступ к query API.

