# CLAUDE.md

Context for Claude when working in this repository. Read this first, then `README.md` and `docs/`.

## The project

A **Kubernetes homelab platform built entirely from code**, on an old laptop at home, reachable safely from anywhere. The platform is the goal; apps on top are bonuses. A local LLM agent is a possible later bonus, but **not realistic on the current hardware**, so it is out of scope for now.

The owner is documenting the journey (guides in `docs/`, and a YouTube series), so every step should be clear enough for a beginner to follow.

## How to work with me

- I'm **learning Linux and Kubernetes**. For Linux, Ansible and Kubernetes commands: give **one command at a time**, explain what it does and each part of it, and tell me what output to expect. Wait for my result before the next step.
- I **know Git**. Don't explain Git or ask me to run `git status`; just say "commit and push" at the right moments.
- Say **where each command runs**: **Mac** (zsh, prompt ends `%`) or **Server** (bash, prompt ends `$`). Linux-only tools (`ip`, `apt`, `systemctl`) don't exist on the Mac.
- I edit files with **Vim** (not nano). Remind me of `:set paste` for indented YAML.
- I'd rather **run commands myself** to learn. Write files for me when useful, but let me run the commands unless I ask you to.
- Swedish Mac keyboard: `|` is Option+7, `~` is Option+¨.
- Ask before anything destructive or hard to undo (deleting files, wiping disks, changing SSH or firewall rules). When changing SSH, firewall or network settings, make sure I keep a way back in (a second SSH session, or the laptop's own keyboard).
- Keep docs up to date: after each phase, update the matching `docs/NN-*.md` guide (goal → steps → what each command does → Done when → Problems I hit) and add new entries to `docs/pitfalls.md`. Guide numbers match phase numbers.

## Never

- Never commit or print passwords, Wi-Fi keys, SSH private keys, tokens, the Sealed Secrets master key, kubeconfigs, OpenTofu state, or the home's public IP. `.gitignore` covers the known ones.
- Never open ports on the router. Remote access goes only through Cloudflare Tunnel with Cloudflare Access in front.

## Machines and network

| Item | Value |
| --- | --- |
| Workstation | MacBook (user `alanhabib`), repo at `~/Desktop/hobby_projects/homelab`, GitHub `alanhabib/homelab` (private, HTTPS remote) |
| Server | Lenovo ThinkPad T480s, hostname `homelab`, user `alandalo` |
| OS | Ubuntu Server 26.04.1 LTS, no desktop |
| CPU / RAM | Intel i5-8350U (4 cores / 8 threads), **8 GB RAM** (soldered; one free slot, max 24 GB) |
| GPU | Intel UHD 620 only, no NVIDIA |
| Disk | 256 GB NVMe, LVM: root `/dev/ubuntu-vg/ubuntu-lv` grown to ~177G, ~58G still free in `ubuntu-vg` |
| Server IP | `192.168.1.193` (static lease in the Telia router) |
| LAN | `192.168.1.0/24`, gateway/router `192.168.1.1` (Telia F1x, admin at `http://teliagateway.lan`) |
| Server network | **Wi-Fi** `wlp61s0` to the Telia router (Ethernet planned later; Ethernet port unused) |
| Mac network | Usually on the Jensen mesh Wi-Fi, which runs its own network `192.168.39.0/24` behind the Telia router (double NAT). Mac → server works; server → Mac does not |
| SSH from Mac | `ssh homelab` (key `~/.ssh/id_ed25519`, label `alan-macbook`, config in `~/.ssh/config`) |

**RAM is the main constraint.** Keep the cluster lean: few replicas, modest resource requests, short monitoring retention.

## What has been done (Phases 0–3)

- **Phase 0:** USB stick made on the Mac with balenaEtcher.
- **Phase 1:** Windows erased; Ubuntu Server installed (no network during install, OpenSSH server installed, no snaps, no disk encryption); fully updated; LVM grown by 80G.
- **Phase 2:** Wi-Fi via `/etc/netplan/02-wifi.yaml` (mode 600, contains the Wi-Fi password, **not in Git**); SSH key login from the Mac; lid setting in `/etc/systemd/logind.conf.d/lid.conf` (`HandleLidSwitch*=ignore`), so the server runs with the lid closed; moved from the mesh to the Telia Wi-Fi; static lease for `192.168.1.193`.
- **Phase 3:** repo created and cloned; docs, `.gitignore` and the folder structure added.

The Wi-Fi and lid settings were made **by hand** on the server. Phase 4 should bring them into Ansible (the Wi-Fi password via Ansible Vault or kept out of Git, not committed in plain text).

## Known open issues

- **Slow boot:** after a reboot, "System is booting up. Unprivileged users are not permitted to log in yet" for several minutes. Most likely `systemd-networkd-wait-online` waiting for the unconnected Ethernet port. Confirm with `systemd-analyze blame`, then fix in Ansible (for example mark Ethernet `optional: true` in netplan, or make wait-online wait for any one interface).
- **Server timezone is UTC**; should be `Europe/Stockholm`.
- **SSH password login is still enabled.** Disable it in Phase 4, only after confirming key login works.
- **Firewall (ufw) is not configured yet.**
- **No backups yet** (Phase 12).

## The plan (Phases 4–14)

| Phase | What | Notes |
| --- | --- | --- |
| **4 (next)** | **Ansible** configures the server | Already installed on the Mac. Files in `ansible/`: `ansible.cfg`, `inventory.ini`, `requirements.yml` (community.general, ansible.posix), `site.yml`. Tasks: apt upgrade, base packages (curl, git, htop, ufw, unattended-upgrades, restic, docker.io, docker-compose-v2), timezone, automatic security updates, SSH hardening (keys only, no root), inotify limits for Kubernetes, Docker + user in docker group, k3d install, `/srv/k3d/storage`, ufw (SSH/6443/8080 from `192.168.1.0/24`, SSH from `172.16.0.0/12` for Cloudflare SSH later, deny the rest), lid setting, slow-boot fix. Run with `--ask-become-pass`; second run should be mostly `ok` |
| 5 | **k3d** multi-node cluster | `k3d/homelab.yaml`: 1 server + 2 agents, API on `192.168.1.193:6443`, port 8080→80 on the load balancer, `/srv/k3d/storage` mounted into all nodes. Kubeconfig to the Mac at `~/.kube/homelab` |
| 6 | First app by hand: Uptime Kuma | `apps/uptime-kuma/`, image `louislam/uptime-kuma:2`, PVC on `local-path`, Ingress `kuma.192.168.1.193.nip.io:8080` |
| 7 | **Argo CD** (GitOps) | Helm install once, read-only deploy key, app-of-apps: `bootstrap/root.yaml` watches `argocd-apps/` |
| 8 | **Sealed Secrets** | Via Argo CD; back up the master key outside Git and the server |
| 9 | **Cloudflare** Tunnel + Access | Domain via Cloudflare; `cloudflared` in the cluster (`infra/cloudflared/`) with a sealed token; Access policy before any public hostname; SSH via `ssh.<domain>` |
| 10 | **OpenTofu** for Cloudflare | `terraform/`, provider `cloudflare/cloudflare ~> 5.0`; state stays on the Mac |
| 11 | Monitoring | kube-prometheus-stack via Argo CD, K3s-irrelevant components disabled, short retention; alerts via Uptime Kuma → ntfy |
| 12 | Backups | restic to an external disk at `/mnt/backup`, systemd timer from Ansible, test restores |
| 13 | CI | GitHub Actions: yamllint, kubeconform, ansible-lint |
| 14 | Routine | Weekly/monthly checks, restore and rebuild drills |

The full step-by-step runbook with all file contents exists as a separate document; the details above are the agreed design.

## Decisions (short)

Ubuntu Server LTS (not Fedora/Omarchy) · no desktop · **k3d** for multi-node on one machine (not Proxmox: the goal is Kubernetes) · Ansible for the host · Argo CD for apps · Sealed Secrets · Cloudflare Tunnel + Access (no port forwarding) · OpenTofu for Cloudflare · restic backups · all free and open source. Reasons are in `docs/decisions.md`.
