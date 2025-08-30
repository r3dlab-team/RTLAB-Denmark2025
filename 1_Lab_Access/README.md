# How to access the LAB?
1. Go to https://dashboard.snaplabs.io/
2. Credentials will be provided during the presentation
<img width="1682" height="1257" alt="image" src="https://github.com/user-attachments/assets/49cb88a4-8925-4a90-9dfb-70476564b59d" />

3. Once logged in, please locate your assigned Range and click on Manage:

<img width="1780" height="1017" alt="image" src="https://github.com/user-attachments/assets/67a26ae3-6da6-47eb-a778-6fc1b19c256c" />

4. Find your assigned Kali Box machine (Kali_labX) and click on Connect:
> **`X` will represent your participant number (1 - 10) on the assigned lab machine on the respective range (1 - 4)** 

<img width="1387" height="1202" alt="image" src="https://github.com/user-attachments/assets/9c02b808-d142-4aaf-ac79-1709349365aa" />

5. Once connected to the Kali Box, open a new Terminal and Documents Folder:

<img width="2413" height="1798" alt="image" src="https://github.com/user-attachments/assets/3b0b22c3-2fbc-477a-bd30-49a99c0566ce" />

6. The next step is to start an RDP session to the target (compromised) W11 workstation:

Start the RDP session using the command below:  
```bash 
xfreerdp3 /u:ricksmith /p:Spring2025! /v:10.10.X.20 /cert:ignore
```    
> **IP:** `10.10.X.20`  
> **User:** `ricksmith`  
> **Password:** `Spring2025!`  

![image](https://github.com/user-attachments/assets/2595d9cd-e127-4af2-bc17-78fa9cc4b5a5)

This RDP connection allows the attacker to interact with the breached system's graphical interface as the specified user, potentially enabling further lateral movement or post-exploitation activities within the environment.
