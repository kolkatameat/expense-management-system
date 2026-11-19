# Apps Script deployment notes

These HTML files are designed to be served by Apps Script, because they call the existing backend with `google.script.run`.

## Required routing change in Code.gs

Replace the existing `doGet()` function with:

```javascript
function doGet(e) {
  const app = String((e && e.parameter && e.parameter.app) || 'details').toLowerCase();
  if (app === 'monitor') {
    return HtmlService.createHtmlOutputFromFile('ExpensesMonitor')
      .setTitle('Expenses Monitor')
      .addMetaTag('viewport', 'width=device-width, initial-scale=1');
  }
  return HtmlService.createHtmlOutputFromFile('ExpenseDetails')
    .setTitle('Expense Details')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}
```

Add two HTML files to the existing Apps Script project using the exact names:
- `ExpenseDetails` (paste `ExpenseDetails.html`)
- `ExpensesMonitor` (paste `ExpensesMonitor.html`)

After deployment, the same Apps Script web app URL serves:
- Expense Details: `YOUR_WEB_APP_URL`
- Expenses Monitor: `YOUR_WEB_APP_URL?app=monitor`

## Important before deployment

1. Do not run `setupExpenseManager()` again on the populated spreadsheet.
2. Do not change or expose the existing administrator PIN or session secret.
3. Deploy as the spreadsheet owner, with access set to the intended audience. The app's own username/PIN authorization remains mandatory.
4. The Monitor page rejects non-admin accounts in its interface, and the backend's admin functions also enforce administrator permission.
5. The Details page uses the existing server-side ledger and own-submission checks. Review and strengthen backend role/visibility validation before production.
6. The receipt file is limited by the current backend to JPG, PNG, WEBP or PDF, up to 5 MB.
7. This first Details build accepts pasted share text and manual receipt selection. It does not yet implement the Android Web Share Target or iPhone Shortcut handoff, nor screenshot OCR. Those require a separate integration and device testing phase.

This is a source-code addition only. It does not deploy or modify the live Apps Script project.
