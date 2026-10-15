# Invisible Pedagogy AI

Ambient AI scaffolding engine. Observes workspace telemetry at 10 Hz, classifies cognitive state
(`FLOW | CONFUSION | FATIGUE | FRUSTRATION`) locally and with Gemini multimodal inference, and injects
peripheral micro-cues into the workspace without modals, prompts, or focus theft.

Research basis: **"Invisible Pedagogy: An Algorithmic and Neuroplasticity Model for Ambient Scaffolding
and Implicit Skill Acquisition"** — Benedict Garcia, Zenodo, 2026-06-30, DOI
[10.5281/zenodo.21052875](https://doi.org/10.5281/zenodo.21052875) (CC-BY-4.0). Implementation maps the
paper's POMDP formulation to the two-speed inference loop, the EKF personalization to the on-device
weight adapter, and the Synaptic Tagging & Capture consolidation layer to the spaced-recall scheduler
(`+10 min` tag, `+24 h` capture). See `NOTICE.md`.

---

## Hard constraints

| Constraint | Value |
| --- | --- |
| Budget | GCP always-free grants only, no billing account required for the default deployment profile |
| Model surface | `gemini-3.8-flash` (structured inference), `gemini-3.8-live` (judge mode) — `gemini-2.0-flash` reached shutdown 2026-06-01 and is not used |
| Measured intervention path | Local rule engine + linear classifier, p99 ≤ 120 ms decision-to-pixel |
| Semantic recalibration path | Gemini call every 2–10 s, p95 ≤ 2.5 s; suppressed if stale > 250 ms at render |
| Client telemetry volume | 10 Hz windows, 16-dim feature vector, ≤ 1.2 KB/frame uplink |
| Firestore write budget | ≤ 744 writes/hour (Spark plan limit: 20,000 writes/day) |

## Repository map

| Path | Contents |
| --- | --- |
| `docs/01-PRD.md` | Product requirements, personas, metrics, acceptance gates |
| `docs/02-TRD.md` | Stack, schemas, endpoints, GCP topology, scaling and quota profiles |
| `docs/03-SYSTEM-FLOW.md` | End-to-end data lifecycle, sequence diagrams, fallback protocols |
| `docs/04-IMPLEMENTATION-PLAN.md` | 48-hour sprint plan, phase gates, demo storyboard, submission checklist |
| `docs/05-UIUX-SPEC.md` | Ambient interface spec, component contracts, motion timings, a11y rules |
| `docs/06-GITHUB-DEPLOYMENT.md` | Git identity, commit sequence, remote binding, push workflow |
| `docs/07-FREE-TIER-LEDGER.md` | Quota ledger with verification dates, per-service budget math |
| `NOTICE.md` | CC-BY-4.0 attribution for the source research paper |

## Architecture (one screen)

```
Client (Next.js 14, static export)
├── TelemetryProvider      60 Hz rAF sampling → 100 ms windows @ 10 Hz
├── Web Worker             softmax LR (16-dim) + EKF adapter → p95 ≤ 12 ms
├── CueOrchestrator        FSM: IDLE→CANDIDATE→RENDER→HOLD→FADE→COOLDOWN
└── AmbientCanvasLayer     WebGL2 full-viewport, pointer-events:none, z=9000
        │ uplink WS (10 Hz)          ▲ RTDB listener + Firestore onSnapshot
        ▼                            │
Gateway (FastAPI + uvicorn)          │
├── /ws/telemetry          JWT verify, 300-frame ring buffer, token bucket (10 RPM)
├── prompt builder         ring buffer + luma keyframe (640×360 JPEG q55, ≤ 45 KB)
├── circuit breaker        3 consecutive failures OR ≥ 20% errors/60 s → OPEN 30 s
└── NDJSON spool           hourly rotation → `bq load` → BigQuery sandbox (60-day expiry)
        │ HTTPS generateContent, 1200 ms timeout, JSON response schema
        ▼
Gemini API (free tier) ──► Firestore (signals, durable) + RTDB (state bus, 10 Hz)
```

## Quickstart (zero-card profile)

```bash
# 1. Client
pnpm install
cp .env.example .env.local        # fill NEXT_PUBLIC_FIREBASE_*, GATEWAY_WS_URL
pnpm dev                          # http://localhost:3000

# 2. Gateway (Python 3.11+)
python -m venv .venv && . .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r services/gateway/requirements.txt
export GEMINI_API_KEY=...                          # AI Studio key, no billing account
uvicorn services.gateway.main:app --host 0.0.0.0 --port 8080

# 3. Public tunnel for remote demo judges (no account required)
cloudflared tunnel --url http://localhost:8080
```

Firebase config and rules live in `infra/firebase/`. Cloud Run profile (billing optional) is documented
in `docs/02-TRD.md` §2.3 with both a `free-safe` and a `pinned-demo` scaling profile.

## Environment variables

| Variable | Scope | Default | Notes |
| --- | --- | --- | --- |
| `GEMINI_MODEL_ID` | gateway | `gemini-3.8-flash` | Config alias; swap without code change |
| `GEMINI_LIVE_MODEL_ID` | gateway | `gemini-3.8-live` | Judge-mode Live API sessions only |
| `FRICTION_T0` | both | `0.62` | Interim cue threshold |
| `ESCALATE_T1` | gateway | `0.72` | Semantic escalation threshold, ≥ 2 consecutive windows |
| `SEMANTIC_MIN_INTERVAL_MS` | gateway | `10000` (demo burst `2000`) | Rate-saver cadence |
| `STALE_DROP_MS` | client | `250` | Suppress semantic cues older than this at render |
| `INTERIM_CUE_BUDGET_MS` | client | `120` | p99 decision-to-pixel for local cues |
| `SCHEMA_VERSION` | both | `1.0.0` | Frame schema, reject on mismatch in debug mode |

## License

Code: Apache-2.0 (recommended default; change before submission if the contest requires otherwise).
Documentation: CC-BY-4.0, matching the source paper. Attribution in `NOTICE.md`.
