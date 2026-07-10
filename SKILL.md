---
name: hyperplan
description: "Adversarial multi-agent planning skill. Self-orchestrates hostile agents (skeptic, validator, researcher, architect, creative) via parallel Agent-tool dispatch for ruthless cross-critique debate (3 rounds). Agents exchange findings through a filesystem message bus (<cache_path>/phase_N/<member>.md) — the Lead never forwards bundles. Each round is a FRESH agent dispatch (no inter-round memory); all context flows through the cache. After 3 rounds the Lead dispatches ONE planner (round-4) to distill survivors and formalize an executable plan. After user confirmation, the Lead dispatches parallel agents in dependency-ordered waves to execute the plan's tasks. Use when planning needs maximum rigor and surfacing of weak assumptions, blind spots, and over-engineering. Triggers: 'hyperplan', 'hpp', '/hyperplan', 'adversarial plan', 'hostile planning', 'cross-critique plan'."
---

# HYPERPLAN — Adversarial Multi-Agent Planning

> **MANDATORY**: First action when this skill loads — say "HYPERPLAN MODE ENABLED!" so the user knows orchestration started.

## WHAT THIS IS

You (the orchestrator) become the **Lead** of an adversarial team. The members are **maximally hostile** to each other — they attack each other's findings ruthlessly. Members exchange their outputs through a **filesystem message bus** (`<cache_path>/phase_N/<member>.md`): each member writes its own output file, and reads the other members' files directly. Every round is a **FRESH agent dispatch** — agents have no memory between rounds. All context flows through the cache. The Lead only dispatches, waits for completion, then dispatches ONE planner (round-4) that reads the cache, distills survivors, and formalizes an executable plan. After the user confirms the plan, the Lead dispatches parallel agents in dependency-ordered waves to execute it.

This is not consensus building. This is intellectual combat. Weakness gets exposed. Lazy thinking gets eviscerated. Only what survives the gauntlet makes it into the plan.

## HARD PRECONDITIONS

Before starting, verify:

1. **The `general-purpose` subagent type is available** via the Agent tool's `subagent_type` parameter. Every dispatch in this skill uses it; the role (skeptic/validator/researcher/architect/creative/planner) comes from the `references/*.md` file the agent reads at dispatch time.
2. **The `references/` folder exists alongside this SKILL.md** and contains all prompt files:
   - Persona prompts: `skeptic.md`, `validator.md`, `researcher.md`, `architect.md`, `creative.md` (one per debate member — add or remove a file to change the roster), `planner.md` (round-4 distiller)
   - Task templates: `round-1-task.md`, `round-2-task.md`, `round-3-task.md`, `round-4-task.md`

   These files ARE the prompts — the orchestrator never inlines them. At runtime the orchestrator dispatches each agent with the path to its `references/<member>.md` system prompt + the path to `<cache_path>/VARIABLES.md`; the agent reads VARIABLES.md itself, uses the `ROUND` value to load `references/round-{ROUND}-task.md`, and interprets the {{VAR}} markers with the values from VARIABLES.md + its MEMBER_NAME. The member roster is whatever set of `references/<member>.md` files exists — the skill is count-agnostic.
3. **You are in the main session** (not a background subagent). Hyperplan only works as a top-level orchestration.

## THE ADVERSARIAL MEMBERS

Every member is dispatched via the `Agent` tool with `subagent_type: "general-purpose"`. `references/<member>.md` IS the persona. Member system prompts include negative tool restrictions (no Agent, no SendMessage, no AskUserQuestion, no plan mode, no task management tools, or any tool requiring user approval or reply) to keep agents scoped to read/analyze/write/return.

