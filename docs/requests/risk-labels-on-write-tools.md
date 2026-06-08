---
id: request-risk-labels-on-write-tool-declarations
title: "Request: risk labels on write-tool declarations"
status: request
version: 26.608.1635
tags: [ sb-core, manifest, safety, request ]
---

# Request: risk labels on write-tool declarations

**Origin.** github-inside-claude-code classifies every write op as
low / medium / high. AI clients can then auto-approve low, prompt on
medium, require explicit confirmation on high. SB today has a binary
`write_*` permission gate — coarser-grained.

## What

Add `risk_level` to `manifest.ToolDecl`:

```go
type ToolDecl struct {
    Name        string
    Description string
    InputSchema map[string]any
    RiskLevel   string `json:"risk_level,omitempty"` // "low" | "medium" | "high"
}
```

Empty (default) = read-only. Set on every write tool per the matrix:

| Risk | Examples |
|---|---|
| `low` | files.create_dir, files.append, clipboard.write, comments, labels |
| `medium` | files.move, files.edit (non-destructive), file.write (overwrite), config.set, db.query (UPDATE) |
| `high` | files.delete, db.query (DROP/TRUNCATE), docker.rm, github.pr_merge, git.push --force |

## Propagation

Update every existing plugin's manifest to tag their write tools.
Backward-compatible: clients that don't read `risk_level` still see
the permission gate.

## Surface

- `mcp.tools_list` returns risk_level per tool.
- AI client harnesses can implement risk-based confirm (auto-allow
  low, prompt high) per their policy.
- `sb stats` slices by risk to surface "you ran 50 high-risk ops
  this session" warnings.

## Acceptance

- Manifest validator accepts `risk_level: "low"|"medium"|"high"` and
  rejects other values.
- `mcp.tools_list` shows risk_level field.
- Audit log records risk_level alongside tool name.
- Every existing write tool in sb-files / sb-git / sb-config / sb-db /
  sb-docker / sb-unreal has a risk label assigned.

## Non-goals (defer)

- Per-call risk override (`risk_override` arg) — not needed v1.
- Dynamic risk derived from args (e.g. delete of /tmp/foo vs delete
  of /etc/passwd) — static labels only.
- Auto-policy generation (deny-list maintenance scripts).
