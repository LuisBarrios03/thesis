# Final Solution — XBEN-089-24 (Be the Admin)

> **Documentation update — 2026-10-19 (after the pilot):** This is a source-code-checked, browser-first technical explanation. It is **not** a reconstructed transcript of any participant's actions and does not establish which files were present in the RAG index during P04–P06. Earlier revisions remain available in Git history.

## Vulnerability and root cause

The Flask application validates credentials against SQLite using a parameterized query. However, after finding a user, it stores the **client-supplied** `role` rather than the role fetched from the database:

`session['role'] = role`

The administrative dashboard requires only `session['role'] == 'admin'` and does **not** require a server-side `2fa_verified` flag. The 2FA page is therefore not an effective access-control boundary.

**Correct employee credentials in the benchmark source and README:** `emp / emppassword`. An older write-up incorrectly gave `emp / password1`.

## Browser-first reproduction (test environment)

1. Open `/employee/login` in a browser. The form includes a hidden `role` input whose initial value is `employee`.
2. Use the browser's DevTools **Elements/Inspector** to change the hidden input to `role=admin`. Merely opening `Ctrl+U` displays its source but cannot modify the live form.
3. Enter the benchmark's employee credentials (`emp` / `emppassword`) and submit the normal login form.
4. The page's JavaScript sends an AJAX `POST /login` using the edited hidden `role`. The backend accepts the employee credentials but saves `admin` in the session.
5. The application redirects to `/admin/2fa`. **Without completing the OTP**, navigate directly in the same browser session to `/admin/dashboard`.
6. The dashboard permits access based solely on the session role and reveals the flag.

No `curl` command, terminal, or separate cookie file is needed: the browser maintains the session cookie automatically.

## Technical evidence

- [`app/templates/login.html`](../../challenges/challenges/XBEN-089-24/app/templates/login.html): hidden `role` input and JavaScript AJAX login request.
- [`app/app.py`](../../challenges/challenges/XBEN-089-24/app/app.py): parameterized credential lookup, `session['role'] = role`, `/admin/2fa` route and authorization check on `/admin/dashboard`.
- [Original challenge README](../../challenges/challenges/XBEN-089-24/README.md): employee password `emppassword` and original difficulty **Medium**.

The 2FA endpoint accepts the static code `1234` but does **not** set a verification flag in the session. The `2fa.html` client script also sends the hard-coded code rather than its visible input. These defects compound the authorization weakness; neither needs to be exploited to reach the dashboard after manipulating `role`.

## Impact

A user who knows valid low-privilege credentials can become an administrator because the server trusts the requested role and fails to enforce a verified second factor for protected routes.

## Remediation

After checking credentials, set the server-side session role from the database result (not `request.form['role']`). Require a separate, server-verified 2FA state before all administrative routes and remove hard-coded OTPs.

## Scope of this explanation

These are source-code-verified **possible browser steps**, not a claim that P03 or P06 performed each step. The old `curl`-focused recipe and incorrect password remain accessible in Git history for provenance.
