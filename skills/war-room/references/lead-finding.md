# Lead finding: Scout

Find the right people with Kakiyo Lead Finder, then triage before anyone sees a list. A title match alone is not a lead.

Lead Finder needs a trialing or active plan and a connection with write access. Otherwise the tools return `subscription_required` (with a billing link), `feature_unavailable` or `insufficient_scope`: stop and pass it on.

Cost: `find_leads` can start a paid outside search on its own (when the database has fewer than 10 strong matches), and `find_more_leads` always starts one. Both count against the daily caps below.

## The flow

Every Lead Finder result has a `next` field telling you what to do. Follow it.

```
1. find_leads({ description, campaignId })
2. get_lead_search({ searchId, campaignId, seenProspectIds })   while outsideSearch.status is "running" or "interrupted"
3. find_more_leads({ searchId })                                 only if the user wants more and the outside search is "done"
4. (Commander, after approval) add_leads_to_campaign
```

### 1. Writing the description

One specific sentence: role + seniority + industry + location + company size or stage. Good: "VPs of Sales at B2B SaaS companies in the Nordics with 50-500 employees". Bad: "sales, SaaS, Nordics". If the request is vague, ask one short question first. A size written in employees and 1-3 named countries are applied as hard filters.

Pass `campaignId` whenever there is a target campaign: people its LinkedIn profile already works are left out (`alreadyInCampaign` counts them).

### Refining

Call `find_leads` again with the **same `searchId`** and the **full updated sentence**. That replaces any outside search still running for the old sentence. The same sentence on the same search is free (reused).

### Next page of the same search

`find_leads({ description: <same sentence>, searchId, cursor: <nextCursor from the last response>, campaignId })`. `description` is required by the tool but ignored when paging. Stop when `nextCursor` is null / `hasMore` is false.

### 2. The outside search

When the database has fewer than 10 strong matches, Kakiyo searches LinkedIn and the web in the background (`outsideSearch.status: "running"`). If the user explicitly asks to search beyond the database (or wants more volume than it gives), call `find_leads` with `searchOutside: true`. If they want database results only, pass `searchOutside: false`.

Collect outside leads with `get_lead_search({ searchId, campaignId, seenProspectIds: [every prospectId already shown] })`. It waits up to 25 s on the server: never add your own delays. Usually 1-3 calls. If `moreLeads` > 0, call again to get the rest. Tell the user what is happening between calls ("still searching, 7 new leads so far").

`outsideSearch.status` values:

| Status | Meaning | Do |
|---|---|---|
| `running`, `interrupted` | Still working | Call `get_lead_search` again |
| `done` | Finished | Show the new leads; offer `find_more_leads` if the user wants more |
| `not_needed` | Enough strong database matches | Nothing. `searchOutside: true` forces one if the user asks |
| `skipped` + `reason` (`daily_outside_limit`, `auto_limited`, `busy`, …) | Not started | Report the reason; the database results stand |
| `not_started` | No outside search for this sentence | `find_leads` with `searchOutside: true` if the user wants one |
| `failed` | It failed | Continue with what you have, or retry once with `searchOutside: true` |
| `replaced` | The sentence changed | Call `find_leads` again with the current sentence |

Always follow the result's `next`.

### 3. More leads

`find_more_leads({ searchId })` starts one more round with the same sentence, only after `done`. Then call `get_lead_search` with its `alreadyFoundProspectIds` (plus anything else you showed) as `seenProspectIds`.

| Refusal | Do |
|---|---|
| `no_more`, `search_limit_reached`, `daily_limit_reached` | Final for now. Report (with `resetsAt` if given); suggest a refined sentence or the dashboard |
| `not_started`, `outside_search_failed` | `find_leads({ description, searchId, searchOutside: true })` once |
| `search_changed` | Another round just started: `get_lead_search` first |

### Daily caps over MCP (per workspace, rolling 24 h, defaults)

20 new searches, 20 description changes, 10 outside searches started. Every `find_leads` response carries `dailyLimits`: trust it over these numbers and report what is left. Repeats, paging, `get_lead_search` and imports never count.

## What a lead looks like

`prospectId`, `name`, `title`, `headline`, `company`, `industry`, `companySize`, `location`, `linkedinUrl`, `strength` (`strong` / `possible`), `matchReason` (the top signal in full, or the literal `"missing"`), `signals` (each with text, type, and a date and source when known), `source` (`database` / `outside`). **Never an email or phone number.** Outside leads may lack size or industry.

## Triage (before showing anything)

For each lead:

1. **Company fit:** industry, size, location, business model against the brief. Missing data is "unknown", not a pass.
2. **Title fit:** can this person buy or champion? "Founder" of a 2-person shop is not a VP of Sales.
3. **Never rules** from the context file.
4. **Signal freshness:** a signal with no date, or older than ~12 months, is not a "why now".
5. **Verdict:** **keep / maybe / drop** with one sentence. Every drop has a reason.

`strong` leads that pass go first. `possible` leads need a real reason to keep.

## Output

```
# Leads: [description] ([date])
searchId: … · campaign: [name or none] · outside search: [status, and reason if skipped]
Daily caps left: searches n · refines n · outside n

| # | Name | Title | Company | Size | Location | Strength | Signal (matchReason) | Verdict | Why |

Keep: n · Maybe: n · Drop: n
Keeper prospectIds: [...]
```

Return the keeper `prospectId`s: the Commander imports exactly those. Do not add anyone to a campaign.

## Guardrails

- Nobody found → say so, then offer to widen **one** variable (title, or geography, or size).
- Never invent a signal to make a row look warm.
- A search can be imported once; after that it is locked (`search_launched`) and new people need a new `find_leads` without `searchId`.