| Member | Role | System prompt file |
|--------|------|--------------------|
| skeptic | The Pragmatist Skeptic — attacks over-engineering, scope creep, premature abstraction | `references/skeptic.md` |
| validator | The Integration Tester — attacks missed edge cases, broken interactions, blast radius | `references/validator.md` |
| researcher | The Autonomous Researcher — attacks unfounded claims, demands evidence | `references/researcher.md` |
| architect | The Architect Strategist — attacks bad architecture, leaky abstractions, hidden coupling | `references/architect.md` |
| creative | The Creative Challenger — attacks orthodox thinking, generates lateral alternatives | `references/creative.md` |
| planner | The Planner — stands above the fray, distills survivors, formalizes the executable plan (round-4 only) | `references/planner.md` |

All debate members (skeptic/validator/researcher/architect/creative) are always dispatched in rounds 1–3. The planner is dispatched once in round 4. If a member agent fails, retry that member once; if it still fails, ask the user whether to proceed with a degraded roster.

## FILESYSTEM MESSAGE BUS (`<cache_path>`)

Members do NOT send messages to each other through the Lead. Instead, every member writes its round output to a fixed path under `<cache_path>`, and members read each other's files directly in the next round. Since each round is a fresh agent with no memory, the cache IS the memory — without it, later rounds have nothing to work with.

```
<cache_path>/
├── VARIABLES.md     # Shared variables (Lead creates in Phase 0, updates ROUND before each phase)
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
├── phase_4/          # Round 4 distilled + formalized plan (planner writes plan.md)
│   └── plan.md
└── phase_5/          # Execution completion summaries (one per task-id)
    └── <task-id>.md
```

- `<cache_path>` is resolved ONCE in Phase 0 (default `<workspace>/.cache`). Every other reference in this skill and in the templates uses the symbolic `<cache_path>` — to relocate the cache, change only the Phase 0 definition.
- `<cache_path>/` is gitignored runtime state. It is left in place after the run for inspection; the user may delete it.
- Directories are pre-created in Phase 0 (the Write tool may not auto-create parent directories).
- The Lead NEVER reads `<cache_path>/phase_1..3/*.md` — only the round-4 planner reads them. Phases 1–3 never put member content into the Lead's context.

## EXECUTION WORKFLOW

You execute this in **7 phases** (0–6). Phase 5 is conditional on user confirmation in Phase 4. The Agent tool runs dispatched agents concurrently when you issue multiple Agent calls in a single message — use this for every parallel round and every execution wave.

**Critical separation**: You (the Lead) do NOT distill insights and do NOT write the plan. The Round 4 planner owns distillation + formalization in a single dispatch — it writes the plan to `<cache_path>/phase_4/plan.md` itself. You Read that file, surface its decision points to the user (not the whole plan verbatim), gate execution on confirmation, then dispatch parallel execution waves. You never inline-plan.

**Variable passing**: Every dispatch uses `subagent_type: "general-purpose"`. Each round is a **FRESH dispatch** — a new agent with no memory of prior rounds. You do NOT resume agents between rounds. You do NOT track agent IDs across rounds. Templates live in `references/` with `{{VAR}}` markers. The Lead does NOT read templates and does NOT substitute placeholders. Instead, the Lead maintains a single file `<cache_path>/VARIABLES.md` containing the shared variables; each dispatch prompt tells the agent to Read that file + its own system prompt + task template. The agent interprets the {{VAR}} markers with the values from VARIABLES.md + its MEMBER_NAME (passed in the dispatch prompt). This keeps the Lead's context free of all template/member content (the Lead only ever reads `<cache_path>/phase_4/plan.md` to surface its decision points).

**VARIABLES.md** (created in Phase 0, updated before each round — the Lead edits the `ROUND` line via Edit; the other 4 values stay untouched):

```
ROUND: <N>
USER_REQUEST: <user request verbatim>
SKILL_PATH: <skill_path>
WORKSPACE: <workspace>
CACHE_PATH: <cache_path>
```

`MEMBER_NAME` is NOT in VARIABLES.md — it is unique per agent and passed in the dispatch prompt. The Lead updates `ROUND` before each round (1 → 2 → 3 → 4); the other 4 values stay constant for the entire run.

