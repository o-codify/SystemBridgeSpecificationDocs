---
id: request-structured-error-codes-vocabulary-internal-errcodes
title: "Request: structured error codes vocabulary (`internal/errcodes`)"
status: request
version: 26.608.1635
tags: [ sb-core, errors, consistency, request ]
---

# Request: structured error codes vocabulary

**Origin.** Various AI client harnesses parse error text to decide
next-step ("retry?", "ask user?", "fall back?"). Each SB plugin emits
ad-hoc error shapes today. github-inside-claude-code standardises on
12 typed codes per their structured error pattern — same idea, our
needs are a bit different.

## What

A shared `internal/errcodes` package that defines a vocabulary
plus a constructor:

```go
package errcodes

type Code string

const (
    NotFound          Code = "not_found"
    PermissionDenied  Code = "permission_denied"
    AuthMissing       Code = "auth_missing"
    AuthInsufficient  Code = "auth_insufficient"
    AuthExpired       Code = "auth_expired"
    ValidationFailed  Code = "validation_failed"
    RateLimited       Code = "rate_limited"
    Network           Code = "network"
    Timeout           Code = "timeout"
    Conflict          Code = "conflict"
    InternalError     Code = "internal_error"
    UnsupportedOp     Code = "unsupported_op"
    UserCanceled      Code = "user_canceled"
    PluginUnavailable Code = "plugin_unavailable"
)

type Error struct {
    Code            Code   `json:"error_code"`
    MessageForUser  string `json:"message_for_user"`
    MessageForAI    string `json:"message_for_ai"`
    SuggestedAction string `json:"suggested_action,omitempty"`
    Recoverable     bool   `json:"recoverable"`
    Cause           string `json:"cause,omitempty"` // underlying err text, redacted
}

func New(code Code, ...) *Error
func (e *Error) ToMCPResult() *mcp.CallToolResult
```

## Why

- AI can switch on `error_code` instead of substring-matching error
  prose.
- `recoverable` flag tells the AI when to retry vs surface to user.
- `suggested_action` is action-oriented ("re-auth via `gh auth login`")
  not just descriptive.

## Migration

Every plugin gradually adopts: new tools use it from day one,
existing tools migrate when touched. **Don't** big-bang migrate
— keep existing ad-hoc errors working alongside.

## Acceptance

- Package exists with the 14 codes above.
- At least one tool in each of sb-files / sb-github (when shipped) /
  sb-db emits typed errors.
- Documentation lists every code with semantics + examples.
- Codes appear in audit log alongside the call record.

## Non-goals (defer)

- Mapping HTTP status codes from network calls (just `Network`
  category in v1).
- Internationalised `message_for_user` (English only).
- Auto-translation of legacy errors via heuristics.
