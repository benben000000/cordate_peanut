# 05 — UI/UX Specification Plan (Aligned with System Flow §3)

System: Invisible Pedagogy AI · Version: 1.0.0 · Token source: §2.1.4 of the TRD

---

## 5.1 Ambient interface philosophy: peripheral awareness

Design rule: **a cue must be skippable by the eye and useful to the mind.** Concretely:

| Rule | Value | Enforcement |
| --- | --- | --- |
| Cue luminance delta vs. local background | ≤ 6% | shader clamp; pixel check in CI screenshot test |
| Max intensity (opacity/alpha of any cue) | 0.35 | validation gate in `cueFsm.ts` (server signals clamped, violation logged) |
| Default intensity band | 0.10–0.24 | cue catalog §5.6 |
| Cue placement band | outside the 20° central cone; peripheral bands: right rail, bottom rail, top-right | layout mask recomputed per frame from caret + focused element |
| Caret exclusion radius | 96 px hard, 160 px preferred | mask applied before render |
| Text bounding boxes | cue never overlaps text bbox + 12 px padding; underlines render below baseline only | DOM rect mask |
| Motion | pulse ≤ 0.42 Hz, no strobe, no parallax; `prefers-reduced-motion` → opacity-only, no pulse, fade 120 ms | media query + FSM variant |
| Sound / vibration / notification API | never | code review checklist + CSP disallows Notification API (`Permissions-Policy`) |
| Focus behavior | never steals focus, never inserts text, never opens overlays | Playwright assertion in CI |
| Cue semantics | re-weight attention to a region; never reveal the answer | prompt contract §2.2.5 + target validator |

Why peripheral: the foveal channel is occupied by the learner's task; a cue that demands foveation
is a task switch (9–23 min re-orientation cost, PRD §1.1). Peripheral motion/color changes are
detected by magnocellular pathways and can guide saccades without verbal processing — the mechanism
the cue catalog is built on.

## 5.2 Screen hierarchy and layout system

Three surfaces, one shell. All surfaces render from the same Zustand store; no surface blocks
another.

