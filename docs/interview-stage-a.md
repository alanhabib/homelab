# Mock interview · Stage A (Linux, SSH, networking, Ansible)

Five questions an interviewer could ask about what I built in Phases 0–4, with strong model answers. Every command in an answer is explained, so I can say *what* I typed and *why*.

[← Back to README](../README.md) · Built in: [Phase 2](02-network-and-ssh.md) · [Phase 4](04-ansible.md)

**How I use this page:** read the question, answer out loud first (no peeking), then compare with the model answer. Aim for about 60 seconds per answer.

**Interview tips**

- **Lead with the why**, then the how, then the proof ("and I verified it by...").
- **Use precise words**: "key-based authentication", not "SSH instead of a password".
- **Admit trade-offs**: "it's one node, so it's not highly available" is a strong sentence, not a weak one.
- **For "tell me about a problem" questions, use STAR**: **S**ituation, **T**ask, **A**ction, **R**esult.

---

## Question 1 · SSH keys

> "You switched off password login on your server. Why, how did you do it, and how did you make sure you didn't lock yourself out?"

### Model answer

> "Password login over SSH is a risk: anything reachable can be brute-forced, and passwords get reused. So I switched to **key-based authentication**.
>
> I made an **ed25519 key pair** on my Mac. The **private key stays on the Mac**, protected by a passphrase. Only the **public key** goes to the server, into `~/.ssh/authorized_keys`, which I did with `ssh-copy-id`.
>
> Then I switched passwords off with **Ansible**: a drop-in file in `sshd_config.d` with `PasswordAuthentication no` and `PermitRootLogin no`, plus a **handler** that restarts SSH only when that file changes.
>
> To avoid locking myself out, I kept a second SSH session open, tested a brand-new connection before closing anything, and verified that password logins now get `Permission denied (publickey)`. Worst case, I still have the physical console."

### The commands behind the answer

**1. Create the key pair (on the Mac)**

```bash
ssh-keygen -t ed25519 -C "alan-macbook"
```

| Part | Means |
| --- | --- |
| `ssh-keygen` | the tool that creates SSH keys |
| `-t ed25519` | the key type: modern, short and secure, today's recommended choice |
| `-C "alan-macbook"` | a label inside the public key, so anywhere it's installed I can see it came from my MacBook |

Result: `~/.ssh/id_ed25519` (private, **never shared**) and `~/.ssh/id_ed25519.pub` (public, safe to share). It asks for a **passphrase**, which encrypts the private key, so a stolen laptop alone isn't enough.

**2. Copy the public key to the server (on the Mac)**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub alandalo@192.168.1.193
```

| Part | Means |
| --- | --- |
| `ssh-copy-id` | logs in once with the password and appends the public key to the server's `~/.ssh/authorized_keys`, the list of trusted keys |
| `-i ~/.ssh/id_ed25519.pub` | **which** public key to copy (`-i` = identity). Being explicit matters when you have several keys |
| `alandalo@192.168.1.193` | user `alandalo` on the server at that address. (When I did it, the server was still on the mesh network at `192.168.39.137`; it moved to `192.168.1.193` later) |

**3. A shortcut in `~/.ssh/config` (on the Mac)**

```
Host homelab
  HostName 192.168.1.193
  User alandalo
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes
  UseKeychain yes
```

Now `ssh homelab` fills in address, user and key. `UseKeychain` stores the passphrase in the macOS Keychain.

**4. Switch passwords off (Ansible task)**

```yaml
    - name: Harden SSH (keys only, no root login)
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/10-hardening.conf
        content: |
          PasswordAuthentication no
          KbdInteractiveAuthentication no
          PermitRootLogin no
        owner: root
        group: root
        mode: "0644"
      notify: Restart ssh
```

- `PasswordAuthentication no` -> no passwords over SSH
- `KbdInteractiveAuthentication no` -> also closes the "question and answer" style of password login
- `PermitRootLogin no` -> nobody logs in directly as root; I log in as myself and use sudo
- `notify: Restart ssh` -> rings the handler, which restarts SSH **only if the file changed**

**5. Prove it (on the Mac)**

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password homelab
# Permission denied (publickey)
```

