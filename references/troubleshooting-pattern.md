# Troubleshooting Pattern

The section operators read most. The section authors most often skip.

## The format

A table. Three columns. Five rows minimum.

```markdown
| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` from the MCP server | Refresh token expired or revoked | Run `google-oauthlib-tool` to mint a new refresh token, update `~/.google-ads.yaml` |
| Claude doesn't see the MCP tools | `command` path in settings.json wrong | Verify path with `which run-mcp-server`, paste into settings.json |
| Daemon fails to start | `ProgramArguments` path not absolute | Replace `$HOME` with the full path, reload plist |
| `503` from Google Ads API | API not enabled in GCP project | Console → APIs & Services → enable "Google Ads API" |
| Refresh token revoked after 7 days | OAuth client in "Testing" mode | Move OAuth consent screen to "In production" in GCP console |
```

## What separates a good troubleshooting section from a bad one

| Bad | Good |
|---|---|
| "Check your configuration" | "Verify `~/.google-ads.yaml` has `developer_token`, `client_id`, `client_secret`, `refresh_token`, and `login_customer_id` all set" |
| "Restart Claude Code" | "Quit Claude Code (cmd-Q), check `ps aux \| grep claude` for stuck processes, kill any zombies, then relaunch" |
| "Auth error" | "`401 Unauthorized` returned by the MCP server when calling `execute_gaql`" |
| "Server won't start" | "`launchctl list` shows the service with a non-zero exit code; logs at `~/Library/Logs/...` show `ImportError: No module named...`" |

Specificity in the symptom (the operator can match what they're seeing) + specificity in the fix (the operator can act on it).

## Writing process

For each entry, answer three questions:

1. **What does the operator literally see?** Exact error message, exact UI state, exact log line.
2. **What's the root cause?** Not "auth problem." The specific root cause.
3. **What three commands or actions fix it?** Not "fix your config." The exact steps.

If you can't answer all three, your entry isn't ready. Either dig deeper or leave it out.

## Common categories every MCP setup hits

The five entries every setup skill should have, by category:

### 1. Auth failure on first call

The operator finishes setup, runs the verify step, and gets a 401 or equivalent. Cause is almost always:

- Token format incorrect in config file (whitespace, quote chars, line endings)
- API not enabled in cloud project
- Token scopes too narrow
- Wrong account in dev console vs the one the operator wants to query

### 2. MCP tools don't appear in Claude

Operator's verify step never gets to the API — the MCP tools just aren't there. Cause:

- settings.json malformed (JSON parse error silently fails)
- `command` path doesn't exist or isn't executable
- Wrong settings.json file edited (`.claude/settings.json` vs `~/.claude/settings.json`)
- Claude Code not restarted after the edit

### 3. Daemon won't start

Operator installs the plist/unit, daemon fails. Cause:

- Path inside the plist uses `$HOME` or `~` (these don't expand)
- Binary not on the system PATH inside the daemon environment
- Permission denied on log directory
- Port already in use (HTTP/SSE MCPs)

### 4. Refresh token expired

Worked for days, suddenly fails. Cause:

- OAuth client in "Testing" mode (Google revokes tokens every 7 days)
- User manually revoked access in their account settings
- Operator changed Google password (some OAuth flows invalidate on password change)
- Token rotation policy on the OAuth client

### 5. Wrong account / scope

Operator can authenticate but can't see the data they expect. Cause:

- For Google Ads: `login_customer_id` (MCC) wrong, or operator authed with the wrong user
- For service accounts: service account doesn't have IAM permission on the target resource
- For OAuth: scopes too narrow, need to re-auth with broader scopes
- For multi-tenant SaaS: wrong workspace/team/org selected

## What to leave OUT

Things that are NOT troubleshooting:

- General "MCP architecture" explainers (those go in the SKILL.md body or references)
- Configuration tips for the MCP's own functionality (that's the MCP's job)
- Pro-tips for advanced usage (those go in a separate "Advanced" section)

Troubleshooting is for "I did the steps and it's not working." Keep it focused.

## Length

5 rows minimum. 15-20 rows is the sweet spot. Above 30, split into categorized sub-tables (Auth, Config, Daemon, etc.).

## Maintenance

When someone opens a GitHub issue against your MCP setup skill, the answer goes into the troubleshooting table. Over time, the table reflects every actual problem operators have hit. That's the goal.

Quarterly review: is anything in the table no longer relevant (the MCP fixed the bug, the cloud provider changed the UI)? Remove or update.
