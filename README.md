# Homelab

A personal homelab built for learning and self-hosting. This repository documents the setup, configuration, and troubleshooting of my homelab infrastructure — both as a personal reference and for anyone interested in building something similar.

---

## Architecture Overview

```
Proxmox VE Host
├── Debian VM
│   ├── Ollama          (local LLM inference)
│   ├── Open WebUI      (chat interface)
│   └── n8n             (workflow automation)
│
└── LXC Container  (Docker host)
    ├── Portainer       (container management)
    └── AdGuard Home    (DNS-level ad blocking)

All services accessed via web browser from main PC and mobile via Tailscale VPN
```

---

## Hardware

| Component | Details |
|---|---|
| CPU | Intel Core i5-8600K |
| Motherboard | ASUS Strix Z370-H Gaming |
| RAM | 32GB DDR4 |
| Storage | 2x 256GB NVMe |
| GPU | ASUS Strix GTX 1080 — 8GB VRAM (passed through to Debian VM) |
| PSU | Silverstone 700W Strider Essential |
| OS | Proxmox VE |

---

## Services

| Service | Host | Purpose |
|---|---|---|
| Ollama | Debian VM | Running local LLMs |
| Open WebUI | Debian VM | Web interface for Ollama |
| n8n | Debian VM | Workflow and AI agent automation |
| Portainer | LXC Container | Docker container management |
| AdGuard Home | LXC Container | Network-wide DNS ad blocking |

---

## Repository Structure

```
homelab/
├── README.md
├── proxmox/
│   └── setup.md
├── debian-vm/
│   ├── ollama.md
│   ├── openwebui.md
│   └── n8n.md
├── lxc-containers/
│   ├── docker-host.md
│   ├── portainer.md
│   ├── adguard.md
│   └── troubleshooting/
│       └── portainer-password-reset.md
└── network/
    └── tailscale.md
```

> ⚠️ Note: Documentation is a work in progress. Some sections may be incomplete as I continue building out the lab.

---

## Goals

- Learn Linux, Docker, and self-hosting hands-on
- Build a local AI agent system (Ollama + Open WebUI + n8n)
- Document everything for future reference and to share with others

---

## Status

🟢 Running: Ollama, Open WebUI, n8n, Portainer, AdGuard Home, Tailscale  
🟡 Planned: Active Directory lab, Continue.dev + VSCode

---

## Notes

This homelab is a learning environment. Configuration choices prioritize understanding over best practices — things are done the long way on purpose.
