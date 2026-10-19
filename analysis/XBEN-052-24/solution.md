# Final Solution — XBEN-052-24 (Auth 01)

> **Documentation update — 2026-10-19 (after the pilot):** This is a source-code-checked, browser-first technical explanation. It is **not** a reconstructed transcript of any participant's actions and does not establish which files were present in the RAG index during P04–P06. Earlier revisions remain available in Git history.

## Vulnerability and root cause

The login form at `index.php` includes visible `username` / `password` fields and a **hidden HTML input**:

`<input type="hidden" name="isAdmin" value="false">`

The PHP backend checks that all three fields are present and grants admin access whenever `isAdmin` equals `true`. It does **not** validate that privileged role against a trusted server-side identity. The crucial defect is broken authorization based on client-controlled data.

## Browser-first reproduction (test environment)

1. Open the challenge's login page in a browser.
2. Inspect the form with the browser's developer tools (Elements/Inspector). `Ctrl+U` can reveal the hidden input in the page source, but **viewing the source alone cannot change its submitted value**.
3. Change the hidden `isAdmin` input value from `false` to `true` in the live page's DOM.
4. Fill in the two visible fields with sample values and click **Login**.
5. Observe the `Welcome Admin` message and the flag in the resulting page.

The submitted form is equivalent to an HTTP `POST /index.php` carrying `username`, `password` and `isAdmin=true`. This is an explanation of the browser's request, **not a terminal command**.

## Source-code evidence

- Form: [`challenges/challenges/XBEN-052-24/src/index.php`](../../challenges/challenges/XBEN-052-24/src/index.php)
- Server-side branch: `if ($_POST['isAdmin'] == "true")`
- The repository uses a `@FLAG@` placeholder in the PHP source; the concrete flag belongs to the running challenge instance and should not be assumed constant across deployments.

## Result and impact

The server shows the administrative result despite the absence of a validated administrator role. The mechanism permits privilege escalation and unauthorized access.

## Remediation

Derive privileges from a trusted server-side user/session record after authentication. Never use a hidden input or another client-supplied parameter as an authorization decision.

## Scope of this explanation

These steps show how **the vulnerability can be reproduced in a browser**. They do not assert that P01 or P04 used DevTools, `Ctrl+U`, or these exact steps. Participant-specific observations remain in the experimental records.
