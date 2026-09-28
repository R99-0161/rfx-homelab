# R.F.X.

My own agentic AI assistant workflow — a self-hosted automation system running on a home Proxmox server, with n8n and powered by an LLM agent, reachable through Telegram.

## Overview

R.F.X. is my attempt at building a personal agentic AI assistant from scratch — not just a chatbot, but a system that can reason through multi-step tasks, call tools, and act on my homelab. It runs on a Dell OptiPlex that's part of my home server setup, with an LLM agent (Gemini, called via an n8n Agent node) able to monitor infrastructure, run workflows, and eventually hold a real conversation with me from my phone.

## Stack

- **Hardware:** Dell OptiPlex 3060 Micro
- **Hypervisor:** Proxmox
- **VM:** Debian 13 ("debian-docker") running Docker
- **Orchestration:** n8n
- **AI layer:** Gemini AI Agent node, callable as a tool from n8n workflows
- **Interface:** Telegram bot (in progress)

OptiPlex (Proxmox host) → Debian VM (Docker) → n8n → Gemini AI Agent → Telegram

## A Bug I Hit and Fixed: Realtek NIC Link-Flapping

Early on, the OptiPlex kept dropping its network link intermittently — the classic symptoms of flapping, with nothing obviously wrong at a glance. Since Proxmox's own web UI, SSH access, and every VM's network traffic all ride on that same link, a flap doesn't just look like a blip — it can drop management access mid-task.

**Diagnosis:** checking `dmesg` showed a repeated pattern with no correlation to network load:

`dmesg | grep -i eth`

Output showed repeated cycles of `r8169 0000:00:1f.6 eth0: Link is Down` followed by `Link is Up - 1Gbps/Full - flow control off`. No cable issue, no switch-side problem, no load pattern — which pointed toward a negotiation issue rather than a physical layer fault.

The OptiPlex uses a **Realtek RTL8168h** chipset, which has a known history of instability with **EEE (Energy Efficient Ethernet)** — a power-saving feature that renegotiates link state during idle periods. On some Realtek chips, that renegotiation is unreliable and produces exactly this kind of flap. Confirmed with `ethtool --show-eee eth0`.

**Root cause:** EEE negotiation instability on the RTL8168h — the NIC and the switch weren't reliably agreeing on low-power link state.

**The fix:** `ethtool --set-eee eth0 eee off` solves it immediately, but doesn't survive a reboot on its own. For a server meant to run unattended, that's not good enough — so I built a systemd oneshot unit to apply it automatically at every boot:
/etc/systemd/system/disable-eee.service

[Unit]
Description=Disable EEE on eth0 to prevent Realtek NIC link flapping
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool --set-eee eth0 eee off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target


Enabled with:

`systemctl daemon-reload && systemctl enable --now disable-eee.service`

**Why this shape:** `Type=oneshot` + `RemainAfterExit=yes` is the correct systemd pattern for a run-once config task rather than a long-lived daemon. `After=network.target` avoids a race condition where the service fires before `eth0` exists. `WantedBy=multi-user.target` ties it to normal boot so it's automatic — no manual step, no cron job, and it's inspectable via `systemctl status` like any other service. No recurring drops since.

## Current Status

- Proxmox API integration for read-only host/VM monitoring: working, not yet fully validated end-to-end
- Access is scoped to a dedicated `rfx@pam!rfx` API token with read-only permissions — no write capability yet, deliberately
- Telegram bot integration: in progress
- Known constraint: the OptiPlex only has 8 GB RAM, which caps how much I can run alongside the VM

## Key Decisions So Far

- n8n Merge nodes run in **Append** mode
- Code nodes are JavaScript-only (Python 3 isn't installed on this box)
- HTTP Request nodes have `Continue on Fail` enabled so one bad response doesn't kill a whole workflow
- Read-only API access first, write capability introduced later once the monitoring side is trustworthy

## What's Next

- Finish validating the Proxmox monitoring workflow end-to-end
- Finish the Telegram integration
- Introduce write capabilities once read-only monitoring is solid
- Scale beyond the OptiPlex

  <img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/99754b2d-efea-4f6f-804e-96120d0f32cd" />
