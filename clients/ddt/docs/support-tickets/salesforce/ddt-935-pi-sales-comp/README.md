# PI cases in sales compensation and the dashboard rebuild

| | |
|---|---|
| **Ticket** | DDT-935, DDT-1191, DDT-1192, DDT-1193 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-22 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex, custom object, formula fields, reports and dashboard) |
| **Area** | Sales compensation: the shared sales pool and the Sales Comp Dashboard |
| **Related** | DDT-902 (derive the case source from attribution instead of by hand) |
| **Code** | PRs #292, #592, #644, #742, #795 and #801 in the ddt-salesforce repo |

---

## 1. Summary

The sales team started selling personal injury (PI) cases as well as defense cases, and a new PI sales compensation SOP took effect on 2026-08-01. The Sales Comp Dashboard only counted defense, so PI signings earned nobody anything and the sales manager was correcting pay by hand.

The hard part: defense cases are Opportunities and PI cases are `Matter__c` records, two unrelated objects, and no Salesforce report can sum one number across both. The fix was built in three steps: get the comp fields right on every Matter, add a new object `Board_Entry__c` that holds one row per boarded case from either side, and rebuild the dashboard on that object.

## 2. What was reported

- **DDT-935 (2026-09-04).** The parent request. The dashboard counted defense only, so PI points were accruing against a screen that could not see them. It set out the pay rules, the target design and nine "done when" checks.
- **DDT-1191 (2026-10-02).** Step A, split out of DDT-935: deploy the two Matter flows, backfill the comp fields on every Matter hired since 2026-07-01, fill missing case sources, and tick the sellers on the comp plan. Done 2026-10-07.
- **DDT-1192 (2026-10-02).** Step B: build `Board_Entry__c` and write one row per boarded case, defense and PI together, plus a one-time load of July to September. Done 2026-10-21.
- **DDT-1193 (2026-10-02).** Step C: rebuild the Sales Comp Dashboard on `Board_Entry__c` and ship it. Live 2026-10-22.

When the work started, prod showed `Disengaged_Date__c` and `Companion_Rank__c` filled on 0 of 305 Matters, and 0 users ticked as comp participants.

## 3. Scope

**In scope**

