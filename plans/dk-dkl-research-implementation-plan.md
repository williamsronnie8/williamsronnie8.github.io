# DK/DKL Research System — Implementation Plan

**Status:** proposed · **Date:** 2026-07-22
**Code home:** `athena-src` (this document lives apart from the code; all paths below are relative to `athena-src`)
**Runtime home:** `~/.athena/investment-research/` (SQLite + reports)

## 1. Purpose

The deployed system is a bounded DK/DKL filing-and-news monitor: deterministic SEC + Google News collection at 1:00/6:30, local-model synthesis (draft + skeptical review on digest-locked `qwen3.6:27b-q8_0`), and an 8:15 iMessage brief with exactly-once delivery. The target system is a longitudinal investment-research assistant with six capabilities the monitor does not yet have:

1. **Living dossiers** — per-company structured state (thesis, metrics, maturities, promise scorecard, related-party ledger, evidence tests, change log)
2. **Self-extending research queue** — proposed questions become validated, stored, executed tasks
3. **Domain-specific data** — crack spreads, EIA/EPA series, transcripts, peer data, deterministic metric math
4. **Opportunity discovery** — outward scanning beyond the hardcoded DK/DKL pair
5. **Multi-horizon reporting** — immediate material alerts, weekly memo, quarterly review, annual reasoning audit
6. **Decision journal** — recorded reactions, thesis changes, purchases, passes, exits

The audit also found four shortcomings in the deployed pipeline that must be fixed **before** building on top of it:

- **S1** — a model failure still marks fetched evidence as processed; the filing never gets another analysis attempt (`scripts/dk_dkl_research.py:965`; already consumed 7 historical filings)
- **S2** — evidence is clipped into a shared 60,000-char packet (~7.5k chars/filing at the default 8-item batch), so the model never sees whole filings (`scripts/dk_dkl_research.py:686`)
- **S3** — citation checks are structural (cited ID exists; high-importance findings carry a primary source) but nothing deterministically verifies the source supports the claim (`scripts/dk_dkl_research.py:721`)
- **S4** — "memory" is the last three brief texts capped at 6,000 chars; no structured per-company state (`scripts/dk_dkl_research.py:404`)

## 2. Anchor map (current system)

| Concern | Anchor |
|---|---|
| Discovery, allowlist, dedup, backlog, report writing | `scripts/dk_dkl_research.py:207` |
| Memory (last 3 briefs, 6k cap) | `scripts/dk_dkl_research.py:404` |
| Evidence packet build (60k shared clip) | `scripts/dk_dkl_research.py:686` |
| Structural citation checks | `scripts/dk_dkl_research.py:721` |
| Draft + skeptic synthesis | `scripts/dk_dkl_research.py:827` |
| Evidence marked processed regardless of synthesis outcome | `scripts/dk_dkl_research.py:965` |
| Exactly-once delivery, shared outbox | `scripts/dk_dkl_research.py:1000` |
| Collector schedule (1:00, 6:30) | `deploy/com.athena.dk-dkl-research-collect.plist:15` |
| Brief schedule (8:15) | `deploy/com.athena.dk-dkl-research-brief.plist:15` |
| Quality gate | investment gate, 9 checks, wired into `make test` |

## 3. Design principles (carried forward, non-negotiable)

1. **Determinism first.** Anything computable in code (metric math, calendars, dedup, triage rules, verification) is computed in code. The model interprets; it does not arithmetic.
2. **Schema-locked model I/O.** Every model call has a JSON schema; invalid output is rejected and bounded-retried, never partially trusted.
3. **Loud degradation.** Every silent-failure path found gets a visible surface (brief footer, alert). A system like this dies from quiet rot, not crashes.
4. **Additive storage.** No destructive updates to dossier state — supersede, version, and log. SQLite migrations are numbered and additive-only.
5. **Bounded autonomy.** The model proposes (research tasks, dossier edits, candidates); code validates against schemas, citations, and budgets; irreversible expansions (new hosts, new full-coverage companies) require user approval.
6. **Local-only, allowlisted.** Digest-locked local model; every fetch goes through the existing host allowlist and backlog machinery. No new hosts without an explicit allowlist entry reviewed by the user.

## 4. Phase map

