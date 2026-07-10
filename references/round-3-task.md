# ROUND 3 TASK TEMPLATE — Defense and Refinement

<hyperplan-round-3-task>
YOUR TASK (Round 3 - Defend, Refine, or Concede):
You are a FRESH agent with no memory of Round 1. Your own findings live on disk.

FIRST, read your own Round 1 findings from {{CACHE_PATH}}/phase_1/{{MEMBER_NAME}}.md to recall what you originally claimed. These are the findings you must defend, refine, or concede.

THEN, read ALL Round 2 cross-attack files from {{CACHE_PATH}}/phase_2/ — use Glob to list every .md file, then Read each one.

In each file, locate the section "## Attacks on {{MEMBER_NAME}}" — that section contains the attacks targeting YOUR Round 1 findings. Collect every attack against you across all files (the other members' files may have a section targeting you; your own file will not, since you did not attack yourself).

For each of YOUR findings under attack, choose one:
- DEFEND: rebut the attack with concrete evidence/reasoning.
- REFINE: acknowledge the attack landed, restate your finding in a stronger form.
- CONCEDE: acknowledge the attack defeated this finding. State what survives, if anything.

Be HONEST. If you were wrong, concede. If you were right, defend with concrete evidence. If you were partially right, refine. Pride is the enemy here — only defensible positions survive.

OUTPUT — Write your defenses to {{CACHE_PATH}}/phase_3/{{MEMBER_NAME}}.md using the Write tool. Use this EXACT format (the round-4 planner reads this to distill surviving insights):

# Round 3 Defenses: {{MEMBER_NAME}}

- Finding #1: DEFEND/REFINE/CONCEDE: [explanation ≤3 sentences]
- Finding #2: DEFEND/REFINE/CONCEDE: [explanation ≤3 sentences]
...

Address EVERY finding of yours that was attacked. Findings not mentioned here are treated as CONCEDED by default. The planner drops conceded findings and keeps defended/refined ones.
</hyperplan-round-3-task>

---
PLACEHOLDERS (orchestrator substitutes before dispatch):
- {{CACHE_PATH}} — absolute path to the cache directory (resolved in Phase 0).
- {{MEMBER_NAME}} — this member's name (the defender).
