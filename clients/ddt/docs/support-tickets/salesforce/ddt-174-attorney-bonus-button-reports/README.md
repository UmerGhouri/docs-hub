# Attorney bonus button and per-attorney bonus reports

| | |
|---|---|
| **Ticket** | DDT-174, DDT-224 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-24 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-16 and 2026-06-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Quick Action, Lightning record page, Reports) |
| **Area** | Attorney bonuses: requesting a bonus on a case and tracking it |
| **Related** | None |
| **Code** | Not applicable: built in Setup and the Report Builder |

---

## 1. Summary

Attorneys could request a bonus from a Criminal Defense Court Case with an **Attorney Bonus** button, but not from a PI case. DDT-174 added the same button to the PI Matter page, so PI bonuses land in the same `Bonus_Amount__c` records and the same **All Unpaid** list. DDT-224 then gave each attorney a report of **their own** bonus requests, grouped by payment status, with the notes visible.

## 2. What was reported

- **DDT-174, 2026-06-16.** Add the Attorney Bonus button to all PI cases, cloning the Criminal Defense setup, so PI bonuses show up in the existing All Unpaid view.
- **DDT-224, 2026-06-24.** A staff member asked whether per-attorney bonus reports were possible: one attorney's bonuses, grouped by status (Requested, Paid and so on), with the notes column showing why a bonus is still waiting (for example, the client still has a balance).

Both tickets were closed within minutes of being created, so the work was logged after it was done.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter__c.Attorney_Bonus` | Quick Action on the PI Matter | Added (DDT-174) |
| `Matter_Record_Page` | Lightning record page | Attorney Bonus action added with a visibility rule (DDT-174) |
| `Bonus_Amount__c` All Unpaid list view | List view | Matter column added (DDT-174) |
| Attorney Bonus Reports | Report folder | New, with two summary reports (DDT-224) |

**Out of scope**

- **The Criminal Defense button** (`Opportunity.Attorney_Bonus`) was the model and was not changed.
- **Dashboards.** The DDT-224 ticket listed a per-attorney dashboard as optional; none is recorded.

## 4. How it works (background)

```
Court Case (Opportunity) or PI Matter
  → Attorney Bonus quick action
  → new Bonus_Amount__c, Payment Status = Requested
  → Bonus Amount Handler flow (record-triggered, after save) keeps the Court Case's
    bonuses-paid flag in step as statuses change
  → All Unpaid list view: every bonus whose status is not Paid or Denied
```

`Bonus_Amount__c` holds the amount (`Bonus_Amount__c`, currency), a `Details__c` text field, a `Court_Case__c` lookup to the Opportunity, a notes field, and `Payment_Status__c` with the values Requested, an approval value, Paid and Denied. On 2026-11-05 production held 19 Requested, 1,360 Paid and 12 Denied bonuses.

## 5. Root cause

Not a defect. PI cases live on `Matter__c`, while the original button was built for Court Cases (`Opportunity`), so the PI page had no way to create a bonus.

## 6. Solution

### 6.1 The button on PI Matters (DDT-174)

The Criminal Defense action shows the shape that was cloned:

```xml
<QuickAction>
    <fieldOverrides>
        <field>Payment_Status__c</field>
        <literalValue>Requested</literalValue>
    </fieldOverrides>
    <label>Attorney Bonus</label>
    <quickActionLayout>...Details__c and Bonus_Amount__c, both Required...</quickActionLayout>
    <targetObject>Bonus_Amount__c</targetObject>
    <targetParentField>Court_Case__c</targetParentField>
    <type>Create</type>
</QuickAction>
```

The PI version, `Matter__c.Attorney_Bonus`, sits on `Matter_Record_Page` with a visibility rule: it shows only when the Matter's `Hire_Status__c` is **Signed Pending Approval, Retained Active or Retained Settled**, and the user's title contains **Attorney**. The All Unpaid list view gained a Matter column so PI bonuses are identifiable.

Why it is done this way:

- **A quick action, not a custom component.** It is a clone of something already working, with the status pre-set to Requested so the request enters the existing approval path.
- **Shown only on hired Matters.** A bonus on a case that never hired makes no sense.
- **The same object and list view.** PI and Criminal Defense bonuses are paid by the same process, so they share one list.

### 6.2 Per-attorney reports (DDT-224)

A report folder, **Attorney Bonus Reports**, was created on 2026-06-23 with two summary reports on `Bonus_Amount__c`. Their saved descriptions read: each attorney's own bonus requests, **grouped by Payment Status** (Requested, approved, Paid, Denied), **scoped to "My Bonus Amounts"** so each attorney sees only their own, with the Details and notes columns.

- **My Bonus Requests**: the single shared report (the ticket's Option A). Because it is scoped to "my" records, one report serves every attorney.
- **One named per-attorney report**, with the same grouping and columns.

Why "My" scope rather than a filter on an attorney field: the ticket asked for each attorney to see only their own bonuses without filtering by hand each time, and a report scoped to the running user does that with one report instead of one per attorney.

## 7. Testing

Not recorded on either ticket. The production state above (the action on the Matter page, the report folder and its two reports) was read on 2026-11-05.

**Not covered:** whether "My Bonus Amounts" matches the attorney who earned the bonus or the user who created the record. If someone requests a bonus on an attorney's behalf, check which one the report follows.

## 8. Step by step

**The button**

1. **Look at the existing action**: Setup → Object Manager → Opportunity → Buttons, Links, and Actions → Attorney Bonus. Note the target object, parent field, required fields and the Requested default.
2. **Create the same action on `Matter__c`**: Object Manager → Matter → Buttons, Links, and Actions → New Action, type Create a Record, target `Bonus_Amount__c`, same fields and default.
3. **Add it to the Matter Lightning page** in App Builder (edit production's current page, do not deploy a sandbox copy) and set the visibility rule: hire status in the three hired values, and user title contains Attorney.
4. **Add a Matter column** to the All Unpaid list view.
5. **Test on a hired PI Matter as an attorney**: create a bonus and check it appears in All Unpaid with status Requested.

**The reports**

6. **Create a report folder** (Attorney Bonus Reports) and share it with the attorneys.
7. **New report on Bonus Amounts**, scope "My Bonus Amounts", format Summary, grouped by Payment Status.
8. **Add columns**: Court Case, Matter, Bonus Amount, Details, the notes field, created date, payment status.
9. **Run it as an attorney** and check only that attorney's bonuses appear, with notes readable.

## 9. Rollback

Remove the action from `Matter_Record_Page` in App Builder, and delete the report folder's reports. Neither changes any bonus records.

## 10. Known limits

- **The PI action and its Matter lookup are not in the ddt-salesforce repo.** The record page and list view that reference them are, so a deploy of those from git to an org without the action or field would fail. Retrieve them before relying on the repo.
- **The visibility rule depends on the user's title.** A user whose title does not contain "Attorney" will not see the button, whatever their role.
- **One named per-attorney report exists.** Named reports need a new copy for every new attorney; the "My" report does not.

## 11. Reuse for similar issues

**How to recognise one:** a request to "add the same button as on the other case type", or to give each user a view of only their own records.

- Read the existing action's metadata before cloning, so the copy matches field for field.
- Put a visibility rule on the action rather than a separate page per audience.
- For "each person sees their own", prefer one report scoped to "My" records over one report per person.
- Retrieve anything built in Setup into git the same day, so the repo does not reference components it does not hold.
