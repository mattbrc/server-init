# Hermes Agent — Setup Playbook

Step-by-step install guide for putting [Hermes Agent](https://github.com/NousResearch/hermes-agent) on a fresh Ubuntu VPS with Discord + Telegram gateways. Produces the same setup documented in [`hermes-ops.md`](./hermes-ops.md).

This playbook is written for someone comfortable with the terminal but not deeply familiar with Linux server admin. You'll spend ~60–90 minutes on this, mostly waiting on installs.

---

## Prerequisites

Complete [`init.md`](./init.md) first — that gets you a hardened Ubuntu VPS with a non-root `deploy` user, firewall, SSH keys, and Node/Claude Code installed. The steps below assume that's done.

You'll also need accounts at:

| Service | What for | Cost |
|---|---|---|
| [OpenRouter](https://openrouter.ai) | LLM access (Claude Sonnet, Gemini Flash) | Pay-per-use, ~$10–30/mo typical |
| [Tavily](https://tavily.com) | Web search backend | Free tier (1000 searches/mo) |
| [Discord Developer Portal](https://discord.com/developers/applications) | Discord bot | Free |
| [Telegram BotFather](https://t.me/botfather) | Telegram bot | Free |

Have all four accounts created and ready to grab API keys before you start. You'll hit explicit STOP points where you need to paste them in.

### Hardware floor

Tested on a Hetzner CPX11 (2 vCPU, 2 GB RAM, 40 GB disk). 2 GB RAM is tight but workable using the `local` terminal backend (which is what this playbook configures). If you have only 1 GB RAM, this will be very rough — upgrade first.

### Decisions baked into this playbook

These are choices the playbook makes for you. If you have a strong opinion otherwise, note it before you start:

- **Dedicated `hermes` Linux user** for isolation from `deploy` and root. The user separation is the real security boundary; if `hermes` gets compromised it can only touch `/home/hermes/`.
- **`local` terminal backend** — commands run as the `hermes` user, not inside a Docker container. Cheaper on RAM, simpler, and adequate when the user separation already does isolation. Switch to `docker` later if you want a second isolation layer.
- **OpenRouter** as the LLM provider, with Claude Sonnet 4.6 as the main model and Gemini 2.5 Flash for auxiliary tasks (compression, vision, web summarization).
- **Both Discord and Telegram** wired in.
- **systemd-managed gateway** with weekly auto-update on Sundays at 06:00.

---

## Step 1 — Create the `hermes` user

Logged in as `deploy` on the VPS:

```bash
# Create the user with its own home directory and bash shell
sudo useradd -m -s /bin/bash hermes

# No password — we'll only access this user via `sudo -u hermes` from deploy
sudo passwd -l hermes

# Verify
id hermes
ls -la /home/hermes
```

You should see `uid=1001(hermes)` (or similar) and an empty home directory.

**Purpose:** `hermes` is the isolation boundary. It's not in the `sudo` group, has no SSH access of its own, and can only touch files in `/home/hermes/`. Even if the Hermes agent goes wild, your `deploy` account and everything under it is untouched.

---

## Step 2 — Install Hermes Agent

Run the upstream installer as the `hermes` user:

```bash
sudo -u hermes -i bash -c 'curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash'
```

This pulls Python 3.11, sets up a virtualenv under `/home/hermes/.hermes/`, and puts the `hermes` CLI at `/home/hermes/.local/bin/hermes`. Takes a few minutes.

**Verify:**

```bash
sudo -u hermes -i hermes --version
```

Should print something like `Hermes Agent v0.13.0`.

---

## Step 3 — Get your API keys (🛑 STOP)

Before continuing, grab these from the dashboards listed in the prerequisites:

- [ ] `OPENROUTER_API_KEY` — from https://openrouter.ai → Keys. Add $10 of credit while you're there.
- [ ] `TAVILY_API_KEY` — from https://tavily.com after signing up.

Keep them in a notes file or password manager. You'll paste them into a `.env` file in the next step.

---

## Step 4 — Configure Hermes (model + secrets)

Set the model provider and default model:

```bash
sudo -u hermes -i hermes config set model.provider openrouter
sudo -u hermes -i hermes config set model.default anthropic/claude-sonnet-4.6
```

> ⚠ **Hermes config gotcha:** setting a value at `model` followed by `model.<subkey>` can silently overwrite. Always use the dotted form (`model.default`) for nested keys. After changing config, read the file back: `sudo -u hermes cat /home/hermes/.hermes/config.yaml | head -10`.

Set the terminal backend to `local` (lighter on RAM than `docker`):

```bash
sudo -u hermes -i hermes config set terminal.backend local
```

Set the web search backend:

```bash
sudo -u hermes -i hermes config set web.backend tavily
```

Now write the API keys to `.env` (this file is read at startup by both the CLI and the gateway):

```bash
sudo -u hermes -i bash -c 'cat > ~/.hermes/.env <<EOF
OPENROUTER_API_KEY=sk-or-v1-REPLACE_ME
TAVILY_API_KEY=tvly-REPLACE_ME
EOF
chmod 600 ~/.hermes/.env'
```

Replace the two `REPLACE_ME` values with your actual keys before saving. The `chmod 600` ensures only `hermes` can read it.

**Verify config loaded correctly:**

```bash
sudo -u hermes -i hermes config       # prints config with secrets redacted
sudo -u hermes -i hermes doctor       # full health check
```

`hermes doctor` should be green across the board. If it complains about a missing key, re-check the `.env` file path and contents.

---

## Step 5 — Create the Discord bot (🛑 STOP)

This step happens in your browser, not on the VPS.

1. Go to https://discord.com/developers/applications → **New Application**.
2. Name it (e.g. `<your-name>-hermes`).
3. Left sidebar → **Bot**:
   - Click **Reset Token**, copy it immediately (you only see it once).
   - Toggle **Message Content Intent** ON. *This is critical — without it, Hermes will see DMs arrive but can't read them.*
   - Toggle **Public Bot** OFF (or leave on — it's a hygiene thing, doesn't affect security since the allowlist gates everything).
4. Left sidebar → **Installation**:
   - Under **Default Install Settings** → **Guild Install**:
     - Scopes: check `bot`
     - Bot Permissions: check **View Channels**, **Send Messages**, **Send Messages in Threads**, **Create Public Threads**, **Embed Links**, **Attach Files**, **Read Message History**, **Add Reactions**, **Use External Emojis**, **Use External Stickers**
   - Save.
   - Copy the **Install Link** at the top of the page.
5. Paste the install link in a new browser tab → pick a server you control (a personal/test server is fine; DMs work without the bot being in any channel) → authorize.
6. **Get your own Discord user ID:** Discord settings → Advanced → enable **Developer Mode**. Then right-click your own name anywhere → **Copy User ID**. It's an 18–19 digit number. This is what Hermes uses to allowlist you — only you can talk to the bot.

Save these two values somewhere:

- `DISCORD_BOT_TOKEN` (from step 3)
- `DISCORD_USER_ID` (your numeric ID from step 6)

---

## Step 6 — Create the Telegram bot (🛑 STOP)

Way simpler than Discord.

1. Open Telegram → search for **@BotFather** → start chat.
2. Send `/newbot`.
3. Pick a display name (e.g. `<your-name>'s Hermes`).
4. Pick a username — must end in `bot` and be globally unique (e.g. `yourname_hermes_bot`).
5. BotFather replies with the bot token — copy it immediately. Looks like `7234891234:AAEhBP8x...`.
6. While in BotFather, lock the bot down:
   - `/setjoingroups` → your bot → **Disable** (prevents random people adding it to groups)
   - `/setprivacy` → your bot → **Enable** (good hygiene)
7. **Get your own Telegram user ID:** search for **@userinfobot** → start chat. It immediately replies with your numeric ID (9–10 digits).

Save:

- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_USER_ID`

---

## Step 7 — Wire the gateway to Discord + Telegram

Append the four new secrets to the `.env`:

```bash
sudo -u hermes -i bash -c 'cat >> ~/.hermes/.env <<EOF
DISCORD_BOT_TOKEN=REPLACE_ME
TELEGRAM_BOT_TOKEN=REPLACE_ME
EOF'
```

Configure the allowlists (replace the IDs with your real ones):

```bash
sudo -u hermes -i hermes config set discord.allowed_users "123456789012345678"
sudo -u hermes -i hermes config set telegram.allowed_users "987654321"
```

If you want quality-of-life streaming in Telegram (response appears word-by-word):

```bash
sudo -u hermes -i hermes config set streaming.enabled true
sudo -u hermes -i hermes config set streaming.transport edit
```

**Test the gateway runs at all** (before wrapping it in systemd):

```bash
sudo -u hermes -i hermes gateway run
```

You should see startup logs, then it sits idle waiting for messages. Open Telegram, DM your bot "hello" — it should respond within ~30 seconds (cold start). Ctrl+C to stop once you've confirmed it works.

If it doesn't respond:
- **Discord:** confirm Message Content Intent is enabled in the developer portal
- **Telegram:** confirm you DM'd the right bot (the username is case-sensitive)
- **Either:** check that your user ID is correctly set in `discord.allowed_users` / `telegram.allowed_users` — if you're not on the allowlist, the bot silently ignores you

---

## Step 8 — systemd unit (run gateway on boot)

Write the unit file:

```bash
sudo tee /etc/systemd/system/hermes-gateway.service > /dev/null <<'EOF'
[Unit]
Description=Hermes Agent Messaging Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=hermes
Group=hermes
WorkingDirectory=/home/hermes
Environment="PATH=/home/hermes/.local/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/home/hermes/.local/bin/hermes gateway run
Restart=on-failure
RestartSec=10
TimeoutStopSec=210
StandardOutput=append:/var/log/hermes-gateway.log
StandardError=append:/var/log/hermes-gateway.log

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/home/hermes /var/log/hermes-gateway.log
ProtectHome=read-only

[Install]
WantedBy=multi-user.target
EOF
```

Create the log file with the right ownership before starting the service:

```bash
sudo touch /var/log/hermes-gateway.log
sudo chown hermes:hermes /var/log/hermes-gateway.log
sudo chmod 644 /var/log/hermes-gateway.log
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-gateway
sudo systemctl status hermes-gateway
```

`Active: active (running)` = good. Send another DM to verify.

> **Note on terminal backends:** if you later switch to `docker` for an extra isolation layer, you'll need to add `Requires=docker.service` and `After=docker.service` to the `[Unit]` section, and add `hermes` to the `docker` group: `sudo usermod -aG docker hermes`.

---

## Step 9 — Log rotation

Keep the gateway log from growing forever:

```bash
sudo tee /etc/logrotate.d/hermes-gateway > /dev/null <<'EOF'
/var/log/hermes-gateway.log {
    su root hermes
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0644 hermes hermes
    copytruncate
}
EOF
```

`copytruncate` is what lets us rotate without restarting the gateway. 14 days of daily logs is enough to debug "what did Hermes do last week."

Test the config:

```bash
sudo logrotate -d /etc/logrotate.d/hermes-gateway
```

---

## Step 10 — Weekly auto-update timer

Hermes does NOT auto-update by default. This step adds a Sunday 6am job that runs `hermes update && hermes skills update && systemctl restart hermes-gateway`.

Write the update script:

```bash
sudo tee /usr/local/bin/hermes-update.sh > /dev/null <<'EOF'
#!/bin/bash
# Hermes Agent weekly auto-update.
# Run by hermes-update.service via hermes-update.timer (Sunday 6am).
set -eu
echo
echo "======================================================================"
echo "  $(date '+%Y-%m-%d %H:%M:%S %Z')  Hermes weekly update starting"
echo "======================================================================"

echo
echo "--- hermes update ---"
sudo -u hermes -i hermes update || echo "(hermes update exited non-zero)"

echo
echo "--- hermes skills update ---"
sudo -u hermes -i hermes skills update || echo "(skills update exited non-zero)"

echo
echo "--- restart hermes-gateway ---"
systemctl restart hermes-gateway
sleep 5
systemctl is-active hermes-gateway

echo
echo "--- post-update sanity ---"
sudo -u hermes -i hermes --version

echo
echo "======================================================================"
echo "  $(date '+%Y-%m-%d %H:%M:%S %Z')  Hermes weekly update finished"
echo "======================================================================"
EOF

sudo chmod +x /usr/local/bin/hermes-update.sh
```

Write the systemd service and timer:

```bash
sudo tee /etc/systemd/system/hermes-update.service > /dev/null <<'EOF'
[Unit]
Description=Hermes Agent weekly auto-update
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/hermes-update.sh
StandardOutput=append:/var/log/hermes-update.log
StandardError=append:/var/log/hermes-update.log
TimeoutStartSec=30min
EOF

sudo tee /etc/systemd/system/hermes-update.timer > /dev/null <<'EOF'
[Unit]
Description=Run Hermes Agent weekly auto-update

[Timer]
OnCalendar=Sun *-*-* 06:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
EOF
```

Monthly log rotation for update logs:

```bash
sudo tee /etc/logrotate.d/hermes-update > /dev/null <<'EOF'
/var/log/hermes-update.log {
    su root root
    monthly
    rotate 6
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    copytruncate
}
EOF
```

Enable the timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-update.timer
sudo systemctl list-timers hermes-update.timer
```

You should see "Sun ... 06:00:00" as the next firing.

**Test it now** (optional but reassuring — runs the update immediately):

```bash
sudo systemctl start hermes-update.service
sudo tail -f /var/log/hermes-update.log    # Ctrl+C when you see "finished"
```

---

## Step 11 — Install starter skills

Three skills worth installing on day one:

```bash
sudo -u hermes -i hermes skills install official/security/1password
sudo -u hermes -i hermes skills install official/devops/docker-management
sudo -u hermes -i hermes skills install official/research/duckduckgo-search
```

- **1password**: lets Hermes fetch secrets from your 1Password vault at runtime instead of you pasting them into `.env`. Optional but high-leverage if you already use 1Password.
- **docker-management**: useful even with the `local` terminal backend if you ever ask Hermes to debug a Docker container elsewhere.
- **duckduckgo-search**: free fallback search if you hit the Tavily rate limit.

Restart the gateway so skills load:

```bash
sudo systemctl restart hermes-gateway
```

Don't install everything at once. Skills consume context budget; three well-chosen skills beat thirty noisy ones. Add more as you find concrete gaps in 2–3 weeks of real usage.

---

## Step 12 — First real conversation

DM your Telegram or Discord bot:

> Hi! Confirm you're working — what model are you using, and what skills do you have access to?

You should get back something like "I'm Claude Sonnet 4.6 via OpenRouter, with skills X, Y, Z available."

If you see that, you're done with setup. Move on to the [ops cheat sheet](./hermes-ops.md) for day-to-day commands.

---

## Troubleshooting

**Gateway exits immediately / keeps restarting**
- `sudo journalctl -u hermes-gateway -n 100`
- Most common cause: missing or malformed `.env` file. Check `sudo -u hermes cat /home/hermes/.hermes/.env`.

**Bot doesn't reply at all**
- `sudo systemctl is-active hermes-gateway` — should be `active`
- Check that your user ID is in `discord.allowed_users` / `telegram.allowed_users`. The bot silently drops messages from non-allowlisted users.
- Discord specifically: verify **Message Content Intent** is still enabled in the developer portal.

**"Out of memory" / OOM killer activity**
- The 2 GB RAM tier is genuinely tight. Check `free -h`. If you're swapping heavily, add a swap file (covered in [`init.md`](./init.md)) or upgrade the VPS.

**`hermes config set X.Y` seems to wipe my X value**
- Yes, this is a known gotcha. Always read the YAML back with `sudo -u hermes cat /home/hermes/.hermes/config.yaml` after using `config set`.

---

## Rollback (full nuke)

> ⚠ **Destructive and irreversible.** `userdel -r hermes` wipes `/home/hermes` including sessions, memories, custom skills, and your `.env` secrets. Back up `/home/hermes/.hermes/` first if you might want any of it.

```bash
sudo systemctl disable --now hermes-gateway hermes-update.timer
sudo rm /etc/systemd/system/hermes-gateway.service
sudo rm /etc/systemd/system/hermes-update.{service,timer}
sudo rm /usr/local/bin/hermes-update.sh
sudo rm /etc/logrotate.d/hermes-gateway /etc/logrotate.d/hermes-update
sudo systemctl daemon-reload
sudo userdel -r hermes
sudo rm /var/log/hermes-gateway.log* /var/log/hermes-update.log*
```

Anything running under other users (e.g., bots under `deploy`) is untouched — the user boundary is the isolation.

---

## What comes next (not in this playbook)

Once Hermes is running and you've used it for a week, the obvious next step is the engineering workflow — letting Hermes implement bug fixes and features in your repos from Discord requests. That has its own design (dedicated GitHub bot account, repo workspace under `/home/hermes/workspace/`, an `emrg-engineering` skill with danger words and a plan-first protocol). Don't tackle it until you have a feel for how Hermes behaves day-to-day.

See [`hermes-ops.md`](./hermes-ops.md) for day-to-day operational commands.
