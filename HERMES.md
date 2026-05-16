# Hermes — Ops Notes

Day-to-day reference for operating a Hermes Agent gateway on a VPS once installed.

> **This is an operational reference, not a fresh-install guide.** For installing Hermes Agent from scratch (creating the `hermes` user, installing the package, wiring up Discord/Telegram, etc.), follow the [upstream Hermes Agent docs](https://github.com/anthropics/hermes-agent) and the systemd unit examples there. The sections below assume Hermes is already running on the box.
>
> Matt-specific values (bot names, workspace paths, GitHub identities) appear throughout — they're examples of what you'll set up, not values you should copy verbatim.

## What this setup looks like

- Hermes Agent v0.13.0 running as a dedicated `hermes` user (uid 1001) — separate user is the isolation boundary from the rest of the system
- Provider: OpenRouter (Claude Sonnet 4.6 main, Gemini 2.5 Flash auxiliary)
- Terminal backend: `local` (commands run as the `hermes` user). Docker backend is also supported if you want container-level isolation on top of the user separation — pick based on your threat model.
- Messaging: Discord DMs + Telegram, supervised by `hermes-gateway.service`
- Web search: Tavily (primary), DuckDuckGo skill as fallback
- GitHub bot: a dedicated bot account (e.g. `emrg-hermes-bot`) — SSH key on the bot account, fine-grained PAT in hermes's `.env` as `GITHUB_TOKEN`. You'll create your own bot account and PAT.
- Engineering workspaces (example layout — yours will differ):
  - Human side: `/home/deploy/projects/<your-project>/` (HTTPS via `gh`)
  - Hermes side: `/home/hermes/workspace/<your-project>/` (SSH, using the bot's key)
- Custom skills go under `~/.hermes/skills/<category>/<name>/SKILL.md`
- Weekly auto-update: Sundays at 06:00 local via `hermes-update.timer`
- Manual update from Discord: DM `/update` to the bot
- Log rotation: `/etc/logrotate.d/hermes-gateway` (daily, keep 14) and `/etc/logrotate.d/hermes-update` (monthly, keep 6)

## Required secrets (`.env`)

Before Hermes will start, `/home/hermes/.hermes/.env` needs these (get each from the respective dashboard):

| Variable | Where to get it |
|---|---|
| `OPENROUTER_API_KEY` | https://openrouter.ai → Keys |
| `TAVILY_API_KEY` | https://tavily.com → API Keys |
| `DISCORD_BOT_TOKEN` | https://discord.com/developers → your app → Bot → Reset Token. Enable **Message Content Intent**. |
| `TELEGRAM_BOT_TOKEN` | DM `@BotFather` on Telegram → `/newbot` |
| `GITHUB_TOKEN` | https://github.com/settings/tokens → fine-grained PAT scoped to the repos Hermes should touch |

## Service control

```bash
sudo systemctl status hermes-gateway       # current state
sudo systemctl restart hermes-gateway      # restart
sudo systemctl stop hermes-gateway         # stop
sudo systemctl start hermes-gateway        # start
```

## Logs

```bash
# Live tail of the gateway log (Ctrl+C to exit)
sudo tail -f /var/log/hermes-gateway.log

# Full file
sudo less /var/log/hermes-gateway.log

# systemd's view (boot events, exit codes, restarts)
sudo journalctl -u hermes-gateway -f          # follow new
sudo journalctl -u hermes-gateway -n 100      # last 100 lines
sudo journalctl -u hermes-gateway --since "1 hour ago"
```

## Hermes CLI (must run as the hermes user)

```bash
sudo -u hermes -i hermes gateway status     # gateway status
sudo -u hermes -i hermes gateway list       # list profiles
sudo -u hermes -i hermes config             # current config (redacted)
sudo -u hermes -i hermes doctor             # full health check
sudo -u hermes -i hermes --version          # version
```

For a proper interactive REPL (only do this when the gateway is stopped — they'd compete for the Discord token):

```bash
sudo systemctl stop hermes-gateway
sudo -u hermes -i hermes
# ...play around...
sudo systemctl start hermes-gateway
```

## Process + connection sanity

```bash
sudo systemctl show hermes-gateway --property=MainPID,MemoryCurrent,ActiveState
sudo ss -tnp 2>/dev/null | grep hermes      # active TCP (Discord = Cloudflare IPs on :443)
```

## Where things live

| Thing | Path |
|---|---|
| Gateway systemd unit | `/etc/systemd/system/hermes-gateway.service` |
| Update timer + service | `/etc/systemd/system/hermes-update.{timer,service}` |
| Update runner script | `/usr/local/bin/hermes-update.sh` |
| logrotate configs | `/etc/logrotate.d/hermes-gateway`, `/etc/logrotate.d/hermes-update` |
| Gateway log | `/var/log/hermes-gateway.log` |
| Update log | `/var/log/hermes-update.log` |
| Hermes config (YAML) | `/home/hermes/.hermes/config.yaml` |
| Hermes secrets (.env) | `/home/hermes/.hermes/.env` (chmod 600) — see "Required secrets" above |
| Hermes source/venv | `/home/hermes/.hermes/hermes-agent/` |
| Persona | `/home/hermes/.hermes/SOUL.md` (optional — defines the agent's voice/style) |
| Skills | `/home/hermes/.hermes/skills/` (bundled skills + any you add) |
| Custom skills | `/home/hermes/.hermes/skills/<category>/<name>/SKILL.md` |
| Project workspace (hermes side) | `/home/hermes/workspace/<your-project>/` |
| Project workspace (your side) | `/home/<you>/projects/<your-project>/` |
| Hermes git identity | Bot's noreply email, e.g. `<id>+<botname>@users.noreply.github.com` |
| Hermes SSH key (private) | `/home/hermes/.ssh/id_ed25519` |
| Sessions | `/home/hermes/.hermes/sessions/` |
| Cron jobs | `/home/hermes/.hermes/cron/` |
| Memories | `/home/hermes/.hermes/memories/` |

## Editing config

Most knobs live in `config.yaml`. Use the CLI (preferred — it validates):

```bash
sudo -u hermes -i hermes config set <key.path> <value>
# e.g.:
sudo -u hermes -i hermes config set model.default anthropic/claude-sonnet-4.6
sudo -u hermes -i hermes config set terminal.container_memory 1024
```

⚠ Watch for the **`model` vs `model.default` gotcha** — setting a scalar at `model` followed by `model.<subkey>` will wipe the scalar. Always use `model.default` for the model name.

For secrets (API keys, tokens), edit `.env` directly:

```bash
sudo -u hermes -i nano /home/hermes/.hermes/.env
```

After any change, restart the gateway: `sudo systemctl restart hermes-gateway`.

## Weekly auto-update

```bash
sudo systemctl list-timers hermes-update.timer     # when does it fire next?
sudo systemctl status hermes-update.timer          # timer state
sudo cat /var/log/hermes-update.log                # what happened last Sunday
sudo systemctl start hermes-update.service         # run it now (skip waiting for Sunday)
sudo systemctl disable --now hermes-update.timer   # pause weekly updates
```

The cron does `hermes update` + `hermes skills update` + `systemctl restart hermes-gateway` every Sunday at 06:00 (+0–10min jitter). Logs persist in `/var/log/hermes-update.log` and rotate monthly.

To trigger an update from Discord any time: DM `/update` to the bot. That runs `hermes update && hermes skills update` but **does NOT restart the gateway** (can't kill itself cleanly). The running gateway will still be the old version until the next Sunday cron restart, or run `sudo systemctl restart hermes-gateway` yourself.

## Common issues

**Gateway exits or restarts in a loop**
- Check `sudo journalctl -u hermes-gateway -n 100`
- Most common cause: Docker daemon down. `sudo systemctl status docker`

**Bot doesn't reply to DMs**
- Verify gateway is `active`: `sudo systemctl is-active hermes-gateway`
- Verify it has Discord connections: `sudo ss -tnp | grep hermes` (should show Cloudflare IPs)
- Confirm the bot user has **Message Content Intent** enabled in the Discord Developer Portal

**Disk filling up**
- The Hermes Docker image is 2.23 GB; sessions and logs accumulate over time
- `sudo docker system prune` periodically removes unused images/containers
- Sessions live in `/home/hermes/.hermes/sessions/`; can be cleared if needed

**Memory pressure**
- This box has only 2 GB RAM. If Hermes starts hitting swap heavily, lower `terminal.container_memory` further (currently 768 MB)

## Outstanding items / known caveats

- **Docker output mount may need configuring.** If you switch to the Docker terminal backend and use media generation (images/audio), the output won't surface back through Discord without a mount in `terminal.docker_run_args`.

## Rollback (full nuke)

> ⚠ **This is destructive and irreversible.** `userdel -r hermes` wipes `/home/hermes` entirely — including sessions, memories, custom skills, the bot's SSH private key, and your `.env` secrets. If you might want any of it back, copy `/home/hermes/.hermes/` somewhere safe **before** running this.

```bash
sudo systemctl disable --now hermes-gateway hermes-update.timer
sudo rm /etc/systemd/system/hermes-gateway.service
sudo rm /etc/systemd/system/hermes-update.{service,timer}
sudo rm /usr/local/bin/hermes-update.sh
sudo rm /etc/logrotate.d/hermes-gateway /etc/logrotate.d/hermes-update
sudo systemctl daemon-reload
sudo userdel -r hermes        # removes /home/hermes too
sudo rm /var/log/hermes-gateway.log /var/log/hermes-gateway.log.* /var/log/hermes-update.log /var/log/hermes-update.log.*
```

Anything running under a different user (e.g. other bots under `deploy`) is untouched — the user boundary is the isolation.
