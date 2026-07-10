# ROUND 3 TASK TEMPLATE — Defense and Refinement

<hyperplan-round-3-task>
YOUR TASK (Round 3 - Defend, Refine, or Concede):
Read ALL Round 2 cross-attack files from {{INPUT_DIR}}/ (all 5 .md files: skeptic.md, validator.md, researcher.md, architect.md, creative.md). Use the Read tool on each file.

In each file, locate the section "## Attacks on {{MEMBER_NAME}}" — that section contains the attacks targeting YOUR Round 1 findings. Collect every attack against you across all 5 files (4 of them will have a section targeting you; your own file will not, since you did not attack yourself).

For each of YOUR findings under attack, choose one:
- DEFEND: rebut the attack with concrete evidence/reasoning.
- REFINE: acknowledge the attack landed, restate your finding in a stronger form.
- CONCEDE: acknowledge the attack defeated this finding. State what survives, if anything.

Be HONEST. If you were wrong, concede. If you were right, defend with concrete evidence. If you were partially right, refine. Pride is the enemy here — only defensible positions survive.

OUTPUT — Write your defenses to {{OUTPUT_PATH}} using the Write tool. Use this EXACT format (the Lead reads this in Phase 4 to distill surviving insights):

# Round 3 Defenses: {{MEMBER_NAME}}

- Finding #1: DEFEND/REFINE/CONCEDE: [explanation ≤3 sentences]
- Finding #2: DEFEND/REFINE/CONCEDE: [explanation ≤3 sentences]
...

Address EVERY finding of yours that was attacked. Findings not mentioned here are treated as CONCEDED by default. The Lead drops conceded findings and keeps defended/refined ones.
</hyperplan-round-3-task>

---
PLACEHOLDERS (orchestrator substitutes before dispatch):
- {{INPUT_DIR}} — absolute path to `<skill_dir>/.cache/phase_2`.
- {{MEMBER_NAME}} — this member's name (the defender).
- {{OUTPUT_PATH}} — absolute path to `<skill_dir>/.cache/phase_3/<member>.md`.
