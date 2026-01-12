## Pentest tools (from my lab writeups)

### 1) Recon and Enumeration

* **nmap** (basic port checks / scanning) 
* **Metasploit port scanner** (`auxiliary/scanner/portscan/tcp`) 
* **LinEnum** (Linux local enumeration script) 
* **Linux Exploit Suggester** (suggests kernel/local exploits based on uname/kernel) 
* **searchsploit** (search exploit-db locally for matching exploits) 
* **gdb** (binary reversing/debugging, registers, stack inspection) 
* **Windows built-ins**: `net users`, `net localgroup Administradores/Administrators` 

### 2) Vulnerability validation and Web exploitation

* **DVWA (web app lab target)** (XSS, CSRF, command injection, file upload) 
* **curl** (replay authenticated requests with cookies, quick testing) 
* **sqlmap** (automated SQL injection discovery + dumping) 

### 3) Exploitation / Initial access

* **Metasploit (msfconsole)** (e.g., Windows SMB MS08-067 exploit) 
* **netcat (nc)** (simple network interaction, reverse shells, listeners, basic IO)
* **Buffer overflow tooling** (pattern/offset workflow + payload structure) 

### 4) Credential attacks / Credential access

* **Hydra** (online password guessing against SSH in the lab) 
* **crunch** (generate small wordlists) 
* **Meterpreter `hashdump`** (dump local hashes once privileged) 
* **Metasploit SMB capture server** (capture NTLM challenge/response in lab) 

### 5) Privilege escalation (Linux)

* **gcc** (compile local exploit code in lab) 
* **find** (enumerate SUID binaries) 
* **sudo / visudo / sudo -l** (check sudo rules, env_keep issues like `LD_PRELOAD`) 
* **Cron inspection** (`/etc/crontab`, `/etc/cron.*`) 

### 6) Post-exploitation (control, persistence-like behavior and host discovery)

* **Meterpreter core commands** (`getuid`, `ps`, `migrate`, `shell`) 
* **Incognito (Meterpreter extension)** (token listing/impersonation concepts) 
* **FTP client** (observing weird backdoor-like login behavior in lab) 

### 7) Lateral movement and Pivoting / Tunneling

* **Metasploit routing** (`route add`, `route print`) 
* **Metasploit SOCKS proxy** (`auxiliary/server/socks_proxy`) 
* **proxychains** (force normal tools through SOCKS)
* **ssh dynamic forwarding** (`ssh -D` to create a SOCKS proxy) 
* **Tor** (SOCKS proxy at `127.0.0.1:9050`, layered proxying idea) 
* **iptables** (lab firewall simulation to force pivoting) 

### 8) Exfiltration and Covering tracks (concepts shown in lab notes)

* **tar / base64 / dd** (wrap + encode + lightly obfuscate data streams) 
* **Sysinternals** (defender tools: Autoruns, Procmon, TCPView, etc.) 
* **ADS (Alternate Data Streams)** (Windows hiding concept) 
* **rkhunter / chkrootkit** (rootkit detection idea) 

---

## How to run it

### Recon / Enumeration

* **nmap (basic check)**: run a scan against a target IP/port to see if it’s open/filtered, and compare behavior when a firewall blocks you. 
* **Metasploit TCP scan**: in `msfconsole`, load the portscan module, set `RHOSTS` (target) and `PORTS`, then `run`. This is useful when you already have Metasploit routing/pivoting set up. 
* **LinEnum**: clone it on the attacker box, copy it to the Linux target (like with `scp`), `chmod +x`, then run it with flags for thorough checks + report output. 
* **Linux Exploit Suggester**: download script, copy to target, run it locally to match kernel/version against known local privesc possibilities. 
* **searchsploit**: search exploit-db locally by keywords/version (ex: linux kernel, distro, and kernel branch), then copy the matching exploit source from `/usr/share/exploitdb/...` into your working folder. 
* **gdb (buffer overflow work)**: run the binary under `gdb`, disassemble interesting functions, run to crash, then inspect registers/stack (`info registers`, stack view) to confirm overwrite behavior. 

### Web exploitation (DVWA)

