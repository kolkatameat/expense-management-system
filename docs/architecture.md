# Architecture and permission model

## Components

1. **Expense Details UI**: sign-in, share/upload intake, extraction review, ledger selection based on server-issued permissions, owner/paid-by fields, and submit confirmation.
2. **Expenses Monitor UI**: admin-only dashboard, ledger summaries, filters, member management, receipts, and CSV export.
3. **Apps Script backend**: authoritative login/session validation, ledger authorization, own-submission filtering, receipt access, duplicate UPI ID checks, and spreadsheet writes.
4. **Google Sheets and private Drive**: existing database and receipt storage; retain current IDs and data.

## Roles

| Role | Ledger access | Record visibility | Monitor |
|---|---|---|---|
| Super Admin | OFFICE + PERSONAL | All records | Full |
| Personal Assistant | PERSONAL | Own submissions only | No |
| Office Staff | OFFICE | Own submissions only | No |

The administrator assigns each member's ledger and active status. The authenticated session determines the submitting username and display name.

## Expense attribution

Keep these concepts distinct:
- **Expense Owner**: the person the expense belongs to.
- **Paid By**: the person who actually made the payment.
- **Submitted By**: the signed-in account that submitted the record; set by the server.

A personal assistant may submit an expense on behalf of the owner, with the assistant recorded as Submitted By.

## Safe integration constraints

- Never expose the spreadsheet or receipt folder publicly.
- Never trust a ledger value merely because the client UI displayed it; validate it against the current session on every write.
- Apply identical authorization checks to expense listing, dashboard aggregation, CSV export, and receipt retrieval.
- Treat shared UPI text and OCR output as untrusted input.
- Check duplicate UPI transaction IDs under a script lock.
- Do not run setup or migration routines that clear, recreate, or overwrite existing sheets.
- The Apps Script HTML service and `google.script.run` remain the reliable current integration path until a separately authenticated cross-origin API design is tested end-to-end.
