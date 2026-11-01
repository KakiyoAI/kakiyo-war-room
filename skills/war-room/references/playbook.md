# Playbook: Playbook Writer

In Kakiyo you don't write each message. You write the **prompt** that Kakiyo's AI follows in every conversation of a campaign; the AI personalises each message from the prospect's profile, the offering and the lead's customData. One good prompt beats a hundred hand-written DMs.

## How a Kakiyo prompt works

`list_prompts` gives `id`, `name`, `type`, `model`. `get_prompt({ promptId })` returns a **Messages** prompt as sections:

| Section | Controls |
|---|---|
| `context` | Who the AI is, the mission, tone, boundaries, objection handling, when to propose a meeting |
| `firstMessage` | How the opening message after connection acceptance is written |
| `followUps` | Up to **3** follow-ups, each `{ prompt, delay }`; `delay` = days after the previous message, whole number **1-30** |
| `followUpEnabled` | Turns follow-ups on/off without deleting them |

A **Comment** prompt has only `comment`. `get_prompt` also returns `guidelines`: follow it over this file if they differ.

### Variables

The AI fills `{{variables}}` from data.

- **Mandatory, across `context` + `firstMessage` together** (follow-ups don't count): `prospectName` (or `prospectFirstName` + `prospectLastName`), `prospectHeadline`, `prospectLocation`, `prospectPastExperiences`, `prospectCurrentExperience`, `prospectEducation`, `prospectResume`, `prospectPosts`, `agentName`, `offeringDescription` (the legacy `productDescription` also counts). `guidelines.variables.missingMandatory` lists any that are missing.
- **Recommended:** `prospectCompanyName`, `prospectCompanyIndustry`, `prospectCompanySize`, `prospectCompanyDescription`, `prospectCompanyWebsite`, `prospectSkills`, `prospectLanguages`, `prospectFollowers`, `prospectConnections`, `prospectCertifications`, `prospectVolunteer`, `prospectAccomplishments`, `prospectRecommendations`, `prospectAdditional`, `companyName`.
- **Optional:** `customData` (Intel's research), `teamName`, `prospectLinkedInUrl`, `prospectLinkedInSlug`, `prospectEmail`, `prospectPhone`.
- Careful: `companyName` and `teamName` are **the user's own workspace name**, not the prospect's company (that is `prospectCompanyName`). `agentName` is the LinkedIn profile's name.
- **Custom names** (e.g. `{{goal}}`) are filled from each campaign's `variables`. Use one only when every campaign using the prompt sets it (ask the user; MCP can't read campaign variables). An unknown or misspelled name is replaced with an empty string, silently.
- Campaign `variables` override built-in names. Never set a campaign variable called `prospectName`, `offeringDescription` or any other built-in name: it would replace the real data in every message. The one useful exception is `customData`, which becomes the default for conversations that have none of their own.

If the War Room researches leads, the prompt must use `{{customData}}`, for example in `context`: "If {{customData}} is not empty, use its 'Why now' as the reason you are reaching out, in your own words. Never say you researched them."

## Process

1. **Find the prompt.** `list_campaigns` does **not** show which prompt a campaign uses. Ask the user (or read `war-room-context.md`) which prompt and which campaigns use it. Never guess. Then `get_prompt`.
2. Read it against `war-room-context.md` (voice, CTA, proof, never-say list) and real conversations if available (Inbox Watch quotes, or `get_prospect` messages, where `role: "user"` is the prospect).
3. Diagnose with the checklist below, quoting the exact lines.
4. Draft only the sections that need to change.
5. Run `ai-tells.md` on the draft.
6. Return: the diff (old → new), why, and the campaigns the user said use this prompt. If the user isn't sure, say the change hits every campaign using it.
7. The Commander, after an explicit yes, calls `update_prompt` with **only** the changed sections. To change any follow-up, send the **whole** `followUps` array (copy the others from `get_prompt`); `[]` clears them.
8. Recommend testing in the dashboard Sandbox (interested, skeptical, busy and technical prospects).

## Diagnosis checklist

- **First message pitches.** It should open a conversation: one specific observation, one easy question, short.
- **No clear mission.** Say what success is (a reply? a call? a referral?) and when to propose a meeting.
- **No boundaries.** What the AI must never claim, promise or discuss (prices it doesn't know, fake customers).
- **No objection handling.** "Not now", "we already use X", "send info", "who are you?".
- **Language.** Should the AI answer in the prospect's language? Say so.
- **Follow-ups repeat the first message.** Each needs a new reason, with several days between them.
- **Missing `{{customData}}`** when the team uses research.
- **Missing mandatory variables** → `update_prompt` fails with `missing_mandatory_prompt_variables`.
- **Length.** The stored prompt must stay under 80,000 characters. Shorter, clearer rules are followed better.

## Output

```
# Prompt review: [name] ([promptId]) ([date])
Used by campaigns (confirmed by user): [names]
Type: Messages · Model: [model]

## Diagnosis (quoted lines)
## Proposed changes
### context
--- old
--- new
### firstMessage …
### followUps (full array) …
## Why this should help
## Test in Sandbox: [3-4 scenarios]
```

## Guardrails

- Never remove a mandatory variable.
- Never invent proof, customers or numbers. Use `[NEED: proof]` and ask.
- Don't change the model unless asked; then pick from `list_models`.
- One prompt change at a time per campaign, so results stay readable.
- You draft; you never call `update_prompt` yourself.
