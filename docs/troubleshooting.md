# Troubleshooting: something's wrong? Start here

This page is for **right now**: something is broken and I want to fix it. Find what you **see** in the table, jump to it, and follow the checks **in order**. Each check says what a good and a bad result looks like.

For the **history** of every problem I've hit, see [Pitfalls and fixes](pitfalls.md).

[← Back to README](../README.md) · [Pitfalls](pitfalls.md) · [Command cheat sheet](#command-cheat-sheet)

---

## The method: 6 steps for any problem

I use the same steps every time, whatever broke:

1. **Read the error, exactly.** Every word. The last line often says the most. Copy it somewhere.
2. **Where am I?** The prompt tells you: `%` = Mac, `$` = server. Half my "weird" errors were the wrong machine.
3. **Look before you change.** Measure first, with commands that only show things (`status`, `cat`, `ls`, `blame`).
4. **What changed last?** A new file, an update, a reboot, a new network? The last change is the first suspect.
5. **Fix one thing at a time**, and keep a way back in (a second SSH tab, the laptop keyboard).
6. **Verify, then write it down.** Prove it works now, then add a row to [pitfalls](pitfalls.md).

```mermaid
flowchart LR
    a["1 Read the error"] --> b["2 Where am I?"]
    b --> c["3 Look first"]
    c --> d["4 What changed?"]
    d --> e["5 Fix one thing"]
    e --> f["6 Verify, write it down"]
```

---

## Find your symptom

| I see... | Go to |
| --- | --- |
| `ssh homelab` hangs, `Operation timed out`, `No route to host` | [C1](#c1--ssh-hangs-or-times-out) |
| `Permission denied (publickey)` | [C2](#c2--permission-denied-publickey) |
| `System is booting up. Unprivileged users are not permitted to log in yet` | [C3](#c3--system-is-booting-up) |
| `zsh: command not found: ip` (or `apt`, `systemctl`...) | [C4](#c4--command-not-found) |
| My terminal froze in an SSH session | [C5](#c5--the-terminal-froze) |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | [C6](#c6--remote-host-identification-has-changed-not-hit-yet) |
| The server has no internet | [N1](#n1--the-server-has-no-internet) |
| I can't reach a port or a web page on the server | [N2](#n2--i-cant-reach-a-port-or-web-page) |
| Rebooting takes forever | [S1](#s1--rebooting-takes-forever) |
| The server feels slow, or programs get killed | [S2](#s2--the-server-is-slow-or-out-of-memory) |
| `No space left on device` | [S3](#s3--the-disk-is-full) |
| A program or service isn't working | [S4](#s4--a-service-isnt-working) |
| Ansible: `Timeout waiting for privilege escalation prompt` | [A1](#a1--timeout-waiting-for-privilege-escalation-prompt) |
| Ansible: `UNREACHABLE!` | [A2](#a2--unreachable) |
| Ansible: `Incorrect sudo password` | [A3](#a3--incorrect-sudo-password) |
| Ansible: a red error inside a task | [A4](#a4--reading-an-ansible-error) |
| Ansible: the dry run was fine, the real run failed | [A5](#a5--the-dry-run-passed-but-the-real-run-failed) |
| Git keeps tracking a file that's in `.gitignore` | [G1](#g1--gitignore-doesnt-ignore-a-file) |
| `mv: ... Directory not empty` | [G2](#g2--mv-directory-not-empty) |

Some sections are marked **(not hit yet)**: things that commonly go wrong in setups like mine, written down before they happen.

---

## Connecting

### C1 · SSH hangs or times out

**You see:** `ssh homelab` waits, then `Operation timed out` or `No route to host`.

**Check, in this order:**

1. **Is the server on and awake?** Open the lid, look at the screen. A login prompt = on. Black = press a key. Nothing at all = power.
2. **Does the Mac reach it at all?** (Mac)
   ```bash
   ping -c 3 <SERVER_IP>
   ```
   Replies = the network path works, so the problem is SSH or the firewall. No replies = the server isn't on the network, go to step 3.
3. **Did the server's address change?** On the router's admin page, open **Devices** and find `homelab`. Is the IP still `<SERVER_IP>`? If not, the static lease is missing, and `HostName` in `~/.ssh/config` is pointing to an old address.
4. **Am I on another network?** At a café or on a phone hotspot, the firewall only lets in my home network, so SSH is blocked on purpose. (Remote access comes with Cloudflare, Phase 9.)
5. **Is the server's Wi-Fi up?** At the laptop's keyboard: `ip -4 addr show wlp61s0`. No `inet` line = Wi-Fi is down. Check `/etc/netplan/02-wifi.yaml` and run `sudo netplan apply`.
6. **Still stuck? Ask SSH to explain itself.** (Mac)
   ```bash
   ssh -v homelab
   ```
   `-v` = verbose: SSH prints every step. The last lines before it stops show where it got stuck.

**Usual causes:** lid closed without `lid.conf` (asleep) · IP changed · outside the home network · Wi-Fi down.

### C2 · `Permission denied (publickey)`

**What it means:** the server is reachable, but it didn't accept my key. Passwords are off on purpose, so a key is the only way in.

**Check:**

1. **Is the right key used?** (Mac) `ssh -v homelab`, then look for `Offering public key: .../id_ed25519`. If it offers another key, fix `IdentityFile` in `~/.ssh/config`.
2. **Does the key file exist?** (Mac) `ls -l ~/.ssh/id_ed25519`. Missing = the key was deleted or lost.
3. **Is it a new computer?** A new computer has a new key, which the server doesn't know yet.

**Fix for a lost or new key:** follow [Phase 2 · If you lose your SSH key](02-network-and-ssh.md#if-you-lose-your-ssh-key). The laptop's keyboard still takes your password.

### C3 · `System is booting up`

**What it means:** the server is still starting, and Linux keeps logins closed until startup is finished. Normal for a few seconds after a reboot.

**If it lasts minutes:** something in startup is waiting. See [S1](#s1--rebooting-takes-forever). This was my 2-minute boot.

### C4 · `command not found`

**You see:** `zsh: command not found: ip` (or `apt`, `systemctl`, `networkctl`...).

**Cause:** I typed a Linux command on the **Mac**. `zsh` and a prompt ending in `%` = Mac. Fix: `ssh homelab` first, then the prompt ends in `$`.

If it happens **on the server** (`bash: ...: command not found`): the program isn't installed. Install it with `sudo apt install <name>`, and add it to the base packages in the playbook.

### C5 · The terminal froze

**Cause:** the SSH connection died (the server rebooted, changed network, or went to sleep) and the terminal doesn't know yet.

**Fix:** press **Enter**, then **`~`**, then **`.`**. That's SSH's hang-up shortcut. On a Swedish Mac keyboard, `~` is **Option + ¨** (you may need Space after it).

### C6 · `REMOTE HOST IDENTIFICATION HAS CHANGED` (not hit yet)

**What it means:** the server at that address has a different "fingerprint" than the one my Mac remembers. After **reinstalling** the server, that's expected. If you **didn't** reinstall anything, stop and find out why first.

**Fix after a reinstall:** (Mac)

```bash
ssh-keygen -R <SERVER_IP>
```

`-R` = remove the old fingerprint for that address from `~/.ssh/known_hosts`. The next `ssh homelab` asks "Are you sure?" again. Answer `yes`.

---

## Network

### N1 · The server has no internet

**Check, on the server, in order:**

1. **Do I have an address?**
   ```bash
   ip -4 addr show wlp61s0
   ```
   An `inet 192.168.1.x` line = yes. None = Wi-Fi is down (see C1 step 5).
2. **Do I know where the router is?**
   ```bash
   ip route show default
   ```
   `default via 192.168.1.1` = yes. Nothing = no route out.
3. **Can I reach the internet by number?**
   ```bash
   ping -c 3 1.1.1.1
   ```
   `1.1.1.1` is a public DNS server; a number skips name lookups.
4. **Can I reach it by name?**
   ```bash
   ping -c 3 ubuntu.com
   ```

**How to read it:** step 3 works but step 4 fails -> the internet works, but **name lookups (DNS)** don't. Both fail -> the connection out is broken (router, Wi-Fi, internet line).

### N2 · I can't reach a port or web page

**Example:** `kubectl` can't reach the cluster on 6443, or a web page on 80/443 doesn't load.

**Check, on the server:**

1. **Is a program listening on that port?**
   ```bash
   sudo ss -tlnp
   ```
   `ss` = socket statistics. `-t` TCP, `-l` listening only, `-n` numbers instead of names, `-p` show the program. Your port missing from the list = **nothing is listening**: the program isn't running (see S4).
2. **Does the firewall allow it?**
   ```bash
   sudo ufw status verbose
   ```
   Is the port in the list, **from** your network? Not there = add it to the firewall task in the playbook.
3. **Is the firewall blocking someone right now?**
   ```bash
   sudo journalctl -k | grep -i ufw | tail -n 20
   ```
   `journalctl -k` = kernel messages, where ufw logs blocked visitors (`[UFW BLOCK]`). `SRC=` is who knocked, `DPT=` is the door they tried.

**Emergency:** `sudo ufw disable` switches the guard off, for testing only. Switch it back on with the playbook.

---

## Slow, full or broken

### S1 · Rebooting takes forever

**Check, on the server:**

1. **What was slow?**
   ```bash
   systemd-analyze blame | head -n 10
   ```
   Lists the startup steps, slowest first. Look for anything near **2 minutes**: that's a timeout, something waited for something that never came.
2. **If it's `systemd-networkd-wait-online`: what is it waiting for?**
   ```bash
   networkctl
   ```
   Look for a connection stuck at `no-carrier` / `configuring`, for example an Ethernet port without a cable.
3. **Who told it to wait?**
   ```bash
   systemctl cat systemd-networkd-wait-online.service
   ```
   Shows the service and its add-on files. Netplan's file lists the interfaces it waits for (`-i ...`).

**My fix:** mark the unused port `optional: true` in netplan. Full story: [Phase 4, Task 9](04-ansible.md#task-9--the-2-minute-boot-fixed).

### S2 · The server is slow, or out of memory

**Check, on the server:**

1. **Live overview:**
   ```bash
   htop
   ```
   Bars at the top = CPU cores and memory (`Mem`). The list is sorted by CPU; press **`M`** to sort by memory instead, **`q`** to quit. A process at 100% CPU, or memory nearly full = found it.
2. **Memory in numbers:**
   ```bash
   free -h
   ```
   Look at **`available`**, not `free`: Linux uses spare memory as cache and gives it back when needed. `available` near zero = really out of memory.
3. **Did Linux kill something to free memory?**
   ```bash
   sudo journalctl -k | grep -i "out of memory"
   ```
   Lines here = the **OOM killer** ("out of memory") stopped a program. With 8 GB, this is the first thing to suspect once Kubernetes runs.

**Fixes:** stop what you don't need, lower memory use (fewer replicas, smaller limits), or the RAM upgrade.

### S3 · The disk is full

**You see:** `No space left on device`, or updates fail.

**Check, on the server:**

1. **Which disk is full?**
   ```bash
   df -h
   ```
   Look at **`/`** (the system) and **`/boot`** (kernels, only about 2 GB). `Use%` near 100% = full.
2. **What is using the space?**
   ```bash
   sudo du -xh / --max-depth=2 2>/dev/null | sort -h | tail -n 15
   ```
   `du` = disk usage. `-x` stay on this disk, `-h` readable sizes, `--max-depth=2` two folder levels deep. `2>/dev/null` hides "permission denied" noise. `sort -h | tail` = the 15 biggest last.

**Usual causes and fixes:**
- `/boot` full -> old kernels: `sudo apt autoremove` (the playbook does this).
- big logs -> `sudo journalctl --vacuum-size=200M` keeps only the newest 200 MB of logs.
- `/` full while ~58G is still unused in LVM -> grow the volume (see [Phase 1](01-install-ubuntu.md)).

### S4 · A service isn't working

A **service** = a program running in the background (`ssh`, `systemd-logind`, later `k3s`).

**Check, on the server:**

1. **Is it running?**
   ```bash
   systemctl status <service>
   ```
   `active (running)` in green = running. `failed` or `inactive` = not. The last lines show its newest log messages.
2. **What did it say?**
   ```bash
   journalctl -u <service> -n 50 --no-pager
   ```
   `-u` = this service's log, `-n 50` = the last 50 lines, `--no-pager` = print it all at once. Look for words like `error`, `failed`, `denied`.
3. **Any errors anywhere since the last boot?**
   ```bash
   journalctl -b -p err --no-pager
   ```
   `-b` = since this boot, `-p err` = errors only.
4. **Did I just change its config?** Read the file back with `cat`. A typo is silently ignored or stops the service (like `HandleLidSwtichDocked`).

**Fix and retry:** `sudo systemctl restart <service>`, then `systemctl status` again.

---

## Ansible

### A1 · `Timeout waiting for privilege escalation prompt`

**Cause:** Ubuntu 26.04 uses **sudo-rs**, which words its password prompt differently. Ansible waits for its own sentence and never hears it.

**Check:** `ansible.cfg` has

```ini
[privilege_escalation]
become_exe = sudo.ws
```

and the server has `/usr/bin/sudo.ws` (`ssh homelab 'ls -l /usr/bin/sudo*'`).

### A2 · `UNREACHABLE!`

**What it means:** Ansible couldn't even log in.

**Check:**

1. Does `ssh homelab` work by hand? No -> see [C1](#c1--ssh-hangs-or-times-out) / [C2](#c2--permission-denied-publickey). Ansible uses the same SSH.
2. Is the address and user right in `inventory.ini`?
3. Test Ansible alone: `ansible homelab -m ping`, which should answer `"ping": "pong"`.
4. The message mentions `privilege escalation`? -> [A1](#a1--timeout-waiting-for-privilege-escalation-prompt).

### A3 · `Incorrect sudo password`

The `BECOME password` is the **server user's password** (the one sudo asks for), not the SSH key's passphrase and not the Mac's password. Nothing shows while typing; run again and type slowly.

### A4 · Reading an Ansible error

Ansible errors look scary but follow a pattern. **Read from the end:**

```
[ERROR]: Task failed: ... Unsupported parameters for (ansible.legacy.copy) module: model.
Supported parameters include: ..., mode, owner, ...
Origin: .../site.yml:111:7
```

- **`Origin: site.yml:111:7`** -> **which task** failed (its first line), not always the exact wrong line
- **The word in the message** -> what's wrong **inside** that task

| The error says | Usually means | My example |
| --- | --- | --- |
| `Unsupported parameters ... X` | a misspelled setting; the list shows the right words | `model` -> `mode` |
| `has no attribute 'X'` | a loop item is missing a part called `X` | `valie` -> `value` |
| `'item' is undefined` | the `loop:` is in the wrong place, or missing | `loop:` indented too far |
| `mapping values are not allowed` / `did not find expected key` | YAML indentation is off | spaces, Tab, or a missing `-` |
| `couldn't resolve module/action` | a typo in the module name, or a missing collection | run `ansible-galaxy collection install -r requirements.yml` |

**Quick checks before running:**

```bash
ansible-playbook site.yml --syntax-check   # reads the file only, no connection
ansible-playbook site.yml --list-tasks     # task names, in order
ansible-playbook site.yml -K -vvv          # very detailed output when the message is unclear
```

### A5 · The dry run passed, but the real run failed

A dry run (`--check`) only **pretends**. Some things are never really tried: owners and groups, commands, services restarting. Example: `group: rott` passed the dry run and would have failed for real.

**So:** read the task yourself, compare with a working task, and run for real with a second SSH tab open.

---

## Git

### G1 · `.gitignore` doesn't ignore a file

**Cause:** Git was **already tracking** the file before it was added to `.gitignore`. `.gitignore` only stops **new** files.

**Fix:** (Mac) `git rm --cached <file>`, then commit. `--cached` = stop tracking it, keep it on disk.

### G2 · `mv: Directory not empty`

`mv` never overwrites a folder that has files in it, not even with `-f`.

**Fix:** remove the old one first with `rm -r <folder>`, then `mv`. Or make it an exact copy with `rsync -av --delete <new>/ <old>/` (the trailing `/` = "the contents of"; `--delete` removes files that no longer exist).

---

## Kubernetes (coming in Phase 5)

This section will grow with the cluster: `kubectl get`, `describe`, `logs`, `events`, and the classic pod problems (`Pending`, `CrashLoopBackOff`, `ImagePullBackOff`).

---

## Command cheat sheet

Every command on this page, by the question it answers. All on the **server** unless it says Mac.

| Question | Command |
| --- | --- |
| Can the Mac reach the server? (Mac) | `ping -c 3 <SERVER_IP>` |
| Why won't SSH connect? (Mac) | `ssh -v homelab` |
| Forget an old server fingerprint (Mac) | `ssh-keygen -R <SERVER_IP>` |
| What's my address? | `ip -4 addr` |
| Where's the router? | `ip route show default` |
| Is the internet up? Is DNS? | `ping -c 3 1.1.1.1` · `ping -c 3 ubuntu.com` |
| Which network connections are ready? | `networkctl` |
| What is listening on which port? | `sudo ss -tlnp` |
| What does the firewall allow? | `sudo ufw status verbose` |
| What did the firewall block? | `sudo journalctl -k \| grep -i ufw` |
| Who am I connected from? | `echo $SSH_CLIENT` |
| What made startup slow? | `systemd-analyze blame \| head -n 10` |
| How is a service set up? | `systemctl cat <service>` |
| Is a service running? | `systemctl status <service>` |
| What did a service log? | `journalctl -u <service> -n 50 --no-pager` |
| Any errors since boot? | `journalctl -b -p err --no-pager` |
| CPU and memory, live | `htop` |
| Memory in numbers | `free -h` |
| Did something run out of memory? | `sudo journalctl -k \| grep -i "out of memory"` |
| Which disk is full? | `df -h` |
| What uses the space? | `sudo du -xh / --max-depth=2 2>/dev/null \| sort -h \| tail -n 15` |
| How long has it been running? | `uptime` |
| Which kernel is running? | `uname -r` |
| Ansible: can it log in? (Mac) | `ansible homelab -m ping` |
| Ansible: is my file valid? (Mac) | `ansible-playbook site.yml --syntax-check` |
| Ansible: what would change? (Mac) | `ansible-playbook site.yml -K --check --diff` |

---

## After fixing: write it down

When I solve something new, I add a row to [pitfalls.md](pitfalls.md):

```markdown
| 40 | **Short name of the problem** | What I saw | What fixed it, and the lesson |
```

And if it's a symptom that could come back, a section here too. Future me (and you) will thank me.
