---
name: superspec
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Three-phase guided workflow: boot workspace, elicit story from the user (scenarios / steps / observable descs), then author spec and register linked observables."
---

# superspec — Three-Phase Guided Spec Authoring

Guide the user through a story-first, spec-second workflow. Act as the user's product manager agent: explain concepts at each level, give concrete examples, let the user fill in, review the answer, confirm, then move on. Three phases, three gates.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until the three-phase gate sequence has all been approved by the user:
- **Gate 1 (Phase 1 → Phase 2):** workspace ready (project directory + observable registry seeded + story.md initialized)
- **Gate 2 (Phase 2 → Phase 3):** story.md fully approved (description + scenarios + steps + observable descs, all at user-observer perspective)
- **Gate 3 (Phase 3 → superplan):** spec approved + all linked observables registered + desc-to-linked mapping complete

This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A todo list, a single-function utility, a config change — all of them. "Simple" projects are where unexamined assumptions cause the most wasted work. The design can be short (a few sentences for truly simple projects), but you MUST present it and get approval.

## Anti-Pattern: Justifications for Added Complexity

Three excuses to reject on sight when tempted to keep scope large:
- "Future extensibility" / "flexibility" — YAGNI. Build it when a caller asks.
- "Symmetry" / "completeness" — no caller asked for the other half.
- "This needs a new mechanism" — compose existing ones first; invent only when composition fails.

## Terminology: observable desc vs linked observable

- **Observable desc** (lives in `story.md`, authored in Phase 2) — the **user-facing acceptance language**. Names a concrete observation point (which page / CLI / API / log / DB / IM / email / notification channel) and the data produced or changed there. (e.g. "on the jobs page, the table gains a row with status=queued"; "`scheduler status <id>` prints `state=running`"; "the IM channel posts a card titled `Run <id> finished`".)
- **Linked observable** (lives in `docs/longspec/observables.md`, committed in Phase 3) — the **implementation-facing unit**: a named runtime behavior with a verifiable surface (service endpoint, callable tool, fixture id, protocol rule) and a status (`verified` / `unverified` / `pending`).

**Relationship:** every observable desc in a scenario step maps to ≥1 linked observable. Descs name what a user witnesses; linked observables name what the implementation exposes. A bare component name (e.g. `ui.api_jobs_list`) is a linked observable at best; never a valid desc.

**Purpose of `observables.md`** — it exists to give an LLM two kinds of signal:
1. **Verification** — after implementation, an LLM reads the registry to check whether a story scenario passed.
2. **Diagnosis** — when a scenario fails, an LLM reads the registry as a trace: each entry is a checkpoint in the data flow that a failing scenario depends on, so the LLM can narrow the failure to one spec's one step.

Both readers are LLMs; the registry is not a human-facing catalog, not a completeness inventory, not an API docs replica.

Add an entry when either use-case needs it — a desc will directly consume it (verification), or a scenario's failure path would blind the debugger without this checkpoint (diagnosis). Do not add entries that serve neither. Merge entries when one rule covers multiple descs; **do not merge when entries sit at different points of the same data flow** — keeping them separate is what makes diagnosis work. Drop entries the moment no scenario consumes them for either verification or diagnosis.

A single-line rule is a valid entry if that is all an LLM needs to verify or to locate a failure — formal fields are optional, not required.

## Progress checkpoint (user-triggered)

When the user signals a checkpoint — any natural-language intent to record progress, save state, or note next steps — append an entry to `docs/longspec/<project>/process.md`. Create the file if missing.

Entry shape, four fields driven by what the next load needs:

```
## YYYY-MM-DD HH:MM

**Stage:** current phase and sub-step (e.g. `Phase 2, Sub-step 2.4 (MECE sweep — L1→L2 gap pending)`).
**Decided:** non-obvious decisions already locked in that the next session must not re-litigate (owners, goals, scenario list, selected approach, etc.).
**Open:** the specific unresolved question or gap the session stopped on.
**Next:** ordered TODOs for the next session.
```

Capture what the user says; if they give no text, ask once: "stage / decided / open / next — what should I record?" and write what they answer. Agent does not infer Decided or Open from the transcript unilaterally.

On next load (see 1.1 slash-with-name branch), agent echoes a single line: `loaded <project> — last checkpoint <when>, stage: <stage>. Next: <first Next item>. Continue here?` Further fields are read only when the user asks.

**Fallback when process.md is missing:** read the project files and infer the resume point; present it as an inference for the user to confirm or override.

