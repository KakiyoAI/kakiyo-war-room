---
description: "Find and triage leads with Kakiyo Lead Finder. Returns a keep / maybe / drop table. Adds nobody to a campaign."
argument-hint: "[who to find, e.g. 'VPs of Sales at B2B SaaS in the Nordics, 50-500 employees'] [optional: campaign]"
---

Find leads for: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

If a campaign is named, resolve its id with `list_campaigns` and pass it so people it already works are left out.

Delegate to the `war-room:scout` subagent when available (give it the sentence and the `campaignId`); otherwise follow `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/lead-finding.md` yourself.

Show the triaged table, the `searchId`, the keeper `prospectId`s and the daily caps left. Do not call `add_leads_to_campaign`. Offer next steps: research the keepers (`/war-room:research`) or stage a launch (`/war-room:launch`).
