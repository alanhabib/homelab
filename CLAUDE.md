# CLAUDE.md

Context for Claude when working in this repository. Read this first, then `README.md` and `docs/`.

## The project

A **Kubernetes homelab platform built entirely from code**, on an old laptop at home, reachable safely from anywhere. The platform is the goal; apps on top are bonuses. A local LLM agent is a possible later bonus, but **not realistic on the current hardware**, so it is out of scope for now.

The owner is documenting the journey (guides in `docs/`, and a YouTube series), so every step should be clear enough for a beginner to follow.

**Career goal:** Azure cloud infrastructure. Learn Kubernetes in this homelab first, then rebuild the platform on AKS (Azure Kubernetes Service) with OpenTofu. Target certificates: AZ-104, then CKA.

## How to work with me

- I'm **learning Linux and Kubernetes**. Give **one step at a time**, explain what it does and each part of it, and tell me what output to expect. Wait for my result before the next step.
- Use **simple, everyday words** and give an example. If a technical word is needed, explain it in plain words. Don't make me guess.
- I **write every file myself**. Never write or change files in the repo. Explain what to write and why, and I type it.
- When I say **"done"**, read the files I changed, give feedback, then give the next step.
- When I say **"proceed"**, the previous step worked. Don't ask me to confirm it; go to the next step.
- Once a command has been shown, don't write it out again. Just name it (for example "run the playbook"), so I learn to remember it.
- I **know Git**. Don't explain Git or ask me to run `git status`; just say "commit and push" at the right moments.
- Say **where each command runs**: **Mac** (zsh, prompt ends `%`) or **Server** (bash, prompt ends `$`). Linux-only tools (`ip`, `apt`, `systemctl`) don't exist on the Mac.
- I edit files with **Vim** (not nano). Remind me of `:set paste` for indented YAML.
- Swedish Mac keyboard: `|` is Option+7, `~` is Option+¨.
- When we build something, mention briefly what the **Azure** equivalent is called.
- Ask before anything destructive or hard to undo (deleting files, wiping disks, changing SSH or firewall rules). When changing SSH, firewall or network settings, make sure I keep a way back in (a second SSH session, or the laptop's own keyboard).
- Keep docs up to date: after each phase, I update the matching `docs/NN-*.md` guide (goal → steps → what each command does → Done when → Problems I hit) and add new entries to `docs/pitfalls.md`. Guide numbers match phase numbers. Review what I write.

## Never

- **This repo is public.** Everything in it, including the Git history, is visible to anyone. Check `git diff --staged` for secrets before every commit.
- Never commit or print passwords, Wi-Fi keys, SSH private keys, tokens, the Sealed Secrets master key, kubeconfigs, OpenTofu state, or the home's public IP. `.gitignore` covers the known ones.
- Never open ports on the router. Remote access goes only through Cloudflare Tunnel with Cloudflare Access in front.

## Machines and network

| Item | Value |
| --- | --- |
| Workstation | MacBook (user `alanhabib`), repo at `~/Desktop/hobby_projects/homelab`, GitHub `alanhabib/homelab` (public, HTTPS remote) |
| Server | Lenovo ThinkPad T480s, hostname `homelab`, user `alandalo` |
| OS | Ubuntu Server 26.04.1 LTS, no desktop |
| CPU / RAM | Intel i5-8350U (4 cores / 8 threads), **8 GB RAM** (soldered; one free slot, max 24 GB) |
| GPU | Intel UHD 620 only, no NVIDIA |
| Disk | 256 GB NVMe, LVM: root `/dev/ubuntu-vg/ubuntu-lv` grown to ~177G, ~58G still free in `ubuntu-vg` |
| Server IP | `192.168.1.193` (static lease in the Telia router) |
| LAN | `192.168.1.0/24`, gateway/router `192.168.1.1` (Telia F1x, admin at `http://teliagateway.lan`) |
| Server network | **Wi-Fi** `wlp61s0` to the Telia router (Ethernet planned later; Ethernet port unused) |
| Mac network | Usually on the Jensen mesh Wi-Fi, which runs its own network `192.168.39.0/24` behind the Telia router (double NAT). Mac → server works; server → Mac does not |
| SSH from Mac | `ssh homelab` (key `~/.ssh/id_ed25519`, label `alan-macbook`, config in `~/.ssh/config`). Password login over SSH is **off** |
| Second computer | Planned. Will join the cluster as a K3s agent (worker) node |

**RAM is the main constraint.** Keep the cluster lean: few replicas, modest resource requests, short monitoring retention.

## What has been done

- **Phase 0:** USB stick made on the Mac with balenaEtcher.
- **Phase 1:** Windows erased; Ubuntu Server installed (no network during install, OpenSSH server installed, no snaps, no disk encryption); fully updated; LVM grown by 80G.
- **Phase 2:** Wi-Fi via `/etc/netplan/02-wifi.yaml` (mode 600, contains the Wi-Fi password, **not in Git**); SSH key login from the Mac; lid setting so the server runs with the lid closed; moved from the mesh to the Telia Wi-Fi; static lease for `192.168.1.193`.
- **Phase 3:** repo created and cloned; docs, `.gitignore` and the folder structure added. Repo made **public** with an MIT license; `.DS_Store` removed from Git.
- **Phase 4 (in progress):** `ansible/` contains:
    - `ansible.cfg`: inventory file, `interpreter_python = auto_silent`, and `[privilege_escalation] become_exe = sudo.ws`
    - `inventory.ini`: group `[servers]` with host `homelab` (`ansible_host=192.168.1.193 ansible_user=alandalo`)
    - `requirements.yml`: `community.general`, `ansible.posix`
    - `site.yml` tasks: timezone `Europe/Stockholm`; updates (`update_cache`, `cache_valid_time: 3600`, `upgrade: dist`, `autoremove`); base packages (curl, git, htop, ufw, unattended-upgrades); automatic security updates (`20auto-upgrades`); SSH hardening (`10-hardening.conf`: no passwords, no root login; **done and verified**); lid setting (`lid.conf`, written; dry run pending)
    - Handlers: `Restart ssh`, `Restart logind`
    - Run with `--ask-become-pass`; dry run with `--check --diff`

The Wi-Fi settings were made **by hand** and are still outside Ansible (the Wi-Fi password must not be committed in plain text; use Ansible Vault if it is ever brought in).

## Known open issues

- **Ubuntu 26.04 uses sudo-rs by default**, which breaks Ansible's password prompt ("Timeout waiting for privilege escalation prompt"). Fixed in `ansible.cfg` with `become_exe = sudo.ws` (the classic sudo, still installed). Remove when Ansible supports sudo-rs.
- **Slow boot:** after a reboot, "System is booting up. Unprivileged users are not permitted to log in yet" for several minutes. Most likely `systemd-networkd-wait-online` waiting for the unconnected Ethernet port. Confirm with `systemd-analyze blame`, then fix in Ansible (for example mark Ethernet `optional: true` in netplan, or make wait-online wait for any one interface).
- **Firewall (ufw) is not configured yet.**
- **No backups yet** (Phase 12).

## The plan

**Learning roadmap (the master plan, kept up to date):** https://claude.ai/artifact/TLFpBth7zacygwzFTFg8NF . It groups the phases into six stages (A server, B Kubernetes basics, C GitOps and secrets, D reach it safely, E run it like a pro, F Azure), with a career track and a parking lot for ideas that are not needed yet. New ideas go to the parking lot, not into the current step.

| Phase | What | Notes |
| --- | --- | --- |
| **4 (now)** | **Ansible** configures the server | Remaining tasks: lid dry run, kernel settings (inotify limits), firewall, slow-boot fix. Firewall (ufw): SSH 22 and Kubernetes API 6443 from `192.168.1.0/24`; web traffic 80 and 443; allow K3s's internal networks (pods `10.42.0.0/16`, services `10.43.0.0/16`); deny the rest. Extra node ports when the second computer joins. Second run of the playbook should be mostly `ok` |
| 5 | **K3s** cluster | K3s installed directly on the ThinkPad (server node), installed via Ansible. A second computer joins later as an agent node. Kubeconfig to the Mac at `~/.kube/homelab` |
| 6 | First app by hand: Uptime Kuma | `apps/uptime-kuma/`, image `louislam/uptime-kuma:2`, PVC on `local-path`, Ingress `kuma.192.168.1.193.nip.io` (port 80) |
| 7 | **Argo CD** (GitOps) | Helm install once, read-only deploy key, app-of-apps: `bootstrap/root.yaml` watches `argocd-apps/` |
| 8 | **Sealed Secrets** | Via Argo CD; back up the master key outside Git and the server |
| 9 | **Cloudflare** Tunnel + Access | Domain via Cloudflare; `cloudflared` in the cluster (`infra/cloudflared/`) with a sealed token; Access policy before any public hostname; SSH via `ssh.<domain>` |
| 10 | **OpenTofu** for Cloudflare | `terraform/`, provider `cloudflare/cloudflare ~> 5.0`; state stays on the Mac |
| 11 | Monitoring | kube-prometheus-stack via Argo CD, K3s-irrelevant components disabled, short retention; alerts via Uptime Kuma → ntfy |
| 12 | Backups | restic to an external disk at `/mnt/backup`, systemd timer from Ansible, test restores. Back up K3s app data (`/var/lib/rancher/k3s/storage` by default) |
| 13 | CI | GitHub Actions: yamllint, kubeconform, ansible-lint |
| 14 | Routine | Weekly/monthly checks, restore and rebuild drills |
| 15 | **Azure** | Rebuild the platform on AKS with OpenTofu, Argo CD pointing at the same repo. Budget alerts first; tear down when not in use |

The full step-by-step runbook (written for k3d) exists as a separate document; the table above is the current agreed design and wins where they differ.

## Decisions (short)

Ubuntu Server LTS (not Fedora/Omarchy) · no desktop · **K3s directly on the server** (not k3d), so the cluster runs like real infrastructure; multi-node when a second computer joins as an agent; multi-cluster would need a third machine · Ansible for the host · `sudo.ws` for Ansible's become (sudo-rs issue) · Argo CD for apps · Sealed Secrets · Cloudflare Tunnel + Access (no port forwarding) · OpenTofu for Cloudflare · restic backups · all free and open source. Reasons are in `docs/decisions.md`.
