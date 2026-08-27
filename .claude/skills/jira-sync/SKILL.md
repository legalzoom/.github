---
name: jira-sync
description: >-
  Create and maintain tickets on the EAI (Enterprise Analytics & Insights) Jira
  board at legalzoom.atlassian.net, following the team's mandate that every
  ticket has a filled-out Task or Bug template, labels, priority, an assignee, a
  due date, and an End Date on completion. Use this skill whenever the user
  wants to file, write, open, or clean up a Jira ticket, task, bug, or issue for
  the EAI board or project; mentions "EAI-" followed by a number; asks to move a
  ticket to In Progress / Needs Review / Done or close one out; asks to link a
  ticket to a GitHub branch or PR; asks whether some piece of work even deserves
  a ticket; or asks to audit their own tickets for missing required fields. Also
  use it when the user describes analytics or data work they just finished or
  are about to start and wants it tracked, even if they never say the word
  "Jira". The board is shared with the rest of the team, so this skill only ever
  writes to the user's own tickets.
---

# EAI Jira board sync

This skill covers the EAI board — project key `EAI`, "Enterprise Analytics &
Insights", a team-managed Jira project. Board:
`https://legalzoom.atlassian.net/jira/software/projects/EAI/boards/3601`

The point of this skill is that a ticket on this board is a real unit of
committed work, not a log entry. The team's mandate is that tickets exist for
work with deliverables — so the first job is deciding whether to create one at
all, and the second is filling it out completely enough that someone reading it
in three months knows what was done and why.

## Connecting

Pass `legalzoom.atlassian.net` as `cloudId` to the Atlassian MCP tools. It
resolves to the site directly, which avoids hardcoding the site UUID in a file
that lives in a public repository.

Resolve the user's own account once per session with `atlassianUserInfo` and use
the returned `account_id` as `assignee_account_id`. New tickets are self-assigned
by default. That keeps an account ID out of this file, and it means the skill
follows whoever is running it rather than a hardcoded name.

## Scope: the user's own tickets only

The EAI board is shared — most tickets on it belong to other people. The user
manages their own work on it and nobody else's, so **every write is scoped to
tickets assigned to them.**

Concretely:

- **Create** — self-assign. If the work really belongs to someone else, say so
  and let the user decide; don't quietly assign a ticket to a teammate.
- **Edit, transition, close, relabel, re-prioritize, set dates** — only where
  `assignee = currentUser()`. Before the first write to a ticket the user
  referred to by key, check the assignee, because a mistyped key (`EAI-4`
  vs `EAI-41`) can easily land on a colleague's ticket.
- **Read** — unrestricted. Reading the whole board is how the skill learns the
  label vocabulary and finds related work, and is worth doing.

When the user asks for something that would write to a ticket that isn't
theirs, don't just refuse — tell them whose it is and offer what you *can* do:
a comment on the ticket, or a summary they can pass along. Editing a
colleague's ticket is the kind of thing that's technically permitted by Jira
and genuinely unwelcome in practice, which is exactly why it needs a rule
rather than judgement in the moment.

The one exception is a ticket the user reported but assigned to someone else —
they have a legitimate claim to its description and priority. Confirm before
writing to it rather than assuming.

If a ticket needs to be assigned to a different person, look them up with
`lookupJiraAccountId` — but treat handing work to someone else as the user's
call to make explicitly, not a default.

## Step 1: Does this deserve a ticket?

Apply this gate before touching Jira, and say which way it went. Creating
tickets for everything is what the mandate is reacting against — a board full
of five-minute favors makes the real commitments invisible.

Create a ticket when the work has a **deliverable** — something that exists
afterward and can be pointed at. A new or changed dashboard, model, pipeline,
table, or report. A bug with a business consequence. An investigation that
produces a written answer someone will act on.

Do **not** create a ticket for:

