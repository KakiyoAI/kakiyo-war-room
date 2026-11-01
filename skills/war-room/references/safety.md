# Safety: LinkedIn profiles, credits, DNC

The LinkedIn account belongs to a real person. Protect it over any volume target.

## Daily limits (`update_agent`)

`update_agent({ agentId, limits: { invitationsMin, invitationsMax, messagesMin, messagesMax } })`. Whole numbers: `invitationsMin` 1-100, `invitationsMax` 2-100, `messagesMin` 1-500, `messagesMax` 2-500. Kakiyo picks a random value between min and max each day. **Always send min and max together**, with min below max: the API doesn't check the order.

You can't read the configured min/max over MCP; `get_agent` only shows today's pick (`usage.invitationsDaily`). A new range applies from the profile's next local midnight, so today's pick doesn't change right after `update_agent` (use `pause_agent` to stop today's sending). If today's pick is above 40, flag it and ask the user what range is set.

The guidance from Kakiyo's LinkedIn safety guide (the dashboard marks up to 40 invitations a day as the safe zone), which the War Room follows:

| Profile | Invitations/day |
|---|---|
| New to Kakiyo, first 3 days | 15-20 |
| Days 3-7 | 25-30 |
| After 7 days | 30-40 |
| Ceiling, never exceed | 40 |

The same guide puts LinkedIn's monthly maximum at about 800 invitations. Never set `invitationsMax` above 40, even in autonomous mode or when asked to "go faster". Add another profile instead and say why. Don't raise message limits without a reason the user gives.

Other settings:

- `workingHours: { days, start, stop }`. `days` = 7 digits separated by `:`, **starting with Sunday** (Sun:Mon:Tue:Wed:Thu:Fri:Sat), at least one active. Weekdays = `"0:1:1:1:1:1:0"`, every day = `"1:1:1:1:1:1:1"`. (The tool's own description says Mon-Sun; that is wrong: Kakiyo reads index 0 as Sunday.) `get_agent` returns the same Sunday-first string. `start` / `stop` in 24 h (`"09:00"`), and `stop` must be later than `start` (no overnight windows). Times follow the profile's UTC offset (`usage.workingHours.timezone`, hours from UTC, set in the dashboard).
- `settings: [{ key: "skipInNetwork", value: true }]` → don't message people already connected (they get status 7).
- `settings: [{ key: "invitationsOnly", value: true }]` → only send invitations, no messages.

## Profile problems

| Status / alert | Tell the user |
|---|---|
| `auth_needed` | LinkedIn wants a new login or 2FA. Reconnect in Dashboard → LinkedIn Profiles. Never ask for the password or code |
| `restricted` | LinkedIn restricted the account. Stop everything on it; contact Kakiyo support. Don't retry |
| `setup_needed`, `configuring`, `auth_in_progress` | Setup unfinished or in progress |
| `credits_needed` | Out of AI credits. Add credits in Billing (or ask the agency); it returns by itself. Never `resume_agent` |
| `team_paused`, `payment_paused` | Subscription paused or payment failed. Billing |
| `disabled` + `no_campaigns` | Normal when no campaign is active; it restarts by itself when one is |
| `ended` | Closed profile. Use another |
| Other alerts | Show `message` and `severity`. `resume_agent` fails with `agent_has_alerts` until they're resolved in the dashboard |

LinkedIn warnings or unusual alerts on a `running` profile → propose `pause_agent` right away, then contact support. Never "test" by resuming. Don't call `pause_agent` on a profile that is already stopped (`credits_needed`, `auth_needed`, `team_paused`, `payment_paused`, `disabled`): it overwrites that status and breaks its automatic recovery (a paused `disabled` + `no_campaigns` profile no longer restarts by itself).

## "Nothing is sending": diagnosis

1. `get_agent`: status (`credits_needed`, `auth_needed`, `paused`…), `alerts`, `usage.nextInvitation` / `nextMessage`, `usage.invitations` vs `invitationsDaily` (today's target already reached?), `workingHours` (outside hours or an off day?).
2. `list_campaigns`: is the campaign `active`? Is it on this profile?
3. `get_campaign_stats`: `pendingInvitations` > 0 means people are queued; 0 means the queue is empty (add people).
4. `get_credits`: `creditsNeeded`?
5. Many rows at 505 → LinkedIn invite cooldown; Kakiyo retries later.

Report the first blocker you find and what fixes it.

## Credits

- `get_credits` → `balance`, `available`, `reserved`, `included`, `purchased`, `allowance`, `resetsAt`, `creditsNeeded`.
- **`creditsNeeded: true` is the stop signal**: new AI work is paused until the balance is positive. Point to Dashboard → Billing.
- `available` = balance minus reservations for work already running. It can read 0 while `creditsNeeded` is false; that's not a reason to block.
- `walletVisible: false` → an agency manages the credits and there is no `creditsNeeded` field. A `credits_needed` profile status is then the only signal, and it only appears on profiles that were `running` when credits ran out; for an idle profile, ask the user or the agency. `initializing: true` → no numbers yet. Neither means zero.
- `get_credit_usage({ groupBy: "campaign" })` shows where credits went.

## Do-not-contact

- `check_dnc(url)` → `data.onList`, `data.entry`. `add_dnc(url)` blocks that person in every campaign (a `reason` is accepted but not stored). `remove_dnc(dncId)` with the entry's id from `check_dnc`.
- `list_dnc` returns only the first 50 entries over MCP. To check a person, use `check_dnc`.
- Company-level and bulk DNC: Dashboard → Prospects → Do Not Contact.
- Kakiyo checks DNC at Lead Finder import, on `add_prospect` (`prospect_on_dnc_list`) and before sending invitations and messages. These checks are best effort: a failed lookup lets the person through, results are cached for a few minutes, and follow-ups are not checked. A `check_dnc` "not on list" is not a guarantee.
- Someone who says stop → `pause_prospect` (the immediate stop) + `add_dnc` (`dnc_entry_exists` means already blocked: fine).
- Never remove a DNC entry unless the user names the person.

## Hard stops (also in autonomous mode)

- Profile can't run (see `statuses.md`) → no launch.
- `creditsNeeded: true` → no launch.
- Invitations above 40/day → never. Configured maximum unknown → the user confirms it before launch. Today's pick (`invitationsDaily`) above 40 → no launch today.
- Credits unknown (agency wallet hidden and the profile is idle) → the user or agency confirms before launch.
- Never-contact rules from the context file → never.
- `delete_workspace` → only on an explicit request that names the workspace; it's permanent.
