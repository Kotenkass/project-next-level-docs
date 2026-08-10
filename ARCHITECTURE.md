# Architecture

This document describes the target architecture for the Next Level system. It covers the existing services, the new services, edge routing, buses, external integrations, and storage responsibilities.

## Services

| Service | Purpose | Main storage / bus | Scaling approach |
|---|---|---|---|
| telegram-api | Existing Telegram adapter. Accepts user input from Telegram and forwards answer requests to `answers` via `POST /answers`. | HTTP to `answers`; Redis pub/sub for send/subscribe messaging where used. | Stateless API instances behind the Telegram callback/load-balanced endpoint; scale horizontally by replica count. |
| scheduler | Existing scheduler. Publishes `weekly_reco` events for periodic recommendations. | Kafka/Redis bus depending on implementation; publishes `weekly_reco`. | Stateless job instances with a single active leader per schedule, or a distributed lock if multiple replicas are used. |
| users | Existing user service. Stores and serves user/account data and integrates with `web-admin`. | PostgreSQL. | Stateless API instances; PostgreSQL is scaled separately using read replicas if needed. |
| web-admin | Administrative web interface. Serves browser clients through Ingress + WAF and integrates with `users`, `answers`, PostgreSQL, and analytics. | PostgreSQL for transactional/admin data; HTTP to `users`, `answers`, and analytics. | Stateless frontend/backend instances behind Ingress + WAF; scale horizontally by replica count. |
| answers | New answer-processing service. Receives requests from `telegram-api`, produces `answers.received`, and uses Redis for cache/pub-sub/message buffering. | Kafka topic `answers.received`; Redis; PostgreSQL where transactional answer state is needed. | Stateless API/event producer instances; Kafka partitions and Redis replicas/shards scale throughput. |
| recommender | New recommendation service. Consumes answer-related data/events, calls the LLM API, and produces recommendations/results for downstream processing. | Kafka for answer events/data; LLM API; optionally Redis/PostgreSQL for recommendation state. | Stateless workers; scale by Kafka consumer group partitions and LLM API rate limits. |
| analytics | New analytics service. Consumes `answers.received`, exposes `GET /aggregates`, and writes/read aggregates. | Kafka topic `answers.received`; PostgreSQL; ClickHouse. | Stateless consumers/API instances; scale Kafka consumer groups and ClickHouse cluster capacity. |

## Buses

| Channel / topic | Producer | Consumer(s) | Transport | Why this transport |
|---|---|---|---|---|
| `answers.received` | `answers` | `analytics`; potentially `recommender` or other future consumers | Kafka | Durable event stream with replay, partitioning, and decoupling of answer production from analytics/recommendation processing. |
| `weekly_reco` | `scheduler` | `recommender` or recommendation pipeline | Kafka | Durable scheduled event stream with replay and retry support for recommendation processing. |
| `publish_send_message` | `telegram-api` or notification layer | Telegram sender workers / Redis subscribers | Redis | Low-latency pub/sub for immediate send commands and transient messaging. |
| `subscribe` | Telegram/message receiver components | Redis subscribers / message handlers | Redis | Fast in-memory messaging for subscription-style updates; Redis already acts as cache and message broker in the architecture. |
| Redis cache keys | Writers such as `answers`, `web-admin`, or API consumers | Readers of cached data | Redis | Low-latency cache for hot data, temporary state, and rate-limit/session-like data. |

## External integrations

| External system | Secret(s) used | Services that call it | Notes |
|---|---|---|---|
| Telegram API | Telegram bot token / webhook secret; optional Telegram API access token if used by infrastructure | `telegram-api`; existing Telegram-facing infrastructure | Receives Telegram user input and sends replies/notifications. Secrets must be stored outside the repository, for example in Kubernetes Secrets, Vault, or CI/CD secret variables. |
| LLM API | LLM provider API key; optional organization/project key and model configuration | `recommender` | Used for recommendation generation or answer enrichment. Rate limits, quotas, and model selection should be configured per environment. |

## Existing

The upper part of the architecture contains services that already existed in the old architecture:

- `telegram-api`
  - accepts `POST /answers` requests;
  - interacts with the new `answers` service.
- `scheduler`
  - publishes the `weekly_reco` message/event;
  - is connected with `telegram-api` and the new system.