`-o` sets an option for this one command: "pretend I have no key, use a password". The server refuses: it only accepts keys now.

### Follow-up questions to expect

- *"What if you lose the key?"* -> The console still takes my password; I'd temporarily move the hardening file away, add a new key, and re-run the playbook to lock it again. Better: a backup of the key and a second key from another device.
- *"Why ed25519 and not RSA?"* -> Shorter keys, faster, and the current recommendation. RSA is older and needs much longer keys for the same security.

---

## Question 2 · IP ranges and the firewall

> "Your firewall allows SSH only from `192.168.1.0/24`. What does `/24` mean? You also allowed `10.42.0.0/16` and `10.43.0.0/16`: what are those, and what happens if you forget them?"

### Model answer

> "An IP address is four numbers, each from 0 to 255. The `/24` is **CIDR notation**: it says how many bits at the start are fixed. Each number is 8 bits, so `/24` fixes the first three numbers. `192.168.1.0/24` means `192.168.1.0` to `192.168.1.255`, 256 addresses: my whole home network. `/16` fixes two numbers, so 65,536 addresses.
>
> `10.42.0.0/16` and `10.43.0.0/16` are **K3s's internal networks**: every **pod** gets an address in `10.42`, every **service** in `10.43`. Pods talk to each other and to the Kubernetes API all the time. If I forgot those rules, the firewall's default deny would block that internal traffic, and the cluster would break in confusing ways: DNS failing, pods unable to reach services.
>
> The default policy is **deny incoming, allow outgoing**. My router has no port forwards either, so the firewall is a **second layer**: defence in depth."

### The commands behind the answer

**1. Check where you come from, before writing the rule (on the server)**

```bash
echo $SSH_CLIENT
# 192.168.1.103 58769 22
```

`$SSH_CLIENT` is set by SSH at login: **the address the server sees me coming from**, my side's port, and the server's port (22). Mine was `192.168.1.103`, the mesh unit's address on the home network, so a `/24` rule covers me. Check this **before** enabling a firewall, or you can lock yourself out.

**2. The rules (Ansible, `community.general.ufw`)**

```yaml
    - name: Firewall - allow SSH, Kubernetes API and web from the home network
      community.general.ufw:
        rule: allow
        from_ip: 192.168.1.0/24
        port: "{{ item }}"
        proto: tcp
      loop: ["22", "6443", "80", "443"]
```

- `from_ip` -> only this address range may knock
- `port` -> which door: 22 SSH, 6443 the Kubernetes API, 80/443 web
- `proto: tcp` -> the kind of traffic these programs use
- `loop` -> one task, run once per port; `{{ item }}` is the current port

The by-hand equivalent of one round:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

**3. Default deny, then switch on (always last)**

```bash
sudo ufw default deny incoming
sudo ufw enable
```

**4. Verify (on the server)**

```bash
sudo ufw status verbose
```

Shows `Status: active`, `Default: deny (incoming), allow (outgoing)`, and the rules.

### Follow-up questions to expect

- *"Why isn't IPv6 in your rules?"* -> My rules name IPv4 ranges, so IPv6 visitors fall under the default deny. I connect over IPv4.
- *"How is this done in Azure?"* -> **Network Security Groups**, with the same kind of rules on subnets written in CIDR, for example `10.0.1.0/24`.

---

## Question 3 · Ansible and idempotence

> "Why did you use Ansible instead of a shell script? What does idempotent mean, and how do you test a change before running it?"

### Model answer

> "A shell script describes **steps**; Ansible describes the **desired state**. Each Ansible task checks first: if the server already matches, it reports `ok` and changes nothing; if not, it fixes it and reports `changed`. That's **idempotence**: running it once or a hundred times gives the same result. So I can re-run my playbook any time to bring the server back to the state in Git, and a rebuild is reinstall Ubuntu plus one command.
>
> It's also **agentless**: it only needs SSH and Python on the server.
>
> Before every real run I do a **dry run** with `--check --diff`: it shows what would change, line by line, without changing anything. Then the real run, then I verify on the server, then I run it again and expect `changed=0`. I also learned a dry run isn't a full test: a typo in a group name passed the dry run and would have failed for real."

### The commands behind the answer

