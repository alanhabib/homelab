# Homelab: a Kubernetes platform on an old laptop

This repository documents and holds everything needed to turn an old Windows laptop into an always-on **Linux server running Kubernetes**, built entirely from code and reachable safely from anywhere.

It is written as a **learning journey**: I started with almost no Linux experience, and every step, command and mistake is written down so you can follow along from zero.

> **Status:** Phase 0–2 done (Ubuntu installed, connected from the MacBook). Phase 3 (this repo) in progress.

> **About this repo:** This is the **public** repository with my real configuration. Secrets are never committed: they stay out via .gitignore, or are encrypted with Sealed Secrets. 

---

## What we are building

The goal is the **platform**, not any single app. Apps (and maybe a local AI agent later) are bonuses that run on top.

```mermaid
flowchart TB
    you["You, anywhere<br/>MacBook or phone"]
    cf["Cloudflare<br/>login + encrypted tunnel"]

    subgraph home["Home network"]
        router["Telia router<br/>no open ports"]
        subgraph server["ThinkPad server: Ubuntu Server LTS"]
            docker["Docker"]
            subgraph k8s["k3d Kubernetes cluster: 1 control plane + 2 workers"]
                argo["Argo CD<br/>GitOps"]
                apps["Apps<br/>Uptime Kuma, ..."]
                mon["Monitoring<br/>Prometheus + Grafana"]
                tunnel["cloudflared<br/>tunnel connector"]
            end
            backup["restic backups"]
        end
    end

    git["GitHub repo<br/>all config as code"]

    you --> cf --> tunnel
    git -- "Argo CD pulls changes" --> argo
    argo --> apps
    argo --> mon
    docker --> k8s
```

**Key ideas**

- **Everything as code.** Server setup (Ansible), cluster (k3d config), apps (Kubernetes manifests) and Cloudflare (OpenTofu) all live in this repo. If the server dies, it can be rebuilt from Git.
- **No open ports.** Remote access goes through an outbound Cloudflare Tunnel with a login in front, never port forwarding.
- **Free and open source.** The only costs are a domain name (about 100–150 kr a year) and electricity.

---

## The roadmap

```mermaid
flowchart LR
    p0["0 Prepare<br/>USB stick"]:::done
    p1["1 Install<br/>Ubuntu"]:::done
    p2["2 Connect<br/>from Mac"]:::done
    p3["3 Git repo"]:::now
    p4["4 Ansible"]:::todo
    p5["5 k3d cluster"]:::todo
    p6["6 First app"]:::todo
    p7["7 Argo CD"]:::todo
    p8["8 Sealed Secrets"]:::todo
    p9["9 Cloudflare"]:::todo
    p10["10 OpenTofu"]:::todo
    p11["11 Monitoring"]:::todo
    p12["12 Backups"]:::todo
    p13["13 CI"]:::todo

    p0 --> p1 --> p2 --> p3 --> p4 --> p5 --> p6 --> p7
    p7 --> p8 --> p9 --> p10 --> p11 --> p12 --> p13

    classDef done fill:#d4edda,stroke:#28a745,color:#155724
    classDef now fill:#fff3cd,stroke:#ffc107,color:#856404
    classDef todo fill:#f1f3f5,stroke:#adb5bd,color:#495057
```

Green = done, yellow = in progress, grey = not started.

---

## Guides

Start with the decisions, then follow the phases in order. **Guide numbers match the phase numbers in the roadmap.** Each guide follows the same pattern: **goal → steps → what each command does → how to check it worked (Done when) → problems I hit**.

| Phase | Guide | What you learn |
| --- | --- | --- |
| – | [Decisions: what we build and why](docs/decisions.md) | Why Ubuntu, k3d, Cloudflare, platform-first; every tool in one line |
| 0 | [Prepare the USB stick](docs/00-prepare-usb.md) | Download Ubuntu, write it to a USB stick from a Mac |
| 1 | [Install Ubuntu Server](docs/01-install-ubuntu.md) | BIOS, every installer screen, first login, growing the disk |
| 2 | [Network, SSH and running headless](docs/02-network-and-ssh.md) | Wi-Fi with netplan, SSH keys, laptop lid, mesh vs main router, fixed IP |
| 3 | The Git repository *(in progress)* | Git basics, the repo structure, the first commit |
| – | [Pitfalls and fixes](docs/pitfalls.md) | Every mistake and surprise so far, a troubleshooting flowchart, golden rules |

Screenshots live in [`docs/images/`](docs/images/README.md), with secrets blurred.

---

## The hardware

| Part | What it is | What it means |
| --- | --- | --- |
| Computer | Lenovo ThinkPad T480s (old Windows laptop) | Quiet, low power, battery works as a built-in UPS |
| CPU | Intel Core i5-8350U, 4 cores / 8 threads | Plenty for a homelab |
| RAM | 8 GB, soldered, plus one free slot (max 24 GB) | The main limit; a 16 GB stick is the best upgrade |
| Disk | 256 GB NVMe SSD | Enough to start |
| GPU | Intel UHD 620 (built in), no NVIDIA | A local LLM is not realistic on this machine |
| Network | Wi-Fi to the main router for now; Ethernet later | |

The day-to-day work happens from a **MacBook** over SSH. The server has no desktop and its lid stays closed.

---

## Conventions used in the guides

- Every command says **where it runs**:
  - **Mac**: the prompt ends with `%`, for example `alan@MacBook-Pro ~ %`
  - **Server**: the prompt ends with `$`, for example `user@homelab:~$`
- Placeholders in `<ANGLE_BRACKETS>` are replaced with your own values:

| Placeholder | Meaning | Example |
| --- | --- | --- |
| `<SERVER_IP>` | Fixed home-network address of the server | `192.168.1.193` |
| `<LAN_CIDR>` | The home network range | `192.168.1.0/24` |
| `<USER>` | Your Linux username on the server | `alan` |
| `<WIFI_NAME>` | The Wi-Fi network the server joins | `Telia-XXXXXX` |

- **Never** commit or show passwords, keys, tokens, your public IP address or router stickers. Blur them in screenshots and videos.

---

## Repository layout

```
homelab/
├── README.md           ← you are here
├── docs/               ← the step-by-step guides (NN = phase number)
│   ├── decisions.md
│   ├── 00-prepare-usb.md, 01-install-ubuntu.md, 02-network-and-ssh.md, ...
│   ├── pitfalls.md
│   └── images/         ← screenshots, secrets blurred
├── ansible/            ← server configuration (Phase 4)
├── k3d/                ← cluster definition (Phase 5)
├── bootstrap/          ← the one Argo CD app applied by hand (Phase 7)
├── argocd-apps/        ← one Argo CD Application per app
├── apps/               ← Kubernetes manifests for apps
├── infra/              ← platform parts: cloudflared, monitoring
├── terraform/          ← Cloudflare as code with OpenTofu (Phase 10)
└── .github/workflows/  ← CI checks (Phase 13)
```
