WORKFLOW.md
markdown
# 🔄 WORKFLOW — Handshaking Between Services

Detailed step-by-step of every cross-service interaction.

---

## 1. Boot sequence
---

## 1. Boot sequence
```
┌──────────────────────────────────────────────────────────────┐
│ docker compose up -d │
└──────────┬───────────────────────────────────────────────────┘
│
▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ zookeeper │ │ kafka │ │ mongodb │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
│ ready │ ready │ ready
▼ ▼ ▼
┌────────────────────────┐ ┌──────────────────┐
│ service-discovery │◀───│ config-server │
└───────────┬────────────┘ └────────┬─────────┘
│ registers │
▼ ▼
┌────────────────────────────────────────────────────┐
│ alert-service │
│ (mongo, redis, kafka consumer, STOMP server, │
│ LLM/RAG pipeline, person indexer, PDF generator) │
└───────────────────────┬────────────────────────────┘
│ registers
▼
┌────────────────────────────────────────────────────┐
│ api-gateway │
│ (routes /api/, /ws/, /api/llm/**) │
└───────────────────────┬────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────┐
│ frontend (Vite dev / nginx) │
└────────────────────────────────────────────────────┘
```

🆕 Startup additions:**
- `alert-service` runs a **PersonIndexer** bean on boot — re-embeds any Persons updated since last shutdown
- `alert-service` exposes `/api/llm/health` for readiness probing
- If `LLM_ENABLED=true`, alert-service validates the provider connection at startup
---

## 2. Login flow (Supabase → gateway → FE)
┌───────┐ ┌──────────┐ ┌──────────┐
│ FE │ │ Supabase │ │ gateway │
└───┬───┘ └────┬─────┘ └────┬─────┘
│ │ │
│ POST /auth/v1/token (email+pw) │
├────────────────────────────▶│ │
│ │ │
│ 200 {access_token, refresh, user} │
│◀────────────────────────────┤ │
│ │ │
│ store in localStorage('access_token') │
│ │ │
│ GET /api/v1/alerts │
│ Authorization: Bearer <jwt> │
├───────────────────────────────────────────────────────▶
│ │ │
│ │ verify JWT via JWKS
│ │◀────────────────────────┤
│ │ JWKS response │
│ ├────────────────────────▶│
│ │ │
│ │ authorities derived │
│ │ (ROLE_, SCOPE_) │
│ │ │
│ │ forward to │
│ │ alert-service │
│ │ │
│ 200 [alerts] │
│◀───────────────────────────────────────────────────────


---

## 3. Alert creation (manual or simulated)

### 3a. FE "Create Alert" click
FE (NewAlertModal)
│
│ POST /api/v1/alerts [Bearer jwt]
▼
api-gateway ──routes──▶ alert-service
│
├──▶ Mongo: alerts.insert
│
├──▶ Redis: PUBLISH le-events:alerts <json>
│
└──▶ Kafka (if enabled): alerts.raw



### 3b. Backend simulator
alert-service @Scheduled(fixedDelay=15s)
│ creates fake alert
▼
AlertService.create()
│
├──▶ Mongo insert
├──▶ Redis PUBLISH "le-events:alerts"
└──▶ PurgeService.trimAlertsIfNeeded()



### 3c. Redis → STOMP fan-out
Redis ──pattern "le-events:*"──▶ RedisEventSubscriber
│
▼
SimpMessagingTemplate
│
▼
STOMP destination
/topic/alerts
│
▼
All connected FE tabs



## 4. WebSocket handshake (STOMP)
FE (stompClient.connect(token))
│
│ WebSocket upgrade to ws://.../ws (SockJS handshake)
▼
alert-service WebSocketConfig
│
│ STOMP CONNECT frame arrives
▼
WebSocketAuthInterceptor
│ extract Bearer token from headers
│ decode + verify via Supabase JWT decoder
│ set Authentication on session
▼
StompSubscription established
│
│ FE subscribes to /topic/alerts, /topic/incidents, /topic/persons
▼
Live messages flow until disconnect

On user login, the FE also fires `POST /api/llm/subscribe` (optional, gated by `VITE_LLM_ENABLED`) which pre-warms the person embedding cache for the user's jurisdiction.

 On user login, the FE also fires `POST /api/llm/subscribe` (optional, gated by `VITE_LLM_ENABLED`) which pre-warms the person embedding cache for the user's jurisdiction.