## Phase 1: Boot

Establish the project workspace and identify the working mode. Ends when Gate 1 passes.

### Sub-step 1.1 — Entry dispatch

Three entry paths; pick exactly one:

- **Slash invocation with a project name** (e.g. `/superspec my-project`): treat `my-project` as the target. If `docs/longspec/my-project/story.md` exists, load it — working mode = **extend**. If `docs/longspec/my-project/process.md` exists, read its last checkpoint and echo a single line: `loaded <project> — last checkpoint <when>, stage: <stage>. Next: <first Next item>. Continue here?` Further checkpoint fields are read only when the user asks. If `process.md` is missing, read the project files and infer the resume point; present it as an inference for the user to confirm or override. Then proceed to 1.3. If the project does not exist, stop and ask: "no project named `my-project` found. Create it as a new project, or pick an existing one?" Do not silently convert into new-project mode.
- **Slash invocation without a name, and existing projects present**: list every `docs/longspec/<name>/` that contains `story.md`, plus a "create new project" option. Ask the user to pick. Existing → working mode = **extend**, skip to 1.3. "Create new" → continue to 1.2.
- **Slash invocation without a name, and no existing projects**: continue directly to 1.2 (new-project path).

### Sub-step 1.2 — Project name & fork check

Applies only when 1.1 dispatched to the new-project path.

Derive from the user's first request a short kebab-case identifier capturing the project's scope (e.g. `order-pipeline`, `auth-refactor`). Confirm with the user in one line.

Then ask: does this new project want to fork an existing `story.md` (from another project in this repo, or from a path the user provides) as its starting baseline?
- **No** → working mode = **fresh-start**.
- **Yes** → working mode = **fork**: copy the source story into the new project's directory; Phase 2 runs as change elicitation on the copy.

### Sub-step 1.3 — Prepare workspace per mode

- **fresh-start:**
  - Ensure `docs/longspec/<projectname>/` exists. Create if missing.
  - Ensure `docs/longspec/observables.md` exists. If missing, create a skeleton with two sections (`## Runtime-provided observables` / `## Spec-committed observables`). Then stop and ask the user how to seed the runtime-provided section. Do NOT proceed until the user has either (a) provided concrete sources (config file paths, doc paths, search terms) from which at least one runtime-provided observable is extracted into the registry, or (b) explicitly confirmed "this project touches no external runtime."
  - Ensure `docs/longspec/<projectname>/story.md` exists; if missing, create an empty skeleton (project description placeholder + Scenarios section placeholder).
- **fork:** copy the source `story.md` into `docs/longspec/<projectname>/story.md`; then apply the extend branch's validation below.
- **extend:**
  - Read the project's `story.md` and `observables.md`; validate they match the expected schema (scenario fields, observable status machine).
  - If schema invalid, ask the user to clean up the baseline before proceeding.
  - Take a snapshot of current state as reference for later diffs.

### Gate 1 (Phase 1 → Phase 2)

Project directory + observables.md + story.md are all in place; working mode confirmed.

## Phase 2: Story Elicitation

Co-author `story.md` with the user (or modify it together in extend mode). Explain each concept, give examples, propose concrete drafts based on the user's intent, let the user accept / amend / reject, confirm the committed wording, then move to the next level.

Drafts are proposals, not decisions — user's words override any wording proposed here. The collaboration shape at every level is: explain → propose draft → user adjusts → confirm → commit.

### What story.md answers

1. **Who it is for:** the north-star document for the superspec → superplan → supertdd → supersdd chain; every downstream decision traces back here.
2. **Proposal:** the current observed pains in the existing system, named as pains not solutions. (e.g. wasted capability on deterministic work; responsibility placed where no real decision exists; business logic hardcoded in the harness; business configuration carried only by natural-language prompts; delivery outputs buried in the filesystem with no business-grade surface.)
3. **Scenarios:** workflows that can be decoupled by goal and owned by different roles at different times; scenarios with different goals or different owning roles split apart. (e.g. "configure an HR agent for a role profile" — owner: hiring manager, goal: best-fit configuration — vs. "execute the configured recruiting run" — owner: recruiter, goal: run-to-rule.) Beyond the role/goal split, prefer finer scenarios over coarser ones: each scenario becomes an independent execution unit downstream (superplan groups it into a batch; supersdd runs it in a fresh subagent), so smaller scenarios shorten subagent context and reduce dialogue turns. Stop splitting when a further split would leave a piece that cannot stand alone.
4. **Steps:** the key rhythm points the owning role cares about within a scenario; not every internal action is a step, only points where the owning role needs visibility or control. (e.g. for an execution scenario: kick off the run, confirm the active configuration, watch critical checkpoints during execution, reach completion.)
5. **Observable descs:** for each step, the concrete signals that confirm the step has progressed — where to look and what data shows up or changes there. (e.g. "the web page shows a table row with status=queued"; "the dialog displays 'N items extracted'"; "the CLI `status <id>` prints a line containing `state=running`"; "the API `GET /jobs/<id>` returns `{state: "done"}`"; "the log stream writes `job-<id> completed`"; "the `jobs` DB row contains status=done, finished_at=<timestamp>"; "the email arrives with subject `Quarterly filing submitted`"; "the IM channel posts a card titled `Run <id> finished`"; "the mobile push notification reads `Filing accepted by authority`".)

