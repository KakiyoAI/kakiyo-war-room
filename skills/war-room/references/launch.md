# Launch: staging and launching campaigns

The Launch Operator runs the read-only pre-flight. You, the Commander, run every write in this file, after the approvals in `SKILL.md`.

## What adding people does

| Campaign | `add_prospect` / `add_prospects_batch` (URLs) | `add_leads_to_campaign` (Lead Finder) |
|---|---|---|
| `active` | Outreach to them starts on the profile's schedule | Same |
| `paused`, `preventAutoStart` off (never set, or cleared by an earlier resume) | **Reactivates the campaign** when the import finishes (if at least one person was imported and the campaign has a prompt) | Stays paused |
| `prospects_needed`, `preventAutoStart` off | Becomes active | Becomes active |
| `paused` with `preventAutoStart` on | Stays paused until `resume_campaign` | Same |
| `draft`, `disabled`, `completed`, unknown | Don't add people; ask the user | Same |

Over the MCP, `preventAutoStart` can only be set by `create_campaign`, and it can't be read back. Any `resume_campaign` clears it for good.

So:

- **Staging (safe for both paths):** `create_campaign` with `preventAutoStart: true` → add the approved people → review → `resume_campaign` on a final yes.
- A campaign counts as **staged** only if you created it in this session with `preventAutoStart: true` and it has never been resumed. Adding people to any other campaign is a launch (Approve level). To add people to an existing **paused** campaign without restarting it, use Lead Finder (`add_leads_to_campaign`); a URL import would restart it.

## Pre-flight (read-only; the Launch Operator returns this)

1. **Profile:** `get_agent(agentId)`.
   - `running` → OK.
   - `disabled` whose only alert is `no_campaigns` → OK: it restarts by itself once the campaign is active.
   - `paused` with no alerts → needs `resume_agent` (Approve).
   - `credits_needed` → **stop**: credits first (Billing).
   - Anything else, or other alerts → **stop** and tell the user what to fix in the dashboard (`statuses.md`, `safety.md`).
