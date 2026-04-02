# runaway-killer

A lightweight macOS daemon that finds and kills orphaned processes draining your battery. Built for developers using AI coding tools (Claude Code, Cursor, Windsurf, Copilot, etc.) where background MCP servers frequently outlive their parent sessions.

## The problem

AI coding tools spawn MCP (Model Context Protocol) server processes to extend their capabilities. When you close a session, these servers often keep running — silently eating CPU in the background. On a laptop, a handful of orphaned MCP servers can cut your battery life in half.

The same happens with other tools: Wine/CrossOver leaves behind `winedevice.exe` processes, Electron apps leave orphaned helpers, and so on.

## What it does

Runs every 5 minutes via launchd and kills processes that meet **structural criteria** — no hardcoded process names or fragile heuristics:

**MCP server detection** (3 independent signals):
1. Process parent is `npm exec` or `npx` (stdio MCP servers)
2. Process runs from an npm/npx cache path (`~/.npm/_npx/`, `node_modules/.bin/`)
3. Process is a direct child of a known AI coding tool and has "mcp" or "server" in its path

**Kill criteria** (both require evidence of abandonment):
- **Broken pipe** — the process's stdin pipe has no live reader on the other end (the host app closed)
- **Excess duplicates** — more than 3 instances of the same MCP server under the same parent

**Runaway process cleanup:**
- Kills orphaned Wine processes (`winedevice`, `wine64-preloader`, `wineserver`) only when CrossOver is not running

### Safety features

- Sends SIGTERM first, waits 0.5s, then SIGKILL only if needed
- Verifies process identity before killing (guards against PID recycling)
- Skips the expensive `lsof` scan when there are no candidates (most runs complete in <0.1s)
- Lockfile prevents concurrent runs
- Log rotation keeps the log file from growing unbounded

## Install

```sh
# Download
curl -o ~/.local/bin/runaway-killer https://raw.githubusercontent.com/radu2lupu/runaway-killer/main/runaway-killer
chmod +x ~/.local/bin/runaway-killer

# Install the launchd agent (runs every 5 minutes)
runaway-killer install
```

Or clone and install:

```sh
git clone https://github.com/radu2lupu/runaway-killer.git
cd runaway-killer
cp runaway-killer ~/.local/bin/
runaway-killer install
```

## Uninstall

```sh
runaway-killer uninstall
rm ~/.local/bin/runaway-killer
```

## Usage

```sh
runaway-killer                # Run once
runaway-killer --dry-run      # Preview what would be killed
runaway-killer --verbose      # Show detailed detection info
runaway-killer --dry-run -v   # Preview with full details
runaway-killer install        # Install launchd agent
runaway-killer uninstall      # Remove launchd agent
runaway-killer --version      # Show version
```

## Configuration

All configuration is via environment variables. Set them in your shell profile or override them when running manually.

| Variable | Default | Description |
|---|---|---|
| `RUNAWAY_MAX_PER_TYPE` | `3` | Max live instances per MCP server type before extras are killed |
| `RUNAWAY_EXTRA_HOSTS` | — | Additional AI host patterns (space-separated, grep -E regex) |
| `RUNAWAY_LOG` | `/tmp/runaway-killer.log` | Log file path |
| `RUNAWAY_DRY_RUN` | `0` | Set to `1` to preview without killing |
| `RUNAWAY_VERBOSE` | `0` | Set to `1` for detailed logging |

### Adding custom AI host patterns

If you use an AI coding tool that isn't detected by default, add it:

```sh
export RUNAWAY_EXTRA_HOSTS="MyTool\\.app /my-custom-ai"
```

## Supported AI coding tools

Detected out of the box:
- Claude Code / Claude Desktop
- Cursor
- Windsurf
- GitHub Copilot (via Codex)
- Zed
- Codeium
- Aide
- Continue

## Logs

```sh
tail -f /tmp/runaway-killer.log
```

The log auto-rotates at 5000 lines.

## Requirements

- macOS (uses launchd, lsof, and zsh)
- zsh (default shell on macOS since Catalina)

## License

MIT
