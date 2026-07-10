# ROUND 4 TASK TEMPLATE — Distillation + Executable Plan Formalization

<hyperplan-round-4-task>
The user's planning request:
<user-request>
{{USER_REQUEST}}
</user-request>

YOUR TASK (Round 4 — Distill + Formalize):
You are the dedicated planner. The adversarial team has completed 3 rounds of debate. You OWN two jobs in this single dispatch: (1) distill the surviving insights from the debate artifacts, (2) formalize them into an executable plan with parallelizable tasks. Do not split these — distillation informs the plan, and the plan is meaningless without the distilled constraints.

STEP 1 — READ THE DEBATE ARTIFACTS
Read ALL .md files under {{CACHE_PATH}}/ using the Read tool. Discover them by listing each phase directory:
- {{CACHE_PATH}}/phase_1/*.md — Round 1 original findings (the raw positions)
- {{CACHE_PATH}}/phase_2/*.md — Round 2 cross-attacks (what was contested, and by whom)
- {{CACHE_PATH}}/phase_3/*.md — Round 3 defenses (DEFEND / REFINE / CONCEDE per finding)

STEP 2 — DISTILL SURVIVING INSIGHTS
Keep findings that:
- Were not attacked at all (uncontested), OR
- Were DEFENDED successfully with concrete evidence in Round 3, OR
- Were REFINED into stronger form in Round 3.
Drop everything CONCEDED. Cross-reference phase_2 (which records who attacked what) against phase_3: a finding that WAS attacked in Round 2 but appears nowhere in the member's phase_3 file (neither defended, refined, nor explicitly conceded) is treated as CONCEDED — the member gave up rather than defend. Uncontested findings (never attacked, so absent from phase_2 attacks) are KEPT per the first criterion.

Categorize survivors into 4 buckets:
- Hard constraints — invariants the plan MUST respect (violating these = guaranteed failure)
- Decisions made — choices the debate converged on, with the reasoning trail (who proposed, who attacked, how it was defended/refined)
- Risks & mitigations — risks surfaced with their explicit mitigations (tied to a specific member's finding)
- Open questions — points the debate did NOT resolve; these become user-input gates in the plan

STEP 3 — FORMALIZE THE EXECUTABLE PLAN
Turn the distilled insights into discrete tasks. Structure each task for PARALLEL EXECUTION — the Lead will dispatch independent tasks concurrently in dependency-ordered waves. Minimize dependencies to maximize parallelism.

Write the full plan to {{CACHE_PATH}}/phase_4/plan.md using the Write tool. Do NOT enter plan mode. Do NOT use task management tools, or any tool that requires user approval or reply. Return a one-line confirmation ('Plan written to {{CACHE_PATH}}/phase_4/plan.md') as your final message. Use this EXACT format:

# Hyperplan Executable Plan: [task title]

## Original User Request
[restate the user's planning request verbatim]

## Distilled Insights

### Hard Constraints (Survived Adversarial Review)
- [constraint] — [which member surfaced it, why it survived attack]

### Decisions (Converged Through Debate)
- [decision] — [reasoning trail: who proposed, who attacked, how defended/refined]

### Risks & Mitigations
- [risk] — [mitigation tied to a specific member's finding]

### Open Questions (Unresolved Debate)
- [question] — [the contention] — [why the debate could not resolve it]

### Adversarial Provenance
- <member-name> findings survived: [count]  (repeat one line per member)
- Total findings filtered out (conceded/destroyed): [count]

## Execution Tasks

### Task 1: [short imperative title]
- Description: [what to do, concretely — a single agent must be able to complete this in one dispatch]
- Dependencies: [task IDs that must complete first, or "none"]
- Success criteria: [how to verify this task is done — observable, testable]
- Files likely touched: [paths, if known]

### Task 2: [short imperative title]
...

## Wave Structure (for the Lead's parallel dispatch)
- Wave 1: [task IDs with no dependencies] — dispatch in parallel
- Wave 2: [task IDs whose dependencies are in Wave 1] — dispatch after Wave 1 completes
- Wave 3: [task IDs whose dependencies are in Wave 2] — dispatch after Wave 2 completes
- ...

CONSTRAINTS:
- Prefer 3-8 tasks total. Fewer = clearer; more = harder to coordinate.
- Each task must be completable by a single agent in one dispatch — no multi-agent coordination within a task.
- Make every task as independent as possible. Fewer dependencies = more parallelism.
- If open questions block a task, mark that task's dependencies as requiring user resolution first.
- Do NOT include a "review" or "cleanup" task — the Lead handles those out-of-band.
</hyperplan-round-4-task>