## 5. Correlation pipeline (Kafka)
Alert created (multiple within 500m / 30min)
│
│ (publisher to Kafka enabled)
▼
Kafka topic: alerts.raw
│
▼
correlation-engine (Quarkus)
│ @Incoming("alerts") → CorrelationService.correlateAlerts()
│ clusters alerts → RuleEngine.evaluate()
│ if cluster meets threshold → CorrelatedIncident
▼
Kafka topic: correlated-events
│ (JSON: { id, timestamp, location{type,coords}, severity, ... })
▼
alert-service CorrelatedIncidentConsumer
│ @KafkaListener(topics="correlated-events")
│ mapper.toDocument()
│ idempotent upsert by id
▼
├──▶ Mongo: incidents collection
├──▶ Redis PUBLISH le-events:incidents
└──▶ CacheManager evicts dashboard-metrics, incident-metrics
│
▼
Redis fan-out → STOMP /topic/incidents
│
▼
FE prepends incident to session cache




## 6. PERSONS CRUD with photo

FE NewPersonModal
│ user picks file
▼
fileToThumbnailBase64() ──▶ local preview (base64 200px)
│
▼
uploadPersonPhoto() ──▶ Supabase Storage
│ bucket: persons-photos
│ returns publicUrl
▼
POST /api/v1/persons { ..., photoUrl, photoThumbnailBase64 }
│
▼
api-gateway ──▶ alert-service PersonController
│
├──▶ Mongo persons.insert
├──▶ Redis PUBLISH le-events:persons
├──▶ PurgeService.trimPersonsIfNeeded()
│
└──▶ 🆕 Publish le-events:persons.index
│
▼
🆕 PersonIndexer (async)
│
├──▶ PII Redactor (per LLM_PII_MODE)
├──▶ Embedding model (OpenAI / BGE / mock)
├──▶ pgvector UPSERT person_embeddings
└──▶ Log to llm_queries (embedding=record)

## 7. Dashboard KPI refresh
FE Dashboard (React Query, 30s interval)
│
│ GET /api/v1/metrics/dashboard [Bearer jwt]
▼
alert-service MetricsController
│ @Cacheable("dashboard-metrics", key="all")
│ → if Redis cache hit (15s TTL): return immediately
▼
MetricsService.dashboardMetrics()
│ Mongo $group aggregations:
│ alerts by priority
│ alerts by source
│ incidents by severity
│ persons by status
│ compute activeIncidents, unresolvedCritical
▼
JSON payload → FE renders KPI cards + chart


## 8. Service shutdown
Ctrl+C / docker compose down
│
▼
Orderly stop (compose respects depends_on):

frontend → HTTP 4xx until re-up

api-gateway → stops accepting

alert-service → drains STOMP, commits Kafka offsets

correlation-engine (Quarkus graceful shutdown)

platform (discovery, config)

infrastructure (mongo, redis, kafka, zk, pg)


## 9. Port matrix

| Service | Container port | Host port | Protocol |
|---|---|---|---|
| frontend | 4000 | 4000 | HTTP |
| api-gateway | 8080 | 8080 | HTTP / WS |
| alert-service | 8080 | (behind gateway) | HTTP / STOMP |
| correlation-engine | 8081 | 8081 | HTTP / Kafka |
| service-discovery | 8761 | 8761 | HTTP |
| config-server | 8888 | 8888 | HTTP |
| mongodb | 27017 | 27017 | Mongo |
| redis | 6379 | 6379 | Redis |
| kafka | 9092 | 9092/9093 | Kafka |
| zookeeper | 2181 | 2181 | ZK |
| postgres | 5432 | 5432 | Postgres |
| prometheus | 9090 | 9090 | HTTP |
| grafana | 3000 | 3001 | HTTP |

## 🧠 10. Ask COP — Query Flow (NEW)

### 10a. Simple question, no person context
FE /ask-cop (user types: "How many HIGH alerts in the last hour?")
│
│ POST /api/llm/query { question, contextTiers:["hot","warm"] }
│ Authorization: Bearer <jwt>
│ Accept: text/event-stream
▼
api-gateway ──▶ alert-service /api/llm/query
│
├──▶ 🔒 JWT verify + SCOPE_llm:query check
│
├──▶ 🧠 QueryClassifier
│ └─→ domain: "statistical"
│
├──▶ 📊 RAGOrchestrator
│ ├─→ tier:hot (Redis Streams) → 42 HIGH alerts last hour
│ ├─→ tier:warm (Redis cache) → confirmed count
│ └─→ tier:cold (Mongo) → only if tier>threshold
│
├──▶ 🧯 PiiRedactor (no PII in this query — nothing to redact)
│
├──▶ 📝 PromptBuilder(template=statistical)
│ └─→ assemble { question, context, instructions }
│
├──▶ 💬 ChatService (OpenAI / Llama / mock)
│ └─→ stream tokens
│
└──▶ 📤 SSE response
event: token data: "In the"
event: token data: " last"
event: token data: " hour,"
...
event: done data: { citations:[...], cost:0.003, tokensIn:120, tokensOut:84 }

