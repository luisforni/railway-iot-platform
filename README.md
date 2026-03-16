# railway-iot-platform

> Real-time IoT monitoring and predictive maintenance platform for railway infrastructure.

Built with **Django 5** · **MQTT 5.0** · **TimescaleDB** · **Next.js 14** · **Celery** · **ML Engine (FastAPI)**

---

## Architecture

```
railway-iot-platform/
├── railway-iot-mqtt-broker/        ← Mosquitto MQTT 5.0 broker
├── railway-iot-django-api/         ← REST API + WebSocket + MQTT ingestion
├── railway-iot-celery-worker/      ← Async workers + Beat scheduler
├── railway-iot-ml-engine/          ← Anomaly detection (Isolation Forest)
├── railway-iot-timescale-db/       ← TimescaleDB schema + retention policies
├── railway-iot-dashboard/          ← Real-time dashboard + railway map
├── railway-iot-sensor-simulator/   ← IoT sensor simulator for dev/demo
└── railway-iot-infra/              ← Docker Compose + Nginx
```

## Data Flow

```
[IoT Sensors] → MQTT → [Mosquitto] → [Django MQTT Consumer]
    → [Celery: persist_reading]       → [TimescaleDB]
    → [Celery: check_threshold_alert] → [Active Alerts]
    → [Django Channels] → WebSocket   → [Next.js Dashboard]
    → [ML Engine]       → Anomaly scores → [Alerts]
```

---

## Quick Start

```bash
git clone --recurse-submodules https://github.com/luisforni/railway-iot-platform.git
cd railway-iot-platform

cp railway-iot-infra/.env.example railway-iot-infra/.env
# Edit .env — change SECRET_KEY, DB_PASSWORD, REDIS_PASSWORD, ML_API_KEY before production use

cd railway-iot-infra
docker compose up -d

# Run the sensor simulator (separate profile)
docker compose --profile simulator up -d simulator
```

Open the dashboard at **http://localhost:3100** and log in.

### Default Dev Credentials

| Service | Username | Password |
|---|---|---|
| Django Admin (`/admin/`) | `admin` | `railway2026!` |
| Dashboard login | `admin` | `railway2026!` |

Obtain a JWT token via CLI:

```bash
curl -s -X POST http://localhost:8003/api/v1/auth/token/ \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"railway2026!"}' | jq .
```

---

## Services & Ports

| Service | Port | Description |
|---|---|---|
| Nginx (reverse proxy) | 8003 | Main entry point — routes API + WebSocket |
| Django API (Daphne ASGI) | 8000 | REST API + Django Channels (internal) |
| ML Engine (FastAPI) | 8004 | Anomaly detection inference |
| Dashboard (Next.js) | 3100 | Real-time monitoring dashboard |
| MQTT Broker (Mosquitto) | 1883 | IoT device connectivity (dev: plaintext) |
| PostgreSQL / TimescaleDB | 5432 | Time-series database (internal) |
| Redis | 6379 | Celery broker + Django Channels layer (internal) |

---

## API Endpoints

### Authentication

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/auth/token/` | Obtain JWT access + refresh tokens |
| POST | `/api/v1/auth/token/refresh/` | Refresh an expired access token |

All other endpoints require `Authorization: Bearer <access_token>`.

### Telemetry

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/readings/` | List sensor readings (filterable by device, metric, time) |
| GET | `/api/v1/readings/latest/` | Latest reading per device |
| GET | `/api/v1/zones/` | List zones |
| GET | `/api/v1/devices/` | List devices |

### Alerts

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/alerts/` | List alerts (`?acknowledged=false&limit=50`) |
| GET | `/api/v1/alerts/<id>/` | Alert detail |
| PATCH | `/api/v1/alerts/<id>/` | Acknowledge: `{"acknowledged": true}` |

### ML Engine

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/anomaly/detect/` | `X-Api-Key` header | Isolation Forest on sensor values |
| GET | `/api/v1/anomaly/health` | — | Service health check |

---

## WebSocket Endpoints

WebSocket connections require a valid JWT token passed as a query parameter.

| Path | Description |
|---|---|
| `ws://localhost:8003/ws/telemetry/?token=<jwt>` | Real-time sensor readings |
| `ws://localhost:8003/ws/alerts/?token=<jwt>` | Real-time alert notifications |

Unauthenticated connections are closed with code `4401`.

---

## Sensor Metrics

