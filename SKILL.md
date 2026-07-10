---
name: hyperplan
description: "Adversarial multi-agent planning skill. Self-orchestrates 5 hostile agents (skeptic, validator, researcher, architect, creative) via parallel Agent-tool dispatch for ruthless cross-critique debate (3 rounds). Agents exchange findings through a filesystem message bus (<cache_dir>/phase_N/<member>.md) — the Lead never forwards bundles. After 3 rounds the Lead dispatches ONE Plan agent (round-4) to distill survivors and formalize an executable plan in a single shot. After user confirmation, the Lead dispatches parallel agents in dependency-ordered waves to execute the plan's tasks. Use when planning needs maximum rigor and surfacing of weak assumptions, blind spots, and over-engineering. Triggers: 'hyperplan', 'hpp', '/hyperplan', 'adversarial plan', 'hostile planning', 'cross-critique plan'."
---

# HYPERPLAN — Adversarial Multi-Agent Planning

> **MANDATORY**: First action when this skill loads — say "HYPERPLAN MODE ENABLED!" so the user knows orchestration started.

## WHAT THIS IS

You (the orchestrator) become the **Lead** of a 5-member adversarial team. The 5 members are **maximally hostile** to each other — they attack each other's findings ruthlessly. Members exchange their outputs through a **filesystem message bus** (`<cache_dir>/phase_N/<member>.md`): each member writes its own output file, and reads the other members' files directly. The Lead never forwards bundles — the Lead only dispatches, waits for completion, then dispatches ONE Plan agent (round-4) that reads the cache, distills survivors, and formalizes an executable plan. After the user confirms the plan, the Lead dispatches parallel agents in dependency-ordered waves to execute it.

This is not consensus building. This is intellectual combat. Weakness gets exposed. Lazy thinking gets eviscerated. Only what survives the gauntlet makes it into the plan.

## HARD PRECONDITIONS

Before starting, verify:

1. **The 5 adversarial subagent types are available** via the Agent tool's `subagent_type` parameter: `skeptic`, `validator`, `researcher`, `architect`, `creative`. If any is missing, STOP and tell the user which are unavailable.
2. **The `Plan` subagent type is available** for the Phase 4 distillation + plan-formalization dispatch.
3. **The `general-purpose` subagent type is available** for Phase 5 task execution.
4. **The `references/` folder exists alongside this SKILL.md** and contains all 9 prompt files:
   - 5 member system prompts: `skeptic.md`, `validator.md`, `researcher.md`, `architect.md`, `creative.md`
   - 4 task templates: `round-1-task.md`, `round-2-task.md`, `round-3-task.md`, `round-4-task.md`

   These files ARE the prompts — the orchestrator never inlines them. At runtime the orchestrator Reads each file, substitutes its `{{PLACEHOLDER}}`, and dispatches the result.
5. **You are in the main session** (not a background subagent). Hyperplan only works as a top-level orchestration.

## THE 5 ADVERSARIAL MEMBERS

Each member is dispatched via the `Agent` tool with the matching `subagent_type`. Their full adversarial system prompts live in `references/<member>.md` — at dispatch time, the orchestrator instructs each agent to read that file and adopt it as the system prompt for the task.

| Member | subagent_type | Role | System prompt file |
|--------|---------------|------|--------------------|
| skeptic | `skeptic` | The Pragmatist Skeptic — attacks over-engineering, scope creep, premature abstraction | `references/skeptic.md` |
| validator | `validator` | The Integration Tester — attacks missed edge cases, broken interactions, blast radius | `references/validator.md` |
| researcher | `researcher` | The Autonomous Researcher — attacks unfounded claims, demands evidence | `references/researcher.md` |
| architect | `architect` | The Architect Strategist — attacks bad architecture, leaky abstractions, hidden coupling | `references/architect.md` |
| creative | `creative` | The Creative Challenger — attacks orthodox thinking, generates lateral alternatives | `references/creative.md` |

If `researcher` is unavailable, retry once without it and state the degraded roster. Do not drop `skeptic`, `validator`, `architect`, or `creative`.

## FILESYSTEM MESSAGE BUS (`<cache_dir>`)

Members do NOT send messages to each other through the Lead. Instead, every member writes its round output to a fixed path under `<cache_dir>`, and members read each other's files directly in the next round.

