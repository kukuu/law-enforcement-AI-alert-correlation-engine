# Real Time Law Enforcement AI ALERT Correlation Engine

 _Author: Alexander Adu-Sarkodie_ : https://www.linkedin.com/in/alexander-adu-sarkodie-063b4b4a/

A correlation engine that ingests alerts from multiple law enforcement systems and uses AI to deduplicate, link, and prioritize related incidents into unified cases. It is a situational awareness for law enforcement operations.


A production-shaped, event-driven platform that ingests alerts, correlates them into incidents, and streams live data to operator dashboards — with tracked persons, geospatial views, full auditability, **and an AI-powered analyst ("Ask COP") that answers natural-language questions grounded in real-time data**.

## 📚 Documentation

| File | Description |
|---|---|
| **[technologies.md](./technologies.md)** | Full tech inventory |


<!--## 🚀 Quick start

```
git clone <repo-url>
cd real-time-alert-correlation-engine/backend

cp .env.example .env
# Edit .env: REDIS_PASSWORD, MONGO_PASSWORD, SUPABASE_*
docker compose up --build -d

# Frontend will be at http://localhost:4000
See Startup.md for the full guide.

```-->
## 🏗️ Architecture at a glance

```
┌──────────┐  REST + STOMP  ┌────────────┐
│ Frontend │◀──────────────▶│API Gateway │
└──────────┘                └─────┬──────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌─────────────┐    ┌─────────────┐   ┌──────────────┐
      │alert-service│    │correlation- │   │  Discovery   │
      │  (Spring)   │    │  engine     │   │  + Config    │
      │             │    │  (Quarkus)  │   │              │
      └──────┬──────┘    └──────┬──────┘   └──────────────┘
             │                  │
       ┌─────┴─────┐            ▼
       ▼           ▼          Kafka
    MongoDB     Redis      (alerts.raw
                          ↔ correlated)
                 │
                 ▼
           Redis Pub/Sub
                 │
                 ▼
           STOMP /topic/*
                 │
                 ▼
           All FE tabs


```
## 🧩 Feature highlights

1. Live dashboard — KPIs refresh within 1 second of an alert

2. Correlation engine — auto-groups related alerts into incidents

3. PERSONS registry — photos, offences, risk levels, filters

4. Geo map — clustering, heatmap, radius, live updates

5. Auth — Supabase JWT, roles & scopes

6. Auto-purge — configurable per-collection caps

7. Monitoring — Prometheus + Grafana out of the box 

<!--## 🧪 Tests
### Frontend
```
cd frontend && npm test
```
#### Backend
```
cd backend && mvn test              # unit + component
cd backend && mvn verify            # + integration (Docker required)
```


## 🚀 Quick start

```

git clone <repo-url>
cd real-time-alert-correlation-engine/backend

cp .env.example .env
# Edit .env: REDIS_PASSWORD, MONGO_PASSWORD, SUPABASE_*
docker compose up --build -d
```
-  Frontend will be at http://localhost:4000
-->
## 🛡️ Safety defaults

- Concern Default Toggle

- External LLM calls  OFF (LLM_ENABLED=false) Env var

- PII to LLM  Simulated (LLM_PII_MODE=simulated)  Env var

- Cost tracking OFF Env var

- Budget enforcement  OFF Env var

- PDF generation  OFF Env var

- Forecasting domain  OFF Env var

- Everything works out of the box in mock mode — full pipeline runs, zero cost, no PII leaves the network.


<!--## 🧪 Tests

### Frontend
cd frontend && npm test

### Backend
cd backend && mvn test              # unit + component
cd backend && mvn verify            # + integration (Docker required)
-->
## 🔐 Security

- JWT verified via Supabase JWKS (RS256) at every layer

- Roles as ROLE_*, scopes as SCOPE_*

- PII encrypted at rest and in transit

- LLM PII handling: simulated / redacted / raw modes (env-controlled)

- Storage RLS policies on photos

<!--## 📞 Support

- Read RUNBOOK.md first

- Check Grafana → Prometheus targets

- Grep logs: docker compose logs -f <service>

- Escalate with: service name, timestamp, error snippet, docker compose ps

- Full audit trail in Postgres + llm_queries
-->
## Snapshots

- https://basepic.com/J9tKBmGT
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/ask-AI-COP.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/backend-build-21-11-2026.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/running-backend-simulation-1.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/running-backend-simulation-2.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/running-backend-simulation-3.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/authenticating-websocket-and-stomp-1.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/authenticating-websocket-and-stomp-2.png
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine/blob/main/document/authenticating-websocket-and-stomp-3.png
<!--https://h.uguu.se/vPCCfXofwkHz.mov-->
  
## Code Repository
- https://github.com/kukuu/real-time-law-enforcement-AI-alert-correlation-engine-vault (**Private**)
        
