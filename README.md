# Server Init

A collection of security hardening steps and configurations for setting up a new VPS (Virtual Private Server) for secure development and deployment.

## Purpose

This repository documents the complete setup process for securing and configuring an Ubuntu VPS, including:
- Security hardening measures
- Development environment setup
- SSH and GitHub configuration
- Monitoring and maintenance procedures

## Contents

- `init.md` - Comprehensive VPS setup and hardening guide with step-by-step instructions
- `HERMES.md` - Operational reference for running a Hermes Agent gateway on the box (assumes Hermes is already installed; points to upstream for fresh install)

## Usage

1. Start with `init.md` on a fresh Ubuntu VPS — it walks through base install, user creation, SSH hardening, firewall, and other defense-in-depth measures.
2. If you also want to run a Hermes Agent on the box, follow the upstream Hermes Agent install instructions and use `HERMES.md` as your day-to-day ops cheat sheet.