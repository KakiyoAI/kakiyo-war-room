---
description: "Triage Kakiyo replies: hot leads to take over, AI mistakes, objections, stop requests, wrong closes. Never sends messages."
argument-hint: "[optional: campaign name]"
---

Triage the inbox for: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

Pick the campaigns (the named one, or every active campaign from `list_campaigns`). Delegate one `war-room:inbox-watch` subagent per campaign when available, **at most 2 running at once** (they share ~30 Kakiyo reads a minute), the rest one after another; otherwise follow `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/inbox.md` yourself. Use `list_campaign_prospects` per campaign, never `search_prospects` status filters.

Hot leads first, in the prospect's own words. Then list the proposed `pause_prospect` / `qualify_prospect` / `add_dnc` actions as one batch; run them only after one yes (Confirm level), one call at a time. Send AI problems and recurring objections to `/war-room:tune-prompt`.
