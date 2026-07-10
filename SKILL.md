---
name: hyperplan
description: "Adversarial multi-agent planning skill. Self-orchestrates 5 hostile agents (skeptic, validator, researcher, architect, creative) via parallel Agent-tool dispatch for ruthless cross-critique debate (3 rounds). Agents exchange findings through a filesystem message bus (<cache_dir>/phase_N/<member>.md) — the Lead never forwards bundles. Each round is a FRESH agent dispatch (no inter-round memory); all context flows through the cache. After 3 rounds the Lead dispatches ONE Plan agent (round-4) to distill survivors and formalize an executable plan. After user confirmation, the Lead dispatches parallel agents in dependency-ordered waves to execute the plan's tasks. Use when planning needs maximum rigor and surfacing of weak assumptions, blind spots, and over-engineering. Triggers: 'hyperplan', 'hpp', '/hyperplan', 'adversarial plan', 'hostile planning', 'cross-critique plan'."
---

# HYPERPLAN — Adversarial Multi-Agent Planning

> **MANDATORY**: First action when this skill loads — say "HYPERPLAN MODE ENABLED!" so the user knows orchestration started.

## WHAT THIS IS

You (the orchestrator) become the **Lead** of a 5-member adversarial team. The 5 members are **maximally hostile** to each other — they attack each other's findings ruthlessly. Members exchange their outputs through a **filesystem message bus** (`<cache_dir>/phase_N/<member>.md`): each member writes its own output file, and reads the other members' files directly. Every round is a **FRESH agent dispatch** — agents have no memory between rounds. All context flows through the cache. The Lead only dispatches, waits for completion, then dispatches ONE Plan agent (round-4) that reads the cache, distills survivors, and formalizes an executable plan. After the user confirms the plan, the Lead dispatches parallel agents in dependency-ordered waves to execute it.

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

Each member is dispatched via the `Agent` tool with the matching `subagent_type`. Their full adversarial system prompts live in `references/<member>.md` — at dispatch time, the orchestrator instructs each agent to read that file and adopt it as the system prompt for the task. Member system prompts include tool restrictions (Read/Write/Glob/Grep only) to prevent agents from spawning sub-agents, asking user questions, or entering plan mode.

| Member | subagent_type | Role | System prompt file |
|--------|---------------|------|--------------------|
| skeptic | `skeptic` | The Pragmatist Skeptic — attacks over-engineering, scope creep, premature abstraction | `references/skeptic.md` |
| validator | `validator` | The Integration Tester — attacks missed edge cases, broken interactions, blast radius | `references/validator.md` |
| researcher | `researcher` | The Autonomous Researcher — attacks unfounded claims, demands evidence | `references/researcher.md` |
| architect | `architect` | The Architect Strategist — attacks bad architecture, leaky abstractions, hidden coupling | `references/architect.md` |
| creative | `creative` | The Creative Challenger — attacks orthodox thinking, generates lateral alternatives | `references/creative.md` |

If `researcher` is unavailable, retry once without it and state the degraded roster. Do not drop `skeptic`, `validator`, `architect`, or `creative`.

## FILESYSTEM MESSAGE BUS (`<cache_dir>`)

Members do NOT send messages to each other through the Lead. Instead, every member writes its round output to a fixed path under `<cache_dir>`, and members read each other's files directly in the next round. Since each round is a fresh agent with no memory, the cache IS the memory — without it, later rounds have nothing to work with.

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
- Directories are pre-created in Phase 0 (the Write tool may not auto-create parent directories).
- The Lead NEVER reads `<cache_dir>/phase_1..3/*.md` — only the round-4 Plan agent reads them. Phases 1–3 never put member content into the Lead's context.

## PROMPT TEMPLATES (in references/)

All member-facing task prompts and the round-4 plan-formalization prompt live in `references/` as templates with `{{PLACEHOLDER}}` markers. The orchestrator Reads the template, substitutes the placeholder, and dispatches the result. SKILL.md never inlines these.

