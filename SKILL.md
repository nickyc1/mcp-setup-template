---
name: mcp-setup-template
description: Author a first-time setup skill for any MCP server, following the proven pattern used by ga4-mcp-setup, google-sheets-mcp-setup, google-ads-mcp-setup, pandadoc-mcp-setup, and slack-second-workspace-mcp-setup. Use when asked to "create an MCP setup skill for [X]", "write a setup guide for [Y] MCP", "scaffold a new MCP onboarding skill", or anytime you need to wire a new MCP server into Claude in a way that's reproducible for other operators.
metadata:
  version: 1.0.0
---

# MCP Setup Template

The meta-skill. Use this to author a new first-time setup skill for any MCP server, following the proven pattern.

## When to use this skill

Three triggers:

1. **"Create an MCP setup skill for [X]"** — Claude scaffolds a new `<service>-mcp-setup` repo with SKILL.md, references, and a configure script
2. **"Walk me through writing an MCP setup skill"** — interactive authoring with you in the loop, answering questions as the structure is built
3. **"Audit this MCP setup skill"** — Claude evaluates an existing setup skill against the pattern and proposes improvements

## The pattern this codifies

The MCP setup skills Anthropic ships (ga4-mcp-setup, google-sheets-mcp-setup, pandadoc-mcp-setup, slack-second-workspace-mcp-setup) share a structure. So does [`google-ads-mcp-setup`](https://github.com/nickyc1/google-ads-mcp-setup). When operators follow the pattern, their setup skills work the same way every time and feel familiar to anyone who has run one before.

The pattern, in 8 steps:

```
1. Prereq check         — confirm the operator has the API access / account they need
2. Auth setup           — OAuth client / API key / service account creation
3. Token exchange       — convert auth into the credential the MCP server expects
4. Install              — pipx, npm, or binary install of the MCP server
5. Config file          — write the credential to disk in the server's expected format
6. Daemon (optional)    — install a launchd plist (macOS) or systemd unit (Linux)
7. Claude registration  — add the MCP server to settings.json or `claude mcp add`
8. Verify               — ask Claude to run a known-good query against the server
```

Every MCP setup skill walks the operator through some subset of these 8. Most need all 8. A few skip the daemon (when the MCP is stdio-only and Claude Code spawns it directly).

## Required file layout

A new `<service>-mcp-setup` repo should look like this:

```
<service>-mcp-setup/
├── SKILL.md                              # the skill prompt Claude Code reads
├── README.md                             # public-facing pitch + install + related skills
├── CONTRIBUTING.md                       # standard contribution guide
├── LICENSE                               # MIT
├── .gitignore                            # always exclude credential files
├── references/
│   ├── auth-setup.md                     # OAuth / API key / service account creation
│   └── daemon-<platform>.md              # launchd plist / systemd unit
└── scripts/
    └── configure.sh                      # writes config + installs daemon
```

Names are stable across the family. An operator who has run one setup skill already knows what to expect from the next.

## Authoring workflow

When invoked, the skill walks the operator through:

### Step 1: Gather facts about the MCP server

The skill asks (or reads from the MCP's documentation):

| Question | Why it matters |
|---|---|
| What's the MCP server's package name? | Drives the install step |
| What auth does it use? (OAuth desktop / OAuth web / API key / service account / token) | Determines section structure |
| What credential format does the server expect? (YAML / .env / JSON / env vars) | Drives the configure.sh |
| Is the server stdio or HTTP/SSE? | Determines whether a daemon is needed |
| What's a known-good first query to verify the setup worked? | Drives the verify step |
| What's the GitHub repo or docs URL? | Used in references |

### Step 2: Scaffold the file structure

The skill creates the standard files from `templates/` with placeholders filled in.

### Step 3: Write the SKILL.md walkthrough

Each section is written conversationally, following the pattern in `references/skill-md-pattern.md`. The skill enforces:

- A `name:` and `description:` frontmatter with sensible triggers
- An 8-step walkthrough (or N-step if some steps don't apply)
- Explicit prerequisites at the top
- Specific commands the operator can copy
- A "things that go wrong" troubleshooting section at the bottom

### Step 4: Build the auth reference doc

Different auth types need different docs. The skill picks from:

- `references/auth-templates/oauth-desktop-client.md` — for OAuth desktop clients (Google, Microsoft)
- `references/auth-templates/oauth-web-flow.md` — for web flow OAuth (Slack, Notion)
- `references/auth-templates/api-key.md` — for simple API keys (most LLM and SaaS APIs)
- `references/auth-templates/service-account.md` — for GCP/AWS service accounts

### Step 5: Build the daemon reference doc

If the MCP server needs to run as a persistent daemon (not stdio-spawned), the skill creates:

- `references/daemon-macos.md` — launchd plist
- `references/daemon-linux.md` — systemd user unit (optional, can defer)

### Step 6: Write the configure.sh script

The script that:

1. Validates required env vars are set
2. Writes the credential file
3. Installs the daemon plist/unit
4. Starts the daemon
5. Verifies the daemon is running

Template at `templates/configure.sh.template`.

### Step 7: Write the README

Public-facing. Positions the skill for its audience. Always includes:

- Why this skill exists (the problem it solves)
- Prereqs
- Install command
- Usage example
- Related skills cross-references
- Repo structure

### Step 8: Set up boilerplate

CONTRIBUTING.md, LICENSE, .gitignore, README badges. Standard across the family.

## Conventions the skill enforces

1. **Skill name format:** `<service>-mcp-setup`. Always. Don't deviate.
2. **Frontmatter description:** must include the service name AND list 3-5 trigger phrases an operator might naturally use.
3. **Auth section first.** Operators get stuck on auth more than anything else. Lead with it.
4. **One section per step.** Don't combine. Don't fold "auth" and "config" into one section — the operator needs to navigate them separately.
5. **Explicit commands.** Every step has at least one command the operator can copy. No "configure your environment" without showing how.
6. **Troubleshooting section at the bottom.** Specific failures, specific fixes. Five entries minimum.
7. **Always include a verify step.** Operators need to know "did this work?" before they try to use the MCP for real work.
8. **Credentials are gitignored.** The `.gitignore` always includes the credential filename patterns.

## What the skill does NOT do

- It does NOT install MCP servers automatically. That's the produced skill's job.
- It does NOT replace reading the MCP server's own docs. The skill helps you translate those docs into a Claude-Code-installable artifact.
- It does NOT skip the human-in-the-loop steps. OAuth flows, dev token requests, account approvals — these need a real person.

## Anti-patterns

| Anti-pattern | Why | Fix |
|---|---|---|
| Skipping the verify step | Operator doesn't know if it worked until they try real work later | Always include a "run this exact query and you should see X" step |
| Hiding the auth complexity | Auth is the failure mode. Document it explicitly. | Put auth in its own reference doc, not in the main SKILL.md |
| Combining the SKILL.md with the README | The SKILL.md is for Claude. The README is for humans. | Separate them. SKILL.md is verbose and structural; README is short and pitchy. |
| Daemon assumed on macOS only | Linux users hit a wall | Note macOS-only OR ship a Linux daemon doc too |
| No troubleshooting section | Every operator hits something the docs don't cover | Five known-failures-with-fixes minimum |
| Credentials hardcoded in examples | Leak risk | Always use env vars in examples; never paste a real token |

## Related skills

| Skill | Relationship |
|---|---|
| [`google-ads-mcp-setup`](https://github.com/nickyc1/google-ads-mcp-setup) | A reference implementation — see how the pattern looks in practice |
| [`marketing-stack`](https://github.com/nickyc1/marketing-stack) | The Tools section lists MCPs you might want to write a setup skill for |
| [`paid-ads-context`](https://github.com/nickyc1/paid-ads-context) | Section 2 (Ad accounts & access) is what your MCP setup skill should connect into |
| [`n8n-recipes`](https://github.com/nickyc1/n8n-recipes) | If your new MCP is for an n8n-orchestratable service, recipe docs may emerge alongside the setup |

## References

- [`references/skill-md-pattern.md`](references/skill-md-pattern.md) — the SKILL.md structure every setup skill follows
- [`references/mcp-server-types.md`](references/mcp-server-types.md) — stdio vs HTTP/SSE, when to pick which
- [`references/auth-flow-variants.md`](references/auth-flow-variants.md) — OAuth desktop, web flow, API key, service account
- [`references/daemon-patterns.md`](references/daemon-patterns.md) — launchd, systemd, when daemons are needed
- [`references/troubleshooting-pattern.md`](references/troubleshooting-pattern.md) — how to write the troubleshooting section
- [`templates/SKILL.md.template`](templates/SKILL.md.template) — fill-in-the-blank skill prompt
- [`templates/README.md.template`](templates/README.md.template) — public-facing README template
- [`templates/configure.sh.template`](templates/configure.sh.template) — the configure script skeleton
- [`templates/daemon-macos.md.template`](templates/daemon-macos.md.template) — launchd reference template
