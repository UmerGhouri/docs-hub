# PI expense amount rounding and invoice file field

| | |
|---|---|
| **Ticket** | DDT-250 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-01 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom fields) |
| **Area** | PI matters: Financials → Expenses |
| **Related** | None recorded |
| **Code** | Commit `8def8fcf0` in the ddt-salesforce repo (merged as `5909a6af3`, no numbered PR); restored by commit `68e83bcd2` |

---

## 1. Summary

On a PI Matter's Expenses list, an amount like `15.30` saved as `15.00`. The `Expense__c.Dollar_Amount__c` currency field had **0 decimal places**, so Salesforce rounded every amount to whole dollars. It was changed to 2 decimal places. The same ticket asked for a way to attach the invoice to each expense, which was added as an **Attach / View invoice** action that takes either an uploaded file or a file picked from the case's Google Drive folder.

## 2. What was reported

- **2026-06-30:** entering `15.30` in an expense's Amount saved and displayed `15.00`. The requester also needed to attach the source invoice to each expense.
- **2026-07-01:** asked how the invoice attachment should behave, the requester confirmed it should work **the same way as the Damages section**, where files can be uploaded and viewed in place.

Every expense with cents was affected. The ticket gives no count.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Expense__c.Dollar_Amount__c` | Currency field | Scale 0 → 2 |
| `Expense__c.Invoice_PDF_Id__c` | Text(18) field | New. The invoice when it is a Salesforce file |
| `Expense__c.Invoice_Drive_File_Id__c` | Text field | New. The invoice when it is a Google Drive file |
| `Expense__c.Invoice_Drive_File_Name__c` | Text field | New. The Drive file's name, for display |
| `ExpenseInvoiceController` | Apex class | New. Link, link-from-Drive and clear the invoice |
| `ExpenseInvoiceTest` | Apex test | New. 5 tests |
| `ExpenseManagerController` | Apex class | Returns each expense's invoice with the row |
| `expenseManager` | LWC | Invoice column (Attach / View / Change) and an Upload or Browse Drive dialog |
| `Matter_Access` | Permission set | FLS for the 3 new fields. Applied directly in production, not in the commit |

**Out of scope**

- **Existing amounts.** Changing the scale does not bring back cents already lost. Amounts saved before the fix stay rounded.
- **Deleting the underlying file.** Clearing an invoice only empties the slot.

## 4. How it works (background)

```
PI Matter page → Financials → Expenses (expenseManager LWC)
  -> ExpenseManagerController.getExpensesForMatter(matterId) returns rows + totals
  -> user edits Amount → saved to Expense__c.Dollar_Amount__c
```

A Salesforce Currency field has a **scale**: the number of decimal places stored. With scale 0, Salesforce rounds the value on save. The field kept the cents in the input box and dropped them on save.

## 5. Root cause

`Dollar_Amount__c` was defined as `Currency` with `scale` = **0**:

```xml
<precision>18</precision>
<scale>0</scale>
<type>Currency</type>
```

So every amount was stored as a whole number. There was no invoice field at all; that part was a new request.

## 6. Solution

### 6.1 Two decimal places

```xml
<scale>2</scale>
```

The field type stays `Currency`, so totals and anything that reads it keep working; they now see the cents.

### 6.2 An invoice slot on each expense

The invoice can be **either** a Salesforce file **or** a Google Drive file, never both. Two sets of fields hold it, and each action clears the other set:

| Action | Sets | Clears |
|---|---|---|
| `linkExpenseInvoice(expenseId, contentDocumentId)` | `Invoice_PDF_Id__c`, and links the file to the expense if not already linked | Drive fields |
| `linkExpenseInvoiceFromDrive(expenseId, driveFileId, driveFileName)` | `Invoice_Drive_File_Id__c`, `Invoice_Drive_File_Name__c` | `Invoice_PDF_Id__c` |
| `clearExpenseInvoice(expenseId)` | nothing | all three; the file itself is not deleted |

Why it's done this way:

- **It mirrors the Damages attach pattern.** That is what the requester asked for, and staff already know it.
- **A Drive option**, because case documents already live in the case's Drive folder. Picking an existing file avoids a second copy.
- **Older expenses still show an invoice.** When neither slot is set, `ExpenseManagerController` falls back to a file already attached to the expense in Salesforce Files. An explicit slot always wins, and Drive wins over a Salesforce file.

### 6.3 The screen

The Expenses table gained an **Invoice** column: **Attach** when empty, **View** and **Change** when set. Attach opens a dialog with **Upload** and **Browse Drive**.

## 7. Testing

`ExpenseInvoiceTest`:

| Test | Proves |
|---|---|
| `linkExpenseInvoice_stampsIdAndLink` | Uploading stamps the file id and links the file to the expense |
| `linkExpenseInvoiceFromDrive_stampsDriveFields` | Picking from Drive stamps the Drive fields |
| `clearExpenseInvoice_clearsSlots` | Clearing empties all three fields |
| `legacyLooseFile_surfacesAsInvoice` | An older expense with a file attached but no slot set still shows an invoice |
| `nullIds_throw` | Missing ids give a clear error |

The ticket does not record a run result. Live check: the requester-facing reply on 2026-07-01 confirmed the amount saves with 2 decimals and the invoice action is live.

**Not covered:** no test checks the scale of `Dollar_Amount__c` itself.

## 8. Step by step

1. **Confirm the field's scale.** Setup → Object Manager → Expense → Fields → Dollar Amount. Decimal places 0 is the bug.
2. **Change the scale to 2** in the field metadata and deploy. Check reports, formulas and roll-ups that read the field; the type does not change, so they keep working.
3. **Add the three invoice fields** and grant FLS on them in the permission set expense users already hold, **before** releasing the screen change.
4. **Deploy `ExpenseInvoiceController`, its test, the `ExpenseManagerController` change and the `expenseManager` LWC.** Validate against production with `ExpenseInvoiceTest`.
5. **Test on a real expense:** enter `15.30`, save, confirm it shows `15.30`. Attach a file by upload, then change it to a Drive file, then clear it.
6. **Commit any production-only change** (such as the permission set FLS) to git the same day.

## 9. Rollback

- **Scale:** setting it back to 0 would round all amounts again. Don't.
- **Invoice feature:** redeploy the previous `expenseManager` and `ExpenseManagerController`. The three fields can stay; they are unused without the screen.

## 10. Known limits

- **Amounts entered before the fix stay rounded.** The cents were never stored.
- **One invoice per expense.** The slot holds a single file.
- **Drift happened here.** On 2026-07-01 a direct deploy from a stale workspace overwrote related code in production, and a 2026-07-03 backfill copied that state into git. Commit `68e83bcd2` (2026-07-08) restored the expense-invoice cluster so `expenseManager` imports the invoice actions from `ExpenseInvoiceController` again. If this feature ever looks half-present, compare production to git first.
- **The FLS lives only in production** at the time of the commit. It was left out to avoid reformatting the whole permission set.

## 11. Reuse for similar issues

**How to recognise one:** a number field "rounds" what users type.

- **Check the field's scale (decimal places) first.** It is the usual cause, and it is a one-line metadata change.
- **Tell the requester that old values stay rounded.** Rounded data cannot be recovered from the field.
- **When adding an attachment slot, copy an existing pattern** staff already use, and decide up front whether it is one file or many.
- **Ship FLS for new fields together with the screen** that reads them, or the screen fails for users without it.
