# Kakiyo MCP tool map

Server: `https://api.kakiyo.com/mcp` (OAuth). Every tool needs the `mcp:read` scope; tools that change data **and the Lead Finder search tools** (`find_leads`, `find_more_leads`) also need `mcp:write`. A missing scope comes back as an `insufficient_scope` result: tell the user to reconnect Kakiyo and grant write access. Do not retry.

The live tool list is the contract. If a name here is missing or different, use the live one.

Most tools proxy Kakiyo's REST API and return its JSON body as text. You never see HTTP status codes or headers, only the body (`{ "error": "...", "message": "..." }` on failure). Lead Finder tools return structured results with a `next` field: **follow `next`**.

## Call budget (read this before any batch)

Kakiyo rate-limits each workspace per minute, and the main assistant and every subagent share the same budget:

| Calls | Limit per minute |
|---|---|
| Most reads (`list_*`, `get_*`, `check_dnc`, stats) | 30 |
| Writes (`add_*`, `create_*`, `update_*`, `pause_*`, `resume_*`, `qualify_prospect`, DNC writes) and `search_prospects` | 15 |
| `verify_api_key`, `list_models` | 60 |
| Lead Finder tools | 120 (separate) |

So:

- Run writes **one at a time**, at most ~12 a minute. A batch of 40 pauses takes ~4 minutes; say so.
- Triage from list rows (`status`, `lastMessage`, `paused`) before calling `get_prospect` on each one.
- At most 2 subagents that call Kakiyo heavily should run at the same time.
- A rate-limit error is not a failure: wait (see the error table), then continue the batch where it stopped.

## Ids

| Id | Get it from | Pass it to |
|---|---|---|
| `agentId` (LinkedIn profile) | `list_agents` → `id` | `get_agent`, `update_agent`, `pause_agent`, `resume_agent`, `create_campaign` |
| `campaignId` | `list_campaigns` → `id` | campaign tools, `find_leads`, `get_lead_search`, `add_*`, `list_campaign_prospects` |
| `productId` (an **offering**) | `list_products` → `id` | `create_campaign` |
| `promptId` | `list_prompts` → `id` | `get_prompt`, `update_prompt`, `create_campaign` |
| **conversation id** | the row's top-level `id` in `list_campaign_prospects` / `search_prospects` | `get_prospect`, `pause_prospect`, `resume_prospect`, `qualify_prospect`, `update_prospect_custom_data` (their input is named `prospectId` or `chatId`, but both mean this conversation id) |
| person id | the row's nested `prospect.id` | Matching only: it equals Lead Finder's `prospectId`. Never pass it to the conversation tools |
| `searchId` | `find_leads` | `get_lead_search`, `find_more_leads`, `add_leads_to_campaign` |
| `dncId` | `check_dnc(url)` → `data.entry` (`$id` or `id`) | `remove_dnc` |

**Lead Finder lead → its conversation:** after an import, page `list_campaign_prospects({ campaignId })` and match `row.prospect.id === lead.prospectId` (fallback: the same LinkedIn slug, i.e. the part after `/in/`, URL-decoded and lower-cased; stored URLs look like `linkedin.com/in/<slug>`, so never compare whole URLs). Then use `row.id`.

## What each read tool returns

