**Linux Privilege Escalation**

# **1\. Introduction**

This report documents the complete walkthrough of the TryHackMe room "Linux Privilege Escalation." The objective of this room is to explore and demonstrate multiple techniques attackers use to escalate privileges on a Linux system — moving from a low-privileged user to root. Each section below covers a distinct exploitation method, supported by screenshots taken during the exercise.

# **2\. Target System Information**

Before attempting privilege escalation, it is essential to enumerate the target system to understand the environment.

## **2.1 Hostname**

Running the hostname command revealed the machine name:

![alt text](<ss/Screenshot from 2026-05-03 19-06-45.png>)
*Figure 1: Hostname \= wade7363*

## **2.2 Kernel Version**

The uname \-a command was used to identify the exact kernel version. This is critical for finding kernel-level exploits.

![alt text](<ss/Screenshot from 2026-05-03 19-07-44.png>)
*Figure 2: Kernel version 3.13.0-24-generic, Ubuntu 14.04, x86\_64*

## **2.3 OS Version**

Checking /etc/issue confirmed the OS as Ubuntu 14.04 LTS:

![alt text](<ss/Screenshot from 2026-05-03 19-08-04.png>)
*Figure 3: Ubuntu 14.04 LTS*

## **2.4 Python Version**

Python 2.7.6 was found installed, which is useful for spawning TTY shells and running exploit scripts:

![alt text](<ss/Screenshot from 2026-05-03 19-08-16.png>)
*Figure 4: Python 2.7.6*

# **3\. Kernel Exploit (CVE-2015-1328)**

The kernel version 3.13.0-24-generic is notoriously vulnerable to a local privilege escalation exploit. Research on Exploit-DB identified CVE-2015-1328 as the most prominent vulnerability for this kernel version.

![alt text](<ss/Screenshot from 2026-05-03 19-11-17.png>)
*Figure 5: CVE-2015-1328 identified as the target vulnerability*

Searching Exploit-DB for "linux 3.13.0" returned two relevant results — both related to the overlayfs Local Privilege Escalation, one of which also grants access to /etc/shadow:

![alt text](<ss/Screenshot from 2026-05-03 19-14-36.png>)
*Figure 6: Exploit-DB results for Linux Kernel 3.13.0 \< 3.19 overlayfs LPE*

# **4\. Flag 1 — Kernel Exploit**

After successfully exploiting the kernel vulnerability and gaining a root shell, Flag 1 was retrieved from /home/matt/flag1.txt:

![alt text](<ss/Screenshot from 2026-05-03 19-20-25.png>)
*Figure 7: Flag 1 \= THM-28392872729920*

# **5\. Sudo Privilege Escalation**

Running sudo \-l as the user karen revealed the following passwordless sudo permissions:

![alt text](<ss/Screenshot from 2026-05-03 19-21-04.png>)
*Figure 8: Karen can run find, less, and nano as root without a password*

Using GTFOBins techniques, these binaries can be leveraged to spawn root shells. After escalating to root, Flag 2 was found at /home/ubuntu/flag2.txt:

![alt text](<ss/Screenshot from 2026-05-03 19-21-48.png>)
*Figure 9: Flag 2 \= THM-402028394*

# **6\. Password Hash Cracking (/etc/passwd & /etc/shadow)**

With root access, the /etc/passwd and /etc/shadow files were read to extract user account hashes for offline cracking.

## **6.1 /etc/passwd**

The passwd file revealed standard system accounts and the users gerryconway and user2:

![alt text](<ss/Screenshot from 2026-05-03 19-26-14.png>)
*Figure 10: /etc/passwd — users gerryconway, user2 identified*

## **6.2 /etc/shadow**

The shadow file contained password hashes for multiple users including gerryconway, user2, and karen:

![alt text](<ss/Screenshot from 2026-05-03 19-26-45.png>)
*Figure 11: /etc/shadow hashes*

