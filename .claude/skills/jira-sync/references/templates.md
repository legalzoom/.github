# EAI ticket description templates

Two templates: Task and Bug. Every ticket on the board gets one, filled out in
full. Use the section headings verbatim — consistent headings are what make the
board skimmable and let the audit queries in SKILL.md spot tickets that skipped
the format.

Write these as markdown (`##` headings). `createJiraIssue` defaults to
`contentFormat: "markdown"` and Jira converts it to its own rich-text format on
the way in, so there's no need to hand-build ADF JSON.

---

## Task template

```markdown
## Description

Summarize the new task in a way that succinctly answers the question,
"What are we going to do?"

## Background

Optional context to help others understand the work.

## Requirements

The specific functional or technical requirements the work must satisfy.

## Success Criteria

A short checklist that will be used to confirm that the task was successfully
completed.
```

Section notes:

- **Description** — one or two sentences answering "what are we going to do?"
  Not the background, not the reason, just the work.
- **Background** — the only optional section. Include it when someone would
  otherwise ask "why now?" or "who asked for this?" Drop the heading entirely
  if there's nothing to say; an empty heading is worse than no heading.
- **Requirements** — what the work must satisfy. Bullets. This is where
  specifics live: which tables, which grain, which metrics, which environments.
- **Success Criteria** — a checklist (`- [ ]`) of verifiable outcomes, not a
  restatement of the requirements. The test is whether someone else could read
  it and confirm the ticket is done without asking you.

### Worked example

Summary: `Publish self-serve LTV view to cut ad hoc marketing pulls`

```markdown
## Description

Build and publish a governed LTV view in Snowflake so marketing can pull
cohort LTV themselves instead of requesting one-off extracts.

## Background

Marketing has requested ad hoc LTV cuts repeatedly this quarter. Each one is a
short query, but the volume is significant and the definitions drift between
requests, so numbers presented in different decks disagree.

## Requirements

- Model LTV at customer-cohort grain, monthly cohorts, trailing 24 months
- Use the metrics-store LTV definition as the single source of truth
- Expose the view to the marketing analyst role
- Document the column definitions and the refresh cadence

## Success Criteria

- [ ] View exists and refreshes daily without manual intervention
- [ ] Marketing analysts can query it with their existing role
- [ ] Output reconciles to the metrics-store LTV within 1%
- [ ] Column definitions documented and linked from the ticket
```

---

## Bug template

```markdown
## Issue Description

A summary description of the issue and clearly explain what is not working
like it should be.

## Root Cause

Include information about the root cause if it is known.

## Business Impact

Describe how the issue affects the business to help determine priority, for
example impacts to processes, systems, or reports.

## Desired Behavior

Explain what the correct behavior should look like when this issue is resolved.
```

Section notes:

- **Issue Description** — what is broken and how it shows up. Include where it
  surfaces (which Hex app, dashboard, table, job) and when it started.
- **Root Cause** — write `Not yet known — to be investigated.` when it isn't
  known. Saying so honestly is useful; guessing is not. Come back and fill it
  in once diagnosed, because this is the section that pays off when the same
  failure recurs.
- **Business Impact** — this is what drives the priority, so be concrete about
  who is affected and what decisions are running on bad numbers. "Report is
  wrong" doesn't justify a priority; "the WBR ran on understated LTV for two
  weeks" does.
- **Desired Behavior** — the correct end state, specific enough to verify
  against.

### Worked example

Summary: `IB / Calendly LCM campaign Hex fails after Calendly table revamp`

```markdown
## Issue Description

The IB / Calendly Performance Template in the LCM Campaign Hex app began
failing on the morning of Aug 26 with query errors. The newly released
Calendly pipeline renamed several backend tables, so the app's queries no
longer resolve.

## Root Cause

The Calendly pipeline revamp renamed source tables and changed the join logic
for campaign attribution. The Hex app's queries reference the old table names
and assume the previous attribution logic.

## Business Impact

Campaign performance reporting for the LCM Calendly program is unavailable.
Marketing cannot assess in-bound campaign performance for the current cycle,
which delays spend decisions on the program.

## Desired Behavior

The Hex app runs on schedule against the revamped Calendly tables and returns
campaign performance matching the new attribution logic, with historical
figures consistent with the pre-revamp reporting.
```

---

## Filling these in honestly

The templates are a prompt for thinking, not a form to satisfy. Two failure
modes to avoid, both of which produce a ticket that looks complete and tells a
reader nothing:

1. **Restating the summary in every section.** If Description, Requirements,
   and Success Criteria all say the same thing in different words, the ticket
   hasn't been thought through — ask the user for the specifics instead.
2. **Inventing detail.** Don't fabricate a business impact, a root cause, or a
   due date to fill space. Ask, or mark it explicitly unknown.
