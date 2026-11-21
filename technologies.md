# 🧰 Technologies — LE Alert Correlation Engine

Every technology used, why, and where.

---

## 🎨 Frontend

| Tech | Version | Purpose |
|---|---|---|
| **React** | 18.3 | UI library |
| **TypeScript** | 5.9 | Type-safe JS |
| **Vite** | 4.5 | Build tool + dev server |
| **TailwindCSS** | 4.1 | Utility-first CSS |
| **React Router** | 6.30 | Client-side routing |
| **TanStack Query** | 4.42 | Server state / caching |
| **React Hook Form** | 7.71 | Form state |
| **Zod** | 3.25 | Schema validation |
| **Axios** | 1.13 | HTTP client |
| **@stomp/stompjs** | 7.2 | STOMP over WebSocket |
| **socket.io-client** | 4.8 | (legacy fallback — not used) |
| **Chart.js / react-chartjs-2** | 4.5 / 5.3 | Priority distribution chart |
| **Recharts** | 2.15 | Additional charts |
| **Leaflet / react-leaflet** | 1.9 / 4.2 | Map rendering |
| **@heroicons/react** | 2.2 | Icons |
| **clsx** | 2.1 | Conditional classnames |
| **crypto-js** | 4.2 | Client-side PII masking |
| **date-fns** | 2.30 | Date formatting |
| **@supabase/supabase-js** | 2.45 | Auth + Storage |
| **react-markdown** | 9.x | **NEW** — Renders LLM markdown answers |
| **remark-gfm** | 4.x | **NEW** — Tables/lists in markdown |
| **dompurify** | 3.x | **NEW** — Sanitizes rendered markdown |
| **Vitest** | 0.34 | Test runner |
| **@testing-library/react** | 14.3 | Component tests |
| **jsdom** | 22 | DOM environment |
| **ESLint + Prettier** | 8 / 3 | Lint + format |

---

## ☕ Backend

### Framework runtimes
| Tech | Version | Where | Purpose |
|---|---|---|---|
| **Spring Boot** | 3.2.5 | alert-service, api-gateway, eureka, config-server | Web / reactive / batch |
| **Spring Cloud** | 2023.0.1 | api-gateway, discovery, config | Service mesh primitives |
| **Quarkus** | 2.16.12 | correlation-engine | High-perf streaming |
| **Spring WebFlux** | (BOM) | api-gateway, security-core | Reactive HTTP |
| **Spring Web MVC** | (BOM) | alert-service | Servlet HTTP + STOMP + **SSE streaming** |

### Data & messaging
| Tech | Version | Purpose |
|---|---|---|
| **MongoDB** | 6.0 | Primary document store |
| **Spring Data MongoDB** | (BOM) | Repositories + geospatial |
| **Redis** | 7-alpine | Cache, session, pub/sub, **hot buffer (Streams)** |
| **Spring Data Redis** | (BOM) | Cache manager, Lettuce, pub/sub, **Stream operations** |
| **Apache Kafka** | Confluent 7.4 | Streaming pipeline |
| **Spring Kafka** | (BOM) | Consumer / producer |
| **SmallRye Reactive Messaging** | 2.16.12 | Quarkus Kafka bindings |
| **PostgreSQL** | 15-alpine | Audit + relational |
| **Supabase** | Cloud | Auth, Storage, Postgres, **pgvector** |

### 🔮 AI / RAG Stack (NEW)

| Tech | Version | Where | Purpose |
|---|---|---|---|
| **OpenAI GPT-4o-mini** | API | alert-service LLM package | Chat/reasoning model (primary) |
| **OpenAI `text-embedding-3-small`** | API | alert-service LLM package | Embedding model (1536-dim, primary) |
| **Ollama + Llama-3** | Local | Optional (fallback) | Self-hosted chat model when cost ceiling hit |
| **BGE-small-en** | Local (via ONNX) | Optional (fallback) | Self-hosted embedding model |
| **pgvector** | Postgres extension | Supabase | Vector similarity store for person embeddings |
| **pgvector-spring** | (via Spring Data JDBC / JPA) | alert-service | pgvector repository adapter |
| **Spring AI (optional)** | 1.0.x | alert-service | Abstraction layer for chat + embeddings |
| **Raw HTTP (WebClient)** | (BOM) | alert-service | Direct OpenAI/Ollama calls (primary approach) |
| **SSE (Server-Sent Events)** | (BOM) | alert-service + frontend | Token-by-token streaming |
| **OpenPDF** | 1.3.x | alert-service | PDF generation (backend) |
| **FreeMarker** | 2.3.x | alert-service | HTML → PDF template engine |
| **tiktoken-java** | 0.6.x | alert-service | Token counting for cost estimation |
| **Bucket4j** | 8.x | alert-service | Per-user rate limiting for LLM queries |
| **Caffeine** | 3.x | alert-service | In-memory hot cache for embeddings |

