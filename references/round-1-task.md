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

OUTPUT — Write your findings to {{CACHE_PATH}}/phase_1/{{MEMBER_NAME}}.md using the Write tool. Use this EXACT format (other members will parse it in Round 2):

# Round 1 Findings: {{MEMBER_NAME}}

1. [finding]
2. [finding]
...

Numbering matters — in Round 2, other members will reference your findings as "{{MEMBER_NAME}} Finding #N". Keep the numbers stable and explicit.
</hyperplan-round-1-task>

---
VARIABLES REFERENCED (the Lead provides these in the dispatch prompt's VARIABLES block; the agent reads this template and interprets the {{VAR}} markers with the provided values — no string substitution by the Lead):
- {{USER_REQUEST}} — the user's planning request, verbatim.
- {{MEMBER_NAME}} — this member's name.
- {{CACHE_PATH}} — absolute path to the cache directory.
