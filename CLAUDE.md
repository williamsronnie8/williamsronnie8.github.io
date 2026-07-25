# CLAUDE.md — williamsronnie8.github.io

## ⚠️ Read this before writing anything here

**This repo is a personal website. That is all it is.**

It is also the repo cloud sessions attach to by default — which is how, by
2026-07-25, three separate sessions had written work here that did not belong:

- an architecture review of the Pi worker
- a DK/DKL research implementation plan
- a Phase 0 work order

None of it was ever going to be found in a repo that serves a homepage. All
three were relocated on 2026-07-25.

**So: if the work you are about to commit is not the website, it does not go
here.** Put it where it belongs:

| Kind of work | Home |
|---|---|
| Athena / DK/DKL code, plans, design docs | `williamsronnie8/athena-src` |
| Cross-project state, direction, standards | `williamsronnie8/hub` |
| Process notes, triage, SOPs | `williamsronnie8/project-reformer` |
| The website itself | here |

If no repo fits, say so and ask — don't default to this one.

---

## Start here — shared context

Before working in this repo, read **[`williamsronnie8/hub`](https://github.com/williamsronnie8/hub)**:

| File | What it gives you |
|---|---|
| `README.md` | Current state of every project, plus the synthesis |
| `DIRECTION.md` | Architecture, open questions, what's already decided |
| `STANDARDS.md` | What may reach a repo — agents, secrets, commits, branches |
| `ROUTING.md` | Local model vs. frontier, with measured evidence |

The hub is the source of truth for cross-project state. **Update it when this
project changes state** — a stale hub teaches you not to trust it.

---

## About this repo

Static personal site published by GitHub Pages from `master` at
https://williamsronnie8.github.io

- `index.html` — the whole site. Single file, no build step, no dependencies.
- Automatic light/dark via `prefers-color-scheme`; responsive.

**Merging to `master` publishes immediately.** There is no staging environment.
Work on a branch and open a PR — see `hub/STANDARDS.md`.

This repo is **public**. Never commit anything here you wouldn't post publicly.
