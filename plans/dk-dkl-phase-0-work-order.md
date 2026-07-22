# Phase 0 Work-Order — run this ON the athena machine

**Why this file exists:** Phase 0 modifies the live DK/DKL research system, which exists only on the athena Mac (`~/athena-src` / `/Users/athena/athena-src`, live DB at `~/.athena/investment-research/`, LaunchAgents). It was never pushed to GitHub, and the cloud planning session cannot reach the Mac. So run Phase 0 in a **Claude Code session on the athena machine itself**. Paste the block below as your first message there.

Full plan for context: `plans/dk-dkl-research-implementation-plan.md` in `williamsronnie8/williamsronnie8.github.io` (branch `claude/dk-dkl-research-plan-upwqdv`). This work-order is self-contained — you do not need the plan to execute, but read Phase 0 there if you want the rationale.

---

## PASTE-READY PROMPT

You are running on the athena machine, which hosts the live DK/DKL investment-research system. Implement **Phase 0 — Pipeline integrity** of the DK/DKL research plan. Phase 0 stops the daily loop from losing work and from overstating what it verified. Nothing else in the plan is in scope; do not build dossiers, alerts, queue, market data, or new reports here.

### Ground rules
- **Additive only.** No destructive DB migrations, no dropped columns, no rewrites. Every schema change is a numbered, forward-only migration.
- **Schema-locked model I/O.** Every model call keeps a JSON schema; invalid output is rejected and bounded-retried, never partially trusted.
- **Loud degradation.** Every failure path Phase 0 touches gets a visible surface (brief footer / report section). No silent drops.
- **Local model only**, the existing digest-locked `qwen3.6:27b-q8_0`. The host allowlist is unchanged — add no new hosts.
- **Feature-flag** every behavior change; flags default **off** until acceptance passes in a live cycle.
- Line anchors below come from an audit and **will have drifted** — re-locate each concern by content, not line number, and confirm the real SQLite schema (table and column names for sources/evidence/reports) before writing any migration.

### Step 0 — Version the current code first
The deployed DK/DKL system is not in git. Before changing anything: create a branch (e.g. `dk-dkl-phase-0`), commit the **current working** `scripts/dk_dkl_research.py`, the `deploy/com.athena.dk-dkl-research-*.plist` files, config, and any existing tests as a clean baseline commit. Push to `williamsronnie8/athena-src` so Phase 0 lands as a reviewable diff. Do not mix the baseline commit with Phase 0 changes.

### Step 1 — Migration infrastructure (do this first)
Add a `schema_version` table and a numbered, additive-only migration runner. All later schema changes in this work-order land as migrations through it.

### Step 2 — S1: decouple fetch success from analysis success
Audit anchor: evidence marked processed regardless of synthesis outcome (~`dk_dkl_research.py:965`). This already consumed 7 historical filings.
- Add to the evidence/sources table: `analysis_state` (`pending|analyzed|failed|exhausted`), `analysis_attempts`, `last_analysis_error`, `analyzed_at`, `analyzed_in_report_id`.
- Backfill migration: rows cited by an existing stored report → `analyzed`; all other fetched-but-unreported rows → `pending`. This re-queues the 7 consumed filings automatically.
- Analysis batch selection becomes `analysis_state IN ('pending','failed') AND analysis_attempts < 3`, ordered primary-sources-first, newest filing first.
- Mark a row `analyzed` **only inside the same SQLite transaction** that persists a validated report. On synthesis failure: increment `analysis_attempts`, store `last_analysis_error`, leave the row eligible. After 3 failures → `exhausted` (this is surfaced in the footer, Step 6 — never silent).
- Add a one-shot CLI command `--requeue-unanalyzed` for manual recovery.

### Step 3 — runs table + observability
Add a `runs` table (job, stage, wall-time, counts, outcome) written by every job. It feeds the health footer and lets batch/chunk caps be tuned from measured data.

