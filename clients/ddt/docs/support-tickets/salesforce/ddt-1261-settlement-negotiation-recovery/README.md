# Demand Tracker negotiation amounts and recoveries

| | |
|---|---|
| **Ticket** | DDT-1261 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-08 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-10-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom fields, permission sets) |
| **Area** | PI settlements: Demand Tracker and Settlement Statement |
| **Related** | DDT-1238 (Demand Phase task assignment), DDT-1347 (Matter Stage and Hire Status sync, settlement amount) |
| **Code** | PR #647 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI team was entering **recovery** rows on the settlement statement as soon as a deal was agreed, to record "a check is coming". A recovery is a check the firm is holding and it feeds the trust ledger, so a placeholder recovery puts a wrong number in trust. The right place for "deal agreed" is the Demand Tracker, but accepting a demand there was buried and never asked for an amount.

The change makes accepting an offer, with its amount, one click in the Demand Tracker. The settlement statement now shows agreed deals that are "waiting on the check", with a button that pre-fills the recovery when the check arrives. A recovery can no longer be saved without a Date Received, or with one in the future.

## 2. What was reported

Read from production on 2026-10-06:

- **0 of 30** demands had ever been closed as Accepted. 17 sat at Sent and 4 at Negotiations.
- **4** recoveries were Pending. Two were created 20 and 25 days before their check arrived, and one had no received date at all.

Firm operations decided two things: (1) block a recovery from saving without a Date Received, and (2) make accepting a negotiation, with its amount, easy and obvious.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DemandController` | Apex class | New `acceptOffer`; shared `closeDemand` helper with `markOutcome`; `markOutcome('Accepted')` refused |
| `DemandControllerTest` | Apex test | +7 tests for the accept path |
| `SettlementStatementController` | Apex class | `createRecovery` / `updateRecovery` refuse a blank or future Date Received; "waiting on the check" list |
| `SettlementStatementControllerTest` | Apex test | +10 tests for dates and the waiting list |
| `demandTracker` | LWC | Accept button and modal, "Record the final number" chain, Accepted band |
| `settlementStatement` | LWC | "Agreed, waiting on the check" section, "Check arrived" button, Date Received required |
| `Demand__c.Accepted_Amount__c` | Custom field (Currency), new | The agreed figure everything else reads |
| `Settlement_Recovery__c.Demand__c` | Custom field (Lookup), new | Links a recovery to the demand it settles |
| `Settlement_Recovery__c.Date_Received__c` | Custom field | Help text only |
| `Demand_Tracker`, `Settlement_Statement` | Permission sets | FLS for the two new fields |

**Out of scope**

- **The 4 existing Pending recoveries.** Firm operations cleaned those up separately. Nothing in this change edits data.
- **A validation rule on `Settlement_Recovery__c`.** Deliberately not used (see 6.3).
- **`Matter__c.Demand_Phase__c`**, which still has no writer.

## 4. How it works (background)

Before the change:

```
Demand Tracker (demandTracker LWC, DemandController)
  Demand__c  -> Offer__c rows (rounds, either direction, optional Is_Final__c)
  Close as Accepted = phase badge -> pick "Close — Accepted" from nine options -> date + notes
     (no amount, Offer__c untouched, no field for the agreed figure)

Settlement Statement (settlementStatement LWC, SettlementStatementController)
  Settlement_Recovery__c rows = checks received -> feed the trust ledger
  (reads nothing from Demand__c or Offer__c)
