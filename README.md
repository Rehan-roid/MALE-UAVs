# Aero Piston Engine Digital Twin — MALE UAV

AI-enabled real-time digital twin for aero piston engine health monitoring, fault
prediction and mission reliability in Medium-Altitude Long-Endurance (MALE) UAVs.

The system pairs an interactive 3D engine twin (React 19 / Three.js / TanStack
Start) with a FastAPI diagnostic backend (L1 telemetry ingestion → L2 physics
digital twin → L3 ML supervision → advisories), talking over REST plus a
real-time WebSocket event stream.

---

## 1. Repository layout

```
.
├── Twin-Piston-eng-frontend/          # UI: 3D digital twin, telemetry, RUL, advisories
│   ├── Dockerfile                     # multi-stage production image (Node SSR server)
│   ├── .env / .env.example            # VITE_* configuration (build-time)
│   ├── public/twin-piston-engine.glb  # 5.9 MB glTF 2.0 engine model
│   └── README.md                      # feature-level frontend documentation
└── Twin_Piston_Engine_project_backend/
    └── Twin_Piston_Engine/            # FastAPI REST + WebSocket API
        ├── Dockerfile                 # multi-stage production image
        ├── docker-compose.yml         # backend-only stack
        ├── .env / .env.example        # APP_* configuration
        ├── src/                       # api, core, l1_data, l2_digital_twin, l3_ml, ...
        ├── models/                    # model cards (no weight artifacts — see §7)
        └── PRODUCTION_DEPLOYMENT.md    # backend deployment guide
```

A **root `docker-compose.yml`** builds and runs both services together.

---

## 2. Architecture

```
Browser ──REST /api/v1/* ───────────────▶ FastAPI (Module 20)
   │                                          │
   └──WS /api/v1/ws/engine ──────────────────▶│  read-only diagnostic event stream
                                              ▼
                                   L1 ingest/validate ─▶ L2 physics ─▶ L3 ML
                                     (HMAC, ranges)      twin     (health index,
                                                         faults,   RUL, risk)
                                                         advisories
```

Ports: **UI 3000**, **API 8000**. Interactive API docs at `http://localhost:8000/docs`.

---

## 3. Quick start (local development)

### Backend

```bash
cd Twin_Piston_Engine_project_backend/Twin_Piston_Engine
uv sync --frozen                              # creates .venv from uv.lock
uv run uvicorn src.api.app:app --host 0.0.0.0 --port 8000 --reload
```

### Frontend

```bash
cd Twin-Piston-eng-frontend
npm ci
npm run dev -- --port 3000 --strictPort --host 127.0.0.1
```

Open <http://127.0.0.1:3000>.

> **Run the UI on port 3000, not 8080.** The backend's CORS allowlist defaults to
> ports 3000 and 8000 only, so a UI on any other port has every REST call blocked
> by the browser. WebSockets are not subject to CORS, which makes the failure
> confusing: the live stream keeps working while REST data silently does not.