| Variable | Source | Value |
|----------|--------|-------|
| `{{ROUND}}` | VARIABLES.md | The round number (1 / 2 / 3 / 4) — determines which `round-N-task.md` template the agent loads |
| `{{USER_REQUEST}}` | VARIABLES.md | The user's planning request, verbatim |
| `{{MEMBER_NAME}}` | dispatch prompt | The dispatched agent's name (skeptic / validator / researcher / architect / creative / planner) |
| `{{SKILL_PATH}}` | VARIABLES.md | `<skill_path>` — the directory this SKILL.md lives in |
| `{{WORKSPACE}}` | VARIABLES.md | `<workspace>` — the current working directory |
| `{{CACHE_PATH}}` | VARIABLES.md | `<cache_path>` — the debate cache directory |

Templates construct all phase paths from `{{CACHE_PATH}}` + `{{MEMBER_NAME}}` — there are no derived path variables.

**Unified dispatch shape** (used by Phase 1–4 — one call per agent; Phase 1–3 dispatch all members in parallel in a single message, Phase 4 dispatches ONE planner):

```
Agent({
  subagent_type: "general-purpose",
  description: "<member_name>",
  run_in_background: false,
  prompt: "You are <member_name>. Read <skill_path>/references/<member_name>.md as your system prompt. Read <cache_path>/VARIABLES.md for shared variables (ROUND, USER_REQUEST, SKILL_PATH, WORKSPACE, CACHE_PATH). Use the ROUND value to load <skill_path>/references/round-{ROUND}-task.md as your task template. Execute the template with the variable values from VARIABLES.md + your MEMBER_NAME (<member_name>). Do NOT modify the template."
})
```

The Lead does NOT read any `references/*.md` — the agent reads its own system prompt + VARIABLES.md + task template. The dispatch prompt only carries `<member_name>`, `<skill_path>`, `<cache_path>` (paths to locate files); all variable VALUES come from VARIABLES.md. To add a new phase: create `references/round-N-task.md`, add one dispatch line to SKILL.md, update VARIABLES.md's ROUND — no other changes needed.

### Phase 0: Acknowledge and capture the request

1. Say "HYPERPLAN MODE ENABLED!" exactly once.
2. Restate the user's planning request in 1 sentence so all members start with the same scope.
3. Create your todo list for the 7 phases (Phase 4 planner dispatch is mandatory; Phase 5 execution is conditional on user confirmation — include it as pending-confirmation).
4. Resolve three absolute paths (the ONLY place these are configured — everything else references them symbolically):
   - `<skill_path>` — the directory this SKILL.md lives in. All `references/` paths derive from it. Stored in VARIABLES.md as `SKILL_PATH`.
   - `<workspace>` — the current working directory of the session (the project being planned). Stored in VARIABLES.md as `WORKSPACE`.
   - `<cache_path>` — the debate cache directory. Defaults to `<workspace>/.cache`. If the user specifies a different location (or the environment prefers one), override here. Stored in VARIABLES.md as `CACHE_PATH`.
5. Pre-create the cache directory structure with `python -c` (creates `phase_1` through `phase_5` under `<cache_path>/`):
   ```
   python -c "import os; [os.makedirs(os.path.join(r'<cache_path>', f'phase_{i}'), exist_ok=True) for i in range(1,6)]"
   ```
   (The Write tool may not auto-create parent directories.)
6. Create `<cache_path>/VARIABLES.md` using the Write tool with the initial shared variables (every dispatch prompt points the agent to this file — the agent Reads it itself; the Lead never inlines variables into a dispatch):
   ```
   ROUND: 1
   USER_REQUEST: <user request verbatim>
   SKILL_PATH: <skill_path>
   WORKSPACE: <workspace>
   CACHE_PATH: <cache_path>
   ```
   Phase 1 uses `ROUND: 1` as set here. Before each subsequent phase the Lead edits the `ROUND` line to `2`, then `3`, then `4` (use Edit — do NOT rewrite the whole file; the other 4 values stay constant).

