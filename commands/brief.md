---
description: "Read-only status of your Kakiyo workspace: profiles, campaigns, credits, replies waiting and anything blocked. Changes nothing."
argument-hint: "[optional: campaign name]"
---

Give a War Room brief for: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

Read tools only:

1. `verify_api_key`, `list_agents`, `list_campaigns`, `get_credits`. Campaign numbers come from `list_campaigns` → `stats` (no per-campaign stats calls needed).
2. Replies waiting: `list_campaign_prospects({ campaignId, status: 4, limit: 100 })` and `status: 5` per active campaign (or only the one named). Write "≥ 100" when a page is full. Stay within ~30 Kakiyo reads a minute: with many active campaigns, do the busiest first and say which you skipped.
3. Profiles that can't run, alerts and `creditsNeeded`: explain with `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/statuses.md` and `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/safety.md`.

Show the board from `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/routing.md` and the next three actions. Change nothing.
