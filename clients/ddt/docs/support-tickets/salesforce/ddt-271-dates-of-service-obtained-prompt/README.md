# Dates of service on the medical damages Obtained prompt

| | |
|---|---|
| **Ticket** | DDT-271 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-24 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | PI damages: logging billed and outstanding amounts once medical records arrive |
| **Related** | DDT-724 (Salesforce dates shown a day early in the same component), DDT-1024 (treatment dates made required and editable on this step) |
| **Code** | PR #227 in the ddt-salesforce repo |

---

## 1. Summary

On the Damages panel, Billed Amount and Outstanding Balance are typed on the **Obtained** tab, but the treatment dates those amounts cover were only visible on the **Treatment** tab. Staff had to navigate back to check the service window before typing an amount. PR #227 added the three date fields, read-only, directly above Billed Amount in the Obtained edit form.

The ticket said the trigger was `Status = 'Obtained'`. **No such value exists.** "Obtained" is a tab label, not a picklist value, and building against it would have produced a change that never fired.

## 2. What was reported

On 2026-07-03 a staff member asked for the dates of service to be shown on the "Obtained" prompt for medical damages, next to Billed Amount and Outstanding Balance, so they could be checked without going back to the treatment record. The request was display-only, with no change to how amounts save.

Measured in production on 2026-08-21, when the fix was built:

```
Medical damages                        769
  First Treatment Date empty           233   (30%)
  Last Treatment Date empty            413   (54%)

Already in Mode 3 - records in hand    271
  First Treatment Date empty            30   (11%)
  Last Treatment Date empty             51   (19%)
```

Those 51 rows are where the firm already held the records but the date range was never captured, which is exactly the screen where the dates couldn't be seen.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `damagesTable` | LWC (HTML) | Three `lightning-output-field` elements above `Billed_Amount__c` in the Mode 3 edit form |
| `damagesTable.test.js` | Jest tests | 6 new tests |

**Out of scope**

- **Editing the dates on this tab.** The spec was display-only. Editing came later, in DDT-1024.
- **Apex and permissions.** The fields come through the `lightning-record-edit-form` already wrapping the block, and all three were already in the `Damages_Records_Mode` permission set.
- **Five other requests in the same thread:** a date range on the Requesting tab for the records-request letter, an "Invoice Paid" status, amounts on the Treatment tab, the other three damage types, and editing the dates here. Each needs its own ticket.

## 4. How it works (background)

There is one object, `Damages__c`, for every damage type. A Medical row moves through three tabs, or modes, on the Damages panel:

```
Mode 1  Treatment   -> treatment dates are entered here
Mode 2  Requesting  -> records / bills requested from the provider
Mode 3  Obtained    -> Billed Amount and Outstanding Balance entered here
```

Which mode a row is in comes from `Records_Status__c`:

```
Records_Status__c   Not Needed · Ready to Request · Requested ·
                    Received Bills · Received Records · Received Both
Status__c           Open · Documented · Resolved · Excluded
```

## 5. Root cause

The ticket described a real gap but named the trigger wrongly.

- **The object** was "API name TBD" in the ticket. It is `Damages__c`; there is no separate medical damages object.
- **"Obtained" is not a status.** It appears on neither picklist and on no record in production. The ticket recorded the user's word as an API value. The word is real: it is the Mode 3 tab label and the heading of the "Mark Records Obtained" modal, which is what the user read off the screen.
- **The real moment** is `Records_Status__c` reaching `Received Bills`, `Received Records` or `Received Both`, which puts the row in Mode 3 where the amounts are entered.

## 6. Solution

Three read-only fields, directly above `Billed_Amount__c`, inside the Mode 3 branch of the edit form:

```html
<lightning-output-field field-name="Service_Date__c"></lightning-output-field>
<lightning-output-field field-name="First_Treatment_Date__c"></lightning-output-field>
<lightning-output-field field-name="Last_Treatment_Date__c"></lightning-output-field>
<lightning-input-field field-name="Billed_Amount__c"></lightning-input-field>
```