- Ongoing support, office hours, "keeping the lights on"
- Ad hoc requests that take a few minutes — a one-off query, a permission
  grant, re-running a failed job, answering a question in Slack
- Recurring routine with no change to it (a report that just runs)
- Work with no artifact at the end: a meeting, a status update, a code review

When it's genuinely borderline, the useful question is: *would anyone ever want
to look this up later?* If not, it's support, not a ticket. Tell the user
you're skipping it and why — one sentence, no lecture. If they want it filed
anyway, that's their call; file it.

If a stream of small requests keeps coming from the same source, the right move
is often one ticket for the pattern ("Reduce ad hoc LTV pulls by publishing a
self-serve view"), which has a deliverable, rather than a ticket per request.

## Step 2: Task or Bug

Both types exist on the board and take the same fields, but they get different
description templates.

- **Bug** — something already built is behaving wrong. Use when there's a
  correct behavior it's failing to meet.
- **Task** — everything else: new work, changes, investigations, migrations.

`Epic` and `Subtask` also exist. Use an Epic only when grouping several
tickets under a program the user names; use Subtask only when the user asks to
break an existing ticket down.

## Step 3: Draft the ticket, then confirm before creating

Read `references/templates.md` and fill in every section of the right template.
Every ticket gets a fully filled-out description — that's the part of the
mandate that carries the most weight, because it's the part that's useless to
reconstruct later.

Write from what the user actually told you. Where a section has no real content
yet, don't pad it with restatements of the summary — ask. The one exception is
Background on a Task and Root Cause on a Bug: Background is explicitly
optional, and Root Cause can honestly say "Not yet known — to be investigated"
when it isn't.

The summary line should read as an outcome, not a topic. "Fix Calendly tables"
is a topic; "IB / Calendly LCM campaign Hex fails after Calendly table revamp"
is an outcome. Keep it under ~90 characters so it doesn't get truncated on the
board.

Then set the metadata. `references/field-reference.md` has the exact field IDs
and allowed values — read it before the first write of a session, because two
of them are easy to get wrong in ways Jira won't reject:

- **Priority** — set it by **ID**, not name. The site has both `P3 - Medium`
  (id `3`) and a stray legacy `P3` (id `10037`) that looks identical in a
  request but sorts and reports differently. The board default is
  `P5 - Lowest`, so priority left unset silently becomes the lowest.
- **End Date** — `customfield_13766`, a separate field from Jira's built-in
  `Start date` and `Due date`.

- **Labels** — at least one, reused from the board's existing vocabulary rather
  than invented. The board already has inconsistent casing (`marketing` vs
  `Reporting`) so match an existing label exactly rather than normalizing it,
  and only coin a new one when nothing fits.
- **Due date** — `duedate`, an estimate, always set at creation. Most tickets
  on this board have no due date, which is what makes the board hard to plan
  against. Propose one from the scope the user described and let them correct
  it; a wrong estimate is more useful than an empty field.
- **End Date** — left empty at creation. It's the record of when work actually
  finished and is filled in at close (Step 5).

Show the user the drafted ticket — summary, type, description, labels,
priority, assignee, due date — and get a yes before creating it. Creating a
Jira ticket is visible to the whole team and annoying to unwind, and the
description is exactly the kind of thing that benefits from a glance. Skip the
confirmation only if the user has already said to just file it.

Create with `createJiraIssue`: `summary`, `issueTypeName`, `description`
(markdown is fine — `contentFormat` defaults to it), `assignee_account_id`, and
everything else in `additional_fields`. Report the ticket key and its
`https://legalzoom.atlassian.net/browse/EAI-NN` URL.

## Step 4: Link the GitHub PR

When a ticket has code behind it, the link is made by putting the ticket key in
the **branch name**. The GitHub-for-Jira integration scans branch names, commit
messages, and PR titles for anything matching `EAI-<number>` and attaches what
it finds to the ticket's Development panel — so the naming convention is the
mechanism, not just tidiness.

LegalZoom's convention (Confluence *GIT Branch Naming*, space `DEV`, page
`1573421132`) is:

