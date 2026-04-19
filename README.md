# longspec

Observable-driven development regimen for long-running AI-assisted engineering tasks.

## Overview

`longspec` is a set of four skills (`superspec`, `superplan`, `supertdd`, `supersdd`) that anchor every step of a long engineering task — from requirements clarification to end-to-end verification — to a single unit of verifiability: the **observable**.

Long tasks drift. Agents forget constraints mid-session, sessions compact away critical context, and subagents invent facts when the spec is ambiguous. `longspec` prevents drift not by asking the agent to remember more, but by making every commitment the spec makes, every task the plan produces, and every commit the implementer writes trace back to the same observable entries in a shared registry. When drift happens, it shows up as an unmet observable — not as a silent failure three hours later.

## Relationship to Superpowers

`longspec` evolved from [Superpowers](https://github.com/obra/superpowers) and inherits its core loop: `brainstorm → plan → TDD → subagent execution → review → finish`. It keeps Superpowers' philosophy — test-driven development, sign-off gates, fresh subagents per task — and extends it in three directions specifically for long-running AI-assisted work:

| Dimension | Superpowers | longspec |
|---|---|---|
| Core unit of verifiability | "Verification anchor" mentioned informally | **observable** — a named, runtime-surfaced behavior with explicit status in a shared registry |
| Environment grounding | None | **observable registry** — a single file describing every external tool, service, fixture, and protocol the system uses, with a verified/unverified status |
| End-to-end verification | Left to a manual `verification-before-completion` step | **story document** — end-to-end scenarios whose steps link to observables; runs incrementally as tasks complete |
| Drift detection | Relies on session continuity | File-level: observable registry + story + plan form a checkable chain |

In short: Superpowers is an **in-session** discipline (brainstorm before code, RED before GREEN, review before merge). `longspec` extends that discipline to be **cross-session** and **cross-agent**, which is what long tasks need.

## What It Solves

1. **Manifest fabrication.** Agents inventing tool names, API shapes, or data fixtures to make unit tests pass — and the fabrication only being caught when the real system is hit. `longspec` requires every external dependency to be a `verified` observable in the registry before it can be referenced.

2. **Lost scenarios.** Plan tasks completing but no one knowing whether the end-to-end user flow still works. `longspec` ties each task to linked observables, ties each linked observable to one or more observable descs in scenario steps (back-propagated into `story.md` after spec approval), and runs scenarios incrementally as their linked observables reach `verified`.

3. **Session-boundary drift.** Context shrinking across `/compact` or subagent dispatches, taking key constraints with it. `longspec` externalizes constraints into the observable registry and story document, so any agent resuming the task reads the same ground truth.

4. **Drift-by-rename.** A function renamed in task 7 that was called by name in task 3's description. `longspec` carries the `Observes:` field on every task and validates type/name consistency against the observable registry during self-review.

5. **Bikeshedding on structure.** Endless discussion on how to organize plans, where to put docs, what to call things. `longspec` fixes a minimal layout and stays out of the way.

## Core Concepts

```
story (one per project)
  └── scenario (= one business event)
        ├── owner + goal
        ├── depends-on (optional; downstream derives batches from it)
        ├── step (input action + visible outcome)
        │     └── observable desc (witness-able phenomenon)
        │           └── links to → linked observable (N:M)
        └── end state (last step's outcome)

spec (multiple per project, each produced by one superspec run)
  └── observable (trigger + action + outcome, status in the registry)

plan (one per spec)
  └── task
        └── Observes: [observable names]

commit
  └── advances observables → Observable Trace updates → registry status flips when scenario passes
```

- **observable** — the single unit of verifiability. A named runtime-surfaced behavior with status `verified` / `unverified` / `pending`.
- **registry** — the shared catalog of observables at `docs/longspec/observables.md`. Any reference to an external tool, service, fixture, or protocol resolves here.
- **story** — the end-to-end business scenarios at `docs/longspec/<project>/story.md`. Each scenario's steps link to observables.
- **scenario** — one complete user journey. Runs end-to-end when all its observables reach `verified`.

## The Four Skills

| Skill | Purpose | Output |
|---|---|---|
| `superspec` | Phase 1 Boot → Phase 2 Story (co-author scenarios / steps / observable descs with MECE completeness sweep) → Phase 3 Spec (author spec, register linked observables, back-propagate linked names into story.md) | `story.md` + `<project>/YYYY-MM-DD-<topic>-spec.md` + registry updated |
| `superplan` | Convert a spec into a task plan; order tasks for earliest scenario completion | `<project>/YYYY-MM-DD-<topic>-plan.md` |
| `supertdd` | RED → GREEN → REFACTOR for each task, with an Observable Trace step per commit | code + tests + commits |
| `supersdd` | Dispatch fresh subagent per task; spec + code quality review each commit; run scenario end-to-end when its observables fully cover | subagent runs + review reports + scenario verifications |

## File Layout

```
docs/longspec/
├── observables.md                             # registry (repo-wide)
└── <project>/
    ├── story.md                               # scenarios
    ├── YYYY-MM-DD-<topic>-spec.md             # one per feature
    └── YYYY-MM-DD-<topic>-plan.md             # one per feature
```

## Quick Start

1. **Install** the four skills to `~/.claude/skills/` (or your platform's skills dir):
   ```sh
   git clone https://github.com/himalayasy/longspec.git ~/src/longspec
   cp -r ~/src/longspec/skills/{superspec,superplan,supertdd,supersdd} ~/.claude/skills/
   ```

2. **Start a new feature**: invoke `superspec`. It boots the workspace (seeds `docs/longspec/observables.md` the first time), co-authors `story.md` with you, runs a MECE completeness sweep across the story's three coverage relations, then authors the spec and back-propagates linked observables into `story.md`.

3. **Plan** the implementation: invoke `superplan`. Tasks order themselves to turn scenarios `passed` as early as possible.

4. **Execute** the plan: invoke `supersdd`. It dispatches subagents (each running `supertdd`) and auto-runs scenarios as their observables fill in.

## Writing Principles

`longspec`'s skills are themselves written under a 16-principle rubric (expression / content-judgment / structure / collaboration / decision / conflict-arbitration layers). See [PRINCIPLES.md](./PRINCIPLES.md) for the full rubric — the key ones:

- **State what *is*, not what isn't.** "Tests use real code" beats "don't over-mock."
- **One goal sentence beats twenty step instructions.** "Verify it independently" beats "run start command, then curl, then check response body."
- **First principles before Occam.** Before picking the smallest solution, return to origin: is this the same problem as an existing one? Is the current path even right? Path dependence dressed as refinement is the subtle drift.
- **Anchors before fields.** Rules for a new process must cover four faces: what good looks like / where things live / what to do when stuck / what the artifact looks like.

## Non-Goals

- Replacing human review. `longspec` makes drift visible earlier; it doesn't decide whether the work is correct.
- Being a CI/CD system. Scenario verification runs where you run it; `longspec` prescribes the inputs and pass criteria, not the execution environment.
- Being opinionated about your language, framework, or deployment model.

## License

MIT — see [LICENSE](./LICENSE).

## Acknowledgements

`longspec` is a derivative work of [Superpowers](https://github.com/obra/superpowers) by Jesse Vincent and the team at Prime Radiant. The core loop, the TDD discipline, and the subagent-driven-development pattern are all Superpowers; `longspec` extends these for long-running AI-assisted tasks with observable anchoring.
