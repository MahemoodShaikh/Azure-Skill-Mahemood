---
name: ticket-actions-and-followup
description: Pulls every active Azure DevOps work item assigned to the current user in the myQuorum Cloud project, sorts them into "needs your input" vs "waiting on someone else," flags anything with no activity in 12+ hours (including linked tickets handed off to Infra/DB/DevOps/Data Services teams), and — after explicit confirmation — posts a follow-up comment on the stale ones. Use this whenever someone asks to check their assigned tickets, wants a status sweep of their open incidents, asks "what needs my attention," "what's stale," "chase down my open tickets," "follow up on anything that hasn't moved," or similar. Also trigger for "check my tickets and the ones I handed off to infra/DB" or any request combining "my tickets" + "follow up" + Azure DevOps. This is distinct from ado-ticket-intelligence (single-ticket deep analysis) and ado-unassigned-incident-scan (unassigned queue triage) — this skill is specifically about the current user's own assigned workload and its downstream dependencies.
---

# Ticket Actions and Followup

Daily/on-demand sweep of a person's own Azure DevOps workload: what needs them right now, what's waiting on someone else, and what's gone quiet long enough to need a nudge. Built from a real triage session — the steps below mirror what actually worked, including a couple of gotchas that will silently give wrong answers if skipped.

## Why this exists

On a ticket queue, "assigned to me" is not the same as "needs me." Half the tickets are sitting in a state like *Additional Information Required* because someone else owes a reply, not you. And a ticket's `System.ChangedDate` field is frequently touched by bulk/automated updates (board moves, bot syncs) that have nothing to do with human activity — using it to judge staleness will flag fresh tickets as stale and miss genuinely dead ones. The fix for both problems is the same: read the actual comment history, not just the state and the changed-date field.

## Step 1 — Identify who "my tickets" means

Call `mcp__2534a4c6-5e6c-4b21-85d1-6a3f79065a2a__get_me` (or the equivalent M365 identity tool available in the session) to get the current user's `mail` field. This is what makes the skill work for any teammate without editing anything — never hardcode an email address in this workflow. If no M365 identity tool is available, ask the person for their myQuorum Cloud email once.

## Step 2 — Pull active assigned work items

Run a WIQL query (`mcp__azure-devops__wit_query`, action `wiql`) rather than the `wit_work_item` action `my` — the latter has been unreliable (returns a client-elicitation error in some environments).

```
SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.ChangedDate], [System.AreaPath]
FROM WorkItems
WHERE [System.TeamProject] = 'myQuorum Cloud'
  AND [System.AssignedTo] = '<their email>'
  AND [System.State] NOT IN ('Closed','Resolved','Removed','Done')
ORDER BY [System.ChangedDate] ASC
```

Then batch-fetch full details for the returned IDs with `wit_work_item` action `get_batch`, requesting at minimum: `System.Title`, `System.State`, `System.ChangedDate`, `System.CreatedDate`, `System.AreaPath`, `Microsoft.VSTS.Common.Priority`, `System.Description`, `Custom.AITriage`. That last one feeds the recommended-action step below — skip it silently if the field doesn't exist on a given work item type instead of erroring.

Get the current time (`date -u +"%Y-%m-%dT%H:%M:%SZ"` via bash) once at the start — every staleness calculation below is relative to this single timestamp, not to whenever each individual check happens to run.

## Step 3 — Read comments to find out whose court the ball is in

For every ticket, pull the last 5 comments (`wit_work_item` action `list_comments`, `top: 5`). This is the part that actually determines the categorization — the state field alone is not enough:

- **Needs your input** — the most recent comment is a *new reply or finding* (from a customer, teammate, or automated system) that nobody has acted on yet, regardless of what `System.State` says. A state of "Additional Information Required" does NOT automatically mean "no action needed" — check who's actually being asked to do something next. Also anything in an active working state (not an "additional info" state) is implicitly on you to move forward.
- **No action required right now** — the most recent comment is *you* (or your team) asking a question or requesting something, and nobody has replied yet. There's nothing to do until they respond.
- Note the timestamp of that last comment specifically — this is what "stale" is measured against, not `System.ChangedDate`.