Any of the five left unanswered is a gap the MECE sweep (Sub-step 2.4) or Gate 2 will catch.

### Flow branches by working mode

- **fresh-start mode:** L0 description first (independent pass), then enter the scenario-loop: for each scenario, cowork its name+owner+goal+steps+observable descs in one session and write it before moving to the next.
- **extend mode:** ask the user which scenario(s) this change touches, or whether L0 changes. Run the scenario-loop only on the selected scenarios (each one fully coworked end-to-end). End with a diff view of story.md for user approval.

**Scenario-loop cowork (shared by both modes).** Within one scenario's session: explain L1/L2/L3 concepts once (only the first time the loop starts, or when extend mode touches a level the user hasn't worked on); propose a concrete draft of the scenario's name+owner+goal, its steps, and each step's observable descs as one block; user approves/amends/rejects the block; write the approved block to story.md; move to the next scenario. Sub-step 2.4 MECE sweep runs once after all scenarios are written.

### Level guide template (used at each level)

For each level you run as an independent pass (L0 in fresh-start; the initial L1-concept explanation at the start of the scenario-loop):
1. **Concept:** explain in 1-2 sentences what this level is
2. **Example:** show a short concrete example from a different domain than the user's project, so they understand the shape without copying
3. **Prompt:** ask the user to list/describe items at this level
4. **Review:** check the user's answer against the level's criteria (see per-level criteria below)
5. **Confirm:** echo back what will be written, wait for explicit "approved" before writing to story.md

Inside the scenario-loop, L2 steps and L3 descs are coworked inline — the scenario is drafted end-to-end and approved as one block, not level by level. The per-level Criteria sections below still apply and are used to review the block.

### L0 — Story description (container)

- **Concept:** see Who it is for (§1) and Proposal (§2) in What story.md answers.
- **Example:** "A small-business owner currently spends a full week every quarter hand-sorting paper receipts to file tax reports; missed line items and late submissions cause fines. This story covers an assistant that takes over the sorting-and-filing burden."
- **Criteria:** identifies the user role, names the current pains (not the solution), states the scope boundary (what's in, what's out).

### L1 — Scenarios (indivisible units of user work)