```
<cache_dir>/
├── phase_1/          # Round 1 findings (each member writes <member>.md)
│   ├── skeptic.md
│   ├── validator.md
│   ├── researcher.md
│   ├── architect.md
│   └── creative.md
├── phase_2/          # Round 2 cross-attacks (each member writes <member>.md)
│   └── ...
├── phase_3/          # Round 3 defenses (each member writes <member>.md)
│   └── ...
├── phase_4/          # Round 4 distilled + formalized plan (Lead writes plan.md)
│   └── plan.md
└── phase_5/          # Execution completion summaries (one per task-id)
    └── <task-id>.md
```

- `<cache_dir>` is resolved ONCE in Phase 0 (default `<workspace>/.cache`). Every other reference in this skill and in the templates uses the symbolic `<cache_dir>` — to relocate the cache, change only the Phase 0 definition.
- `<cache_dir>/` is gitignored runtime state. It is left in place after the run for inspection; the user may delete it.
- The Write tool auto-creates these directories on first write — no need to mkdir.
- The Lead reads `<cache_dir>/phase_1..3/*.md` ONLY inside the round-4 Plan agent's dispatch prompt path — the Plan agent reads them, not the Lead. Phases 1–3 never put member content into the Lead's context.

## PROMPT TEMPLATES (in references/)

All member-facing task prompts and the round-4 plan-formalization prompt live in `references/` as templates with `{{PLACEHOLDER}}` markers. The orchestrator Reads the template, substitutes the placeholder, and dispatches the result. SKILL.md never inlines these.

| Template file | Used in | Placeholders |
|---------------|---------|--------------|
| `references/round-1-task.md` | Phase 1 (Round 1 dispatch) | `{{USER_REQUEST}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-2-task.md` | Phase 2 (Round 2 SendMessage) | `{{INPUT_DIR}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-3-task.md` | Phase 3 (Round 3 SendMessage) | `{{INPUT_DIR}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-4-task.md` | Phase 4 (Plan agent dispatch) | `{{CACHE_DIR}}`, `{{USER_REQUEST}}` |

## EXECUTION WORKFLOW

You execute this in **7 phases** (0–6). Phase 5 is conditional on user confirmation in Phase 4. The Agent tool runs dispatched agents concurrently when you issue multiple Agent calls in a single message — use this for every parallel round and every execution wave.

**Critical separation**: You (the Lead) do NOT distill insights and do NOT write the plan. The round-4 Plan agent owns distillation + formalization in a single dispatch. You dispatch it, persist its output, present it to the user, gate execution on confirmation, then dispatch parallel execution waves. You never inline-plan.

### Tooling map (how each phase talks to agents)

- **Dispatch a member (Round 1)**: `Agent` tool with `subagent_type: "<member>"`, `run_in_background: false`. The agent runs, completes, and returns. Capture the returned agent identifier (the `description` you passed, or the returned agentId) — you reuse it to resume the same agent in later rounds.
- **Resume a member for the next round**: `SendMessage` with `to: <member's identifier>` and the next round's task body. Completed agents resume in the background with full prior context.
- **Wait for a resumed member to finish writing**: `TaskOutput` with `block: true` against the member's task id. You do NOT need the returned content — the canonical output is the file the agent wrote under `<cache_dir>/`. TaskOutput is just the completion gate.
- **Round-4 dispatch (distillation + plan)**: `Agent` tool with `subagent_type: "Plan"`, `run_in_background: false`. The Plan agent reads the 15 cache files itself, distills, and returns the executable plan as its output. You persist it to `<cache_dir>/phase_4/plan.md`.
- **Execution dispatch (Phase 5)**: `Agent` tool with `subagent_type: "<task's agent type>"` (default `general-purpose`), `run_in_background: false`, one call per task in a wave, all in a single message for parallelism.

### Phase 0: Acknowledge and capture the request

1. Say "HYPERPLAN MODE ENABLED!" exactly once.
2. Restate the user's planning request in 1 sentence so all members start with the same scope.
3. Create your todo list for the 7 phases (Phase 4 plan-agent dispatch is mandatory; Phase 5 execution is conditional on user confirmation — include it as pending-confirmation).
4. Resolve three absolute paths (the ONLY place these are configured — everything else references them symbolically):
   - `<skill_dir>` — the directory this SKILL.md lives in. All `references/` paths derive from it.
   - `<workspace>` — the current working directory of the session (the project being planned).
   - `<cache_dir>` — the debate cache directory. Defaults to `<workspace>/.cache`. If the user specifies a different location (or the environment prefers one), override here. All phase paths and template substitutions use `<cache_dir>`.