```
{branchtype}/{TICKET-KEY}_{short-description}
```

```
feature/EAI-42_selfserve_ltv_view      # new work
bug/EAI-43_calendly_table_rename       # fixing something broken
chore/EAI-44_bump_dbt_version          # config, deps, tooling
refactor/EAI-45_split_clickstream_dag  # restructuring existing code
```

Match the branch type to the ticket type — `bug/` for a Bug, `feature/` for
most Tasks. Put the key in the PR title too (`[EAI-42] Add self-serve LTV
view`), since that survives a squash-merge into the commit history where the
branch name doesn't.

**Verify before promising it worked.** The Development panel on EAI tickets is
currently empty across the board (`customfield_10000` reads `{}`), and the
panel shows "Connect development tools" — so the GitHub app does not appear to
be connected for this project's repositories yet. Branch naming alone won't
produce a link until it is. After a PR exists, check the ticket with
`getJiraIssue` on `customfield_10000`; if it's still `{}`, fall back to posting
the PR URL as a comment with `addCommentToJiraIssue` so the trail exists, and
tell the user the integration looks unconnected rather than reporting a link
that isn't there. `references/github-linking.md` has the details and how to get
it switched on.

## Step 5: Keep it current, and close it out properly

The board's four statuses are `To Do`, `In Progress`, `Needs Review`, `Done`.
Transition IDs are in `references/field-reference.md`. `Needs Review` is unused
so far but is the natural home for a ticket whose PR is open and awaiting
review.

Check the assignee before transitioning a ticket the user named by key — moving
a colleague's ticket to Done is a conspicuous thing to do by accident.

When work finishes, closing out is **two** operations, and the second is the
one that gets forgotten:

1. Transition to `Done` (transition id `41`)
2. Set `End Date` (`customfield_13766`) to the date work actually finished

Jira does not populate End Date on its own — this is the "manually entered once
completed" step in the mandate, and it's why the field exists alongside the
status. Use the real completion date, not today's date, if they differ. When
transitioning a ticket to Done and the user hasn't given a date, ask rather
than assuming — or set today's and say so, so they can correct it.

## Auditing the user's tickets

When asked to check for gaps against the mandate, query with
`searchJiraIssuesUsingJql`. Every audit query carries
`assignee = currentUser()` — without it these return the whole team's tickets,
which is both noise and a temptation to "fix" work that isn't the user's.

```sql
-- Done but no End Date: the close-out that got missed
project = EAI AND assignee = currentUser() AND status = Done
  AND "End Date" IS EMPTY

-- No due date: unplannable
project = EAI AND assignee = currentUser() AND statusCategory != Done
  AND duedate IS EMPTY

-- Missing labels, or on the wrong-priority trap
project = EAI AND assignee = currentUser()
  AND (labels IS EMPTY OR priority IN ("P3", "P1", "P5"))
```

That last one catches tickets on the legacy bare-name priorities that should be
the `P<n> - <Name>` variants instead. Fetch the description too and note
tickets that don't follow either template.

Propose the fixes as a batch for the user to approve, then apply them — they're
the user's own tickets, but a sweep of silent edits is still unpleasant to
discover after the fact.

Dropping `assignee = currentUser()` is reasonable when the user explicitly
asks how the *board* is doing overall — a read-only question. Report what you
find and stop there; don't follow a board-wide audit with fixes.

## Reference files

- `references/templates.md` — the Task and Bug description templates, with a
  worked example of each. Read before drafting any ticket.
- `references/field-reference.md` — field IDs, priority and transition IDs,
  the board's existing labels, and the API call shapes. Read before the first
  write in a session.
- `references/github-linking.md` — branch naming, how the Development panel
  gets populated, and what to do while the integration is unconnected.