### 10b. Person-scoped question
FE /ask-cop (user types: "Summarize P-10001's risk profile")
│
│ POST /api/llm/query { question, personId:"P-10001" }
▼
api-gateway ──▶ alert-service /api/llm/query
│
├──▶ Classifier: domain = "profile forecasting" (or "statistical" if simple)
│
├──▶ RAGOrchestrator
│ ├─→ tier:cold pgvector search (person_id=P-10001) → 3 chunks
│ ├─→ tier:cold Mongo fetch (person doc) → 1 record
│ └─→ tier:warm Mongo incidents where alertIds overlap → 2 incidents
│
├──▶ PiiRedactor (person name → "Subject P-10001", DOB → age band)
│
├──▶ PromptBuilder(template=profile)
│ └─→ instructions + redacted context + retrieved chunks
│
├──▶ ChatService → stream tokens
│
└──▶ SSE response (with citations array pointing to source doc IDs)

### 10c. PDF report generation
FE Persons page or /ask-cop response
│
│ POST /api/llm/report/person/{personId}?template=summary
▼
api-gateway ──▶ alert-service /api/llm/report/person/{id}
│
├──▶ RAGOrchestrator (same as 10b, but non-streaming)
│ └─→ full generated narrative
│
├──▶ PdfReportService
│ ├─→ FreeMarker template + generated narrative
│ ├─→ OpenPDF render
│ └─→ stamp: PII mode banner ("SIMULATED PII — NOT FOR OPERATIONAL USE")
│
└──▶ Content-Type: application/pdf
Content-Disposition: attachment; filename="P-10001-report.pdf"

## 🧠 11. RAG Pipeline — Internal Detail (NEW)

### 11a. Query classification

| Domain | Trigger patterns | Template |
|---|---|---|
| `statistical` | "how many", "count", "trend", "distribution", "average" | Statistical narrative + tables |
| `profile_forecasting` | "predict", "likely", "risk score", "next offence" | Risk assessment + reasoning |
| `performance` | "response time", "best district", "which unit" | Comparative metrics + rank |
| `optimization` | "where should", "recommend", "improve", "allocate" | Recommendation + rationale |
| `person_profile` | "summarize P-XXXXX", "who is", "tell me about" | Full RAG profile |

**Default fallback:** `statistical` if no pattern matches.


### 11b. Retrieval strategy per tier

| Question contains | Reads hot | Reads warm | Reads cold |
|---|---|---|---|
| "now", "live", "currently" | ✅ | ✅ | ❌ |
| "today", "last N hours" | ✅ | ✅ | ✅ |
| "this week", "historical" | ❌ | ⚠️ if cached | ✅ |
| "all time", "ever" | ❌ | ❌ | ✅ |
| Person-scoped (personId present) | ❌ | ✅ | ✅ |


### 11c. PII modes

| Mode | What's sent to LLM | When to use |
|---|---|---|
| `simulated` (default) | Synthetic surrogates | Dev, test, demos |
| `redacted` | PII fields stripped entirely | Governance pending |
| `raw` | Real PII | **Requires legal approval** |

**Implementation:** PiiRedactor bean runs on **every retrieval chunk** before prompt assembly. The mode is stamped on every response and every PDF.

### 11d. Cost & caching flow
Query arrives
│
▼
Compute cache key: sha256(question + personId + piiMode)
│
▼
Redis GET llm:cache:<key>
│
├─── HIT (TTL 1h) → return cached response, cost=0
│
└─── MISS
│
▼
BudgetEnforcer check (if enabled)
│
├─── OVER → 429 Budget exceeded
│
└─── UNDER
│
▼
RateLimiter check (per-user)
│
├─── OVER → 429 Rate limit exceeded
│
└─── UNDER
│
▼
Run LLM
│
├──▶ CostTracker records tokens + USD
├──▶ Redis SET llm:cache:<key> TTL 1h
└──▶ Return stream

## 🧠 12. Person indexer flow (NEW)
PersonService.create() / .update() / .delete()
│
▼
Publish EventPublisher.publish("persons.index", { personId, op })
│
▼
Redis Pub/Sub le-events:persons.index
│
▼
PersonIndexer.onMessage()
│
├──▶ Fetch Person from Mongo
├──▶ Compose chunks:
│ chunk1: name + aliases + DOB band + status + risk
│ chunk2: offences (one chunk per offence)
│ chunk3: lastKnownLocation + jurisdiction
│
├──▶ For each chunk:
│ ├─▶ PiiRedactor.redact(chunk, mode)
│ ├─▶ EmbeddingService.embed(text)
│ └─▶ VectorStoreClient.upsert(personId, chunkId, vector)
│
└──▶ Log index event to llm_queries (embedding=record)