## **6.3 Unshadowing and Cracking**

The passwd and shadow files were combined using unshadow and saved to hash.txt. John the Ripper with the rockyou.txt wordlist was used to crack the hashes:

![alt text](<ss/Screenshot from 2026-05-03 19-27-12.png>)
*Figure 12: Preparing hash.txt from /etc/passwd and /etc/shadow*

![alt text](<ss/Screenshot from 2026-05-03 19-27-35.png>)
*Figure 13: John the Ripper cracked the hash — Password: Password1*

# **7\. Flag 3 — SUID / Base64**

Enumerating SUID binaries led to finding /usr/bin/base64 with the setuid bit. This was exploited to read the flag3.txt file in the ubuntu home directory by encoding and decoding the file content:

![alt text](<ss/Screenshot from 2026-05-03 19-27-53.png>)
*Figure 14: Flag 3 \= THM-3847834 (read via base64 SUID)*

# **8\. Linux Capabilities**

The getcap \-r / command was used to enumerate binaries with special Linux capabilities set:

![alt text](<ss/Screenshot from 2026-05-03 19-29-31.png>)
*Figure 15: Capabilities enumeration — /home/karen/vim and /home/ubuntu/view have cap\_setuid+ep*

Binaries with cap\_setuid+ep can be used via GTFOBins to set the UID to 0 and spawn a root shell.

# **9\. Flag 4 — Capabilities Exploit**

Using the view binary (which has cap\_setuid+ep), a root shell was obtained and Flag 4 was retrieved from /home/ubuntu/flag4.txt:

![alt text](<ss/Screenshot from 2026-05-03 19-29-57.png>)
*Figure 16: Crontab showing /home/karen/backup.sh and /tmp/test.py running as root*

# **10\. Cron Job Privilege Escalation**

Examining /etc/crontab revealed several jobs running as root, including antivirus.sh, /home/karen/backup.sh, and /tmp/test.py — all of which were writable or injectable:

![alt text](<ss/Screenshot from 2026-05-03 19-30-28.png>)
*Figure 17: Crontab — scripts running as root*

A reverse shell payload was placed into the writable script. A netcat listener was set up on port 6666\. When the cron job fired, a root shell was received:

![alt text](<ss/Screenshot from 2026-05-03 19-30-47.png>)
*Figure 18: Netcat listener received root shell, Flag 5 \= THM-383000283*

# **11\. /etc/passwd Writable — Second System**

On a second target system, the /etc/shadow was read and the hash for user "matt" was extracted:

![alt text](<ss/Screenshot from 2026-05-03 19-31-18.png>)
*Figure 19: /etc/shadow on second target — matt hash highlighted*

John the Ripper successfully cracked the hash with the rockyou.txt wordlist:

![alt text](<ss/Screenshot from 2026-05-03 19-31-51.png>)
*Figure 20: Cracked hash — Password: 123456*

After gaining access as matt, Flag 6 was retrieved:

![alt text](<ss/Screenshot from 2026-05-03 19-33-54.png>)
*Figure 21: Flag 6 \= THM-736628929*

# **12\. PATH Variable Manipulation**

The $PATH variable was inspected:

![alt text](<ss/Screenshot from 2026-05-03 19-32-42.png>)
*Figure 22: $PATH variable*

PATH hijacking involves placing a malicious binary earlier in the PATH than the legitimate one. When a SUID binary calls a command without an absolute path, the malicious version runs as root.

# **13\. NFS (No Root Squash) Privilege Escalation**

Checking /etc/exports revealed NFS shares with no\_root\_squash configured on /home/backup, /tmp, and /home/ubuntu/sharedfolder:

![alt text](<ss/Screenshot from 2026-05-03 19-34-19.png>)
*Figure 23: /etc/exports — no\_root\_squash enabled*

