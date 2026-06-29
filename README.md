# Homelab
![Status](https://img.shields.io/badge/status-active-brightgreen) ![Proxmox](https://img.shields.io/badge/Proxmox-VE-orange) ![Linux](https://img.shields.io/badge/Linux-Debian-lightgrey) ![AI](https://img.shields.io/badge/AI-Ollama-green) ![Experimental](https://img.shields.io/badge/status-experimental-orange)

A personal homelab built for learning, experimentation and self-hosting. 

This repository documents the setup, configuration, and troubleshooting of my homelab infrastructure — both as a personal knowledge base and as a way to share real-world learning (including failures).

---

## Highlights

- Local AI agent system using Ollama + Open WebUI + n8n
- LLM-powered command execution via workflow automation
- GPU passthrough (GTX 1080) for local AI inference
- Containerized services managed with Docker + Portainer
- Secure remote access using Tailscale VPN
- Real-world debugging of AI reliability and system integration

---

## Why this exists

I built this homelab to move beyond tutorials and actually understand how systems behave in practice — including when they break.

This repository focuses not just on working setups, but also:
- what failed
- why it failed
- how it was fixed

---

## Architecture Overview

```
Proxmox VE Host
├── Debian VM
│   ├── Ollama          (local LLM inference)
│   ├── Open WebUI      (chat interface)
│   └── n8n             (workflow automation / agent ecexution)
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

## Quick Navigation

- `proxmox/setup.md` — hypervisor setup  
- `debian-vm/` — AI stack and services  
- `lxc-containers/` — containerized infrastructure  
- `network/tailscale.md` — remote access setup  
- `troubleshooting/` — solved issues and fixes

---

## Goals

- Learn Linux, Docker, and self-hosting hands-on
- Build a local AI agent system (Ollama + Open WebUI + n8n)
- Document everything for future reference and to share with others

---

## Status

🟢 Running: Ollama, Open WebUI, n8n, Portainer, AdGuard Home, Tailscale  
🟡 Planned: Active Directory lab, Continue.dev + VSCode   
🔴 Experimental: AI agent workflow (command execution reliability issues)

---

## Notes

This homelab is a learning environment. 
Configuration choices prioritize understanding over best practices — things are done the long way on purpose.

