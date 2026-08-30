# GME InvoiceApp — Handoff Document

**Scope of this project:** everything lives in `F:\Claude Code` only (no other folders). No git / version control is set up here — that has actively shaped some decisions below (e.g. old code left in place as a rollback fallback instead of deleted).

**Test tenant used throughout development:** "Diners Fire Engineers"

**See also:** [SECURITY_AUDIT.md](SECURITY_AUDIT.md) — a full audit; all 7 findings now have fixes applied (XSS escaping across the whole app including `super-admin.html`, login moved off GET, real server-side logout, upgraded password hashing with transparent migration, generic pre-auth error messages, an activated-but-still-public `LOGIN_SECRET`/`API_SECRET` pair, and `Role` enforcement on the two genuinely admin-level actions). Two things still need action from you, not from code: (1) set the `API_SECRET` Script Property to match `LOGIN_SECRET` in index.html — see Finding 4's note for the exact value and why it's a very weak control regardless; (2) the XSS fix was done via careful manual reading, not an automated scan — treat it as thoroughly-checked, not provably exhaustive. Read that file before touching auth/session/login code or adding any new place that renders sheet-sourced text.

## 1. What this app is

A multi-tenant Google Apps Script + HTML invoicing app for small fire-safety/engineering businesses. Each tenant gets their own login, their own business-data spreadsheet, and their own branding (logo/signature/stamp/watermark). A separate control spreadsheet holds tenant/user/session records shared across all tenants.

## 2. File map

| File | Role |
|---|---|
| [Gscript.txt](Gscript.txt) | The entire Google Apps Script backend (paste into the Apps Script editor as `Code.gs`). Single-file `doGet`/`doPost` router. |
| [index.html](index.html) | The entire frontend SPA — served as the Apps Script web app's HTML output. All tenant-facing pages (Invoices, Quotations, Challans, Credit/Debit Notes, AMC, Certificates, Settings) live in this one file. |
| [super-admin.html](super-admin.html) | Standalone portal (separate URL/deployment target) for managing tenants and their users. Not tenant-facing. |
| `HANDOFF.md` (this file) | You are here. |

**No build step.** These are plain files — edit them directly, then paste/redeploy into Apps Script.

## 3. Architecture essentials

