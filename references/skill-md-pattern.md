# SKILL.md Pattern

The structure every `<service>-mcp-setup` SKILL.md follows. Deviate only with reason.

## Frontmatter

```yaml
---
name: <service>-mcp-setup
description: First-time setup guide for the <Service> MCP server (<package-name>). Walks through <auth type>, <token exchange>, <install>, <config>, <daemon>, and Claude Code registration. Works with both Claude Code and Claude Desktop. Run once, then <high-value capability the MCP unlocks>.
metadata:
  version: 1.0.0
---
```

Rules:

- `name` is always `<service>-mcp-setup`. No exceptions.
- `description` starts with "First-time setup guide for..." so triggers fire on common phrasings
- `description` mentions the package name in parens so Claude can route to the right install command
- `description` ends with the high-value capability — the WHY of doing the setup

## Section 1: What this skill does

One paragraph. Restates the description for humans. Mentions the package, the auth flow, the resulting capability.

Example:

> Connect Claude (Code or Desktop) to your Google Ads manager account (MCC) via a local MCP server. Once set up, Claude can run any GAQL query, pull spend/ROAS/conversions, mine search terms, optimize budgets, and manage every sub-account under your MCC from natural language.

## Section 2: Prerequisites

Bulleted list of everything the operator needs BEFORE starting. Be ruthless about specificity:

- Specific OS (macOS X.Y+ if relevant)
- Specific Claude version (Code, Desktop, both)
- External accounts (with link to where to create one)
- Developer-tier access if required (with note on approval timing)
- Local tools (`pipx`, `npm`, `gh` CLI, Homebrew)

Example:

```markdown
## Prerequisites

- macOS (the daemon setup is launchd-based)
- [Claude Code](https://claude.com/claude-code) or [Claude Desktop](https://claude.ai/download)
- A Google Cloud Platform account
- A Google Ads developer token — request from your Google Ads account (Basic access approves within ~24 hours)
- A Google Ads MCC account with at least one linked sub-account
- `pipx` (`brew install pipx`)
- Python 3.10+
```

## Section 3: The walkthrough

The 8-step pattern, one section per step. Each step has:

- A header (`## Step 1: <verb> the <thing>`)
- A 1-2 sentence preamble explaining why this step
- Either inline commands or a link to a reference doc that goes deeper
- A confirmation line ("If this worked, you should see X")

Sections that always exist:

```
## Step 1: Create the <auth> credential
## Step 2: Get the <secondary token> (if applicable)
## Step 3: Install the MCP server
## Step 4: Generate the refresh / access token
## Step 5: Write the config file
## Step 6: Install the daemon (if applicable)
## Step 7: Register with Claude Code / Desktop
## Step 8: Verify
```

For steps that have non-trivial detail (OAuth client creation, daemon setup), use a one-paragraph summary in the SKILL.md and link out to a reference doc for the full walkthrough.

## Section 4: Verify

The most-skipped section. Make it explicit.

```markdown
## Step 8: Verify

In Claude Code, say:

> "List my accessible <service> accounts."

You should see a list of accounts. If you see auth errors, jump to Troubleshooting below.
```

Always pick a query that exercises auth + read access without modifying anything. Listing accounts is the universal choice.

## Section 5: Use with the rest of the stack

A short section explaining what this MCP setup enables. Examples:

> Once this is set up, you can use:
> - [`google-ads-manager`](https://github.com/nickyc1/google-ads-manager) — weekly review, search-term mining, budget reallocation
> - [`paid-channel-recap`](https://github.com/nickyc1/paid-channel-recap) — stakeholder-facing performance dashboards

This is also the "Related skills" table from the SKILL.md pattern.

## Section 6: Troubleshooting

A table with 5+ rows minimum. Format:

```markdown
| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` from the MCP server | Refresh token expired | Re-run `google-oauthlib-tool` to get a new one, update yaml |
| Claude doesn't see the MCP tools | settings.json path wrong | Verify the `command` path points to your pipx-installed Python |
| Daemon fails to start | `RUN_MCP_SERVER_PATH` not absolute | Use `$(which run-mcp-server)` output, paste the absolute path |
```

Specific symptoms. Specific causes. Specific fixes. No "check your config" generics.

## Section 7: Uninstall

The forgotten section. Include it.

```markdown
## Uninstall

1. Stop the daemon: `launchctl bootout gui/$UID ~/Library/LaunchAgents/com.<service>-mcp.plist`
2. Remove the daemon: `rm ~/Library/LaunchAgents/com.<service>-mcp.plist`
3. Remove the config: `rm ~/.<service>.yaml`
4. Uninstall the package: `pipx uninstall <package>`
5. Remove from Claude: edit `.claude/settings.json`, delete the `mcpServers.<service>` block
```

Operators sometimes need to clean up. Don't make them figure it out from scratch.

## Total length

A good MCP setup SKILL.md is 6,000-15,000 characters. Below 4,000 you're under-specifying. Above 20,000 you're padding — extract sections into reference docs.

`google-ads-mcp-setup/SKILL.md` is the reference implementation. Check its proportions.
