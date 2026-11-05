# A deposit not at the bank after one business day turns red


| | |
|---|---|
| **Ticket** | DDT-1416 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-10-28 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-10-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom field, permission set) |
| **Area** | Trust Tie-Out reconciliation screen, waiting deposits |
| **Related** | DDT-1312 |
| **Code** | PR #821, #823, #833 and #840 in the ddt-salesforce repo |


---


## 1. Summary

Firm leadership wanted a deposit that the bank had not shown for too long to stand out on the Trust Tie-Out screen so it cannot be ignored. The screen already listed waiting deposits and coloured one after 3 calendar days from the received date, which confused money sitting in the office with money the bank was holding. It now runs two clocks in business days: from the date handed to the bank (until the bank file shows it) and from the date received (until it is taken to the bank). Either turns red after one business day, and each red deposit needs a typed reason before sign-off. The at-bank clock stops at the bank file's "through" date, so a deposit the file could not yet show is not red.

## 2. What was reported

- 2026-10-19 (on DDT-1312): firm leadership asked that a deposit still waiting after 5 business days turn red. Moved to its own ticket on 2026-10-26.
- 2026-10-26: threshold set to one business day; cover money never taken to the bank as well; red as a warning only.
- 2026-10-27: changed to require a typed reason for each red deposit, and later that day the at-bank clock was ruled to count to the bank file's through date.

## 3. Scope

**In scope**

- `TrustReconciliationController` (`businessDaysBetween`, `isWeekend`, `depositClock`, the sign-off guard and storing reasons) and its test class.
- LWC `trustTieOut`: the age cell names its clock, a reason box under each red deposit.
- New field `Trust_Credit_Explanation__c.Deposit__c` (lookup to `Settlement_Recovery__c`); read/edit access in permission set `Trust_Credit_Explanations`.

**Out of scope**

- A holiday calendar: the firm keeps none in Salesforce, and a guessed holiday would hide a late deposit on a day the bank was open. Weekends only.
- Blocking sign-off until the bank moves: the bank is not the firm's to hurry.

## 4. How it works (background)

A waiting deposit is a `Settlement_Recovery__c` with two dates: `Date_Received__c` (money arrived) and `Date_Deposited_To_Bank__c` (handed to the bank). It stops waiting when its landed date is confirmed, usually from the uploaded bank file (DDT-898). Raised bank credits could already be explained with a typed reason stored on `Trust_Credit_Explanation__c` at sign-off.

## 5. Root cause

Not a defect at first: a missing control. The existing rule in `trustTieOut.js` counted 3 calendar days from the received date, so a deposit taken to the bank that morning could read as weeks late, and money never taken to the bank looked the same as money the bank was holding. Defects found during the build:

- Weekend arrivals turned red on Monday morning, before any working day had passed (found in review of PR #821).
- A test assertion only held on weekdays, so a weekend deploy would have failed on this class.
- Both clocks ran to today, so a deposit handed over Monday was red on Tuesday against a Tuesday export that only runs through Monday: one false alarm, and one throwaway reason, per handover.

## 6. Solution

**Two clocks (PR #821, 2026-10-26).** Handed to the bank and not on a file: count from `Date_Deposited_To_Bank__c`. Never handed over: count from `Date_Received__c`. Red at 1 business day, counted exclusive of the starting day (handed over Monday, red Tuesday). A deposit the bank file shows is never red. The cell says which clock, for example "3d at bank" or "6d in office", because the two problems have different owners. The 3-day rule was removed. `isWeekend` works from a known Sunday rather than comparing an English day name, which a locale change would silently break.

**Weekend roll-forward (PR #823, 2026-10-27).** A weekend start date rolls to the next Monday before counting, so Saturday money is red on Tuesday and Friday money on Monday. The count is closed-form (five business days per whole week, at most six days stepped), because a day-by-day walk on up to 200 rows on the screen's load path could stop the board loading. The weekday-only test assertion was replaced with one that holds every day (business age is under calendar age).

**Typed reason (PR #833, deployed 2026-10-27).** Each red deposit shows a box; type one line and sign-off works, leave it empty and sign-off is refused, naming the deposits. Decisions:

- The refusal is in Apex: `signOff` rebuilds which deposits are red itself, so a stale browser tab cannot skip it.
- It runs after the file is matched and after the closed-day check, so it never demands a reason for a deposit the screen showed no box for.
- Only deposits the server itself calls red are stored.
- Reasons are keyed on the deposit (the new `Deposit__c` lookup), not on amount and bank date, which a deposit the bank has not shown does not have.
- The permission set grant is needed so the stored reason can be traced to its deposit in records and reports, even though Apex writes it either way.

**Clock stops at the file (PR #840, merged 2026-10-28).** One helper, `depositClock(dateToBank, dateReceived, bankClockEnd)`. With a file on screen the at-bank clock ends at the file's through date; otherwise at today. The office clock always ends today. The sign-off guard uses the same date, read from the same file payload the screen used, so the screen and the server agree. A stale file excuses nothing: past the same one-day threshold, the clock goes back to today, so uploading an old file cannot clear the reason boxes.

## 7. Testing

- PR #821: 98 of 98 Apex in the production validation; 4 new Jest tests, confirmed to execute.
- PR #823: 101 of 101; the replacement assertion was checked on all seven weekdays.
- PR #833: 106 of 106 Apex and 140 of 140 Jest; full sandbox deploy 106 of 106. On the screen in production the waiting deposit showed red with the box under it.
- PR #840: 111 Apex tests pinned to fixed dates and Jest 144 of 144. The production deploy of PR #840 is not recorded in the ticket.
- Live: on 2026-10-28 firm leadership's sign-off found the one waiting deposit on the bank file, so it was not red and no reason was needed. Firm leadership closed the ticket anyway. Not covered: a typed reason has not yet been saved by a real sign-off in production; that save is proven only in tests. The "never taken to the bank" case had no production record to look at.

## 8. Step by step

1. Decide which date starts each clock and what each clock is waiting for.
2. Count business days closed-form, rolling weekend starts to Monday; no guessed holidays.
3. End the at-bank clock at the uploaded file's through date, unless the file is itself older than the threshold.
4. Name the clock in the cell, not just the colour.
5. Require a typed reason per red item, enforced on the server with the same inputs the screen used.
6. Store reasons keyed on the record they explain, with field access on a permission set.
7. Remove the old rule rather than running two.
8. Have a real user sign off with a red item to prove the save.

## 9. Rollback

Redeploy `TrustReconciliationController` and the `trustTieOut` bundle from before the relevant PR. Reverting PR #821 brings back the 3-day calendar warning; reverting #823 makes weekend money red on Monday; reverting #840 makes both clocks end today. Leave the `Deposit__c` field and its permission in place: removing a field that holds reasons destroys the record they explain. Nothing else is stored; all ages are computed on read.

## 10. Known limits

- Weekends only; bank holidays count as business days.
- Signing the same day twice starts with empty reason boxes; reasons must be retyped.
- The reason save has not yet been seen in a real production sign-off.
- "After 1 business day" is read as red on the next business day; it is one constant, `STALE_DEPOSIT_BUSINESS_DAYS`, if that proves too eager.

## 11. Reuse for similar issues

- Signs of the same problem: an age warning that counts from the wrong event, or to today when the evidence (a bank export) only runs to an earlier date.
- Separate clocks for problems with different owners, and say which one is running.
- A control that needs a reason must be enforced on the server with the same inputs the screen used, or the screen and server disagree.
- Make test dates fixed when the subject is which end date is used; a `Date.today()` fixture can pass on weekdays and fail on weekends.