| Phase | Delivers | Fixes/Builds | Depends on | Size |
|---|---|---|---|---|
| 0 | Pipeline integrity | S1, S2, S3 | — | M |
| 1 | Living dossiers | Capability 1, S4 | 0 | L |
| 2 | Immediate alerts + journal capture | Cap. 5 (partial), 6 (partial) | 0 | M |
| 3 | Research queue controller | Capability 2 | 1 | M |
| 4 | Domain data engine | Capability 3 | 0 (feeds 1) | L |
| 5 | Weekly memo + quarterly review | Capability 5 (rest) | 1, 4 | M |
| 6 | Opportunity discovery | Capability 4 | 1, 4 | M |
| 7 | Annual audit + journal completion | Cap. 5/6 completion | 2, 5 | S |

Each phase ships behind a config flag, extends the investment gate, and must survive one full live cycle (collect → analyze → deliver) before the next phase starts. Phases 2 and 4 can proceed in parallel with 1 if desired; everything else is ordered.

---

## Phase 0 — Pipeline integrity

Goal: the existing daily loop stops losing work and stops overstating what it verified. Nothing new is built on the pipeline until this lands.

### 0.1 Decouple fetch success from analysis success (S1)

- Add to the evidence/sources table: `analysis_state` (`pending | analyzed | failed | exhausted`), `analysis_attempts`, `last_analysis_error`, `analyzed_at`, `analyzed_in_report_id`.
- Migration rule: rows cited by an existing stored report → `analyzed`; everything else fetched-but-unreported → `pending`. This automatically re-queues the 7 consumed historical filings.
- Analysis batch selection becomes `analysis_state IN ('pending','failed') AND analysis_attempts < 3`, ordered primary-sources-first, newest filing first.
- Mark `analyzed` **only** inside the same SQLite transaction that persists a validated report. On synthesis failure: increment attempts, store the error, leave the row eligible. After 3 failures → `exhausted`, which is *not* silent: exhausted count appears in the brief's pipeline-health footer (see 0.4).
- One-shot command `--requeue-unanalyzed` for manual recovery; run it once at rollout to sweep the historical 7.

### 0.2 Whole-filing coverage via per-filing map-reduce (S2)

Replace the shared 60k packet at `:686` with a per-filing budget and a three-stage flow:

- **Split (code):** strip inline XBRL/HTML to text (we already download up to 6 MB), split by SEC structure — 10-K/10-Q by Item heading, 8-K by Item number, exhibits indexed by type/description. Financial-statement sections are always retained; boilerplate (certifications, XBRL tag dumps) always dropped.
- **Map (model):** per-chunk extraction pass with a small schema — facts, verbatim quotes, section reference — at ~24k chars/chunk, hard cap ~12 chunks/filing. Anything skipped is recorded in a `coverage` field and **printed in the report's Coverage section**. Truncation becomes visible instead of silent.
- **Reduce (model):** the existing draft + skeptic synthesis (`:827`) consumes extracted facts instead of raw clipped text.

Runtime consequence: a 27B Q8 doing map passes cannot live inside the 8:15 brief window. **Recommendation: move analysis into the 1:00/6:30 collect runs; the 8:15 job becomes assemble-and-deliver only** (reads stored per-filing analyses, composes the brief). This also makes brief delivery immune to a single filing's model failure. Log per-stage wall time to a `runs` table so batch caps can be tuned against measured throughput.

### 0.3 Deterministic citation verification (S3)

- Findings schema gains `quotes: [{source_id, verbatim}]`; numeric claims the model marks as sourced gain `numbers: [{surface_form, source_id}]`.
- Code verifies, after whitespace/HTML-entity normalization: each verbatim quote appears in the stored text of its cited source; each numeric surface form (e.g. `$118.7 million`, `(3.2)%`) appears in its cited source. No unit conversion or arithmetic in v1 — verify surface forms only, conservatively.
- Findings failing verification are dropped and fed back to the skeptic pass with reasons for one bounded redraft; still failing → excluded and counted in the health footer.
- The brief labels each finding **verified** (quote-checked) or **model-attested** (structural checks only), so the reader always knows the verification level. Existing structural checks at `:721` remain as gate one.

### 0.4 Pipeline-health footer

Every brief ends with one deterministic line: backlog depth, pending/failed/exhausted analysis counts, last collect run status, model failure count in window, delivery outbox state. This is the loud-degradation surface the rest of the plan relies on.

