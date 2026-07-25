# Architecture Review: Generic Pi Worker

> Independent review of the proposed "Pi as reusable local worker" architecture.
> Review-only: no software installed, no Studio changes, no configuration edited.
>
> **Verification note:** This repository contains only `README.md` on an initial
> commit. None of the systems in the packet — Pi, Athena, Laguna, the Studio,
> Ollama, or the "CloudMind operating contract" — exist in or are reachable from
> this environment. Every "current configuration observation" below is
> **unverified**; each place needing live inspection is flagged. This is a
> reasoning review of the proposal, not a verification of the machines.

---

## Answers to the 12 questions (explicit)

1. **Generic worker, Athena as first project?** Yes in principle — but the
   abstraction is premature at *one* project. Build the worker; defer the "pack
   framework" until a second project forces the interface (see #3).
2. **Pi MacBook-side, Laguna on Studio?** Yes mechanically. But this creates a
   hard runtime dependency: no Studio/Ollama → no Pi, and Pi's inference load
   competes with Athena production on the same box. That partially contradicts
   "Pi should not affect the Studio." Acceptable only with resource isolation
   and an acknowledged single-point-of-failure.
3. **Core vs project-pack separation correct?** Correct as a pattern (it's the
   plugin model). Incomplete: you need an explicit, testable *interface
   contract* (what a pack must supply, what core guarantees) and a lint that
   fails if Athena specifics leak into core. Don't design the pack interface
   from a single example.
4. **Packet protocol sufficient? Missing fields?** Close, but under-specified on
   reproducibility and trust. Missing (task): schema version, **base commit
   SHA**, model+quantization+params pinned, secrets policy, network policy,
   token/time budget, parent-task lineage, idempotency key. Missing (result):
   schema version, **base and result SHAs**, **model+params actually used**,
   full tool-call log with args, test command + exit codes + full logs (not
   pass/fail prose), files *read outside allowed paths* (leak signal), resource
   usage, and an attestation. Core principle it misses: results must be
   verifiable **without trusting Pi's narrative**.
5. **Worktrees + separate promotion enough?** No. A git worktree isolates one
   repo's tracked files; it does **not** stop a shell tool from `cd ..`,
   following symlinks, reading `~/.ssh`, or SSHing to the Studio. You need
   OS-level sandboxing, a **credential-less low-privilege user**, network egress
   control, and enforced (not requested) resource limits. This is the biggest
   gap.
6. **Invocation for v0?** CLI, JSON-in/JSON-out subprocess (`task.json` →
   `result.json` + exit code). Not RPC/daemon — that adds state and attack
   surface for zero v0 benefit and gives you no natural sandbox boundary. Defer
   RPC until you have concurrent long-running workers needing streaming.
7. **Smallest useful v0?** Much smaller than the proposed 8 steps — see
   "Recommended v0" below. Prove the *loop*, not the framework.
8. **Deterministic vs model instructions?** Every safety boundary and
   correctness gate must be deterministic code: path enforcement, worktree/SHA
   checkout, running tests, time/resource limits, kill switch, concurrency
   locks, schema validation, promotion gate. Model instructions only for:
   interpreting the objective, writing code inside the sandbox, summarizing
   evidence. Rule: if a wrong model output could cause damage or a false
   "success," deterministic code must sit between the model and the consequence.
9. **Failure modes?** Worktree escape via relative paths/symlinks; ambient
   credentials letting Pi push or edit live Athena (directly violates your own
   rule — nothing *prevents* it, the proposal only *forbids* it); context/secret
   leakage through the shared Ollama server and shared logs/caches;
   misrepresented success (self-reported "tests pass"); plausible-but-wrong code
   passing weak tests; Studio resource contention degrading Athena prod;
   non-reproducibility from unpinned model (your `q4_k_m` vs `128k` mismatch is
   already this failure mode). Detailed mitigations under "Risks."
10. **Rename away from `athena`?** Yes — cheap, reversible, improves the
    boundary. Prefer `laguna-studio` (names model + host) or `studio-ollama`
    (names the service). But the rename is cosmetic; it must not create false
    confidence, because the *runtime* coupling to the Studio remains.
11. **Measuring learning / recovery?** Make it an operational drill, not a
    feeling — see "Exit test." Key metrics: can Ronnie complete a delegated
    task, review it, promote, and roll back with Claude/Codex disabled; a "no
    trapped knowledge" audit (reproduce a recent change from committed artifacts
    only); and the trend in fraction of tasks Ronnie finishes without asking
    Claude/Codex.
12. **Conflicts with Athena / single-writer / release / CloudMind contract?**
    **Cannot verify — requires live inspection.** The specific things to check:
    does the dev worktree use its own database/state (if it points at the live
    Athena DB, you violate single-writer even from a "dev" worktree); does your
    invented promotion flow conform to an existing CloudMind release/rollback
    contract rather than replacing it; is the Studio Ollama treated as a
    production resource that dev inference load would violate.

