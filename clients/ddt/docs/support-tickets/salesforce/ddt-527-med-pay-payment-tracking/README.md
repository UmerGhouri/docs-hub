# Med Pay payment tracking fields on Coverage

| | |
|---|---|
| **Ticket** | DDT-527, DDT-740, DDT-1124 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, picklist, LWC, record-triggered flow, permission set, Lightning record page) |
| **Area** | PI coverage: tracking Med Pay money from request to check |
| **Related** | DDT-528 (Coverage record page and master report), DDT-625 (Med Pay paid amount added to the report type) |
| **Code** | PR #177 and PR #512 in the ddt-salesforce repo. The DDT-527 field was built directly in production and recorded in git by PR #177. |

---

## 1. Summary

Three requests from the PI team, built one on top of the other, to track Med Pay money on an auto policy:

1. **DDT-527:** a currency field for how much Med Pay has been paid out.
2. **DDT-740:** two new Med Pay status values, **Payment Requested** and **Check Received**, for the early check the team can ask for once a few bills are in.
3. **DDT-1124:** a date for each of those two statuses, filled in automatically when the status changes.

All three live on the **Coverage** object (`Coverage__c`), in its Med Pay section.

## 2. What was reported

- **DDT-527, 2026-07-27.** The Med Pay section recorded the limit, claim number, LOR dates, claim status and adjuster, but had nowhere for the amount actually paid. The team asked for a "MedPay Paid" currency field that saves exactly what is typed. Live 2026-07-29, and the requester confirmed it worked.
- **DDT-740, 2026-08-14.** Med Pay — Status only offered Confirmed, Not Available and Unknown, which describe whether coverage exists, not where an early payment request stands. The team asked for Payment Requested and Check Received. Live 2026-08-17.
- **DDT-1124, 2026-09-25.** The team wanted to know when a payment was requested and when the check arrived, to chase stale requests. Asked whether the dates should be typed or automatic, the requester answered "auto please". Live 2026-09-28.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `Coverage__c.Med_Pay_Paid__c` | Currency field (16, 2), label "Med Pay — Paid" | DDT-527 | New |
| `CoveragePanelController` | Apex class | DDT-527 | Reads the paid amount and shows it on the Med Pay line |
| `coveragePanel` | LWC | DDT-527, DDT-740 | Paid amount input and display; Med Pay gets its own five-value status list |
| `Coverage__c.Med_Pay_Status__c` | Picklist field | DDT-740 | Two new values |
| `Coverage__c.Med_Pay_Payment_Requested_Date__c` | Date field, "Med Pay — Payment Requested Date" | DDT-1124 | New |
| `Coverage__c.Med_Pay_Check_Received_Date__c` | Date field, "Med Pay — Check Received Date" | DDT-1124 | New |
| `Coverage_Med_Pay_Stamp_Status_Dates` | Before-save record-triggered flow | DDT-1124 | New, stamps the two dates |
| `Coverage_Admin` | Permission set | DDT-527, DDT-1124 | Read and edit on the new fields |
| `Coverage_Record_Page` | Lightning record page | DDT-1124 | Both dates directly under Med Pay — Status |

**Out of scope**

- **A field on the Matter.** DDT-527 asked for the field on the PI Matter's Coverage tab. That tab shows Coverage records, so the field was built on `Coverage__c`, named to match the other Med Pay fields.
- **Backfilling the dates.** Four Coverage records already had Payment Requested (2) or Check Received (2) when DDT-1124 shipped. Their real dates aren't recorded anywhere, so a backfill would invent them. Staff can type them in.
- **Showing the dates in the `coveragePanel` wizard.** The wizard on the Matter doesn't display them. The flow still stamps them when the status is changed there.
- **The classic Coverage page layout.** The Lightning record page doesn't use it.

## 4. How it works (background)

