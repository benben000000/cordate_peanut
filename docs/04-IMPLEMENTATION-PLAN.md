# 04 — Implementation Plan (48-Hour Execution Sprint)

Team model: 3 builders — **A** (infra/backend), **B** (frontend/UX), **C** (AI/eval/demo).
Solo fallback: run phases linearly; the cut list in each gate is mandatory reading before hour 12.

Operating rules:
1. `main` stays deployable; feature work happens in `feat/*` branches merged at gate completion.
2. Every gate is a go/no-go with a measured number, not a judgment call.
3. Cut list items are decided at the gate, not mid-phase.
4. All secrets stay in `.env.local` / CI secrets; nothing sensitive is committed (§06).

---

## 4.1 Milestone breakdown

### Phase 1 — Hours 0–12: core infrastructure and telemetry

| Hours | Task | Owner | Deliverable | Gate evidence |
| --- | --- | --- | --- | --- |
| 0–1 | Repo init, identity, branch protection off, `.env.example`, Firebase project + Spark verify | A | repo pushed, `firebase-debug.log` clean | `git log --oneline`, console screenshot |
| 1–3 | Gateway skeleton: FastAPI app, `/healthz`, `/ws/telemetry` echo, JWT verify (JWKS) | A | WS round-trip to `ws://localhost:8080` | `wscat` transcript |
| 3–5 | TelemetryProvider: rAF sampler, 16-dim feature extraction, 10 Hz window emit | B | frames in console, `local.score` present | 60 s capture, overhead check |
| 5–7 | Firebase binding: Auth sign-in, RTDB `/sessions/{sid}/state` write from gateway, `onValue` client | A+B | state doc updates visible live | 2 terminals screenshot |
| 7–9 | Ring buffer 300 fr, token bucket, reconnect ladder, seq gap resend | A | kill/restart gateway mid-session → frames repaired | lossy-network simulation (`tc`/devtools offline toggling) |
| 9–11 | cloudflared quick tunnel + judge access path; CORS, 426 non-WS | A | `wss://*.trycloudflare.com` reachable from a phone on cellular | device test |
| 11–12 | **Gate G1**: 2-hour soak run; CPU ≤ 1.5%; no memory growth > 5%/h; 10 Hz jitter p95 ≤ 12 ms | all | soak report in `docs/evidence/G1.md` | numbers filled, or apply cut list |

G1 cut list (if behind): drop screen-delta channel (keep DOM-derived channels); drop Auth to
anonymous mode for the demo (keep the code path, disable enforcement via env flag).

### Phase 2 — Hours 12–24: Gemini pipeline and classifier

| Hours | Task | Owner | Deliverable | Gate evidence |
| --- | --- | --- | --- | --- |
| 12–14 | Prompt builder: 30-window digest, keyframe encoder (worker, 640×360 q55), redaction rectangles | C | payload ≤ 45 KB keyframe, ≤ 2 KB text | sample payload dumped |
| 14–16 | Gemini adapter: `generateContent`, responseSchema validation, 1200 ms timeout, 1 retry | C | strict JSON in, validated `Signal` out | 20 consecutive live calls |
| 16–18 | Token bucket (10 RPM), breaker FSM, fallback library (`cues.fallback.json`, 200 entries), `breaker` downlink frames | A | drill: force 429s via rapid calls → OPEN → HALF_OPEN → CLOSED | breaker log transcript |
| 18–20 | EKF adapter + STC scheduler in worker; versioned weights `lr-7` | C | weights update on ≥ 0.70-confidence labels | before/after weight diff |
| 20–22 | Eval harness: 500 annotated windows (recorded sessions + scripted friction scripts), confusion matrix, top-2 accuracy | C | eval report | **macro-F1 ≥ 0.85, top-2 ≥ 0.95, false-friction ≤ 8%** |
| 22–24 | **Gate G2**: escalation p95 ≤ 1.8 s over 50 live calls; eval thresholds met | all | `docs/evidence/G2.md` | numbers filled, or cut list |

