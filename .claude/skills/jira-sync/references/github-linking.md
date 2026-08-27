# Linking EAI tickets to GitHub branches and PRs

## How the link is actually made

There is no manual "attach this PR" action on a Jira ticket. The link is made
by **naming**: the GitHub-for-Jira app watches the connected repositories and
scans for anything matching an issue key pattern (`EAI-42`, case-insensitive)
in three places:

1. **Branch names** — picked up when the branch is pushed
2. **Commit messages** — picked up on push
3. **Pull request titles** (and the PR body)

Anything it matches gets attached to that ticket's **Development** panel, which
is the `customfield_10000` field on the issue. The key does not have to be at
the start of the string, and one branch or PR can reference several tickets.

This is why the branch naming convention matters beyond tidiness — it's the
integration's only input.

## The convention

LegalZoom's standard, from the Confluence page *GIT Branch Naming*
(space `DEV`, page `1573421132`):

```
{branchtype}/{story-id}_{story-or-task-description}
```

The ticket key and the description are separated by an **underscore**; the
description is a short slug of what changed.

```
feature/EAI-42_selfserve_ltv_view
bug/EAI-43_calendly_table_rename
chore/EAI-44_bump_dbt_version
refactor/EAI-45_split_clickstream_dag
```

Branch types, and the flow each belongs to:

| Type | Cut from | Merges to | For |
|---|---|---|---|
| `feature/` | develop | develop | New work — the default for a Task |
| `bug/` | develop | develop | Fixing something broken in develop — the default for a Bug |
| `hotfix/` | master | master | A production fix |
| `release/` | develop | master | Release candidate |
| `refactor/` | develop | develop | Restructuring existing code |
| `chore/` | develop | develop | Config, dependency bumps, tooling, build |

Match the branch type to the ticket type: a Bug ticket gets `bug/` (or
`hotfix/` if it's live in production), a Task gets `feature/` unless it's
plainly a chore or a refactor.

Repositories vary in whether they use a develop/master flow or trunk-based
`main`. The prefix convention holds either way; check the repo's own default
branch rather than assuming `develop` exists.

## Put the key in the PR title too

```
[EAI-42] Publish self-serve LTV view
```

Belt and braces, for a practical reason: many repos squash-merge, and a squash
discards the branch name while keeping the PR title in the commit message. If
the key only ever lived in the branch name, the association disappears from
history once the branch is deleted.

## Verify — don't assume the link landed

**Current state of this project: the integration does not appear to be
connected.** Every EAI ticket checked returns `customfield_10000` as `{}`, and
the ticket's Development panel shows the "Connect development tools" empty
state rather than branch or PR entries.

So branch naming alone will *not* produce a visible link on an EAI ticket right
now. Naming branches correctly is still the right thing to do — it's what makes
the links appear retroactively once the app is connected, since the integration
backfills the repository history it can see.

After a PR exists, check:

```json
// getJiraIssue
{"cloudId": "legalzoom.atlassian.net", "issueIdOrKey": "EAI-42",
 "fields": ["customfield_10000"]}
```

- Returns branch/PR detail → the link is live, say so.
- Returns `{}` → don't report a link that doesn't exist. Post the PR URL as a
  comment so the trail exists, and tell the user the integration looks
  unconnected.

```json
// addCommentToJiraIssue
{"cloudId": "legalzoom.atlassian.net", "issueIdOrKey": "EAI-42",
 "commentBody": "PR: https://github.com/legalzoom/<repo>/pull/123"}
```

A comment is a weaker link than the Development panel — it isn't queryable in
JQL and won't drive any automation — but it beats an untraceable ticket, and
it's visible to reviewers.

## Getting the integration switched on

Turning this on is a one-time admin action, outside what this skill does:

1. The **GitHub for Jira** app must be installed on the `legalzoom.atlassian.net`
   site (Jira admin) and connected to the `legalzoom` GitHub organization
   (requires GitHub org owner permission).
2. The specific repositories the team works in must be selected in the app's
   configuration — connecting the org does not automatically include every
   repo.
3. In **Project settings → Features** for EAI, development information must be
   enabled.

Once connected, the app backfills existing commits and PRs, so correctly named
branches created before the connection will still show up. If the user wants
this pursued, the path is a Jira admin plus a GitHub org owner — worth flagging
rather than attempting.

## Optional: smart commits

With the integration connected, commit messages can also drive Jira from git:

```
EAI-42 #comment reconciled LTV to metrics store
EAI-42 #time 2h
EAI-42 #done
```

Smart commits are a separate toggle from branch linking and are off unless
enabled. Note that `#done` moves the status but **does not** set End Date
(`customfield_13766`) — that close-out step stays manual either way.