Requirements: Python ≥ 3.11 with [uv](https://docs.astral.sh/uv/); Node.js 20.19+
or 22.12+ (Vite 8), Node 22 LTS is used in the container images.

---

## 4. Container deployment

```bash
docker compose up --build          # UI :3000  +  API :8000
```

Individual images:

```bash
# Backend
docker build -t piston-engine-twin-api \
  Twin_Piston_Engine_project_backend/Twin_Piston_Engine

# Frontend — VITE_* are build-time values, they cannot be changed by -e
docker build \
  --build-arg VITE_API_BASE_URL=https://api.example.com/api/v1 \
  --build-arg VITE_WS_URL=wss://api.example.com/api/v1/ws/engine \
  -t piston-engine-twin-web Twin-Piston-eng-frontend
```

### Production checklist

1. `export TELEMETRY_SECRET_KEY=$(openssl rand -hex 32)` — required, see §5.
2. Set `APP_APP_ENV=production`. Startup **fails fast** if the secret above is
   missing.
3. Set `APP_CORS_ALLOW_ORIGINS` to the real UI origin (comma-separated).
4. Rebuild the UI with public `VITE_API_BASE_URL` / `VITE_WS_URL`
   (`https://` + `wss://` — never mix a secure page with a plain `ws://`).
5. Terminate TLS in front of both services.

---

## 5. Environment variables

Both applications ship a committed `.env` containing **non-secret development
defaults** so a clone runs with zero setup. `.env.local` is git-ignored for
per-developer overrides.

### Frontend (`Twin-Piston-eng-frontend/.env`)

| Variable | Default | Notes |
| --- | --- | --- |
| `VITE_API_BASE_URL` | `http://localhost:8000/api/v1` | REST base URL |
| `VITE_WS_URL` | `ws://localhost:8000/api/v1/ws/engine` | WebSocket stream |

Vite **inlines `VITE_*` at build time** — changing them requires a dev-server
restart or an image rebuild, and they are excluded from the Docker build context
so local URLs can never leak into a production bundle.

### Backend (`Twin_Piston_Engine_project_backend/Twin_Piston_Engine/.env`)

| Variable | Default | Notes |
| --- | --- | --- |
| `APP_APP_ENV` | `development` | `development` / `staging` / `production` |
| `APP_LOG_LEVEL` | `INFO` | |
| `APP_CONFIG_PATH` | `config/default.yaml` | core physics configuration |
| `APP_DATABASE_URL` | `sqlite+aiosqlite:///./digital_twin.db` | |
| `APP_API_HOST` / `APP_API_PORT` | `0.0.0.0` / `8000` | |
| `APP_CORS_ALLOW_ORIGINS` | localhost/127.0.0.1 on 3000+8000 | comma-separated; `*` allowed (dev only) |
| `APP_TELEMETRY_SOURCE` | `simulator` | `simulator` / `csv_replay` / `live` |
| `APP_MODEL_DIR` | `models/` | ML artifact directory |
| `TELEMETRY_SECRET_KEY` | *(unset)* | HMAC secret — **name is fixed, no `APP_` prefix** |

### ⚠️ Two naming rules that bite

1. **Every app setting needs the `APP_` prefix.** `AppSettings` uses
   `env_prefix="APP_"`, so `API_PORT=9000` is ignored — the real name is
   `APP_API_PORT`. (Note the double prefix on `APP_APP_ENV`.)
2. **`TELEMETRY_SECRET_KEY` must be a genuine process environment variable.**
   The production check reads `os.environ` directly, so a value that only exists
   inside `.env` does **not** satisfy it. Export it, or set it in the
   orchestrator's environment (the root compose file does the latter).

Also worth knowing: `config/default.yaml` is supplied as constructor arguments
to `AppSettings`, and those outrank environment variables. Settings present in
that YAML (for example `version`) cannot be overridden by env vars; those absent
from it can.

---

## 6. Verification

```bash
curl -s http://localhost:8000/api/v1/system/health      # {"status":"ok",...}
curl -s http://localhost:8000/api/v1/engine/health      # health index + degradation state
curl -s http://localhost:8000/api/v1/engine/rul
curl -s http://localhost:8000/api/v1/mission/risk
curl -s http://localhost:8000/docs                       # Swagger UI
```

In the UI, confirm the twin renders, the header `DATA MODE` badge and the RPM
read-out track the backend, and the browser console is free of CORS errors and of
`WebSocket ... ERR_CONNECTION_REFUSED`.

---

## 7. Known limitations

These are real and current — the stack runs, but it is a prototype:

- **No ML weight artifacts are committed.** `models/` holds only model cards, so
  `GET /api/v1/diagnostics/anomaly` and `/diagnostics/fault` return
  `MODEL_UNAVAILABLE` and deterministic physics rules handle diagnostics instead.
  Regenerate artifacts with the `train` extra and `scripts/train_models.py`.
- **Telemetry is synthetic and deterministic.** Every endpoint derives from a
  seeded 10-second scenario built at startup, not from hardware.
- **The WebSocket stream is finite.** It replays its records, emits
  `STREAM_COMPLETE`, then closes; the UI reconnects every 5 s. Because
  `isConnected` is driven by socket state, the `DATA MODE` badge oscillates
  between `LIVE WEBSOCKET` and `SIMULATION (MOCK)`.
- **The header mixes live and mock values.** `telemetry_update` carries only
  `rpm` and `power_kw`, so other header fields fall back to the mock table.
- **`GET /api/v1/telemetry/latest` returns 404** — nothing ingests into the raw
  telemetry repository. The UI defines but never calls this endpoint.
- **CORS is an explicit allowlist.** Deploying the UI on an unlisted origin
  silently breaks REST data while the WebSocket keeps working.
- **The default Nitro preset is Cloudflare** (`cloudflare-module`). Container
  builds set `NITRO_PRESET=node-server`; a Wrangler deploy needs no such change.

---

## 8. License and provenance

The UI is MIT licensed. The backend declares a **proprietary** license and was
authored separately — confirm ownership and licensing before publishing or
redistributing this backend outside your own accounts.

Backend-specific guidance lives in
[Twin_Piston_Engine_project_backend/Twin_Piston_Engine/PRODUCTION_DEPLOYMENT.md](Twin_Piston_Engine_project_backend/Twin_Piston_Engine/PRODUCTION_DEPLOYMENT.md)
and the module documentation in that directory; UI feature documentation is in
[Twin-Piston-eng-frontend/README.md](Twin-Piston-eng-frontend/README.md).