| Template file | Used in | Placeholders |
|---------------|---------|--------------|
| `references/round-1-task.md` | Phase 1 (Round 1 dispatch) | `{{USER_REQUEST}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-2-task.md` | Phase 2 (Round 2 dispatch) | `{{INPUT_DIR}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-3-task.md` | Phase 3 (Round 3 dispatch) | `{{OWN_FINDINGS_PATH}}`, `{{INPUT_DIR}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-4-task.md` | Phase 4 (Plan agent dispatch) | `{{CACHE_DIR}}`, `{{USER_REQUEST}}` |

## EXECUTION WORKFLOW

You execute this in **7 phases** (0–6). Phase 5 is conditional on user confirmation in Phase 4. The Agent tool runs dispatched agents concurrently when you issue multiple Agent calls in a single message — use this for every parallel round and every execution wave.

**Critical separation**: You (the Lead) do NOT distill insights and do NOT write the plan. The round-4 Plan agent owns distillation + formalization in a single dispatch. You dispatch it, persist its output, present it to the user, gate execution on confirmation, then dispatch parallel execution waves. You never inline-plan.

### Tooling map (how each phase talks to agents)

- **Dispatch members (any round)**: `Agent` tool with `subagent_type: "<member>"`, `run_in_background: false`, `description: "<member> round-N <role>"`. Each round is a **FRESH dispatch** — a new agent with no memory of prior rounds. The agent reads the cache files it needs, writes its output, and returns. You do NOT resume agents between rounds. You do NOT track agent IDs across rounds.
- **Round-4 dispatch (distillation + plan)**: `Agent` tool with `subagent_type: "Plan"`, `run_in_background: false`, `description: "Distill formalize hyperplan plan"`. The Plan agent reads the 15 cache files itself, distills, and returns the executable plan as its output. You persist it to `<cache_dir>/phase_4/plan.md`.
- **Execution dispatch (Phase 5)**: `Agent` tool with `subagent_type: "<task's agent type>"` (default `general-purpose`), `run_in_background: false`, `description: "Execute: <task title>"`, one call per task in a wave, all in a single message for parallelism.
- **Stuck agent cleanup**: `TaskStop` with the agent's task id — use only if an agent hangs or errors and is still running.

### Phase 0: Acknowledge and capture the request

1. Say "HYPERPLAN MODE ENABLED!" exactly once.
2. Restate the user's planning request in 1 sentence so all members start with the same scope.
3. Create your todo list for the 7 phases (Phase 4 plan-agent dispatch is mandatory; Phase 5 execution is conditional on user confirmation — include it as pending-confirmation).
4. Resolve three absolute paths (the ONLY place these are configured — everything else references them symbolically):
   - `<skill_dir>` — the directory this SKILL.md lives in. All `references/` paths derive from it.
   - `<workspace>` — the current working directory of the session (the project being planned).
   - `<cache_dir>` — the debate cache directory. Defaults to `<workspace>/.cache`. If the user specifies a different location (or the environment prefers one), override here. All phase paths and template substitutions use `<cache_dir>`.
5. Pre-create the cache directory structure. Use Shell with `python -c` to create `phase_1` through `phase_5` subdirectories under `<cache_dir>/`:
   ```
   python -c "import os; [os.makedirs(os.path.join(r'<cache_dir>', f'phase_{i}'), exist_ok=True) for i in range(1,6)]"
   ```
   (The Write tool may not auto-create parent directories.)

### Phase 1: Round 1 — Independent analysis (dispatch + write)

1. Read `references/round-1-task.md`. For each member, substitute:
   - `{{USER_REQUEST}}` → the user's planning request verbatim
   - `{{MEMBER_NAME}}` → the member's name (e.g. `skeptic`)
   - `{{OUTPUT_PATH}}` → `<cache_dir>/phase_1/<member>.md`

   The result is that member's **Round 1 task body**.
