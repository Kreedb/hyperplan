# ROUND 4 TASK TEMPLATE — Distillation + Executable Plan Formalization

<hyperplan-round-4-task>
The user's planning request:
<user-request>
{{USER_REQUEST}}
</user-request>

YOUR TASK (Round 4 — Distill + Formalize):
You are the dedicated planner. The 5-member adversarial team has completed 3 rounds of debate. You OWN two jobs in this single dispatch: (1) distill the surviving insights from the debate artifacts, (2) formalize them into an executable plan with parallelizable tasks. Do not split these — distillation informs the plan, and the plan is meaningless without the distilled constraints.

STEP 1 — READ THE DEBATE ARTIFACTS
Read ALL 15 files under {{CACHE_DIR}}/ using the Read tool:
- {{CACHE_DIR}}/phase_1/{skeptic,validator,researcher,architect,creative}.md — Round 1 original findings (the raw positions)
- {{CACHE_DIR}}/phase_2/{skeptic,validator,researcher,architect,creative}.md — Round 2 cross-attacks (what was contested, and by whom)
- {{CACHE_DIR}}/phase_3/{skeptic,validator,researcher,architect,creative}.md — Round 3 defenses (DEFEND / REFINE / CONCEDE per finding)

STEP 2 — DISTILL SURVIVING INSIGHTS
Keep findings that:
- Were not attacked at all (uncontested), OR
- Were DEFENDED successfully with concrete evidence in Round 3, OR
- Were REFINED into stronger form in Round 3.
Drop everything CONCEDED. Findings absent from a member's phase_3 file are treated as CONCEDED by default — the member gave up rather than defend.

Categorize survivors into 4 buckets:
- Hard constraints — invariants the plan MUST respect (violating these = guaranteed failure)
- Decisions made — choices the debate converged on, with the reasoning trail (who proposed, who attacked, how it was defended/refined)
- Risks & mitigations — risks surfaced with their explicit mitigations (tied to a specific member's finding)
- Open questions — points the debate did NOT resolve; these become user-input gates in the plan

STEP 3 — FORMALIZE THE EXECUTABLE PLAN
Turn the distilled insights into discrete tasks. Structure each task for PARALLEL EXECUTION — the Lead will dispatch independent tasks concurrently in dependency-ordered waves. Minimize dependencies to maximize parallelism.

RETURN the full plan as your final message. Do NOT use the Write tool — you do not have it. Do NOT enter plan mode. Do NOT use task management tools. The Lead will persist your output to {{CACHE_DIR}}/phase_4/plan.md. Use this EXACT format:

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
- skeptic findings survived: [count]
- validator findings survived: [count]
- researcher findings survived: [count]
- architect findings survived: [count]
- creative findings survived: [count]
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

---
PLACEHOLDERS (orchestrator substitutes before dispatch):
- {{CACHE_DIR}} — absolute path to the debate cache directory (resolved in Phase 0). The Plan agent reads the 15 debate files from here.
- {{USER_REQUEST}} — the user's planning request, verbatim. Included so the planner can check distilled insights against the original intent.
