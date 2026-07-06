# Plugsky CLI — User Guide

## Overview

Plugsky CLI is a terminal-native AI coding agent powered by the Plugsky API. It reads, edits, and runs commands in your codebase — one command or an interactive REPL.

**One command to start:**

```sh
plugsky "explain this project"
```

---

## Installation

### macOS / Linux (curl)

```sh
curl -fsSL https://plugsky.com/install | sh
```

This downloads the latest binary for your OS/arch to `~/.plugsky/bin/plugsky` and adds it to your `PATH` in `.bashrc` / `.zshrc`.

### From source

```sh
git clone https://github.com/coinplannet/plugskyCLI.git
cd plugskyCLI
bun install
bun run build
./dist/index.js --version
```

### Environment variables

| Variable | Overrides |
|---|---|
| `PLUGSKY_API_KEY` | API key (takes priority over `~/.plugsky/auth.json`) |
| `PLUGSKY_BASE_URL` | API base URL (default: `https://api.plugsky.com/v1`) |
| `PLUGSKY_MODEL` | Default model (default: `auto`) |
| `PLUGSKY_LOG` | Log level: `debug`, `info`, `warn`, `error` |

---

## Authentication

```sh
plugsky login
```

You'll be prompted for your API key (`sk-live-...`). Input is masked — not echoed to the terminal.

The key is stored in `~/.plugsky/auth.json` (mode 0600).

To log out:

```sh
rm ~/.plugsky/auth.json
```

---

## Usage modes

### 1. One-shot agentic task

```sh
plugsky "add error handling to src/api.ts"
```

The CLI creates a session, streams the model's response, and lets it call tools (read/write/edit files, run shell commands) with approval prompts.

### 2. Interactive TUI (REPL)

```sh
plugsky
```

Opens an Ink-based terminal UI with streaming responses, tool call display, and slash commands.

**Slash commands:**

| Command | Description |
|---|---|
| `/model <id>` | Switch model (e.g. `/model plugsky-pro`) |
| `/mode <mode>` | Set approval mode: `suggest`, `auto-edit`, `full-auto` |
| `/clear` | Clear conversation |
| `/sessions` | List and resume past sessions |
| `/usage` | Show token usage |
| `/mcp` | List connected MCP servers |
| `/help` | Show this list |
| `/exit` | Quit |

### 3. Plain chat (no tools)

```sh
plugsky chat "explain the visitor pattern"
```

Streams a single reply with no tool access.

### 4. Pipe mode

```sh
cat package.json | plugsky "summarize the dependencies"
```

Standard input is prepended to the prompt.

---

## Commands

| Command | Description |
|---|---|
| `login` | Authenticate with your Plugsky API key |
| `run <task>` | One-shot agentic task (default) |
| `chat <msg>` | Plain streaming reply (no tools) |
| `models` | List the Plugsky ladder + all available models |
| `usage` | Show token usage and cost |
| `image <prompt>` | Generate an image (saves to `plugsky-image.png`) |
| `speech <text>` | Text-to-speech to `plugsky-speech.mp3` |
| `transcribe <file>` | Transcribe an audio file |
| `moderate <text>` | Run a moderation check |
| `mcp <sub>` | List or test MCP servers |
| `index` | Build/refresh the RAG index for this project |
| `config get\|set` | Read or write configuration |
| `update` | Re-run the install script to get the latest version |

### Useful flags

| Flag | Applies to | Effect |
|---|---|---|
| `-m, --model <id>` | run, chat | Override model (`auto`, `plugsky-micro`, etc.) |
| `-y, --yes` | run | Full-auto mode (no prompts) |
| `--auto-edit` | run | Auto-approve file edits, prompt for shell |
| `--cwd <dir>` | run | Working directory |
| `--resume` | run | Resume the latest session in this directory |
| `-o <file>` | image, speech | Output file path |

---

## Approval modes

The CLI has three approval levels that control when the AI can modify files or run commands:

| Mode | File edits | Shell commands | Use case |
|---|---|---|---|
| `suggest` (default) | Prompts y/N | Prompts y/N | Safe — review every action |
| `auto-edit` | Auto-approved | Prompts y/N | Fast edits, careful with shell |
| `full-auto` | Auto-approved | Auto-approved | Dangerous — full autonomy |

Set via flag: `plugsky --auto-edit "fix the typo in README"` or in the TUI: `/mode auto-edit`.

In the TUI, approve with `y` + Enter, reject with `n` + Enter.

---

## Model ladder