- **Concept:** see Scenarios (§3) in What story.md answers.
- **Example:** For the invoice assistant:
  - "upload a new batch of receipts and have them extracted" (owner: business owner; goal: ingest this week's receipts)
  - "generate the quarterly tax filing" (owner: business owner; goal: submit to the authority)
- **Criteria:** each scenario names its owner + goal, and must (a) be completable on its own, (b) not overlap with another scenario's intent, (c) have a clear start and a user-visible end. If a "scenario" requires another to have happened first, it may be a later step of a larger scenario — consider merging.
- **Optional field — `depends-on`:** list scenario names that must already be `passed` before this scenario can start (e.g. an "execute the configured recruiting run" scenario lists `depends-on: [configure an HR agent for a role profile]` because execution requires a committed configuration). Omit when there is no such precondition. downstream superplan derives batches from owner + depends-on; users do not declare batches manually.

### L2 — Steps (the sequence inside one scenario)

- **Concept:** see Steps (§4) in What story.md answers.
- **Example:** For "upload a new batch of receipts":
  ```
  step 1: user uploads a zip of receipt images via the web upload form
  step 2: system extracts line items and displays them for review
  step 3: user reviews the extracted items, correcting any mistakes
  step 4: user confirms and the batch is filed into the receipts library
  ```
- **Criteria:** steps in order, each step one action, sum of steps completes the scenario, no step can be safely removed without breaking the flow.

### L3 — Observable descs (witness-able consequences per step)

- **Concept:** see Observable descs (§5) in What story.md answers. ID each desc as `<step>.<letter>` (1a, 1b, 2a, ...).
- **Example:** For step 2 "system extracts line items and displays them for review":
  ```
  2a. On the review page, the table shows one row per extracted line item with columns date / vendor / amount / category / source filename.
  2b. The `receipts` table gains N rows with status=extracted and reviewed_at=null.
  ```
  Two descs cover two observer faces (UI + persistence); add more faces only when this step's consequences genuinely land there.
- **Criteria:** each desc (a) is witness-able by a user or external observer, (b) names where to look and what data shows up / changes there, (c) specific enough that two people checking would agree on pass/fail.

### extend mode: change elicitation

When the user has indicated extend mode and chosen which level(s) to modify:

1. For each selected level, ask: "what's changing — add new item(s) / modify existing / remove?" Walk the relevant level guide for each addition or modification.
2. When all requested changes are drafted, present a diff of story.md (before / after for each changed section).
3. Wait for explicit user approval on the diff.
4. If approved, write to story.md and proceed to Gate 2.

### Sub-step 2.4 — MECE completeness sweep

Applies to both fresh-start and extend modes, before Gate 2. Check each of the three relations below as a MECE classification (mutual exclusion + collective exhaustion). For each relation:

1. **Judge**: pass or fail, with a one-line reason naming the specific ME or CE violation.
2. **Propose repartition** (only when fail): show one complete repartition of the affected layer that restores MECE.
3. **Offer alternatives** (only when multiple equally valid repartitions exist): present each as a labeled option for the user to choose; do not invent options when the correct partition is unique.

Three relations:

- **L0 → L1 (scenarios partition user-intent phrases):** every user-intent phrase in the L0 description belongs to exactly one scenario. ME violation = one phrase appears in two scenarios' intents. CE violation = a phrase belongs to no scenario.
- **L1 → L2 (steps partition the scenario's progress):** each scenario's steps form an ordered chain from trigger to end state with no repeated state transitions and no gaps. ME violation = two steps advance the same state. CE violation = an adjacent step pair leaves an unexplained jump.
- **L2 → L3 (descs partition the step's observable consequences):** list the observer faces relevant to this project (candidate faces drawn from context — e.g. user UI / log / persistence / external system / metric / notification) and ask the user once to confirm which faces apply to this project. Per step, each confirmed face has ≥1 desc or an explicit "not applicable" tag; no two descs witness the same observation point and same data.

**Output shape, unique-repartition case:**
```
L0 → L1: FAIL — CE violation: phrase "历史申报查阅" in L0 is not covered by any scenario.
Repartition: add scenario "查看历史申报记录" between existing scenarios 3 and 4.
User: approve / amend / reject?
```

**Output shape, multiple-repartition case:**
```
L1 → L2 (scenario "提交申报"): FAIL — CE violation: outcome chain ends at "user clicked submit", but the scenario's end state is "回执归档".
Alternative repartitions:
  1. Add step "系统接收税务局回执并归档" after current last step.
  2. Split scenario into "提交申报" (ends at submit) + "接收回执" (new scenario from回执 arrival to归档).
User: pick 1 / 2 / amend / reject.
```

When a relation cannot be judged (e.g. observer faces for the domain are unclear), name the single blocker and ask the user for that one datum. Escalation is always specific; never "please double-check the story."

### Gate 2 (Phase 2 → Phase 3)

User has approved the final story.md content. Every scenario carries name + steps; every step has ≥1 observable desc; every desc is phrased as a witness-able phenomenon; the MECE sweep reports PASS on all three relations.

## Phase 3: Spec Authoring

Based on the approved story.md, author the spec and register the linked observables that implement each observable desc.

### Sub-step 3.0 — Spec decomposition (MECE by domain)

**Cut the work into specs by the same axis Phase 2 used for scenarios: different role, different benefit goal.**

Split drivers — cut further when any of these apply:
- **Decoupling:** one part must keep working when the other changes.
- **Reusability / extensibility:** one part is meant to serve domains beyond the current project.
- **Independent verification:** one part can be signed off and verified end-to-end without the other being done.

**MECE check:**
- **Mutual exclusion:** spec responsibilities must not overlap. Overlap means the same change would touch both specs; cut again or merge.
- **Collective exhaustion:** every scenario in story.md traces to at least one spec. Homeless scenarios are either out-of-scope (add as non-goal) or signal a missing spec.

### Sub-step 3.1 — Technical clarification

Ask the user clarifying questions about technical constraints the story didn't cover (one at a time, multiple choice where possible): architecture constraints, existing components to reuse, deployment targets, data model choices. Skip levels the user doesn't care about.

### Sub-step 3.2 — Approach selection (Occam)

Propose 2-3 approaches with trade-offs.
- Rank options by (new components, files touched, new concepts introduced) ascending; recommend the smaller unless the dropped scope is the actual task.
- Before proposing, cut each option in half and check if the task still gets done; if yes, that halved version is the proposal.
- User picks one.

### Sub-step 3.3 — Present design

Present design in sections scaled to complexity: architecture, components, data flow, error handling, testing. Each section traces back to one or more scenarios from Phase 2. Get user approval after each section.

### Sub-step 3.4 — Write spec file

Save to `docs/longspec/<projectname>/YYYY-MM-DD-<topic>-spec.md` and commit.

**Spec writing constraints:**

The spec is read by humans (for sign-off) and downstream skills (for plan derivation). It must stay short enough to actually read and precise enough to act on:

- Each section earns its words by removing ambiguity downstream skills would otherwise resolve by guessing
- Lead with the goal; add path detail only when downstream plan tasks would have to invent it. "Authenticated requests get rate-limited per user" suffices; the exact bucket algorithm only matters if it's novel
- Skip what the reader already knows from the codebase, the inventory, or general engineering knowledge
- Examples appear only when they prevent a real misreading; mark them as examples (`e.g.`, `for reference`)
- Any constraint the agent or user might be tempted to ignore (because "this case is special") gets stated as a rule, not a suggestion

If a section runs past a screen of text, it is either covering multiple sub-features (decompose) or repeating itself (compress).

### Sub-step 3.5 — Spec self-review

Look at the written spec with fresh eyes:

1. **Placeholder scan:** any "TBD", "TODO", incomplete sections, or vague requirements? Fix them.
2. **Internal consistency:** do any sections contradict each other? Does the architecture match feature descriptions?
3. **Scope check:** focused enough for a single implementation plan, or does it need decomposition?
4. **Ambiguity check:** could any requirement be interpreted two different ways? Pick one explicitly.
5. **MECE check:** delete any section — does the spec lose unique information? If not, merge or drop. Then verify each decision category has a home (not necessarily its own section): WHO/WHEN triggers, WHAT contracts, HOW mechanisms, failure semantics, observability, explicit non-goals.
6. **Non-goals present:** has explicit "Non-goals" section listing what's deliberately deferred this round. Each line one sentence.

Fix inline. No need to re-review — just fix and move on.

### Sub-step 3.6 — User reviews spec

> "Spec written and committed to `<path>`. Please review it and let me know if you want to make any changes before we move to observable mapping."

Wait for user's response. Request changes → re-run self-review. Only proceed on approval.

### Sub-step 3.7 — Observable Readiness Check

Each linked-observable registry entry carries whatever an LLM needs to verify the scenario or to locate a failure along the data flow, and no more. A name and a one-line rule are the minimum; belongs-to (`runtime-provided` vs a spec name), how to observe, evidence (≤50-char note for `verified` runtime-provided entries, e.g. `curl GET /jobs → 3 rows, 2025-04-19`), snapshot timestamp, and status (`verified` / `unverified` / `pending`) are added only when the LLM reader — whether verifying or diagnosing — would otherwise be unable to act; skip any field whose absence blocks neither purpose.

**Default entry shape** — one line: `` `<name>` [<status>] — <rule an LLM can turn into an assertion>. `` Extra fields inline when an LLM reader would otherwise be stuck (e.g. a ≤50-char evidence note tacked onto a verified runtime-provided entry). Multi-line frontmatter — separate `how to observe` / `snapshot` / `status` bullets per entry — is the anti-pattern: it multiplies the file without adding LLM-actionable information.

**Status transitions are restricted:**
- `unverified → verified` (runtime-provided): agent exercises the runtime through any reachable channel — **including the entry's own `how to observe` method even though the entry is still `unverified`** (that method is a proposal to be tested, not a prerequisite), plus any other reachable channel when needed (HTTP call, CLI, browser interaction, MCP tool, direct DB query, existing test fixture, etc.). Judge the output against the observable's description (data shape, fields, or text the description promises). Record a ≤50-char evidence note in the registry entry. Present the evidence to the user for approval. Flip to `verified` only after the user approves. No evidence, no approval, no flip — no matter how obvious the tool's existence seems.
- `pending → verified` (spec-committed): only after the scenario that uses it passes end-to-end (triggered by supersdd).
- New entries start as `unverified` (runtime-provided) or `pending` (spec-committed). Never create with `verified`.

**For this spec:**
- For each scenario step this spec touches, map every observable desc to the linked observables that jointly satisfy it; record the mapping in the spec's "Observable Impact" section.
- Every linked observable this spec commits (spec-committed) must be added to the registry with status `pending`; the spec describes its trigger + action + outcome and names the runtime surface.
- Every linked observable this spec consumes (runtime-provided) must already exist in the registry as `verified`. Missing → append as a stub with status `unverified`. Unverified → exercise the runtime through any reachable channel (including the entry's own `how to observe` method, which an `unverified` status does not disqualify), judge the output against the observable's description, record a ≤50-char evidence note in the registry, then present the evidence to the user for approval; flip to `verified` only after the user approves.
- Every observable desc a scenario step introduces must map to at least one linked observable that physically exists in `docs/longspec/observables.md` (not just promised in text).

### Sub-step 3.8 — Observable Impact section

Append to spec's end a short "Observable Impact" section with two lists:
1. **Linked observables introduced** — names this spec commits (registry status `pending` until implemented).
2. **Desc-to-linked mapping** — for each scenario step this spec touches, list which observable descs (by id like `2a`) map to which linked observables. This is the bridge between user acceptance (desc) and implementation (linked).

### Sub-step 3.9 — Back-propagate linked observables into story.md

After the spec and registry are committed, update `story.md` so each observable desc touched by this spec carries the name(s) of its linked observable(s) inline (e.g. `2a. on the jobs page... [linked: ops-console.jobs-list]`). Then verify coverage:

- Every desc this spec touches has ≥1 linked observable attached; an orphan desc is a coverage gap and blocks Gate 3.
- Every spec-committed linked observable introduced in 3.8 appears under at least one desc; an unused linked observable is dead commitment and must be dropped from the registry, or the spec must justify why it exists without a desc consumer.
- Present the story.md diff for user approval before writing; the user signs off on the desc ↔ linked mapping in the context where acceptance will eventually be checked.

**Single source of truth after back-propagation** — once 3.9 completes, each desc's inline `[linked: ...]` in story.md is the authoritative view of the desc↔linked association. The spec's "Desc-to-linked mapping" written in 3.8 stays in the spec as the authoring artifact (proof the author did the mapping work). A separate mapping section inside `observables.md` is a third copy and is the anti-pattern — the registry stays a pure entry list.

### Gate 3 (Phase 3 → superplan)

Spec approved + all linked observables registered + desc-to-linked mapping complete + story.md back-propagation approved (every desc touched has ≥1 linked, every introduced linked has ≥1 desc consumer).

### Sub-step 3.10 — Transition

Invoke superplan to create the implementation plan. Do not invoke any other skill.

## Visual Companion

A browser-based companion for showing mockups, diagrams, and visual options during brainstorming. Available as a tool — not a mode. Accepting the companion means it's available for questions that benefit from visual treatment; it does NOT mean every question goes through the browser.

**Offering the companion:** When you anticipate that upcoming questions will involve visual content (mockups, layouts, diagrams), offer it once for consent:
> "Some of what we're working on might be easier to explain if I can show it to you in a web browser. I can put together mockups, diagrams, comparisons, and other visuals as we go. This feature is still new and can be token-intensive. Want to try it? (Requires opening a local URL)"

**This offer MUST be its own message.** Do not combine it with clarifying questions, context summaries, or any other content. The message should contain ONLY the offer above and nothing else. Wait for the user's response before continuing. If they decline, proceed with text-only brainstorming.

**Per-question decision:** Even after the user accepts, decide FOR EACH QUESTION whether to use the browser or the terminal. The test: **would the user understand this better by seeing it than reading it?**

- **Use the browser** for content that IS visual — mockups, wireframes, layout comparisons, architecture diagrams, side-by-side visual designs
- **Use the terminal** for content that is text — requirements questions, conceptual choices, tradeoff lists, A/B/C/D text options, scope decisions

A question about a UI topic is not automatically a visual question. "What does personality mean in this context?" is a conceptual question — use the terminal. "Which wizard layout works better?" is a visual question — use the browser.
