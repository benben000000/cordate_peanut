# 01 — Product Requirement Document (PRD)

System: Invisible Pedagogy AI · Owner: Benedict Garcia · Status: build-ready · Version: 1.0.0
Source model: DOI [10.5281/zenodo.21052875](https://doi.org/10.5281/zenodo.21052875) (CC-BY-4.0)

---

## 1.1 Executive summary and problem statement

Explicit digital learning (tutorial pop-ups, chat assistants, modal hints, notification-driven nudges)
charges the learner three measurable costs:

| Cost | Observed range (baseline studies on task-switching and interruption) | Mechanism |
| --- | --- | --- |
| Re-orientation time per interruption | 9–23 min to recover deep-focus state | Prefrontal context reload after modal switch |
| Working-memory load | +2–3 concurrent items while parsing a pop-up | Dual-task interference with the primary task |
| Task abandonment | 30–50% in self-directed async tutorials | Frustration accumulates with no bounded-resolution path |

Explicit help also externalizes the answer: the learner copies a solution without generating the
retrieval attempt that consolidation requires (cognitive offloading → skill decay).

Invisible Pedagogy AI inverts the interaction. The system observes the workspace continuously and
cheaply (typing cadence, idle duration, screen delta, cursor jitter), maintains a probabilistic
belief over cognitive state, and injects **ambient micro-cues** — peripheral glows, hue shifts,
code-range highlights, autocomplete pacing, distractor dimming — that guide the learner's own next
retrieval attempt. No prompt is required from the user, and no modal is shown at any time.

Product thesis: if intervention cost is made sub-perceptual (< 120 ms local path, ≤ 0.35 intensity
ceiling) and semantically targeted (Gemini multimodal state + cue plan every 2–10 s), learners
sustain flow longer, abandon tasks less, and fade from cue dependence per the paper's neuroplastic
adaptation model.

## 1.2 Target audience and personas

| Persona | Context | Failure mode today | What the system changes |
| --- | --- | --- | --- |
| Self-directed learner (P1) | Async courses, tutorials, docs | Opens AI chat every error; answer copied, skill not formed | Cue triggers self-retrieval; STC scheduler re-tests at +10 min/+24 h |
| Neurodivergent student — ADHD (P2) | Long-form tasks, 20–40 min attention windows | Low-yield idle loops; tab drift; fatigue unlogged | Fatigue detection (idle + cadence decay) → `DIM_DISTRACTORS` + `AUTOCOMPLETE_PACING` |
| Neurodivergent student — ASD (P3) | Literal instruction following; ambiguity stalls | Hard stall on underspecified steps; no help request | `ANCHOR_PULSE` on the ambiguous token/control, low-arousal violet palette, no social prompts |
| Software engineer, new stack (P4) | IDE, unfamiliar SDK | Compile-error spray; destructive retry cycles | Repeat-error signature detection → `HIGHLIGHT_RANGE` on the fault line |
| High-stakes operator (P5) | Incident commands, data pipelines, live config | High-cost error under time pressure; alert fatigue | Additive-only cues, fail-silent degradation, breaker status surfaced to operators |

Explicitly out of scope: children under 13, clinical diagnosis of any condition, surveillance of user
content (keystroke *content* is never captured; timing only — see §1.6).

## 1.3 Social impact and community value

| Claim | Measurement | Target |
| --- | --- | --- |
| Skill retention improves vs. answer-giving baselines | Delayed post-test (D7) on the same task class, A/B, n ≥ 200 | +15 pp correct-without-help |
| Task frustration/dropout falls | Abandonment rate (session ends within 3 min of ≥ 2 friction windows) | −40% relative |
| Learning is accessible to neurodiverse users | Reduced-motion compliance, zero audio, zero focus theft, 0 modal focus traps | 100% WCAG 2.2 AA on cue surfaces |
| Privacy-preserving by default | Data classes collected; opt-in screen capture; no content keystrokes; no PII in BigQuery | Audit-pass on all three |
| Community reuse | Open docs, schemas, and eval harness (Apache-2.0), paper attribution (CC-BY-4.0) | Public repo, reproducible demo |

The neurodiverse-accessibility claim is structural, not cosmetic: cues are peripheral
(≤ 6% luminance delta), non-modal, non-audio, and removable via `prefers-reduced-motion` and a
one-toggle opt-out that also clears the RTDB state path within 5 s.

## 1.4 Core product features

### F1 — Ambient telemetry ingestion

- Sampling: 60 Hz on `pointermove/input/scroll/focus` via `requestAnimationFrame` accumulator;
  emitted as 100 ms windows at 10 Hz.
- Channels: keystroke inter-arrival stats (p50, p95, CV), backspace ratio, idle ratio (5 s), max idle,
  cursor path RMS deviation from a 200 ms moving fit, cursor velocity p95, jitter burst count,
  scroll reversal rate, screen delta (2 fps luma-grid XOR diff, opt-in), focus-switch rate,
  error events (lint/compile), undo rate, repeat-error signature ratio.
- Constraint: keystroke *content*, clipboard, and form values are never read. Feature vector only
  (16 dims, normalized 0–1) + editor metadata (language, file-hash, line/col, scroll %).
- Overhead budget: ≤ 1.5% CPU on a 4-core laptop; ≤ 1.2 KB/s uplink.

### F2 — Cognitive load classifier

- Fast path: multinomial logistic regression (16→4 softmax), ~12 KB weights JSON, in a Web Worker;
  p95 ≤ 12 ms; outputs state + friction score `1 − P(FLOW)`.
- Slow path: Gemini multimodal (`gemini-3.8-flash`) receives the last 30 telemetry windows
  (~1.2 KB JSON) plus an optional 640×360 JPEG keyframe (q55, ≤ 45 KB) and returns a strict JSON
  object: `{state, confidence, friction_score, cues[≤2], reasoning ≤ 60 tokens}`.
- Personalization: EKF adapter (paper §EKF) fuses Gemini labels into the local weights
  (observation noise R = 0.15², process noise Q = 1e-4·I, update only when confidence ≥ 0.70).
- Accuracy acceptance gate (held-out 500 annotated windows): 4-class macro-F1 ≥ 0.85,
  top-2 accuracy ≥ 0.95, false-friction rate (friction flagged during self-reported flow) ≤ 8%.

### F3 — Implicit scaffolding engine

- Cue channels: `GLOW_RADIAL`, `PERIPHERAL_HUE`, `ANCHOR_PULSE`, `HIGHLIGHT_RANGE`,
  `AUTOCOMPLETE_PACING`, `DIM_DISTRACTORS`.
- Injection rules: max 1 active cue, intensity ≤ 0.35, per-channel cooldown 4000 ms,
  fade-in 300 ms ease-in-out, hold 6–12 s, fade-out 800 ms; never within the caret exclusion
  radius (96 px) or over any text bounding box + 12 px.
- Latency: local interim cue p99 ≤ 120 ms decision-to-pixel; semantic cue p95 ≤ 2.5 s
  end-to-end; semantic cue dropped if its queue age exceeds 250 ms at render.
- Zero-friction failures: any subsystem failure degrades silently to local-only mode; no error
  modal, no toast, no badge. Debug panel (opt-in) exposes breaker state.

### F4 — Neuroplasticity and skill-acquisition analytics

- Per-episode record: friction window count, cue applied, resolution mode (self-resolved within
  2 windows / cue-resolved / abandoned), then STC schedule (+10 min tag, +24 h capture).
- Mastery curve: weekly scaffolding dependency `D(w) = friction windows receiving a cue / total
  friction windows`; logistic fit `D(t) = L / (1 + e^(−k(t − t0)))`; report `k` (adaptation rate,
  target ≥ 0.12/week) and `t0` (adaptation onset).
- Prompt-independence index `PII(w)`: share of friction episodes self-resolved without a cue,
  rising week over week; primary long-horizon KPI.
- Storage: local NDJSON spool → `bq load` → BigQuery sandbox dataset `pedagogy.*`;
  60-day partition expiry respected (paper-grade retention handled by scheduled export if needed).

## 1.5 Success metrics and KPIs

| KPI | Definition | Target | Source of truth |
| --- | --- | --- | --- |
| Intervention latency (fast path) | decision → pixel, local engine, p99 | ≤ 120 ms | `scaffolding_events.latency_decision_to_pixel_ms` |
| State dispatch latency | RTDB write → client listener callback, p95 | ≤ 80 ms | client perf counters |
| Semantic cue latency | escalation → pixel, p95 | ≤ 2.5 s | gateway + client timestamps |
| Friction classification quality | macro-F1 / top-2 accuracy on held-out set | ≥ 0.85 / ≥ 0.95 | eval harness report |
| False-friction rate | friction flagged in self-reported flow windows | ≤ 8% | eval harness |
| Non-intrusive cue rate | injections with no typing halt ≥ 3 s and no undo within 3 s | ≥ 80% | `scaffolding_events.user_reaction` |
| Task abandonment reduction | A/B vs. no-scaffold control | ≥ 40% relative | BigQuery cohort query |
| Flow retention | mean uninterrupted flow interval; % sessions ≥ 60% flow time | +35% / ≥ 60% | RTDB session rollups |
| Cue dependence decay | `D(w)` slope `k`; `PII(w)` trend | k ≥ 0.12/week; PII monotonically increasing over ≥ 4 weeks | `mastery_weekly` |
| Telemetry overhead | CPU on 4-core baseline; frames ≥ 58 fps during cue render | ≤ 1.5%; ≥ 95% frames | client profiler |

Guardrail KPIs (must not regress): cue-induced error rate increase ≤ 2%; opt-out rate ≤ 10%;
suppressed-stale-cue rate ≤ 15% of escalations (above this, semantic cadence is the bottleneck —
raise `SEMANTIC_MIN_INTERVAL_MS` and re-measure).

## 1.6 Compliance and data policy

| Data class | Collected | Stored | Retention |
| --- | --- | --- | --- |
| Keystroke timing (intervals only) | yes | Firestore state doc (rolling, 5 s) + NDJSON spool | 60 days (BigQuery partition expiry) |
| Keystroke content | never | — | — |
| Cursor/scroll/pointer telemetry | yes | as above | 60 days |
| Screen pixels | opt-in, in-transit only | never persisted anywhere | 0 |
| Gemini prompt payloads | in-transit (free tier may retain per AI Studio terms) | not persisted by us | 0 |
| User identity | Firebase Auth UID + `user_hash = SHA-256(uid + project_salt)` | BigQuery `user_hash` only | 60 days |

Redaction before any screen encode: `input[type=password]`, `[data-private]`, and OS-level
password fields are covered by black rectangles; capture region is user-drawn; capture is off by
default and revoked on page reload. For user-facing deployments, the Vertex AI path (billing
profile) replaces the AI Studio free tier to disable prompt retention; the adapter flips with
`GEMINI_TRANSPORT=vertex|aistudio`.