- **Multi-tenancy:** one control spreadsheet has `Tenants`, `Users`, `Sessions` sheets. Each tenant additionally has their *own* Google Sheet (its ID stored in the tenant's `Spreadsheet ID` column) holding their actual business data (Invoices, Quotations, Challans, AMC, Certificates, etc.).
- **Per-request tenant context:** `applyTenantContext(session)` (Gscript.txt) sets a global `ACTIVE_SPREADSHEET_ID` and replaces the global `COMPANY` object from the session's cached snapshot, on every request. `getSheet(name)` then resolves to the right spreadsheet automatically.
- **Sessions are sheet rows, not CacheService** (CacheService proved unreliable — could fail to persist for an immediate read-back). `validateSession(token)` re-reads fresh from the sheet every request; sliding expiration is applied on every valid use.
- **Session-cache staleness is a recurring theme.** `COMPANY` is a snapshot cached in the session at login time. Any settings change that should take effect immediately must call `updateSessionCompany(token, COMPANY)` — this is already wired for `handleSaveCompanySettings`, but keep it in mind if you add more editable company fields.
- **`ensureHeaders(sheet, expectedHeaders)`** — the standard pattern for schema migrations. Non-destructive: appends any missing header as a new trailing column, never touches existing data. Used for `INVOICE_HEADERS`, `TENANT_HEADERS`, `AMC_HEADERS`, `CERTIFICATE_HEADERS`. **Always read/write sheet rows by header name** (`headers.forEach((h,j)=>{obj[h]=row[j]})`), never by fixed column index — positional arrays drifting out of sync with an evolving header list has caused real bugs this session.
- **No PDF library exists in Apps Script.** The only way to produce a PDF is: write a self-contained HTML string to a temp Drive file, call `.getAs('application/pdf')` on it, then trash the temp file. This conversion is done by Google Drive's own HTML-to-PDF converter, which has real, confirmed quirks (see §6).

## 4. Features implemented this session

### AMC Contracts
- Image + GPS location capture per contract.
- Google Calendar reminder auto-created/updated on save, landing **on the exact expiry date** (not 30 days before) — with a separate 30-day-before **popup notification** on that same event. Manual "Add to Calendar" / "Sync now" button available too.
- Distinct status reporting (`skipped-no-calendar`, `skipped-no-expiry`, `skipped-invalid-date`, `created`, `updated`) so silent failures are distinguishable from "no calendar configured."

### Invoices — WhatsApp & Email sending
- "WhatsApp" and "Email" buttons send the *exact same-looking* PDF as Print/Preview (see §5 — this took several iterations).
- Backend actions: `sendInvoiceToBuyer` (email, always regenerates fresh), `getInvoiceWhatsappLink` (returns a wa.me link with a Drive-shared PDF link, **always regenerates fresh — do not reintroduce caching here**, see §7).
- `findInvoiceRow` / `buildInvoicePayloadFromRow` reconstruct the full print payload straight from the sheet row (including `subTotal`/CGST/SGST/IGST *amounts*, not just rates — the print engine trusts stored figures, never recomputes them).

### GST Type — "No GST" option (Invoice + Credit/Debit Note)
- The existing `CGST_SGST` / `IGST` doc-level dropdown (`inv-gst-type`, `cn-gst-type`) gained a third option, `NO_GST`, for tax-exempt bills.
- `getComputedTotals()`/`getCNComputedTotals()` now also return `cgstRate`/`sgstRate`/`igstRate` (0 for `NO_GST`) alongside the amounts — every save/print/preview call site was switched to read rates from that same object instead of `invState.taxes.*`/`cnState.taxes.*` directly, specifically so a No-GST bill never saves/prints a stale "@ 9%" label next to a ₹0.00 amount. `invState.taxes`/`cnState.taxes` themselves are left untouched, so switching back to CGST+SGST or IGST later restores the originally-configured rates.
- Each line item's own `gstPct` is also forced to 0 when saving under `NO_GST` — otherwise the itemized HSN-wise tax table would still show each item's originally-configured per-item rate even though the doc-level total is ₹0.00.
- `totalsBlock()` (both the client `LedgerPrint` and server `LedgerPrintGas` copies) skips the CGST/SGST/IGST row entirely when all three amounts are zero, instead of printing a misleading "IGST: ₹0.00" line. **Deliberately did NOT** apply the same suppression to the itemized `taxTable()` — pagination's height math (`footerHeight`/`taxBlockHeight`) reserves space for that table based on the document's *kind* (`DOC.tax`) alone, not on whether the amounts happen to be zero, so hiding it at render time only would leave an unexplained blank gap. A No-GST invoice's itemized tax table still shows "0%, ₹0.00" per row — correct data, just not maximally polished. Fixing that properly would mean threading the zero-check into the pagination functions themselves; flagged here rather than risked without live testing.
- Quotations don't have this toggle at all (no `qt-gst-type` exists) — out of scope, not touched.

### Certificates — full feature, added from scratch this session
- `CERTIFICATE_HEADERS` (+ `Certificate PDF URL` column), backend `buildCertificateHtml`/`renderCertificatePdf` (a Drive-safe, full-page port of the browser's `printCertificateHtml()`), `saveCertificatePdfToDrive`, `sendCertificateCopyToBuyer`.
- New actions: `sendCertificateToBuyer`, `getCertificateWhatsappLink` (also always-fresh, same reasoning as invoices).
- Frontend: WhatsApp/Email buttons in the certificates table, `sendCertificateWhatsapp`/`sendCertificateEmailToBuyer`.

### Branding — Logo / Signature / Stamp / Watermark (tenant-provisioned)
- Settings → Branding: upload Logo, Signature, Stamp. Signature/Stamp get automatic background removal (`chromaKeyToTransparentPng` — samples the 4 corner pixels as "background" color, alpha=0 for near-matches). Not applied to Logo (a logo's corners aren't necessarily background).
- **Stamp is auto-darkened 30%** at upload time (`darkenFactor` param on `chromaKeyToTransparentPng`) — stamp ink often photographs faded.
- **Watermark:** its own dedicated upload (Settings → Watermark card), deliberately separate from Logo — not auto-derived from it. Tenant also picks a style: **Tilted** (-22°) or **Straight** (0°), via `WATERMARK_ANGLES` in index.html. The raw uploaded image is kept as-is (`COMPANY.watermarkSrc` / Tenant sheet column `Watermark Image`) so switching styles later recomputes cleanly from the original photo rather than re-processing an already-rotated/faded copy. `makeWatermarkImage(img, angleDeg, opacity)` (renamed from `makeWatermarkFromLogo`) does the actual canvas rotate+fade (7% opacity, padded bounding box so corners aren't clipped), and the result is stored as `COMPANY.watermark` / Tenant sheet column `Watermark` — the ONE field every print engine actually reads. That indirection means the print/PDF code itself never changed when this was redesigned: `COMPANY.watermark` is still just a `background-image` behind every document (invoices, challans, quotations, credit/debit notes, certificates) in both Print and WhatsApp/Email PDFs, regardless of where that image came from.
- Stamp+Signature overlap: **blended to look like one mark** — signature drawn crossing the stamp. Browser engines use `position:absolute` + `transform:translate(-50%,-50%)` (safe in real browsers); Drive-rendered (server) engines use a `vertical-align:middle` + negative-margin technique instead (see §6 for why).
- Images >~a few KB as base64 blow past Sheets' 50,000-char cell limit — all branding images are uploaded to a Drive folder and only a short thumbnail-proxy URL (`https://drive.google.com/thumbnail?id=X&sz=w1000`) is stored. `maybeUploadBrandingImage(value, label)` does this uniformly for Logo/Signature/Stamp/Watermark.
- All darkening/rotation/fading is baked into the PNG's pixels **once, client-side, at upload time** — never a live CSS filter/transform at render time. This is deliberate (see §6) so the same processed asset renders identically in the browser and via Drive's PDF converter.

### Super Admin Portal ([super-admin.html](super-admin.html))
- Add/Edit/Delete tenants with full onboarding fields; per-tenant user management (add/edit/delete users, blocks deleting a tenant's last user).
- Gated by a `SUPER_ADMIN_SECRET` Script Property (`isSuperAdminSecretValid`), **not** a session token — it operates outside any tenant's login. `listTenants`/`listTenantUsers` (GET) and `provisionTenant`/`updateTenant`/`deleteTenant`/`addTenantUser`/`updateTenantUser`/`deleteTenantUser` (POST) must stay gated **before** the normal session-token check in the router (this was a real bug once — moving it after the session check always failed with "session expired").

### Dev login bypass
- `DEV_SKIP_PASSWORD_CHECK` (top of Gscript.txt, currently **`false`** — re-enabled/production-safe right now). Flip to `true` only for local testing, and remember to flip it back.

### The big one: server-side print engine port (`LedgerPrintGas`)
- The browser's `LedgerPrint` module (invoice/challan/quotation/credit-debit-note print engine) was **ported line-by-line into Apps Script** as `LedgerPrintGas`, so WhatsApp/Email PDFs are generated by (almost) the same code as Print/Preview, instead of a hand-maintained parallel implementation that kept drifting out of visual sync.
- Wired into `saveInvoicePdfToDrive`, `sendInvoiceCopyToBuyer`, `sendInvoiceEmail` via a shared `renderLedgerPrintPdf(data, COMPANY, kind, fileName)`.
- **The OLD renderer is still in the file, unused** (`buildInvoiceHtml`, `computeInvoiceLineCalc`, `sellerHtml`, `partiesHtml`, `bankBandHtml`, `sigStampCellHtml`, `totalsHtml`, `itemsTableHtml`, `INV_GEO`, `INV_AFM`) — deliberately kept as a rollback fallback since there's no version control. **Do not delete it** until the new engine has been confirmed fully correct in production for a while.

## 5. Bugs found & fixed this session (useful context, don't reintroduce)

- **Numeric-type coercion crashes** — Sheets can return `Invoice No`/`Buyer Phone`/etc. as JS `Number`, breaking `.replace()`/`.trim()`. Always `String(x || '')` before string methods on sheet-sourced values.
- **Double-print-dialog bug** — an `iframe.onload` handler could fire twice (once for the initial blank frame, once after `document.write()`). Fixed everywhere by calling the print logic **synchronously right after `iDoc.write();iDoc.close();`**, waiting only for images via `waitForImagesToLoad(doc, maxWaitMs)` (Drive-hosted signature/stamp/logo/watermark URLs need a real network fetch) — never rely on `onload`.
- **Signature/stamp cut off "Authorised signatory" text** — the bank/sign band has a fixed height budget (`GEO.bankH` = 22mm) shared with two text lines; oversized sig/stamp images (16mm/12mm) overflowed it, and since ancestors use `overflow:hidden`, the text silently got clipped instead of the box growing. Fixed by shrinking to sizes that actually fit the budget (11mm/9mm overlapping, 12mm single). **If you resize these images again, do the arithmetic against `GEO.bankH` first.**
- **Certificate PDF not using the full page** — first attempt at the server-side certificate renderer used content-sized (auto) height instead of the full A4 flex-column layout the browser version uses, so the border box only wrapped its content instead of stretching to the page. Fixed by mirroring the browser's full-page flex layout exactly.
- **WhatsApp PDF caching bug (important, see §7 for the fix)** — `handleGetInvoiceWhatsappLink`/`handleGetCertificateWhatsappLink` used to reuse whatever was already in the `Invoice/Certificate PDF URL` column and only regenerate when empty. This meant ANY later fix (branding, layout, watermark, etc.) would never show up on WhatsApp for an already-tested invoice/certificate — only Email (which always regenerated fresh) would reflect it. **Both handlers now always regenerate fresh on every call.**
- **Router-ordering bug** — super-admin GET actions (`listTenants`, `listTenantUsers`) were briefly placed after the session-token check, so they always failed with "Session expired." They must be gated by the super-admin secret **before** any session check.

## 6. Google Drive's HTML-to-PDF converter — confirmed quirks

This converter is what turns Gscript-generated HTML into the WhatsApp/Email PDFs. It is **not** a full modern browser, and everything server-side is written around its limitations:

- **`position:absolute` + `transform` is unreliable** — confirmed via an actual stray full-page-height vertical line appearing when this was tried for the stamp/signature overlap. Server-side engines use `vertical-align:middle` + a negative margin instead. **Do not reach for `position:absolute`+`transform` in any Gscript.txt-side rendering code without testing carefully** — this is why the watermark feature bakes its rotation into the PNG's pixels via canvas instead of using a live CSS `transform`.
- Flexbox itself is now **confirmed to work fine** (the whole `LedgerPrintGas`/`.inv-page` layout and the certificate's full-page layout both use it successfully) — an older code comment claiming Drive can't do flexbox is outdated; only `position:absolute`+`transform` is the actual problem.
- Doesn't zero default `<img>` borders the way browsers do — always add `border:0` explicitly on server-rendered `<img>` tags.
- `width:100%` + `border` without `box-sizing:border-box` lets the border render outside the specified width, which can get clipped asymmetrically at a page edge (this caused a left/right border-thickness mismatch in the old renderer once).
- Percentage heights are unreliable when the ancestor chain's height isn't fully explicit — prefer fixed mm heights (the `GEO`/`INV_GEO` pattern already used throughout).

## 7. Known-good invariants — please don't regress these

1. **WhatsApp/Email PDF generation must always be fresh**, never read a cached URL column first. (Was a real bug, just fixed — see §5.)
2. **Money/totals are never recomputed at render time** — always read stored figures (`Grand Total`, `CGST Amount`, etc.) directly from the sheet row, never re-derive from line items in the print engines.
3. **Never use `position:absolute`+`transform` in Gscript.txt's HTML-generation code** for anything that needs to render via Drive's converter — bake any rotation/overlap effect into pixels client-side first, or use the negative-margin/vertical-align technique already established.
4. **All new sheet columns go through `ensureHeaders()`**, added to the relevant `*_HEADERS` constant, never a one-off `sheet.appendRow([...])` with a hand-typed list (multiple headers arrays already exist as the single source of truth — `TENANT_HEADERS`, `INVOICE_HEADERS`, `AMC_HEADERS`, `CERTIFICATE_HEADERS`).
5. **Any tenant-editable COMPANY field must be added in *four* places** to actually take effect immediately: the `COMPANY` default object (client AND server), `handleLogin`'s `company` object (built from the Tenants sheet row), `handleSaveCompanySettings` (upload/set-cell/merge-into-COMPANY), and `TENANT_HEADERS`. Missing any one of these is exactly how the "logo missing on WhatsApp" investigation started (that turned out to be the caching bug instead, but the four-places rule is still the thing to check first for "my new company field doesn't show up somewhere").
6. **No `Date.now()`/`Math.random()`/argless `new Date()` assumptions carry over between client and server** — timestamps are computed independently in each engine; this hasn't caused a bug yet but keep it in mind if you add anything date-sensitive to the shared print payloads.

## 8. Explicitly NOT implemented (discussed, not built)

- **Legally-binding PKI Digital Signature (DSC/eSign)** — discussed feasibility only, no code written. Conclusion: **not possible purely within Apps Script** (no PDF-signing library, no access to a physical USB DSC token or HSM). The only real path is integrating a licensed third-party eSign API (Digio, Leegality, NSDL/CDAC Aadhaar eSign, DocuSign, etc.) via `UrlFetchApp` — a materially bigger project (vendor onboarding, per-document cost, OTP-based signer flow, real API integration work), not a quick addition. Worth first confirming whether the tenant's actual documents (standard GST tax invoices) legally require this at all — normally they don't; a visual signature/stamp (already built) is standard and sufficient.

## 9. Deployment reminders (Apps Script, not this repo)

- There is **no CI/CD** — after editing Gscript.txt/index.html locally, you must paste the changes into the Apps Script editor and go **Deploy → Manage deployments → (pencil/edit icon) → Version: "New version" → Deploy**. Saving the script alone does **not** update the live `/exec` URL — this has caused confusion more than once this session (a user testing against a stale deployed version looks identical to a bug that "isn't fixed").
- Local syntax validation (since there's no real GAS test environment available here) has been done via Node: `node -e "const fs=require('fs'); new Function(fs.readFileSync('Gscript.txt','utf8')); console.log('OK')"`. This only catches syntax errors, not runtime/GAS-API issues — always ask the user to test against a real deployment.
- `super-admin.html` is a **separate** page/deployment target from `index.html` — don't assume changes to one automatically apply to the other's copy of any shared-looking code (e.g. `gasPost`/`gasGet` helpers are duplicated, not shared).

## 10. Suggested next steps for whoever picks this up

- Confirm the WhatsApp-PDF-caching fix (§5/§7) is deployed and that a previously-stale invoice/certificate now reflects current branding.
- Once the new `LedgerPrintGas` engine has been trusted in production for a while, delete the old unused renderer functions listed in §4 to reduce dead code (holding off only because there's no version control to fall back on).
- If a genuine legal-compliance need for DSC/eSign turns up, scope out a third-party eSign API integration as its own project (see §8) rather than trying to build it into Gscript.txt directly.
