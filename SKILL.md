---
name: hyperplan
description: "Adversarial multi-agent planning skill. Self-orchestrates 5 hostile agents (skeptic, validator, researcher, architect, creative) via parallel Agent-tool dispatch for ruthless cross-critique debate (3 rounds). Agents exchange findings through a filesystem message bus (<cache_dir>/phase_N/<member>.md) — the Lead never forwards bundles. After 3 rounds the Lead distills survivors and MANDATORILY hands the insight bundle to the `Plan` agent for executable plan formalization. Use when planning needs maximum rigor and surfacing of weak assumptions, blind spots, and over-engineering. Triggers: 'hyperplan', 'hpp', '/hyperplan', 'adversarial plan', 'hostile planning', 'cross-critique plan'."
---

# HYPERPLAN — Adversarial Multi-Agent Planning

> **MANDATORY**: First action when this skill loads — say "HYPERPLAN MODE ENABLED!" so the user knows orchestration started.

## WHAT THIS IS

You (the orchestrator) become the **Lead** of a 5-member adversarial team. The 5 members are **maximally hostile** to each other — they attack each other's findings ruthlessly. Members exchange their outputs through a **filesystem message bus** (`<cache_dir>/phase_N/<member>.md`): each member writes its own output file, and reads the other members' files directly. The Lead never forwards bundles — the Lead only dispatches, waits for completion, and finally reads `<cache_dir>/phase_3/` to distill survivors into an insight bundle, which is MANDATORILY handed to the `Plan` agent.

This is not consensus building. This is intellectual combat. Weakness gets exposed. Lazy thinking gets eviscerated. Only what survives the gauntlet makes it into the plan.

## HARD PRECONDITIONS

Before starting, verify:

1. **The 5 adversarial subagent types are available** via the Agent tool's `subagent_type` parameter: `skeptic`, `validator`, `researcher`, `architect`, `creative`. If any is missing, STOP and tell the user which are unavailable.
2. **The `Plan` subagent type is available** for the Phase 5 handoff.
3. **The `references/` folder exists alongside this SKILL.md** and contains all 9 prompt files:
   - 5 member system prompts: `skeptic.md`, `validator.md`, `researcher.md`, `architect.md`, `creative.md`
   - 4 task/handoff templates: `round-1-task.md`, `round-2-task.md`, `round-3-task.md`, `plan-handoff.md`

   These files ARE the prompts — the orchestrator never inlines them. At runtime the orchestrator Reads each file, substitutes its `{{PLACEHOLDER}}`, and dispatches the result.
4. **You are in the main session** (not a background subagent). Hyperplan only works as a top-level orchestration.

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
└── phase_3/          # Round 3 defenses (each member writes <member>.md)
    └── ...