| Model | Tier | Best for |
|---|---|---|
| `plugsky-micro` | 1 | Quick edits, simple Q&A (cheapest) |
| `plugsky-minimax` | 2 | Everyday coding, balanced |
| `plugsky-pro` | 3 | Complex refactors, reasoning |
| `plugsky-frontier` | 4 | Hardest tasks, deep reasoning |
| `auto` (default) | — | Smart routing via `/v1/plugsky/route` — picks the right tier per request |

```sh
plugsky models          # see the ladder + all available models
plugsky -m plugsky-pro "refactor this module"
```

---

## Configuration

Layered: **global** ← **project** ← **environment**.

**Global:** `~/.plugsky/config.json`
**Project:** `.plugsky/config.json` or `plugsky.json` in the working directory

```sh
plugsky config get                        # dump the merged config
plugsky config set defaultModel plugsky-pro  # set a value
```

| Key | Type | Default | Description |
|---|---|---|---|
| `baseUrl` | string | `https://api.plugsky.com/v1` | API base URL |
| `defaultModel` | string | `auto` | Model to use |
| `approvalMode` | string | `suggest` | `suggest`, `auto-edit`, or `full-auto` |
| `maxTurns` | number | `25` | Max agent loop iterations |
| `temperature` | number | `0.2` | Response temperature (0–2) |
| `showUsage` | boolean | `true` | Show token counts after responses |
| `telemetry` | boolean | `false` | Opt-in telemetry |
| `allowedTools` | string[] | `[]` | Tool allowlist (empty = all tools) |

---

## MCP (Model Context Protocol)

Plugsky CLI can connect to any MCP server and expose its tools to the agent alongside the built-in tools.

### Configure MCP servers

Add to `~/.plugsky/config.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_..." }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    }
  }
}
```

### List and test

```sh
plugsky mcp              # list configured servers
plugsky mcp test         # connect to all and list their tools
```

MCP tools appear as `mcp__<server>__<toolname>` in the agent's tool list.

---

## RAG (Codebase Indexing)

Build a local semantic search index of your project for context-aware prompts:

```sh
plugsky index
```

This walks `.ts`, `.js`, `.py`, `.go`, `.rs`, `.java`, `.rb`, `.php`, `.md` files, chunks them (1600 chars each), embeds them via `POST /v1/embeddings`, and stores in `~/.plugsky/rag/<project-hash>.db`.

The index is automatically queried before each agent task in `run` mode and the TUI — relevant chunks are injected into the system prompt.

---

## Sessions

Every conversation is persisted to `~/.plugsky/sessions.db` (SQLite).

```sh
plugsky --resume         # resume the latest session in this cwd
```

Sessions are automatically created and appended to. The TUI creates one session per launch.

---

## Practical examples

### Debug a failing test

```sh
plugsky "run the tests, look at the failing one, and fix it"
```

### Generate a component

```sh
plugsky "create a React component at src/components/UserCard.tsx that shows a user avatar, name, and email. Follow the existing pattern in src/components/"
```

### Multi-step research + report

```sh
plugsky "search for best practices on REST API versioning, save the results to api-versioning.md, then create a summary"
```

### Interactive code review

```sh
plugsky                              # opens TUI
> review the changes in src/utils/
> (reads the code, suggests improvements, asks for approval before editing)
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Not authenticated` | No API key configured | Run `plugsky login` or set `PLUGSKY_API_KEY` |
| `401` errors | Invalid or expired API key | Re-run `plugsky login` |
| Model not found | The model ID is wrong | Run `plugsky models` to see available IDs |
| `auto` always uses micro | `/v1/plugsky/route` not implemented or returning wrong shape | Check with `curl` against the route endpoint |
| SSH / git errors | `rg` (ripgrep) not installed | `brew install ripgrep` or `apt install ripgrep` |
| TUI doesn't render terminal | Terminal doesn't support Ink (e.g., CI, pipe) | Use `plugsky "task"` instead |
| Release workflow fails | GitHub billing issue | Resolve at `https://github.com/settings/billing` |

---

## Building from source

```sh
git clone https://github.com/coinplannet/plugskyCLI.git
cd plugskyCLI
bun install
bun run build                      # produces dist/index.js
bun run typecheck                  # type-check with TypeScript
bun test                           # 24+ tests, all pass
bun run build.ts --compile        # standalone binaries in dist/bin/
```

---

## Repository

[github.com/coinplannet/plugskyCLI](https://github.com/coinplannet/plugskyCLI)

License: MIT
