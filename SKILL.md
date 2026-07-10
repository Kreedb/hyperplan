---
name: hyperplan
description: "Adversarial multi-agent planning skill. Self-orchestrates 5 hostile agents (skeptic, validator, researcher, architect, creative) via parallel Agent-tool dispatch for ruthless cross-critique debate (3 rounds), then MANDATORILY delegates distillation to the `Plan` agent for executable plan formalization. Use when planning needs maximum rigor and surfacing of weak assumptions, blind spots, and over-engineering. Triggers: 'hyperplan', 'hpp', '/hyperplan', 'adversarial plan', 'hostile planning', 'cross-critique plan'."
---

# HYPERPLAN — Adversarial Multi-Agent Planning

> **MANDATORY**: First action when this skill loads — say "HYPERPLAN MODE ENABLED!" so the user knows orchestration started.

## WHAT THIS IS

You (the orchestrator) become the **Lead** of a 5-member adversarial team. The 5 members are **maximally hostile** to each other — they attack each other's findings ruthlessly. You then synthesize only the **defensible insights** that survived the attacks into an insight bundle, and MANDATORILY hand it to the `Plan` agent for executable plan formalization.

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

## PROMPT TEMPLATES (in references/)

All member-facing task prompts and the final Plan-handoff prompt live in `references/` as templates with `{{PLACEHOLDER}}` markers. The orchestrator Reads the template, substitutes the placeholder, and dispatches the result. SKILL.md never inlines these.

| Template file | Used in | Placeholder |
|---------------|---------|-------------|
| `references/round-1-task.md` | Phase 1 (Round 1 dispatch) | `{{USER_REQUEST}}` |
| `references/round-2-task.md` | Phase 2 (Round 2 SendMessage) | `{{ROUND1_BUNDLE}}` |
| `references/round-3-task.md` | Phase 3 (Round 3 SendMessage) | `{{ATTACKS_ON_YOU}}` |
| `references/plan-handoff.md` | Phase 5 (Plan agent dispatch) | `{{INSIGHT_BUNDLE}}` |

## EXECUTION WORKFLOW

You execute this in **7 phases** (0–6). The Agent tool runs dispatched agents concurrently when you issue multiple Agent calls in a single message — use this for every parallel round.

**Critical separation**: You (the Lead) **distill** the surviving insights in Phase 4, but you DO NOT write the work plan. The work plan is produced by the `Plan` agent in Phase 5 — this handoff is **mandatory**, not optional.

### Tooling map (how each phase talks to members)

- **Dispatch a member (Round 1)**: `Agent` tool with `subagent_type: "<member>"`, `run_in_background: false`. The agent runs, completes, and returns its findings directly. Capture the returned agent identifier (the `description` you passed, or the returned agentId) — you reuse it to resume the same agent in later rounds.
- **Resume a member for the next round**: `SendMessage` with `to: <member's identifier>` and the next round's task. Completed agents resume in the background with full prior context.
- **Collect a resumed member's output**: `TaskOutput` with `block: true` against the member's task id — waits for the round's response.
- **Final handoff**: `Agent` tool with `subagent_type: "Plan"`, `run_in_background: false`.

### Phase 0: Acknowledge and capture the request

1. Say "HYPERPLAN MODE ENABLED!" exactly once.
2. Restate the user's planning request in 1 sentence so all members start with the same scope.
3. Create your todo list for the 7 phases (the Phase 5 plan-agent handoff is mandatory — include it explicitly).
4. Resolve the absolute path to this skill's `references/` directory (the same directory SKILL.md lives in). You will pass `<skill_dir>/references/<member>.md` to each dispatched agent so it can Read its own system prompt, and you will Read `<skill_dir>/references/round-*-task.md` + `plan-handoff.md` yourself to assemble each dispatch prompt.

### Phase 1: Round 1 — Independent analysis (dispatch + analyze)

1. Read `references/round-1-task.md`. Substitute `{{USER_REQUEST}}` with the user's planning request verbatim. The result is the **Round 1 task body**.
2. Dispatch all 5 members in **parallel** — issue 5 `Agent` tool calls in a **single message** so they run concurrently. Each member's prompt is:

   ```
   Read the file at <skill_dir>/references/<member>.md and adopt it as your system prompt for this task.

   [Round 1 task body from step 1]
   ```

   Substitute `<member>` with `skeptic` / `validator` / `researcher` / `architect` / `creative`, and use the matching `subagent_type` on each `Agent` call. Use `run_in_background: false` so all 5 return their findings to you directly.

3. Capture each member's returned identifier (description / agentId) — you need it to resume them in Round 2.

### Phase 2: Round 2 — Cross-attack

1. When all 5 Round 1 replies have arrived, aggregate them into one bundle:

   ```
   === Round 1 Findings Bundle ===
   [skeptic]:
   1. ...
   2. ...

   [validator]:
   1. ...

   [researcher]:
   1. ...

   [architect]:
   1. ...

   [creative]:
   1. ...
   === End ===
   ```