```

- `<cache_dir>` is resolved ONCE in Phase 0 (default `<skill_dir>/.cache`). Every other reference in this skill and in the templates uses the symbolic `<cache_dir>` — to relocate the cache, change only the Phase 0 definition.
- `<cache_dir>/` is gitignored runtime state. It is left in place after the run for inspection; the user may delete it.
- The Write tool auto-creates these directories on first write — no need to mkdir.
- The Lead reads `<cache_dir>/phase_3/*.md` (plus `phase_1/` and `phase_2/` for provenance) ONLY in Phase 4 for distillation. Phases 1–3 never put member content into the Lead's context.

## PROMPT TEMPLATES (in references/)

All member-facing task prompts and the final Plan-handoff prompt live in `references/` as templates with `{{PLACEHOLDER}}` markers. The orchestrator Reads the template, substitutes the placeholder, and dispatches the result. SKILL.md never inlines these.

| Template file | Used in | Placeholders |
|---------------|---------|--------------|
| `references/round-1-task.md` | Phase 1 (Round 1 dispatch) | `{{USER_REQUEST}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-2-task.md` | Phase 2 (Round 2 SendMessage) | `{{INPUT_DIR}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/round-3-task.md` | Phase 3 (Round 3 SendMessage) | `{{INPUT_DIR}}`, `{{MEMBER_NAME}}`, `{{OUTPUT_PATH}}` |
| `references/plan-handoff.md` | Phase 5 (Plan agent dispatch) | `{{INSIGHT_BUNDLE}}` |

## EXECUTION WORKFLOW

You execute this in **7 phases** (0–6). The Agent tool runs dispatched agents concurrently when you issue multiple Agent calls in a single message — use this for every parallel round.

**Critical separation**: You (the Lead) **distill** the surviving insights in Phase 4, but you DO NOT write the work plan. The work plan is produced by the `Plan` agent in Phase 5 — this handoff is **mandatory**, not optional.

### Tooling map (how each phase talks to members)

- **Dispatch a member (Round 1)**: `Agent` tool with `subagent_type: "<member>"`, `run_in_background: false`. The agent runs, completes, and returns. Capture the returned agent identifier (the `description` you passed, or the returned agentId) — you reuse it to resume the same agent in later rounds.
- **Resume a member for the next round**: `SendMessage` with `to: <member's identifier>` and the next round's task body. Completed agents resume in the background with full prior context.
- **Wait for a resumed member to finish writing**: `TaskOutput` with `block: true` against the member's task id. You do NOT need the returned content — the canonical output is the file the agent wrote under `<cache_dir>/`. TaskOutput is just the completion gate.
- **Final handoff**: `Agent` tool with `subagent_type: "Plan"`, `run_in_background: false`.

### Phase 0: Acknowledge and capture the request

1. Say "HYPERPLAN MODE ENABLED!" exactly once.
2. Restate the user's planning request in 1 sentence so all members start with the same scope.
3. Create your todo list for the 7 phases (the Phase 5 plan-agent handoff is mandatory — include it explicitly).
4. Resolve two absolute paths (the ONLY place these are configured — everything else references them symbolically):
   - `<skill_dir>` — the directory this SKILL.md lives in. All `references/` paths derive from it.
   - `<cache_dir>` — the debate cache directory. Defaults to `<skill_dir>/.cache`. If the user specifies a different location (or the environment prefers one), override here. All phase paths and template substitutions use `<cache_dir>`.

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

### Phase 4: Insight distillation (the Lead's job — YOU)

The team is done debating. Your job at this phase is **distillation only** — you do NOT write the work plan. You read the `<cache_dir>/` artifacts and produce a structured insight bundle that the `Plan` agent will consume in Phase 5.

1. **Read the artifacts**:
   - `<cache_dir>/phase_1/<member>.md` for all 5 members — the original Round 1 findings.
   - `<cache_dir>/phase_2/<member>.md` for all 5 members — the cross-attacks (to see what was contested, and by whom).
   - `<cache_dir>/phase_3/<member>.md` for all 5 members — the defenses (DEFEND / REFINE / CONCEDE per finding).

2. **Filter to defensible insights only.** Keep findings that:
   - Were not attacked at all (uncontested), OR
   - Were DEFENDED successfully with concrete evidence in Round 3, OR
   - Were REFINED into stronger form in Round 3.
   Drop everything that was CONCEDED. Findings absent from a member's phase_3 file are treated as CONCEDED by default.

3. **Categorize the surviving insights** into 4 buckets:
   - **Hard constraints** — invariants the plan MUST respect.
   - **Decisions made** — choices the debate converged on, with the reasoning trail.
   - **Risks & mitigations** — risks surfaced with their explicit mitigations.
   - **Open questions** — points where the debate did NOT converge; these become user-input gates in the plan.

4. **Build the insight bundle** in this exact shape (this is the payload you hand to the `Plan` agent in Phase 5):

   ```markdown
   # Hyperplan Insight Bundle: [task title]

   ## Original User Request
   [restate the user's planning request verbatim]

   ## Hard Constraints (Survived Adversarial Review)
   - [constraint] — [which member surfaced it, why it survived attack]

   ## Decisions (Converged Through Debate)
   - [decision] — [reasoning trail: who proposed, who attacked, how it was defended/refined]

   ## Risks & Mitigations
   - [risk] — [mitigation tied to a specific member's finding]

   ## Open Questions (Unresolved Debate)
   - [question] — [the contention] — [why the debate could not resolve it]

   ## Adversarial Provenance
   - skeptic findings that survived: [count]
   - validator findings that survived: [count]
   - researcher findings that survived: [count]
   - architect findings that survived: [count]
   - creative findings that survived: [count]
   - Total findings filtered out (conceded/destroyed): [count]
   ```

5. Briefly tell the user: "Adversarial distillation complete. Handing the surviving insights to the Plan agent for executable plan formalization." DO NOT present this bundle as the final plan — it is raw input for Phase 5, not the deliverable.

### Phase 5: MANDATORY plan agent handoff

You MUST dispatch the insight bundle to the `Plan` agent. The Lead does NOT write executable plans in hyperplan — that responsibility is delegated, by contract, to the dedicated planner. This separation is non-negotiable.

1. Read `references/plan-handoff.md`. Substitute `{{INSIGHT_BUNDLE}}` with the full insight bundle from Phase 4. The result is the **Plan handoff prompt**.
2. **Dispatch the handoff** as a foreground task (you wait for the plan):

   ```
   Agent({
     subagent_type: "Plan",
     run_in_background: false,
     description: "Formalize hyperplan-distilled insights into executable plan",
     prompt: "[Plan handoff prompt from step 1]"
   })
   ```

3. **Do NOT invent or pre-write the plan yourself.** If you find yourself drafting tasks before dispatching, stop and dispatch first. The plan agent's output is the deliverable.

4. **Present the plan agent's output to the user verbatim**, prefixed with one provenance line:

   ```
   *Plan derived from hyperplan adversarial review (5 members, 3 rounds) and formalized by the Plan agent.*

   [plan agent output]
   ```

5. If the plan agent returns clarifying questions instead of a plan, forward them to the user without modification — the planner is allowed to interview before committing.

DO NOT save the plan to disk unless the user asks. Hyperplan is a planning consultation, not a file-emitting workflow — the plan lives in your conversation output. (The `<cache_dir>/` artifacts are debate transcripts, not the deliverable.)

### Phase 6: Cleanup

After the plan agent's output has been presented to the user:

1. The 5 adversarial members should already have completed after Round 3 (their `TaskOutput` was collected). If any is still running, call `TaskStop` against its task id.
2. Leave `<cache_dir>/` in place for the user to inspect the debate transcript if desired. Mention its location. The user may delete it.
3. Confirm cleanup to the user with one line: "Hyperplan team disbanded."

If any step fails, surface the error and note that stray agents can be stopped via `TaskStop` with their task ids.

## ANTI-PATTERNS — DO NOT DO THESE

| Anti-pattern | Why it fails |
|--------------|--------------|
| Skipping rounds to "save time" | The adversarial filter is the entire value. Skipping rounds = vanilla planning. |
| Soft-pedaling member prompts ("be respectful") | Adversarial pressure is the mechanism. Politeness defeats the skill. |
| Synthesizing findings before Round 3 completes | Premature synthesis preserves weak findings. |
| Including conceded findings in the insight bundle | Conceded = defeated. Bundle must contain only survivors. |
| **Lead writing the plan in Phase 4 instead of handing off in Phase 5** | **The handoff is the contract. Hyperplan = adversarial distillation + dedicated planner formalization. Lead-written plans skip the planner's value-add (sequencing, dependencies, success criteria) and turn this back into vanilla orchestration.** |
| **Skipping the `Plan` agent dispatch ("the bundle is already a plan")** | **The bundle is INPUT, not output. The Plan agent owns sequencing, parallelization, and verification gates. Without the dispatch, hyperplan loses half its value.** |
| **Pre-writing tasks before dispatching to Plan agent** | **Anchors the plan agent to your draft and undermines its independent judgment. Dispatch raw insights, let the planner structure.** |
| **Inlining member system prompts or task templates instead of reading `references/*.md`** | **Prompts drift from the canonical files. Always Read the relevant `references/` file, substitute its `{{PLACEHOLDER}}`, and dispatch the result. SKILL.md is the workflow skeleton, not the prompt source.** |
| **Lead forwarding bundles between members instead of using the `<cache_dir>` message bus** | **Defeats the filesystem-bus design. Members read each other's files directly from `<cache_dir>/phase_N/`. The Lead's context must stay clean — never paste member output into a `SendMessage`.** |
| **Lead reading `<cache_dir>/` files before Phase 4** | **Unnecessary context pressure. Phases 1–3 only need the completion signal from `TaskOutput`, not the content.** |
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
- The orchestrator (you) Reads `references/round-*-task.md` and `references/plan-handoff.md` fresh each phase, substitutes the placeholders, and dispatches the result. Never inline the template text.
- `<cache_dir>` is resolved once in Phase 0 and referenced symbolically everywhere after. The orchestrator computes `{{INPUT_DIR}}` / `{{OUTPUT_PATH}}` from `<cache_dir>` when substituting template placeholders — never hardcode `.cache` in a dispatch.
- The orchestrator Reads `<cache_dir>/` artifacts ONLY in Phase 4. Phases 1–3 coordinate timing only — member content flows member-to-member through the filesystem, never through the Lead.
- Members read each other's files directly. The Round 2 template tells each member to Read all 5 `phase_1/*.md` files; the Round 3 template tells each member to Read all 5 `phase_2/*.md` files and locate the `## Attacks on <their-name>` sections. File formats in the templates are parse-critical — do not improvise them.
- The skill explicitly forbids you from softening adversarial prompts. The hostility IS the mechanism.
- The Phase 5 plan-agent handoff runs **synchronously** (`run_in_background: false`) — you wait for the planner before Phase 6 cleanup.
- The Plan agent does NOT have access to the member agents or to `<cache_dir>/`. Everything it needs must be in the handoff prompt (the distilled insight bundle). If the planner asks for additional context, you fetch it (via Read on `<cache_dir>/` or Explore) and re-dispatch with a `SendMessage` to the Plan agent's task id — do NOT spin up a new Plan agent.