2. **Credits:** `get_credits`. `creditsNeeded: true` → stop (new AI work is paused). `available` can be 0 while credits are only reserved; that alone is not a stop. `walletVisible: false` (agency-managed) or `initializing: true` → no number to read, and don't treat it as zero: if the profile is `running` (not `credits_needed`), credits are fine; if it is idle (`disabled`, `paused`), the user or agency must confirm there are credits before launch, otherwise STOP (unknown).
3. **Offering and prompt:** the offering exists (`list_products`); the user confirms which prompt (`list_prompts`). The prompt should use `{{customData}}` if you will send customData. Missing → created in the dashboard.
4. **DNC:** Kakiyo checks the DNC list at Lead Finder import, on `add_prospect`, and before sending invitations and messages, but these checks are best effort (a failed lookup lets the person through) and follow-ups are not checked. Drop never-contact matches from the context file yourself, and `check_dnc` anyone the user flagged.
5. **Daily limit and capacity:** MCP can't read the configured invitation range, only today's pick (`usage.invitationsDaily`). So the user must confirm the profile's maximum is 40 or less (Dashboard → LinkedIn Profiles), or accept `update_agent` to set it (Approve; pick the range from the tiers in `safety.md` by asking how long the profile has run on Kakiyo, and use 15-20 when unknown): a NEEDS CONFIRMATION question, asked before staging. Unknown or above 40 → no launch. Today's pick above 40 → no launch today even after lowering the range (the new range applies from the profile's next local midnight): recheck `get_agent` tomorrow. Capacity: people ÷ `usage.invitationsDaily` ≈ days of work (e.g. 300 people at ~30/day ≈ 10 working days).
6. **Overlap:** is there an existing campaign for the same audience and profile (`list_campaigns`)? Prefer reusing it.

Output, plus the numbers above:

- `STOP` + reasons: something only the dashboard or billing can fix (profile can't run, `creditsNeeded: true`, today's pick above 40, no offering).
- `NEEDS CONFIRMATION` + the exact questions: things only the user can answer (configured invitation maximum ≤ 40, which prompt, credits when the wallet is hidden, resume a `paused` profile). The Commander asks these **before staging**. A maximum above 40 or unknown → offer `update_agent` with a safe range (Approve); a `paused` profile → offer `resume_agent` (Approve; it also restarts that profile's other active campaigns). Run what the user approves, read back with `get_agent`, and treat any declined or unanswered question as STOP.
- `GO`: nothing left to check.

## Staging a new campaign

1. **Confirm:** `create_campaign({ name, productId, promptId, agentId, variables?, preventAutoStart: true })`. Name it audience + date, e.g. "RevOps DE 50-500 (Nov 2026)". It is created `paused`.
2. **Approve** (adding people to a staged campaign sends nothing yet, but ask once anyway, listing who):
   - **Lead Finder:** `add_leads_to_campaign({ searchId, campaignId, prospectIds: [Scout's keepers] })`. Use `count` ("top N") only if the user explicitly asks for Kakiyo's top N instead of the triaged list. Then call `get_lead_search({ searchId, campaignId })` while `phase` is `"importing"`. `"imported"` and `"failed"` are final: report imported / skipped as already contacted / skipped for DNC / errors. On `"failed"`, report the partial counts and send the user to Dashboard → Searches.
   - **URLs:** check every URL yourself first: `https://www.linkedin.com/in/<slug>`, no commas, brackets or braces. One malformed URL rejects the whole call with `validation_error`, and `details` names only the **first** bad item (its index is in `message`, e.g. `"prospects[3].url" …`): drop or fix it and resend, repeating until accepted. Then one `add_prospects_batch({ campaignId, prospects: [{ linkedinUrl, name?, customData? }] })` with the **whole** approved list. A second import into the same campaign while the first runs fails with `campaign_busy`. The response ("N prospects added") only means accepted; duplicates, unusable profiles and DNC entries are dropped while it processes. If the user supplies `email`, `company` or `jobTitle`, pass them; they become customData only when no `customData` is given.
   - **Overwrite trap:** if a person already has a conversation on the same LinkedIn profile (in any campaign), a URL import creates nothing new but **replaces that conversation's customData**. `email`, `company` and `jobTitle` count too: without `customData` they are turned into customData. You can't reliably prove over MCP that someone is new (a name search can miss). So, whenever the profile already has other campaigns, either:
     - tell the user "anyone already contacted from this profile will get their context replaced" and get approval for that, or
     - send those people with only `linkedinUrl` and `name` (no `customData`, `email`, `company` or `jobTitle`).
     A `search_prospects({ query })` hit or a slug match in the campaign's rows confirms someone is not new; a miss proves nothing.
   - **Checking a URL import:** no tool reports when it is finished, and a steady `stats.prospects` is not proof. Page `list_campaign_prospects({ campaignId })` and match each approved person by **slug**: the part after `/in/`, URL-decoded, lower-cased, without trailing slash (Kakiyo stores URLs as `linkedin.com/in/<slug>`, so whole URLs won't match). Report "n of N matched so far"; unmatched means "not matched", not "not imported" (enrichment can change a slug). If many are unmatched after a few minutes, ask the user to check the import in the dashboard before launching.
3. **Confirm:** Lead Finder leads that need customData → `update_prospect_custom_data` per conversation (matching in `intel.md`), one call at a time (see the call budget), while the campaign is still paused.
4. **Read back:** `get_campaign_stats(campaignId)` and `list_campaign_prospects({ campaignId, limit: 25 })`.
5. **Ask:** "Launch now? [campaign] on [profile] (daily maximum confirmed ≤ 40), [n] people, ≈ [days] days at ~[invitationsDaily]/day."
6. **Approve:** `resume_campaign(campaignId)`. Read back with `list_campaigns` (status `active`). Re-check the profile with `get_agent` (a `disabled` / `no_campaigns` profile restarts within minutes).

## Managing live outreach

| Request | Tool | Level |
|---|---|---|
| Stop a campaign | `pause_campaign` | Confirm (safe brake). Only on an `active` campaign |
| Restart it | `resume_campaign` | Approve. Only on a `paused` campaign (staged or user-paused), or a `prospects_needed` campaign you just added people to: the tool sets `active` from any status, so never use it on an empty campaign, `draft`, `disabled` or `completed`. Re-run pre-flight steps 1, 2 and 5 |
| Stop a profile everywhere | `pause_agent` | Confirm. Only on a `running` profile. It overwrites any other status: on `credits_needed` it cancels the automatic return after a top-up, on `auth_needed` it hides the login problem, on `disabled` + `no_campaigns` it blocks the automatic restart. Those are already stopped: report, don't pause |
| Restart a `paused` profile | `resume_agent` | Approve. Only from `paused`, only with no alerts |
| Change hours / limits | `update_agent` | Approve. Inside `safety.md`. New limits apply from the profile's next local midnight; to stop today's sending, `pause_agent` |
| Rename / change campaign variables | `update_campaign` | Approve. `variables` replaces the whole set and MCP can't read the current one: ask the user for the full set. Never reuse a built-in variable name |
| Take over one conversation | `pause_prospect(conversation id)` | Confirm |

## Output

```
# Launch ([date])
Pre-flight: GO | NEEDS CONFIRMATION ([questions]) | STOP ([reasons])
Mode: staged | launched | paused
Campaign: [name] ([campaignId]), status [x], created this session: yes/no
Profile: [name] ([agentId]), status [x], invitationsDaily [n], alerts [none | …]
Offering: [name] · Prompt: [name] (confirmed by user)
Added: n · Skipped (already contacted): n · Skipped (DNC): n · Errors: n
Estimated pace: ~n days
Next: [approval question or next check]
```

## Guardrails

- Never launch on a profile that can't run, or while `creditsNeeded` is true.
- Never `create_campaign` without `preventAutoStart: true`.
- Never create duplicate campaigns for the same audience and profile.
- Never resume something the user paused unless they ask.
- Never raise limits to "go faster": more volume means another profile, not a riskier one.
