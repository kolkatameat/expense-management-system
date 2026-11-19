# Backend security patch — Expense Management System

This patch tightens server-side access rules for the existing Apps Script backend. It does not create, clear, rename, or migrate any spreadsheet tabs or Drive folders.

## Before applying
1. Open the existing Apps Script project.
2. Make a copy of the current `Code.gs` text and save it privately as a backup.
3. Do not run `setupExpenseManager()`.
4. Replace only the functions listed below with the complete versions in this file.
5. Save the project. Do not deploy until the next test checklist is completed.

The patch intentionally limits newly created non-admin accounts to the two approved staff roles:
- `PERSONAL_ASSISTANT` → `PERSONAL` only
- `OFFICE_STAFF` → `OFFICE` only

Super Admin remains the only role with access to both ledgers and all submissions. Every non-admin sees only records submitted by their own username, even if an old member row has `Visibility = ALL`.

---

## 1. Replace adminCreateMember

```javascript
function adminCreateMember(token, data) {
  const session = getSession_(token);
  requireAdmin_(session);

  data = data || {};

  const username = cleanText_(data.username, 40);
  const name = cleanText_(data.name, 100);
  const pin = String(data.pin || '');
  const role = String(data.role || '').toUpperCase();

  if (!/^[a-zA-Z0-9._-]{3,40}$/.test(username)) {
    throw new Error('Username must be 3–40 characters using letters, numbers, dot, dash or underscore.');
  }
  if (!name) throw new Error('Enter the member name.');
  if (pin.length < 6 || pin.length > 30) {
    throw new Error('Member PIN must be at least 6 characters.');
  }
  if (!['PERSONAL_ASSISTANT', 'OFFICE_STAFF'].includes(role)) {
    throw new Error('Choose Personal Assistant or Office Staff.');
  }
  if (findMember_(username)) {
    throw new Error('This username already exists.');
  }

  const ledgers = normalizeLedgers_(data.ledgers);
  validateStaffRoleLedger_(role, ledgers);

  const salt = Utilities.getUuid();
  const hash = hashPin_(pin, salt);

  getSheet_(CONFIG.SHEETS.MEMBERS).appendRow([
    username,
    name,
    hash,
    salt,
    role,
    ledgers.join(','),
    'OWN',
    true,
    new Date()
  ]);

  return { success: true, message: 'Member created.' };
}
```

## 2. Replace adminUpdateMember

```javascript
function adminUpdateMember(token, data) {
  const session = getSession_(token);
  requireAdmin_(session);

  data = data || {};
  const username = cleanText_(data.username, 40);
  const member = findMember_(username);

  if (!member) throw new Error('Member not found.');
  if (member.role === 'ADMIN') {
    throw new Error('The administrator account cannot be changed here.');
  }

  const role = String(data.role || member.role).toUpperCase();
  if (!['PERSONAL_ASSISTANT', 'OFFICE_STAFF'].includes(role)) {
    throw new Error('Choose Personal Assistant or Office Staff.');
  }

  const ledgers = data.ledgers === undefined
    ? normalizeLedgers_(member.ledgers)
    : normalizeLedgers_(data.ledgers);

  validateStaffRoleLedger_(role, ledgers);

  const name = data.name === undefined
    ? member.name
    : cleanText_(data.name, 100);

  if (!name) throw new Error('Enter the member name.');

  const active = data.active === undefined
    ? member.active
    : data.active === true;

  const sheet = getSheet_(CONFIG.SHEETS.MEMBERS);
  sheet.getRange(member.row, 2).setValue(name);
  sheet.getRange(member.row, 5).setValue(role);
  sheet.getRange(member.row, 6).setValue(ledgers.join(','));
  sheet.getRange(member.row, 7).setValue('OWN');
  sheet.getRange(member.row, 8).setValue(active);

  return { success: true, message: 'Member updated.' };
}
```

## 3. Replace normalizeVisibility_ and add validateStaffRoleLedger_

```javascript
function normalizeVisibility_(value, role) {
  // Only the ADMIN role may have ALL visibility.
  return String(role || '').toUpperCase() === 'ADMIN' ? 'ALL' : 'OWN';
}

function validateStaffRoleLedger_(role, ledgers) {
  const expected = {
    PERSONAL_ASSISTANT: 'PERSONAL',
    OFFICE_STAFF: 'OFFICE'
  }[String(role || '').toUpperCase()];

  if (!expected) {
    throw new Error('Unsupported staff role.');
  }

  if (!Array.isArray(ledgers) || ledgers.length !== 1 || ledgers[0] !== expected) {
    throw new Error(
      role === 'PERSONAL_ASSISTANT'
        ? 'A Personal Assistant can be assigned to PERSONAL only.'
        : 'Office Staff can be assigned to OFFICE only.'
    );
  }
}
```

