# 🛠️ Setup Guide — SSH Honeypot Lab (Cowrie)

Detailed, step-by-step build instructions for reproducing this lab environment, including troubleshooting notes for issues encountered during the build.

---

## Prerequisites

- VirtualBox (7.x) installed on the host machine
- Ubuntu Server 22.04 LTS ISO
- Kali Linux VM (or ISO to build one)
- ~4 GB RAM and ~20 GB disk space available for the honeypot VM

---

## Part 1 — Create the Honeypot VM (Ubuntu Server)

1. In VirtualBox, create a new VM: **2048 MB RAM**, **2 CPUs**, **20 GB dynamically allocated disk**, Ubuntu 22.04 ISO attached.
2. During install, when prompted for SSH setup, check **"Install OpenSSH server"**.
3. Complete the install with a standard user account (this guide uses `akk`).

### Network Configuration

The honeypot needs two network adapters:

| Adapter | Type | Purpose |
|---|---|---|
| Adapter 1 | Host-only Adapter | Isolated lab network — where the "attack" traffic happens |
| Adapter 2 | NAT | Internet access — only needed for installing packages |

> ⚠️ **Known issue:** After every VM reboot, the NAT adapter (`enp0s8`) can come back in a `DOWN` state, breaking internet access and causing `apt` errors like `Temporary failure resolving 'archive.ubuntu.com'`.
>
> **Permanent fix** — edit the netplan config so both adapters get DHCP automatically on boot:
> ```bash
> sudo nano /etc/netplan/50-cloud-init.yaml
> ```
> Add `enp0s8` alongside the existing `enp0s3` entry:
> ```yaml
> network:
>   version: 2
>   ethernets:
>     enp0s3:
>       dhcp4: true
>     enp0s8:
>       dhcp4: true
> ```
> Then apply: `sudo netplan apply`

---

## Part 2 — Install Cowrie

Cowrie should run as a **dedicated, non-root user** — never as root, and never as your main admin user.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-venv python3-pip libssl-dev libffi-dev build-essential authbind
sudo adduser --disabled-password cowrie
sudo su - cowrie
```

Clone the repository and set up a Python virtual environment:

```bash
git clone https://github.com/cowrie/cowrie
cd cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
```

> ⚠️ **Known issue:** Recent Cowrie versions removed the old `bin/cowrie` script entirely — the project is now installed as a proper Python package instead.

Install Cowrie itself (not just its dependencies):

```bash
pip install -e '.[dev]'
```

Verify the install:

```bash
which cowrie
```

Generate the config and runtime folders:

```bash
cowrie init
```

(Optional) Edit `etc/cowrie.cfg` to set a custom hostname so the honeypot looks like a real server:

```bash
nano etc/cowrie.cfg
# change: hostname = svr04
# to:     hostname = ubuntu-web01
```

---

## Part 3 — Run Cowrie

```bash
cowrie start
cowrie status
```

Expected: `cowrie is running (PID: xxxx)`.

Verify it's listening:

```bash
ss -tlnp | grep 2222
```

Test locally before attacking from another VM:

```bash
ssh -p 2222 root@127.0.0.1
```

Any password will be accepted — this is expected honeypot behavior.

> **Note:** Cowrie does not start automatically on boot in this setup. After every VM restart, reconnect as the `cowrie` user and run `cowrie start` again:
> ```bash
> sudo su - cowrie
> cd ~/cowrie
> source cowrie-env/bin/activate
> cowrie start
> ```

---

## Part 4 — Set Up the Attacker VM (Kali)

1. Shut down the Kali VM.
2. In VirtualBox Settings → Network → Adapter 1, set **Attached to: Host-only Adapter**, using the **same** host-only network as the honeypot VM.
3. Boot Kali and confirm connectivity:
   ```bash
   ip a
   ping -c 4 <honeypot-ip>
   ```

---

## Part 5 — Run the Attack Simulation

**Reconnaissance:**
```bash
nmap 192.168.56.101
nmap -sV -p 2222 192.168.56.101
```

**Brute-force:**

Create simple username/password lists:
```bash
nano usernames.txt   # root, admin, user, test
nano passwords.txt   # 123456, password, admin, root, toor, qwerty, letmein, 12345678
```

Run Hydra:
```bash
hydra -L usernames.txt -P passwords.txt -s 2222 ssh://192.168.56.101
```

---

## Part 6 — Review the Logs

Back on the honeypot VM:

```bash
tail -50 /home/cowrie/cowrie/var/log/cowrie/cowrie.log
```

Useful analysis one-liners:

```bash
# Count total login attempts
sudo grep "login attempt" /home/cowrie/cowrie/var/log/cowrie/cowrie.log | wc -l

# Most attempted usernames
sudo grep "login attempt" /home/cowrie/cowrie/var/log/cowrie/cowrie.log | grep -oP 'attempt \[\K[^/]+' | sort | uniq -c | sort -nr

# Most attempted passwords
sudo grep "login attempt" /home/cowrie/cowrie/var/log/cowrie/cowrie.log | grep -oP '/\K[^\]]+' | sort | uniq -c | sort -nr

# Source IPs
sudo grep "login attempt" /home/cowrie/cowrie/var/log/cowrie/cowrie.log | grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c
```

---

## Troubleshooting Reference

| Symptom | Cause | Fix |
|---|---|---|
| `Temporary failure resolving 'archive.ubuntu.com'` | NAT adapter down after reboot | See Part 1 netplan fix |
| `ping: connect: Network is unreachable` | Second network interface not brought up | `sudo dhclient enp0s8` |
| `bin/cowrie: No such file or directory` | Newer Cowrie versions dropped the `bin/` script | Use `pip install -e '.[dev]'` and the `cowrie` command instead |
| `cp: cannot stat 'etc/cowrie.cfg.dist'` | Dist file no longer shipped | Use `cowrie init` instead |
| `ssh: connect to host ... port 22: Connection refused` | OpenSSH server not installed/running | `sudo apt install -y openssh-server && sudo systemctl enable --now ssh` |
| `sudo: 3 incorrect password attempts` as the `cowrie` user | The `cowrie` user has no sudo rights (by design) | Run admin commands as your main user instead |
