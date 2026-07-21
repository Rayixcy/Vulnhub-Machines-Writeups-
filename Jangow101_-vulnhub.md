**Jangow: 1.0.1**

Prepared by: Raya

Target: Jangow 1.0.1 (VulnHub)

Environment: VirtualBox --- Kali Linux (Attacker) / Jangow 1.0.1
(Target)

# **Table of Contents**

1\. Executive Summary

2\. Scope & Objectives

3\. Reconnaissance

4\. Scanning & Enumeration

5\. Web Enumeration

6\. Gaining a Foothold --- OS Command Injection

7\. FTP Access & User Flag

8\. Privilege Escalation

9\. Root Flag & Full Compromise

10\. Findings Summary

11\. Recommendations

12\. Conclusion

## 1. Executive Summary

Jangow: 1.0.1 is a beginner-friendly Boot2Root machine hosted on VulnHub
that demonstrates how several low-severity issues, when chained
together, can lead to full system compromise. This assessment identified
an unauthenticated OS command injection vulnerability in a custom PHP
web application, which was used to disclose hardcoded credentials. Due
to credential reuse across the web application, an FTP service, and the
operating system itself, these credentials provided an initial foothold.
A subsequent local kernel exploit (CVE-2017-16995) was used to escalate
privileges from a standard user to root, resulting in complete
compromise

## 2.Scope and Objectives 

The objective of this exercise was to perform a full compromise of the
Jangow 1.0.1 VulnHub machine, simulating a black-box external
penetration test. The scope was limited strictly to the target virtual
machine, running in an isolated lab network with no internet-facing
exposure.

Goals:

-   identify exposed services

-   enumerate the web application,

-   obtain an initial low-privilege foothold,

-   escalate to root

-   document the full attack chain along with remediation guidance.

## 3.Reconnaissance

Before scanning, the attacker machine\'s network configuration was
verified using ifconfig to confirm the correct subnet, and connectivity
to the target was confirmed with a simple ping.

<img width="776" height="215" alt="image" src="https://github.com/user-attachments/assets/359b23a0-c6a6-4975-9f08-90d550bf3378" />


<img width="601" height="275" alt="image" src="https://github.com/user-attachments/assets/d4927786-b340-4a1c-926b-51cd5d59bc1d" />


## 4.Scanning and Enumeration

Service enumeration was performed using Nmap:

  -----------------------------------------------------------------------
  nmap -T4 -sC -sV -A \<target-IP\>

  -----------------------------------------------------------------------

The -T4 flag speeds up scan timing (acceptable in a lab environment),
while -sC, -sV, and -A enable default scripts, service version
detection, and aggressive enumeration (OS detection and traceroute)
respectively.

<img width="591" height="477" alt="image" src="https://github.com/user-attachments/assets/d214cc91-38cf-428c-879f-03f3c3b814bd" />


## Open services identified:

-   **FTP (21)** --- File Transfer Protocol

-   **HTTP (80)** --- Web server hosting a custom PHP application

The web application presented the largest attack surface and was
prioritized for further testing.

## 5. Web Enumeration

Browsing to the target\'s IP address returned an Apache directory
listing rather than a homepage, indicating directory indexing was
enabled with no default index page configured. The listing contained a
single directory, site/. The page footer confirmed the server version as
Apache/2.4.18 (Ubuntu), consistent with the Nmap results.

<img width="687" height="230" alt="image" src="https://github.com/user-attachments/assets/a02da2a9-d7a7-4319-bbd2-4ba3d6d7005e" />


Navigating into site/ loaded the application\'s landing page, with
About, Projects, and Buscar navigation options. Reviewing the page
source and testing each link revealed a reference to busque.php with a
buscar parameter. The bare path returned a 404, but the parameter itself
--- clearly user-controlled --- was flagged for further testing.

<img width="679" height="293" alt="image" src="https://github.com/user-attachments/assets/14e6c4e9-e7d6-499c-a5c4-18fac993954e" />


<img width="701" height="232" alt="image" src="https://github.com/user-attachments/assets/58ee30a3-66aa-406d-ac33-adfcd42785dd" />

## 6. Gaining a Foothold -- OS Command Injection