**Gate additions (0):** no evidence row reaches `exhausted` without 3 recorded attempts; report-persist and `analyzed`-mark are atomic (crash-injection test); a fixture filing with a known quote passes verification and a perturbed quote fails; a filing larger than the packet budget yields a non-empty Coverage section; brief renders with health footer even when synthesis produced zero findings.

**Acceptance:** the 7 historical filings are re-analyzed or visibly exhausted; a forced model failure leaves the filing eligible and the brief says so; a full-size 10-K produces per-section extraction with coverage accounting.

---

## Phase 1 — Living dossiers (capability 1, retires S4)

Goal: replace "last three briefs" with structured, versioned, per-company state that every later phase reads and writes.

### Data model (new tables, additive)

- `entities` — ticker, CIK, tier (`subject | peer | candidate | retired`), news keyword (replaces the hardcoded "Delek" title filter), cadence.
- `thesis_versions` — entity, version, stance (`long | short | neutral | watch`), summary, key drivers, risks, valuation notes, `created_from_report_id`, `superseded_by`. Never edited; only superseded.
- `metrics` — entity, metric, period, value, unit, `source_id`, `computed_by` (`code | model`), formula version. Model-written values must pass Phase 0.3 verification; code-written values carry the formula version that produced them.
- `debt_maturities` — instrument, principal, rate, maturity date, source, status. Rendered as a ladder.
- `promises` — who promised, verbatim promise, when, source, due hint, status (`open | kept | broken | moved`), resolution source. The management promise scorecard.
- `related_party` — entity pair, relationship, terms, source, first seen, last confirmed. For DK/DKL specifically this carries the intercompany agreements (dropdowns, throughput commitments) that make the pair worth tracking as a pair.
- `evidence_tests` — falsifiable statements: hypothesis, the observable that would confirm/refute, status, resolving source, linked research task. This is what keeps the thesis honest.
- `dossier_log` — every mutation: table, summary, from/to, originating report, timestamp. The dossier change log the audit asked for.

### Mutation mechanism

Synthesis output gains a `dossier_ops` list — typed, schema-locked operations (`add_promise`, `resolve_promise`, `add_maturity`, `update_metric`, `propose_thesis_change`, `add_related_party`, `add_evidence_test`, `resolve_evidence_test`, …). Code validates every op: referential integrity, citation verification (0.3), no deletes. Valid ops apply in one transaction and append to `dossier_log`; invalid ops are dropped and counted. A `propose_thesis_change` never auto-applies the first time it appears — it is flagged in the brief and applies on the next cycle unless the user vetoes via journal reply (Phase 2), keeping a human on the highest-consequence mutation with zero required interaction.

### Rendering and context

- Deterministic Markdown snapshot per entity (`dossier-DK.md`, `dossier-DKL.md`) regenerated after every apply, stored alongside reports.
- A budgeted **context pack** (~12k chars: current thesis, open promises, next 8 quarters of maturities, open evidence tests, latest key metrics, last 5 log entries, plus the single most recent brief) replaces the three-brief memory at `:404` as model context.

### Bootstrap

One-shot backfill: run the Phase 0 analyzer over the already-stored reports and re-queued filings emitting `dossier_ops` only, to seed maturities/promises/related-party entries; initial thesis drafted by the model, marked `stance: watch`, and surfaced in the next brief for the user's veto/edit.

**Gate additions (1):** every dossier mutation row has an originating report and a verifying source; thesis versions form an unbroken supersession chain; context pack respects its byte budget on a maximal fixture; snapshot render is deterministic (byte-identical on re-render).

**Acceptance:** both dossiers populated from history; a live cycle where a new filing produces at least one applied, logged, cited dossier op; brief shows "dossier changes" section instead of relying on prose memory.

---

## Phase 2 — Immediate material alerts + minimal decision journal

Goal: cut latency on material events from "next 8:15" to minutes, and start accruing the decision history that Phase 7 audits. Both are small once Phase 0 exists.

### Alerts

