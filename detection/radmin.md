# Trojanized "Legit" Remote Admin Tool and Reverse Shell Callbacks (Windows) 

## What this attack is

This is basically **malware delivery + persistence-by-user-execution**:

* A trusted-looking EXE (remote admin tool) is modified to include a hidden payload.
* The payload is a **reverse TCP connection** back to the attacker (C2-like behavior).
* Every time the EXE runs, it can spawn a new inbound session.

It’s not "persistence" in the registry/service sense; it’s more like **"booby-trapped tool" persistence**: the attacker gets repeated access as long as users keep running the program.

MITRE ATT&CK mapping:

* **T1059 / Execution** (running the trojanized binary)
* **T1105 Ingress Tool Transfer** (malicious binary introduced)
* **T1071 / C2** or **T1571 Non-Standard Port** (reverse TCP on a chosen port like 4444)
* **T1036 Masquerading** (legit program name/behavior)

---

## What to monitor on the Windows machine

### 1) Suspicious process execution 

Things that stand out:

* A "normal" EXE (like `radmin.exe`) executed from an unusual location (e.g. `C:\`, `Downloads`, `Temp`, user profile folders).
* Child processes or weird behavior right after it runs (like network connections opening immediately).
* If the payload injects/migrates: process injection indicators (depending on technique).

**Windows logs to rely on:**

* Security Event Log **4688** (Process Creation) if enabled.
* **Sysmon** Event ID **1** (Process Create) is much better if available.
* Sysmon Event ID **7** (Image loaded) and **10** (ProcessAccess) can help for injection patterns (optional).

### 2) Outbound network connections to unusual ports

Reverse shells are noisy from a monitoring perspective:

* The victim initiates an outbound connection to an attacker IP (e.g. `192.168.0.1`) on a port like **4444**.
* If the tool is run repeatedly, you’ll see repeated outbound connections and multiple sessions.

**Network telemetry logs:**

* Sysmon Event ID **3** (Network connection) is the easiest way to catch this locally.
* Windows Firewall logs (if enabled) also show allowed outbound connections.

### 3) File creation / binary replacement (trojanization evidence)

The trojanized EXE being dropped onto the system is itself an artifact:

* File written to disk (often in a weird directory).
* Hash mismatch compared to known-good.
* PE file with suspicious characteristics (unsigned, weird compilation timestamp, anomalous sections).

**Useful sources:**

* Sysmon Event ID **11** (FileCreate)
* Wazuh **FIM** (file integrity monitoring) if configured on directories like `C:\Windows\`, `C:\Program Files\`, `C:\Users\*\Downloads`, etc...

### 4) "Initial access" artifact (if MS08-067 was used)

Even if the focus is the trojanized EXE, the initial compromise matters:

* SMB exploitation often precedes the drop.
* There may be suspicious inbound SMB activity around the time of compromise.

If the Windows host is old (XP era), logging is limited; still, Wazuh can capture event logs where possible, and network sensors can help.

---

## How to monitor it on the machine

### Process and network quick checks (Windows)

* Look for `radmin.exe` running and inspect its path.
* Check active network connections:

  * On  Windows: `netstat -ano`
  * Map PID -> process to see which program owns the connection.

Key things that look wrong:

* a remote admin tool creating an outbound connection immediately on start,
* outbound to an IP that is not part of the normal admin infrastructure,
* use of a port like `4444`.

---

## How to hunt it in Wazuh 
### What Wazuh needs enabled for good detection

Best case:

* Wazuh agent on Windows
* Sysmon installed + configured
* Wazuh collecting:

  * Windows Security logs (4688)
  * Sysmon operational logs (Event IDs 1, 3, 11, etc.)
* Wazuh FIM enabled on relevant directories