---

## Required response

### 1. Verdict

**Agree with revisions.** The direction is sound; the separation of concerns is
right; but as written it relies on model instructions where it needs
deterministic enforcement, under-specifies reproducibility, and builds a
framework before proving the thesis. Fix those before implementation.

### 2. Restatement (to prove the intended position is understood)

You want Pi to be a *reusable, project-agnostic execution harness* that lives on
the MacBook and does bounded, well-scoped work — implementation, investigation,
testing, mechanical chores — inside an isolated git worktree, using the local
Laguna model (served by Ollama on the headless Studio) for inference. Claude and
Codex are the *senior layer*: they understand the goal, decompose it, hand Pi a
structured task packet with explicit scope/allow-lists/acceptance tests, then
**review Pi's evidence rather than trusting its self-report**, and hand Ronnie an
owner-readable brief. Ronnie is final authority and is deliberately kept able to
operate the system without external AI. Athena is merely the first *project
pack* riding on top of the generic core; the worker must not absorb Athena's
identity, paths, or deployment rules. Crucially, **Pi prepares candidates but
never promotes**: writing a dev worktree and creating/deploying a release
artifact to the Studio are separate, human-gated, reversible operations, and Pi
never edits the live Studio checkout as a normal path. The long game is to
convert Claude/Codex's recurring guidance into durable local artifacts
(runbooks, schemas, validators, ADRs, golden cases) so capability doesn't
evaporate if either vendor disappears.

### 3. What is correct (strongest parts)

- **Worker/identity separation.** Decoupling the execution harness from project
  identity is the right backbone, and naming Athena as "just the first pack" is
  the correct framing.
- **Keeping write/edit tools MacBook-side, off the production box.** Good
  instinct.
- **Promotion as a separate, human-approved, reversible operation** with a
  release artifact and rollback. This is the most important correct decision in
  the packet — keep it non-negotiable.
- **"Review evidence, don't trust self-reported success."** Correct — and it
  should be hardened into "orchestrator re-runs the gate on the diff," not left
  as a review habit.
- **Vendor-independence as an explicit, first-class goal** rather than an
  afterthought.
- **The instinct to convert guidance into durable artifacts** (runbooks, ADRs,
  golden cases).

### 4. Required revisions (before implementation)

1. **Enforcement, not instructions.** Allowed/forbidden paths must be an OS
   sandbox (macOS `sandbox-exec`/seatbelt, or a restricted user + jail), not
   text in a packet. A model told "don't touch these paths" is not a control.
2. **Credential-less worker.** Run Pi's tools as a dedicated low-privilege user
   with **no push rights, no Studio SSH keys, no access to `~/.ssh` or
   keychains**. This is what actually makes "Pi never edits the live Studio"
   true. Today, under Ronnie's user, nothing stops it.
3. **Pin the model, fix the config mismatch first.** The `laguna-s-2.1:128k`
   (registered) vs `laguna-s-2.1:q4_k_m` (default) discrepancy is a live
   reproducibility bug. Pin model name, quantization, temperature, and context
   window in the packet and record what was *actually* used in the result.
   *(Requires live inspection to confirm the current values.)*
4. **Trust boundary in the result packet.** The orchestrator re-runs acceptance
   tests on the returned diff in a **clean checkout at the recorded base SHA** —
   the result packet's "tests passed" is evidence to reproduce, not a fact to
   believe. Forbid the worker from modifying test files unless the task is
   explicitly about tests; hash the tests.
5. **Reproducibility fields.** Add base/result SHAs, model+params-used, full
   tool-call and test logs, and a "files read outside allowed paths" list to the
   result schema.
6. **Defer the pack framework.** Ship the worker with Athena
   hard-wired-but-clearly-isolated first; extract the generic pack interface
   when project #2 arrives. Designing the interface from one example will
   produce the wrong interface.
7. **Address the Studio coupling explicitly.** Either accept and document that
   Pi depends on and loads the production Studio, or give Ollama dev inference
   its own resource envelope. Don't let the rename imply an isolation that
   doesn't exist at runtime.

### 5. Risks (ranked by severity)

1. **Worker escapes isolation / uses ambient credentials → damages or deploys to
   production.** Highest severity. Directly defeats the central safety claim.
   *Mitigation:* OS sandbox + credential-less user + no Studio keys. Until this
   exists, treat "Pi never edits the live Studio" as an aspiration, not a
   property.