### Phase 1: Round 1 — Independent analysis (dispatch + write)

1. Dispatch all members in **parallel** using the Unified dispatch shape (VARIABLES.md already has `ROUND: 1` from Phase 0). The calls go in a **single message** so they run concurrently.
2. All agents return when complete. Their outputs are on disk in `<cache_path>/phase_1/`. Do NOT read those files yourself.
3. **Verify all member output files exist** in `<cache_path>/phase_1/` (enumeration command in NOTES). If any are missing, retry the failed member(s) once; if any still fail, ask the user whether to proceed with a degraded roster. Do NOT dispatch Round 2 until all member files are present (or the user approves a degraded roster).

### Phase 2: Round 2 — Cross-attack (read phase_1, write phase_2)

1. Update `<cache_path>/VARIABLES.md`: set `ROUND: 2` (use Edit to change only the `ROUND` line; leave the other 4 values untouched).
2. Dispatch all members in **parallel** using the Unified dispatch shape. Each fresh agent reads all `phase_1/*.md` files, attacks all other members' findings, and writes to `<cache_path>/phase_2/<member_name>.md`.
3. All agents return when complete. Their outputs are on disk in `<cache_path>/phase_2/`.
4. **Verify all member output files exist** in `<cache_path>/phase_2/` (enumeration command in NOTES). If any are missing, retry the failed member(s) once; if any still fail, ask the user whether to proceed with a degraded roster.

### Phase 3: Round 3 — Defense and refinement (read phase_2 + own phase_1, write phase_3)

1. Update `<cache_path>/VARIABLES.md`: set `ROUND: 3` (use Edit to change only the `ROUND` line; leave the other 4 values untouched).
2. Dispatch all members in **parallel** using the Unified dispatch shape. Each fresh agent reads its own `phase_1/<member_name>.md` AND all `phase_2/*.md` files, then defends/refines/concedes per finding and writes to `<cache_path>/phase_3/<member_name>.md`.
3. All agents return when complete. Their outputs are on disk in `<cache_path>/phase_3/`.
4. **Verify all member output files exist** in `<cache_path>/phase_3/` (enumeration command in NOTES). If any are missing, retry the failed member(s) once; if any still fail, ask the user whether to proceed with a degraded roster.

### Phase 4: Round 4 — Distillation + plan formalization + user confirmation