| Tool | Returns | Notes |
|---|---|---|
| `verify_api_key` | `{ status: "success", teamId }` | Read scope only |
| `list_agents` / `get_agent` | `id`, `name`, `status`, `linkedinUrl`, `usage { invitations, invitationsDaily, messages, messagesDaily, nextInvitation, nextMessage, workingHours { days, start, stop, timezone } }`, `alerts [{ type, subType, message, severity }]` | `invitationsDaily` = today's target (picked at random between the configured min and max at the profile's local midnight), `invitations` = sent today. `timezone` is a number of hours from UTC (0 = UTC or unset). `workingHours.days` is 7 digits **starting with Sunday** (`"0:1:1:1:1:1:0"` = Monday-Friday). The configured min/max and the `skipInNetwork` / `invitationsOnly` settings are **not** returned |
| `list_campaigns` | `id`, `name`, `status`, `agent` (profile id), `product` (offering id), `createdAt`, `stats` | `stats` holds all-time counters: `prospects`, `invitationsSent`, `invitationsAccepted`, `alreadyConnected`, `waitingForConnection`, `invalidURL`, `needEmail`, `unreachable`, `noLongerInNetwork`, `blocked`, `initialMessages`, `followUpsSent`, `messagesSent`, `messagesReceived`, `prospectsAnswers`, `qualified`, `closed`, `pendingInvitations`. **No prompt id, no variables, no `preventAutoStart`.** Ask the user or `war-room-context.md` for those |
| `get_campaign_stats(campaignId)` | `stats { prospects, pendingInvitations, prospectsAnswers, messages, qualified, closed, conversionRate, responseRate }` | All-time counters. `pendingInvitations` = people **queued, not yet invited**. Rates are strings (`"12.50%"`) |
| `get_analytics_overview` | Workspace totals | |
| `list_campaign_prospects({ campaignId, status?, isPaused?, hasResponded?, limit?, offset?, cursor? })` | A plain array of rows: `id` (conversation id), `status`, `paused`, `lastMessage` (a **timestamp** of the last message, either side; no text, no author), `prospect { id, name, url, headline, currentCompanyName, currentJobTitle, … }` | **The reliable way to list conversations.** Filters run in the database; rows are ordered by when the conversation was created, newest first. `limit` 1-100 (default 25). **Paging:** no "next" marker reaches you. Pass the **last row's `id` as `cursor`** for the next page, and stop when a page has fewer rows than `limit` (rarely a short page comes early; when an exact count matters, request one more page with the last `id`). (`offset` also works but can skip a row when one is dropped because its profile can't be loaded.) No total count |
| `search_prospects({ query, campaignId? })` | `{ results: [rows with id, prospectId, status, campaign {id, name}, prospect…], total }` | **Use only to look someone up by name (`query`).** Its other filters (`status`, `qualification`, `hasResponded`, `isPaused`, `city`, `country`) are applied after it takes one page of the newest conversations, so they silently miss older ones, and `total` is only that page. A name miss is not proof the person isn't there: pass `campaignId`, or page `list_campaign_prospects` |
| `get_prospect(conversation id)` | `id`, `status`, `paused`, `campaign`, `prospect { …, about }`, `messages` | In `messages`, `role: "user"` = the prospect, `role: "assistant"` = your side (Kakiyo or a human). An assistant message containing `[0null0]` means the AI chose not to reply; it was never sent: ignore it |
| `list_products` | Offerings | |
| `list_prompts` | `id`, `name`, `type` (`Messages` / `Comment`), `model` | No section text |
| `get_prompt(promptId)` | `id`, `name`, `type`, `model`, `followUpEnabled`, `variables` (custom names already on the prompt), `context`, `firstMessage`, `followUps` (or `comment`), `guidelines` (`guidelines.variables` has `mandatory`, `present`, `missingMandatory`, `customAllowed`) | Always read before `update_prompt`, and follow `guidelines` |
| `list_models` | AI models for prompts | |
| `list_dnc` | First 50 DNC entries only (no paging over MCP) | Use `check_dnc(url)` for a specific person |
| `check_dnc(url)` | `{ data: { onList, entry } }` | |
| `get_credits` | `walletVisible`, `initializing`, `balance`, `available`, `reserved`, `included`, `purchased`, `allowance`, `resetsAt`, `creditsNeeded` | See `safety.md` |
| `get_credit_usage({ from?, to?, groupBy? })` | Totals, daily series, groups | ≤ 90 days. `groupBy`: `campaign`, `agent`, `prompt`, `configuration`, `action`, `model` |
| `list_credit_requests` / `get_credit_request` | Detailed AI usage | |
| `list_webhooks` / `list_webhook_events` | Webhooks | Real events: `linkedin.message.sent`, `linkedin.message.received`, `linkedin.invitation.sent`, `linkedin.invitation.accepted`. Ignore other names in tool descriptions |
| `list_workspaces` | Agency client workspaces | Agency plan |
| `get_lead_search({ searchId, campaignId?, waitSeconds?, seenProspectIds?, limit? })` | Outside-search leads (`phase: "searching"`) or import progress (`"importing"`, `"imported"`, `"failed"`) | Waits up to 25 s on the server. Do not add delays |

**Do not call** `list_prospects` or `get_campaign_analytics`: both point at routes that do not exist and always fail. Use `list_campaign_prospects` and `get_campaign_stats` + `get_analytics_overview`.