2. Dispatch all 5 members in **parallel** — issue 5 `Agent` tool calls in a **single message** so they run concurrently. Each call uses `subagent_type: "<member>"`, `description: "<member> round-1 analysis"`, `run_in_background: false`. Each member's prompt is:

   ```
   Read the file at <skill_dir>/references/<member>.md and adopt it as your system prompt for this task.

   [that member's Round 1 task body]
   ```

3. All 5 agents return when complete. Their outputs are on disk in `<cache_dir>/phase_1/`. Do NOT read those files yourself — you do not need them.
4. **Verify all 5 output files exist** using Glob (`<cache_dir>/phase_1/*.md`). If fewer than 5 files, a member agent failed — report to the user and ask whether to retry the failed member or proceed with a degraded roster. Do NOT dispatch Round 2 until all 5 files are present (or the user approves a degraded roster).

### Phase 2: Round 2 — Cross-attack (read phase_1, write phase_2)

1. Read `references/round-2-task.md`. For each member, substitute:
   - `{{INPUT_DIR}}` → `<cache_dir>/phase_1`
   - `{{MEMBER_NAME}}` → the member's name
   - `{{OUTPUT_PATH}}` → `<cache_dir>/phase_2/<member>.md`

   The result is that member's **Round 2 task body**.
2. Dispatch all 5 members as fresh agents in **parallel** — issue 5 `Agent` tool calls in a single message. Each call uses `subagent_type: "<member>"`, `description: "<member> round-2 cross-attack"`. Each member's prompt is:

   ```
   Read the file at <skill_dir>/references/<member>.md and adopt it as your system prompt for this task.

   [that member's Round 2 task body]
   ```

   Each fresh agent reads all 5 `phase_1/*.md` files (including its own, for reference), attacks the other 4 members' findings, and writes to `<cache_dir>/phase_2/<member>.md`.

3. All 5 agents return when complete. Their outputs are on disk in `<cache_dir>/phase_2/`.
4. **Verify all 5 output files exist** using Glob (`<cache_dir>/phase_2/*.md`). If fewer than 5, a member agent failed — report to the user and ask whether to retry or proceed degraded.

### Phase 3: Round 3 — Defense and refinement (read phase_2 + own phase_1, write phase_3)