The buscar parameter was first tested for local file disclosure by
requesting /etc/passwd:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=/etc/passwd

  -----------------------------------------------------------------------

<img width="525" height="403" alt="image" src="https://github.com/user-attachments/assets/b67195ad-56da-4d4c-949a-68ad4b6e9446" />


The server returned the full contents of /etc/passwd, revealing local
user accounts including one named jangow01. To determine whether this
was limited to file disclosure or allowed full command execution, a
shell command was appended using a semicolon separator, with spaces
URL-encoded as %20:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=;ls -la

  -----------------------------------------------------------------------

<img width="754" height="233" alt="image" src="https://github.com/user-attachments/assets/caaa916a-8efa-4055-99dc-12f646928baa" />


The server executed the injected command and returned a directory
listing, confirming that busque.php passed the buscar value directly
into a shell execution function (such as PHP\'s system() or
shell_exec()) without sanitization --- a classic OS command injection
vulnerability.

Commands were chained further to move up a directory and list hidden
files:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=;cd%20..;ls%20-la

  -----------------------------------------------------------------------

<img width="797" height="207" alt="image" src="https://github.com/user-attachments/assets/dc8833a8-84c2-4f6e-9c81-6de0c3b2485a" />


This revealed a hidden file, .backup, which was read with:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=;cd%20..;cat%20.backup

  -----------------------------------------------------------------------

<img width="823" height="285" alt="image" src="https://github.com/user-attachments/assets/3339dda1-eeff-4dc0-b962-ac6a4bd808e0" />


The file contained a PHP database connection script with hardcoded
credentials: username jangow01, password abygurl69. Since jangow01 was
already confirmed as a valid Linux user, and credential reuse across
services is a common weakness, this credential pair was tested against
the FTP service identified earlier.

## 7. FTP Access and User Flag

  -----------------------------------------------------------------------
  ftp \<target-IP\>

  -----------------------------------------------------------------------

Authentication succeeded using jangow01 / abygurl69, confirming
credential reuse between the web application\'s database configuration
and the system\'s FTP account.

<img width="341" height="197" alt="image" src="https://github.com/user-attachments/assets/c2679af2-1e1d-4457-8796-3a788d19f354" />


The user flag was retrieved from the home directory:

-----------------------------------------------------------------------
ftp\> cd /home     
ftp\>cd jangow01  
ftp\> get user.txt                

-----------------------------------------------------------------------

<img width="636" height="270" alt="image" src="https://github.com/user-attachments/assets/438b0c47-0317-40c4-8138-77a60d24402e" />

<img width="536" height="59" alt="image" src="https://github.com/user-attachments/assets/93664e81-a8cc-4c60-99db-a9618a81bff0" />


With filesystem access via FTP, enumeration continued under
/var/www/html/site, revealing a wordpress/ directory containing
config.php. This file disclosed a second set of database credentials ---
a different username and database, but the same reused password,
abygurl69 --- reinforcing the scope of the credential reuse issue across
services.
<img width="588" height="229" alt="image" src="https://github.com/user-attachments/assets/97a763a6-30ea-44f0-82a6-e1712eb16167" />
<img width="635" height="185" alt="image" src="https://github.com/user-attachments/assets/bada26bc-f9b0-41a6-b500-87b831c1553b" />
<img width="636" height="186" alt="image" src="https://github.com/user-attachments/assets/2e0a34d4-b1e3-45b4-b6e9-706c82062b0a" />


## 8. Privilege Escalation

No SSH service was identified during the Nmap scan, so the recovered
credentials (jangow01 / abygurl69) were used to authenticate directly
through the VM\'s local console. The second credential pair (desafio02 /
abygurl69) recovered from config.php was also tested but was not valid
for a system login; only jangow01 provided an interactive shell.

<img width="683" height="385" alt="image" src="https://github.com/user-attachments/assets/8906aedb-c31b-4f58-b416-a5c2dbab6317" />


The kernel version was then enumerated:

  -----------------------------------------------------------------------
  uname -a

  -----------------------------------------------------------------------
<img width="835" height="66" alt="image" src="https://github.com/user-attachments/assets/80ee2cac-2abf-4ba3-9bc6-29200c051651" />


