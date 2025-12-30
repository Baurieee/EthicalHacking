# Post-Exploitation and Covering Tracks 

---

## 1. Backdoor Behaviour

In this lab scenario, we had an FTP service on the Linux machine (`192.168.0.101`) that behaved like this:

```bash
ftp 192.168.0.101
Name (192.168.0.101:kali): foo:)
Password:
230 Login successful.
ftp>
```

* Username: `foo:)`
* Password: *blank*


* It might be a **hardcoded backdoor** added by an attacker to guarantee easy access later.

From an attacker’s perspective:

* They might create "hidden" accounts or credentials that **don’t show up in obvious lists**.
* Or they modify existing services (like FTP, SSH, or web apps) to accept a magic username/password combination.

From a defender’s perspective:

* Strange accepted credentials are an **Indicator of Compromise**.
* Log analysis (FTP logs, auth logs) or honeypot accounts can help detect this behaviour.

---

## 2. Hiding the Origin: Proxies, Chains and "Loops"

Once an attacker is inside, they usually don’t want the defender to easily trace back where the attack came from. This is where **proxies** and **proxy chains** come in.

### 2.1 Application proxies vs SOCKS vs "generic proxies"

* **Application proxies**

  * Typically HTTP/HTTPS proxies that understand one protocol.
  * Example: a corporate web proxy that inspects HTTP/HTTPS traffic.
  * Tools: Burp, ZAP, Squid, ...

* **SOCKS proxies**

  * More "generic": operate at the TCP/UDP level and don’t care about higher-level protocols.
  * You can send **ANY** TCP-based traffic (HTTP, SSH, RDP, FTP…) through them.
  * Metasploit’s `socks_proxy` module is an example.

* **"Proxysites" / generic public proxies**

  * Random web pages that forward HTTP requests for you.
  * Very noisy and often logged, but easy to use as an extra layer in a chain.

From the attacker’s view:

* They might chain several of these so that the defender only sees the **last hop**.
* In extreme cases, they make **loops** or multiple hops to confuse investigators.

---

## 3. SSH Dynamic Tunnels (SOCKS Proxy via SSH)

A simple way to create your own SOCKS proxy is using **SSH dynamic port forwarding**.

Command:

```bash
ssh -D 12345 -N -f user@server
```

Breakdown:

* `-D 12345`

  * Open a local SOCKS proxy on port `12345`.
* `-N`

  * Don’t execute a command, just set up the tunnel.
* `-f`

  * Go to background after authentication.

Now local machine has a SOCKS proxy listening at `127.0.0.1:12345`. Any traffic you send through that proxy will appear to come from `server`.

In a red-team context, this can be:

* A jump host inside the target network.
* A VPS somewhere on the Internet used to hide your real IP.

---

## 4. Tor and Proxychains – Layering Anonymity

### 4.1 Tor basics

Tor uses **onion routing**:

* Your traffic goes:
  **Tor client -> entry guard -> middle relay -> exit relay -> destination**
* Each relay only knows the previous and next hop, not the entire path.
* This makes it much harder (not impossible, but harder) to correlate source and destination.

You can interact with Tor either via:

* **Tor Browser** (for normal browsing), or
* Running a **Tor service** and pushing arbitrary traffic through it.

On Linux:

```bash
sudo apt install tor
sudo systemctl start tor.service
```

Tor typically opens a SOCKS proxy on `127.0.0.1:9050`.

### 4.2 Proxychains configuration (with Tor or Metasploit SOCKS)

`/etc/proxychains4.conf`. For Tor, it might look like:

```bash
sudo nano /etc/proxychains4.conf
```

At the bottom:

```text
# old default
#socks4  127.0.0.1 9050

# example if we want to use port 1080 instead
socks4  127.0.0.1 1080
```

**Tor’s default**, you usually set:

```text
socks4  127.0.0.1 9050
```

### 4.3 Dynamic vs strict chain (proxychains)

In `proxychains.conf`:

* **dynamic_chain**

  * Proxychains tries proxies in the list in order.
  * If one fails, it skips and continues with the next.

* **strict_chain**

  * Proxychains will use **all** proxies in the order defined.
  * If one fails, the whole chain fails.

For maximum stealth (and complexity), an attacker might use strict chains: traffic goes through all specified proxies. For reliability, dynamic chains are easier.

### 4.4 Using Firefox through proxychains

Once Tor is running and proxychains is configured, we can force Firefox to use the chain:

```bash
proxychains firefox
```

Firefox’s own proxy settings can be left as default ("No proxy") because `proxychains` intercepts the traffic at the OS level.

If configured properly, Firefox’ connections (HTTP/HTTPS) will now go:

`Firefox -> proxychains -> Tor SOCKS -> Tor network -> destination`.

This is something a defender might see as "traffic from random Tor exit nodes" rather than "traffic from the actual attacker".

---

## 5. Classic "Covering Tracks" on the Host

Once an attacker is inside the system, beyond just hiding where they came from, they also try to hide **what they did**.

This usually involves:

* **Logs** (system logs, application logs, security logs).
* **Malware / tools** left on disk.
* **Alternate data streams** and other ways to hide content.
* **Persistence mechanisms** (backdoors that don’t look obvious).

### 5.1 Disabling logging or wiping log files

Examples:

* Linux:

  * Stop logging services: `systemctl stop rsyslog`, `systemctl stop auditd`
  * Clear logs: `> /var/log/auth.log`, `> /var/log/syslog`
    (note: doing this blindly is very noisy; defenders hate empty logs)

* Windows:

  * Use event log management tools or `wevtutil cl System` to clear logs.
  * Malware/rootkits may hook logging APIs to drop or modify entries.

From defender’s perspective, logs that suddenly stop or **big gaps / empty files** are themselves suspicious.

### 5.2 Rootkits and hiding files

Rootkits aim to:

* Hide processes
* Hide files
* Hide network connections

They do this by:

* Hooking system calls in the kernel or userland.
* Modifying outputs of tools like `ps`, `netstat`, etc.

Notes:

* Just because `ls` doesn’t show a file doesn’t mean it’s not there.
* Tools like **rkhunter**, **chkrootkit**, and integrity-check tools (Tripwire, AIDE) are used to detect them.

---

## 6. Alternate Data Streams (ADS) on Windows

NTFS has a feature called **Alternate Data Streams** (ADS), originally used for compatibility and metadata. Attackers abuse this to hide content "behind" normal files.

Example:

```bash
notepad test.txt
notepad test.txt:secret.txt 
```

What’s happening:

1. `notepad test.txt`

   * Opens the **main** data stream of `test.txt`.

2. `notepad test.txt:secret.txt`

   * Opens an **alternate stream** named `secret.txt` attached to `test.txt`.

So:

* The file listing shows only `test.txt`.
* But there is hidden content in the `secret.txt` stream that normal tools might not show.

Attackers can:

* Store **malware** or scripts inside `test.txt:secret.txt`.
* Then call it with something like:

  * `more < test.txt:secret.txt`
  * Or run it if the program knows how.

From a defense POV:

* Tools like `streams.exe` (from Sysinternals) or advanced forensic tools can enumerate ADS.
* ADS is another reason why "file looks normal" is **not enough**.

---

## 7. Sysinternals for Detection

**Sysinternals** is a suite of tools by Microsoft that defenders use heavily. Examples:

* **Process Explorer** – advanced Task Manager.
* **Autoruns** – lists all persistence mechanisms (startup items, services, scheduled tasks, etc.).
* **TCPView** – shows network connections in detail.
* **PsExec** – remote execution (also used by attackers!).
* **Procmon** – shows file/registry/process activity.

Understanding tools:

* Blue team analysts will run them to look for your processes, services, DLLs.
* If you know what they look for, you know **what you need to hide**.

Typical defensive use:

* `autoruns.exe` to find weird startup entries or unknown EXEs.
* `tcpview.exe` to see unusual external connections (like to Tor exit nodes or weird C2 servers).
* `procmon.exe` to see which malware process is touching which files/registry keys.

---

## 8. Summary: Attacker View vs Defender View

From the **attacker’s** lab perspective, post-exploitation covering tracks often involves:

1. **Hiding origin:**

   * Use proxies (HTTP, generic, SOCKS), SSH tunnels, Tor, proxychains.
   * Potentially chain multiple proxies in strict order to make tracing harder.

2. **Hiding activity:**

   * Delete or alter logs (dangerous and noisy).
   * Disable logging systems.
   * Use rootkits or stealth mechanisms.

3. **Hiding payloads/backdoors:**

   * Backdoor accounts (like FTP user `foo:)` with blank password).
   * Store payloads in alternate data streams (e.g., `file.txt:secret.txt`).
   * Persistent but unobvious services or scheduled tasks.

4. **Maintaining access:**

   * Drop web shells, SSH keys, RDP backdoors.
   * Make sure they’re hard to notice but easy for you to reuse.

From the **defender’s** point of view:

* Check for odd accounts and credentials.
* Look at logs for gaps, anomalies, or disabled services.
* Use tools like Sysinternals, rkhunter, auditd, etc. to detect hidden processes and ADS.
* Monitor for Tor exit-node activity, unusual proxies, or odd SSH tunnels.