## Write tools

| Tool | Inputs | Effect |
|---|---|---|
| `find_leads` | `description` (always required, even when paging), `campaignId?`, `searchId?`, `cursor?` (the previous `nextCursor`), `limit?` 1-50, `searchOutside?` | Starts or refines a Lead Finder search. See `lead-finding.md` |
| `find_more_leads` | `searchId` | One more outside round (uses an outside-search slot) |
| `add_leads_to_campaign` | `searchId`, `campaignId`, and exactly one of `prospectIds` (preferred) or `count` | Imports Lead Finder leads. Skips people the campaign already works and DNC entries. A search can be imported once. **Real outreach** (see `launch.md`) |
| `list_campaign_groups` | — | Campaign groups: `id`, `name`, `paused`, `members: [{ campaignId, name, weight, agentId, status }]`, `stats` (`people`, `contacted`, `connected`, `replied`, `qualified`, `retargeted`, `rescued`, `review`). A group is several campaigns (each on its own LinkedIn profile) sharing one lead pool; inside a group a person has one active conversation |
| `add_prospect` | `campaignId` **or** `groupId`, `linkedinUrl`, `name?`, `customData?` (≤ 3000), `email?`, `company?`, `jobTitle?` | One URL. A DNC match fails at once with `prospect_on_dnc_list`. Otherwise it starts a background import (returns a `listId`, can fail with `campaign_busy`). **Real outreach** |
| `add_prospects_batch` | `campaignId` **or** `groupId`, `prospects: [{ linkedinUrl, name?, customData?, email?, company?, jobTitle? }]` | Starts a background import and returns at once with a `listId`. "N prospects added" means accepted, not imported. One malformed URL rejects the whole call with `validation_error`. A person who already has a conversation on the same profile gets no new one, but its customData is **replaced** by the one you send, or by the one built from `email` / `company` / `jobTitle` (both URL tools; see `launch.md`). With `groupId` the list is split across the group's campaigns by their weights (the response lists each campaign's `listId`), and a person already in the group is skipped. **Real outreach** |
| `update_prospect_custom_data` | `chatId` (conversation id), `customData` (≤ 3000, `""` clears) | Replaces that conversation's customData. You cannot read the current value first |
| `create_campaign` | `name`, `productId`, `promptId`, `agentId`, `variables?`, `preventAutoStart?` | Created `paused` with `preventAutoStart: true`, otherwise `active`. Always pass `true` |
| `update_campaign` | `campaignId`, `name?`, `variables?` | `variables` **replaces the whole set**; you cannot read the current set |
| `pause_campaign` / `resume_campaign` | `campaignId` | Resume sets `active` from **any** status and clears `preventAutoStart`; fails only with `prompt_required` or `agent_restricted`. Only resume a `paused` campaign, or a `prospects_needed` one you just added people to |
| `update_agent` | `agentId`, `workingHours?`, `limits?`, `settings?` | See `safety.md`. `workingHours.days` starts with **Sunday**, despite the tool description saying Mon-Sun. New limits apply from the profile's next local midnight |
| `pause_agent` / `resume_agent` | `agentId` | Neither checks the current status. Pause overwrites any status except `restricted`: only pause a `running` profile. Resume fails with `agent_has_alerts` while there are alerts: only resume a `paused` profile |
| `pause_prospect` / `resume_prospect` | `chatId` | AI off / on for one conversation |
| `qualify_prospect` | `chatId` | Sets status 5 |
| `update_prompt` | `promptId` + only the sections you change | Affects every campaign using the prompt. See `playbook.md` |
| `add_dnc` | `url` (`reason` is accepted but not stored) | Blocks that person in every campaign |
| `remove_dnc` | `dncId` | Unblocks |
| Webhooks | `create_webhook`, `update_webhook`, `delete_webhook` | |
| Workspaces | `create_workspace`, `delete_workspace` (permanent), `invite_client`, `remove_client`, `assign_agent_to_workspace`, `unassign_agent_from_workspace` | Agency plan |

## Not possible over MCP (send the user to the dashboard)

Connect or re-login a LinkedIn profile · create an offering or prompt · Sandbox tests · custom qualification criteria (campaign → Qualification) · send a manual message (Inbox) · Monitors (Dashboard → Monitors) · Sales Navigator / CSV imports · company or bulk DNC · delete prospects or campaigns · see a campaign's prompt, variables or `preventAutoStart` · see a profile's configured daily min/max · billing.