With no\_root\_squash, files mounted from an attacker machine retain root ownership. A SUID binary was compiled on the attacker machine and placed on the NFS share. Executing it on the target yielded a root shell, and Flag 7 was retrieved:

![alt text](<ss/Screenshot from 2026-05-03 19-34-52.png>)
*Figure 24: Flag 7 \= THM-89384012 (via NFS exploit)*

# **14\. Summary of Flags**

The table below summarises all flags captured during this exercise:

| Flag | Technique | Value |
| :---- | :---- | :---- |
| **Flag 1** | Kernel Exploit (CVE-2015-1328) | THM-28392872729920 |
| **Flag 2** | Sudo Abuse (find/less/nano) | THM-402028394 |
| **Flag 3** | SUID base64 | THM-3847834 |
| **Flag 4** | Linux Capabilities (view) | THM-9349843 |
| **Flag 5** | Cron Job (writable script) | THM-383000283 |
| **Flag 6** | /etc/passwd \+ John the Ripper | THM-736628929 |
| **Flag 7** | NFS no\_root\_squash | THM-89384012 |

# **15\. Reflection**

Completing the Linux Privilege Escalation room on TryHackMe was a highly valuable learning experience that significantly deepened my understanding of how attackers move from a low-privileged foothold to full root access on a Linux system. Prior to this room, I was aware of privilege escalation as a concept, but working through each technique hands-on made the risks tangible and reinforced why thorough system hardening is so important.

## **15.1 Key Concepts Learned**

One of the most important takeaways was understanding that privilege escalation rarely relies on a single vulnerability — it is usually the result of accumulated misconfigurations. A writable cron job script, a passwordless sudo rule, or an NFS share with no\_root\_squash may each seem minor in isolation, but any one of them is sufficient for a full system compromise.

Working with kernel exploits (CVE-2015-1328) highlighted the critical importance of keeping systems patched. An unpatched kernel from 2014 running in a live environment is a significant risk, and this exercise demonstrated exactly how straightforward exploitation can be when a known CVE exists and the system has not been updated.

The sudo misconfiguration task was particularly eye-opening. Standard utilities like find, less, and nano can be weaponised for privilege escalation when granted with NOPASSWD in the sudoers file. This reinforced the principle of least privilege — users should only have the permissions they absolutely need, and nothing more.

## **15.2 Real-World Relevance**

This room demonstrated that many vulnerabilities exploited here are not theoretical — they represent real-world misconfigurations commonly found during penetration tests of Linux environments. Password hashes crackable with rockyou.txt serve as a reminder that password complexity policies matter enormously. Weak passwords such as "Password1" or "123456" being sufficient to crack hashes underscores the need for enforcing strong password policies at the organisational level.

The cron job exploitation also resonated strongly. Many organisations run automated maintenance scripts as root, and if those scripts are stored in world-writable directories, they present a trivial escalation path for any user with shell access.


# **16\. Conclusion**

This TryHackMe room provided hands-on experience with seven distinct Linux privilege escalation techniques. Each method exploited a different misconfiguration or vulnerability:

* Kernel Exploit (CVE-2015-1328): Outdated kernels expose critical LPE vulnerabilities.  
* Sudo Abuse: Passwordless sudo on binaries like find, less, and nano trivially leads to root.  
* SUID Binaries: Binaries like base64 with the SUID bit can read any file as root.  
* Linux Capabilities: cap\_setuid+ep on binaries such as vim/view enables UID manipulation.  
* Cron Jobs: Writable scripts executed by root cron jobs allow reverse shell injection.  
* Password Cracking: Readable /etc/shadow combined with weak passwords enables lateral movement.  
* NFS no\_root\_squash: Misconfigured NFS exports allow SUID payloads to run as root.

Defenders should regularly patch kernel and OS packages, audit sudo rules, remove unnecessary SUID bits and capabilities, restrict cron job script permissions, enforce strong password policies, and properly configure NFS exports with root\_squash.