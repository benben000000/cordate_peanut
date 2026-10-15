# 03 — System Flow Document

System: Invisible Pedagogy AI · Version: 1.0.0

---

## 3.1 End-to-end system architecture

```
[Client Web App (Next.js 14 / Extension)]  ──(WebSocket telemetry, JSON v1.0.0 @10 Hz)──►
      │                                                                                  │
      │  worker: LR softmax + EKF + STC                                                   ▼
      │  overlay: WebGL2 ambient canvas                                        [Cloud Run / local
      │                                                                         FastAPI gateway]
      │                                                                                  │
      │                                                                   (multimodal payload:
      │                                                                    ≤30 windows + keyframe)
      │                                                                                  ▼
      │                                                                       [Gemini API
      │                                                                        gemini-3.8-flash:
      │                                                                        state + cue plan]
      │                                                                                  │
      │                                                                        (signal write)
      │                                                                                  ▼
      └──────(RTDB onValue ≤80 ms p95 + Firestore onSnapshot)──────────── [Firestore/RTDB
                                                                           realtime bus]
                                                                                      │
                                                                        (state mutation)
                                                                                      ▼
                                                                    [Client Ambient UI Overlay]
                                                                     (cue FSM → WebGL render,
                                                                      p99 ≤120 ms local path)
```

Component inventory:

| # | Component | Runtime | Input | Output | Failure behavior |
| --- | --- | --- | --- | --- | --- |
| C1 | TelemetryProvider | browser main thread | DOM events 60 Hz | 100 ms windows | local-only mode |
| C2 | InferenceWorker | Web Worker | windows | `{score, state}` ≤12 ms | last-known state decays to FLOW over 30 s |
| C3 | CueOrchestrator | browser main thread | local + server signals | FSM transitions | drops stale cues > 250 ms |
| C4 | AmbientCanvasLayer | WebGL2 | FSM render commands | peripheral cues | CSS fallback, then no-op |
| C5 | Gateway | FastAPI/uvicorn | WS frames | Gemini calls, RTDB/Firestore writes | breaker → local-only instructions |
| C6 | Spooler | gateway thread | windows/events | NDJSON files | disk guard 2 GB, then drop-oldest |
| C7 | Gemini adapter | gateway | prompt | strict JSON | fallback model → breaker |
| C8 | BigQuery loader | local cron/script | NDJSON | `pedagogy.*` tables | backlog queue, load window widens |

## 3.2 Step-by-step data lifecycle

Scenario: learner hits an off-by-one boundary error in a TypeScript loop
(`for (let i = 0; i <= arr.length; i++)`), at line 142, after 4 minutes in the file.

| t | Actor | Action | State after |
| --- | --- | --- | --- |
| t+0.0 s | Learner | Deletes and retypes the loop bound for the 3rd time; 14 backspaces in 4.2 s | — |
| t+0.1 s | C1 | Window 1041 freezes: `il_p50=212, bs_ratio=0.41, err_repeat=0.50, undo=1` | frame queued, seq=1041 |
| t+0.102 s | C2 | Softmax: `P(FLOW)=0.29 → score=0.71`, state=CONFUSION; crosses `FRICTION_T0=0.62` for the 2nd consecutive window | friction episode opened |
| t+0.118 s | C3 | Interim cue policy fires: `PERIPHERAL_HUE` amber (38°), intensity 0.10, ttl 4000 ms | cue committed at **94 ms decision-to-pixel** |
| t+0.1 s | C5 | Frame 1041 uplinked; gateway appends to 300-frame ring buffer | buffer depth 300 |
| t+2.0 s | C2/C5 | Score stays ≥ 0.72 for 2 windows → escalation token; gateway builds prompt (last 30 windows, keyframe if enabled) | escalation queued, 1 of 10 RPM budget |
| t+2.03 s | C7 | `generateContent` → `gemini-3.8-flash`, 1200 ms timeout | request in flight |
| t+2.71 s | C7 | Response validates: `state=CONFUSION, confidence=0.83, cue=HIGHLIGHT_RANGE(line 142, cols 18–42), intensity=0.24, ttl=9000` | signal `sg_...` created |
| t+2.73 s | C5 | Firestore write `sessions/{sid}/signals/sg_...`; RTDB write `/state` (`src=GEMINI`); NDJSON spool append | durable + fast paths updated |
| t+2.79 s | C3 | RTDB `onValue` fires (63 ms after write); cue queued; age at render = 60 ms < 250 ms stale limit | cue accepted |
| t+2.81 s | C4 | Render: 300 ms fade-in underline-glow on the boundary expression; outside caret exclusion (96 px); luma delta 4% | **semantic cue visible; semantic path latency = 810 ms from escalation, inside the p95 ≤ 2500 ms budget** |
| t+2.8–9.0 s | Learner | Sees the `<=` region glow; changes `<=` to `<`; lint error at 142 clears | friction score 0.74 → 0.31 over 2 windows |
| t+11.8 s | C2 | Score < 0.45 for 2 windows → episode closed as **self-resolved after cue**; `user_reaction.typing_resume_ms=740`, `undo_within_3s=false` | episode record appended |
| t+11.9 s | C4 | `FLOW` confirmation pulse: teal 168°, intensity 0.06, ttl 2400 ms, single pulse | positive signal, sub-perceptual |
| t+11.9 s | C2 | STC scheduler: tag cue queued for `+10 min`, capture cue for `+24 h`; IndexedDB entries written | consolidation scheduled |
| nightly | C8 | `bq load` of spool files → `pedagogy.telemetry_windows`, `pedagogy.scaffolding_events` | analytics ready, 60-day expiry |

