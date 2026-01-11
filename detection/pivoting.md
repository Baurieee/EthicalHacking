# Pivoting + SOCKS Proxy (Metasploit) + proxychains + TCP Exfiltration 

## What this is 

This is a **pivoting** scenario, one compromised Windows host is used as a **jump point** to reach a Linux host that the attacker machine cannot access directly (firewall/segmentation). Metasploit routing (`route add`) + a SOCKS proxy turns the compromised host into a proxy for scanning and access. 

---

## What to monitor 

### Pivoting indicators (Windows pivot host)

On the Windows machine acting as pivot:

* **New outbound connections** to internal targets (Linux IP/ports) that the Windows host normally wouldn’t talk to
* If pivoting uses Meterpreter: suspicious process + network behavior (reverse session + traffic tunneling)
* If SOCKS proxy is used on attacker side: the Windows host still becomes the "real source" of scans

What looks suspicious:

* many connections from Windows -> Linux across many ports (scan pattern)
* repeated short-lived TCP connects

### Pivoting indicators (Linux target behind firewall)

On the Linux machine:

* connections coming from **Windows IP** (allowed by firewall), not from Kali
* if scanning: many SYNs or connection attempts from the same internal host
* service logs show access coming from the pivot IP (`192.168.0.100`)

### Exfiltration indicators (Linux victim)

The exfiltration demo stands out by:

* `nc` running on victim
* outgoing connection to attacker IP on **odd port** (12345)
* use of pipelines: `tar | base64 | dd | nc` (command-line behavior)


## Whats "suspicious" 

* A Windows host suddenly making lots of internal connections (scan-like), plus a Linux host seeing bursts of traffic from that same Windows IP, and/or Linux running `nc` to send data out on a random port. 