## 🧠 13. Streaming (SSE) contract (NEW)
POST /api/llm/query
Accept: text/event-stream

← HTTP/1.1 200 OK
← Content-Type: text/event-stream
← Cache-Control: no-cache
← Connection: keep-alive

← event: meta
← data: {"queryId":"q-abc123","domain":"statistical","piiMode":"simulated"}

← event: token
← data: {"t":"In "}
← event: token
← data: {"t":"the "}
← event: token
← data: {"t":"last "}
...
← event: citations
← data: {"sources":[{"type":"alert","id":"seed-alert-42"},{"type":"person","id":"P-10001"}]}

← event: usage
← data: {"tokensIn":120,"tokensOut":84,"costUsd":0.003,"cached":false}

← event: done
← data: {}

text

**Contract rules:**
- `event: meta` — always first
- `event: token` — 0 or more
- `event: citations` — optional, only if RAG returned sources
- `event: usage` — always (even on cache hit, with `cached:true`, `costUsd:0`)
- `event: done` — always last
- On error: `event: error` with `data: {code, message}` then close

## 🧠 14. Complete request path for "Ask COP"
Browser
│
│ 1. User types question, presses Enter
│
▼
AskCopForm.tsx
│
│ 2. POST /api/llm/query (Accept: text/event-stream)
│
▼
api-gateway
│
│ 3. Verify JWT (Supabase JWKS)
│ 4. Check SCOPE_llm:query
│ 5. Forward to alert-service
│
▼
alert-service /api/llm/query
│
│ 6. LlmController.query()
│ - Opens SSE emitter
│ - Passes to LlmQueryOrchestrator
│
│ 7. QueryClassifier.classify(question) → domain
│
│ 8. Cache check (Redis)
│ ├─ HIT: replay cached SSE events → done
│ └─ MISS: continue
│
│ 9. RateLimit + Budget check
│ ├─ OVER: emit error event → done
│ └─ UNDER: continue
│
│ 10. RAGOrchestrator.retrieve(question, domain, personId?)
│ ├─ Hot: Redis Streams (LIVE indicators)
│ ├─ Warm: Redis cache (recent aggregate)
│ └─ Cold: Mongo + pgvector (persistent context)
│
│ 11. PiiRedactor.redact(chunks, mode)
│
│ 12. PromptBuilder.build(domain, question, chunks)
│
│ 13. ChatService.stream(prompt)
│ ├─ provider=openai → stream from GPT-4o-mini
│ ├─ provider=ollama → stream from Llama-3
│ └─ provider=mock → stream deterministic template
│ └─ Each token → emit SSE event
│
│ 14. CostTracker.record(usage)
│
│ 15. CacheStore.set(key, fullResponse, TTL)
│
│ 16. Emit citations + usage + done
│
▼
Browser
│
│ 17. AskCopResponse renders tokens as they arrive
│ 18. Citations become clickable links to source
│ 19. "Download PDF Report" button appears if personId present


## 🧠 15. Complete request path for "Download PDF"
Browser
│
│ 1. User clicks "Download PDF Report" (on /ask-cop or /persons)
│
▼
│ 2. POST /api/llm/report/person/{personId}?template=summary
│ Accept: application/pdf
│
▼
api-gateway → alert-service /api/llm/report/person/{id}
│
│ 3. Same RAG pipeline as Ask COP (10b)
│ - But non-streaming (accumulate full response)
│
│ 4. PdfReportService.generate(person, narrative, citations, meta)
│ - FreeMarker template render (HTML)
│ - OpenPDF render (PDF)
│ - Stamp PII mode banner
│ - Stamp model version + query timestamp
│
│ 5. Return Content-Type: application/pdf
│
▼
Browser
│
│ 6. Browser downloads file: P-10001-2026-10-18.pdf


## 📊 16. Data flow summary — before vs after AI layer

**Before AI:**
Alert events → Mongo + Redis → STOMP → FE (display only)

text

**After AI (additive):**
Alert events → Mongo + Redis → STOMP → FE (display)
│ │
▼ ▼
Redis Streams /api/llm/query
(hot buffer, 2s) │
│ ▼
└───────▶ RAG reads all three tiers ──▶ streaming answer
│
▼
person_embeddings (pgvector)
▲
│
PersonIndexer (re-embeds on CRUD)

text

**Existing flows are untouched.** The AI layer is purely additive; every tier above Mongo is read-only from the AI's perspective.



---
