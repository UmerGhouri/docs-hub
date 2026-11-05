# Hired Closed Cases card missing court dates

| | |
|---|---|
| **Ticket** | DDT-1376 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-20 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-19 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger service) |
| **Area** | Court cases: the Next Court Date shown for closed matters on the client record |
| **Related** | DDT-1301 (Next Court Date worked out from the appearances), DDT-1478 (same field, open matters) |
| **Code** | PR #776 in the `ddt-salesforce` repo |

---

## 1. Summary

The "Hired Closed Cases" card on the client record stopped showing a court date for matters closed after 2026-10-14. A change shipped that day under DDT-1301 blanks Next Court Date on an open matter once no hearing is left. Staff enter the last hearing's disposition first and close the matter afterwards, so the date was already blank by the time the matter closed.

The fix writes the latest hearing's date back onto the matter when it leaves Open with the date blank. The affected matters were restored.

## 2. What was reported

On 2026-10-19 a staff member reported that the card showed a blank Next Court Date row on closed cases. To find the last court date the team had to open each case and go into Appearances. They asked whether this was intentional.

It was not. In production, of matters closed from 2026-09-01 to 2026-10-13, every one with a hearing kept a date. Of matters closed since 2026-10-14, **32** with a hearing had a blank date, all with the last hearing dispositioned on or before the close.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `NextCourtDateService` | Apex class | `applyOnUpdate` writes the latest hearing's date back when a matter leaves Open with the date blank; new helper `lastHearing` |
| `NextCourtDateServiceTest` | Apex test | Three new tests (see section 7) |

**Out of scope**

- **Open matters.** Their behaviour from DDT-1301 is unchanged: an open matter shows the nearest undisposed hearing, today or later, and is blank when there is none.
- **A date that is still set when the matter closes** (an upcoming hearing). It is kept as it is, per the decision recorded on DDT-1301.
- **The card itself.** It was not changed; it was showing the field correctly.

## 4. How it works (background)

The matter is an Opportunity (labelled "Court Case"). Its hearings are `Court_Appearance__c` records.

```
Hearing created, dated, dispositioned or deleted
  -> Court_Appearance__c trigger -> NextCourtDateService.onCourtAppearanceChange
  -> Opportunity.Next_Court_Date_Time__c = nearest hearing with no disposition, today or later
     (blank if there is none)            -- open matters only (DDT-1301, live 2026-10-14)

Opportunity saved
  -> before-update trigger -> NextCourtDateService.applyOnUpdate
```

Before DDT-1301, the field was plain data that flows overwrote with "the date of the appearance just saved", so on a closed matter it held the last hearing. DDT-1301 deliberately leaves closed matters alone, because on a closed matter the field is the record of its last hearing.

## 5. Root cause

The order of work in the office and the DDT-1301 rule combined:

1. Staff disposition the last hearing while the matter is still Open.
2. With no undisposed hearing left, DDT-1301 blanks Next Court Date on the open matter. That is correct for an open matter.
3. Staff then close the matter. DDT-1301 never touches closed matters, so nothing put the last hearing's date back.
4. The Hired Closed Cases card read the blank field.

**Evidence:** the before/after split on close dates above. Every matter closed before 2026-10-14 with a hearing kept a date; the 32 blank ones were all closed from 2026-10-14 on, with the last hearing dispositioned on or before the close.

## 6. Solution

### 6.1 Write the last hearing back when a matter leaves Open

`applyOnUpdate` now collects matters moving from Open to any other status whose date fields are both blank, and fills them from the latest dated hearing, dispositioned or not:

```apex
} else if (
  old.Case_Status__c == OPEN &&
  o.Case_Status__c != OPEN &&
  o.Next_Court_Date_Time__c == null &&
  o.Next_Court_Date__c == null
) {
  leavingOpen.add(o);
}
// ...
Map<Id, Datetime> last = lastHearing(idsOf(leavingOpen));
for (Opportunity o : leavingOpen) {
  Datetime was = last.get(o.Id);
  o.Next_Court_Date_Time__c = was;
  o.Next_Court_Date__c = dateCopy(was);
}
```

`lastHearing` is one aggregate query: `MAX(Court_Date__c)` per matter over `Court_Appearance__c`, grouped by `Opportunity__c`.

Why it's done this way:

- **In the before-update trigger, so no extra DML.** The value is set in the same save that closes the matter.
- **Only when the date is blank.** A date still set at close is kept, which is what was agreed on DDT-1301.
- **Both fields are set together.** `Next_Court_Date__c` is the Date copy another flow keeps, and DDT-1301 already writes the pair in one go so they never disagree.
- **Open-matter logic is untouched.** The new branch is an `else if` after the existing open-matter branch.

### 6.2 Restore the matters already affected

The matters that lost their date since 2026-10-14 were restored, including the example in the ticket (confirmed in the ticket on 2026-10-20). How the restore was run is not recorded in the ticket or the PR.

## 7. Testing

Three Apex tests in `NextCourtDateServiceTest`:

| Test | What it proves |
|---|---|
| `closingAfterTheLastDispositionWritesBackTheLastHearing` | Repeats the real sequence: disposition the last hearing (date goes blank while Open), then close. The closed matter shows the last hearing's date. |
| `closingWithAnUpcomingHearingKeepsThatDate` | A matter closed with an upcoming hearing keeps that date. |
| `closingAMatterWithNoHearingsLeavesItBlank` | A matter with no hearing at all stays blank when closed. |

Validated check-only against production: `NextCourtDateServiceTest` 16 of 16 passing.

**Live check:** the requester refreshed and confirmed on 2026-10-20 that it was fixed.

**Not covered:** whether the first test fails without the fix is not recorded. The tests exercise Apex only; nothing rendered the card.

## 8. Step by step

1. **Confirm the pattern.** Find closed matters with a blank `Next_Court_Date_Time__c` that have at least one `Court_Appearance__c`, and compare close dates before and after the change you suspect.
2. **Read the service.** Open `NextCourtDateService` and check which statuses each entry point touches.
3. **Add the leaving-Open branch** to `applyOnUpdate` as in 6.1, with the `lastHearing` helper.
4. **Add the three tests** from section 7.
5. **Validate check-only against production** with `NextCourtDateServiceTest`.
6. **Get the PR approved**, then deploy.
7. **Restore the matters closed since the regression**, setting each one's date to its latest hearing.
8. **Ask the requester** to refresh the card on an example matter.

## 9. Rollback

Redeploy the previous `NextCourtDateService`. That brings the blank dates back for every matter closed afterwards. The service also has an off switch that needs no deploy: a `Triggers_Settings__c` record named `NextCourtDate` with `Disabled__c` checked. That turns off all of DDT-1301, not just this branch.

## 10. Known limits

- **Only matters closed after the fix, or restored, get a date.** The write-back runs at the moment a matter leaves Open.
- **A matter closed with no dated hearing stays blank.** There is nothing to write back.
- **Reopening and re-closing:** reopening puts the nearest upcoming hearing back (DDT-1301); closing again writes the last hearing back only if the date is blank at that moment.

## 11. Reuse for similar issues

**How to recognise one:** a field is right for records in one state and wrong for records that moved through that state into another.

- **Think about the order users actually do things in.** The DDT-1301 rule was correct for open matters and correct for closed matters on their own; the gap was the transition between them.
- **Split by date of change.** Compare records closed before and after a deploy date to tie a regression to a release.
- **Fix it in the before-update trigger** when the value can be worked out from child records in the same save, so no extra DML or flow is needed.
- **Keep a paired field in step.** When one field has a copy another automation reads, write both in the same place.