### 5.2.1 Workspace Studio (primary surface)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ status bar    ambient status dot (debug only)      ambient intensity meter    │
├──┬────────────────────────────────────────────────────────────────────────────┤
│  │  editor (JetBrains Mono 14/22, max-width 980 px, centered)                 │
│ f│  ┌──────────────────────────────────────────────────────────────────────┐  │
│ i│  │ 141  function walk(arr) {                                            │  │
│ l│  │ 142    for (let i = 0; i <= arr.length; i++) {   ◄ glow band (38°)   │  │
│ e│  │ 143      visit(arr[i]);                                              │  │
│ s│  └──────────────────────────────────────────────────────────────────────┘  │
│  │                                                                            │
│  │  problem panel (collapsible, right 380 px — DOM only, no cue rendering)    │
├──┴────────────────────────────────────────────────────────────────────────────┤
│  run console (virtualized, error lines link to editor positions)              │
└───────────────────────────────────────────────────────────────────────────────┘
  ambient canvas layer: full viewport, z-index 9000, pointer-events none, ARIA-hidden
```

Scale: works at 1280×720 minimum; cue masks assume ≥ 480 px between caret and viewport edge,
else cue relocates to the nearest compliant band (right rail → bottom rail → suppressed).

### 5.2.2 Cognitive State Debugger (judge / presenter view, `?debug=1`)

```
┌──────────────────────────── right rail (380 px, fixed) ──────────────────────┐
│ STATE  ● CONFUSION  conf 0.83  score 0.79   [FLOW][CONF][FATI][FRUS]         │
│ SPARK  ▁▂▃▅▆▅▇▆▅▇▆▅▃▂▁ (120 samples @10 Hz)                                  │
│ REASON "error repeat 0.50 + undo storm; anchor to bounds expression"          │
│ CUE    HIGHLIGHT_RANGE  i=0.24  ttl=9000 ms  src=GEMINI                      │
│ WATERFALL capture 4ms ▸ worker 9ms ▸ uplink 38ms ▸ gemini 672ms ▸ rt 63ms ▸  │
│           render 14ms  (fast path 94 ms | semantic 810 ms)                    │
│ BREAKER CLOSED   escalations 12   suppressed 1   rpm 4/10                    │
│ LADDER  full ▸ [local+cache] ▸ [rtdb down] ▸ [local-only]                    │
└───────────────────────────────────────────────────────────────────────────────┘
```

Presenter requirements: entire rail visible at 1920×1080 without scrolling; monospace numerals
(`font-variant-numeric: tabular-nums`); colors match cue palette; no animations except the score
sparkline (updates ≤ 10 Hz, GPU-cheap).

### 5.2.3 Skill Mastery Dashboard (`/dashboard`)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ user_hash 7b1f…   range [30d ▾]   sessions 14   episodes 212                   │
├──────────────────────────────────┬────────────────────────────────────────────┤
│ Scaffolding dependency D(w)      │ Prompt-independence index PII(w)           │
│  1.0 ┤●●●●                       │  1.0 ┤                              ●●●●   │
│  0.5 ┤     ●●●                    │  0.5 ┤              ●●●                   │
│  0.0 ┤        ●●●●●  ├ 0.62 target│  0.0 ┤ ●●●                                │
│      logistic fit k=0.17/wk t0=w3│      monotone rising required             │
├──────────────────────────────────┴────────────────────────────────────────────┤
│ Episode table: ts · state · channel · latency ms · resolution (self|cue|aband)│
└───────────────────────────────────────────────────────────────────────────────┘
```

Data: `pedagogy.mastery_weekly` + episode rows; when BigQuery is unreachable, the dashboard renders
a clearly labeled `DEMO_DATA` watermark (seeded JSON) — never silently fakes numbers.

## 5.3 Component breakdown and state management

| Component | Kind | Inputs (props/store) | Internal state | Failure mode |
| --- | --- | --- | --- | --- |
| `AppShell` | layout | route, session | — | static render |
| `TelemetryProvider` | headless | opt-ins (capture, reduced motion) | ring buffers, fps counters | emits at 1 Hz with `degraded:true` |
| `InferenceWorkerClient` | headless | postMessage windows | weights version | falls back to threshold rules |
| `AmbientCanvasLayer` | WebGL2 canvas | render commands | GL context, uniform interpolators | CSS fallback layer |
| `CueOrchestrator` | headless FSM | local score, server signals, cooldowns | FSM state, queue(1) | drops to IDLE, logs reason |
| `PeripheralGlow` | render primitive | channel, geometry, intensity, ttl | phase (in/hold/out) | clamped to 0.35 |
| `UnderlineAnchor` | editor decoration | code range | range validity | re-resolves once, else drop |
| `CaretHalo` | render primitive | caret rect | pulse phase | disabled under reduced motion |
| `DistractionDimmer` | render primitive | peripheral rect list | dim level ≤ 6% | no-op |
| `AutocompletePacer` | editor behavior | cadence state | debounce timers | disabled default-off |
| `StateChip` | debug UI | state, confidence | — | renders `UNKNOWN` |
| `ScoreSparkline` | debug UI | 120 samples ring | — | flat line |
| `ReasoningTrace` | debug UI | last reasoning string | — | empty state text |
| `LatencyWaterfall` | debug UI | stage timings | — | shows missing stages |
| `MasteryCurve` | dashboard | weekly rows, logistic fit | — | DEMO_DATA watermark |
| `ReducedMotionProvider` | context | media query | — | motion disabled |

State management: single Zustand store (`useSession`) + FSM reducer (`cueFsm.ts`). Store slices:
`telemetry` (ring buffers, never re-rendered), `belief` (score/state, throttled 2 Hz to React),
`cues` (active + queue), `debug` (timings), `breaker`. React renders only from `belief/cues/debug`
throttled views; the canvas layer reads the store directly in its rAF loop (no React re-render for
cue animation).

Transition timings (single source of truth, consumed by tests):

| Transition | Time | Curve | Notes |
| --- | --- | --- | --- |
| Fade-in any cue | 300 ms | ease-in-out | token `--ip-fade-in-ms` |
| Hold | ttl 6000–12000 ms | — | server-clamped |
| Fade-out | 800 ms | ease-in-out | starts on ttl end or user input at target |
| Per-channel cooldown | 4000 ms | — | channel cannot re-fire during cooldown |
| Global concurrency | 1 cue | — | queue depth 1, replace-oldest |
| Pulse period | 2400 ms (0.42 Hz) | sine | destination-out breathing, amplitude ≤ 0.06 |
| Reduced-motion fade | 120 ms | linear | opacity-only, no pulse |
| Debug sparkline refresh | 100 ms | — | matches window cadence |

## 5.4 Zero-friction error states

| Failure | UI response | Debug-panel visibility | Never happens |
| --- | --- | --- | --- |
| WS disconnected | nothing changes visually; local mode continues | ladder shows `local-only` + reason | no toast/banner/sound |
| Gemini breaker OPEN | cue stream continues from local cache; cadence may slow | breaker chip `OPEN(30 s)` | no apology dialogs |
| Target unresolvable (code moved) | cue dropped silently | `TARGET_UNRESOLVED` counter | no re-targeting loop |
| WebGL context lost | CSS fallback variant of current cue; future cues use CSS | renderer chip shows `css-fallback` | no reload prompt |
| BigQuery unreachable | dashboard shows cached snapshot + data age | `stale 14 m` label | fake live numbers |
| Auth token expired mid-session | silent re-auth (refresh token), WS re-auth frame | auth chip flashes 1 s | logout/redirect, data loss |
| Spool disk full | analytics pause; interaction path unaffected | `spool_overflow` counter | blocking writes |

## 5.5 Performance budgets (UI)

| Budget | Value | Measurement |
| --- | --- | --- |
| Main-thread work per window | ≤ 3 ms p95 | `performance.mark` ranges, 600-sample ring |
| Worker inference | ≤ 12 ms p95 | worker timer |
| Cue decision → pixel (local) | ≤ 120 ms p99 | timestamp diff, logged per injection |
| Canvas GPU frame | ≤ 2.5 ms, 60 fps ≥ 95% frames | `EXT_disjoint_timer_query` where available, else rAF delta |
| React commits during cue animate | 0 | render counter assertion in tests |
| Bundle (gzip, initial route) | ≤ 2.5 MB | `next build` report |
| Lighthouse a11y score | ≥ 95 | CI run |
| Total telemetry CPU | ≤ 1.5% on 4-core baseline | Task Manager / `perf` two-minute sample |

## 5.6 Cue catalog v1

| Channel | What the user sees | When selected | Intensity | TTL | Suppression rules |
| --- | --- | --- | --- | --- | --- |
| `GLOW_RADIAL` | soft radial gradient bloom near the fault region's periphery | FRUSTRATION, fast-path first response | 0.10–0.20 | 4–6 s | never during active text selection |
| `PERIPHERAL_HUE` | 4% hue shift in one peripheral band (blue 217° confusion, violet 262° fatigue) | CONFUSION/FATIGUE without target resolution | 0.08–0.14 | 6–10 s | disabled if color-blind simulation test fails for the palette pair |
| `ANCHOR_PULSE` | small breathing anchor next to an exact token/control | ASD-persona stall, STC recall cues | 0.10–0.15 | 6 s | max 1 per 30 s per region |
| `HIGHLIGHT_RANGE` | below-baseline gradient underline on a code range | error repeats, semantic escalation | 0.16–0.24 | 8–12 s | never over live diagnostics squiggles; offset +4 px |
| `AUTOCOMPLETE_PACING` | suggestion appears 400 ms later or earlier (no content change) | FATIGUE recovery and recall tags | n/a (timing) | 1 edit | off by default in production profile |
| `DIM_DISTRACTORS` | peripheral non-task UI dims 6% for 20 s | sustained FATIGUE + focus-switch rate | 0.06 | 20 s | never dims controls the user recently clicked (< 60 s) |
| `FLOW` pulse | single teal breath at 168°, 0.06 | episode self-resolved | 0.06 | 2.4 s | ≤ 1 per 5 min |

Palette accessibility: hue pairs selected for deuteranopia/protanopia separation (168° teal vs
217° blue vs 262° violet vs 38° amber); saturation capped on violet to avoid contrast reduction on
dark themes; all cues tested against WCAG 2.2 AA contrast floors on both light and dark editor
themes (cue must not reduce text contrast below 4.5:1).

WebGL fallback mapping: `GLOW_RADIAL → radial-gradient CSS`, `PERIPHERAL_HUE → box-shadow inset`,
`ANCHOR_PULSE → opacity keyframe`, `HIGHLIGHT_RANGE → border-image gradient`, `DIM_DISTRACTORS → filter
brightness(0.94)`. Fallback ladder is automatic and logged.