## Errors

| `error` | Meaning | Do |
|---|---|---|
| `insufficient_scope` | Connection lacks a scope | Ask the user to reconnect and grant it |
| `validation_error` | Bad input. `details[]` holds only the **first** problem; for lists, its index is in `message` (e.g. `"prospects[3].url" …`) | Fix or drop that item and resend; repeat until accepted. Check all LinkedIn URLs before sending (`https://www.linkedin.com/in/<slug>`, no commas, brackets or braces) |
| `invalid_request` | Missing or conflicting arguments | Read `message`, fix, resend once |
| `subscription_inactive`, `subscription_required`, `invalid_plan` | No active or trial plan | Stop. Point to Dashboard → Billing |
| `feature_unavailable` | Feature switched off for this workspace | Tell the user; contact Kakiyo |
| `forbidden`, `not_found`, `campaign_not_found`, `search_not_found` | Wrong or other team's id | Re-list; never guess ids |
| `product_not_found`, `prompt_not_found`, `agent_not_found` | Wrong id for `create_campaign` | Re-list offerings / prompts / profiles |
| `limit_reached` | Plan's campaign limit | Reuse a campaign or upgrade |
| `team_limit_reached` | Lead Finder workspace limit | Report the message; continue in the dashboard |
| `agent_restricted` | LinkedIn restricted the profile | Never use it. See `safety.md` |
| `agent_has_alerts` | Profile has alerts | Show `get_agent` alerts; fix in the dashboard |
| `prompt_required` | Campaign has no prompt | Fix in the dashboard |
| `campaign_missing_agent` / `invalid_campaign` | No profile on the campaign | Assign one in the dashboard |
| `campaign_busy` | An import into this campaign is still running | Wait, re-check `get_campaign_stats`, retry once |
| `prospect_on_dnc_list` | That person is on DNC | Skip them |
| `dnc_entry_exists` | Already on DNC | Treat as success |
| `missing_mandatory_prompt_variables` | Prompt lacks required variables (`missingMandatory` lists them) | Add them back |
| `prompt_too_long`, `model_not_found` | Prompt over 80,000 characters / unknown model | Shorten / pick from `list_models` |
| `invalid_follow_ups`, `too_many_follow_ups`, `invalid_follow_up_delay` | Follow-up rules broken | Max 3, delay 1-30 |
| `daily_limit_reached` | Lead Finder MCP cap | Report `resetsAt`. Do not loop |
| `search_launched`, `invalid_status` | Search already imported, now locked | Follow it with `get_lead_search`, or start a new `find_leads` without `searchId` |
| `invalid_selection` | A `prospectId` not from this search | Pick from that search's results |
| `invalid_description` | Lead Finder couldn't use the sentence | Rewrite it as one specific sentence |
| `not_started`, `outside_search_failed` (from `find_more_leads`) | No usable outside search for the current sentence | `find_leads({ description, searchId, searchOutside: true })` once, then `get_lead_search` |
| `search_changed` | Another round started meanwhile | `get_lead_search` first, then decide |
| `busy`, `conflict` | Something else is running on this search | Wait a few seconds, retry once |
| `launch_failed`, `outside_search_unavailable`, `preview_timeout`, `internal_error` | Lead Finder failed or timed out | Retry once if the message suggests it; otherwise report it and continue in Dashboard → Searches |
| `invalid_cursor`, `semantic_cursor_expired`, `semantic_cursor_mismatch`, `semantic_cursor_invalid`, `semantic_snapshot_conflict` | The page cursor no longer works | Call `find_leads` again with the same `searchId` and sentence, without `cursor` |
| `rate_limit_exceeded` / `rate_limited` | Over the call budget | Wait `retryAfter` / `retryAfterSeconds` seconds, or until `resetTime` (epoch ms); if none is given, wait 60 s. Then continue the batch from where it stopped |

## Call policy

1. Read before writing. Batch reads; don't call `get_prospect` 50 times when a list answers the question.
2. After every write, read back and report the ids and counts you actually saw.
3. On `insufficient_scope` or a subscription error, stop all writes for the session.
4. Never print tokens or API keys.
