# Collections welcome call tracking

| | |
|---|---|
| **Ticket** | DDT-1351 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula and roll-up fields, quick action, validation rule, before-save flow, reports, permission set), fw1 invoicing package |
| **Area** | Collections: flagging new hires who need a welcome call, logging it, and reporting the KPI |
| **Related** | Not recorded in the ticket |
| **Code** | PR #785, PR #787 and PR #793 in the `ddt-salesforce` repo |

---

## 1. Summary

The collections team makes a Collections Welcome Call (CWC) on some new hires, and its completion feeds a staff KPI and bonus, so the numbers had to be exact. Salesforce now flags each invoice that needs a CWC, offers a **Log CWC** button that stamps the date, user and outcome, locks the logged date so the KPI can't be shifted, and reports open CWCs and the monthly KPI.

It shipped in three steps: the flag, button and reports (PR #785); a lock on the logged date (PR #787); and three business decisions applied after go-live (PR #793).

## 2. What was reported

This was a feature request. A case needs a CWC when **all three** are true:

1. The case fee is **over** $2,000 (exactly $2,000 does not qualify).
2. The client is on a payment plan.
3. The plan runs **longer than 3 calendar months** from the first scheduled payment to the last (exactly 3 months does not qualify).

The request asked for a self-setting flag, a log action that shows only when flagged, an open-CWC report (oldest first, overdue past 14 days highlighted) and a monthly KPI: CWCs logged within 14 days of hire divided by CWCs needed, where any logged outcome counts.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `fw1__Invoice__c.CWC_Needed__c` | Formula checkbox | The flag |
| `fw1__Invoice__c.CWC_Outcome__c` | Restricted picklist | Completed, Client declined, No response after attempts |
| `fw1__Invoice__c.CWC_Logged_Date__c`, `CWC_Logged_By__c` | Date, Lookup(User) | Stamped by Log CWC |
| `fw1__Invoice__c.CWC_Logged_On_Time__c` | Formula | 1 if logged 14 days or less after hire |
| `fw1__Invoice__c.CWC_Days_Since_Hire__c` | Formula | Today minus the hire date |
| `fw1__Invoice__c.First_Due_Date__c` | Roll-up | Minimum installment due date (used by PR #785; no longer read after PR #793) |
| `fw1__Installment__c.CWC_Due_Date_At_Hire__c` | Date | The installment's due date as the plan stood at hire (PR #793) |
| `CWC_Plan_Start_At_Hire__c`, `CWC_Plan_End_At_Hire__c`, `CWC_Plan_Payments_At_Hire__c` | Invoice roll-ups | MIN, MAX and COUNT of the at-hire due dates (PR #793) |
| `Installment_CWC_Plan_At_Hire` | Before-save flow on `fw1__Installment__c` | Freezes the plan at hire (PR #793) |
| `fw1__Invoice__c.Log_CWC` | Update quick action | Stamps today and the running user; outcome required |
| `Invoice_Record_Page2` | Existing record page | Log CWC added as a dynamic action, visible only when the flag is on |
| `fw1__Invoice__c.CWC_Logged_Date_Locked` | Validation rule | Blocks any change to a logged date (PR #787) |
| `CWC_Invoices` | Report type | Invoice-based, one row per invoice |
| `CWC_Open`, `CWC_KPI_by_Month` | Reports | In the collections staff member's report folder |
| `Collections_Welcome_Call` | Permission set | Field access for the new fields |

**Out of scope**

- **Apex and triggers.** None; everything is declarative.
- **The Opportunity.** It holds no plan dates and is at 498 of 500 custom fields.
- **Cases hired before go-live, and Did Not Hire cases.** Excluded by decision (section 6.3).

## 4. How it works (background)

The payment plan lives on the **invoice** (`fw1__Invoice__c`, from the fw1 package): the schedule is its `fw1__Installment__c` child rows. So the flag sits on the invoice.

```
Case hired -> invoice + installments created (fw1)
  -> CWC_Needed__c (formula) evaluates the three rules
  -> flagged invoices appear on CWC Open
  -> staff click Log CWC -> outcome, date, user stamped -> invoice leaves CWC Open
  -> CWC KPI by Month: logged within 14 days of hire / needed, by hire month
```

## 5. Root cause

Not applicable: this was a new feature. Three findings shaped it:

- **The fee:** the request suggested the invoice total "if it's reliable". It isn't for this rule: `fw1__Total_Invoice_Amount__c` also carries payment-plan fees and late fees added after hire. Using it would have flagged 8 more 2026 invoices with a fee of $1,000 to $2,000, three of them at exactly $2,000. The **net amount** (`fw1__Net_Amount__c`, invoice lines less discounts) was used instead.
- **The first payment:** `Oldest_Due_Date__c` already existed but counts only **unpaid** installments, so it moves forward as the client pays and would shrink the span. A new roll-up was needed.
- **Package fields checked against raw data:** for all 2,384 invoices on 2026 hires, `fw1__Installment_Count__c` and `Final_Due_Date__c` agreed with the raw installments on every row.

## 6. Solution

### 6.1 The flag (PR #785, then PR #793)

The final formula:

```
AND(
    NOT(fw1__Is_Voided__c),
    NOT(Did_Not_Hire__c),
    Court_Case_Hire_Date__c >= DATE(2026, 10, 20),
    fw1__Net_Amount__c > 2000,
    CWC_Plan_Payments_At_Hire__c > 1,
    CWC_Plan_End_At_Hire__c > ADDMONTHS(CWC_Plan_Start_At_Hire__c, 3)
)
```

- **Strictly greater** on both thresholds, so exactly $2,000 and exactly 3 months don't qualify.
- **`ADDMONTHS`** is calendar months and clamps month-ends (Nov 30 + 3 months = Feb 28).
- **Voided invoices are excluded** so a re-issued invoice can't count a case twice in the KPI.

### 6.2 Logging and locking (PR #785, PR #787)

**Log CWC** is an Update quick action whose field overrides stamp `TODAY()` and the running user and ask for the outcome. It shows on the invoice page only when the flag is on.

Without a guard, a second click on a later day would re-stamp the date and turn an on-time CWC late in the KPI. The validation rule `CWC_Logged_Date_Locked` blocks any change to a logged date, including clearing it:

```
AND(
    ISCHANGED(CWC_Logged_Date__c),
    NOT(ISBLANK(PRIORVALUE(CWC_Logged_Date__c)))
)
```

It fires only when that field changes, so ordinary invoice saves and the fw1 package's own updates are unaffected. Making the two stamp fields read-only in the permission set was tried and withdrawn, because it couldn't be confirmed that the quick action would still stamp them for a read-only user.

**Why a new days-since-hire field.** The existing `Opportunity.Days_Since_Hired__c` stops counting when the case closes, so a case closed a week after hire would read "7 days" forever.

### 6.3 Three decisions after go-live (PR #793)

After the first release the firm decided:

1. **Only new hires from go-live (2026-10-20).** Older cases are not flagged and are out of the KPI.
2. **Leave out Did Not Hire cases.**
3. **Judge the 3-month plan on the plan as it stood at hire**, not counting fees or reschedules added later. This affected 31 of 476 cases.

The first two are formula terms. The third needed the plan **frozen**, because installments change after hire. The before-save flow `Installment_CWC_Plan_At_Hire` copies each installment's due date (`fw1__Installment_Date__c` + `fw1__Terms__c`, the same as fw1's own due-date formula) into `CWC_Due_Date_At_Hire__c` while the case is not yet hired, or until the day after hire. After that window it never writes again, and rows added later stay blank. Three invoice roll-ups over that field feed the formula. A correction on hire day or the day after still counts.

**Why a before-save flow on the installment, not a snapshot on the invoice.** Writing the invoice would fire every invoice automation (admin-fee charges, demand letters, recalculations) on each write. A before-save flow on the installment adds no DML and no SOQL, and the roll-ups recalculate on the installment save that already happens.

A backfill script for the at-hire field (dry run by default, with an expected-count guard) shipped in the deploy doc for rows the flow hadn't seen. Whether and when it was run is not recorded in the ticket.

## 7. Testing

**Formula against an independent calculation.** In the sandbox, a script recomputed the three rules from raw installments for all 10,052 invoices: **0 mismatches**, 707 flagged. The sandbox held 82 plans at exactly $2,000 and 53 plans of exactly 3 months; none were flagged.

**Done-when cases** (anonymous Apex, rolled back):

| Case | CWC Needed |
|---|---|
| $2,500, Nov 1 → Feb 2 | **true** |
| Exactly $2,000, same plan | false |
| $2,500, one payment | false |
| Nov 1 → Feb 1 (exactly 3 months) | false |
| Nov 30 → Feb 28 (month-end, exactly 3 months) | false |
| Nov 30 → Mar 1 | true |

**Log CWC took the case off the open report:** the real quick action stamped date and user with no input, and CWC Open went from 707 rows to 706.

**KPI boundary:** logged 0 and 14 days after hire → on time; 15 days → late.

**Overwrite guard:** a second click 3 days later, a hand edit and clearing the date were all rejected; a second click the same day and an unrelated invoice edit succeeded. The Apex test classes that write `fw1__Invoice__c` were run against production with the rule in place: **102 of 102 passing.**

**Plan at hire** (sandbox, rolled back): qualifying plans hired on go-live day flagged; plans hired before go-live, Did Not Hire cases, and short plans with a fee added later not flagged; a qualifying plan rescheduled shorter after the window stayed flagged; a short plan corrected inside the window became flagged. The plan-at-hire change was validated check-only against production with 103 of 103 tests passing.

**Live:** PR #785 and PR #787 went live on 2026-10-20 and PR #793 on 2026-10-21. One case was flagged on 2026-10-21.

**Not covered:** the Log CWC button rendering on the page was not tested in the sandbox, because the invoice page can't deploy there. It was validated against production only, and the collections staff member's live check of the button was the final step.

## 8. Step by step

1. **Find where the data lives.** Confirm which object holds the plan (here the invoice's installments) before adding a field.
2. **Pick the fee field deliberately.** Compare total and net amounts on real invoices; choose the one that excludes fees added after hire.
3. **Check package fields against raw data** before relying on them.
4. **Build the formula** with strict comparisons and `ADDMONTHS`, excluding voided invoices.
5. **Add the quick action** with field overrides for date and user, and add it to the invoice page as a dynamic action visible only when the flag is on. Hand-apply the page change on top of production's current page.
6. **Add the validation rule** that locks the logged date, and run every Apex test class that writes the invoice against production with the rule in place.
7. **Build the report type and reports**, and grant field access through the permission set; assign it after deploy.
8. **For a frozen-at-hire value**, add the before-save flow on the child row and roll-ups on the parent. The flow deploys as Draft; activate it after deploy.
9. **Backfill** rows the flow hasn't seen: dry run first, set the expected count, then apply.
10. **Deploy away from 12:00 UTC.** New roll-ups recalculate every invoice, and the fw1 package runs its own daily chain at that time.
11. **Check:** `SELECT COUNT() FROM fw1__Invoice__c WHERE CWC_Needed__c = true` returns only cases hired on or after go-live, and CWC Open lists only those.
12. **Live check by the user:** open a flagged invoice, log a CWC, confirm the case leaves CWC Open, and confirm a second click on a later day is refused.

## 9. Rollback

- **Flag and reports:** a rollback procedure is not recorded in the ticket. The build adds new components only (plus one action on the invoice page), so removing the action from the page hides the feature.
- **Plan at hire:** deactivate the flow. Values already frozen stay on the installments.
- **Logged CWCs** are business records. Correcting a wrong logged date needs the validation rule deactivated for that edit.

## 10. Known limits

- **A deleted installment leaves the frozen plan.** In the 15 days to 2026-10-21, 7 invoices had at-hire rows deleted, 6 of them within 14 days of hire, which reads as corrections.
- **`CWC_Logged_By__c` can still be hand-edited.** The KPI reads only the date, which the rule protects.
- **The validation rule applies to admins too.**
- **The go-live date is hard-coded** in the formula.
- **The reports' folder is shared with one user.** Others can't see them until it is shared.

## 11. Reuse for similar issues

- **Put the flag where the data lives**, not where it's convenient. Here that was the invoice, not the near-full Opportunity.
- **"Total" is rarely the right amount.** Check what each currency field includes before using it in a business rule.
- **Strict thresholds:** say in the formula whether the boundary qualifies, and test exactly-on-the-boundary cases.
- **Lock what a KPI reads.** A validation rule on `ISCHANGED` and `PRIORVALUE` stops a re-click or hand edit from shifting a logged date.
- **To judge something "as it stood at a point in time"**, freeze the value on the child row with a before-save flow inside a time window, and roll it up.
- **Recompute the rule independently** over every record and compare, rather than only testing hand-picked cases.