* **Stored/reflected XSS**: inject a `<script>` payload into the vulnerable input and verify JS executes in the DVWA origin; for session theft the writeup shows sending `document.cookie` to an attacker-controlled endpoint and then replaying the session cookie. 
* **CSRF**: build a simple external HTML page that triggers a state-changing request (DVWA low security accepts it), host it with a basic Python web server, and load it while the victim is logged in so their browser auto-sends cookies. 
* **Command injection**: test the normal input first (like an IP), then try chaining with shell separators (the writeup uses `;`) to confirm you can append system commands. 
* **File upload**: upload a file that will execute or run in a web-accessible location (DVWA low), then browse to the uploaded path to confirm impact (the notes discuss both harmless HTML/XSS and web shell style outcomes). 

### SQL injection

* **Manual SQLi checks**: start with a basic logic-bypass payload to confirm injection, then use `UNION SELECT` with `information_schema` to enumerate tables/columns and extract data. 
* **Blind SQLi**: infer data by asking true/false questions (page output changes), usually by checking one character at a time with substring conditions. 
* **Time-based SQLi**: if content doesn’t change, use a delay function (like `sleep`) to detect true/false based on response time differences. 
* **sqlmap**: run against the vulnerable URL + parameter, include your DVWA cookie so you stay authenticated, then target the DB/table and dump columns/rows automatically. 

### Exploitation and post-exploitation (Metasploit / Meterpreter)

* **MS08-067 foothold (lab)**: in `msfconsole`, select the SMB exploit module, set `RHOSTS` and `LHOST`, run `exploit`, then confirm you got a session with `sessions -l`. 
* **Basic Meterpreter situational awareness**: use `getuid` to confirm privilege, `getpid`/`ps` to see your process, and (in the notes) migrate to a more stable user process like `explorer.exe` when needed. 
* **Dropping to shell**: `shell` for a normal `cmd.exe` context, then run basic local enumeration like users/groups. 
* **Hashes and movement**: dump hashes (when privileged), then use SMB execution modules for lateral movement using creds/hashes (the writeup focuses on PSExec-style movement inside Metasploit). 
* **Token concepts (incognito)**: load the extension, list available tokens, impersonate a token to act as that user, and revert when done. 

### Pivoting / Proxying

* **Firewall simulation (lab)**: add an `iptables` rule on Linux to only allow traffic from the Windows pivot host, drop everything else, then verify Kali direct scans fail while pivoted scans work. 
* **Metasploit pivot route**: background your Windows session, add a route for the internal target through that session, and verify the route table with `route print`. 
* **SOCKS + proxychains**: start Metasploit’s SOCKS proxy, point `proxychains` at the local SOCKS port, then wrap normal tools (`nmap`, `ssh`, `curl`) so their TCP traffic goes through the pivot.
* **SSH dynamic SOCKS**: create a local SOCKS proxy with `ssh -D` when you have SSH access to a jump host, then route apps through it. 
* **Tor + proxychains**: run Tor locally, configure proxychains for Tor’s SOCKS port, then run apps (the notes use Firefox) through the chain. 

### Linux privilege escalation

* **Kernel exploit flow (lab)**: enumerate kernel version, use `searchsploit` to find a close match, transfer exploit source to target, compile with `gcc`, then run it (the writeup shows using a listener + a script to get a privileged shell in a controlled lab). 
* **Sudo / LD_PRELOAD idea**: check sudoers rules for unsafe environment keeping, compile a shared object, and run an allowed command with `LD_PRELOAD` preserved (the notes frame this as a misconfig example). 
* **SUID and GTFOBins**: enumerate SUID binaries and check if any common tools (like `less`) can be abused to read protected files or escape to a shell when running with elevated privileges. 
* **Cron jobs**: inspect system cron locations and look for writable scripts, missing scripts you can recreate, or PATH issues (commands called without absolute paths). 

### Credential attacks (lab)

* **crunch and hydra**: generate a small wordlist for a constrained password space, then run Hydra against SSH and watch for a valid login result (notes also mention rate-limiting/lockout as the real-world blocker). 

### Exfiltration 

* **Simple TCP exfil pipeline**: on attacker, listen with `nc` and write output to a file; on victim, archive + encode + send over TCP; then reverse the transforms back on attacker (decode + extract). 

### Covering tracks

* **Proxy layers to hide origin**: SOCKS proxies (SSH, Tor, Metasploit) chained with proxychains so the target sees the last hop instead of you. 
* **Host artifacts defenders look for**: strange accounts/credentials, logging gaps, ADS usage, and suspicious persistence entries (Sysinternals tools are listed as common blue-team checks). 