The target was running Linux kernel 4.4.0-31, an outdated Ubuntu kernel
affected by several known local privilege escalation vulnerabilities.
Research on Exploit-DB identified CVE-2017-16995 (EDB-ID: 45010) --- a
Linux kernel BPF verifier sign-extension vulnerability enabling local
privilege escalation --- as a confirmed match, as the public exploit is
explicitly documented as tested against Ubuntu 16.04 running kernel
4.4.0-31-generic, an exact match for this target.

<img width="532" height="230" alt="image" src="https://github.com/user-attachments/assets/93d0ed63-7651-4410-aa00-4ede9bdb905f" />


The exploit source code was transferred to the target over the existing
FTP session into jangow01\'s home directory, then compiled and executed:

-----------------------------------------------------------------------
 gcc jangow01.c -o jangow01                                            
 chmod +x jangow01                                                    
./jangow01

-----------------------------------------------------------------------

<img width="671" height="154" alt="image" src="https://github.com/user-attachments/assets/cef542a2-5cbf-48e7-8fea-c8efcfcef63d" />
<img width="219" height="66" alt="image" src="https://github.com/user-attachments/assets/ab7530a4-86ed-4f19-ae67-43b79e2474f3" />
<img width="773" height="367" alt="image" src="https://github.com/user-attachments/assets/16db7d7d-073d-49a8-8376-d1554d2d0054" />

## 9. Root Flag & Full Compromise 

Privilege escalation was verified with:

  -----------------------------------------------------------------------
  whoami

  -----------------------------------------------------------------------

The output returned root, confirming successful privilege escalation.
The /root directory was accessed and proof.txt was read, completing full
compromise of the target machine.

<img width="490" height="402" alt="image" src="https://github.com/user-attachments/assets/2eac9ae4-430e-4b29-be92-0fe914dc3853" />


## 10. Findings Summary

  -------------------------------------------------------------------------------------
  **\#**   **Finding**        **Severity**   **Description**
  -------- ------------------ -------------- ------------------------------------------
  1        Directory Listing  Low            Apache directory indexing exposed the
           Enabled                           site/ directory structure with no index
                                             page.

  2        OS Command         Critical       The buscar parameter was passed
           Injection                         unsanitized into a shell execution
           (busque.php)                      function, allowing arbitrary command
                                             execution.

  3        Hardcoded & Reused High           Database credentials were hardcoded in
           Credentials                       .backup and wordpress/config.php, and the
                                             same password was reused for the
                                             FTP/system account jangow01.

  4        Outdated Kernel    Critical       Kernel 4.4.0-31 was vulnerable to a known,
           (CVE-2017-16995)                  publicly available local privilege
                                             escalation exploit.
                                             
  -------------------------------------------------------------------------------------

## 11. Recommendations 

-   **Disable directory listing** on the Apache web server and ensure a
    proper index page is configured for all directories.

-   **Eliminate OS command injection** by never passing user input
    directly to shell execution functions. Use parameterized APIs,
    allow-list validation, and language-native functions instead of
    shell calls.

-   **Remove hardcoded credentials** from application source and
    configuration files. Use environment variables or a secrets manager
    instead.

-   **Eliminate password reuse** across services and accounts; enforce
    unique, strong credentials per system and service.

-   **Patch and update the kernel** regularly, and track CVEs against
    the running kernel version to close known local privilege escalation
    vectors.

-   **Apply the principle of least privilege** to service accounts so
    that a single compromised credential does not grant broad system
    access.

## 12. Conclusion

Jangow: 1.0.1 is a beginner-friendly machine that demonstrates how
seemingly minor security issues can be chained together to achieve full
system compromise. Starting with web enumeration, the attack progressed
through OS command injection, credential disclosure, credential reuse,
FTP access, and ultimately a kernel-based privilege escalation to obtain
root.

This exercise reinforces several core penetration testing principles:
the importance of thorough enumeration, rigorous validation of
user-controlled input, checking for credential reuse across services,
and always assessing the underlying operating system for known privilege
escalation vulnerabilities. Individually, each issue identified here
appears low risk. Chained together, they formed a complete attack path
from an unauthenticated web request to full administrative control.
