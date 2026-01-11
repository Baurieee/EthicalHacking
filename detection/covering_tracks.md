# Post-Exploitation: Covering Tracks 

## What this is from attackers perspective

* **hide the origin** (proxies / SOCKS / SSH tunnels / Tor / proxychains)
* **hide actions on the host** (log clearing, disabling logging, rootkits)
* **hide payloads/data** (backdoor accounts, NTFS ADS, disguised files) 

---

## What to monitor on machines 

### 1) Backdoor credentials / weird service behavior (FTP example)

If an FTP service accepts unusual credentials (like odd usernames + blank password), that’s an IOC. Monitor:

* FTP auth logs (vsftpd/proftpd logs, `/var/log/auth.log`, `/var/log/vsftpd.log` depending setup)
* spikes of logins from unexpected IPs
* logins with empty password / strange usernames 

### 2) Proxy / Tor usage (hiding origin)

Host indicators:

* processes: `tor`, `proxychains`, browsers started through proxy wrapper
* outbound connections to Tor network / SOCKS ports:

  * local SOCKS often `127.0.0.1:9050`
  * SSH dynamic tunnel example uses local SOCKS like `127.0.0.1:12345` (`ssh -D`) 
    Network indicators:
* traffic to known Tor entry/relay/exit nodes (IP reputation helps)
* lots of short connections to many random IPs (Tor circuits)

### 3) Log tampering (Linux + Windows)

Linux indicators:

* `systemctl stop rsyslog` / `systemctl stop auditd`
* log files suddenly truncated: `auth.log` size drops to 0
  Windows indicators:
* event logs cleared (common event IDs exist for "log cleared")
* use of `wevtutil cl <logname>` 

### 4) Rootkit-style hiding

Hard to spot directly, but common signs:

* mismatch between tools (e.g. `ps` doesn’t show a process but network shows it)
* kernel/module changes
* suspicious binaries replacing system tools (trojaned `ls`, `netstat`) 

### 5) NTFS Alternate Data Streams (ADS)

Indicators:

* file access using `filename:streamname`
* hidden content inside "normal" files
  Example concept: `test.txt:secret.txt` isn’t obvious in a normal directory listing. 

---

## Wazuh: what to collect 

* Linux: `/var/log/auth.log`, syslog/journald, service logs (FTP), **auditd** if possible
* Windows: Security + System event logs, and ideally **Sysmon**
* Wazuh **FIM** on critical paths:

  * Linux: `/etc`, `/var/log`
  * Windows: sensitive directories + persistence locations


## whats suspicious

* Tor/SSH tunnels + proxychains on endpoints that shouldn’t need them. 
* Logging stopped or logs suddenly emptied. 
* Weird FTP authentication behavior (unexpected usernames/blank passwords). 
* File activity using ADS (`file:stream`) patterns on Windows. 
