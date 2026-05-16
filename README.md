# Server Init

A collection of security hardening steps and configurations for setting up a new VPS (Virtual Private Server) for secure development and deployment.

## Purpose

This repository documents the complete setup process for securing and configuring an Ubuntu VPS, including:
- Security hardening measures
- Development environment setup
- SSH and GitHub configuration
- Monitoring and maintenance procedures

## Contents

- `init.md` — VPS setup and hardening playbook (firewall, SSH, user creation, fail2ban, sysctl, etc.)
- `hermes-setup.md` — Install playbook for a [Hermes Agent](https://github.com/NousResearch/hermes-agent) gateway with Discord + Telegram on the same box
- `hermes-ops.md` — Day-to-day operational reference for the running Hermes gateway (logs, config, troubleshooting, rollback)

## Usage

1. Start with `init.md` on a fresh Ubuntu VPS — it walks through base install, user creation, SSH hardening, firewall, and other defense-in-depth measures.
2. If you want to run a Hermes Agent on the box, follow `hermes-setup.md` after `init.md` is done.
3. Once Hermes is running, keep `hermes-ops.md` handy as the day-to-day cheat sheet.