# 02 — Technical Requirement Document (TRD)

System: Invisible Pedagogy AI · Schema version: 1.0.0 · Document version: 1.0.0

Verification log for external facts in this document: see `docs/07-FREE-TIER-LEDGER.md`.

---

## 2.1 Target tech stack

### 2.1.1 Frontend client

| Layer | Choice | Version / constraint |
| --- | --- | --- |
| Framework | Next.js App Router, **static export** (`output: 'export'`) | 14.2.x — static export because Firebase Hosting Spark cannot run SSR (no Cloud Functions/Cloud Run on Spark) |
| Language | TypeScript, `strict: true`, `noUncheckedIndexedAccess: true` | 5.6.x |
| Styling | Tailwind CSS | 3.4.x, design tokens in §2.1.4 |
| Icons | lucide-react | tree-shaken; editor UI only, never inside cues |
| Client state | Zustand store + one custom FSM module (`cueFsm.ts`) | 5.x |
| Ambient rendering | WebGL2 single-quad radial-gradient shader on a full-viewport `<canvas>` (`pointer-events: none`, `z-index: 9000`); CSS custom-property fallback when WebGL2 is unavailable | GPU frame budget ≤ 2.5 ms |
| Worker compute | One `SharedWorker`-capable module worker: feature freeze, softmax LR, EKF adapter, STC scheduler | p95 inference ≤ 12 ms |
| Real-time listeners | Firebase Web SDK v10 modular (`onSnapshot` for Firestore, `onValue` for RTDB) | 10.x |
| Screen capture | `getDisplayMedia` + `OffscreenCanvas` downscale in worker | opt-in; 2 fps delta, 0.5 fps keyframes |

### 2.1.2 Backend and compute

| Layer | Choice | Constraint |
| --- | --- | --- |
| Gateway | Python 3.12, FastAPI 0.115+, uvicorn (1 worker, uvloop), WebSocket endpoint | ring buffer, token bucket, breaker, NDJSON spool |
| Model transport | HTTPS `generateContent` (canonical) and BidiGenerateContent WS (judge mode only) | 1200 ms timeout on generateContent |
| Auth verification | `firebase-admin` 6.x `verify_id_token` with JWKS cache (refresh on unknown `kid`, TTL 6 h) | no per-request Admin SDK calls beyond verification |
| Judge mode Live API | `gemini-3.8-live` session, 10-min rolling reconnect | presenter toggle only |

### 2.1.3 AI, data, auth services

| Concern | Service | Tier | Notes |
| --- | --- | --- | --- |
| Multimodal classification | Gemini API (`gemini-3.8-flash`) | Free tier via AI Studio key (no billing account) | `gemini-2.0-flash` shut down 2026-06-01; alias registry §2.2.5 |
| Judge-mode streaming | Gemini Live API (`gemini-3.8-live`) | Free tier | 10-min rolling sessions |
| Fast state bus | Firebase Realtime Database | Spark | 10 Hz writes; metered by storage/download, not op count |
| Durable signals + session metadata | Cloud Firestore | Spark | 20,000 writes/day budget → ≤ 744 writes/h design |
| Auth | Firebase Auth (Google + Email link) | Spark (50k MAU) | ID token, 1 h expiry |
| Analytics warehouse | BigQuery sandbox dataset `pedagogy` | No-card sandbox: 1 TiB queries/mo, no streaming inserts, 60-day table expiry | Batch `bq load` only |
| Hosting | Firebase Hosting (static Next export) | Spark (10 GB storage, 360 MB/day transfer) | Demo bundle ≤ 2.5 MB gzip |
| Tunnel (zero-card public URL for WS) | `cloudflared` quick tunnel | free, no account | `wss://*.trycloudflare.com`; adds 40–110 ms RTT |

### 2.1.4 Design tokens (single source of truth)

```css
:root {
  --ip-fade-in-ms: 300;      --ip-fade-out-ms: 800;
  --ip-cooldown-ms: 4000;    --ip-max-intensity: 0.35;
  --ip-pulse-hz: 0.42;       --ip-luma-delta-max: 0.06;
  --ip-caret-exclusion-px: 96;
  --ip-flow-hue: 168;  --ip-confusion-hue: 217;
  --ip-fatigue-hue: 262; --ip-frustration-hue: 38;
  --ip-font-ui: "Inter", system-ui, sans-serif;
  --ip-font-editor: "JetBrains Mono", monospace;
}
```