What does **not** happen at any point: no modal, no toast, no sound, no focus change, no layout
shift, no text inserted, no autocomplete content generated, no answer revealed. The learner's
retrieval attempt remains theirs; the cue only re-weights attention.

## 3.3 Fallback protocols and circuit breakers

### 3.3.1 WebSocket transport failures

| Trigger | Detection | Action | Recovery |
| --- | --- | --- | --- |
| Packet loss (seq gap > 2) | gateway seq check per session | downlink `{"type":"resend","from_seq":N}`; client replays from 300-frame ring buffer (30 s coverage) | automatic; gap ≤ 30 s fully repaired |
| Client network drop | server ping/pong (20 s / 10 s) | close 1001; client reconnect ladder 250→500→1000→2000→4000→8000 ms, ±20% jitter | buffered frames flush in order on reconnect; if > 30 s buffered, drop oldest-first and mark `telemetry_gap` |
| Server instance recycle (scale-in) | close 1001 on shutdown | same ladder; gateway replays last RTDB state on reconnect via `GET /sessions/{sid}/state` | ≤ 2 s to resumed full mode |
| Tunnel/edge failure (cloudflared) | repeated close 1013/1006 | client enters local-only mode, badge only in debug panel | probe `/healthz` every 30 s; auto-recover |

### 3.3.2 Vertex/Gemini failures (rate limits, errors)

| Trigger | Detection | Action | Recovery |
| --- | --- | --- | --- |
| 429 rate limit | HTTP status on call | breaker OPEN 30 s; escalations rerouted to local-only; next call delayed to respect token bucket | HALF_OPEN probe after 30 s; CLOSED on 1 success |
| 5xx / timeout > 1200 ms | httpx timeout | count failure; ≥ 3 consecutive or ≥ 20% errors over 60 s → OPEN | as above |
| Malformed JSON response | schema validation | 1 retry (`temperature=0`), else count as failure | as above |
| Model unavailable | 404/400 on model ID | failover to `GEMINI_MODELS.fallback` (`gemini-3.6-flash`) once; second failure → OPEN | manual alias update |
| Live session refused (judge mode) | setup error | non-blocking fallback to `generateContent` path | automatic |

Circuit breaker state machine: `CLOSED → (3 fail | 20%/60 s) → OPEN(30 s) → HALF_OPEN(1 probe)
→ CLOSED | OPEN(backoff ×2, cap 300 s)`. State mirrored to RTDB so the judge panel shows it.
While OPEN: local engine keeps a **cue library fallback** (`cues.fallback.json`, 200 entries keyed
by dominant feature cluster = nearest of 200 centroids, cosine ≥ 0.75 required); unmatched
clusters produce no cue — silence is the safe default.

### 3.3.3 Latency guards

| Guard | Rule | Effect |
| --- | --- | --- |
| Stale semantic cue | queue age at render > `STALE_DROP_MS=250` | signal marked `SUPPRESSED_STALE`, counted, never rendered |
| Slow local frame | worker inference p95 > 25 ms over 30 s | reduce sampling to 5 Hz (interim cue budget preserved) |
| Render budget breach | ≥ 5% frames < 58 fps during cue | cue downgraded to CSS opacity-only variant |
| Escalation flood | > 10 escalations/min | token bucket drops with `E_RATE` semantics; local path unaffected |
| Spool disk guard | > 2 GB pending | drop-oldest rotation, `spool_overflow` counter (analytics degradation is acceptable; interaction path never blocks on disk) |

### 3.3.4 Degradation ladder (single source of truth)

```
Full: local + Gemini + RTDB + Firestore
  ↓ Gemini fails        → Local + cached cue library (state shown locally only)
  ↓ RTDB fails          → Local + Firestore long-poll (§listener latency ≤ 400 ms, budgets relax)
  ↓ Firestore fails     → Local + Last signal TTL (cues expire naturally, no new server cues)
  ↓ WS fails            → Local-only (full classifier + cues run on-device; analytics buffered)
  ↓ WebGL fails         → CSS opacity/hue fallback (no radial gradients)
  ↓ Web Worker fails    → 1 Hz rule engine (thresholds on bs_ratio/idle only)
```

Every transition is silent to the learner. The debug panel (`?debug=1`) shows the current rung and
the reason code; the submission demo shows at least one rung transition on purpose.

## 3.4 CueOrchestrator FSM (shared contract for §5)

```
IDLE ──candidate(score ≥ T0 for 2 windows, or signal received)──► CANDIDATE
CANDIDATE ──age ≤ STALE_DROP_MS ∧ target resolves ∧ channel cooldown elapsed──► RENDER
CANDIDATE ──age > STALE_DROP_MS ∨ target unresolved ∨ cooldown active──► DROPPED
RENDER ──fade-in 300 ms complete──► HOLD
HOLD ──ttl elapsed ∨ score < 0.45 for 2 windows ∨ user input at target──► FADE
FADE ──fade-out 800 ms complete──► COOLDOWN
COOLDOWN ──4000 ms per-channel cooldown elapsed──► IDLE
Any state ──user opt-out ∨ prefers-reduced-motion──► IDLE (reduced-motion: opacity-only variants)
Any state ──second cue arrives──► queue depth 1 (replace oldest queued, never render two)
```

Legal transitions are enforced in one module (`cueFsm.ts`, pure reducer, 100% branch-covered in
unit tests); the UI spec (§5.3) consumes the same transition table for timings.