| Component | Type | Ticket |
|---|---|---|
| Matter comp fields: `Sales_Case_Source__c`, `Disengaged_Date__c`, `Companion_Rank__c`, and formulas `Sales_Tier__c`, `Sales_Tier_Points__c`, `Sales_Board_Points__c`, `Board_Date__c`, `Board_Month__c`, `Boarded__c` | Fields on `Matter__c` | DDT-935 (PR #292, live 2026-09-04) |
| `User.Sales_Comp_Participant__c` (the roster) and permission set `PI_Sales_Comp_Tracking` | Field, permission set | DDT-935 (PR #292) |
| `Matter_Stamp_Disengaged_Date`, `Matter_Stamp_Companion_Rank` | Before-save record-triggered flows on `Matter__c` | DDT-935 (PR #592) |
| `Boarded__c`, `Board_Date__c` | Formula and help text: a drop on day 30 does not board | DDT-1191 (PR #644) |
| `Board_Entry__c` with its fields, `Source_Key__c`, validation rules `Insert_Only` and `Exactly_One_Source`, a read-only layout | New custom object | DDT-1192 (PR #742) |
| `Board_Entry_Config__mdt` (`Enabled__c`, `Cutover_Date__c`, `Backfill_Floor__c`) | Custom metadata switch | DDT-1192 |
| `BoardEntryService` and `BoardEntryServiceTest` | Apex: builders, insert and the hourly sweep | DDT-1192 (PRs #742, #795) |
| `GlobalMinuteScheduler` | One new rider, `dispatchBoardEntry` | DDT-1192 |
| `Board_Entry__c.Case_Type__c`, report types `Board_Entries`, `Sales_Comp_Matters`, `Sales_Comp_Opportunities`, eight reports in the `SalesCompDash` folder, the Sales Comp Dashboard | Reports and dashboard | DDT-1193 (PR #801) |

**Out of scope**

- **The SOP v1.1 rewrite.** Owned by firm leadership, not engineering.
- **Deriving the case source automatically.** That is DDT-902. `Sales_Case_Source__c` stays a manager-entered field.
- **The Matter record page drift.** The comp fields were added to the page by hand in App Builder, because the repo's copy of the page differs from prod.

## 4. How it works (background)

The pay rules, from the ticket:

```
pool = board_dollars > 300000
     ? 15000 + (board_dollars - 300000) * 0.08
     : board_dollars * 0.05

seller_pay = pool / total_points * seller_points
```

- **Defense** contributes the case's initial payment total and its sales points.
- **PI** contributes a flat $5,000 per boarded case, plus points by source tier: Organic 5, PI Advertising 2, Referral 1, Arrived Pre-Signed nothing. A companion case (another client from the same accident) earns 2 whatever its tier.
- **A PI case boards 30 days after signature** and is then earned for good. No clawback.
- **A case signed by anyone not on the comp roster is a "house deal".** No points and no dollars, so it cannot raise everyone else's per-point pay.
- The PI $5,000 counts toward the $300,000 threshold like any other dollar (SOP v1.1).

Before the change, the dashboard's reports were all on Opportunity, filtered by a hardcoded list of first names, and the one PI report counted a retired checkbox.

## 5. Root cause

Not a bug: a new line of business that the reporting could not see. Three separate gaps had to be closed.

1. **The Matter fields that decide pay were empty.** `Boarded__c` read a blank `Disengaged_Date__c` as never dropped, and `Sales_Board_Points__c` treated a blank `Companion_Rank__c` as a first case. Nothing stamped either field.
2. **No report can sum across Opportunity and `Matter__c`.** They are unrelated objects, and a joined report needs a shared primary object. So the single pool number could not be produced by any report or dashboard filter.
3. **The day-30 rule was off by one.** `Boarded__c` read `Disengaged_Date__c >= Hire_Date__c + 30`, so a case dropped on day 30 still boarded. Leadership ruled on 2026-10-06 that the firm has 30 days to drop, so a day-30 drop must not pay.

**Two first guesses that turned out wrong:**

- **Companion rank by "earlier Hire Date".** The ticket asked for it. Measured in prod on 2026-10-01: in 25 of 30 multi-matter incidents created in 2026, two or more companions were signed the same day. Ranking by strictly earlier hire date would have scored 5 + 5 + 5. Creation order matched hire order in all 30, so the flow counts the matters already on the incident instead.
- **Writing the defense row when the case becomes Hired.** In August, the first payment landed after the Hired transition on 193 of 250 defense cases. A row written at Hired would have carried $0 for most of defense.

## 6. Solution

### 6.1 Stamp the Matter fields (DDT-935, PR #592)

- **Matter - Stamp Disengaged Date** (before-save, create and update). Stamps today when `Hire_Status__c` first becomes `Disengaged/Dropped`, `Disengaged - Has New Counsel` or `Disengaged - Referred Out`. Never on `To Disengage`, which is an intention, not a disengagement. Never overwrites an existing date.
- **Matter - Stamp Companion Rank** (before-save, create only). Rank = the number of Matters already on the same `Incident__c`, plus one. Matters are inserted one per transaction, so the ones already saved are exactly the earlier cases.

The four existing before-save flows on Matter were checked; none writes the fields these flows read or write, so trigger order does not matter.

### 6.2 Backfill, roster and the day-30 fix (DDT-1191)

- Both fields were backfilled once from history on every Matter hired since 2026-07-01: drop date from the first Hire Status change into a Disengaged value in field history (Chicago time), rank from creation order on the incident. Each month's list was posted for review before anything was written.
- The one Matter with no case source was set by leadership.
- Leadership named five sellers. `Sales_Comp_Participant__c` was ticked on those five only, and each got `PI_Sales_Comp_Tracking`.
- PR #644 changed `Boarded__c` from `>=` to `>`, and rewrote the `Board_Date__c` help text to say day 30 is the last day the firm can drop.

### 6.3 `Board_Entry__c`: one row per boarded case (DDT-1192, PRs #742 and #795)

- **PI row:** written when `Boarded__c` is true **and** `Board_Date__c` is before today in Chicago. The second condition is needed because `Boarded__c` already reads true on day 30 itself. Points from Board Points, $5,000 board dollars, $0 collected cash.
- **Defense row:** written when the invoice's initial payment total is above 0, and only for a seller on the roster. Line Type is SOS when the sales category contains SOS, otherwise Defense.
- **Defense dollars top up during the hire month, then lock** (leadership, 2026-10-20). While the row's Board Month is the current month, each sweep sets Board Dollars and Collected Cash to the current initials. When the month ends the row never changes again. Under the earlier "frozen at first payment" rule, August would have missed $24,891 because 26 cases split their initials.
- **Insert-only by design.** Validation rule `Insert_Only` rejects every edit except those two defense fields during the hire month. `Source_Key__c` holds the source record Id and is unique, so a second row for the same case is impossible even if two sweeps overlap. This is how "no clawback" is enforced by the data model rather than by someone remembering.
- **No new scheduled job.** Production was near the scheduled Apex limit, so the sweep rides the existing `GlobalMinuteScheduler` at :02 every hour, Chicago time, and enqueues a Queueable (never a batch). Both queries are standing "qualifies and has no row yet" queries, so a missed hour is picked up by the next.
- **Ships switched off.** `Board_Entry_Config__mdt.Enabled__c` was false until the July to September load was reviewed and written (780 rows), then switched on.
- **Stragglers are logged, never written.** Once a day the sweep checks the reviewed months for cases that qualify late (a late payment entry, a late roster tick, a late case source) and writes one error log row naming them, so a closed month never changes without the sales manager seeing it.

### 6.4 Rebuild the dashboard (DDT-1193, PR #801)

- All 10 existing components read one `Board_Entries` report type and keep their positions and headers. "Initials Pool" became "Initials Pool incl. PI".
- Every hardcoded first-name filter was replaced with `Seller__r.Sales_Comp_Participant__c = true`.
- The "Hire Date" dashboard filter now links to `Board_Month__c`.
- The pool formula keeps its shape on the new fields:

```
IF(board > 300000, 15000 + (board - 300000) * 0.08, board * 0.05)
  / total points * seller points
```

- New tiles: Matters Sold This Month, PI Points This Month, Collected Cash, Signed vs Boarded (PI signed but not yet earned, beside PI boarded), and House Deals for PI and for defense. House deals have no `Board_Entry__c` row, so those two tiles read their source objects directly.
- The Initials vs Team Goal gauge max went from $400,000 to $500,000.
- The old Organic PI Hires report moved out of the unfiled folder into `SalesCompDash`.

## 7. Testing

**Matter flows (PR #592).** Two savepoint scripts in `scripts/apex/` ran in the sandbox: 7 of 7 and 3 of 3 cases passed and rolled back, and both scripts were shown to fail against deliberately broken flows. In prod, a test Matter moved to Disengaged/Dropped got today's date and a second Matter on the same incident got rank 2; the test was rolled back.

**Day 30 (PR #644).** `scripts/apex/verify_pi_comp_board_formulas.apex` gained case 12, a drop on day 30 exactly. In the sandbox it failed against `>=` (11 pass, 1 fail) and passed against `>` (12 of 12). No Matter changed value in prod: the only two day-30 drops were house deals.

**Board entries (PRs #742, #795).** `BoardEntryServiceTest` covers each DDT-935 check:

| Check | Test |
|---|---|
| Dropped on day 40 keeps its row | `aCaseDroppedOnDay40KeepsItsRow` |
| Dropped on day 25 never gets one | `aCaseDroppedOnDay25NeverGetsARow` |
| House deal gets no row | `aHouseDealGetsNoRow` |
| 3 Matters on one incident score 5 + 2 + 2 | `threeMattersOnOneIncidentScore5Plus2Plus2` |
| Pre-signed gets no row | `aPreSignedMatterGetsNoRow` |
| Running twice leaves the count unchanged | `runningTheSweepTwiceLeavesTheRowCountUnchanged` |

The same checks were then run on real rows in prod as rolled-back anonymous Apex, and a non-admin seller was confirmed able to read rows but not edit them. The idempotency check was confirmed again on a live hourly run.

**Dashboard (PR #801).** `scripts/ddt-1193-verify-sales-comp.py` sums raw `Board_Entry__c` rows for a closed month in Python `Decimal`, runs every report through the Reports API, and reads the refreshed dashboard. September matched to the cent for the pool and every seller's projected pay. With `--break` it runs the formula with 7% instead of 8%, in memory only, and failed for every seller as it should. The dashboard was opened as a real seller and all 17 tiles rendered.

**Not covered:** a defense row built from a real payment-package invoice in an Apex test. Those invoices can only be inserted with `SeeAllData=true`, which the code analyzer blocks; the defense mapping is tested in memory and checked on real rows instead. The sandbox could not hold the new object (it is at its custom-object limit), so validation ran check-only against production.

## 8. Step by step

1. **Deploy the Matter fields and roster field** (PR #292) and assign `PI_Sales_Comp_Tracking`. Add the manager-entered fields to the Matter page in App Builder, by hand.
2. **Deploy the two stamp flows** (PR #592) and read back that both are Active.
3. **Backfill rank and drop date** on every Matter in scope, from field history and creation order. Post the list, get it reviewed, then write it in one DML and read it back.
4. **Tick the roster**: only the sellers the sales manager confirms. Query the ticked users and check the list matches exactly.
5. **Fix the day-30 boundary** (PR #644), proving the new verify case fails on the old formula first.
6. **Deploy `Board_Entry__c`** (PR #742) with `Enabled__c = false`. Run the done-when checks in prod as rolled-back Apex and confirm the row count is unchanged.
7. **Build the one-time load** for the months before the cutover, get it approved, run it, and read the count back.
8. **Deploy the top-up rule** (PR #795) and switch `Enabled__c` on only after the load is written.
9. **Rebuild the dashboard** (PR #801) and run the verify script for a closed month, then with `--break`.
10. **Open the dashboard as a seller**, not an admin. It runs as one specified user, so a missing permission shows as a blank tile, not an error.

## 9. Rollback

- **Sweep:** set `Board_Entry_Config__mdt.Enabled__c = false`. New rows stop within the hour and nothing else changes.
- **Full removal:** remove `dispatchBoardEntry` from the dispatcher first, then the class, then the object. Deleting the object deletes its rows, so it needs sign-off.
- **Dashboard:** redeploy the dashboard and the four rebuilt reports from the commit before PR #801. The new report types, reports and `Case_Type__c` can stay; nothing else uses them.
- **Day-30 fix:** reverting it would pay day-30 drops again, so there is no reason to.

## 10. Known limits

- **Unticking a seller removes their rows from closed months too.** The roster filter is evaluated today, not when the row was written, so per-point pay for past months would change on screen.
- **A defense case with points but $0 initials earns no points**, because it has no row.
- **April to June 2026 show empty.** Board rows start at July 2026.
- **The last hour of the month.** The final top-up runs at 23:02 Chicago on the last day; a payment landing after that is not on the row.
- **The flows act only going forward.** Disengaged Date stamps on a status change and Companion Rank only on create, so any Matter created outside them needs the same backfill.
- **`Matter__c` is at its field-history limit**, so changes to `Sales_Case_Source__c` are not audited. Edit rights through the permission set are the only control.
- **Check the House Deals tiles before each month closes.** They are the only thing that catches a seller losing a case because someone off the roster pressed the hire button.

## 11. Reuse for similar issues

**How to recognise one:** a number that has to combine two unrelated objects in one report.

- **Add a ledger object** with one row per event from either side, and report off that. Do not try a joined report across objects with no shared parent.
- **Make "no clawback" structural:** insert-only validation, a unique source key, and a sweep that only ever looks for "qualifies and has no row yet".
- **Ship the writer switched off**, load and review history first, then switch it on.
- **Check timing before choosing a trigger point.** Measure when the money actually lands before deciding when a row is written.
- **Test the boundary day.** A `>=` against a `>` on day 30 is exactly the kind of bug a verify script with a day-30 case catches.
- **Prove each check can fail:** break the formula or the flow on purpose and watch the check go red, then restore it.
- **Never hardcode names in report filters.** Use a roster flag on the User.
