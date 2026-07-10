# ROUND 1 TASK TEMPLATE — Independent Analysis

<hyperplan-round-1-task>
The user's planning request:
<user-request>
{{USER_REQUEST}}
</user-request>

YOUR TASK (Round 1 - Independent Analysis):
Apply your adversarial role to this request. Produce 3-7 numbered findings.
Each finding must be ≤3 sentences and SPECIFIC (cite files, line numbers, alternatives, or evidence as required by your role).

DO NOT critique anything yet. DO NOT propose a synthesized plan. JUST findings from your role's perspective.

OUTPUT — Write your findings to {{OUTPUT_PATH}} using the Write tool. Use this EXACT format (other members will parse it in Round 2):

# Round 1 Findings: {{MEMBER_NAME}}

1. [finding]
2. [finding]
...

Numbering matters — in Round 2, other members will reference your findings as "{{MEMBER_NAME}} Finding #N". Keep the numbers stable and explicit.
</hyperplan-round-1-task>

---
PLACEHOLDERS (orchestrator substitutes before dispatch):
- {{USER_REQUEST}} — the user's planning request, verbatim.
- {{MEMBER_NAME}} — this member's name (skeptic / validator / researcher / architect / creative).
- {{OUTPUT_PATH}} — absolute path to `<skill_dir>/.cache/phase_1/<member>.md`.
