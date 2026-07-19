# 🍯 SSH Honeypot Lab — Catching Attackers with Cowrie

A hands-on cybersecurity project simulating a vulnerable Linux server, deployed to capture, log, and analyze real attacker behavior using the **Cowrie SSH/Telnet honeypot**.

---

## 📌 Objective

Modern servers are constantly probed by automated bots scanning for weak credentials. This project builds a controlled, isolated lab environment to:

- Deploy a realistic-looking SSH honeypot
- Simulate real-world attack techniques (reconnaissance + brute-force)
- Capture and analyze attacker behavior from logs
- Practice core skills relevant to **network forensics, log analysis, and blue-team defense**

---

## 🏗️ Architecture

```
┌─────────────────────┐         Host-Only Network        ┌──────────────────────────┐
│   Kali Linux VM      │  ───────────────────────────►    │  Ubuntu Server 22.04 VM   │
│   (Attacker)          │      192.168.56.0/24             │  (Honeypot Target)         │
│   192.168.56.102      │                                   │  192.168.56.101            │
│                        │  ◄───────────────────────────    │                             │
│  - nmap                │        Attack Traffic            │  - Cowrie (port 2222)      │
│  - hydra                │                                  │  - OpenSSH (port 22, mgmt) │
└─────────────────────┘                                   └──────────────────────────┘
```

Both VMs run in **VirtualBox**, isolated on a **Host-Only Network** (no internet exposure), with a secondary **NAT adapter** on the honeypot VM used only for package installation.

---

## 🛠️ Tools & Technologies

| Category | Tool |
|---|---|
| Virtualization | VirtualBox |
| Honeypot Target OS | Ubuntu Server 22.04 LTS |
| Attacker OS | Kali Linux |
| Honeypot Software | [Cowrie](https://github.com/cowrie/cowrie) (SSH/Telnet honeypot) |
| Reconnaissance | Nmap |
| Brute-force Simulation | Hydra |
| Log Analysis | grep, sort, uniq (Bash) |

---

## ⚙️ Setup Overview

1. Created two isolated VMs in VirtualBox: an Ubuntu Server (target) and Kali Linux (attacker), connected via a Host-Only Network.
2. Installed Cowrie inside a Python virtual environment on the Ubuntu VM, configured to listen on port 2222 and impersonate a real OpenSSH server.
3. Verified the honeypot manually via local SSH connection before exposing it to the simulated attacker.
4. From Kali, performed reconnaissance (`nmap`) and a credential brute-force attack (`hydra`) against the honeypot.
5. Reviewed Cowrie's logs to confirm full capture of the attack lifecycle.

> Full step-by-step build guide (including troubleshooting notes) is available in [`SETUP.md`](./SETUP.md).

---

## 🎯 Attack Simulation

**Phase 1 — Reconnaissance**

```
nmap -sV -p 2222 192.168.56.101
```
Result: Cowrie successfully impersonated a real service, returning a fake banner (`OpenSSH 9.2p1 Debian`) — the attacker had no way to distinguish it from a genuine server.

**Phase 2 — Brute-Force Attack**

```
hydra -L usernames.txt -P passwords.txt -s 2222 ssh://192.168.56.101
```
32 username/password combinations were attempted.

---

## 📊 Key Findings

| Metric | Result |
|---|---|
| Total login attempts | 32 |
| Unique attacker IPs | 1 |
| Usernames targeted | `root`, `admin`, `user`, `test` (8 attempts each) |
| Passwords targeted | `toor`, `root`, `qwerty`, `password`, `letmein`, `admin`, `12345678`, `123456` (4 attempts each) |
| Attack duration | ~0.4 seconds |
| Attack signature | Automated/scripted — the speed (32 attempts in under half a second) is far beyond human capability, a clear indicator of tool-based brute-forcing rather than manual login attempts |

Every attempt — including timestamp, session ID, source IP, and credentials used — was fully captured in Cowrie's structured logs, demonstrating the honeypot's core value: turning attacker noise into forensic evidence.

---

## 📷 Evidence / Screenshots

<img width="1920" height="922" alt="Nmap Scan Result" src="https://github.com/user-attachments/assets/46559479-c438-42c9-a1d2-5e3c381e9a7f" />
<img width="1920" height="922" alt="Hydra Output" src="https://github.com/user-attachments/assets/e820f7b6-3f40-44ec-a2ab-302d5d39863a" />
<img width="1917" height="997" alt="image" src="https://github.com/user-attachments/assets/29b5c3d1-f39e-47d1-8572-c28491474db1" />


---

## 💡 Lessons Learned

- How honeypots use deception (fake banners, fake filesystems) to lure and log attackers without risking a real system
- How to isolate a lab environment safely using VirtualBox networking (Host-Only vs NAT)
- How to read and parse Cowrie's JSON/text logs to extract actionable data
- How automated attacks differ from manual ones at the log level (timing patterns)

---

## Development Notes

---

## 👤 Author

Akk — Cybersecurity and Forensics student (CSF), University of Computer Studies, Yangon (UCSY)