### Phase 1: Round 1 — Independent analysis (dispatch + write)

1. Read `references/round-1-task.md`. For each member, substitute:
   - `{{USER_REQUEST}}` → the user's planning request verbatim
   - `{{MEMBER_NAME}}` → the member's name (e.g. `skeptic`)
   - `{{OUTPUT_PATH}}` → `<cache_dir>/phase_1/<member>.md`

   The result is that member's **Round 1 task body**.
2. Dispatch all 5 members in **parallel** — issue 5 `Agent` tool calls in a **single message** so they run concurrently. Each member's prompt is:

   ```
   Read the file at <skill_dir>/references/<member>.md and adopt it as your system prompt for this task.

   [that member's Round 1 task body]
   ```

   Use the matching `subagent_type` on each `Agent` call. Use `run_in_background: false` so all 5 return to you directly when done.

3. Capture each member's returned identifier (description / agentId) — you need it to resume them in Round 2. Do NOT read the `<cache_dir>/phase_1/` files yourself; you do not need them yet.

### Phase 2: Round 2 — Cross-attack (read phase_1, write phase_2)

1. Read `references/round-2-task.md`. For each member, substitute:
   - `{{INPUT_DIR}}` → `<cache_dir>/phase_1`
   - `{{MEMBER_NAME}}` → the member's name
   - `{{OUTPUT_PATH}}` → `<cache_dir>/phase_2/<member>.md`

   The result is that member's **Round 2 task body** (identical instruction, different output path per member).
2. Resume all 5 members in **parallel** — issue 5 `SendMessage` calls in a single message, one to each member's identifier, each with that member's Round 2 task body.
3. Issue 5 `TaskOutput` calls in a single message (one per member, `block: true`) to wait for all 5 to finish writing. The returned content is just a completion signal — the canonical output is on disk in `<cache_dir>/phase_2/`.

### Phase 3: Round 3 — Defense and refinement (read phase_2, write phase_3)

1. Read `references/round-3-task.md`. For each member, substitute:
   - `{{INPUT_DIR}}` → `<cache_dir>/phase_2`
   - `{{MEMBER_NAME}}` → the member's name
   - `{{OUTPUT_PATH}}` → `<cache_dir>/phase_3/<member>.md`

   The result is that member's **Round 3 task body**.
2. Resume all 5 members in parallel — 5 `SendMessage` calls in a single message, each with its member's Round 3 task body.
3. Issue 5 `TaskOutput` calls in a single message to wait for all 5 to finish writing.

### Phase 4: Round 4 — Distillation + plan formalization (ONE Plan agent) + user confirmation

The team is done debating. You dispatch ONE Plan agent to read all 15 cache files, distill the survivors, and formalize an executable plan. You do NOT distill yourself — the Plan agent owns both jobs in a single dispatch.

1. Read `references/round-4-task.md`. Substitute:
   - `{{CACHE_DIR}}` → `<cache_dir>` (resolved in Phase 0)
   - `{{USER_REQUEST}}` → the user's planning request verbatim

   The result is the **Round 4 task body**.

2. Dispatch the Plan agent in the foreground (you wait for the plan):

   ```
   Agent({
     subagent_type: "Plan",
     run_in_background: false,
     description: "Distill + formalize hyperplan plan",
     prompt: "[Round 4 task body]"
   })
   ```

   The Plan agent reads the 15 cache files itself, distills surviving insights (drops CONCEDED, keeps DEFEND/REFINE/uncontested), and returns the executable plan as its output.

3. **Persist the plan** — Write the Plan agent's returned output to `<cache_dir>/phase_4/plan.md`. This is the canonical plan artifact; Phase 5 execution agents may reference it.

4. **Present the plan to the user verbatim**, prefixed with one provenance line:

   ```
   *Plan derived from hyperplan adversarial review (5 members, 3 rounds), distilled and formalized by the Plan agent.*

   [plan content]
   ```

