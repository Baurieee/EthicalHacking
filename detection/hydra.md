# SSH Online Brute-Force (Hydra) 

## What this attack is

An SSH online brute-force attack is when an attacker repeatedly tries passwords against an SSH service over the network until one works. Tools like Hydra automate this by sending many authentication attempts quickly. The main risk is account compromise when passwords are weak or when there is no rate limiting/lockout.

MITRE ATT&CK: **T1110 – Brute Force** (Credential Access).

---

## What to monitor on the machine

### Authentication logs (primary evidence)

On Linux hosts running OpenSSH, brute-force activity shows in authentication logs:

* `/var/log/auth.log` (Debian/Ubuntu/Kali)
* `/var/log/secure` (RHEL/CentOS)
* `journalctl -u ssh` / `journalctl -u sshd` (systemd)

Common log patterns that appear during brute force:

* `Failed password for <user> from <ip> port <port> ssh2`
* `Failed password for invalid user <user> from <ip>`
* `Invalid user <user> from <ip>`
* `Accepted password for <user> from <ip>` (if the brute force succeeds)

A typical brute-force pattern is **many failed logins in a short time** from the **same source IP**. If it succeeds, it’s common to see an `Accepted password` shortly after the failures.

### Network indicators

At network level the pattern is:

* repeated TCP connections to **port 22**
* bursts of short SSH sessions (many connects, quick disconnects)
* usually one external IP repeatedly hitting the same host


## Response / what to check after an alert

If Wazuh flags an SSH brute-force pattern:

* confirm if there is any `Accepted password` from the same IP
* validate whether the account exists and whether the login time matches expected user activity
* check for persistence indicators after login:

  * changes in `~/.ssh/authorized_keys`
  * new users or sudoers changes
  * unusual processes started soon after the login
* containment options:

  * temporary firewall block for the source IP
  * lock/reset account credentials
  * enforce rate limiting / lockout / MFA where possible...
