# ROUND 2 TASK TEMPLATE — Cross-Attack

<hyperplan-round-2-task>
YOUR TASK (Round 2 - Cross-Attack):
Read ALL Round 1 findings files from {{INPUT_DIR}}/ (all 5 .md files: skeptic.md, validator.md, researcher.md, architect.md, creative.md — INCLUDING your own, for reference). Use the Read tool on each file.

ATTACK the OTHER 4 members' findings ruthlessly from your adversarial role. Do NOT critique your own findings.

Be HOSTILE. Be RELENTLESS. No collegial hedging. If a finding is weak, EVISCERATE it. If you find a finding strong, say "STANDS — [reason]" and move on.

OUTPUT — Write your cross-attacks to {{OUTPUT_PATH}} using the Write tool. Use this EXACT format (in Round 3, each member will scan this file for sections targeting them):

# Round 2 Cross-Attacks: {{MEMBER_NAME}} (attacker)

## Attacks on skeptic
- skeptic Finding #1: [their claim]
  ATTACK: [your specific attack — ≤3 sentences. Concrete. Backed by evidence/reasoning per your role.]
- skeptic Finding #2: [their claim]
  ATTACK: [your attack]

## Attacks on validator
- validator Finding #1: [their claim]
  ATTACK: [your attack]

## Attacks on researcher
- researcher Finding #1: [their claim]
  ATTACK: [your attack]

## Attacks on architect
- architect Finding #1: [their claim]
  ATTACK: [your attack]

## Attacks on creative
- creative Finding #1: [their claim]
  ATTACK: [your attack]

Format is critical — in Round 3, each member finds attacks on themselves by locating the "## Attacks on <their-name>" section. Omit any member section where you have no attacks (e.g., your own section, since you do not attack yourself). Keep the "## Attacks on <name>" headers verbatim.
</hyperplan-round-2-task>

---
PLACEHOLDERS (orchestrator substitutes before dispatch):
- {{INPUT_DIR}} — absolute path to `<cache_dir>/phase_1`.
- {{MEMBER_NAME}} — this member's name (the attacker).
- {{OUTPUT_PATH}} — absolute path to `<cache_dir>/phase_2/<member>.md`.
