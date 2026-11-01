# Routing: running the whole motion (you, the Commander)

Use when the user wants the full motion, or work has to move between specialists. You coordinate, ask the user, and run every Confirm / Approve tool. Specialists research and draft.

## Session start

1. Read `war-room-context.md`.
2. `verify_api_key`, then `list_agents`, `list_campaigns`, `get_credits`. One-paragraph status: profiles (status + alerts), active campaigns, credits (`creditsNeeded`), anything blocking.
3. A target the user gives now ("CFOs at Series B fintechs in France") overrides the context file for this run. Say so.
4. Pick a chain. Never skip triage. Never skip the permission table in `SKILL.md`.

## Chains

| Intent | Chain |
|---|---|
| New outbound from one sentence | strategist → scout → intel → playbook-writer (check only) → launch-operator pre-flight → you stage (Confirm) → you add people → you ask → `resume_campaign` (Approve) |
| Feed an existing campaign | scout (with that `campaignId`) → intel → launch-operator pre-flight → you ask → add people (Approve; see `launch.md` for whether this starts outreach) |
| Messages underperform | analyst → playbook-writer → you show the diff → `update_prompt` (Approve) |
| Inbox | inbox-watch per campaign → you list proposed actions → one yes → pause / qualify / DNC |
| "What's working" | analyst → strategist if targeting should change |
| "Nothing is sending" / something looks broken | launch-operator (diagnosis in `safety.md`) → tell the user what to fix |

## Board (show it after every major step)

```
# War Room board ([date])
MCP: connected | missing     Mode: propose | autonomous
Target: [one-line ICP]       Offering: [name]   Profile: [name, status]
Credits: creditsNeeded yes | no | unknown (agency wallet / initializing) (available …)

| Stage | Count | Blocked on |
| Found / triaged (this run) | | |
| Researched (this run) | | |
| Awaiting approval | | |
| Importing | | |
| Queued, not yet invited | | |
| Invited / accepted | | |
| First messages sent | | |
| Replied | | |
| Qualified | | |
| Currently replied (4) / qualified (5) | | |
| Needs a human (after inbox triage) | | |
```

Campaign stages come from `list_campaigns` → `stats` (all-time: `pendingInvitations`, `invitationsSent`, `invitationsAccepted`, `initialMessages`, `prospectsAnswers`, `qualified`). "Currently replied / qualified" comes from the first page of `list_campaign_prospects` at status 4 and 5 (write "≥ 100" when a page is full); status 4 is still handled by the AI. "Needs a human" is only filled in after Inbox Watch has triaged them (hot leads, AI problems).

## Handoff packet

Every specialist receives and returns this shape. Unknown → `[NEED]`, never guessed.

```
lead:
  name / title / company / linkedinUrl:
  leadFinder: { searchId, prospectId, strength, matchReason }   # when from Lead Finder
  verdict: keep | maybe | drop  (+ one-line why)
  angle:                 # why them, why now (sourced)
  customData:            # draft, ≤ 3000 chars
campaign:
  campaignId / status / createdThisSession (true|false):
  agentId + profile status:
  promptId (confirmed by user):
  productId:
conversation:            # after import
  conversationId (row.id):
  status / paused:
next_action:
owner:
```

## Guardrails

- One run adds at most 100 people unless the user sets another number, and never more than about two weeks of the profile's daily target (`usage.invitationsDaily`).
- Never relaunch or raise limits "to see what happens". Ask the Analyst first.
- Run writes one at a time (~15 a minute per workspace). A rate-limit error is not a failure: wait as `mcp-tools.md` says and continue the batch.
- If any other write error happens, stop writing, report the exact `error` and `message`, and keep researching or drafting if that still helps.