Watch for tickets where the customer has explicitly flagged something as critical/urgent in a comment but the ticket's `Priority` field hasn't been updated to match — call this out even if it's not stale, since it's a mismatch worth surfacing on its own.

### Work out a recommended action for every "needs your input" ticket

A bare "this needs you" isn't enough to be useful — the report should tell the person what to actually go do next. For each ticket landing in "needs your input," write one specific, concrete recommendation (a sentence or two, not a vague "follow up"), built from whatever's available:

- **The `Custom.AITriage` field**, if the work item has one — this environment auto-generates a triage note per ticket that usually already contains a "Recommended first response" and an escalation assessment. Pull this field when fetching ticket details (add it to the `get_batch`/`get` field list) and lean on it as a starting point rather than re-deriving root cause from scratch — but always sanity-check it against the actual last comment, since the AI triage note can be a few comments out of date.
- **The most recent comment's specific ask** — if a teammate just reported "43 processes still stuck at QIMPEXP" or provided IPs/credentials that were requested, the recommendation is the literal next step implied by that (e.g. "clear the stuck processes" or "verify these IPs against the firewall ACL"), not a generic "review and respond."
- **Any explicit customer request still unactioned** — e.g. a customer asking for the ticket to be marked Critical, or asking for an ETA — surface that as the recommended action even if it's a small field change rather than investigation work.

Keep each recommendation to one line so the report stays scannable; if the reasoning is longer, summarize it and note where the detail lives (which comment, or the AITriage field).

## Step 4 — Check linked tickets on other teams

Tickets frequently get split off to Infra, Data Services/DB, or DevOps as a separate linked work item while the original stays assigned to the requester. These linked tickets are easy to lose track of because they don't show up in "my tickets" once handed off — but the person still owns following up on them.

Re-fetch each ticket from Step 2 with `expand: "Relations"` to get its actual `relations` array (don't rely on IDs merely mentioned in comment text — cross-check against the real link graph). For every `System.LinkTypes.Related` (or parent/child) link:
1. Batch-fetch state, area path, assigned-to, and `ChangedDate` for the linked IDs.
2. Skip anything already `Closed`/`Resolved`/`Removed` — no action needed on closed history.
3. For anything still open, pull its last 5 comments the same way as Step 3, and measure staleness from the last actual comment.

## Step 5 — Compute staleness and present the report

A ticket (or linked ticket) is stale if the last human comment is 12+ hours before the "now" timestamp from Step 2. Present three groups, each as a table with WI #, title, state, and hours-since-last-activity:

1. **Needs your input** — action required, not necessarily stale yet. Add a **Recommended action** column here using the guidance above — this is the group where the person is about to go do something, so tell them what.
2. **No action required** — waiting on someone else's reply, nothing to do.
3. **Stale (12h+, candidate for follow-up)** — split into "your own tickets" and "linked tickets on other teams," since the latter need to name the linked WI # and which team owns it.

Don't silently skip the "Priority mismatch" flag from Step 3 — mention it even if that ticket isn't stale.

## Step 6 — Confirm before posting anything

Never post a follow-up comment without showing the person the draft text first and getting an explicit go-ahead. These comments land on tickets that customers and other teams can see — a wrong or premature nudge is a lot more visible than a wrong read of a report. Use a single batched confirmation (e.g. via a yes/no/edit-first choice) rather than one prompt per ticket if there are several — but always show every draft before anything is sent.

Once confirmed, post with `wit_work_item_comment_write`, action `add`, format `Markdown`. Keep the comment short and specific: reference what's actually being waited on (not a generic "any update?"), and for linked tickets, mention which parent ticket is blocked on it so the other team has context.

## Notes for teammates running this themselves

- This only reads/writes Azure DevOps — no calendar, email, or Teams access required.
- Nothing gets posted without your sign-off, every time you run it.
- If your queue is large, it's fine to run Steps 1–5 as a quick status check and stop there without ever getting to Step 6 — the report alone is useful on its own.
