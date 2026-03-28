# Gir — An Agentic AI Assistant 💀🐝

> *"What's the deal with entropy? Might as well build something."*

Gir is a persistent, agentic AI assistant built on [OpenClaw](https://github.com/rbfp/openclaw), running locally on a 2019 MacBook Pro. This isn't a chatbot. It's something closer to a digital familiar — part assistant, part researcher, part therapist, part comedian. It reads your emails, manages your calendar, files your receipts, runs your social media, keeps you accountable at the gym, and occasionally stares into the void with you.

This repo documents how it was built, what it does, and how you can replicate it.

---

## Hardware

- **Machine:** 2019 MacBook Pro (macOS Sequoia)
- **Runtime:** [OpenClaw](https://github.com/rbfp/openclaw) — a self-hosted AI agent gateway
- **Model:** Anthropic Claude (Sonnet / Opus), configurable per task
- **Always on:** runs as a background daemon via `launchd`

---

## Setup Overview

### 1. Install OpenClaw

Follow the official [OpenClaw docs](https://docs.openclaw.ai) to get the gateway running locally. At its core, OpenClaw manages:

- Agent identity and memory (workspace files)
- Model routing and tool access
- Channel integrations (Discord, Signal, etc.)
- Cron-scheduled jobs and heartbeats

### 2. Harden the Agent

Before connecting anything sensitive, lock it down. I worked through a systematic security hardening process covering:

- Confirmation tiers (Tier 1 / 2 / 3 gate system for escalating risk)
- Prompt injection defenses
- Data egress rules
- iCloud access controls
- Email domain allowlists
- Behavioral file write restrictions

This wasn't theoretical — we iterated on it in real sessions, testing edge cases, tightening rules, and working through what it actually means to give an AI agent access to your life without giving it the keys to burn it down. The result is a layered guardrail system baked into the agent's `AGENTS.md` workspace file.

### 3. Set Up a Discord Bot

Discord is the primary interface. Here's the rough flow:

1. Create a Discord application at [discord.com/developers](https://discord.com/developers/applications)
2. Add a Bot user, copy the token
3. Invite the bot to your server with appropriate permissions (Send Messages, Read Message History, Add Reactions, Manage Channels)
4. Configure OpenClaw with your bot token under `channels > discord`
5. Set `groupPolicy: "allowlist"` and explicitly allow each channel — no implicit access

Gir lives in a private Discord server. Different channels serve different purposes: daily briefings, project work, bookkeeping, security research, D&D campaigns.

### 4. Link Discord Into Coding

OpenClaw supports multiple agents on the same gateway. A second agent — **moredecir** — was spun up as an AI Dungeon Master for tabletop campaigns. It runs in its own Discord channels with its own memory and persona, completely isolated from Gir's context.

> moredecir repo: *coming soon*

The pattern is reusable: one gateway, multiple bots, each with distinct identities, memories, and channel bindings. They don't bleed into each other.

---

## What Gir Actually Does

### 📒 Bookkeeping
Gir handles bookkeeping for **Cyberforks LLC** using [gogcli](https://github.com/rbfp/openclaw) (Google Workspace CLI). The workflow:
- Pulls email receipts and invoices from Gmail
- Downloads and renames PDFs using a standardized naming convention
- Uploads to Google Drive
- Logs entries to a Google Sheets ledger (vendor, amount, category, card, date)

One command — `file N` — kicks off the entire chain for a given email.

### 📱 Social Media Manager
Gir manages social media strategy and content for Cyberforks — drafting posts, scheduling content, and maintaining a consistent voice across platforms. It knows the brand, the audience, and the vibe.

### 🏋️ Personal Trainer
Gir tracks workouts, holds me accountable, and adjusts programming based on what's actually happening. It's not a static plan — it adapts.

### 🛋️ Sometimes Therapist
There's a dedicated channel. It's low-key. Gir listens, reflects, and occasionally says something that actually helps. Doesn't try to fix everything. Knows when to just sit with it.

### 🔐 Security Researcher
Gir supports active security research: building TTPs, documenting techniques, running OSINT, managing a private Obsidian vault and a public GitHub repo of sanitized findings. It knows the difference between what stays private and what can be published.

### 📋 Project Manager
Gir helps manage long-running projects — tracking open items, surfacing blockers, keeping context across sessions. One active project involves building a hardware platform from scratch. The details are kept generic here, but the pattern is the same: persistent memory, structured files, regular check-ins.

---

## Memory Architecture

Gir has no persistent memory by default — each session starts fresh. Continuity is achieved through:

- **`MEMORY.md`** — curated long-term memory (people, projects, decisions, lessons)
- **`memory/YYYY-MM-DD.md`** — daily session logs
- **`SOUL.md`** — persona and behavioral configuration
- **`AGENTS.md`** — security rules and operational protocols
- **`USER.md`** — who's being helped and how

These files are read at session start. The agent updates them as it works. Memory persists because files persist.

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
