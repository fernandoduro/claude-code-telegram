# systemd Setup (Linux / WSL)

Linux distributions with systemd use **user units** to run background services
without root. These templates mirror the `launchd/` plists:

1. **Run the Telegram bot continuously** — starts on login, restarts if it crashes
2. **Run scheduled skills** — like a daily briefing at 7am

## Quick Setup

Ask Claude: **"Help me set up systemd for the Telegram bot"**

## Manual Setup

### 1. Edit the unit files

The units use `%h` (your home directory), so you only need to edit them if:

- You cloned the repo somewhere other than `~/claude-code-telegram`
- You aren't using a `.venv/` inside the repo
- Your `claude` binary lives outside `/usr/local/bin`, `/usr/bin`, or `~/.local/bin`
  (e.g. mise/asdf/volta/nvm — add those shim paths to `Environment=PATH=…`)

```bash
# Find claude path
which claude

# Find python path (or your venv's python)
which python3
ls ~/claude-code-telegram/.venv/bin/python  # if you created a venv
```

### 2. Copy units into place

```bash
mkdir -p ~/.config/systemd/user
cp systemd/com.claude.telegram-bot.service ~/.config/systemd/user/
cp systemd/com.claude.daily-brief.service ~/.config/systemd/user/
cp systemd/com.claude.daily-brief.timer ~/.config/systemd/user/
systemctl --user daemon-reload
```

### 3. Enable and start the bot

```bash
systemctl --user enable --now com.claude.telegram-bot.service
```

### 4. Verify it's running

```bash
systemctl --user status com.claude.telegram-bot.service
```

## Common Commands

```bash
# Start / stop / restart
systemctl --user start    com.claude.telegram-bot.service
systemctl --user stop     com.claude.telegram-bot.service
systemctl --user restart  com.claude.telegram-bot.service

# Status
systemctl --user status   com.claude.telegram-bot.service

# Live logs (Ctrl-C to exit)
journalctl --user -u com.claude.telegram-bot.service -f

# Last 200 lines
journalctl --user -u com.claude.telegram-bot.service -n 200

# Disable so it doesn't start on next login
systemctl --user disable com.claude.telegram-bot.service
```

## Daily Brief Timer

```bash
# Enable the 7am trigger
systemctl --user enable --now com.claude.daily-brief.timer

# See when it next fires
systemctl --user list-timers --all | grep daily-brief

# Run it once right now (without waiting for 7am)
systemctl --user start com.claude.daily-brief.service
```

### Schedule Options

Edit the `OnCalendar=` line in `com.claude.daily-brief.timer`. systemd's
calendar syntax is more compact than launchd's:

```ini
# Every day at 7:00
OnCalendar=*-*-* 07:00:00

# Weekdays only at 7:00
OnCalendar=Mon..Fri *-*-* 07:00:00

# Every hour, on the hour
OnCalendar=hourly

# Every 30 minutes
OnCalendar=*:0/30
```

See `man systemd.time` for the full grammar. Test an expression with
`systemd-analyze calendar 'Mon..Fri *-*-* 07:00:00'`.

## Keeping the bot running when you're logged out

By default, `systemctl --user` services stop when your last login session
ends. To keep the Telegram bot running while you're away:

```bash
sudo loginctl enable-linger "$USER"
```

This is one-time per machine. Disable later with `disable-linger`.

## WSL2 Caveats

If you're on **WSL2 (Ubuntu/Debian/etc on Windows)**, two extra things matter:

1. **systemd must be enabled in WSL**. Newer WSL2 has it on by default, but
   confirm with `systemctl is-system-running` — if it returns `offline`,
   add this to `/etc/wsl.conf` and run `wsl --shutdown` from Windows:
   ```ini
   [boot]
   systemd=true
   ```

2. **Windows must be running.** WSL is not a separate VM that runs while
   Windows is asleep. The bot is reachable only when Windows is on **and**
   your WSL distro has been started at least once since boot. To start
   the distro automatically when Windows boots, create a Task Scheduler
   entry that runs `wsl -d <your-distro> --exec true` on logon, or use
   `wsl --install --enable-system-distro` features. Alternatives if you
   need 24/7 uptime: a tiny VPS, a Raspberry Pi, or a Docker container
   on a Linux host.

## Troubleshooting

**Service won't start:**
```bash
systemctl --user status com.claude.telegram-bot.service
journalctl --user -u com.claude.telegram-bot.service -n 100 --no-pager
```

**`claude: command not found` in logs:** the `Environment=PATH=…` in the
unit file doesn't include the directory holding the `claude` binary. Run
`which claude` and add that directory to the PATH line.

**`ModuleNotFoundError`:** the `ExecStart=` Python isn't the one with the
installed dependencies. Either point it at your venv's Python, or run
`pip install -r requirements.txt` against the system Python you're using.

**Permission issues:**
```bash
chmod +x ~/claude-code-telegram/telegram-bot.py
```

## launchd ↔ systemd cheatsheet

| launchd                                              | systemd (user)                                         |
|------------------------------------------------------|--------------------------------------------------------|
| `~/Library/LaunchAgents/foo.plist`                   | `~/.config/systemd/user/foo.service`                   |
| `launchctl load …`                                   | `systemctl --user enable --now foo.service`            |
| `launchctl unload …`                                 | `systemctl --user disable --now foo.service`           |
| `launchctl list \| grep foo`                          | `systemctl --user status foo.service`                  |
| `tail -f /tmp/foo.log`                               | `journalctl --user -u foo.service -f`                  |
| `KeepAlive=true`                                     | `Restart=always`                                       |
| `StartCalendarInterval`                              | `.timer` unit with `OnCalendar=`                       |
