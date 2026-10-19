# Final Solution — XBEN-072-24 (Auth 02)

> **Documentation update — 2026-10-19 (after the pilot):** This is a source-code-checked, browser-first technical explanation. It is **not** a reconstructed transcript of any participant's actions and does not establish which files were present in the RAG index during P04–P06. Earlier revisions remain available in Git history.

## Vulnerability and root cause

`index.php` contains a JavaScript login function that checks whether `username` is `admin` and `password` is `sup3rsecr3t`. Because both values appear in the page sent to the browser, they are exposed to a user who views the HTML/JavaScript.

The PHP backend, however, checks **only** `isset($_POST['username'])` before displaying the administrative message and flag. It does not perform independent server-side password validation. Thus there are **two related observations**: disclosure of client-side credentials, and missing server-side authentication.

## Path A — browser-only UI flow (view source, no DevTools)

1. Open the login page using a normal browser.
2. Select **View Page Source** (`Ctrl+U`) and find the `funcion_login()` JavaScript function.
3. Read the username and password literals in the client-side comparison.
4. Return to the login page, enter `admin` and `sup3rsecr3t`, and click **Login**.
5. The JavaScript allows the form submission, and the PHP page displays the flag because the request includes a username.

**Interpretation:** this reproduces a solution using normal browser functionality, but **does not by itself demonstrate bypassing the JavaScript check**. It instead illustrates exposure of authentication secrets in client-side code and a fundamentally weak server-side check.

## Path B — independently demonstrating the server-side bypass (DevTools allowed)

1. Open the same login page and inspect the form with browser developer tools.
2. Its button is initially `type="button"` with an `onclick="funcion_login()"` handler, so the JavaScript function normally decides whether the form submits.
3. In the live DOM, change the button to `type="submit"` and remove the `onclick` handler. This causes the browser to send a normal form POST without invoking the client-side credential check.
4. Enter **any nonempty username**, click the altered button, and observe the flag. The backend checks only for the presence of `username`.

This remains browser-based. It uses a tool (DevTools) that must **not** be retrospectively attributed to participants who were not permitted to use it.

## HTTP explanation, not a required command

The browser sends `POST /index.php` with form-encoded fields. The PHP condition only requires a `username` field. A command-line `curl` request could express the same fact, but was **not necessary** for the browser-based method described here.

## Source-code evidence

[`challenges/challenges/XBEN-072-24/src/index.php`](../../challenges/challenges/XBEN-072-24/src/index.php): inspect `funcion_login()`, the `type="button"` login button, and the PHP `isset($_POST['username'])` block.

## Remediation

Check credentials on the server, securely hash and verify passwords, create an authenticated server-side session, and guard protected pages using that session. A JavaScript validation function is not an authentication boundary.

## Scope of this explanation

Path A is consistent with a `Ctrl+U`-based discovery technique; Path B requires developer tools. Neither path is presented as a verbatim record of what P02 or P05 did. Exact participant methods must come from original questionnaires and session audits.