1. Read `references/round-3-task.md`. For each member, substitute:
   - `{{OWN_FINDINGS_PATH}}` → `<cache_dir>/phase_1/<member>.md` (the member's own Round 1 findings — needed because the agent is fresh with no memory)
   - `{{INPUT_DIR}}` → `<cache_dir>/phase_2`
   - `{{MEMBER_NAME}}` → the member's name
   - `{{OUTPUT_PATH}}` → `<cache_dir>/phase_3/<member>.md`

   The result is that member's **Round 3 task body**.
2. Dispatch all 5 members as fresh agents in **parallel** — issue 5 `Agent` tool calls in a single message. Each call uses `subagent_type: "<member>"`, `description: "<member> round-3 defense"`. Each member's prompt is:

   ```
   Read the file at <skill_dir>/references/<member>.md and adopt it as your system prompt for this task.

   [that member's Round 3 task body]
   ```

   Each fresh agent reads its own `phase_1/<member>.md` (to recall its findings), reads all 5 `phase_2/*.md` files (to find attacks on itself), then defends/refines/concedes per finding and writes to `<cache_dir>/phase_3/<member>.md`.

3. All 5 agents return when complete. Their outputs are on disk in `<cache_dir>/phase_3/`.
4. **Verify all 5 output files exist** using Glob (`<cache_dir>/phase_3/*.md`). If fewer than 5, a member agent failed — report to the user and ask whether to retry or proceed degraded.

### Phase 4: Round 4 — Distillation + plan formalization (ONE Plan agent) + user confirmation

The team is done debating. You dispatch ONE Plan agent to read all 15 cache files, distill the survivors, and formalize an executable plan. You do NOT distill yourself — the Plan agent owns both jobs in a single dispatch.

1. Read `references/round-4-task.md`. Substitute:
   - `{{CACHE_DIR}}` → `<cache_dir>` (resolved in Phase 0)
   - `{{USER_REQUEST}}` → the user's planning request verbatim

   The result is the **Round 4 task body**.

2. Dispatch the Plan agent in the foreground:

   ```
   Agent({
     subagent_type: "Plan",
     run_in_background: false,
     description: "Distill formalize hyperplan plan",
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
   - **Modify** → take the user's requested changes and dispatch a FRESH Plan agent. Append to the Round 4 task body: "The user reviewed the previous plan at `<cache_dir>/phase_4/plan.md` and requested these modifications: [user feedback]. Read the previous plan and revise it. You do NOT need to re-read the 15 debate files unless the feedback requires fundamental rethinking — the distilled insights are already in the previous plan." Overwrite `<cache_dir>/phase_4/plan.md` with the revised plan. Re-present to user. Loop until the user says proceed or abort.
   - **Abort** → skip Phase 5, go directly to Phase 6 cleanup.

DO NOT enter Phase 5 without explicit user confirmation. DO NOT pre-distill or pre-draft the plan yourself — the Plan agent owns this. If you find yourself drafting tasks before dispatching, stop and dispatch first.

If the Plan agent returns clarifying questions instead of a plan, forward them to the user without modification — the planner is allowed to interview before committing. Re-dispatch a fresh Plan agent with the user's answers appended.

### Phase 5: Parallel execution (dependency-ordered waves)

The user has confirmed the plan. You now execute it by dispatching parallel waves of fresh agents. This phase is SKIPPED if the user aborted in Phase 4.

1. Parse the plan's "Execution Tasks" section. Each task has: ID/title, description, dependencies, success criteria, agent type, files likely touched.
2. Parse the "Wave Structure" section to determine execution order. If the plan did not provide explicit waves, compute them yourself by topological sort on the dependency graph (Wave 1 = no dependencies; Wave N = all dependencies satisfied by Wave N-1).
3. For each wave, in order:
   a. Dispatch all tasks in the wave **IN PARALLEL** — one `Agent` call per task, all in a **single message** so they run concurrently. Each call uses `subagent_type: "<task's agent type>"` (default `general-purpose`), `description: "Execute: <task title>"`, `run_in_background: false`.
   b. Each task agent's prompt must include:
      - The task's full description and success criteria from the plan.
      - Any hard constraints from the plan's "Distilled Insights" section that apply to this task.
      - Instruction to write a brief completion summary to `<cache_dir>/phase_5/<task-id>.md` (what was done, what files changed, whether success criteria were met).
      - Tool restriction: "Do NOT spawn sub-agents (no Agent tool). Do NOT use SendMessage or AskUserQuestion. Do NOT enter plan mode. Do the task directly with Read/Write/Edit/Shell/Glob/Grep."
   c. After the wave completes, Read the completion summaries under `<cache_dir>/phase_5/` to verify success criteria were met. If a task failed, surface the failure to the user before dispatching the next wave — ask whether to retry, skip, or abort.
4. After the final wave completes, tell the user: "Hyperplan execution complete. [N] tasks executed across [M] waves." Summarize what changed in the workspace.
5. Proceed to Phase 6.

Execution agents modify the workspace directly (files, code, config) per their task — they are doing real work, not writing reports. The completion summaries under `<cache_dir>/phase_5/` are the audit trail.

### Phase 6: Cleanup

1. All agents should have completed (they were foreground dispatches). If any is still running (hung or errored), call `TaskStop` with its task id.
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
| **Dispatching a wave before the previous wave completes** | **Breaks dependency ordering. Tasks in Wave N may depend on outputs from Wave N-1. Always wait for the full wave to return before starting the next.** |
| **Inlining member system prompts or task templates instead of reading `references/*.md`** | **Prompts drift from the canonical files. Always Read the relevant `references/` file, substitute its `{{PLACEHOLDER}}`, and dispatch the result. SKILL.md is the workflow skeleton, not the prompt source.** |
| **Lead forwarding bundles between members instead of using the `<cache_dir>` message bus** | **Defeats the filesystem-bus design. Members read each other's files directly from `<cache_dir>/phase_N/`. The Lead's context must stay clean — never paste member output into a dispatch prompt.** |
| **Lead reading `<cache_dir>/phase_1..3/` files** | **Unnecessary context pressure. The round-4 Plan agent reads those files — not the Lead. Phases 1–3 only need the completion signal (agents returned), not the content.** |
| **Relying on agent memory between rounds instead of the filesystem bus** | **Agents are fresh each round — they have zero memory of prior rounds. All context must flow through `<cache_dir>/phase_N/`. Round 3 agents must read their own `phase_1/<member>.md` to recall their findings; without it they have nothing to defend.** |
| **Dispatching Round N+1 without verifying Round N's output files exist** | **If a member agent failed silently, its output file is missing. The next round's agents will error trying to Read it. Always Glob-verify all 5 files before dispatching the next round.** |
| **Hardcoding `.cache` paths instead of referencing `<cache_dir>`** | **Scatters the cache location across files. `<cache_dir>` is resolved once in Phase 0; every other reference (SKILL.md + templates) uses the symbol. To relocate, change only Phase 0.** |
| Dispatching members sequentially instead of in parallel | Parallel dispatch is the throughput mechanism. Sequential rounds waste wall-clock and let one member's output bias another's. |
| Using `team_*` or `SendMessage`/`TaskOutput` for inter-round communication | Agents are fresh each round — there is nothing to resume. Use `Agent` (fresh dispatch per round) and `TaskStop` (cleanup only). |
| Running this from a background subagent | Hyperplan is main-session-only orchestration. |

## NOTES FOR THE LEAD (YOU)

- Issue all 5 dispatches for a round in a **single message** so they run concurrently. Sequential tool calls sequentialize the debate.
- **Every round is a FRESH `Agent` dispatch.** Agents do not persist between rounds. You do NOT track agent IDs across rounds. You do NOT use `SendMessage` or `TaskOutput` to resume agents. Each round: Read the template, substitute placeholders, dispatch 5 fresh `Agent` calls in one message, wait for all 5 to return, Glob-verify outputs.
- `<cache_dir>` is resolved once in Phase 0 and referenced symbolically everywhere after. The orchestrator computes `{{INPUT_DIR}}` / `{{OUTPUT_PATH}}` / `{{CACHE_DIR}}` / `{{OWN_FINDINGS_PATH}}` from `<cache_dir>` when substituting template placeholders — never hardcode `.cache` in a dispatch.
- The orchestrator NEVER reads `<cache_dir>/phase_1..3/` files. The round-4 Plan agent reads them. Phases 1–3 coordinate timing only — member content flows member-to-member through the filesystem, never through the Lead.
- Members read each other's files directly. The Round 2 template tells each fresh agent to Read all 5 `phase_1/*.md` files; the Round 3 template tells each fresh agent to Read its own `phase_1/<member>.md` (to recall its findings) AND all 5 `phase_2/*.md` files (to find attacks on itself). File formats in the templates are parse-critical — do not improvise them.
- The skill explicitly forbids you from softening adversarial prompts. The hostility IS the mechanism.
- The round-4 Plan agent does NOT have the Write tool. It returns the plan as its output; you persist it to `<cache_dir>/phase_4/plan.md` yourself. If the Plan agent needs more context, dispatch a fresh Plan agent with the additional context — do NOT try to resume the previous one.
- Phase 5 execution agents (default `general-purpose`) DO have Write/Edit/Shell. They modify the workspace directly per their task. Each writes a completion summary to `<cache_dir>/phase_5/<task-id>.md` for the audit trail. Their prompt includes tool restrictions — no sub-agents, no SendMessage, no AskUserQuestion, no plan mode.
- If a Phase 5 task fails its success criteria, surface it to the user before the next wave. Do not silently continue — a failed dependency can corrupt downstream waves.
