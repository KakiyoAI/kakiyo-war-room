---
description: "Funnel and performance report for your Kakiyo campaigns: where it leaks and the top three fixes."
argument-hint: "[optional: campaign name, or 'credits']"
---

Report on: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

Delegate to `war-room:analyst` when available; otherwise follow `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/analytics.md` yourself.

Every number from a tool, counts next to every rate, all-time counters labelled as such. End with at most three changes, each with an owner, and what you could not determine. Write the report to `war-room-report-[date].md` if the host can write files.
