---
description: "Pre-flight, stage a paused campaign with the approved leads, then launch on your final yes."
argument-hint: "[audience or campaign] [optional: searchId or lead list]"
---

Prepare a launch for: $ARGUMENTS

Follow the War Room skill: `${CLAUDE_PLUGIN_ROOT}/skills/war-room/SKILL.md` (session start, permission table, honesty rules). If that path is not expanded in your host, use `skills/war-room/SKILL.md` in the War Room plugin folder; reference files are in its `references/` folder.

Follow `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/launch.md` and `${CLAUDE_PLUGIN_ROOT}/skills/war-room/references/safety.md`.

1. **Pre-flight** (read-only): delegate to `war-room:launch-operator` when available, otherwise run it yourself.
   - `STOP` → explain what to fix and stop.
   - `NEEDS CONFIRMATION` → ask the user those questions now (always including: is the profile's daily invitation maximum 40 or less?). Then:
     - Maximum above 40 or unknown → offer `update_agent` with a safe range (Approve). Ask how long the profile has run on Kakiyo and use the tiers in `safety.md` (first 3 days 15-20, days 3-7 25-30, later 30-40); if unknown, 15-20. If accepted, run it and read back with `get_agent`; if today's `invitationsDaily` is still above 40, launch tomorrow. If declined → stop.
     - Profile `paused` → offer `resume_agent` (Approve), saying it also restarts that profile's other active campaigns. If accepted, run it and read back with `get_agent` (must be `running`; `agent_has_alerts` → stop). If declined → stop.
     - Hidden-wallet credits not confirmed, or any other "no" / "don't know" → stop.
   - `GO` → continue.
2. **Choose the path.**

   **A. New campaign (default):**
   1. Confirm: `create_campaign` with `preventAutoStart: true` (it is created `paused`).
   2. Approve: add only the approved people (`add_leads_to_campaign` with the keeper `prospectIds`, or one `add_prospects_batch` call). Follow the import as `launch.md` says.
   3. Confirm: save customData for Lead Finder leads while the campaign is still paused.
   4. Read back with `get_campaign_stats` and `list_campaign_prospects`, then ask one question: campaign, profile, number of people, pace.
   5. Approve: `resume_campaign` only on an explicit yes (or inside autonomous limits the user switched on in this session). Read back (`list_campaigns` shows `active`).

   **B. Existing campaign (only if the user asks for it):** check its status in `list_campaigns` and use the table in `launch.md`.
   - `active`: adding people **is** the launch. Ask for approval before the add (campaign, profile, people, pace). Lead Finder leads start without customData; say so, or use URLs with customData instead. No `resume_campaign` afterwards.
   - `paused`: a URL import **may** restart it (when it no longer has `preventAutoStart`, at least one person is imported and it has a prompt); a Lead Finder import keeps it paused. Ask for approval with that consequence spelled out.
   - `prospects_needed` (no people yet): either import **may** activate it. Treat the add as the launch and ask for approval first.
   - `draft`, `disabled`, `completed`, unknown: don't add people; ask the user to fix it in the dashboard.

   Before adding to an existing campaign, note its `stats.prospects` from `list_campaigns` (the baseline). After an add on a `paused` or `prospects_needed` campaign, **wait for the import before deciding anything**: Lead Finder → `get_lead_search` until `phase` is `imported` or `failed` and read `import.imported`; URLs → `stats.prospects` above the baseline, plus the slug check from `launch.md` (completion can't be proven over MCP; ask the user to check the dashboard if many are missing). Then read the status from `list_campaigns`:
   - `active` → it launched; report it.
   - Still `paused` / `prospects_needed` and at least one person was newly imported by this run → follow steps A3-A5, and say that resuming also restarts outreach to the people who were already in the campaign. Don't guess why it didn't start.
   - Nobody newly imported → don't resume; report what was skipped.
3. Report with the launch output format from `launch.md`.
