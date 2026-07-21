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

![](media/image1.png){width="5.173611111111111in"
height="1.4375in"}

![](media/image2.png){width="4.006944444444445in"
height="1.8333333333333333in"}

## 4.Scanning and Enumeration

Service enumeration was performed using Nmap:

  -----------------------------------------------------------------------
  nmap -T4 -sC -sV -A \<target-IP\>

  -----------------------------------------------------------------------

The -T4 flag speeds up scan timing (acceptable in a lab environment),
while -sC, -sV, and -A enable default scripts, service version
detection, and aggressive enumeration (OS detection and traceroute)
respectively.

## ![](media/image3.png){width="3.94375in" height="3.1805555555555554in"}

## 

## 

## 

## 

## 

## 

## 

## 

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

![](media/image4.png){width="4.583333333333333in"
height="1.5375951443569553in"}

Navigating into site/ loaded the application\'s landing page, with
About, Projects, and Buscar navigation options. Reviewing the page
source and testing each link revealed a reference to busque.php with a
buscar parameter. The bare path returned a 404, but the parameter itself
--- clearly user-controlled --- was flagged for further testing.

![](media/image5.png){width="4.527777777777778in"
height="1.9588932633420821in"}

![](media/image6.png){width="4.674304461942257in"
height="1.5479166666666666in"}

## 6. Gaining a Foothold -- OS Command Injection

The buscar parameter was first tested for local file disclosure by
requesting /etc/passwd:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=/etc/passwd

  -----------------------------------------------------------------------

![](media/image7.png){width="3.5in" height="2.6882895888014in"}

The server returned the full contents of /etc/passwd, revealing local
user accounts including one named jangow01. To determine whether this
was limited to file disclosure or allowed full command execution, a
shell command was appended using a semicolon separator, with spaces
URL-encoded as %20:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=;ls -la

  -----------------------------------------------------------------------

![](media/image8.png){width="5.029331802274716in"
height="1.5479166666666666in"}

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

![](media/image9.png){width="5.3125in"
height="1.383277559055118in"}

This revealed a hidden file, .backup, which was read with:

  -----------------------------------------------------------------------
  http://\<target-IP\>/site/busque.php?buscar=;cd%20..;cat%20.backup

  -----------------------------------------------------------------------

![](media/image10.png){width="5.486111111111111in"
height="1.9018438320209974in"}

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

![](media/image11.png){width="2.270832239720035in"
height="1.3125in"}

The user flag was retrieved from the home directory:

+-----------------------------------------------------------------------+
| ftp\> cd /home                                                        |
|                                                                       |
| ftp\>cd jangow01                                                      |
|                                                                       |
| ftp\> get user.txt                                                    |
+-----------------------------------------------------------------------+

![](media/image12.png){width="4.2363287401574805in"
height="1.7987040682414699in"}

![](media/image13.png){width="3.5694444444444446in"
height="0.3888888888888889in"}

With filesystem access via FTP, enumeration continued under
/var/www/html/site, revealing a wordpress/ directory containing
config.php. This file disclosed a second set of database credentials ---
a different username and database, but the same reused password,
abygurl69 --- reinforcing the scope of the credential reuse issue across
services.

![](media/image14.png){width="4.236111111111111in"
height="1.2361111111111112in"}![](media/image14.png){width="4.236111111111111in"
height="1.2361111111111112in"}![](media/image15.png){width="3.9238123359580053in"
height="1.5278565179352581in"}

## 8. Privilege Escalation

No SSH service was identified during the Nmap scan, so the recovered
credentials (jangow01 / abygurl69) were used to authenticate directly
through the VM\'s local console. The second credential pair (desafio02 /
abygurl69) recovered from config.php was also tested but was not valid
for a system login; only jangow01 provided an interactive shell.

![](media/image16.png){width="4.548845144356956in"
height="2.5695767716535434in"}

The kernel version was then enumerated:

  -----------------------------------------------------------------------
  uname -a

  -----------------------------------------------------------------------

![](media/image17.png){width="5.5697309711286085in"
height="0.4444674103237095in"}

The target was running Linux kernel 4.4.0-31, an outdated Ubuntu kernel
affected by several known local privilege escalation vulnerabilities.
Research on Exploit-DB identified CVE-2017-16995 (EDB-ID: 45010) --- a
Linux kernel BPF verifier sign-extension vulnerability enabling local
privilege escalation --- as a confirmed match, as the public exploit is
explicitly documented as tested against Ubuntu 16.04 running kernel
4.4.0-31-generic, an exact match for this target.

![](media/image18.png){width="3.5458333333333334in"
height="1.5305555555555554in"}

The exploit source code was transferred to the target over the existing
FTP session into jangow01\'s home directory, then compiled and executed:

+-----------------------------------------------------------------------+
| gcc jangow01.c -o jangow01                                            |
|                                                                       |
| chmod +x jangow01                                                     |
|                                                                       |
| ./jangow01                                                            |
+-----------------------------------------------------------------------+

![](media/image19.png){width="4.479397419072616in"
height="1.027830271216098in"}

![](media/image20.png){width="5.152777777777778in"
height="2.4444444444444446in"}![](media/image21.png){width="1.465353237095363in"
height="0.4375229658792651in"}

## 9. Root Flag & Full Compromise 

Privilege escalation was verified with:

  -----------------------------------------------------------------------
  whoami

  -----------------------------------------------------------------------

The output returned root, confirming successful privilege escalation.
The /root directory was accessed and proof.txt was read, completing full
compromise of the target machine.

![](media/image22.png){width="4.25in"
height="3.486878827646544in"}

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
