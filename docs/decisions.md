# Decisions: what we build and why

**Goal:** decide what to build and which tools to use, before touching any hardware.

[← Back to README](../README.md) · [Next: Phase 0, Prepare the USB stick →](00-prepare-usb.md)

---

## How the idea changed

The project started as *"run my own LLM agent at home and reach it from anywhere"*. While planning, it became clear that the interesting, lasting part is the **platform underneath**: a Kubernetes setup built entirely from code. The AI agent became an optional bonus.

```mermaid
flowchart TD
    a["Idea: run my own LLM agent at home"] --> b{"Is the hardware good enough<br/>for a useful local model?"}
    b -- "Unknown at first" --> c["Plan around the platform<br/>so nothing depends on a GPU"]
    c --> d["Goal: Kubernetes platform<br/>built from code"]
    d --> e["Apps on top are bonuses:<br/>Uptime Kuma, photos, passwords, ..."]
    d --> f["Local AI agent later,<br/>only if the hardware allows"]
    b -- "Checked later: 8 GB RAM, no NVIDIA GPU" --> f
```

---

## The decisions, and why

| Question | Decision | Why | Alternatives considered |
| --- | --- | --- | --- |
| Which Linux? | **Ubuntu Server LTS** | 5 years of free updates, best NVIDIA and Kubernetes support, most tutorials assume it | Fedora (new release about every 13 months), Omarchy (a desktop, not a server) |
| Desktop on the server? | **No**, server edition only | Saves RAM; everything is done from the Mac. A desktop can be added later with one command | Ubuntu Desktop |
| Which Kubernetes? | **K3s, installed directly on the server** | Runs like real infrastructure, uses less RAM than k3d, and a second computer can join later as a real second node | k3d (Kubernetes nodes as Docker containers: multi-node on one machine, but mainly a dev tool), Proxmox with VMs (rejected: the goal is Kubernetes, not a hypervisor) |
| Multi-node? | **Later, with a second computer** | One machine = one real node. A second computer joins as a K3s agent; multiple clusters would need a third | Faking nodes with k3d |
| sudo for Ansible | **`sudo.ws`** (classic sudo) | Ubuntu 26.04's new sudo-rs breaks Ansible's password prompt | Waiting for an Ansible fix |
| Remote access | **Cloudflare Tunnel + Cloudflare Access** | Free, no open router ports, login with MFA in front of everything | Tailscale, port forwarding (unsafe) |
| Configure the server | **Ansible** | The whole server setup in a file; re-runnable and rebuildable | Doing it by hand |
| Things outside the server | **OpenTofu** (open-source Terraform) | Cloudflare DNS and access rules as code | Clicking in the dashboard |
| Deploy apps | **Argo CD (GitOps)** | Git is the single source of truth; push a change and the cluster follows | `kubectl apply` by hand |
| Secrets in Git | **Sealed Secrets** | Secrets are encrypted so only the cluster can read them | GitHub Secrets (those are for CI, not for the cluster) |
| CI | **GitHub Actions** | Free checks on every push | – |
| Backups | **restic** to an external disk | Simple, encrypted, free | – |

### Is it all free?

Yes, all the software is free and open source. The only real costs:

- a domain name: about 100–150 kr a year
- electricity for a machine that is always on
- optional: a 16 GB RAM stick, and an external backup disk

---

## Tools explained in one line each

| Tool | In one line |
| --- | --- |
| **Containers** | Programs packed with everything they need, running isolated from each other |
| **Kubernetes** | Runs and looks after many containers across several machines (nodes) |
| **K3s** | A small, lightweight Kubernetes you install directly on a machine |
| **Ingress / Traefik** | The front door that routes web requests to the right app |
| **TLS** | The encryption behind `https://` |
| **Ansible** | Describes how a server should be set up, then makes it so |
| **OpenTofu** | Describes cloud resources (like Cloudflare settings), then creates them |
| **Argo CD** | Watches the Git repo and makes the cluster match it |
| **Sealed Secrets** | Encrypts secrets so they are safe to store in Git |
| **CI/CD** | Automatic checks (CI) and automatic deployment (CD) on every change |
| **Backups** | Copies of data stored somewhere else, so a broken disk isn't the end |

---

## About the local AI agent

The long-term dream is a terminal agent, similar to Claude Code, running on a free open-source model on this server and reachable with `ssh` from any computer. The research showed:

- Open-source terminal agents exist (for example OpenCode and Aider) and can use local models through Ollama.
- Useful coding models need a graphics card with roughly 16–24 GB of video memory.
- This laptop has 8 GB of RAM and no NVIDIA GPU, so the agent waits until there is better hardware.

That is why the platform comes first.

---

## How the plan changed along the way

- **k3d -> K3s (November 2026).** I first planned k3d to fake three nodes on one laptop. I switched to K3s directly on the server: it's how Kubernetes runs on real machines, it uses less of my 8 GB, and a second computer can join as a real node. The full reasoning, and the four rules that keep a later migration easy, are in my learning roadmap.
- **The career goal.** Learn Kubernetes here, then move the platform to **Azure** (AKS), aiming for AZ-104 and CKA.

---

**Done when:** the tools are chosen and the order of the phases is clear.