## 2.2 API architecture and data schemas

### 2.2.1 REST surface

| Method | Path | Auth | Request | Response | Notes |
| --- | --- | --- | --- | --- | --- |
| POST | `/api/v1/sessions` | Firebase ID token | `{client_version, profile}` | `{session_id, ws_url, rtdb_path, expires_at}` | 1 Firestore write |
| GET | `/api/v1/sessions/{sid}/state` | ID token, owner check | — | RTDB state snapshot + last 10 signals | debug/recovery |
| POST | `/api/v1/signals` | ID token with `debug` claim | `Signal` object (§2.2.3) | `{accepted: bool}` | judge/operator injection |
| GET | `/api/v1/analytics/mastery/{user_hash}?from=&to=` | ID token | — | weekly `D(w)`, `PII(w)`, logistic `{L,k,t0}` | proxies BigQuery; 60 s cache |
| GET | `/healthz` | none | — | `{ok, breaker, gemini_p95_ms, queue_depth, spool_backlog}` | liveness |
| GET | `/readyz` | none | — | `{ready}` | readiness (JWKS loaded, spool writable) |
| WS | `/ws/telemetry?sid={sid}` | first-frame token (§2.2.2) | uplink frames | downlink frames | upgrade required; non-WS → 426 |

Error envelope: `{"error":{"code":"E_...","message":"...","retry_after_ms":n}}`. Codes:
`E_AUTH` 401, `E_SCHEMA` 422, `E_RATE` 429, `E_UPSTREAM` 502, `E_BREAKER_OPEN` 503.

### 2.2.2 WebSocket protocol `/ws/telemetry`

Transport: JSON text frames (v1.0.0). Compression: `permessage-deflate` enabled by server.
Close codes: `4401` auth frame missing after 3000 ms; `4408` schema mismatch; `4429` uplink rate
violation (token bucket, 12 tokens/s burst 36); `1011` internal; `1001` rolling restart.

Auth handshake (client → server, within 3000 ms of open):

```json
{"type":"auth","v":"1.0.0","id_token":"<Firebase ID token>","sid":"s_9f3c","client_version":"web-1.0.0"}
```

Server acks: `{"type":"auth_ok","session_id":"s_9f3c","server_ts":1760562301123,"ws_interval_ms":100}`

**Uplink frame (client → server, 10 Hz)** — `type: "w"`:

```json
{
  "type": "w", "v": "1.0.0", "seq": 18422, "ts": 1760562301123, "sid": "s_9f3c",
  "w": {
    "dur_ms": 100,
    "il_p50": 184, "il_p95": 402, "il_cv": 0.71,
    "bs_ratio": 0.34, "idle_ms": 0, "idle_ratio_5s": 0.12,
    "cur_rms": 2.8, "cur_vel_p95": 1180, "jitter_bursts": 2,
    "scroll_rev": 0, "scr_d": 0.041, "focus_sw": 0,
    "err": 2, "undo": 1, "err_repeat": 0.50
  },
  "ctx": {"lang": "typescript", "file_hash": "a91f", "line": 142, "col": 18, "scroll_pct": 0.44, "focused_el": "editor"},
  "local": {"score": 0.68, "state": "CONFUSION", "w_ver": "lr-7"},
  "keyframe": null
}
```

`keyframe` is `{"mime":"image/jpeg","b64":"...","hash64":"0x8f3a..."}` at ≤ 0.5 fps, ≤ 45 KB,
only when capture is enabled and `dHash` Hamming distance ≥ 6 from the previous keyframe.
`seq` is monotonic; receiver requests resend if a gap > 2 (§3.3).

**Downlink frame (server → client)** — `type: "scaffold"`:

```json
{
  "type": "scaffold", "v": "1.0.0",
  "signal": {
    "id": "sg_01JQF3K7M2", "schema": "1.0.0",
    "state": "CONFUSION", "confidence": 0.83, "friction_score": 0.79,
    "channel": "HIGHLIGHT_RANGE",
    "target": {"kind": "code_range", "file_hash": "a91f", "line": 142, "start_col": 18, "end_col": 42},
    "intensity": 0.24, "fade_ms": 300, "ttl_ms": 9000,
    "reason_code": "OFF_BY_ONE_BOUNDS",
    "source": "GEMINI", "decision_ts": 1760562302903
  }
}
```