- **Deterministic triage first.** At collect time, classify filings by rule: 8-K item codes (1.01, 1.03, 2.03, 2.04, 3.01, 4.01, 4.02, 5.02, 7.01+size), Form 4 clusters (≥2 insiders / 5 days) or large notional, SC 13D/G, credit-agreement exhibits, going-concern language hits. Tier map lives in config.
- **Model second, optional.** A short schema-locked pass adds headline + why-it-matters. If the model fails, the alert still goes out with deterministic fallback text (form type, items, link) — alerting never blocks on the model.
- **Delivery** through the existing exactly-once outbox (`:1000`) with a distinct alert tag. Quiet hours: immediate 7:00–22:00; overnight alerts roll into the 8:15 brief marked "overnight." Dedup: one alert per accession number, ever; the brief references alerts already sent rather than repeating them.
- **Schedule:** add midday collector runs (12:30, 17:30) so market-hours filings alert same-day. Same plist, two more calendar intervals.

### Journal (minimal capture now, workflows later)

- `journal_entries` — timestamp, kind (`reaction | thesis | order | pass | exit | note`), text, entity, linked report/alert, source (`cli | imessage`).
- Capture v1: a tiny CLI (`athena-journal "took starter position DKL" --kind order`) plus a watched drop-file; iMessage *reply* ingestion only if the messaging infra already supports inbound — otherwise defer, don't build inbound plumbing for this phase.
- The brief includes yesterday's journal entries and uses them for the thesis-change veto window (Phase 1) and queue vetoes (Phase 3).

**Gate additions (2):** same accession can never produce two alerts (idempotency test); alert path emits fallback text under forced model failure; journal entries round-trip and appear in the next brief.

**Acceptance:** a synthetic 8-K item 2.03 fixture produces an alert in the collect run that found it; a journal CLI entry shows up in the next morning's brief.

---

## Phase 3 — Self-extending research queue (capability 2)

Goal: close the loop the audit called "proposal only" — `next_research` questions become validated, stored, executed, and answered into dossiers.

- `research_tasks` — question, origin report/finding, entity, status (`proposed | accepted | active | answered | stale | rejected`), priority, source plan (which registered source kinds could answer it), attempts, answering report.
- **Validation of model proposals:** dedup against open tasks by normalized token-set similarity (deterministic; no embeddings in v1); must name at least one registered source kind to be answerable; open-queue cap (12) with lowest-priority eviction to `stale`; up to 2 auto-accepted per day, the rest listed in the brief as `proposed` for veto via journal.
- **Executor:** a budgeted slot in the 1:00 collect run — max 2 tasks/night, max fetches per task — that plans fetches from registered sources only (SEC EDGAR full-text search, already-stored filing sections, targeted Google News queries; EIA series once Phase 4 lands), synthesizes under the same citation gates, writes a `research-answer` report, applies `dossier_ops`, closes the task. Tasks that fail 3 executions → `stale`, surfaced in the brief.
- **Hard guardrail:** the queue can recombine registered source kinds but can never add hosts or source kinds. Allowlist changes remain human-only.

**Gate additions (3):** proposals duplicating an open task are rejected in test; queue never exceeds its cap; an executed task's answer report carries verified citations; a task with no registered source kind is rejected at validation.

**Acceptance:** one full autonomous loop observed in production: question proposed in a brief → auto-accepted → executed overnight → answer applied to a dossier → surfaced in the next brief.

---

## Phase 4 — Deterministic domain data engine (capability 3)

Goal: the refining-specific numbers the model currently cannot see, computed by code, stored as metrics, feeding dossiers and reports. Independent of Phases 1–3; feeds all of them.

- **EIA v2 API** (free key, official, allowlist addition): WTI Cushing, WTI Midland, Brent, Gulf Coast RBOB and ULSD spot; weekly PADD 3 refinery utilization; crude/gasoline/distillate stocks. Fetched by the collector at its existing cadence.
- **Crack spreads in code:** Gulf Coast 3-2-1 and 5-3-2 from the series above, formulas config-driven and versioned, with unit tests against hand-computed fixtures. The WTI Midland–Cushing spread is tracked explicitly (Big Spring economics). Stored to `metrics` with `computed_by=code`.
- **RINs:** EPA weekly D4/D6 RIN price aggregates (public data; RFS exposure is a real DK earnings driver). Marked optional — if the source shape proves unstable, it degrades to absent, never to wrong.
- **XBRL companyfacts:** pull `companyfacts` JSON from EDGAR for every entity — deterministic structured fundamentals (revenue, debt, distributions) without parsing HTML. This becomes the backbone of quarterly metric tables in Phase 5.
- **Transcripts:** no clean free source. v1 policy: rely on 8-K EX-99 earnings releases and prepared-remarks exhibits (already in scope via EDGAR). Third-party transcript sites are a user allowlist decision, not a default.
- **Peers:** watchlist config (proposed defaults — DK peers: VLO, MPC, PSX, PBF, DINO, PARR; DKL peers: PAA, WES, HESM, GEL; user-editable). Peer collection is EDGAR-only, weekly cadence, `tier=peer` (metrics + filings, no thesis). Feeds the relative-value table in Phase 5 and candidate scoring in Phase 6.
- All of this rides the existing allowlist/dedup/backlog machinery at `:207` — extended, not forked. The brief gains a one-line deterministic "market tape" (cracks, Mid-Cush spread, utilization, WoW deltas).