### 🔐 PII Simulation (NEW)

| Tech | Purpose |
|---|---|
| **Custom PII redactor** | Maps real fields → synthetic surrogates before LLM calls |
| **Mode flag** | `simulated` / `redacted` / `raw` — env-controlled, no code change |

### Security
| Tech | Purpose |
|---|---|
| **Spring Security 6** | Servlet + reactive filters |
| **Spring OAuth2 Resource Server** | JWT verification |
| **Nimbus JOSE + JWT** | JWKS decode |
| **Supabase JWT** | Identity provider |
| **HMAC-SHA256** | HS256 fallback |
| **Authority `SCOPE_llm:query`** | **NEW** — Gates Ask COP access |

### Geospatial
| Tech | Purpose |
|---|---|
| **JTS (Java Topology Suite)** | Correlation-engine geometry |
| **MongoDB `2dsphere`** | Geo indexing |
| **GeoJSON** | Wire format |

### Observability
| Tech | Purpose |
|---|---|
| **Micrometer** | Metrics abstraction |
| **Prometheus** | Metrics store + scrape |
| **Grafana** | Dashboards (**+ 6 new LLM panels**) |
| **Spring Actuator** | `/health`, `/metrics` |
| **Custom `/api/llm/health`** | **NEW** — Composite LLM + data health |
| **SLF4J + Logback** | Logging |
| **Quarkus JSON logging** | Structured logs |

### Testing (backend)
| Tech | Purpose |
|---|---|
| **JUnit 5** | Test runner |
| **Mockito** | Mocks |
| **AssertJ** | Fluent assertions |
| **Spring Boot Test** | Test slices |
| **Testcontainers 1.19.7** | Mongo + Redis + **pgvector** in tests |
| **Spring Kafka Test** | Embedded Kafka |
| **WireMock** | **NEW** — Mock OpenAI API responses |

---

## 🏗️ Infrastructure

| Tech | Purpose |
|---|---|
| **Docker** | Containerisation |
| **Docker Compose** | Local orchestration |
| **Eureka** | Service discovery |
| **Spring Cloud Config** | Centralised config |
| **Spring Cloud Gateway** | Edge routing (**+ `/api/llm/**` route**) |
| **Maven** | Build system |
| **GitHub** | VCS |

---

## 💰 Cost Management Stack (NEW)

| Tech | Purpose |
|---|---|
| **Custom CostTracker** | Records tokens in/out + USD per query |
| **BudgetEnforcer** | Blocks queries when monthly cap reached |
| **Redis cache** | Dedupe identical questions (reduces calls) |
| **pgvector top-K cap** | Limits retrieval cost |

---

## 🌊 Data Tiering Stack (NEW)

| Tier | Tech | Lifespan |
|---|---|---|
| **Hot** | Redis Streams (maxlen, TTL) | Seconds to minutes |
| **Warm** | Redis cache (keys) | Minutes to hours |
| **Cold** | MongoDB + pgvector | Long-term (purged at cap) |

**Stream : Store ratio** — configurable (`PERSISTENCE_SAMPLE_RATE`, default 1/30), CRITICAL-priority always persisted.

---

## 📊 LLM Pipeline Summary

Question → Classifier (4 domains)
→ Retrieval (hot/warm/cold tiers)
→ Reranker
→ Prompt (template per domain)
→ Chat model (OpenAI / Llama)
→ Streaming response (SSE)
→ PDF report option (OpenPDF)


**All models global.** Each Person has one embedding row; the chat model answers cross-person queries.

**Default state:** `LLM_ENABLED=false` → mock chat + mock embeddings → zero cost, full pipeline exercisable.

---

## 📦 Build tooling

| Tool | Purpose |
|---|---|
| **npm** | FE package manager |
| **Maven** | BE build |
| **Vite** | FE bundler |
| **Spring Boot Plugin** | Executable JAR |
| **Quarkus Maven Plugin** | Quarkus fast-jar |
| **Vitest** | FE tests |
| **Maven Surefire** | Unit tests |
| **Maven Failsafe** | Integration tests |