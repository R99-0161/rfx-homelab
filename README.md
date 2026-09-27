# rfx-homelab

A self-hosted personal AI assistant and automation system running on a home Proxmox server, built around n8n as the orchestration layer and reachable through Telegram.

## Overview

R.F.X. started as a way to turn a spare Dell OptiPlex into something more useful than a box collecting dust — a personal automation layer that can monitor my homelab, run workflows, and eventually act as a conversational assistant I can talk to from my phone.

## Stack

- **Hardware:** Dell OptiPlex 3060 Micro
- **Hypervisor:** Proxmox
- **VM:** Debian 13 ("debian-docker") running Docker
- **Orchestration:** n8n
- **AI layer:** Gemini AI Agent node, callable as a tool from n8n workflows
- **Interface:** Telegram bot (in progress)

The rough pipeline looks like this:

## A bug I hit and fixed

Early on, the OptiPlex kept dropping its network link intermittently — classic symptoms of flapping, nothing obviously wrong in the logs at first glance.

Turned out to be the Realtek RTL8168h NIC and its EEE (Energy Efficient Ethernet) negotiation being unstable on this hardware. The fix was a systemd oneshot service that runs at boot to disable EEE on the interface, so it's applied automatically every time the machine starts rather than needing a manual fix after every reboot.

## Current status

- Proxmox API integration for read-only host/VM monitoring: working, not yet fully validated end-to-end
- Access is scoped to a dedicated `rfx@pam!rfx` API token with read-only permissions — no write capability yet, deliberately
- Telegram bot integration: in progress
- Known constraint: the OptiPlex only has 8 GB RAM, which caps how much I can run alongside the VM

## Key decisions so far

- n8n Merge nodes run in **Append** mode
- Code nodes are JavaScript-only (Python 3 isn't installed on this box)
- HTTP Request nodes have `Continue on Fail` enabled so one bad response doesn't kill a whole workflow
- Read-only API access first, write capability introduced later once the monitoring side is trustworthy

## What's next

- Finish validating the Proxmox monitoring workflow end-to-end
- Finish the Telegram integration
- Introduce write capabilities once read-only monitoring is solid
- Possibly scale beyond the OptiPlex if the 8 GB ceiling becomes a real bottleneck

  <img width="1919" height="960" alt="image" src="https://github.com/user-attachments/assets/99754b2d-efea-4f6f-804e-96120d0f32cd" />