Also on the downlink: `{"type":"ack","seq":18422}`, `{"type":"resend","from_seq":18405}`,
`{"type":"breaker","state":"OPEN","retry_in_ms":30000}`, `{"type":"pong","ts":...}`.

### 2.2.3 Persistence schemas

**RTDB (fast bus, 10 Hz)** — `/sessions/{sid}/state`:

```json
{"ts":1760562302903,"state":"CONFUSION","score":0.79,"conf":0.83,"src":"GEMINI",
 "cue":{"ch":"HIGHLIGHT_RANGE","i":0.24,"ttl":9000},"breaker":"CLOSED"}
```

Write policy: server writes at most once per 5 s (dirty-check: state change or score delta ≥ 0.12).
Client `onValue` latency SLO: p95 ≤ 80 ms.

**Firestore (durable, low-frequency)**:

| Path | Written | Fields |
| --- | --- | --- |
| `sessions/{sid}` | create + close | `owner_uid`, `started_at`, `ended_at`, `client_version`, `counters{friction_windows, cues, escalations, suppressed}` |
| `sessions/{sid}/signals/{signalId}` | per cue injection only | full `Signal` object (§2.2.2) + `latency_decision_to_pixel_ms`, `user_reaction{typing_resume_ms, undo_within_3s}` |
| `users/{uid}/mastery/{week}` | weekly rollup (cron script) | `d_ratio`, `pii`, `k`, `t0`, `episodes`, `self_resolved` |

