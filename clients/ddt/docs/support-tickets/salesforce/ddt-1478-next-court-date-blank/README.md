# Next Court Date blank on Opportunity

| | |
|---|---|
| **Ticket** | DDT-1478 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-30 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record pages, page layout, field help text) |
| **Area** | Court cases: Next Court Date/Time on Hired court cases |
| **Related** | DDT-1301 (Next Court Date worked out from the appearances), DDT-1376 (same field, closed matters), DDT-1318 (court-appearance banner flags) |
| **Code** | PR #861 in the `ddt-salesforce` repo |

---

## 1. Summary

Next Court Date/Time was blank on a Hired court case that had a hearing on file, and a value typed into it would not save. Nothing was broken: since DDT-1301 the field is calculated from the court appearances, as the nearest hearing today or later with **no disposition**. The hearing on file had passed without a disposition being entered, so the date correctly went blank, and any typed value was replaced in the same save. The real defect was the page offering an edit pencil on a field whose edits can never stick. The field is now read-only on the Hired pages and layout, with help text saying where the value comes from.

## 2. What was reported

On 2026-10-29 a staff member reported that on one court case, Next Court Date/Time (`Next_Court_Date_Time__c`) was blank although the Appearances tab showed a 10/28 hearing with no disposition. The court banner said no future appearance was scheduled and that appearances needed a disposition. Typing a value into the field on the Details tab did not save. The same day they named a second case with the same symptom and wondered if a missing attorney disposition was the cause.

On 2026-10-30, **9 of 810** open matters were blank for the same reason.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity_Record_BB_Default_Hired` | Lightning record page | `Next_Court_Date_Time__c` `uiBehavior` from `none` to `readonly` |
| `Opportunity_Hired_BB_Mobile_Only` | Lightning record page | Same |
| `Opportunity-Hired Layout` | Page layout | `Next_Court_Date_Time__c` `behavior` from `Edit` to `Readonly` |
| `Opportunity.Next_Court_Date_Time__c` | Custom field | Adds help text explaining where the value comes from |

**Out of scope**

- **The calculation rule.** Unchanged; that was decided on the ticket.
- **The other Opportunity record pages.** Three of them do not show this field. One shows `Next_Court_Date__c`, the Date copy the same service writes, as editable; a typed value there is reverted by the nightly sweep. Same defect on a slower clock, left for its own ticket.
- **The Court Appearance pages.** They show `Court_Appearance__c.Next_Court_Date_Time__c`, a different field that staff do enter on an appearance.
- **No Apex, permission set, FLS or data change.**

## 4. How it works (background)

Since DDT-1301 (live 2026-10-14), `NextCourtDateService` keeps `Opportunity.Next_Court_Date_Time__c` on the nearest hearing:

```
Next_Court_Date_Time__c = the nearest Court_Appearance__c with
                          no disposition AND date today or later (Chicago time),
                          blank if there is none            -- open matters only

Court Appearance created / dated / dispositioned / deleted -> recalculated
Opportunity saved with the field changed by anything else  -> recalculated in the same save
Nightly sweep at 00:17 Chicago                              -> rechecks every open matter
```

## 5. Root cause

The ticket guessed that automation was clearing or overwriting the field, possibly the DDT-1376 change. Automation was setting it, but correctly.

1. Both named cases had a 10/28 hearing with no disposition.
2. The nightly sweep just after midnight on 10/29 ran with no errors. 10/28 was no longer "today or later", and there was no later undisposed hearing, so the date went blank. That is the rule from DDT-1301.
3. A value typed into the field is replaced by the before-update trigger in the same save, so it never sticks.
4. The page still showed an edit pencil on the field, inviting an edit that could never work.

The first case was dispositioned on 10/29 (continued to a 2027 date) and then showed that date. The banner messages were correct: the case did need a disposition.

**Evidence** (2026-10-30): 9 open matters blank because a past hearing has no disposition.

```sql
SELECT COUNT() FROM Opportunity
WHERE Case_Status__c = 'Open' AND Next_Court_Date_Time__c = null
  AND Id IN (SELECT Opportunity__c FROM Court_Appearance__c
             WHERE Disposition__c = null AND Court_Date__c < TODAY)
```

## 6. Solution

Make the field read-only everywhere it is shown on a Hired court case, and explain it:

- **Both Hired record pages:** the Dynamic Forms field instance's `uiBehavior` set to `readonly`.
- **The Hired page layout:** the field set to `Readonly`. This is needed as well as the pages, because the Default Hired page is also served on phones through the mobile record-detail component, which draws the layout, and the highlights panel's inline edit follows the layout too. Without it the pencil survives in both places.
- **Help text on the field:**

> Filled in automatically from Court Appearances: the nearest hearing today or later that has no disposition. Blank when there is none. To change it, enter the disposition on the last hearing (with its next court date) or add a Court Appearance.

**Reconciled against production first.** All four components were retrieved from production on 2026-10-30. The pages and the field matched the repo apart from line endings. The layout did not: production lacked four fields the repo still listed. The deployed layout is production's copy plus the one-line change, so it does not put those four fields back.

The 9 blank cases fill in on their own once their dispositions are entered; no data was changed.

## 7. Testing

Validated check-only against production, naming `NextCourtDateServiceTest` because production requires a test and that class covers the logic behind the field. There is no Apex in this change, so the tests show only that nothing around it broke.

**The validation rendered no page.** The read-only state and the help icon were left to a live check: on desktop and on a phone, a Hired court case shows no pencil on the field in the details section, the header or the mobile record detail, and shows the help icon.

**Not covered:** whether that live check was carried out is not recorded on the ticket.

## 8. Step by step

1. **Read the field's rule** before treating "won't save" as a bug. Here `NextCourtDateService` owns the field.
2. **Check the named records' appearances** for a past hearing with no disposition.
3. **Count other affected records** with the query in section 5.
4. **Find every place the field is editable**: each record page (Dynamic Forms field instances), the page layout (for mobile record detail and the highlights panel), and any other page.
5. **Retrieve those components from production** and diff them against the repo.
6. **Set the field read-only** on each page and the layout, and add help text to the field.
7. **Validate check-only against production**, get the PR approved, re-retrieve at deploy time, then deploy.
8. **Live check on desktop and phone**: no pencil anywhere, help icon present.
9. **Tell the requester** that the date fills in once the hearing's disposition is entered.

## 9. Rollback

Set `uiBehavior` back to `none` on both pages and `behavior` back to `Edit` on the layout, and remove the help text, in metadata or in App Builder and Setup. No data is written, so nothing needs restoring. It would bring back an edit that cannot save.

## 10. Known limits

- **A hearing with no disposition** leaves the date blank from the night after it. The fix is entering the disposition, not editing the matter.
- **`Next_Court_Date__c`** is still editable on one other Opportunity page, where the nightly sweep reverts a typed value.
- **Admin edits** through other tools (data loader, API) are still replaced by the trigger.

## 11. Reuse for similar issues

**How to recognise one:** "I type a value and it won't save" on a field automation owns.

- **Find what owns the field** before looking for a bug. If code recalculates it on every save, a manual edit can never stick.
- **A blank calculated value usually means its source data is incomplete.** Here, a missing disposition.
- **Make calculated fields read-only and say why in help text**, so users stop trying to edit them.
- **Read-only on a Lightning page is not enough on its own.** Check the page layout too: mobile record detail and the highlights panel follow it.
- **Edit production's copy of pages and layouts**, never a stale repo copy; production can differ in ways that matter.