### Step 4 — S2: whole-filing coverage via per-filing map-reduce
Audit anchor: evidence clipped into a shared 60,000-char packet, ~7.5k chars/filing at the 8-item batch (~`dk_dkl_research.py:686`).
- **Split (code):** strip inline XBRL/HTML to text (downloads already allow up to 6 MB), split by SEC structure — 10-K/10-Q by Item heading, 8-K by Item number, exhibits indexed by type. Always retain financial-statement sections; always drop boilerplate (certifications, XBRL tag dumps).
- **Map (model):** per-chunk extraction with a small schema (facts, verbatim quotes, section reference), ~24k chars/chunk, hard cap ~12 chunks/filing. Anything skipped is recorded in a `coverage` field and **printed in the report's Coverage section** — truncation becomes visible.
- **Reduce (model):** the existing draft + skeptic synthesis (~`:827`) consumes extracted facts, not raw clipped text.
- **Runtime:** a 27B Q8 doing map passes will not fit the 8:15 brief window. **Move analysis into the 1:00/6:30 collect runs; make the 8:15 job assemble-and-deliver only** (reads stored per-filing analyses, composes the brief). This also makes delivery immune to a single filing's model failure. Log per-stage wall-time to the `runs` table. If throughput is too slow, note it and tune caps from `runs` data — do not switch models yet.

### Step 5 — S3: deterministic citation verification
Audit anchor: citation checks are structural only (~`dk_dkl_research.py:721`).
- Findings schema gains `quotes: [{source_id, verbatim}]`; numeric claims the model marks as sourced gain `numbers: [{surface_form, source_id}]`.
- Code verifies, after whitespace/HTML-entity normalization: each verbatim quote appears in the stored text of its cited source; each numeric surface form (e.g. `$118.7 million`, `(3.2)%`) appears in its cited source. **No unit conversion or arithmetic in v1** — verify surface forms only, conservatively.
- Findings failing verification are dropped and fed back to the skeptic pass with reasons for **one** bounded redraft; still failing → excluded and counted in the footer.
- The brief labels each finding **verified** (quote-checked) or **model-attested** (structural only). Keep the existing structural checks as gate one.

### Step 6 — Pipeline-health footer
Every brief ends with one deterministic line: backlog depth, pending/failed/exhausted analysis counts, last collect run status, model-failure count in window, delivery outbox state. This is the loud-degradation surface the whole plan depends on.

### Step 7 — Module extraction (strangler, no big-bang)
Extract the DB/store concerns touched above into `research/store.py`; the script becomes the entrypoint that wires the module. Only extract what Phase 0 touches.

### Step 8 — Gate additions (extend the investment gate / `make test`)
- No evidence row reaches `exhausted` without 3 recorded attempts.
- Report-persist and `analyzed`-mark are atomic (crash-injection test).
- A fixture filing with a known quote passes verification; a perturbed quote fails.
- A filing larger than the packet budget yields a non-empty Coverage section.
- The brief renders with the health footer even when synthesis produced zero findings.

### Step 9 — Operational rollout (only possible here, on the Mac)
1. Run migrations against the live DB; run `--requeue-unanalyzed` and confirm the 7 historical filings become eligible again.
2. Run `make test` / the investment gate — must be green, including the new checks.
3. Keep feature flags off, then enable and observe **two clean live cycles** (collect → analyze → deliver). Confirm the health footer and Coverage sections appear in real briefs, and that a re-queued filing gets a real analysis attempt.
4. If analysis moved into the collect runs (Step 4), update `deploy/com.athena.dk-dkl-research-collect.plist` / the brief plist accordingly, reload the LaunchAgents, and keep the checked-in plist definitions in sync with what's loaded.

### Acceptance (report these back)
- The 7 historical filings are re-analyzed or visibly `exhausted` (not silently dropped).
- A forced model failure leaves the filing eligible and the brief's footer says so.
- A full-size 10-K produces per-section extraction with a populated Coverage section.
- `make test` green with the Phase 0 gate additions.
- Phase 0 changes committed on a branch and pushed to `williamsronnie8/athena-src`, separate from the Step 0 baseline commit.

Work in reasonable increments, keep commits scoped, and stop to check with me before anything destructive or anything that changes the host allowlist or the model digest.

---

## Notes for the operator (you, not the pasted session)
- The pasted session needs shell + file access to `~/athena-src` and the live DB on the Mac; run it there, not in the cloud.
- If the deployed code differs from what the audit described, trust the code — tell the session to adapt anchors to reality.
- Phase 1+ stays blocked until Phase 0 clears two clean live cycles, per the plan's phase gate.
