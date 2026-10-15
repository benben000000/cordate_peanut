# 07 — Free-Tier Compliance Ledger (Appendix A)

System: Invisible Pedagogy AI · Constraint: **no billing account, no credit card, $0.00 profile**
Verification date for all machine-checked rows: **2026-10-15**. Re-verify checklist in §7.5.

---

## 7.1 Verified quotas and status

### Zero-card profile (default)

| Service | Tier | Quota (verified 2026-10-15) | Source | Used by design? |
| --- | --- | --- | --- | --- |
| Firebase Auth | Spark | 50,000 MAU | firebase.google.com/pricing | yes (1–2 users demo) |
| Cloud Firestore (Standard) | Spark | 1 GiB stored · 20K writes/day · 50K reads/day · 20K deletes/day · 10 GiB/mo egress | firebase.google.com/pricing | yes, low-frequency |
| Realtime Database | Spark | 100 simultaneous connections · 1 GB stored · 10 GB/mo download | firebase.google.com/pricing | yes, 10 Hz state bus |
| Firebase Hosting | Spark | 10 GB stored · 360 MB/day transfer | firebase.google.com/pricing | yes, static export |
| Cloud Functions | Spark | **not available** ("Not applicable"; free grant exists only on Blaze) | firebase.google.com/pricing | **no — replaced by local FastAPI gateway** |
| Cloud Run / Cloud Build / Artifact Registry / Secret Manager | Spark | **not available** (Google Cloud services require billing) | firebase.google.com/pricing | only in the billing-optional profile |
| BigQuery | Sandbox | no credit card; 1 TiB queries/mo; no streaming inserts; all tables/partitions expire after 60 days | cloud.google.com/bigquery/docs/sandbox | yes, batch `bq load` |
| Gemini API | Free tier (AI Studio key) | per-project RPM/TPM/RPD shown in AI Studio; not statically published (docs page defers to AI Studio) | ai.google.dev/gemini-api/docs/rate-limits | yes, rate-budgeted (§7.3) |
| Gemini models | — | `gemini-2.0-flash` **shut down 2026-06-01**; active: `gemini-3.8-flash` (GA 2026-09-02), `gemini-3.8-live` (GA 2026-09-15) | ai.google.dev/gemini-api/docs/deprecations | model registry in TRD §2.2.5 |
| Public WS endpoint | cloudflared quick tunnel | free, account-less | cloudflare docs | yes, `wss://*.trycloudflare.com` |
| Demo video hosting | YouTube unlisted | free | — | yes (link only, not committed) |

### Billing-optional profile (only if a card is added later)

| Service | Free grant (verified 2026-10-15) | Source |
| --- | --- | --- |
| Cloud Run (request-based, us-central1) | 180,000 vCPU-s · 360,000 GiB-s · 2M requests per month | cloud.google.com/run/pricing |
| Outbound internet transfer | 1 GiB/month within North America (Premium tier) | cloud.google.com/run/pricing |
| Cloud Logging | 50 GiB/month ingest (free tier) | cloud.google.com/stackdriver/pricing — re-verify |
| Secret Manager | free tier: 6 active versions, 10k access ops/month — re-verify | cloud.google.com/secret-manager/pricing |

## 7.2 Design budget math (why this fits)

**Firestore writes (hardest 20K/day constraint).** Per active session hour:
state rollup 720 (1 write / 5 s, dirty-checked) + signal docs ≈ 20 + session doc ≤ 2 = **≈ 742 writes/h**.
20,000 ÷ 742 ≈ **26.9 continuous hours/day of headroom** — 10× a realistic demo day.

**Firestore reads.** Client: `onSnapshot` on `limit(10)` signals query + session doc; ≈ 25 reads/h
per client × 2 clients (learner + judge) = 50/h → **≈ 5% of the 50K/day budget at 20 h/day**.

**RTDB.** Gateway state writes at 10 Hz are not op-counted on Spark; the metered dimension is
download. State payload ≈ 200 B × 10 Hz = 2 KB/s = **7.2 MB/h**;
10 GB/month ≈ **1,388 listening-hours/month**. Connections: 2–5 of 100. Storage stays < 1 MB
(single rolling node per session, pruned at close).

**BigQuery sandbox.** Compact NDJSON window record ≈ 350 B at 10 Hz = 3.5 KB/s ≈ **12.6 MB/h**.
A 2-hour demo day = 25 MB loading, querying a single day-partition scans ≤ 30 MB
(≤ 0.003% of the 1 TiB/month). Table expiry is set to 60 days (sandbox maximum), matching PRD §1.6.

**Cloud Run (billing-optional).** A WebSocket holds its request open for the whole session, so
session time ≈ instance time: 1 vCPU × 3,600 s = 3,600 vCPU-s/h. Free grant 180,000 vCPU-s =
**50 instance-hours/month** (CPU is the binding dimension; memory at 512 MiB is 1,800 GiB-s/h vs
360,000 GiB-s = 200 h). Demo usage (2 h/day × 5 days = 10 h) = **20% of grant**.
`min_instances=0` keeps idle cost at exactly $0.