2. Read `references/round-2-task.md`. Substitute `{{ROUND1_BUNDLE}}` with the bundle from step 1. The result is the **Round 2 task body**.
3. Resume all 5 members in **parallel** — issue 5 `SendMessage` calls in a single message, one to each member's identifier. Each receives the SAME Round 2 task body.
4. After sending, issue 5 `TaskOutput` calls in a single message (one per member, `block: true`) to collect all 5 cross-attacks.

### Phase 3: Round 3 — Defense and refinement

1. Aggregate the cross-attacks BY ORIGINAL FINDING. For each Round 1 finding, list all the attacks that targeted it. Then, for each member, assemble the block of attacks against THEIR OWN findings only, in this shape:

   ```
   [member]'s Finding #N: [your original claim]
     - [attacker-name] said: [attack]
     - [attacker-name] said: [attack]
   ...
   ```

2. Read `references/round-3-task.md`. For EACH member, substitute `{{ATTACKS_ON_YOU}}` with that member's attack block from step 1. Each member gets a DIFFERENT Round 3 task body.
3. Resume all 5 members in parallel — 5 `SendMessage` calls in a single message, each sending its member-specific Round 3 task body.
4. Issue 5 `TaskOutput` calls in a single message to collect all 5 refinements.

### Phase 4: Insight distillation (the Lead's job — YOU)

The team is done debating. Your job at this phase is **distillation only** — you do NOT write the work plan. You produce a structured insight bundle that the `Plan` agent will consume in Phase 5.

1. **Filter to defensible insights only.** Keep findings that:
   - Were not attacked at all (uncontested), OR
   - Were defended successfully with concrete evidence in Round 3, OR
   - Were refined into stronger form in Round 3.
   Drop everything that was conceded.

2. **Categorize the surviving insights** into 4 buckets:
   - **Hard constraints** — invariants the plan MUST respect.
   - **Decisions made** — choices the debate converged on, with the reasoning trail.
   - **Risks & mitigations** — risks surfaced with their explicit mitigations.
   - **Open questions** — points where the debate did NOT converge; these become user-input gates in the plan.

3. **Build the insight bundle** in this exact shape (this is the payload you hand to the `Plan` agent in Phase 5):

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

4. Briefly tell the user: "Adversarial distillation complete. Handing the surviving insights to the Plan agent for executable plan formalization." DO NOT present this bundle as the final plan — it is raw input for Phase 5, not the deliverable.

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

DO NOT save the plan to disk unless the user asks. Hyperplan is a planning consultation, not a file-emitting workflow — the plan lives in your conversation output.

### Phase 6: Cleanup

After the plan agent's output has been presented to the user:

1. The 5 adversarial members should already have completed after Round 3 (their `TaskOutput` was collected). If any is still running, call `TaskStop` against its task id.
2. Confirm cleanup to the user with one line: "Hyperplan team disbanded."

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
| Dispatching members sequentially instead of in parallel | Parallel dispatch is the throughput mechanism. Sequential rounds waste wall-clock and let one member's output bias another's. |
| Dispatching a fresh agent each round instead of resuming via `SendMessage` | Loses member continuity. Round 3 defense needs the member's memory of WHY it made each Round 1 finding. |
| Using `team_*` tools (they do not exist in this environment) | Use `Agent` / `SendMessage` / `TaskOutput` / `TaskStop` only. |
| Running this from a background subagent | Hyperplan is main-session-only orchestration. |

## NOTES FOR THE LEAD (YOU)

- Issue all 5 dispatches / messages / output-fetches for a round in a **single message** so they run concurrently. Sequential tool calls sequentialize the debate.
- After Round 1 (foreground dispatch), members complete and return. To continue them in Round 2 / Round 3, use `SendMessage` (resumes the completed agent in the background with full context), then `TaskOutput` with `block: true` to collect the response.
- Track each member's identifier (the `description` you passed to `Agent`, or the returned agentId) from Phase 1 — you need it for every `SendMessage` and `TaskOutput` call afterward.
- Members read their `references/<member>.md` system prompt ONCE in Round 1. Do NOT re-instruct them to read it in Round 2 / Round 3 — they resume with full context via `SendMessage`.
- The orchestrator (you) Reads `references/round-*-task.md` and `references/plan-handoff.md` fresh each phase, substitutes the placeholder, and dispatches the result. Never inline the template text in SKILL.md.
- The members do not see each other's responses directly — only what you forward via `SendMessage`. You are the information broker. The bundles you forward in Phases 2 and 3 are the entire context they have.
- Keep bundles concise — ≤32KB per message. If aggregated findings exceed this, summarize before forwarding (preserve the spirit of each finding).
- The skill explicitly forbids you from softening adversarial prompts. The hostility IS the mechanism.
- The Phase 5 plan-agent handoff runs **synchronously** (`run_in_background: false`) — you wait for the planner before Phase 6 cleanup.
- The Plan agent does NOT have access to the member agents. Everything it needs must be in the handoff prompt. If the planner asks for additional context, you fetch it (via Explore / Read) and re-dispatch with a `SendMessage` to the Plan agent's task id — do NOT spin up a new Plan agent.