```
PI Matter
  -> Coverage tab (coveragePanel LWC) lists the matter's Coverage__c records
       -> each auto policy has lines: Liability BI, Liability PD, UM/UIM, Med Pay, Umbrella
       -> the "Add/Edit Coverage" wizard edits those lines
  -> opening a Coverage record shows the Coverage record page (Dynamic Forms)
```

Two things in this design shaped the changes:

- **The wizard does not read picklist values from Salesforce.** It builds its status dropdown from a list written into the LWC's JavaScript, shared by all five auto lines.
- **A blank combobox hides a real value.** `lightning-combobox` shows nothing when the saved value isn't in its options list. The value is still saved; it just looks empty.

## 5. Root cause

Not applicable: these were feature requests. One finding is worth knowing, though.

**DDT-527 was built directly in production with no record in git.** When DDT-740 was prepared, the support team retrieved production's version of every component it touched, as the deploy rules require. That showed a complete feature in production that git did not have: the `Med_Pay_Paid__c` field, about 25 lines in `CoveragePanelController`, and the paid-amount input and display in `coveragePanel`. **Had DDT-740 been deployed from git as it stood, it would have wiped the DDT-527 feature from production.** A validation-only deploy doesn't catch this: it proves a deploy is valid, not that it won't overwrite newer code.

## 6. Solution

### 6.1 Med Pay — Paid (DDT-527)

- **Field:** `Med_Pay_Paid__c`, Currency, 16 digits with 2 decimals. Help text: "How much Med Pay the carrier has actually paid so far. Enter the exact amount including cents — it is stored as typed."
- **Where it shows:** in the wizard's Med Pay section directly under Med Pay — Limit, and on the Coverage record page. Once saved, the amount also appears on the Med Pay row under the limit, for example "paid $1,234.56".
- **`applyMedPayPaid`** in `CoveragePanelController` builds a Med Pay line even when the paid amount is the only thing filled in, so a payment is never hidden just because the rest of the line is empty.

### 6.2 Two new status values (DDT-740)

The value set is unsorted, so the order in the metadata is the order staff see. The new values go in workflow order rather than at the end:

```
Confirmed, Payment Requested, Check Received, Not Available, Unknown
```

`Unknown` stays the default and the existing values keep their API names, so no existing record changes.

The `coveragePanel` wizard also needed changing, which the ticket didn't mention. Without it, the wizard couldn't offer the new values, and a Med Pay line already set to one of them would show a blank status (see section 4). Med Pay now gets its own `MED_PAY_STATUS_OPTIONS` list, chosen by `line.prefix === 'Med_Pay'`. The other four lines keep the original three values, because only `Med_Pay_Status__c` gained new ones. The help text that listed the three old values is now `{line.statusHelp}`, so each list documents its own values.

No record type or page layout change was needed: `Coverage__c` has no record types, and the record page reads picklist values from the field.

### 6.3 Automatic dates (DDT-1124)

A **before-save** flow, `Coverage_Med_Pay_Stamp_Status_Dates`, runs on create and update of `Coverage__c` when Med Pay — Status is Payment Requested or Check Received. It writes today's date (`$Flow.CurrentDate`) into the matching date when both are true:

- the record is new, or the status changed in this save; **and**
- the user did not enter the date in the same save.

So:

| What the user does | Result |
|---|---|
| Sets Payment Requested, leaves the date blank | Payment Requested Date = today |
| Sets the status and types an earlier date | The typed date is kept |
| Moves from Payment Requested to Check Received | Check Received Date = today; Payment Requested Date kept |
| Re-requests (Payment Requested, then something else, then Payment Requested) | Payment Requested Date re-stamped with the latest date |
| Edits anything else | Neither date touched |

Why a before-save flow: it changes the record being saved with no extra DML and can't loop. It also runs on every save path, so a status change made in the `coveragePanel` wizard stamps the date even though the wizard doesn't show it. There was no other automation on `Coverage__c` at the time.

Both dates are always visible on the record page, not hidden by status. Once a record moves on to Check Received, the Payment Requested date is still the history staff need.

