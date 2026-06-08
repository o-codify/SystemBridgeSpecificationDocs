---
id: request-sb-github-github-plugin-pr-issue-ci-release-search
title: "Request: sb-github — GitHub plugin (PR / issue / CI / release / search)"
status: request
version: 26.608.1635
tags: [ sb-core, github, git, request ]
---

# Request: sb-github — GitHub operations as MCP tools

**Origin.** Comparative scan against `github-inside-claude-code`
(PyPI). They expose 88 specific `gh_*` MCP tools — each a single
operation with predictable JSON. SB today has `sb-git` (local repo
ops) but no dedicated GitHub-side plugin; AI shells out to `gh` for
PR / issue / CI / release / search constantly.

## What

Mirror their structure with specific-tool-per-op (not one
parameterised tool). 30-40 most-used ops in v1.

## Tools (grouped)

### PR (10)

| Tool | Purpose |
|---|---|
| `github_pr_list` | List PRs in a repo with filter (open / closed / merged). |
| `github_pr_read` | One PR with title, body, status, reviewers, checks. |
| `github_pr_create` | Create PR from branch. |
| `github_pr_merge` | Merge / squash / rebase. **Risk: high.** |
| `github_pr_close` | Close without merge. |
| `github_pr_review` | Add review (comment / approve / request_changes). |
| `github_pr_comment` | Add a PR-level comment. |
| `github_pr_files` | Files changed in the PR with diffstat. |
| `github_pr_diff` | Full unified diff. |
| `github_pr_checks` | CI / Actions check status. |

### Issue (7)

`github_issue_list / read / create / close / reopen / comment / search`.

### Repo (6)

`github_repo_read / list / create / branches / tags / topics`.

### CI / Actions (6)

`github_workflow_list / run_list / run_view / run_logs / run_rerun /
run_cancel / workflow_dispatch`.

### Release (4)

`github_release_list / view / create / upload_asset`.

### Search (5)

`github_search_code / issues / prs / users / repos`.

### Session (3)

`github_whoami / rate_limit / scopes_check`.

## Implementation route

Go plugin `cmd/sb-github/`. Two strategies:

- **(a) gh CLI wrapper** — fast ship. Shell out to `gh` (single-shot
  subprocess), parse structured output where `gh` supports
  `--json` flags. Bundles a 0-line dep, just needs `gh` on PATH.
- **(b) go-github SDK** — cleaner but needs auth wiring (PAT or
  device flow). More work upfront.

Going with **(a)** for v1 — `gh` already on most dev boxes; existing
auth state reused. Move to (b) if specific operations need richer
filtering.

## Risk labels (write tools)

| Risk | Operations |
|---|---|
| `low` | comment, label add, branch create |
| `medium` | close, reopen, draft conversion |
| `high` | merge, force-push, release create with asset, delete |

Surfaced in tool description as `**Risk: <level>**`. (Once the
manifest gets a structured `risk_level` field — separate request —
this moves to the metadata.)

## Structured errors

When the wrapped `gh` returns non-zero, the plugin maps stderr to
typed error codes:

```
auth_missing / auth_insufficient_scope / auth_expired
not_found / permission_denied / validation_failed
merge_conflict / branch_protection
rate_limited / network / user_cancelled / unknown
```

Each result: `{error_code, message_for_user, message_for_claude,
suggested_action, recoverable}`.

## Acceptance

- `github_pr_list("o-codify/SystemBridge", state="open")` returns the
  same set `gh pr list` shows, structured as `[{number, title, head,
  base, status, author, draft, mergeable}]`.
- `github_pr_create` from a feature branch creates a PR and returns
  its URL + number.
- `github_run_logs` returns the last 100 lines of a workflow run.
- `auth_missing` is returned (not raw stderr) when `gh auth status`
  fails.

## Non-goals (defer)

- Self-hosted GitHub Enterprise auth wiring (works via `gh` default).
- Org-wide audit tools.
- Notifications / mentions feed.
- GraphQL-only ops that need a hand-written query.
