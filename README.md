# mcp-setup-template

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://makeapullrequest.com)

A [Claude Code](https://claude.com/claude-code) skill (and template kit) for authoring a new first-time MCP setup skill — following the proven pattern used by `ga4-mcp-setup`, `google-sheets-mcp-setup`, `google-ads-mcp-setup`, `pandadoc-mcp-setup`, and `slack-second-workspace-mcp-setup`.

This is the meta-skill. Use it when you need to wire a new MCP server into Claude in a way that's reproducible for other operators.

## Why this exists

Every new MCP server brings its own auth flow, its own install path, its own quirks. Operators connecting to a new MCP for the first time hit the same questions every time:

- Which OAuth flow am I supposed to use?
- Where does the config file live?
- Do I need a daemon, or does Claude spawn this?
- How do I verify it actually worked?

The pattern is the same across MCPs. Only the specifics change. This skill codifies the pattern, so any operator authoring a new `<service>-mcp-setup` skill produces something that looks and works the same as the others.

## What you get

A skill that:

- **Walks you through authoring** a new MCP setup skill, asking the right questions
- **Scaffolds the file structure** following the proven layout
- **Generates the SKILL.md** with the 8-step walkthrough pattern
- **Generates the configure.sh** with credential validation, plist installation, daemon startup
- **Generates the reference docs** for auth setup and daemon configuration
- **Generates the README** positioned for the new skill's audience

Plus reference docs you can read directly:

- The SKILL.md structure every setup skill follows
- MCP server transport types (stdio vs HTTP/SSE) and when to pick each
- Auth flow variants (OAuth desktop, web flow, API key, service account, PAT)
- Daemon patterns (launchd, systemd, Task Scheduler)
- Troubleshooting section structure (the section operators read most)

## Requirements

- [Claude Code](https://claude.com/claude-code)

That's it. The skill produces a new public repo; you don't need anything extra installed.

## Install

```bash
git clone https://github.com/nickyc1/mcp-setup-template.git ~/.claude/skills/mcp-setup-template
```

Restart Claude Code. The skill becomes available.

## Usage

In Claude Code, say:

```
Create an MCP setup skill for Pinecone.
```

Claude reads `SKILL.md` and walks you through:

1. Gathering facts about the Pinecone MCP server (package name, auth type, transport)
2. Scaffolding the `pinecone-mcp-setup` repo structure
3. Filling in the SKILL.md with the 8-step pattern
4. Writing the auth reference doc
5. Writing the daemon reference (if applicable)
6. Writing the configure.sh
7. Writing the README
8. Adding CONTRIBUTING / LICENSE / .gitignore / badges

Output: a complete new skill repo, ready to push to GitHub.

## The 8-step pattern

Every MCP setup skill walks operators through some subset of these:

1. **Prereq check** — confirm API access, account, dev tier
2. **Auth setup** — OAuth client / API key / service account creation
3. **Token exchange** — convert auth into the credential the server expects
4. **Install** — pipx, npm, or binary install of the MCP server
5. **Config file** — write the credential to disk
6. **Daemon (optional)** — install a launchd plist or systemd unit
7. **Claude registration** — add the server to settings.json
8. **Verify** — run a known-good query

Most setups need all 8. Some skip the daemon (stdio-only servers). A few combine steps.

## Reference implementation

[`google-ads-mcp-setup`](https://github.com/nickyc1/google-ads-mcp-setup) was authored with this pattern. Open it to see the proportions in practice — SKILL.md length, references depth, the configure.sh shape.

## Related skills

| Skill | Relationship |
|---|---|
| [`google-ads-mcp-setup`](https://github.com/nickyc1/google-ads-mcp-setup) | A reference implementation following this pattern |
| [`marketing-stack`](https://github.com/nickyc1/marketing-stack) | The Tools section lists MCPs you might want to write setup skills for |
| [`paid-ads-context`](https://github.com/nickyc1/paid-ads-context) | Section 2 (Ad accounts & access) is what new MCP setup skills feed into |
| [`n8n-recipes`](https://github.com/nickyc1/n8n-recipes) | If your new MCP is for an n8n-orchestratable service, recipe docs may emerge alongside |

## Repo structure

```
mcp-setup-template/
├── SKILL.md                              # the skill prompt Claude Code reads
├── references/
│   ├── skill-md-pattern.md              # SKILL.md structure every setup skill follows
│   ├── mcp-server-types.md              # stdio vs HTTP/SSE
│   ├── auth-flow-variants.md            # OAuth, API key, service account, PAT
│   ├── daemon-patterns.md               # launchd, systemd, Task Scheduler
│   └── troubleshooting-pattern.md       # how to write the troubleshooting section
├── templates/
│   ├── SKILL.md.template                # fill-in-the-blank skill prompt
│   ├── README.md.template               # public-facing README
│   ├── configure.sh.template            # the configure script skeleton
│   └── daemon-macos.md.template         # launchd reference template
└── README.md
```

## License

MIT — see [LICENSE](LICENSE).

Built by [Nick Christensen](https://github.com/nickyc1).