G2 cut list: miss accuracy → ship rule-engine-only mode and keep Gemini as a visible-but-advisory
overlay (judge panel shows reasoning while cues come from rules); miss latency → raise cadence to
`SEMANTIC_MIN_INTERVAL_MS=10000` and present the 2.5 s p95 budget as measured-against-plan in docs.

### Phase 3 — Hours 24–36: ambient UI

| Hours | Task | Owner | Deliverable | Gate evidence |
| --- | --- | --- | --- | --- |
| 24–26 | AmbientCanvasLayer: WebGL2 radial shader, viewport-independent, `pointer-events:none`, luma cap 6% | B | 60 fps at 1920×1080, GPU ≤ 2.5 ms | perf capture |
| 26–28 | CueOrchestrator FSM (§3.4), queue depth 1, cooldowns, carets exclusion (96 px), text bbox mask + 12 px | B | FSM unit tests 100% branches | `vitest --coverage` |
| 28–30 | Cue catalog v1: GLOW_RADIAL, PERIPHERAL_HUE, ANCHOR_PULSE, HIGHLIGHT_RANGE | B | all four renderable from injected `Signal`s | 4 screenshots |
| 30–32 | AUTOCOMPLETE_PACING (editor timing only, no content), DIM_DISTRACTORS (peripheral dimming ±6%) | B | both behind flags | screenshots |
| 32–34 | Debug/presenter panel: state chip, score sparkline (120 samples), reasoning trace, latency waterfall, breaker status | B | `?debug=1` full panel at 1920×1080 without scrolling | screenshot |
| 34–36 | **Gate G3**: blind intra-team test — 20 injections; non-intrusive ≥ 80%; false-friction ≤ 8%; reduced-motion pass | all | `docs/evidence/G3.md` | numbers filled, or cut list |

G3 cut list: drop DIM_DISTRACTORS and AUTOCOMPLETE_PACING to flags-off; if WebGL is unstable,
ship CSS fallback as default and keep WebGL behind a flag.

### Phase 4 — Hours 36–48: analytics, demo, submission

| Hours | Task | Owner | Deliverable | Gate evidence |
| --- | --- | --- | --- | --- |
| 36–38 | Spool → BigQuery: `bq load` script, DDL applied, verify counts match spool lines | A | tables populated; `mastery_weekly` view works | row counts |
| 38–40 | Skill Mastery Dashboard: mastery curve (logistic fit), PII trend, episode table (seeded demo data clearly labeled) | B | dashboard route renders ≤ 800 ms | screenshot |
| 40–42 | Deploy: Firebase Hosting (static export) + free-safe Cloud Run **or** zero-card local serving; verify judge path end-to-end from a clean browser profile | A | public URL + `wss` tunnel | checklist ticked |
| 42–45 | Demo video: 3:00 script (§4.4) recorded at 1080p60, three friction types shown, reasoning panel on screen | C+B | mp4 + YouTube unlisted link | video link |
| 45–47 | Deck PDF (10 slides: problem, model, architecture, live metrics, free-tier ledger, roadmap) | C | `slides/invisible-pedagogy-deck.pdf` | rendered pages |
| 47–48 | **Gate G4 (submission freeze)**: README final, repo public, video link in README, deck linked, tag `v1.0.0` | all | submission checklist §4.2 all checked | tag pushed |

## 4.2 Submission requirements checklist (Google Cloud AI Builder Cup)

Re-verify item wording against the current published rubric before submitting; the four hard
deliverables from the brief are tracked here:

| # | Requirement | Artifact | Status check |
| --- | --- | --- | --- |
| 1 | Working GCP prototype | Public URL (Hosting or tunnel) + `/healthz` returning `ok:true` during judging window | uptime note in README, 1-min health probe log |
| 2 | Public GitHub repo | `https://github.com/benben000000/cordate_peanut`, Apache-2.0, no secrets, README quickstart works from a clean machine | fresh-clone dry run on a second machine/VM |
| 3 | 3-minute video | YouTube unlisted link in README §Demo; ≤ 3:00; shows live friction detection + cue + debug panel | duration check |
| 4 | Deck PDF | `slides/invisible-pedagogy-deck.pdf` in repo; ≤ 10 slides; cites DOI and free-tier ledger | exported PDF committed |
| 5 | Judges' test path | `docs/06` §6.7 judge quickstart: 3 commands, demo script triggers (keyboard shortcuts to inject scripted friction) | walkthrough timed ≤ 5 min |
| 6 | AI usage disclosure | README section: model IDs (`gemini-3.8-flash`, `gemini-3.8-live`), prompt contract link, limits hit | disclosure present |
| 7 | Attribution | `NOTICE.md` CC-BY-4.0 for the source paper | present |
| 8 | Free-tier compliance claim | `docs/07-FREE-TIER-LEDGER.md` with verification dates | present |

## 4.3 Risk register

| # | Risk | P | Impact | Mitigation | Trigger to act |
| --- | --- | --- | --- | --- | --- |
| R1 | Gemini free-tier RPM/RPD lower than assumed | M | cue cadence too slow | cadence 10 s default; burst 2 s ≤ 30 s; local path independent | 429 observed twice → apply G2 cut list |
| R2 | Cloud Run WS idle timeout drops connection during quiet judging | M | visible gap in judge panel | 20 s pings; reconnect ladder; RTDB state replay on reconnect | any 1001 in soak → shorten ping to 15 s |
| R3 | Firestore write budget exceeded (unbounded retry loop) | L | writes throttled → no durable signals | RTDB for 10 Hz, Firestore only on signal/session events; write counter alarm in `/healthz` | write counter > 80% of budget in a day |
| R4 | WebGL context lost on judge machine | L | cues invisible | CSS fallback auto-engage; preflight `webgl2` check at boot | context-lost event |
| R5 | Screen capture permission denied | H (expected) | no keyframe context | system is designed without it: DOM/luma-grid fallback; demo records with capture on, judges can run without | capture denied → luma-grid only |
| R6 | Stale cue suppression rate > 15% | M | semantic cues rarely land | measure at G2; raise cadence, pre-render templates | suppressed rate metric on panel |
| R7 | BigQuery sandbox 60-day expiry surprises judges post-event | L | history gone | nightly export to repo-ignored local archive; document the constraint in README | day-50 calendar note |
| R8 | Demo video reveals a visual bug live | M | credibility hit | record 3 takes; overlay debug panel so any dip is self-evident; never cut live | take 1 review at hour 44 |

## 4.4 Demo storyboard (video, 3:00)

| t | Scene | On-screen proof |
| --- | --- | --- |
| 0:00–0:15 | Problem card: interruption cost numbers (PRD §1.1) | static card, no voice-over padding |
| 0:15–0:35 | Architecture diagram (§3.1) with the two-speed loop highlighted | diagram zoom |
| 0:35–1:20 | Live: learner hits off-by-one error; backspace storm; interim `PERIPHERAL_HUE` at 94 ms; debug panel shows local score 0.71 → escalation | latency waterfall visible |
| 1:20–1:50 | Live: semantic cue underline-glow at line 142; fix typed; `FLOW` pulse; episode closes | reasoning trace + confidence |
| 1:50–2:20 | Two more friction types: fatigue (cadence decay → `AUTOCOMPLETE_PACING`), ASD-persona stall (underspecified step → `ANCHOR_PULSE`) | debug panel state chips |
| 2:20–2:40 | Skill Mastery Dashboard: `D(w)` decay curve, `PII` rise, logistic `k` | chart with axes labeled |
| 2:40–2:55 | Free-tier ledger slide: quotas consumed vs. grants; breaker drill clip (429 → OPEN → CLOSED) | ledger table |
| 2:55–3:00 | Repo URL, DOI, license | end card |

## 4.5 Hour-0 bootstrap (copy-paste)

```bash
git init -b main
git config user.name "benben000000"
git config user.email "bmgarcia0121@gmail.com"
cp .env.example .env.local            # fill Firebase + GEMINI_API_KEY
python -m venv .venv
.venv/Scripts/activate                 # Windows; POSIX: source .venv/bin/activate
pip install -r services/gateway/requirements.txt
pnpm install
```
