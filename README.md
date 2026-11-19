# Expense Management System

Two separate interfaces backed by the existing Google Apps Script and Google Sheets expense system.

## Applications

- **Expense Details** — mobile-first expense capture, UPI share intake, receipt upload, OCR-assisted field extraction, and confirmation before saving.
- **Expenses Monitor** — restricted management dashboard for ledger totals, expense history, receipts, reports, CSV export, and member administration.

## Existing data source (preserve)

The system must continue using the existing spreadsheet and tabs:

- Spreadsheet ID: configured only in the private Apps Script project
- `EXP_MEMBERS`
- `EXP_TRANSACTIONS`
- `EXP_SETTINGS`

Receipt files remain in the existing private Google Drive folder. Do not make the spreadsheet or receipt folder public. Do not rerun the initial setup function on a populated system.

## Important deployment design

GitHub Pages can host static files, but it cannot execute Apps Script server functions or safely hold secrets. The existing Apps Script project remains the authentication, authorization, data, and receipt backend.

The current Apps Script UI uses `google.script.run`, which is available only to pages served by Apps Script. Therefore the production screens must either be served from Apps Script or use a separately verified secure API bridge. Do not replace `google.script.run` with an unverified cross-origin `fetch` workaround.

Android share-target support can be hosted as a PWA on GitHub Pages. iPhone users need an Apple Shortcut or the in-app upload/share fallback; iOS does not provide the same general PWA share-target registration as Android.

## Security rules

- Every user signs in with their own username and PIN.
- Authorization is enforced on the server, not by hiding interface elements.
- Super Admin can use OFFICE and PERSONAL and can see all records.
- Personal Assistant is assigned PERSONAL; Office Staff is assigned OFFICE.
- Non-admin members see only their own submissions.
- The server sets Submitted By from the authenticated session; never trust a client-supplied identity.
- Never commit PINs, PIN hashes, session tokens, Apps Script secrets, spreadsheet credentials, or private receipt URLs.
- UPI/OCR extraction is a suggestion only. Require the user to review and confirm every transaction before saving.

## Current phase

This repository is the source-control home for the separated application interfaces and share-intake PWA. Existing production data and Apps Script deployment are not modified by adding files here.