**Field-level security.** In production, `Coverage_Admin` is the only non-managed permission set granting the Med Pay fields, so the new fields were granted there and no profile was touched.

## 7. Testing

- **DDT-740:** a dry-run deploy against production succeeded. The LWC test suite passed (17 suites, 137 tests), but no suite covers `coveragePanel`, so that only shows nothing else broke. The post-deploy checks handed to the requester were: all five values offered and saving on the Coverage record; five on the Med Pay line in the wizard and three on the other lines; a Med Pay line set to Payment Requested reads back correctly; the paid amount still shows under the limit.
- **DDT-1124:** deployed to the partial sandbox and run through nine scenarios in anonymous Apex, rolled back afterwards: insert with a blank date, insert with a typed date, insert as Unknown, Unknown to Payment Requested, editing only the date, an unrelated edit, Payment Requested to Check Received, a status change with a typed date, and a re-request. All passed, including a 200-record insert where all 200 were stamped. A check-only deploy to production also succeeded.
- **Live check:** the requester confirmed DDT-527 on 2026-07-29. For DDT-1124 the support team confirmed the dates were live and filling in automatically.
- **Not covered:** no Apex or LWC test covers the `coveragePanel` changes. `Med_Pay_Status__c` is hidden by field-level security from the deploy user, so value counts couldn't be read from a terminal.

## 8. Step by step

1. **Retrieve production's version of every component you will touch**, one at a time, and diff each against git. If production has something git lacks, record it in git first, in its own commit, before making your change.
2. **Add the field or values.** Put new picklist values in the order staff use them, keep the default, and keep existing API names.
3. **Search the LWCs for hardcoded copies** of the picklist: `grep -rn "Confirmed" force-app/main/default/lwc/coveragePanel`. Update any list that shows the field.
4. **Grant field-level security** on new fields through `Coverage_Admin`, the same as the other Med Pay fields.
5. **Add new fields to `Coverage_Record_Page`.** Retrieve production's page first, add the fields, and deploy that.
6. **For automatic dates, build a before-save flow** with the "status changed and date not supplied" rule from 6.3, and test each row of the table there.
7. **Deploy, then activate the flow.** Flows reach production as Draft; deploy the `FlowDefinition` with the active version separately, and confirm it is active.
8. **Ask the requester to check** on a real Coverage record.

## 9. Rollback

- **Automatic dates:** deploy the `FlowDefinition` with active version 0 to stop stamping. The fields can stay; they hold no derived data.
- **Status values:** removing a picklist value once staff have used it means replacing it on every record that holds it. Do it while usage is still zero, or not at all. Reverting the `coveragePanel` change alone is safe; it only narrows the Med Pay list back to three.
- **Med Pay — Paid:** do not revert the commit that recorded it in git. It is the only record of that feature.

## 10. Known limits

- **Two lists must stay in step.** A value added to `Med_Pay_Status__c` that is missing from `MED_PAY_STATUS_OPTIONS` shows as a blank status in the wizard. There is a comment in the file saying so.
- **The dates use the date of the save**, not the date the request was actually sent. If staff record a status a few days late, they need to correct the date by hand.
- **Records before 2026-09-28 have no dates** unless someone types them in.
- **Field-level security.** Only users with `Coverage_Admin` see these fields. Anyone verifying from a terminal under another user gets "Invalid field".

## 11. Reuse for similar issues

**How to recognise one:** a request to add a field or a picklist value to Coverage, or to any object edited through a custom LWC.

- **Check production before touching anything.** Here a whole feature was live with no git record, and a routine deploy would have deleted it.
- **Look for hardcoded picklist copies in LWCs.** A record page picks up new values automatically; a custom component often doesn't.
- **Stamp dates with a before-save flow** that only stamps when the status changed and the user didn't supply a date. It respects a typed date and doesn't fire on unrelated edits.
- **Don't backfill dates you don't know.** A blank date is honest; an invented one ends up in a demand letter.