5. **User confirmation gate** — ask the user whether to proceed with execution. Three outcomes:
   - **Proceed** → advance to Phase 5 (parallel execution).
   - **Modify** → take the user's requested changes and re-dispatch the round-4 Plan agent via `SendMessage` to its task id with the feedback. When it returns the revised plan, goto step 3 with the new output. Loop until the user says proceed or abort.
   - **Abort** → skip Phase 5, go directly to Phase 6 cleanup.

DO NOT enter Phase 5 without explicit user confirmation. DO NOT pre-distill or pre-draft the plan yourself — the Plan agent owns this. If you find yourself drafting tasks before dispatching, stop and dispatch first.

If the Plan agent returns clarifying questions instead of a plan, forward them to the user without modification — the planner is allowed to interview before committing. Re-dispatch with the user's answers via `SendMessage`.

### Phase 5: Parallel execution (dependency-ordered waves)

The user has confirmed the plan. You now execute it by dispatching parallel waves of agents. This phase is SKIPPED if the user aborted in Phase 4.

1. Parse the plan's "Execution Tasks" section. Each task has: ID/title, description, dependencies, success criteria, agent type, files likely touched.
2. Parse the "Wave Structure" section to determine execution order. If the plan did not provide explicit waves, compute them yourself by topological sort on the dependency graph (Wave 1 = no dependencies; Wave N = all dependencies satisfied by Wave N-1).
3. For each wave, in order:
   a. Dispatch all tasks in the wave **IN PARALLEL** — one `Agent` call per task, all in a **single message** so they run concurrently. Use each task's specified `subagent_type` (default `general-purpose`). Use `run_in_background: false` so the wave's agents all return to you when done.
   b. Each task agent's prompt must include: the task description, the success criteria, the relevant context from the plan, and an instruction to write a brief completion summary to `<cache_dir>/phase_5/<task-id>.md` (what was done, what files changed, whether success criteria were met).
   c. After the wave completes, Read the completion summaries under `<cache_dir>/phase_5/` to verify success criteria were met. If a task failed, surface the failure to the user before dispatching the next wave — ask whether to retry, skip, or abort.
4. After the final wave completes, tell the user: "Hyperplan execution complete. [N] tasks executed across [M] waves." Summarize what changed in the workspace.
5. Proceed to Phase 6.

Execution agents modify the workspace directly (files, code, config) per their task — they are doing real work, not writing reports. The completion summaries under `<cache_dir>/phase_5/` are the audit trail.

### Phase 6: Cleanup

1. Stop any still-running agents — adversarial members from Rounds 1–3, the round-4 Plan agent, execution agents from Phase 5 — via `TaskStop` with their task ids.
2. Leave `<cache_dir>/` in place for the user to inspect the debate transcript (phases 1–3), the formalized plan (phase 4), and execution summaries (phase 5). Mention its location.
3. Confirm to the user with one line: "Hyperplan team disbanded."

If any step fails, surface the error and note that stray agents can be stopped via `TaskStop` with their task ids.

## ANTI-PATTERNS — DO NOT DO THESE

