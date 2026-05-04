# claude-code-telegram

A personal AI assistant you can message from anywhere via Telegram.

## What is this?

You know how Claude Code runs in your terminal and can read files, run commands, search the web, etc? This lets you talk to it from your phone via Telegram instead of being tied to your computer.

**Text your bot → it runs Claude Code → sends back the response.**

Some things you can do:
- "What's on my calendar today?"
- "Search the web for the latest news on X"
- "Read my notes and summarize them"
- "Send me a daily briefing every morning at 7am"

It also works the other way - Claude can message YOU via Telegram (notifications when tasks finish, scheduled briefings, alerts, etc).

**The key is skills.** Out of the box, Claude Code can read files and search the web. But to make it truly YOUR assistant, create skills that give it access to your stuff:
- Google Calendar (read your schedule)
- Gmail (read-only access to emails)
- Notes app (Obsidian, Apple Notes, etc.)
- Weather for your location
- Whatever else you want

The `skills/` folder has examples to get you started.

## Quick Start

Open Claude Code and paste:

```
https://github.com/seedprod/claude-code-telegram - help me set this up
```

Claude will clone the repo and walk you through the entire setup.

Claude will walk you through:
1. Creating a Telegram bot with @BotFather
2. Getting your user ID from @userinfobot
3. Configuring the `.env` file
4. Installing the telegram-sender skill globally
5. Setting up launchd to run the bot continuously
6. (Optional) Setting up scheduled skills like daily briefings

---

## What's Included

| File | Purpose |
|------|---------|
| `telegram-bot.py` | Receives messages from Telegram → sends to Claude Code |
| `skills/telegram-sender/` | Lets Claude send messages TO you |
| `skills/daily-brief/` | Example scheduled skill using telegram-sender |
| `launchd/` | macOS templates for running bot + scheduled skills |
| `systemd/` | Linux/WSL user-unit templates for running bot + scheduled skills |

## How It Works

**Inbound (you → Claude):**
```
Telegram → telegram-bot.py → claude -p "message" → response → Telegram
```

**Outbound (Claude → you):**
```
Claude skill → telegram-sender/send.sh → Telegram API → you
```

## Manual Setup

If you prefer to set up manually instead of having Claude guide you:

### 1. Prerequisites
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- Python 3.10+

### 2. Create Telegram Bot
1. Message [@BotFather](https://t.me/botfather) on Telegram
2. Send `/newbot` and follow prompts
3. Copy the bot token

### 3. Get Your User ID
1. Message [@userinfobot](https://t.me/userinfobot) on Telegram
2. Copy your user ID

### 4. Configure
```bash
cp .env.example .env
# Edit .env with your token and user ID
```

### 5. Install Dependencies
```bash
pip install -r requirements.txt
```

### 6. Install Skill Globally
```bash
cp -r skills/telegram-sender ~/.claude/skills/
```

### 7. Run Bot (Manual)
```bash
python telegram-bot.py
```

### 8. Run Bot Continuously

**macOS (launchd):**
```bash
# Edit launchd/com.claude.telegram-bot.plist with your paths
cp launchd/com.claude.telegram-bot.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.claude.telegram-bot.plist
```

**Linux / WSL (systemd user units):**
```bash
mkdir -p ~/.config/systemd/user
cp systemd/com.claude.telegram-bot.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now com.claude.telegram-bot.service
```

See [`systemd/README.md`](systemd/README.md) for full instructions, the
daily-brief timer, the `loginctl enable-linger` step, and WSL2-specific
caveats.

## Bot Commands

- `/new` - Clear session, start fresh
- `/status` - Show session status

## Voice Messages

**Apple Silicon (built in):**
```bash
pip install mlx-whisper
```

**Linux / WSL / Intel Mac:** `mlx-whisper` is Apple-Silicon-only. Use
`faster-whisper` instead — it runs on CPU (and CUDA if available) and is
a drop-in replacement for the `transcribe_audio()` call. A small patch to
`telegram-bot.py` is needed; see issue/PR if not yet merged.

Voice messages are transcribed locally and sent to Claude.

## Example: Daily Briefing

The `skills/daily-brief/` shows how to create a scheduled skill that sends you a morning briefing via Telegram. See `skills/daily-brief/SKILL.md` for details.

To schedule it:

**macOS (launchd):**
```bash
# Edit launchd/com.claude.daily-brief.plist with your paths
cp launchd/com.claude.daily-brief.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.claude.daily-brief.plist
```

**Linux / WSL (systemd user units):**
```bash
cp systemd/com.claude.daily-brief.service ~/.config/systemd/user/
cp systemd/com.claude.daily-brief.timer   ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now com.claude.daily-brief.timer
```

## Security

### ALLOWED_USERS is critical

**Always set `ALLOWED_USERS` in your `.env` file.** If left empty, anyone who discovers your bot's username can send it messages and run Claude with full tool access on your machine.

```bash
# .env - always set this
ALLOWED_USERS=123456789
```

Get your user ID from [@userinfobot](https://t.me/userinfobot) on Telegram.

### Understand the tool access

The bot runs Claude with these tools enabled:
- `Read` / `Write` / `Edit` - file system access
- `Bash` - shell command execution
- `Glob` / `Grep` - file search
- `WebFetch` / `WebSearch` - internet access
- `Task` / `Skill` - agent spawning and skill execution

This is powerful and intentional for a personal assistant, but understand that messages you send can trigger real actions on your system.

### Protect your tokens

- Never commit `.env` (already in `.gitignore`)
- Your bot token lets anyone impersonate your bot
- Your chat ID lets anyone send you messages via the bot

### Session file

Sessions are stored in `~/.telegram-claude-sessions.json`. Default file permissions apply. On shared systems, consider restricting access:

```bash
chmod 600 ~/.telegram-claude-sessions.json
```

## License

MIT