Note: `normalizeVisibility_` is retained for compatibility with any other existing calls. Non-admin accounts must always be stored as `OWN`.

## 4. Replace getExpenses

```javascript
function getExpenses(token, filters) {
  const session = getSession_(token);
  filters = filters || {};

  const from = String(filters.from || '');
  const to = String(filters.to || '');
  const requestedType = String(filters.expenseType || 'ALL').toUpperCase();

  if (!['ALL', 'OFFICE', 'PERSONAL'].includes(requestedType)) {
    throw new Error('Invalid ledger filter.');
  }

  const rows = getSheet_(CONFIG.SHEETS.EXPENSES)
    .getDataRange().getValues();

  const results = [];

  for (let i = 1; i < rows.length; i++) {
    const r = rows[i];
    if (!r[0]) continue;

    const ledger = String(r[2] || '').toUpperCase();

    if (!['OFFICE', 'PERSONAL'].includes(ledger)) continue;
    if (!canUseLedger_(session, ledger)) continue;
    if (requestedType !== 'ALL' && ledger !== requestedType) continue;

    // Critical rule: every non-admin can see only their own submissions.
    // This ignores legacy Visibility=ALL values on non-admin member rows.
    if (!isAdmin_(session) &&
        String(r[14] || '').toLowerCase() !== String(session.username || '').toLowerCase()) {
      continue;
    }

    const dateText = normalizeDate_(r[1]);
    if (from && dateText < from) continue;
    if (to && dateText > to) continue;

    results.push({
      expenseId: String(r[0]),
      date: dateText,
      expenseType: ledger,
      category: String(r[3]),
      description: String(r[4]),
      amount: Number(r[5]) || 0,
      expenseOwner: String(r[6]),
      paidBy: String(r[7]),
      submittedBy: String(r[8]),
      upiTransactionId: String(r[9]),
      paymentMode: String(r[10]),
      hasReceipt: Boolean(r[11]),
      receiptFileName: String(r[12]),
      status: String(r[15])
    });
  }

  results.sort((a, b) => b.date.localeCompare(a.date));
  return results;
}
```

## 5. Replace getReceipt

```javascript
function getReceipt(token, expenseId) {
  const session = getSession_(token);
  expenseId = String(expenseId || '');

  const sheet = getSheet_(CONFIG.SHEETS.EXPENSES);
  const rows = sheet.getDataRange().getValues();

  for (let i = 1; i < rows.length; i++) {
    const r = rows[i];
    if (String(r[0]) !== expenseId) continue;

    const ledger = String(r[2] || '').toUpperCase();

    if (!canUseLedger_(session, ledger)) {
      throw new Error('You do not have access to this receipt.');
    }

    // Receipt access follows the same own-submission rule as expense access.
    if (!isAdmin_(session) &&
        String(r[14] || '').toLowerCase() !== String(session.username || '').toLowerCase()) {
      throw new Error('You do not have access to this receipt.');
    }

    if (!r[11]) throw new Error('No receipt is attached.');

    const file = DriveApp.getFileById(String(r[11]));
    const blob = file.getBlob();

    return {
      name: file.getName(),
      mimeType: blob.getContentType(),
      base64: Utilities.base64Encode(blob.getBytes())
    };
  }

  throw new Error('Expense or receipt not found.');
}
```

---

## Verification checklist (before deployment)

Use test accounts with dummy expenses only. Do not test by changing or deleting real expense rows.

1. Super Admin can select OFFICE and PERSONAL and can view both.
2. Personal Assistant can submit to PERSONAL, but a direct server call attempting OFFICE is rejected.
3. Office Staff can submit to OFFICE, but a direct server call attempting PERSONAL is rejected.
4. A staff account sees only rows whose **Created By Username** (column O / index 14) matches that account's username.
5. A staff account cannot retrieve another member's receipt by calling `getReceipt` with another expense ID.
6. Super Admin can still retrieve both ledgers' receipts and export both ledgers.
7. Deactivating a member blocks subsequent requests because `getSession_` rechecks the member's active status.

## Important integration note

The current member-creation screen may still offer older role choices such as MEMBER or PARTNER. After this backend patch, only Personal Assistant and Office Staff are accepted for new non-admin accounts. The member-management screen must be updated to match before creating accounts from that screen.

This is a source-code patch only. It has not been applied to the live Apps Script project, and it has not been deployed. Do not paste it into the live project until the member screen is updated or you are ready to create accounts using the exact approved role names.