**Gate additions (4):** crack-spread math matches hand-computed fixtures exactly; a missing EIA series yields an absent metric and a health-footer note, never a stale value presented as current; companyfacts ingestion is idempotent per (entity, tag, period).

**Acceptance:** a week of live cracks/RIN/utilization series stored and rendered in the brief tape; DK and DKL companyfacts fundamentals present in dossier metrics.

---

## Phase 5 — Weekly memo + quarterly review (capability 5, remainder)

- **Weekly memo** (new LaunchAgent, Sunday 18:00): inputs are the week's reports, `dossier_log` window diff, code-computed metric moves (WoW/QoQ), queue state, journal entries. Sections: what changed; thesis health (evidence-test status); promise movement; market tape trend; relative value vs peers; queue review; next-week calendar. The calendar is deterministic — seeded from historical filing cadence and parsed earnings-date announcements.
- **Quarterly review** — event-triggered, not scheduled: detection of a subject's 10-Q/10-K plus earnings 8-K opens a quarterly cycle with a larger analysis budget. Code builds the QoQ/YoY metric table from companyfacts; the model must explicitly resolve or carry **every** open promise touching the quarter, with citations; output is a quarterly memo plus a proposed thesis version bump through the normal Phase 1 veto flow.
- Both deliver through the existing outbox; the daily brief links rather than repeats them.

**Gate additions (5):** weekly memo renders from fixtures with an empty week (degrades gracefully); quarterly cycle cannot close while any open promise for that quarter is unaddressed; calendar predictions carry their derivation.

**Acceptance:** one live weekly memo; one full quarterly cycle at the next DK/DKL earnings (expected early August 2026 — Phase 5 should be live before then if possible, else the first cycle runs on the following quarter).

---

## Phase 6 — Opportunity discovery (capability 4)

Goal: turn the hardcoded two-ticker system outward, with promotion gates so the universe grows deliberately.

- **Candidate sources:** the related-party ledger (counterparties surface here automatically once Phase 1 runs); 13D/G filers on the subjects; EDGAR full-text search for "Delek" mentions in *other* companies' filings; peer-screen outliers from Phase 4 metrics.
- **Pricing for screens:** requires a daily-close price source — one new allowlist host (recommendation: Stooq CSV; decision listed in §10). Valuation screens stay out of scope until this is approved.
- `candidates` table with score and rationale; the weekly memo proposes top-N with promote/dismiss. Promotion path: user journal reply promotes to `candidate` tier (metrics-only dossier, reduced cadence); promotion to full `subject` tier is **always** user-approved. The model never expands full coverage on its own.
- Config guardrails: max active entities per tier, per-tier fetch budgets, allowlist unchanged.

**Gate additions (6):** a candidate cannot reach `subject` tier without a journal approval entry; entity caps enforced; discovery fetches stay within budget in test.

**Acceptance:** at least one organically discovered candidate (e.g., a counterparty from the related-party ledger) tracked at candidate tier with metrics flowing.

---

## Phase 7 — Annual reasoning audit + journal completion (capabilities 5/6, completion)

- **Journal completion:** order/pass/exit entries gain optional size/price fields; every decision entry freezes a hash of the dossier context pack at decision time — the audit can later reconstruct exactly what was believed when the decision was made.
- **Annual audit** (January, scheduled): code assembles per-thesis timelines (thesis versions, decisions, subsequent 12 months of metrics/events); the model writes the audit with mandatory sections — what we believed, what happened, where reasoning failed (bad evidence / bad weighting / bad timing), proposed process changes. Each accepted process change becomes a research task or a config diff proposal, so the audit feeds back into the system rather than ending as prose.
- **Calibration table, computed by code:** evidence-test confirm/refute rates, management promise-keeping base rate, alert precision (alerts later judged material vs not). The deterministic scorecard of both the company and the system.