| Metric | Unit | Normal Range | Warning | Critical |
|---|---|---|---|---|
| `temperature` | °C | 15–85 | ≥ 70 | ≥ 82 |
| `vibration` | mm/s | 0–10 | ≥ 7 | ≥ 9 |
| `rpm` | rpm | 400–1800 | ≥ 1600 | ≥ 1750 |
| `brake-pressure` | bar | 2–8 | ≥ 7 | ≥ 7.8 |
| `load-weight` | kg | 0–80,000 | ≥ 72,000 | ≥ 78,000 |

---

## Security

This platform implements mitigations for the [OWASP Top 10:2021](https://owasp.org/Top10/).

### A01 — Broken Access Control
- All REST endpoints require JWT authentication (`IsAuthenticated` default permission class)
- WebSocket connections authenticated via `?token=<jwt>`; unauthenticated connections closed with code `4401`
- Alert PATCH restricted to `acknowledged` field only

### A02 — Cryptographic Failures
- JWT signed with HS256; `SECRET_KEY` has no default — server refuses to start if unset
- Passwords and API keys in `.env` only, never committed
- Recommended: enable TLS on Nginx and MQTT for production

### A03 — Injection
- All DB queries use Django ORM parameterized queries — no raw SQL string formatting
- `device_id` and `zone` validated with regex `^[\w\-]{1,64}$`
- Metric names validated against a fixed whitelist
- ML Engine caps input at 1,000 values per request

### A04 — Insecure Design
- Inbound MQTT payloads fully validated before persistence (required fields, type bounds, regex, metric whitelist)
- Invalid payloads are silently dropped — no stack traces exposed to clients

### A05 — Security Misconfiguration
- `DEBUG=False` enforced in production; `ALLOWED_HOSTS` explicitly set
- `server_tokens off` in Nginx; `CORS_ALLOW_ALL_ORIGINS = False`
- Security headers: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `X-XSS-Protection`, CSP, `Referrer-Policy`

### A06 — Vulnerable and Outdated Components
- Dependencies pinned in `requirements.txt` / `package.json`; base images use version tags

### A07 — Identification and Authentication Failures
- JWT access token: 60 min lifetime; refresh token: 1 day
- Token blacklist enabled (`rest_framework_simplejwt.token_blacklist`)
- Auth endpoint rate-limited: 10 requests/minute per IP

### A08 — Software and Data Integrity Failures
- ML Engine protected by `X-Api-Key` header validated against `ML_API_KEY` env var
- Docker images built from source

### A09 — Security Logging and Monitoring
- `SecurityAuditMiddleware` logs all 401/403 responses and write operations (POST/PUT/PATCH/DELETE) with client IP and authenticated username

### A10 — Server-Side Request Forgery (SSRF)
- No user-controlled URL fetching; ML Engine only accepts numeric arrays
- Internal services not exposed on host network in production

### Rate Limits

| Layer | Zone / Scope | Limit |
|---|---|---|
| Nginx | `auth` (login/token) | 10 req/min per IP |
| Nginx | `api` (all `/api/v1/`) | 60 req/min per IP |
| Nginx | `ws` (WebSocket upgrades) | 20 conn/min per IP |
| DRF | Anonymous | 30 req/min |
| DRF | Authenticated | 300 req/min |

---

## Modules

| Module | Tech | README |
|---|---|---|
| [railway-iot-mqtt-broker](./railway-iot-mqtt-broker) | Mosquitto 2.x | [→](./railway-iot-mqtt-broker/README.md) |
| [railway-iot-django-api](./railway-iot-django-api) | Django 5 + DRF + Channels | [→](./railway-iot-django-api/README.md) |
| [railway-iot-ml-engine](./railway-iot-ml-engine) | FastAPI + scikit-learn | [→](./railway-iot-ml-engine/README.md) |
| [railway-iot-timescale-db](./railway-iot-timescale-db) | TimescaleDB + PostgreSQL 16 | [→](./railway-iot-timescale-db/README.md) |
| [railway-iot-dashboard](./railway-iot-dashboard) | Next.js 14 + Leaflet | [→](./railway-iot-dashboard/README.md) |
| [railway-iot-sensor-simulator](./railway-iot-sensor-simulator) | Python + paho-mqtt | [→](./railway-iot-sensor-simulator/README.md) |
| [railway-iot-infra](./railway-iot-infra) | Docker Compose + Nginx | [→](./railway-iot-infra/README.md) |
| [railway-iot-celery-worker](./railway-iot-celery-worker) | Celery + Redis | [→](./railway-iot-celery-worker/README.md) |

---

## Target Metrics

| Metric | Target |
|---|---|
| Sensor → Dashboard latency | < 500 ms |
| MQTT throughput | 10,000 msg/min |
| Uptime | 99.9% |
| Telemetry retention | 1 year |

---

*Railway IoT Platform — luisforni*