2. **False success — self-reported tests, edited tests, or plausible-but-wrong
   code passing weak tests.** *Mitigation:* orchestrator re-runs the gate on the
   diff; forbid/hash test edits; invest in strong acceptance tests (the whole
   scheme's guarantee is only as strong as the tests).
3. **Single-writer / live-DB violation.** If the dev worktree points at Athena's
   live database or runtime state, you break single-writer even from "dev."
   *Mitigation:* dev worktree gets its own state; verify. **Requires live
   inspection.**
4. **Cross-project context/secret leakage** through the shared Ollama server,
   reused sessions, and shared logs/caches. *Mitigation:* fresh context per
   task, per-project scratch/log dirs, no session reuse across projects.
5. **Non-reproducibility from model nondeterminism + unpinned model** (already
   manifesting as the name mismatch). *Mitigation:* pin and record everything;
   the orchestrator's re-run is the source of truth.
6. **Studio resource contention** degrading Athena production during
   large-context Pi runs. *Mitigation:* resource envelope / scheduling; monitor.
   **Requires live inspection.**
7. **The local-model-capability premise is untested.** If Laguna can't reliably
   produce mergeable diffs, the elaborate harness is over-built for the value.
   *Mitigation:* measure Laguna's ceiling before building the framework (see
   v0).
8. **Over-engineering / trapped complexity.** Building packs, bridges, RPC, and
   schemas up front risks a system only Claude/Codex can operate — the opposite
   of the vendor-independence goal. *Mitigation:* minimal v0.

### 6. Recommended v0 (minimal components and sequence)

Cut the proposed 8 steps to this. The thesis to prove is *"a local model can
produce a trustworthy, reviewable candidate through the packet → sandbox →
independent-gate → owner-brief loop"* — not "we have a framework."

1. **Fix + pin** the provider/model config; rename provider to `laguna-studio`.
   Confirm one explicit `laguna-s-2.1:128k` run and one `read`-tool run actually
   work. *(Verifies your observations.)*
2. **One JSON contract** (`task.json` / `result.json`) with the reproducibility
   fields above. No schema library yet.
3. **~100-line runner:** validates the packet → creates a git worktree at a
   **pinned SHA** → launches Pi under a **sandbox + low-priv, credential-less
   user** with the allow-list → captures the diff → **re-runs the acceptance
   test in a clean checkout** → writes `result.json` + patch. Deterministic; no
   model in the safety path.
4. **Run ONE small, reversible, real Athena task** end-to-end: delegate →
   implement → orchestrator re-runs tests on the diff → owner-readable brief →
   **stop before deploy.** No Studio changes.
5. **Then run 5–10 representative tasks** and measure: what fraction produce a
   mergeable diff with no human rewrite. *That number* — not the infrastructure
   — tells you whether the thesis holds.

Explicitly **defer**: the pack framework, dual Claude+Codex bridges, RPC,
concurrency controls, and release-artifact machinery. Add each only when a
concrete need appears.

### 7. Usage model (real project)

- **Ronnie** states the objective and constraints, receives the owner brief,
  gives/withholds approval, and owns promotion + rollback.
- **Claude/Codex** decompose the objective into bounded task packets (explicit
  scope, allow-list, acceptance tests, pinned model, base SHA), shell out to the
  runner, then **re-run the gate on the returned diff**, read the evidence, and
  write Ronnie a plain-language brief with honest uncertainty. They convert
  anything recurring into a committed runbook/script/ADR.
- **Pi + Laguna** execute inside the sandboxed worktree, return diff + evidence
  + uncertainty, and **never** approve, merge, or deploy.
- **Promotion** is a separate human-gated step producing a versioned,
  rollback-capable release artifact — Pi doesn't touch the live Studio.

### 8. Exit test (proof of non-helplessness without Claude/Codex)

A scheduled **recovery drill with Claude and Codex disabled**. Using only
committed local artifacts (runbooks, runner, schemas, validators), Ronnie must:

1. Take a pre-written task packet, run it through the runner, and get a
   `result.json` + diff.
2. Independently verify the result (re-run the gate) and read the diff.
3. Promote it to a release artifact and **then roll it back**.
4. Recover from three seeded failures: Studio/Ollama down, wrong model loaded,
   and a bad promotion.

Pass = all four completed unaided, within a time box, with zero reference to
chat history. Run it periodically; a drill that only passes with Claude/Codex
present means the knowledge is still trapped. Complement with a "no trapped
knowledge" audit (reproduce a recent real change from committed artifacts alone)
and track the trend in tasks Ronnie completes without asking either vendor.

### 9. Final recommendation

**Revise first, then proceed** — incrementally. The architecture is
directionally right and worth building. But do **not** build the framework
version. Ship the minimal v0 (Section 6) with real enforcement (sandbox +
credential-less user) and real reproducibility (pinned model, base SHA,
orchestrator-side re-run) as non-negotiables, prove the loop on one real Athena
task, then measure Laguna's ceiling across 5–10 tasks before investing in packs,
bridges, or RPC. Before any of it, reconcile against the live Athena
single-writer rule, release process, and CloudMind contract — **all of which
require inspection I cannot perform from here.**

**Verified vs inferred:** everything about the current Pi/Athena/Laguna/Studio
state is *inferred from the packet* — I confirmed only that this repository is
empty. The config mismatch, the provider name, the successful runs, and any
contract conflicts all require live inspection on the actual machines.
