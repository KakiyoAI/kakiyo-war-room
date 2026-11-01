---
description: "Review a Kakiyo AI prompt (context, first message, follow-ups) and propose a diff. Updates it only after you approve."
argument-hint: "[prompt name] [optional: campaigns using it, what feels wrong]"
---

Review and improve this prompt: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

`list_campaigns` can't show which prompt a campaign uses: confirm the prompt and the campaigns using it with the user. Resolve the `promptId` with `list_prompts`.

Delegate to `war-room:playbook-writer` when available (give it the `promptId`, the campaigns, and any conversation ids that show the problem); otherwise follow `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/playbook.md` and `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/ai-tells.md` yourself.

Show the diagnosis with quoted lines, the old → new sections and the affected campaigns. `update_prompt` is Approve level: call it only after an explicit yes, with only the changed fields. Suggest Sandbox scenarios.