**1. Can Ansible reach the server? (on the Mac)**

```bash
ansible homelab -m ping
```

`-m ping` runs the ping **module**: it logs in over SSH and checks Python works. The answer `"ping": "pong"` = ready.

**2. Dry run (on the Mac, in `ansible/`)**

```bash
ansible-playbook site.yml --ask-become-pass --check --diff
```

| Part | Means |
| --- | --- |
| `ansible-playbook site.yml` | run my playbook |
| `--ask-become-pass` (or `-K`) | ask once for my sudo password; the playbook uses `become: true` |
| `--check` | pretend: change nothing |
| `--diff` | show the exact lines that would change in files |

**3. Real run, then the idempotence proof**

```bash
ansible-playbook site.yml -K
```

Run it twice. The recap of the second run:

```
homelab : ok=12  changed=0  unreachable=0  failed=0 ...
```

`changed=0` = the server already matches the recipe.

**4. Handlers (from my playbook)**

```yaml
  handlers:
    - name: Restart ssh
      ansible.builtin.service:
        name: ssh
        state: restarted
```

A handler only runs when a task that `notify`s it reports `changed`, and once at the end of the play. So SSH restarts only when its config changed.

### Follow-up questions to expect

- *"What was the hardest Ansible problem?"* -> Ubuntu 26.04's new `sudo-rs` words its password prompt differently, so Ansible timed out waiting for it. I fixed it with `become_exe = sudo.ws` in `ansible.cfg`, using the classic sudo that's still installed.
- *"Ansible vs Terraform?"* -> Ansible configures machines that exist; Terraform/OpenTofu creates infrastructure (networks, VMs, DNS). They're often used together.

---

## Question 4 · A problem you troubleshot

> "Tell me about a technical problem you had to troubleshoot. How did you find the cause?"

### Model answer (STAR)

> "**Situation:** after every reboot, my server refused logins for over two minutes with 'System is booting up, unprivileged users are not permitted to log in yet'.
>
> **Task:** find what was slowing the boot, and fix it permanently.
>
> **Action:** I measured instead of guessing. `systemd-analyze blame` showed `systemd-networkd-wait-online` taking exactly two minutes, its timeout. `networkctl` showed the Wi-Fi was ready but the Ethernet port was stuck in `no-carrier`, `configuring`: no cable plugged in. Then `systemctl cat` on the service showed that **netplan** told it to wait for **both** interfaces. So the root cause was the installer's netplan config listing the Ethernet port without marking it optional. I added a netplan file through Ansible that marks the port `optional: true`, and ran `netplan generate` instead of `netplan apply`, so my SSH session over Wi-Fi wasn't disrupted.
>
> **Result:** the boot wait went from two minutes to a few seconds, and the fix is in Git, so a rebuilt server gets it too."

### The commands behind the answer

**1. Which part of startup is slow? (on the server)**

```bash
systemd-analyze blame | head -n 10
# 2min 67ms systemd-networkd-wait-online.service
```

| Part | Means |
| --- | --- |
| `systemd-analyze blame` | list every startup step with how long it took, slowest first |
| `\|` | send the output into the next command |
| `head -n 10` | show only the first 10 lines |

**2. What is the network waiting for?**

```bash
networkctl
# enp0s31f6  ether  no-carrier  configuring
# wlp61s0    wlan   routable    configured
```

`no-carrier` = no cable. `configuring` = never finishes. `routable` = fully connected.

**3. Who told it to wait?**

```bash
systemctl cat systemd-networkd-wait-online.service
```

Shows the service file **and** its drop-ins. Netplan's drop-in had `-i wlp61s0:degraded -i enp0s31f6:degraded` = "wait for both".

**4. The fix (Ansible task)**

```yaml
    - name: Make ethernet port optional, so boot doesn't wait for cable
      ansible.builtin.copy:
        dest: /etc/netplan/01-ethernet.yaml
        content: |
          network:
            version: 2
            ethernets:
              enp0s31f6:
                dhcp4: true
                optional: true
        owner: root
        group: root
        mode: "0600"
      notify: Regenerate netplan
```