1. Update `<cache_path>/VARIABLES.md`: set `ROUND: 4` (use Edit to change only the `ROUND` line; leave the other 4 values untouched).
2. Dispatch the planner using the Unified dispatch shape (`MEMBER_NAME: planner`). The planner reads all cache files (phases 1–3), distills survivors, and writes the executable plan to `<cache_path>/phase_4/plan.md`.
3. **Read the plan** — Read `<cache_path>/phase_4/plan.md`. Do NOT dump it verbatim to the user. Identify the points that need user decision (the plan's "Open Questions" section + any unresolved trade-offs), and ask the user those questions via `AskUserQuestion`.
4. **Execution gate** — ask the user whether to proceed to Phase 5 parallel execution. **Proceed** → advance to Phase 5. **Abort** → skip Phase 5, go to Phase 6.

DO NOT enter Phase 5 without explicit user confirmation. DO NOT pre-distill or pre-draft the plan yourself — the planner owns this.

### Phase 5: Parallel execution (dependency-ordered waves)

**Execution dispatch shape** (N parallel calls per wave, one per task, all in a single message):

```
Agent({
  subagent_type: "general-purpose",
  description: "Execute: <task title>",
  run_in_background: false,
  prompt: "[task description + success criteria + hard constraints + completion summary instruction + tool restrictions]"
})
```

The user has confirmed the plan. You now execute it by dispatching parallel waves of fresh agents. This phase is SKIPPED if the user aborted in Phase 4.

1. Parse the plan's "Execution Tasks" section. Each task has: ID/title, description, dependencies, success criteria, files likely touched.
2. Parse the "Wave Structure" section to determine execution order. If the plan did not provide explicit waves, compute them yourself by topological sort on the dependency graph (Wave 1 = no dependencies; Wave N = all dependencies satisfied by Wave N-1).
3. For each wave, in order:
   a. Dispatch all tasks in the wave **IN PARALLEL** using the Execution dispatch shape above, with `description: "Execute: <task title>"`. Each task agent's prompt must include:
      - The task's full description and success criteria from the plan.
      - Any hard constraints from the plan's "Distilled Insights" section that apply to this task.
      - Instruction to write a brief completion summary to `<cache_path>/phase_5/<task-id>.md` (what was done, what files changed, whether success criteria were met).
      - Tool restriction: "Do NOT spawn sub-agents (no Agent tool). Do NOT use SendMessage, AskUserQuestion, NotifyUser, EnterPlanMode, task management tools, or any tool that requires user approval or reply."
   b. After the wave completes, Read the completion summaries under `<cache_path>/phase_5/` to verify success criteria were met. If a task failed, surface the failure to the user before dispatching the next wave — ask whether to retry, skip, or abort.
4. After the final wave completes, tell the user: "Hyperplan execution complete. [N] tasks executed across [M] waves." Summarize what changed in the workspace.
5. Proceed to Phase 6.

Execution agents modify the workspace directly (files, code, config) per their task — they are doing real work, not writing reports. The completion summaries under `<cache_path>/phase_5/` are the audit trail.

### Phase 6: Cleanup

1. All agents should have completed (they were foreground dispatches). If any is still running (hung or errored), call `TaskStop` with its task id — use only if an agent hangs or errors and is still running.
2. Leave `<cache_path>/` in place for the user to inspect the debate transcript (phases 1–3), the formalized plan (phase 4), and execution summaries (phase 5). Mention its location.
3. Confirm to the user with one line: "Hyperplan team disbanded."

If any step fails, surface the error and note that stray agents can be stopped via `TaskStop` with their task ids.

## ANTI-PATTERNS — DO NOT DO THESE

| Anti-pattern | Why it fails |
|--------------|--------------|
| Skipping rounds to "save time" | The adversarial filter is the entire value. Skipping rounds = vanilla planning. |
| Soft-pedaling member prompts ("be respectful") | Adversarial pressure is the mechanism. Politeness defeats the skill. |
| Synthesizing findings before Round 3 completes | Premature synthesis preserves weak findings. |
| Lead distilling insights manually instead of dispatching the round-4 planner | The round-4 planner owns distillation + formalization. Lead-manual distillation skips the planner, pollutes the Lead's context with all files of debate, and turns this back into vanilla orchestration. |
| Pre-writing or pre-drafting the plan before dispatching the planner | Anchors the planner to your draft and undermines its independent judgment. Dispatch raw — let the planner read the cache and structure. |
| Entering Phase 5 execution without explicit user confirmation | The confirmation gate is mandatory. Executing before confirmation means acting on an unapproved plan — potentially modifying the workspace against the user's intent. |
| Dispatching execution tasks sequentially instead of in parallel waves | Parallel waves are the throughput mechanism. Sequential execution wastes wall-clock and defeats the wave structure the planner designed. |
| Dispatching a wave before the previous wave completes | Breaks dependency ordering. Tasks in Wave N may depend on outputs from Wave N-1. Always wait for the full wave to return before starting the next. |
| Inlining member system prompts or task templates instead of dispatching agents to read `references/*.md` | Prompts drift from the canonical files. Dispatch the agent with the path to the relevant `references/` file + the path to `<cache_path>/VARIABLES.md`; the agent reads both files itself and interprets the {{VAR}} markers. SKILL.md is the workflow skeleton, not the prompt source. |
| Inlining variable values into a dispatch prompt instead of using `<cache_path>/VARIABLES.md` | Scatters shared state across prompts and breaks the self-driven shape. The Lead maintains VARIABLES.md once in Phase 0; every dispatch points the agent to that file. Adding a phase = one ROUND update + one dispatch line, nothing else. |
| Lead forwarding bundles between members instead of using the `<cache_path>` message bus | Defeats the filesystem-bus design. Members read each other's files directly from `<cache_path>/phase_N/`. The Lead's context must stay clean — never paste member output into a dispatch prompt. |
| Lead reading `<cache_path>/phase_1..3/` files OR `references/round-N-task.md` templates | Unnecessary context pressure. The round-4 planner reads phase_1..3 files; each member agent reads its own `references/round-N-task.md` template. The Lead only reads `<cache_path>/phase_4/plan.md` (to surface its decision points). Phases 1–3 only need the completion signal (agents returned), not the content. |
| Relying on agent memory between rounds instead of the filesystem bus | Agents are fresh each round — they have zero memory of prior rounds. All context must flow through `<cache_path>/phase_N/`. Round 3 agents must read their own `phase_1/<member>.md` to recall their findings; without it they have nothing to defend. |
| Dispatching Round N+1 without verifying Round N's output files exist | If a member agent failed silently, its output file is missing. The next round's agents will error trying to Read it. Always verify all member output files before dispatching the next round. |
| Hardcoding `.cache` paths instead of referencing `<cache_path>` | Scatters the cache location across files. `<cache_path>` is resolved once in Phase 0; every other reference (SKILL.md + templates) uses the symbol. To relocate, change only Phase 0. |
| Dispatching members sequentially instead of in parallel | Parallel dispatch is the throughput mechanism. Sequential rounds waste wall-clock and let one member's output bias another's. |
| Using `SendMessage`/`TaskOutput` for inter-round communication | Agents are fresh each round — completed agents cannot be resumed. Use `Agent` (fresh dispatch per round) and `TaskStop` (cleanup only). |
| Running this from a background subagent | Hyperplan is main-session-only orchestration. |

## NOTES FOR THE LEAD (YOU)

- Issue all member dispatches for a round in a **single message** so they run concurrently. Sequential tool calls sequentialize the debate.
- **Verify a round completed** by enumerating its output files (Phases 1–3 reference this as "the enumeration command in NOTES"): `python -c "import os; d=os.path.join(r'<cache_path>','phase_N'); print(sorted(f for f in os.listdir(d) if f.endswith('.md')))"` — replace `phase_N` with the round just completed. Do NOT use Glob; it silently returns no results when the full path is passed in `pattern`.
- **Every round is a FRESH `Agent` dispatch.** Each round: update `ROUND` in `<cache_path>/VARIABLES.md`, then dispatch via the Unified dispatch shape (one fresh `Agent` call per member in one message, `MEMBER_NAME` in the dispatch prompt), wait for all to return, verify outputs. Agents read VARIABLES.md + their own system prompt + task template; the Lead does NOT read templates.
- Members read each other's files directly. The Round 2 template tells each fresh agent to Read all `phase_1/*.md` files; the Round 3 template tells each fresh agent to Read its own `phase_1/<member_name>.md` (to recall its findings) AND all `phase_2/*.md` files (to find attacks on itself). File formats in the templates are parse-critical — do not improvise them.
- The Round 4 planner writes the plan directly to `<cache_path>/phase_4/plan.md` (instructed in `references/round-4-task.md`). You Read that file, surface its decision points to the user (not the whole plan), then gate execution. If the planner needs more context, dispatch a fresh planner using the Unified dispatch shape with the additional context appended to the prompt — do NOT try to resume the previous one.
- Phase 5 execution agents modify the workspace directly per their task. Each writes a completion summary to `<cache_path>/phase_5/<task-id>.md` for the audit trail. Their prompt includes tool restrictions — no sub-agents, no SendMessage, no AskUserQuestion, no plan mode.
- If a Phase 5 task fails its success criteria, surface it to the user before the next wave. Do not silently continue — a failed dependency can corrupt downstream waves.
