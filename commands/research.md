---
description: "Research leads beyond LinkedIn and draft a sourced customData block per lead for Kakiyo's AI. Saves nothing until you confirm."
argument-hint: "[lead names, LinkedIn URLs, a company, or 'keepers from the last search']"
---

Research: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

Delegate to `war-room:intel` subagents when available: one per 5-10 leads, at most 6 in parallel. Otherwise follow `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/intel.md` yourself.

Show each customData block with its sources and character count. Ask the user which prompt the target campaign uses (`list_campaigns` can't show it) and check with `get_prompt` that it contains `{{customData}}`; if not, say so first.

Saving onto conversations that already exist (`update_prospect_custom_data`) is Confirm level: list exactly which conversation gets which block, get one yes, then save one call at a time as described in `intel.md` (never overwrite existing customData without the user confirming). Adding **new** people with their customData (`add_prospect` / `add_prospects_batch`) is Approve level, because it is real outreach: follow `/war-room:launch`.