**Pinned-demo anti-profile (documented because the original brief asked for `min_instances=1`).**
1 vCPU / 1 GiB always-on: 2.63M vCPU-s and 1.31M GiB-s per month = **14.6× and 3.65× the free
grants**; it is not free-tier compliant, is used only while recording the 3-minute demo, and is
scaled back to 0 immediately after. The free-safe profile (`min=0, max=4, concurrency=80`) is the
deployment described everywhere else in this suite.

## 7.3 Gemini free-tier rate budget

Not statically quotable (AI Studio shows per-project live limits), so the system treats the quota
as configurable and never assumes it:

| Guard | Value | Effect |
| --- | --- | --- |
| Token bucket (hard) | 10 RPM | 11th escalation in a minute → local-only, no error surface |
| Default cadence | `SEMANTIC_MIN_INTERVAL_MS=10000` | ≥ 6 s between calls per session |
| Demo burst | `SEMANTIC_MIN_INTERVAL_MS=2000`, ≤ 30 s | used when a judge is watching |
| RPD budget | `GEMINI_RPD_BUDGET` (default 200) | when exhausted → breaker OPEN until midnight PT reset |
| 3-minute demo consumption | ≈ 40 calls | measured in Phase 2 rehearsal |
| 1-hour live session | ≈ 350–600 calls | only in burst windows |

The interaction path never depends on the quota: when the bucket empties, the local classifier,
EKF adapter, and cue library continue with full latency guarantees (TRD §3.3 ladder).

Free-tier caveat recorded for honesty: prompts sent to the AI Studio free tier may be retained and
used per Google's terms, while Vertex AI (billing) does not train on customer data. The demo uses
consented/scripted sessions only; the adapter flips to Vertex with `GEMINI_TRANSPORT=vertex`.

## 7.4 Cost exposure statement

Zero-card profile: **$0.00** — no GCP billing account exists; nothing in the architecture calls a
service that requires one (no Cloud Run, no Cloud Functions, no Secret Manager, no GCS, no Vertex,
no streaming BigQuery inserts).

Billing-optional profile: **$0.00** while all of: `min_instances=0`, ≤ 50 WS instance-hours/month,
≤ 2M requests, ≤ 1 GiB egress, sandbox BigQuery. Any breach of those is caught by the alarm thresholds
in §7.5 before charges post.

## 7.5 Re-verification checklist (run before submission and after any promotion)

| Check | Method | Owner | Frequency |
| --- | --- | --- | --- |
| Spark quotas unchanged (Firestore/RTDB/Hosting/Auth) | firebase.google.com/pricing table | A | weekly, and 24 h before submission |
| Cloud Run grant unchanged (if used) | cloud.google.com/run/pricing | A | before deploy |
| BigQuery sandbox constraints (60-day expiry, no streaming) | cloud.google.com/bigquery/docs/sandbox | A | monthly |
| Gemini live model IDs + free-tier status | ai.google.dev/gemini-api/docs/models + deprecations | C | weekly (deprecation watch: the 2.0 shutdown was the near-miss this suite corrects) |
| AI Studio per-project RPM/RPD | AI Studio → active rate limits panel | C | at each rehearsal |
| Usage alarms | `/healthz` counters: Firestore writes/day ≥ 16K, esc/min ≥ 8, spool ≥ 1 GB | A | continuous during demo week |

## 7.6 Verification log (append-only)

| Date | Fact | How verified |
| --- | --- | --- |
| 2026-10-15 | `gemini-2.0-flash` / `gemini-2.0-flash-001` shutdown 2026-06-01; replacement `gemini-3.6-flash`; `gemini-3.8-flash` GA 2026-09-02; `gemini-3.8-live` GA 2026-09-15; 2.5 models limited to prior users | ai.google.dev deprecations + models pages, fetched live |
| 2026-10-15 | Gemini rate limits no longer statically published; AI Studio is authoritative | ai.google.dev/gemini-api/docs/rate-limits |
| 2026-10-15 | Cloud Run free grant 180K vCPU-s / 360K GiB-s / 2M requests (request-based, us-central1); 1 GiB NA egress | cloud.google.com/run/pricing |
| 2026-10-15 | BigQuery sandbox: no credit card, 1 TiB queries/mo, no streaming, 60-day table expiry | cloud.google.com/bigquery/docs/sandbox |
| 2026-10-15 | Firebase Spark quotas per §7.1; Cloud Functions + all Google Cloud services marked "Not applicable" on Spark | firebase.google.com/pricing comparison table, extracted live |
| 2026-10-15 | Remote `github.com/benben000000/cordate_peanut` exists, public, zero branches | `git ls-remote --heads` (exit 0, empty refs) |
| 2026-10-15 | Source paper metadata (author, date, CC-BY-4.0, POMDP+EKF+STC constructs) | doi.org/10.5281/zenodo.21052875 record |