Netplan merges all files in `/etc/netplan/` in number order, so `01-` adds `optional: true` to the installer's `00-` file without editing it. The handler runs `netplan generate`: it validates the config and prepares the next boot without touching the live connection.

### Follow-up questions to expect

- *"Why not just disable the wait-online service?"* -> Other services may rely on "network is online" before starting. Fixing the cause keeps that guarantee; disabling it hides the symptom.
- *"How did you verify?"* -> Rebooted, logged in within seconds, and `systemd-analyze blame` no longer listed the waiter near the top.

---

## Question 5 · Preparing Linux for Kubernetes

> "Before installing Kubernetes you changed some Linux kernel settings. Which ones, and why?"

### Model answer

> "I raised the **inotify** limits: `fs.inotify.max_user_instances` from 128 to 512, and `fs.inotify.max_user_watches` from about 57,000 to 524,288.
>
> inotify is how programs get notified when a file changes, instead of polling. On a Kubernetes node, lots of components use it: the kubelet, the container runtime, log collectors watching every container's log file, apps reloading ConfigMaps, and `kubectl logs -f`. The limits are **per user**, and many containers run as the same user, so they share one budget. When it runs out, you get misleading 'too many open files' errors.
>
> 512 and 524,288 are the values the Kubernetes **kind** project recommends. It's a **limit, not a reservation**, so it costs no memory until watches are actually used. I set it with Ansible **before** installing K3s, so the cluster never runs with the low limits, and every future node gets the same setting."

### The commands behind the answer

**1. Look at the current values (on the server)**

```bash
sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches
cat /proc/sys/fs/inotify/max_user_instances
```

`sysctl` reads kernel settings. `/proc/sys/...` shows the same settings as files: `/proc` is a window into the running kernel.

**2. By hand: temporary, then permanent**

```bash
sudo sysctl -w fs.inotify.max_user_instances=512        # now, lost at reboot
echo 'fs.inotify.max_user_instances=512' | sudo tee /etc/sysctl.d/90-kubernetes.conf
sudo sysctl --system                                     # reload all settings files
```

- `-w` = write a new value
- `/etc/sysctl.d/` = files read at every boot; `90-` makes mine read late, so it wins
- `tee` writes what it receives into a file (with `sudo`, because `/etc` needs root)
- `--system` = reload all the files now, as at boot

**3. With Ansible (what I actually use)**

```yaml
    - name: Raise file-watch limits for Kubernetes
      ansible.posix.sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        sysctl_file: /etc/sysctl.d/90-kubernetes.conf
        reload: true
      loop:
        - { name: fs.inotify.max_user_instances, value: "512" }
        - { name: fs.inotify.max_user_watches, value: "524288" }
```

`reload: true` applies it now; `sysctl_file` makes it permanent. Each loop item has two parts, used as `item.name` and `item.value`.

### Follow-up questions to expect

- *"What if a second node joins without this?"* -> Kernel settings are per machine. Pods on that node would get the errors, which is confusing because it only happens there. That's why every node goes through the same playbook.
- *"How is this done on AKS?"* -> Microsoft sets sensible values, and you can change them per node pool with **custom node configuration** (`sysctls`).

---

## My scorecard

| Question | First try | Notes for next time |
| --- | --- | --- |
| 1 · SSH keys | Good instincts, details vague | Say *key-based authentication*; private key never leaves the Mac; mention the passphrase, Ansible and the verification |
| 2 · IP ranges | Right idea, fuzzy details | `/24` = first three numbers fixed (CIDR, 24 bits); `10.42` pods **and** `10.43` services; don't mix up node and pod; it's the firewall's default deny that would block them; give the CoreDNS example |
| 3 · Ansible | Solid core, great reuse point | Answer "why not a script" directly (steps vs end state); explain `ok`/`changed`; full test routine: `--check --diff`, real run, verify, second run `changed=0` |
| 4 · Troubleshooting | Good story, real command | Use STAR and say "I"; start with the symptom; name `systemd-networkd-wait-online`; three tools (`blame`, `networkctl`, `systemctl cat`); end with the result in numbers |
| 5 · Kernel settings | Didn't know | Learn it: inotify limits 128 -> 512 and ~57k -> 524,288, why Kubernetes needs them, set before K3s |
