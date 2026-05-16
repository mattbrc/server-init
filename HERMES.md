# Hermes — Ops Notes

Quick reference for operating the Hermes Agent gateway on this VPS (`first-server`). The setup playbook is in [`hermes.md`](./hermes.md); this file is the day-to-day cheat sheet.

## What we built

- Hermes Agent v0.13.0 running as a dedicated `hermes` user (uid 1001)
- Provider: OpenRouter (Claude Sonnet 4.6 main, Gemini 2.5 Flash auxiliary)
- Terminal backend: `local` (runs as hermes user; Docker isolation rejected as theater for engineering work — hermes user separation is the real boundary)
- Messaging: Discord DMs + Telegram, supervised by `hermes-gateway.service`
- Web search: Tavily (primary), DuckDuckGo skill as fallback
- GitHub bot: `emrg-hermes-bot` (id 285031989) — SSH key on bot account, fine-grained PAT in hermes's `.env` as `GITHUB_TOKEN`
- Engineering workspaces:
  - Yours: `/home/deploy/projects/emrg-master/{rxr,emrg-backend,emrg-cdk-infra,emrg-docs}` (HTTPS via gh)
  - Hermes's: `/home/hermes/workspace/emrg-master/{rxr,emrg-backend,emrg-cdk-infra,emrg-docs}` (SSH)
- Custom skill: `~/.hermes/skills/engineering/emrg-engineering/SKILL.md` — plan-first protocol, danger words, PR template, model escalation triggers
- Weekly auto-update: Sundays at 06:00 local via `hermes-update.timer`
- Manual update from Discord: DM `/update` to the bot
- Log rotation: `/etc/logrotate.d/hermes-gateway` (daily, keep 14) and `/etc/logrotate.d/hermes-update` (monthly, keep 6)
- Fully isolated from the `deploy` user's EMRG bot — no shared credentials or filesystem access

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
| Hermes secrets (.env) | `/home/hermes/.hermes/.env` (chmod 600) — `OPENROUTER_API_KEY`, `TAVILY_API_KEY`, `GITHUB_TOKEN`, `DISCORD_BOT_TOKEN`, `TELEGRAM_BOT_TOKEN`, etc. |
| Hermes source/venv | `/home/hermes/.hermes/hermes-agent/` |
| Persona | `/home/hermes/.hermes/SOUL.md` |
| Skills | `/home/hermes/.hermes/skills/` (91 total: 87 bundled + DDG fallback + emrg-engineering) |
| Custom skill (engineering) | `/home/hermes/.hermes/skills/engineering/emrg-engineering/SKILL.md` |
| EMRG workspace (Hermes) | `/home/hermes/workspace/emrg-master/{rxr,emrg-backend,emrg-cdk-infra,emrg-docs}` |
| EMRG workspace (yours) | `/home/deploy/projects/emrg-master/{rxr,emrg-backend,emrg-cdk-infra,emrg-docs}` |
| Hermes git identity | `emrg-hermes-bot <285031989+emrg-hermes-bot@users.noreply.github.com>` |
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

- **Docker output mount not configured.** Media file delivery (images/audio Hermes generates) may not surface back through Discord. Address by adding a mount to `terminal.docker_run_args` if you start using image/audio generation.

## Rollback (full nuke)

If you ever want to remove Hermes completely:

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

EMRG bot is untouched by any of the above — it runs as `deploy` under PM2, completely separate.