- `users`
  - existing user service;
  - interacts with `web-admin`.

These services are inherited from the previous 12-factor-app architecture and are kept as separate responsibilities during the migration.

## Edge

The edge layer sits before the new services:

```text
User -> Browser -> Ingress + WAF
```

`Ingress + WAF` is the single entry point for browser HTTP requests and routes them to `web-admin`. It also terminates or protects public HTTP access before traffic reaches application services.

Telegram traffic continues through the existing Telegram-facing infrastructure and reaches the system through `telegram-api`.

## New services — Next Level

The new part of the system consists of four services:

### web-admin

Administrative web interface.

Responsibilities:

- receives requests through `Ingress + WAF`;
- interacts with `users`;
- interacts with `answers`;
- works with PostgreSQL;
- connects to analytics, including `GET /aggregates`.

### answers

Answer-processing service.

Responsibilities:

- receives requests from `telegram-api`;
- interacts with `web-admin`;
- produces `answers.received` events;
- publishes those events to Kafka;
- interacts with Redis for cache, pub/sub, and intermediate message storage.

### recommender

Recommendation service.

Responsibilities:

- receives answer-related events/data from `answers`;
- uses Kafka as the source of answer events;
- interacts with the LLM API;
- forms recommendations/results for further processing.

### analytics

Analytics service.

Responsibilities:

- receives data through `GET /aggregates`;
- consumes `answers.received` events;
- works with PostgreSQL;
- works with ClickHouse.

## Kafka and Redis

### Redis

Redis is used as:

- cache;
- publish/subscribe mechanism;
- intermediate message storage.

In the diagram, `answers` is connected to Redis, and Redis is connected to `publish_send_message` / `subscribe` mechanisms.

### Kafka

Kafka contains the topic:

```text
answers.received
```

`answers` produces events to this topic:

```text
answers
   |
   | produce
   v
Kafka topic: answers.received
   |
   | consume
   v
analytics
```

This allows analytics to process answers asynchronously without forcing `answers` to wait for analytical processing to finish.

## Stores

The storage block contains:

- PostgreSQL
- ClickHouse

### PostgreSQL

PostgreSQL is responsible for transactional data, including:

- user/system data;
- data required by `web-admin`;
- transactional state for services that need durable operational records.

### ClickHouse

ClickHouse is responsible for analytical data, including:

- aggregates;
- large event/metric volumes;
- analytics queries over answer and recommendation events.

`analytics` interacts with both stores.

## External systems

The external block contains:

- Telegram API
- LLM API

### Telegram API

The system interacts with Telegram through the external Telegram API.

Typical user path:

```text
User
 |
 v
Telegram
 |
 v
Telegram API
 |
 v
telegram-api
 |
 v
answers
```

`telegram-api` also sends `POST /answers` requests to `answers`.

### LLM API

`recommender` calls the external LLM API to generate or process recommendations.

## Main data flow

Simplified main flow:

```text
                    ┌──────────────┐
                    │    User      │
                    └──────┬───────┘
                           │
                     Browser / Telegram
                           │
                           ▼
                    ┌──────────────┐
                    │ Ingress + WAF│
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  web-admin   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           users        answers      PostgreSQL
                           │
                           ▼
                    ┌──────────────┐
                    │    Kafka     │
                    │answers.received
                    └──────┬───────┘
                           │
                           ▼
                      analytics
                       │     │
                       ▼     ▼
                  PostgreSQL ClickHouse
```

Parallel recommendation flow:

```text
answers
   |
   v
recommender
   |
   v
LLM API
```

Redis is used as a separate infrastructure layer for caching and pub/sub.

## Summary

The architecture moves from a relatively simple set of existing services to a service-oriented architecture with separated responsibilities:

- `telegram-api` — Telegram integration;
- `answers` — answer processing;
- `recommender` — recommendation generation;
- `analytics` — analytics and aggregates;
- `web-admin` — administrative interface;
- Kafka — asynchronous event delivery;
- Redis — cache and pub/sub;
- PostgreSQL — main transactional store;
- ClickHouse — analytical store;
- Ingress + WAF — external HTTP entry;
- LLM API — external AI dependency.

The central event of the new architecture is `answers.received`: `answers` publishes it to Kafka and `analytics` consumes it asynchronously. This separates the user/transactional path from analytical processing.