| Anti-pattern | Why it fails |
|--------------|--------------|
| Skipping rounds to "save time" | The adversarial filter is the entire value. Skipping rounds = vanilla planning. |
| Soft-pedaling member prompts ("be respectful") | Adversarial pressure is the mechanism. Politeness defeats the skill. |
| Synthesizing findings before Round 3 completes | Premature synthesis preserves weak findings. |
| **Lead distilling insights manually instead of dispatching the round-4 Plan agent** | **The round-4 Plan agent owns distillation + formalization. Lead-manual distillation skips the planner, pollutes the Lead's context with 15 files of debate, and turns this back into vanilla orchestration.** |
| **Pre-writing or pre-drafting the plan before dispatching the Plan agent** | **Anchors the planner to your draft and undermines its independent judgment. Dispatch raw — let the Plan agent read the cache and structure.** |
| **Entering Phase 5 execution without explicit user confirmation** | **The confirmation gate is mandatory. Executing before confirmation means acting on an unapproved plan — potentially modifying the workspace against the user's intent.** |
| **Dispatching execution tasks sequentially instead of in parallel waves** | **Parallel waves are the throughput mechanism. Sequential execution wastes wall-clock and defeats the wave structure the Plan agent designed.** |
| **Dispatching a wave before the previous wave completes** | **Breaks dependency ordering. Tasks in Wave N may depend on outputs from Wave N-1. Always `TaskOutput`/wait for the full wave before starting the next.** |
| **Inlining member system prompts or task templates instead of reading `references/*.md`** | **Prompts drift from the canonical files. Always Read the relevant `references/` file, substitute its `{{PLACEHOLDER}}`, and dispatch the result. SKILL.md is the workflow skeleton, not the prompt source.** |
| **Lead forwarding bundles between members instead of using the `<cache_dir>` message bus** | **Defeats the filesystem-bus design. Members read each other's files directly from `<cache_dir>/phase_N/`. The Lead's context must stay clean — never paste member output into a `SendMessage`.** |
| **Lead reading `<cache_dir>/phase_1..3/` files before Phase 4** | **Unnecessary context pressure. Phases 1–3 only need the completion signal from `TaskOutput`, not the content. The round-4 Plan agent reads those files — not the Lead.** |
| **Hardcoding `.cache` paths instead of referencing `<cache_dir>`** | **Scatters the cache location across files. `<cache_dir>` is resolved once in Phase 0; every other reference (SKILL.md + templates) uses the symbol. To relocate, change only Phase 0.** |
| Dispatching members sequentially instead of in parallel | Parallel dispatch is the throughput mechanism. Sequential rounds waste wall-clock and let one member's output bias another's. |
| Dispatching a fresh agent each round instead of resuming via `SendMessage` | Loses member continuity. Round 3 defense needs the member's memory of WHY it made each Round 1 finding. |
| Using `team_*` tools (they do not exist in this environment) | Use `Agent` / `SendMessage` / `TaskOutput` / `TaskStop` only. |
| Running this from a background subagent | Hyperplan is main-session-only orchestration. |

## NOTES FOR THE LEAD (YOU)

- Issue all 5 dispatches / messages / output-fetches for a round in a **single message** so they run concurrently. Sequential tool calls sequentialize the debate.
- After Round 1 (foreground dispatch), members complete and return. To continue them in Round 2 / Round 3, use `SendMessage` (resumes the completed agent in the background with full context), then `TaskOutput` with `block: true` as the completion gate.
- Track each member's identifier (the `description` you passed to `Agent`, or the returned agentId) from Phase 1 — you need it for every `SendMessage` and `TaskOutput` call afterward.
- Members read their `references/<member>.md` system prompt ONCE in Round 1. Do NOT re-instruct them to read it in Round 2 / Round 3 — they resume with full context via `SendMessage`.
- The orchestrator (you) Reads `references/round-*-task.md` fresh each phase, substitutes the placeholders, and dispatches the result. Never inline the template text.
- `<cache_dir>` is resolved once in Phase 0 and referenced symbolically everywhere after. The orchestrator computes `{{INPUT_DIR}}` / `{{OUTPUT_PATH}}` / `{{CACHE_DIR}}` from `<cache_dir>` when substituting template placeholders — never hardcode `.cache` in a dispatch.
- The orchestrator NEVER reads `<cache_dir>/phase_1..3/` files. The round-4 Plan agent reads them. Phases 1–3 coordinate timing only — member content flows member-to-member through the filesystem, never through the Lead.
- Members read each other's files directly. The Round 2 template tells each member to Read all 5 `phase_1/*.md` files; the Round 3 template tells each member to Read all 5 `phase_2/*.md` files and locate the `## Attacks on <their-name>` sections. File formats in the templates are parse-critical — do not improvise them.
- The skill explicitly forbids you from softening adversarial prompts. The hostility IS the mechanism.
- The Phase 4 round-4 dispatch runs **synchronously** (`run_in_background: false`) — you wait for the plan before presenting it.
- The round-4 Plan agent does NOT have the Write tool. It returns the plan as its output; you persist it to `<cache_dir>/phase_4/plan.md` yourself. If the Plan agent needs more context, re-dispatch via `SendMessage` to its task id — do NOT spin up a new Plan agent.
- Phase 5 execution agents (default `general-purpose`) DO have Write/Edit/Shell. They modify the workspace directly per their task. Each writes a completion summary to `<cache_dir>/phase_5/<task-id>.md` for the audit trail.
- If a Phase 5 task fails its success criteria, surface it to the user before the next wave. Do not silently continue — a failed dependency can corrupt downstream waves.