```

The two screens did not know about each other, so someone working the statement could not see a deal had been agreed. That is why placeholder recoveries were typed.

## 5. Root cause

Two gaps together:

1. **Accepting a demand was hard and amount-less.** `markOutcome` never touched `Offer__c`, and `Demand__c` had no agreed-amount field. Nothing tied "Accepted" to a dollar figure, so the feature went unused (0 of 30).
2. **The settlement statement could not see agreed deals,** and a recovery could be saved with no Date Received. The easiest way to record "deal agreed" was a placeholder recovery, which is the wrong record.

## 6. Solution

### 6.1 Accept an offer (Demand Tracker)

- **An Accept button on every offer** in the timeline, in both directions (a carrier can accept our counter). Hidden once the demand is closed. The offer's delete icon moved to a footer row to make room.
- **The Accept modal** shows the amount (read-only), carrier and coverage line, an Accepted Date (defaults to today, cannot be in the future) and Notes. One button: **Accept $X**.
- **`DemandController.acceptOffer(offerId, acceptedDate, notes)`** runs in one transaction with a savepoint. It marks that offer `Is_Final__c`, clears any other Final offer on the demand, closes the demand as Accepted, and stamps `Demand__c.Accepted_Amount__c` from the offer amount.
- **`markOutcome` and `acceptOffer` share one `closeDemand` helper,** so they cannot drift apart on what closing stamps.
- **No Accepted demand without an amount.** "Close — Accepted" in the phase dropdown opens the same Accept modal with an offer picker. With no offers it says to record the offer first. `markOutcome('Accepted')` is refused on the server, so the dropdown cannot be used as a back door.
- **"Record the final number"** chains into Record Offer and back into Accept on the new offer, for a deal that closed at a number not yet on the timeline.
- **"Accepted: $X on [date]"** band across the top of an Accepted demand.

### 6.2 "Agreed, waiting on the check" (Settlement Statement)

- **A section above the recovery list,** one line per Accepted demand on the matter that no recovery points at: carrier, coverage line, accepted date, days waiting, accepted amount. It never counts toward gross.
- **"Check arrived"** opens the existing Add Recovery modal pre-filled with amount, carrier, coverage and coverage line. The user adds Date Received and the check number.
- **Matching uses the new lookup `Settlement_Recovery__c.Demand__c`,** set only by that button. Never by amount, because checks come split or a few cents off.
- **A difference is shown, never blocked,** under the amount in the modal (for example a lien paid directly by the carrier).
- A demand's policy can belong to a companion matter on the same incident. The pre-filled modal adds that coverage as an option, or "Check arrived" would show an empty Coverage box.

### 6.3 No recovery without a Date Received

Enforced in **`SettlementStatementController.createRecovery` and `updateRecovery`**, not in a validation rule:

- Blank: *"Enter the date the check arrived. If the check isn't here yet, record the agreement in the Demand Tracker instead."*
- Future: *"Date Received cannot be in the future."*
- `updateRecovery` checks the **stored** date when the payload leaves it out, so the leftover dateless Pending rows must get a date the next time anyone edits them.
- Date Received is marked **required** in the modal, and its help text says where an expected check goes.

**Why not a validation rule.** A rule would break existing test fixtures that insert recoveries with no date, and a trust-reconciliation test that covers old dateless rows on purpose. The UI is the only path that creates recoveries, so enforcing it in the save path covers every real case.

### 6.4 Field access

| Field | Permission set | Access |
|---|---|---|
| `Demand__c.Accepted_Amount__c` | `Demand_Tracker` | Read only. Only `acceptOffer` writes it. |
| `Settlement_Recovery__c.Demand__c` | `Settlement_Statement` | Read/edit, matching the sibling lookups |

## 7. Testing

**Apex.** `DemandControllerTest` +7: accept marks Final, clears the other Final, stamps the amount, defaults the date, accepts a firm counter-offer, refuses a closed demand, a future date and no offer, refuses the dropdown path (`markOutcome('Accepted')`), and the snapshot returns the amount. `SettlementStatementControllerTest` +10: missing, future and valid dates on create and update, today as the boundary, an old dateless row edited without a date, and the waiting list (shown, excluded for an open demand, cleared by a linked recovery).

**Jest.** `demandTracker` 29 to 41 tests, `settlementStatement` 22 to 31.

**Mutation checks.** Each behaviour was broken on purpose and the suite re-run: **11 of 11** caught in JavaScript, **9 of 9** caught in Apex (in the sandbox).

**Production deploy.** 2026-10-07, with `DemandControllerTest`, `SettlementStatementControllerTest`, `SettlementRecoveryTriggerHandlerTest`, `SettlementStatementPdfControllerTest` and `TrustReconciliationControllerTest`: **236 of 236 passed**.

**On screen.** The full path was clicked through end to end in the sandbox, per the ticket: record an offer, accept it, see the waiting line on the statement, click Check arrived, try to save without a date (blocked), save with one (line disappears). The PI team was asked to try it once on a real matter with a PI user's login; their result is not recorded.

**Not covered:** the Apex validation compiles and runs tests but renders no component, so the screens were covered only by Jest and the sandbox click-through.

## 8. Step by step

1. **Measure the problem** in production: demands by Phase, and Pending recoveries with their created and received dates.
2. **Add the two fields** (`Demand__c.Accepted_Amount__c`, `Settlement_Recovery__c.Demand__c`) and their FLS on `Demand_Tracker` and `Settlement_Statement`.
3. **Add `acceptOffer`** and the shared `closeDemand` helper in `DemandController`. Make `markOutcome` refuse Accepted.
4. **Add the Accept button, modal, chain and band** in `demandTracker`.
5. **Add the date checks** to `createRecovery` and `updateRecovery`, and the waiting list to `SettlementStatementController`.
6. **Add the waiting section and "Check arrived"** to `settlementStatement`; mark Date Received required.
7. **Write the Apex and Jest tests,** and mutation-check them.
8. **Click the full path in the sandbox** (see 7).
9. **Validate against production** with the five test classes in 7, get the PR approved, deploy. No post-deploy steps: FLS rides on the permission sets.
10. **Ask a PI user to try it** on a real matter.

## 9. Rollback

Redeploy the previous versions of `DemandController`, `DemandControllerTest`, `SettlementStatementController`, `SettlementStatementControllerTest`, the `demandTracker` and `settlementStatement` bundles, both permission sets, and `Date_Received__c` (help text only). The two new fields can stay: nothing reads them once the code is reverted, and `Accepted_Amount__c` values written meanwhile are harmless. Deleting them needs a destructive deploy and drops that data.

## 10. Known limits

- **Dateless recoveries created before the change are still there** until someone edits them, at which point a date is demanded.
- **Recoveries created outside the UI** (one-off scripts) are not checked, because the rule lives in the controller, not in a validation rule.
- **Matching is by the lookup only.** A recovery added through the plain Add Recovery button, not "Check arrived", is not linked, so the waiting line stays until it is.

## 11. Reuse for similar issues

**How to recognise one:** users entering a "placeholder" record in the wrong object because the right one is hard to reach or cannot hold what they need.

- **Ask what the placeholder is standing in for,** and make that action easy in the right place.
- **Show the state across screens.** The statement not seeing agreed deals was half the cause.
- **Enforce in the save path when a validation rule would break legitimate old data or tests,** and say why in the PR.
- **Close the back doors on the server,** not only in the UI (here, `markOutcome('Accepted')`).
- **Match records by an explicit link, not by amount.**
