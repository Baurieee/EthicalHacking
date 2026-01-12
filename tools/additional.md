## 0) Scoping 

* **Dradis / Serpico / Ghostwriter** (note-taking and reporting): create a project, log findings as you go, export report at the end.

**How are they used**

* Start a notes template (scope, dates, targets, exclusions, credentials provided, rules of engagement), then record every command/output you rely on for findings.

---

## 1) Passive recon / OSINT (no direct interaction, mostly)



* **theHarvester**: collect emails, subdomains, and org info from public sources.
* **crt.sh** (certificate transparency): spot subdomains via issued certs.
* **Shodan**: find exposed services by search queries.

**How are they used**

* Pull subdomains/domains, then dedupe into a single target list, then save the raw outputs as evidence for methodology.

---

## 2) Active recon / Discovery


* **masscan**: very fast what ports respond discovery (usually followed by deeper scans).
* **rustscan**: fast port discovery to feed into Nmap.
* **dnsx**: validate which discovered subdomains actually resolve.

**How are they used**

* Identify live hosts and open ports first then do a slower, more accurate scan on only the live targets.

---

## 3) Scanning and Service enumeration


* **Nmap NSE scripts** (beyond basic scanning): service-specific checks (SMB, HTTP, SSL, etc.).
* **WhatWeb**: quick web tech fingerprinting.
* **httpx**: confirm which hosts/ports are actually running HTTP(S), grab titles/status codes.
* **enum4linux-ng, smbclient / smbmap**: SMB share/user/policy enumeration.

**How are they used**

* For each open service, run the matching enumerator and capture version/config details.

---

## 4) Web app testing (content discovery and manual testing)


* **Burp Suite**: intercept traffic, repeat requests, test auth/session handling, automate basic checks.
* **OWASP ZAP**: similar role to Burp (often used for spidering and baseline scans).
* **gobuster / dirsearch**: content discovery (hidden paths, files, backups).
* **Nuclei**: template-based checks for known web misconfigs/CVEs (good for quick coverage).
* **Nikto**: older but still used for basic risky-file/misconfig checks.
* **WPScan** (when WordPress): enumerate plugins/themes and known issues.

**How are they used**

* Map the app (endpoints, roles, auth flows), then test input points (forms, JSON, headers) and access control (IDOR/auth bypass) with an intercepting proxy.

---

## 5) Vulnerability assessment 


* **Nessus / OpenVAS (Greenbone)**: broad vuln scanning with plugins/signatures.
* **Trivy** (containers/images/filesystems): common in cloud environments.

**How are they used**

* Treat scanner results as leads, then manually confirm the vuln is real, in-scope, and exploitable/impactful.

---

## 6) Exploitation

* **Metasploit Framework** (broader modules than just one exploit): controlled exploitation and session handling.
* **CrackMapExec**: internal network enumeration and auth testing across many hosts.
* **exploitdb**: confirm exploit details and prerequisites.

**How are they used**

* Match a confirmed vuln and exact version prerequisites, then attempt exploitation in the least disruptive way and document success criteria.

---

## 7) Credential testing (password/audit workflows)

* **John the Ripper / Hashcat**: offline password auditing when you have *authorized* hashes.
* **Medusa / Patator**: online login testing (like Hydra, but different modules/workflows).

**How are they used**

* Prefer offline auditing (safer and less noisy) when possible, for online testing, keep strict rate limits and follow the engagement rules.

---

## 8) Privilege escalation (local)

* **linPEAS / winPEAS**: automated checks for common misconfigs and escalation paths.
* **Seatbelt** (Windows): local security posture enumeration.
* **PowerUp** (Windows): common escalation checks (services, permissions).
* **PrivescCheck** (Windows): structured checks and reporting.

**How are they used**

* Run an enum tool to generate leads, then manually verify the exact permission/misconfig and reproduce the escalation with clear evidence.

---

## 9) Lateral movement and AD analysis 


* **BloodHound and SharpHound**: collect AD relationship data and graph attack paths.
* **rpcclient / net / PowerShell AD cmdlets**: targeted enumeration of users, groups, shares.

**How are they used**

* Collect AD data, identify shortest highest-confidence paths to objectives, then validate each step against scope and allowed techniques.

---

## 10) Reporting / Remediation guidance

* **CVSS calculators**: consistent severity scoring and remediation references.
* **Screenshots and command logs and timeline**: evidence package.

**How are they used**

* For each finding: What is it -> How to reproduce -> Impact -> Fix -> Proof, plus affected assets and exact dates.