Firestore security rules (server-only writes; owner-scoped reads):

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /sessions/{sid} {
      allow read: if request.auth != null && request.auth.uid == resource.data.owner_uid;
      allow write: if false;                       // Admin SDK only
      match /signals/{signalId} {
        allow read: if request.auth != null &&
          request.auth.uid == get(/databases/$(db)/documents/sessions/$(sid)).data.owner_uid;
        allow write: if false;
      }
    }
    match /users/{uid}/{doc=**} {
      allow read: if request.auth != null && request.auth.uid == uid;
      allow write: if false;
    }
  }
}
```

RTDB rules: `/sessions/{sid}/state` read = owner only (`auth.uid == root.child('sessions').child($sid).child('owner_uid').val()` style check with a mirrored `owner_uid` node); writes: `false` (Admin SDK only).

### 2.2.4 On-device classifier, EKF adapter, STC scheduler

Feature vector `x ∈ [0,1]^16` in this fixed order:
`[il_p50, il_p95, il_cv, bs_ratio, idle_ratio_5s, idle_max_norm, cur_rms, cur_vel_p95, jitter_bursts_norm, scroll_rev, scr_d, focus_sw, err_norm, undo_norm, err_repeat, session_minutes_norm]`.

Classifier: `P(y|x) = softmax(Wx + b)`, `W ∈ R^{4×16}`, weights JSON ~12 KB, shipped read-only;
versions tagged `lr-{n}`; mismatch between client and gateway versions is logged, not blocked.

EKF adapter (paper's personalization construct, operationalized):

- State: `θ = [W_row(3−y_FLOW rows flattened), b, f]` where `f` = latent fatigue (0–1).
- Predict: `θ⁻ = θ`, `P⁻ = P + Q`, `Q = 1e-4·I`.
- Update on Gemini label `z` with confidence `c ≥ 0.70`: `R = (0.15/c)²`;
  `K = P⁻Hᵀ(H P⁻ Hᵀ + R)⁻¹`, `θ = θ⁻ + K(z − h(θ⁻))`, `P = (I − KH)P⁻`.
- Constraints: clamp `f ∈ [0,1]`; weight L2 drift from base ≤ 0.25 per session; reset on new user.
- Rationale: keeps the fast path personalized without retraining and without server round-trips.

STC scheduler (paper's consolidation layer, operationalized):

- Episode = friction windows with score ≥ `FRICTION_T0` ending when score < 0.45 for 2 windows.
- On episode close: schedule tag cue at `+10 min ± 30 s` and capture cue at `+24 h ± 2 h`
  (client-side queue in IndexedDB; delivered only if a session is active, else dropped after 2× window).
- Budget: max 2 scheduled items per episode; recall cues use `ANCHOR_PULSE` intensity 0.10–0.15.

### 2.2.5 Gemini prompt contract

Model registry (config aliases, not hard-coded IDs):

| Alias | Model ID | Use | Status check |
| --- | --- | --- | --- |
| primary | `gemini-3.8-flash` | structured state + cue plan | GA 2026-09-02 |
| fallback | `gemini-3.6-flash` | on 429/5xx escalation | GA 2026-07-21 |
| live | `gemini-3.8-live` | judge-mode streaming | GA 2026-09-15 |
| retired | `gemini-2.0-flash` | not selectable; retained in docs only | shut down 2026-06-01 |

System instruction (verbatim, ~90 tokens):

```
You classify a learner's cognitive state from interaction telemetry and an optional screen keyframe.
Output only the JSON object matching the provided schema. States: FLOW, CONFUSION (wrong mental
model, active search), FATIGUE (cadence decay + long idle), FRUSTRATION (error repeats + undo/backspace
storms). Cue selection: prefer the least intrusive channel that restores the next retrieval attempt.
Never suggest the literal answer; anchor attention to the fault region. reasoning ≤ 60 tokens.
```

Request body (HTTPS `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent`):

```json
{
  "systemInstruction": {"parts": [{"text": "<SYS>"}]},
  "contents": [{"role": "user", "parts": [
    {"text": "{\"ctx\":{\"lang\":\"typescript\",\"line\":142},\"windows\":[ ...last 30 uplink `w` objects... ],\"local\":{\"score\":0.71,\"state\":\"CONFUSION\"},\"recent_errors\":[\"TS2532@142\",\"TS2532@142\"]}"},
    {"inlineData": {"mimeType": "image/jpeg", "data": "<base64 keyframe, optional>"}}
  ]}],
  "generationConfig": {
    "temperature": 0.1, "topP": 0.8, "maxOutputTokens": 220,
    "responseMimeType": "application/json",
    "responseSchema": {
      "type": "OBJECT",
      "properties": {
        "state": {"type": "STRING", "enum": ["FLOW","CONFUSION","FATIGUE","FRUSTRATION"]},
        "confidence": {"type": "NUMBER"},
        "friction_score": {"type": "NUMBER"},
        "cues": {"type": "ARRAY", "maxItems": 2, "items": {"type": "OBJECT", "properties": {
          "channel": {"type": "STRING", "enum": ["GLOW_RADIAL","PERIPHERAL_HUE","ANCHOR_PULSE","HIGHLIGHT_RANGE","AUTOCOMPLETE_PACING","DIM_DISTRACTORS"]},
          "target": {"type": "STRING"},
          "intensity": {"type": "NUMBER"},
          "ttl_ms": {"type": "INTEGER"},
          "reason_code": {"type": "STRING"}
        }, "required": ["channel","intensity","ttl_ms"]}},
        "reasoning": {"type": "STRING"}
      },
      "required": ["state","confidence","friction_score","cues"]
    }
  },
  "safetySettings": [
    {"category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_ONLY_HIGH"},
    {"category": "HARM_CATEGORY_DANGEROUS_CONTENT", "threshold": "BLOCK_ONLY_HIGH"}
  ]
}
```

Validation gate before applying: JSON parses, `state` in enum, `confidence` ∈ [0,1],
`intensity ≤ 0.35` (clamped, violation logged as `INTENSITY_CLAMPED`), `ttl_ms ∈ [6000, 12000]`
(clamped), target resolves to a DOM/editor anchor within 150 ms else the cue is dropped
(`TARGET_UNRESOLVED`). Malformed JSON: 1 retry with `temperature: 0` and schema reminder, then breaker.

Judge-mode Live session setup (BidiGenerateContent, WS):

```json
{"setup": {
  "model": "models/gemini-3.8-live",
  "generationConfig": {"responseModalities": ["TEXT"], "temperature": 0.1},
  "systemInstruction": {"parts": [{"text": "<SYS> + stream brief reasoning; JSON only on state change"}]}
}}
```

Frames sent: `realtimeInput.mediaChunks` (image/jpeg, 1 per 2 s) + `realtimeInput.text` (telemetry
digest every 2 s). Session capped at 10 min, then rolling reconnect; reconnect failure falls back
to the `generateContent` path without user-visible change.

### 2.2.6 BigQuery dataset and load procedure

DDL (create once in the sandbox project):

```sql
CREATE SCHEMA IF NOT EXISTS pedagogy
OPTIONS (default_table_expiration_days = 60, location = 'US');

