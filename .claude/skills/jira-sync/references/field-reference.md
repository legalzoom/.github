# EAI field reference

Verified against the project's create metadata. `EAI` is a **team-managed**
(next-gen) project, project id `12875`, on `legalzoom.atlassian.net`.

## Contents

- [Issue types](#issue-types)
- [Fields](#fields)
- [Priority — read this before setting it](#priority--read-this-before-setting-it)
- [Labels](#labels)
- [Statuses and transitions](#statuses-and-transitions)
- [Call shapes](#call-shapes)

## Issue types

| Type | id | Use |
|---|---|---|
| Task | `12412` | Default for new work, changes, investigations |
| Bug | `12413` | Something built is behaving incorrectly |
| Epic | `12411` | Grouping tickets under a named program |
| Subtask | `12410` | Breaking down an existing ticket |

`createJiraIssue` takes `issueTypeName` (`"Task"`, `"Bug"`), so the ids matter
only when reading metadata back.

## Fields

Task and Bug expose an **identical** field set — there are no Bug-specific
fields, which is why both templates live in `description`.

| Field | Key | Type | Notes |
|---|---|---|---|
| Summary | `summary` | string | Required |
| Description | `description` | rich text | Template goes here; markdown accepted |
| Assignee | `assignee` | user | `assignee_account_id` on create |
| Priority | `priority` | option | Set by id — see below |
| Labels | `labels` | string[] | Free-form; reuse existing values |
| Due date | `duedate` | date | `YYYY-MM-DD`. Estimate, set at creation |
| **End Date** | **`customfield_13766`** | date | `YYYY-MM-DD`. Set manually at close |
| Start date | `customfield_10015` | date | Jira's built-in; not part of the mandate |
| Development | `customfield_10000` | read-only | GitHub dev panel; `{}` when unconnected |
| Parent | `parent` | issuelink | For subtasks / epic membership |
| Flagged | `customfield_10021` | option[] | Only value is `Impediment` |
| Linked Issues | `issuelinks` | issuelinks[] | `createIssueLink` to relate tickets |

Note there is no Story Points, Sprint, or Components field on this project, and
`fixVersions` has no configured values — don't try to set them.

The three date fields are distinct and easy to confuse. `duedate` is the
estimate made up front; `customfield_13766` (End Date) is the record of actual
completion; `customfield_10015` (Start date) is unused by the team's mandate.

## Priority — read this before setting it

The site carries two overlapping priority schemes, and the legacy one will be
accepted silently if you set priority by name. **Set priority by id.**

| Use these | id | | Not these | id |
|---|---|---|---|---|
| `P1 - Highest` | `1` | | `P1` | `10039` |
| `P2 - High` | `2` | | `P3` | `10037` |
| `P3 - Medium` | `3` | | `P5` | `10038` |
| `P4 - Low` | `4` | | `Urgent` | `10004` |
| `P5 - Lowest` | `5` | | `High` / `Medium` / `Low` | `10003`/`10001`/`10002` |
| | | | `No priority` | `10000` |

```json
{"priority": {"id": "3"}}
```

Two things to know:

- **The project default is `P5 - Lowest`.** A ticket created without an explicit
  priority is not "unprioritized", it's lowest — so always set it.
- The mandate uses P1 / P3 / P5 as the working scale (highest / medium /
  lowest). P2 and P4 exist and are in use on the board; prefer the three-point
  scale unless the user asks for a finer distinction.

Rough guide: **P1** — something in production is broken or a committed deadline
is at risk. **P3** — normal committed work. **P5** — worth doing, nothing
breaks if it slips.

## Labels

At least one label per ticket. Reuse from what's already on the board rather
than coining new ones — the vocabulary is already inconsistent in casing and
separators, and adding variants makes filtering worse.

Match an existing label **exactly**, including its casing:

| Label | In use |
|---|---|
| `marketing` | 10 |
| `Reporting` | 8 |
| `ltv` | 8 |
| `analytics_engineering` | 4 |
| `clickstream` | 3 |
| `Sales-Data` | 2 |
| `data_science` | 2 |
| `partnerships` | 2 |
| `LegalPlans` | 1 |
| `difm` | 1 |
| `guardian_wbr` | 1 |
| `hex` | 1 |
| `invoca` | 1 |
| `metrics_store` | 1 |

Most tickets carry two: a domain (`marketing`, `partnerships`, `ltv`) and a
kind-of-work or system (`Reporting`, `hex`, `analytics_engineering`). Refresh
this list with `searchJiraIssuesUsingJql` on `project = EAI` if it looks stale.

Don't "fix" the casing of an existing label — `Reporting` and `reporting` would
become two labels, which is the actual harm.

## Statuses and transitions

| Status | Status id | Transition id | Category |
|---|---|---|---|
| To Do | `14038` | `11` | To Do |
| In Progress | `14039` | `21` | In Progress |
| Needs Review | `14040` | `31` | In Progress |
| Done | `14041` | `41` | Done |

All four transitions are global — any status can go to any other, and none has
a transition screen, so no fields are required to move a ticket.

`Needs Review` is unused on the board so far. It's the right status for a
ticket whose PR is open and waiting on a reviewer.

There is no resolution field to set, and no automation setting End Date on the
Done transition — hence the two-step close-out.

## Call shapes

Create:

```json
{
  "cloudId": "legalzoom.atlassian.net",
  "projectKey": "EAI",
  "issueTypeName": "Task",
  "summary": "Publish self-serve LTV view to cut ad hoc marketing pulls",
  "description": "## Description\n\n...",
  "assignee_account_id": "<from atlassianUserInfo>",
  "additional_fields": {
    "priority": {"id": "3"},
    "labels": ["ltv", "marketing"],
    "duedate": "2026-09-15"
  }
}
```

Everything that isn't summary / type / description / assignee goes in
`additional_fields` — including priority, labels, and dates.

Close out (two calls, both needed):

```json
// 1. transitionJiraIssue
{"cloudId": "legalzoom.atlassian.net", "issueIdOrKey": "EAI-42",
 "transition": {"id": "41"}}

// 2. editJiraIssue
{"cloudId": "legalzoom.atlassian.net", "issueIdOrKey": "EAI-42",
 "fields": {"customfield_13766": "2026-09-12"}}
```

`editJiraIssue` can also set End Date and transition-independent fields in one
call if the ticket is already Done. To clear a date, pass an explicit `null`.

Reading End Date back requires asking for it — the default field set for
`getJiraIssue` and `searchJiraIssuesUsingJql` does not include custom fields:

```json
{"fields": ["summary", "status", "duedate", "customfield_13766", "labels",
            "priority", "assignee"]}
```

In JQL the field is quoted by name: `"End Date" IS EMPTY`.
