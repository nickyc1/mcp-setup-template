# MCP Server Types

stdio vs HTTP/SSE — when each is right, and how it affects your setup skill.

## stdio servers

The MCP server runs as a subprocess of Claude Code. Claude spawns it, talks to it over stdin/stdout, kills it when the session ends.

**Setup characteristic:** No daemon needed. The server only runs while Claude Code has a session open.

**Pros:**
- Simpler setup (no launchd / systemd)
- Server uses zero resources when Claude isn't running
- Credentials only loaded when needed
- Easiest for end users to install

**Cons:**
- Slow first-call latency (server has to start up each Claude session)
- Server can't be shared between multiple Claude clients
- If the server is heavyweight (long startup, big indexes), every session reboot pays the cost

**Examples:**
- `google-ads-mcp` runs as stdio when invoked directly, but the typical pattern uses a daemon for faster reconnects
- `mcp-google-sheets`
- Most filesystem/local-tool MCPs

**Setup skill structure:**

For a pure stdio server, your setup skill skips the daemon step. The Claude Code registration is:

```json
{
  "mcpServers": {
    "<service>": {
      "command": "/path/to/run-mcp-server",
      "args": ["--flag", "value"],
      "env": {
        "API_KEY": "...",
        "CONFIG_PATH": "..."
      }
    }
  }
}
```

## HTTP/SSE servers

The MCP server runs as a long-lived HTTP service. Claude Code connects to a URL.

**Setup characteristic:** Daemon required. The server must be running before Claude Code tries to connect.

**Pros:**
- Fast reconnect (server stays warm)
- Can serve multiple Claude clients (Code + Desktop + n8n + others)
- Better for servers with expensive initialization (loaded indexes, opened DB connections)
- Better for remote MCPs (server lives on a different machine)

**Cons:**
- Setup is more complex (daemon, network)
- Server is always running (resource usage)
- Credential lifecycle is longer

**Examples:**
- Hosted MCPs (Notion, Slack, the ones running on remote servers)
- Self-hosted MCPs you run on a Tailscale machine
- Some heavy-init MCPs (vector stores, large knowledge bases)

**Setup skill structure:**

Your setup skill MUST include the daemon step. The Claude Code registration is:

```json
{
  "mcpServers": {
    "<service>": {
      "url": "http://localhost:PORT/sse"
    }
  }
}
```

Or for `claude mcp add` syntax:

```bash
claude mcp add --transport sse <service> http://localhost:PORT/sse
```

## When to pick which

This decision is usually made by the MCP server itself — the server is designed for one or the other. But when you have a choice (some servers support both modes):

| Situation | Pick |
|---|---|
| Server starts in < 500ms | stdio |
| Server takes 2-10s to start | HTTP/SSE (avoid per-session startup tax) |
| You'll only use it from Claude Code on one machine | stdio |
| You'll use it from Code, Desktop, AND n8n | HTTP/SSE |
| Server is on a different machine (Tailscale, internal network) | HTTP/SSE |
| Server holds expensive state (DB connections, loaded models) | HTTP/SSE |
| Simplicity matters more than latency | stdio |

## Hybrid pattern

Some servers (including `google-ads-mcp`) ship a `run-mcp-server` binary that can be invoked as both stdio AND HTTP/SSE. The setup pattern:

1. Install the package
2. Configure auth and credentials
3. Run as HTTP/SSE under launchd/systemd for warm-start
4. Register with Claude as HTTP/SSE pointing to localhost

This is the pattern `google-ads-mcp-setup` uses. It gives you the warm-start speed of HTTP/SSE with the simplicity of "the daemon runs on this Mac, Claude connects to localhost."

## How this affects your setup skill

Decide the transport up front. Then:

- **stdio:** skip the daemon step. Your `configure.sh` writes the credential file; Claude Code spawns the server itself.
- **HTTP/SSE:** include the daemon step. Your `configure.sh` writes the credential, installs the launchd/systemd unit, starts the daemon.
- **Hybrid:** default to the HTTP/SSE pattern. Note in the skill that operators can run stdio-direct if they want to skip the daemon (advanced).

Document the choice in the SKILL.md upfront. Operators want to know what they're committing to.
