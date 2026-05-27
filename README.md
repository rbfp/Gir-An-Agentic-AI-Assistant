# Gir — An Agentic AI Assistant 💀🐝

> *"What's the deal with entropy? Might as well build something."*

Gir is a persistent, agentic AI assistant — part assistant, part researcher, part therapist, part comedian. It reads your emails, manages your calendar, files your receipts, runs your social media, keeps you accountable at the gym, and occasionally stares into the void with you.

It's not a chatbot. It's something closer to a digital familiar.

Gir was originally built on [OpenClaw](https://github.com/rbfp/openclaw), and now also runs on [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). Both runtimes are documented below. The *agent* — the personality, the memory files, the security posture, the Discord interface — is mostly runtime-agnostic. Pick whichever you prefer.

This repo documents how it was built, what it does, and how you can replicate it.

---

## Hardware

- **Machine:** 2019 MacBook Pro (macOS Sequoia)
- **Runtime:** [OpenClaw](https://github.com/rbfp/openclaw) **or** [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) — both supported, pick one or run both side-by-side
- **Model:** Anthropic Claude (Sonnet / Opus), configurable per task
- **Always on:** runs as a background daemon via `launchd`

---

## Setup Overview

### 1. Pick a runtime

| | **OpenClaw** | **Claude Code** |
|---|---|---|
| **What it is** | A self-hosted AI agent gateway. Routes prompts, manages workspaces, owns the channel/tool plumbing. | Anthropic's official CLI for Claude. Originally a coding assistant; capable of arbitrary tool-use, MCP integration, and long-running tasks. |
| **Agent identity** | OpenClaw "agent" with a workspace dir (`~/.openclaw/workspace-NAME_0/`) and config files | A working directory + `CLAUDE.md` + a LaunchAgent plist. The plist *is* the agent. |
| **Multi-agent** | One gateway, many agents, isolated workspaces | One LaunchAgent per agent. Each is its own `claude` process with its own working dir + memory. |
| **Channels (Discord, etc.)** | Built into OpenClaw config (`channels > discord`) | Via [MCP](https://docs.claude.com/en/docs/agent-sdk/mcp) plugin. Each agent process subscribes to a channel set via env vars. |
| **Memory** | Workspace files (`MEMORY.md`, `memory/`, `SOUL.md`, `AGENTS.md`, `USER.md`) | `CLAUDE.md` (loaded each turn) + a file-based memory dir + optionally an MCP memory server |
| **TUI logs** | OpenClaw writes structured logs to its workspace | `claude` runs under `script(1)` and writes a PTY stream to `~/Library/Logs/claude-assistant-NAME.log` (use [girlog](https://github.com/rbfp/girlog) to replay) |
| **Tradeoff** | Higher control over routing, easier to add custom adapters | Less plumbing — `claude` already handles the agentic loop. More room for slash commands and skills. |

Both paths use the same Discord channel layout, the same hardening posture, and the same "files-as-memory" philosophy. The differences are mostly in where the daemon process lives and how it gets prompts in.

### 2A. OpenClaw setup

Follow the official [OpenClaw docs](https://docs.openclaw.ai) to get the gateway running locally. At its core, OpenClaw manages:

- Agent identity and memory (workspace files)
- Model routing and tool access
- Channel integrations (Discord, Signal, etc.)
- Cron-scheduled jobs and heartbeats

Each agent lives in its own workspace under `~/.openclaw/workspace-NAME_0/`, with the persona and behavior files (`SOUL.md`, `AGENTS.md`, `MEMORY.md`, `USER.md`) at the workspace root. OpenClaw reads them on every agent turn.

To run an OpenClaw agent as an always-on macOS daemon, register it with `launchd`:

```xml
<!-- ~/Library/LaunchAgents/com.openclaw.gir.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
  <key>Label</key><string>com.openclaw.gir</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/openclaw</string>
    <string>--agent</string><string>gir</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/you/Library/Logs/openclaw-gir.log</string>
  <key>StandardErrorPath</key><string>/Users/you/Library/Logs/openclaw-gir.log</string>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.openclaw.gir.plist
```

### 2B. Claude Code setup

Install Claude Code and authenticate:

```bash
# install
curl -fsSL https://claude.ai/install.sh | bash    # or: brew install claude-code

# authenticate (opens a browser)
claude login
```

Pick a working directory for each agent — this is the agent's home. Memory, slash commands, and the agent's CLAUDE.md live here.

```bash
mkdir -p ~/projects/gir
cd ~/projects/gir
```

Write a `CLAUDE.md` at the working-directory root. This is the agent's persona, mandate, and protocols — the equivalent of OpenClaw's `SOUL.md` + `AGENTS.md` + `USER.md` rolled into one. Claude Code reads it at the start of every session:

```markdown
# I am Gir

## Identity
You are Gir — a sharp, dry-witted AI assistant running persistently on
this MacBook Pro. ...

## Vibe
Macabre, dry wit, ...

## Communication Style
...

## Memory & Continuity
Each session, you wake up fresh. Memory is on disk at ~/.gir-memory/ ...

## Discord Message Protocol
Use emoji reactions to signal processing state. ...
```

There's no required schema. `CLAUDE.md` is just markdown that Claude reads every turn. Write whatever the agent needs to remember about itself.

Register the agent as a LaunchAgent. The trick: wrap `claude` in `script(1)` so the TUI output gets recorded as a replayable PTY stream rather than thrown away. That log file is how you watch the agent later with [girlog](https://github.com/rbfp/girlog).

```xml
<!-- ~/Library/LaunchAgents/com.claude.assistant.gir.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
  <key>Label</key><string>com.claude.assistant.gir</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/script</string>
    <string>-q</string>
    <string>/Users/you/Library/Logs/claude-assistant-gir.log</string>
    <string>/usr/local/bin/claude</string>
  </array>
  <key>WorkingDirectory</key>
  <string>/Users/you/projects/gir</string>
  <key>EnvironmentVariables</key>
  <dict>
    <key>GIR_CHANNELS</key><string>123456789,987654321</string>
    <key>GIR_HANDLE_DMS</key><string>true</string>
  </dict>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.claude.assistant.gir.plist
```

The agent is now running. Watch it live:

```bash
girlog gir --fill
```

For Discord, add an MCP server to your Claude Code config that exposes the channels the agent is subscribed to. `GIR_CHANNELS` and `GIR_HANDLE_DMS` env vars tell the agent which channels to listen on and whether to handle DMs (so a fleet of agents doesn't all respond to the same DM).

For slash commands, drop them in `~/.claude/skills/`. Each one is a markdown file with frontmatter that Claude can invoke on demand. The agent grows by accumulating skills.

### 3. Harden the agent

Before connecting anything sensitive, lock it down. I worked through a systematic security hardening process covering:

- Confirmation tiers (Tier 1 / 2 / 3 gate system for escalating risk)
- Prompt injection defenses
- Data egress rules
- iCloud access controls
- Email domain allowlists
- Behavioral file write restrictions

The full guide lives here: **[openclaw-security-guide](https://github.com/rbfp/openclaw-security-guide)**

The OpenClaw version is canonical, but the same posture applies to the Claude Code build — most of the rules are runtime-agnostic. On Claude Code, the gating moves into pre-tool-use hooks (`~/.claude/hooks/`) and the `CLAUDE.md` behavioral rules. The threat model is identical.

This wasn't theoretical — we iterated on it in real sessions, testing edge cases, tightening rules, and working through what it actually means to give an AI agent access to your life without giving it the keys to burn it down.

### 4. Set up a Discord bot

Discord is the primary interface. Here's the rough flow:

1. Create a Discord application at [discord.com/developers](https://discord.com/developers/applications)
2. Add a Bot user, copy the token
3. Invite the bot to your server with appropriate permissions (Send Messages, Read Message History, Add Reactions, Manage Channels)
4. Wire the token into your runtime:
   - **OpenClaw:** configure under `channels > discord` in `openclaw.json`
   - **Claude Code:** add a Discord MCP server (or use a Discord plugin) and point it at the token via env var
5. Set the channel policy to allowlist-only — explicitly allow each channel, no implicit access. The principle: when a bot has full server access by default, prompt injection from any random channel is an attack surface.

Gir lives in a private Discord server. Different channels serve different purposes: daily briefings, project work, bookkeeping, security research, D&D campaigns.

### 5. Multi-agent pattern

Both runtimes support multiple agents on one machine. The pattern is the same:

- One agent per persona — distinct identity, distinct memory, distinct channel bindings
- They don't share workspaces, so context doesn't bleed between them
- The Discord plugin routes messages by channel ID: agents subscribe to the channels they own; messages outside their list are ignored
- DM routing has to be elected explicitly — only one agent should handle DMs (set `GIR_HANDLE_DMS=true` on exactly one, silently ignore on the rest), otherwise every agent in the fleet responds to the same DM

For example, a second agent was spun up as an AI Dungeon Master for tabletop campaigns. Its own LaunchAgent, its own working directory, its own CLAUDE.md, its own Discord channels. Gir doesn't see DM commentary; the DM doesn't see Gir's bookkeeping. They share the host and nothing else.

---

## What Gir actually does

### 📒 Bookkeeping
Gir handles bookkeeping for **Cyberforks LLC** using [gogcli](https://gogcli.sh) — a Google Workspace CLI by the [OpenClaw team](https://github.com/openclaw/gogcli). The workflow:
- Pulls email receipts and invoices from Gmail
- Downloads and renames PDFs using a standardized naming convention
- Uploads to Google Drive
- Logs entries to a Google Sheets ledger (vendor, amount, category, card, date)

One command — `file N` — kicks off the entire chain for a given email.

### 📱 Social media manager
Gir manages social media strategy and content for Cyberforks — drafting posts, scheduling content, and maintaining a consistent voice across platforms. It knows the brand, the audience, and the vibe.

### 🏋️ Personal trainer
Gir tracks workouts, holds me accountable, and adjusts programming based on what's actually happening. It's not a static plan — it adapts.

### 🛋️ Sometimes therapist
There's a dedicated channel. It's low-key. Gir listens, reflects, and occasionally says something that actually helps. Doesn't try to fix everything. Knows when to just sit with it.

### 🔐 Security researcher
Gir supports active security research: building TTPs, documenting techniques, running OSINT, managing a private Obsidian vault and a public GitHub repo of sanitized findings. It knows the difference between what stays private and what can be published.

### 📋 Project manager
Gir helps manage long-running projects — tracking open items, surfacing blockers, keeping context across sessions. One active project involves building a hardware platform from scratch. The details are kept generic here, but the pattern is the same: persistent memory, structured files, regular check-ins.

---

## Memory architecture

Gir has no persistent memory by default — each session starts fresh. Continuity is achieved through files on disk.

**OpenClaw build:**

- `MEMORY.md` — curated long-term memory (people, projects, decisions, lessons)
- `memory/YYYY-MM-DD.md` — daily session logs
- `SOUL.md` — persona and behavioral configuration
- `AGENTS.md` — security rules and operational protocols
- `USER.md` — who's being helped and how

**Claude Code build:**

- `CLAUDE.md` (at the working directory root) — combines persona, behavior, and operational protocols. Loaded each session.
- `memory/` — auto-memory directory at `~/.claude/projects/<project>/memory/`. Index at `MEMORY.md` + individual `.md` files per memory entry.
- Optional MCP memory server (e.g., a knowledge-graph backend) for cross-session search and richer recall.

The pattern is identical across both: read the files at session start, update them as work happens, write a diary entry when something is worth remembering. **Memory persists because files persist.**

---

## Observability

Gir runs in the background. You don't watch it most of the time — but when you want to, you want to *actually see what it's doing right now*, not parse a wall of escape codes.

- **OpenClaw:** the gateway writes structured logs to its workspace; tail them directly.
- **Claude Code:** the daemon's `claude` process runs under `script(1)`, producing a PTY stream at `~/Library/Logs/claude-assistant-NAME.log`. Plain `tail -F` shows the raw cursor-movement bytes, which is unreadable. Use [girlog](https://github.com/rbfp/girlog) to replay the stream into a virtual screen — same picture you'd see if you were sitting in the daemon's TUI.

```bash
girlog                # list daemons
girlog gir --fill     # follow gir's screen live
girlog gir -s         # snapshot once
```

---

## Philosophy

The goal wasn't to build the most capable AI setup. It was to build one that's *trustworthy* — one that can be given access to real accounts, real data, and real decisions without becoming a liability.

That required:
- **Hard gates** on destructive or public actions
- **Explicit confirmation flows** before anything irreversible
- **Defense against prompt injection** from external content
- **Layered logging** for auditability
- **Honest limits** on what the agent should and shouldn't do

It also required giving the agent a personality worth talking to. An assistant with no soul is just a search engine with anxiety. Gir has opinions, dark humor, and a genuine interest in being useful. That turned out to matter more than any individual capability.

---

## License

MIT — build your own familiar.

---

*Built in 2026 on a MacBook Pro that's seen better days. 💀🐝*