CREATE TABLE IF NOT EXISTS pedagogy.telemetry_windows (
  ts TIMESTAMP NOT NULL, session_id STRING NOT NULL, user_hash STRING NOT NULL,
  window_ms INT64, friction_score FLOAT64, state STRING, confidence FLOAT64,
  source STRING, client_version STRING, payload JSON
)
PARTITION BY DATE(ts) CLUSTER BY session_id, user_hash;

CREATE TABLE IF NOT EXISTS pedagogy.scaffolding_events (
  ts TIMESTAMP NOT NULL, session_id STRING NOT NULL, user_hash STRING NOT NULL,
  signal_id STRING, channel STRING, intensity FLOAT64, source STRING, reason_code STRING,
  latency_decision_to_pixel_ms INT64, suppressed BOOL, typing_resume_ms INT64, undo_within_3s BOOL
)
PARTITION BY DATE(ts) CLUSTER BY channel, session_id;

CREATE VIEW IF NOT EXISTS pedagogy.mastery_weekly AS
SELECT user_hash, DATE_TRUNC(DATE(ts), WEEK) AS week,
       SAFE_DIVIDE(COUNTIF(suppressed = FALSE), COUNT(*)) AS d_ratio,
       COUNT(DISTINCT session_id) AS sessions
FROM pedagogy.scaffolding_events GROUP BY user_hash, week;
```

Load procedure (sandbox-compatible, no streaming inserts):

```bash
# gateway spool: ./spool/YYYY-MM-DD/{sid}.ndjson  (30 s flush, hourly file rotation)
bq --project_id="$IP_PROJECT" load \
  --source_format=NEWLINE_DELIMITED_JSON \
  pedagogy.telemetry_windows \
  spool/2026-10-15/*.ndjson
```

Cost guard: one load job per day normally; demo mode allows hourly; each file ≤ 200 MB
(a 4-hour session at 10 Hz ≈ 29 MB raw JSONL).

## 2.3 GCP infrastructure and deployment topology

### 2.3.1 Profiles

| | **zero-card (default, contest requirement)** | **free-safe cloud (billing optional)** | **pinned-demo (billing, not free)** |
| --- | --- | --- | --- |
| Gateway host | local uvicorn + `cloudflared` quick tunnel | Cloud Run, `us-central1` | Cloud Run, `us-central1` |
| Scaling | n/a (single process) | `min_instances=0`, `max_instances=4`, `concurrency=80`, `cpu=1`, `mem=512Mi`, `timeout=3600s`, request-based billing, CPU throttling on | `min_instances=1`, `max_instances=10`, `concurrency=80`, `cpu=1`, `mem=1Gi`, always-allocated CPU |
| Frontend | `pnpm dev` / `next start` on `localhost:3000` | Firebase Hosting static export | Firebase Hosting |
| AI transport | AI Studio free tier (`GEMINI_TRANSPORT=aistudio`) | same, or Vertex after enabling billing | Vertex AI (`aiplatform.googleapis.com`) |
| Analytics | BigQuery sandbox | BigQuery sandbox | BigQuery (scheduled loads) |
| Cost | $0 | $0 within grants | see §2.3.3 quota overage |

### 2.3.2 Zero-card topology (canonical for the hackathon)

```
┌─ Browser (judge or learner) ─────────────────────────────────────────────┐
│ Next.js static bundle (Firebase Hosting or localhost:3000)              │
│ Worker: LR + EKF · CueOrchestrator FSM · AmbientCanvasLayer (WebGL2)    │
└──────┬──────────────────────────────────────────────▲───────────────────┘
       │ wss (telemetry 10 Hz, JSON)                  │ RTDB onValue (state)
       │                                              │ Firestore onSnapshot (signals)
┌──────▼──────────────────────────┐     ┌─────────────┴───────────────┐
│ Gateway: local uvicorn :8080    │     │ Firebase Spark              │
│ exposed via cloudflared quick   │     │  Auth · Firestore · RTDB    │
│ tunnel (wss://*.trycloudflare)  │     │  Hosting                    │
│ JWT verify · ring buf 300 fr    │     └─────────────────────────────┘
│ bucket 10 RPM · breaker · spool │
└──────┬──────────────────────────┘
       │ HTTPS generateContent (1200 ms timeout)
┌──────▼──────────────────────────┐     ┌─────────────────────────────┐
│ Gemini API free tier            │     │ BigQuery sandbox            │
│ gemini-3.8-flash / 3.8-live     │     │ pedagogy.* (60-day expiry)  │
└─────────────────────────────────┘     └──────────▲──────────────────┘
                                                   │ bq load (batch)
                                          spool/*.ndjson (local disk)
```

### 2.3.3 Cloud Run deployment (free-safe and pinned-demo profiles)

```bash
gcloud run deploy ip-gateway \
  --source services/gateway \
  --region us-central1 --allow-unauthenticated \
  --min-instances 0 --max-instances 4 --concurrency 80 \
  --cpu 1 --memory 512Mi --timeout 3600 \
  --set-env-vars GEMINI_TRANSPORT=aistudio,SCHEMA_VERSION=1.0.0 \
  --set-secrets GEMINI_API_KEY=ip-gemini-key:latest    # Secret Manager (billing profile)
```

WebSocket behavior on Cloud Run: connections are pinned to one instance; `--timeout 3600` allows
60-minute sessions; uvicorn `--ws-ping-interval 20 --ws-ping-timeout 10`. Client reconnects with
the backoff ladder in §3.3, so instance recycling after idle scale-in is invisible.

Instance math: 1 vCPU FastAPI instance sustains ~80 concurrent WS connections at 10 Hz with
≤ 35% CPU (measured target), so `max_instances=4` covers 320 concurrent sessions;
`concurrency=80` matches the WS capacity per instance. Cloud Load Balancing is **not required**:
Cloud Run's managed front end provides TLS, HTTP/2, and WS upgrades. A global external
Application Load Balancer is only justified past ~1,000 concurrent sessions or for multi-region.

Free-grant fit check (request-based, us-central1, verified 2026-10-15):
`180,000 vCPU-s + 360,000 GiB-s + 2M requests/month`.
Free-safe profile at 1 vCPU / 512 MiB: idle-to-zero outside demos costs $0; a 2-hour demo day
(active ~20 min at low concurrency) consumes < 4,000 vCPU-s. Fits with ≥ 40× headroom.

Pinned-demo profile cost warning: `min_instances=1` at 1 vCPU / 1 GiB consumes ≈ 2.63M vCPU-s and
≈ 1.31M GiB-s per month — **14.6× and 3.65× the free grants**; the pinned profile is not
free-tier-compliant and is documented only because the original brief requested `min_instances=1`.
Use it for the recorded demo only, then scale back to 0.

### 2.3.4 Least-privilege IAM (billing profile only)

| Principal | Roles | Scope |
| --- | --- | --- |
| `sa-ip-gateway@$PROJECT` (runtime) | `roles/aiplatform.user`, `roles/datastore.user`, `roles/bigquery.dataEditor`, `roles/bigquery.jobUser` | project; no Editor/Owner |
| `sa-ip-deploy@$PROJECT` (CI/deploy) | `roles/run.admin`, `roles/artifactregistry.writer`, `roles/iam.serviceAccountUser` (only on `sa-ip-gateway`) | project |
| Human operator | `roles/run.viewer`, `roles/logging.viewer` | project |

Zero-card profile requires no GCP IAM: Firebase service-account key (from Firebase console,
generated on Spark at no cost) is used by the gateway for `firebase-admin`, stored in
`secrets/firebase-admin.json` (git-ignored) and passed via `GOOGLE_APPLICATION_CREDENTIALS`.

### 2.3.5 Observability

Logging: structured JSON on stdout (`ts, level, sid, seq, stage, latency_ms, decision, reason`),
which Cloud Logging ingests within the 50 GiB/month free grant. No paid dashboards.
Metrics surface: `/healthz` counters (`escalations_total, suppressed_total, breaker_opens_total,
spool_backlog_files, gemini_p95_ms`) scraped by an optional 1-minute local cron script into RTDB
for the judge panel. Latency percentiles are computed client-side (ring buffer 600 samples =
1 minute of windows) and mirrored to RTDB every 30 s under `/sessions/{sid}/metrics`.