**Gate additions (7):** every decision entry has a context-pack hash that resolves to a stored snapshot; calibration table math from fixtures.

**Acceptance:** a dry-run annual audit over the partial year's data produces a coherent document with a populated calibration table.

---

## 8. Cross-cutting engineering

- **Migrations:** introduce a `schema_version` table and numbered, additive-only SQLite migrations in Phase 0. Every phase's schema lands as a migration; no drops, no rewrites.
- **Structure:** `dk_dkl_research.py` is already ~1,000+ lines and this plan multiplies its responsibilities. Strangler-pattern extraction, one module per phase as it's touched: `research/store.py` (Phase 0), `research/dossier.py` (1), `research/alerts.py`, `research/journal.py` (2), `research/queue.py` (3), `research/markets.py` (4), `research/reports.py` (5). No big-bang refactor; the script becomes the entrypoint that wires modules.
- **Observability:** a `runs` table (job, stage, wall time, counts, outcome) written by every job from Phase 0 on; it feeds the health footer and answers tuning questions (batch sizes, chunk caps) with data instead of guesses.
- **Model budgets:** per-run hard caps (chunks, tasks, redrafts) enforced in code so the overnight window holds. If map-stage throughput on the 27B proves too slow, evaluate a smaller digest-locked model for extraction only (27B stays for reduce + skeptic) — decide from the `runs` data, not upfront.
- **Testing:** every phase extends the investment gate (the named checks above); parsers and formulas get recorded-fixture tests (real filing excerpts committed as fixtures); renderers get golden-file tests. `make test` remains the single entry point.
- **Config:** all new knobs (entities, tiers, cadences, alert tier map, budgets, quiet hours, EIA key) join the existing validated config; each phase's feature flag defaults off until its acceptance criteria pass in production.

## 9. Schedule changes (before → after)

| Job | Now | After |
|---|---|---|
| Collect + analyze | 1:00, 6:30 (collect only) | 1:00, 6:30, **12:30, 17:30** (collect + analyze + alerts; Phase 0/2) |
| Daily brief | 8:15 (synthesis + deliver) | 8:15 (assemble + deliver only; Phase 0.2) |
| Weekly memo | — | Sunday 18:00 (Phase 5) |
| Quarterly review | — | event-triggered by earnings filings (Phase 5) |
| Annual audit | — | January, scheduled (Phase 7) |
| Research executor | — | inside the 1:00 run, budgeted (Phase 3) |

## 10. Decisions needed from the user (recommendations attached)

1. **Move synthesis from the 8:15 brief into collect runs** (Phase 0.2). Recommended: yes — it unblocks whole-filing coverage and makes the brief delivery-only.
2. **Midday collector runs + alert quiet hours** (Phase 2). Recommended: 12:30/17:30 runs; immediate alerts 7:00–22:00, overnight rolls into the brief.
3. **Transcript policy** (Phase 4). Recommended: EDGAR exhibits only for now; third-party transcript hosts are an explicit allowlist decision.
4. **Price-data host for screens** (Phase 6). Recommended: Stooq daily CSV; without approval, discovery runs filing-driven screens only.
5. **Peer sets** (Phase 4). Defaults proposed above; edit in config at rollout.
6. **Smaller extraction model** (Phase 0.2/8). Recommended: defer; decide from measured `runs` throughput after two weeks.

## 11. Non-goals

No brokerage integration or automated trading; no paid data vendors; no cloud LLM calls (the local digest-locked model stays the only model); no scraping outside the allowlist or against source terms; no model-initiated allowlist or schedule changes, ever.

## 12. One-time tasks checklist (rollout order)

1. Land migrations + `runs` table; run `--requeue-unanalyzed` (recovers the 7 consumed filings).
2. Observe two clean live cycles of Phase 0; confirm health footer and coverage sections.
3. Dossier backfill from stored history; review seeded theses in the brief.
4. Install midday collect runs and the weekly-memo LaunchAgent when their phases land (plists live in `deploy/`, keep checked-in definitions in sync with loaded agents — verified drift-free at last audit).
5. Register EIA API key in config (Phase 4).
6. Target: Phase 5 live before the next DK/DKL earnings cycle (early August 2026) so the first quarterly review runs on real events; if the date slips, the first cycle is the following quarter — do not rush gates to hit it.
