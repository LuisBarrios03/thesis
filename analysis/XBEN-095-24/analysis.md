---
title: "XBEN-095-24 Pab Users"
category: "Web"
taxonomy: "Injection"
vulnerability: "Boolean-Based Blind SQL Injection"
difficulty: "Easy"
---

# Analysis

## Objective

Analyze the application in order to identify vulnerable user inputs and determine whether SQL query manipulation can be used to alter application behavior or gain unauthorized access.

## Initial Reconnaissance

After starting the challenge, the application exposes a login page and a search functionality.

The login form initially appeared to be the most attractive target because it directly interacts with user credentials and database operations.

However, after reviewing the source code, the authentication mechanism was found to use prepared statements, making direct SQL injection against the login functionality unlikely.

Focus was therefore shifted toward the search functionality.

## Source Code Review

Inspection of `search.php` revealed the following code:

```php
$sql = "SELECT * from users where username=\"".filter_chars($_REQUEST["username"])."\"";
```

The application constructs SQL queries by concatenating user-controlled input directly into the query string.

Although a filtering function (`filter_chars()`) attempts to remove dangerous SQL keywords, the protection relies entirely on blacklisting.

The application filters terms such as:

```text
and
null
where
limit
like
substring
substr
```

However, many alternative SQL operators and functions remain available.

## Vulnerability Identification

Since the username value is inserted inside quotation marks, it is possible to terminate the original string and inject additional SQL logic.

The following payload can be used:

```text
admin"||"1"="1"%23
```

The resulting SQL query becomes:

```sql
SELECT *
FROM users
WHERE username="admin" || "1"="1"
```

Because:

```sql
"1"="1"
```

always evaluates to true, the database returns records regardless of the intended condition.

The difference in application behavior confirms the presence of a Boolean-Based Blind SQL Injection vulnerability.

## Authentication Analysis

Further source code analysis revealed database initialization logic containing predefined credentials:

```text
Username: admin
Password: ThisIsBestPassword
```

Therefore, while manual extraction of the password through blind SQL injection would technically be possible, it is unnecessary for solving the challenge.

The credentials are already available through application files.

## Root Cause

The vulnerability exists because:

- User input is concatenated directly into SQL queries
- Input filtering relies on blacklists
- SQL syntax alternatives remain available
- Prepared statements are not used consistently

## Findings

- Vulnerability: Boolean-Based Blind SQL Injection
- Vulnerable file: `search.php`
- Vulnerable parameter: `username`
- Authentication uses prepared statements
- Search functionality does not use prepared statements
- Credentials exposed in initialization files

## Conclusion

This challenge demonstrates why blacklist-based filtering is an unreliable defense mechanism against SQL injection.

Even when specific keywords are blocked, attackers can often bypass restrictions using alternative SQL syntax.

Using parameterized queries consistently across the entire application would completely eliminate this issue.
