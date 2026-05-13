# Daemon Patterns

When (and how) to run the MCP server as a persistent daemon.

## When you need a daemon

Daemon required if:

- The MCP server is HTTP/SSE transport (server must be running before Claude connects)
- The server has expensive startup (loaded indexes, opened DB connections, warmed caches)
- You want the MCP available to multiple clients (Claude Code + Claude Desktop + n8n)

Daemon NOT required if:

- The MCP is stdio transport and Claude spawns it per-session
- Server startup is < 500ms and you don't mind paying it each session
- You only ever use the MCP from one Claude client

## macOS: launchd

The standard. User-level daemons go in `~/Library/LaunchAgents/`.

### Plist structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.<service>-mcp</string>

    <key>ProgramArguments</key>
    <array>
        <string>ABSOLUTE_PATH_TO_BINARY</string>
        <string>--flag</string>
        <string>value</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
        <key>API_KEY_OR_CONFIG_PATH</key>
        <string>VALUE</string>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/USERNAME/Library/Logs/<service>-mcp.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/USERNAME/Library/Logs/<service>-mcp.log</string>
</dict>
</plist>
```

### Key flags

| Key | Purpose |
|---|---|
| `Label` | Reverse-DNS identifier, must be unique on the system |
| `ProgramArguments` | Argv. First entry is the binary path. MUST be absolute (no `$HOME` expansion). |
| `EnvironmentVariables` | Env vars passed to the process. PATH is the easy-to-miss one. |
| `RunAtLoad` | true = start when launchd loads the plist (on login) |
| `KeepAlive` | true = restart if the process dies |
| `StandardOutPath` / `StandardErrorPath` | Logs. Without these, debugging is painful. |

### Common commands

```bash
# Install / start (since macOS 11+)
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/com.<service>-mcp.plist

# Check status
launchctl list | grep <service>

# Stop / uninstall
launchctl bootout gui/$UID ~/Library/LaunchAgents/com.<service>-mcp.plist

# Force-reload after editing the plist
launchctl bootout gui/$UID ~/Library/LaunchAgents/com.<service>-mcp.plist
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/com.<service>-mcp.plist

# Watch the logs
tail -f ~/Library/Logs/<service>-mcp.log
```

### macOS gotchas

- **PATH is not inherited.** Always set `PATH` explicitly in `EnvironmentVariables`.
- **$HOME doesn't expand in plists.** Use absolute paths everywhere.
- **`KeepAlive: true` is aggressive.** If your server has a startup bug, launchd will restart it in a tight loop. For initial setup, use `false`. Flip to `true` after verifying stable startup.
- **Permissions.** If the binary needs Tailscale/Network/etc access, the launchd-spawned process inherits the user's permissions, not the user's keychain access. Some services prompt; others fail silently.

## Linux: systemd user units

User-level systemd units go in `~/.config/systemd/user/`.

### Unit structure

```ini
[Unit]
Description=<Service> MCP Server
After=network.target

[Service]
Type=simple
ExecStart=ABSOLUTE_PATH_TO_BINARY --flag value
Restart=on-failure
RestartSec=5
Environment="API_KEY_OR_CONFIG_PATH=VALUE"
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
StandardOutput=append:/home/USERNAME/.local/state/<service>-mcp.log
StandardError=append:/home/USERNAME/.local/state/<service>-mcp.log

[Install]
WantedBy=default.target
```

### Common commands

```bash
# Reload after editing
systemctl --user daemon-reload

# Start / enable on login
systemctl --user enable --now <service>-mcp.service

# Check status
systemctl --user status <service>-mcp.service

# Stop
systemctl --user stop <service>-mcp.service

# Disable
systemctl --user disable <service>-mcp.service

# Watch the logs
journalctl --user -u <service>-mcp.service -f
```

### Linux gotchas

- **User units don't run if the user isn't logged in.** Enable lingering: `loginctl enable-linger USERNAME` for headless servers.
- **`Restart=on-failure` is the right choice.** `always` is too aggressive; `no` is too lazy.

## Windows: Task Scheduler (the rough one)

If you must support Windows, use Task Scheduler with a startup trigger.

```powershell
$action = New-ScheduledTaskAction -Execute "C:\Path\To\binary.exe" -Argument "--flag value"
$trigger = New-ScheduledTaskTrigger -AtLogon
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable
Register-ScheduledTask -TaskName "<service>-mcp" -Action $action -Trigger $trigger -Settings $settings -User $env:USERNAME
```

Restart-on-crash is harder on Windows. Practical advice: document Windows as "supported but lower priority" and ship macOS + Linux first.

## What your setup skill should ship

For each platform you support:

- A reference doc (`references/daemon-macos.md`, `references/daemon-linux.md`)
- The plist / unit file content with placeholders
- The install commands
- The verification command
- The uninstall commands
- Common gotchas specific to that platform

For your `configure.sh`:

- Detect the platform (`uname -s`)
- Install the appropriate plist / unit
- Start the daemon
- Verify it's running before exiting

The script template at `templates/configure.sh.template` shows the pattern.

## When to defer daemon support

If you don't want to deal with daemons in v1 of your setup skill, that's fine. Ship as stdio-only and add a note:

> This skill installs the MCP server in stdio mode. Claude Code spawns the server per-session. For warm-start (faster reconnects) or use from multiple Claude clients, see `references/daemon-macos.md` for the daemon upgrade — coming in v2.

Shipping is better than perfect. v2 can add the daemon when operators ask.
