# Member management UI patch

This patch describes the UI changes for `ExpensesMonitor.html`. The monitor currently supports member creation and listing. The backend functions already exposed for administrators are:
- `adminListMembers(token)`
- `adminCreateMember(token, data)`
- `adminUpdateMember(token, data)`
- `adminResetPin(token, username, newPin)`

The current member form has been narrowed to the approved staff roles:
- `OFFICE_STAFF` → `OFFICE`
- `PERSONAL_ASSISTANT` → `PERSONAL`

## Current status
The member screen in the repository has the corrected role choices and no longer offers the unsupported PARTNER or MEMBER role values. The create-member action still calls the backend role/ledger validation.

The following controls are still pending in the screen:
1. Edit member name and active status using `adminUpdateMember`.
2. Deactivate/reactivate an account by changing the `active` field.
3. Reset a staff PIN using `adminResetPin`.

Do not treat this screen as complete for member administration until those controls have been added and tested.

## Required edit payload

When editing an existing staff member, call:

```javascript
google.script.run
  .withSuccessHandler(handleSuccess)
  .withFailureHandler(handleError)
  .adminUpdateMember(token, {
    username: selectedUsername,
    name: editedName,
    role: existingRole,
    ledgers: [existingRole === 'PERSONAL_ASSISTANT' ? 'PERSONAL' : 'OFFICE'],
    active: activeCheckbox.checked,
    visibility: 'OWN'
  });
```

Do not offer a role change in this first UI version. Role changes should be a separate explicit admin workflow because they change ledger access.

## Required PIN reset payload

Collect the new PIN in a password input, validate at least 6 characters in the UI, then call:

```javascript
google.script.run
  .withSuccessHandler(handleSuccess)
  .withFailureHandler(handleError)
  .adminResetPin(token, selectedUsername, newPin);
```

Never display or retrieve the stored PIN hash or salt. Never log a PIN to the browser console or spreadsheet.

## Required testing

Use a disposable test staff account:
1. Edit its display name and verify the member list updates.
2. Deactivate it and verify its next server request is rejected.
3. Reactivate it and verify login works.
4. Reset its PIN and verify the old PIN fails and the new PIN works.
5. Confirm the Super Admin account cannot be edited or reset through these staff controls.

This file is a development checklist, not a live deployment. The repository source has not been copied into the live Apps Script project.
