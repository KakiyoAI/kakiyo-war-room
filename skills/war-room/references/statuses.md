# Status codes

## Conversation status (`status` on rows from `list_campaign_prospects`)

| Code | Meaning | Who acts |
|---|---|---|
| 0 | Not started (queued) | Kakiyo, on the profile's schedule |
| 1 | Invitation sent, waiting for acceptance | LinkedIn |
| 2 | Invitation accepted | Kakiyo reads the profile and writes the first message |
| 3 | First message sent (follow-ups go out from here if enabled) | Waiting for the prospect |
| 4 | Prospect replied | Kakiyo's AI is handling it; Inbox Watch reviews |
| 5 | Qualified | Kakiyo (or a human) marked it qualified. Inbox Watch reviews it; the AI stays on until someone pauses the conversation |
| 6 | Unqualified / closed (includes "not interested") | Nobody, but Inbox Watch checks for stop requests and wrong closes |
| 7 | Skipped: already connected and `skipInNetwork` is on | Nobody |
| 404 | Invalid LinkedIn URL | Nobody |
| 501 | Needs an email to invite | Nobody |
| 502 | Invites disabled for this person | Nobody |
| 503 | Unreachable | Nobody |
| 504 | No longer in network | Nobody |
| 505 | Invite cooldown (LinkedIn blocks re-inviting for now) | Kakiyo retries later |

`list_campaign_prospects` accepts `status` filters 0-7 and 502-505 only (404 and 501 can appear in rows but cannot be filtered).

`paused: true` on a conversation = the AI is off for it; a human owns it.

Kakiyo qualifies conversations itself (auto-qualification). `qualify_prospect` is for when a human decides.

## LinkedIn profile (agent) status (`status` from `list_agents` / `get_agent`)

| Status | Meaning | Can a campaign on it run? |
|---|---|---|
| `running` | Working on its schedule | Yes |
| `disabled` with a `no_campaigns` alert | Idle because it has no active campaign | Yes: Kakiyo restarts it by itself once one of its campaigns is active. Do not call `resume_agent` or `pause_agent` (pausing blocks the automatic restart) |
| `disabled` with any other alert | Stopped | No. Show the alert; fix in the dashboard |
| `paused` | Paused by the user | Only after `resume_agent` (Approve level) and only with no alerts |
| `setup_needed` | Not set up | No. Finish in the dashboard |
| `configuring`, `auth_in_progress` | Being set up / logging in | Wait |
| `auth_needed` | LinkedIn wants a new login or 2FA | No. User reconnects in Dashboard → LinkedIn Profiles. Never ask for the password |
| `credits_needed` | The workspace ran out of AI credits | No. Credits are added in Dashboard → Billing (or by the agency); the profile then returns to its previous status by itself. Never `resume_agent` |
| `team_paused` | Subscription paused | No. Billing |
| `payment_paused` | Payment failed | No. Billing |
| `restricted` | LinkedIn restricted the account | Never. Contact Kakiyo support |
| `ended` | Closed | No |
| anything else | Unknown | Treat as not runnable; ask the user |

Any entry in `alerts` (`type`, `subType`, `message`, `severity`) is worth showing the user.

## Campaign status (`status` from `list_campaigns`)

| Status | Meaning |
|---|---|
| `active` | Outreach runs (when its profile can run) |
| `paused` | Nothing new goes out |
| `prospects_needed` | Set up but has no people yet |
| `draft` | Not finished in the dashboard |
| `disabled` | Switched off in the dashboard. Nothing goes out; re-enable it there |
| `completed` | Finished |
| anything else | Unknown: don't add people; ask the user |

What adding people does to each status: see the table in `launch.md`.