Why `lightning-output-field`:

- **It is read-only**, with no edit control.
- **It formats the date itself.** A Salesforce Date reaches JavaScript as a bare `2026-08-10`, which `new Date()` reads as UTC and shows a day early in Central time. That is DDT-724, and it happened in this same component. Letting the base component format the value keeps JavaScript away from the date.
- **It shows an empty field rather than hiding the row**, so "no service date recorded" is visible when someone is about to type an amount. Per the numbers in section 2, empty is common.

**One deliberate widening.** The spec named `Received Both` and `Received Bills`. `Received Records` is in the same mode and shows the same two amount inputs, so it gets the dates too. The rule is "wherever the amounts are", which can't drift out of step the way a separate status list could.

## 7. Testing

Six Jest tests in `damagesTable.test.js` (12 in the file, all passing):

| Test | Proves |
|---|---|
| All three render when records are in | The fields are there |
| They render when all three are empty | The row doesn't collapse (the 51-row case) |
| They sit above `Billed_Amount__c` | The ordering, which is the point |
| Absent on the Treatment tab | No read-only copy beside the editable one |
| Absent on the Requesting tab | No amount there to mis-key |
| Present for `Received Bills` | Mode 3 statuses other than `Received Both` get the dates too |

**They fail without the fix:** reverting the template leaves 4 of the 6 failing. The other two are negative checks that pass either way.

**A bug the tests caught in themselves.** The first version read `getAttribute('field-name')`. In Jest that is a property on a base component, not an attribute, so it returned `null` and every assertion compared `null` with `null` and passed. Changed to `el.fieldName`.

**Live:** the change was announced to the requester on 2026-08-21. When it didn't appear for them, support pointed to the browser cache and asked for a hard refresh. The requester's final confirmation is not recorded; the ticket was closed on 2026-08-24.

**Not covered:** Jest doesn't render the real base component, so the date formatting itself is Salesforce's behaviour, not tested here.

## 8. Step by step

1. **Find the real field values** behind the words in the ticket. Query the picklist values and record counts; a value with no records is the warning sign.
2. **Find the edit form** in `damagesTable.html` for Mode 3 (`editIsMode3`), where `Billed_Amount__c` is.
3. **Add the three `lightning-output-field` elements** directly above `Billed_Amount__c`.
4. **Add Jest tests** for presence, empty values, order, and absence on the other tabs. Read `el.fieldName`, not `getAttribute`.
5. **Confirm the tests fail** with the template change reverted.
6. **Compare the bundle with production** before deploying. `damagesTable` deploys as a whole bundle.
7. **Deploy the bundle**, then open a Medical row on `Received Both`, click Edit, and check the three dates sit above Billed Amount. Ask users to hard-refresh if they don't see it.

## 9. Rollback

Revert the bundle. The three fields disappear; nothing else in the component reads them.

## 10. Known limits

- **`Service_Date__c` is almost always blank.** It was populated on 1 of 769 Medical rows, because the Damages panel has no input for it; only the record page does.
- **Read-only.** Empty dates were visible but still needed a trip to the Treatment tab to fill. DDT-1024 later made them editable and required on this step.
- **Medical only in practice.** Other damage types use different amount fields (`Amount__c`, `Documented_Amount__c`).
- **`Status__c` looks unused.** In production: Open 771, blank 13, Excluded 1, Documented 0, Resolved 0. Not part of this ticket.

## 11. Reuse for similar issues

**How to recognise one:** a ticket that names a status, stage or value in the user's own words.

- **Check the words against the picklist and the data** before building. A trigger on a value that doesn't exist looks correct in review and never fires.
- **Look for the word on screen.** Users read tab labels and headings; the label tells you where they were.
- **Show Salesforce Dates with base components** (`lightning-output-field`, `lightning-formatted-date-time`) or the repo's date helper, never `new Date()` on a Date field.
- **Show empty values on purpose** when the gap is the information.
- **Prove a Jest test can fail** by reverting the change, and read base-component properties, not attributes.
