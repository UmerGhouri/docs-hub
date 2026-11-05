# Autopay never charges more than the client owes


| | |
|---|---|
| **Ticket** | DDT-1398 |
| **Project** | DDT Pay |
| **Status** | Done 2026-10-28 (built and tested in the Partial Copy sandbox; not in production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata, tab and list views), Authorize.Net |
| **Area** | DDT Pay: autopay |
| **Related** | DDT-1252, DDT-249, DDT-1395, DDT-1315 |
| **Code** | PR #38 in the ddt-salesforce repo (the DDT Pay package branch, still open) |


---


## 1. Summary

DDT Pay's autopay charged the sum of installments due today without checking it against what the client actually owes. If the schedule had drifted, autopay would overcharge, even a client who owed nothing. Autopay now caps every charge at the invoice balance before calling the card processor, charges nothing if nothing is owed, and logs a reconcile exception so billing can repair the schedule. A Reconcile Exceptions tab and related list were added so those exceptions can be seen.

## 2. What was reported

- 2026-10-22: go-live scope for "Phase 1 live". The DDT-1252 tie-out found 13 invoices in the previous payments package (fw1) on autopay whose unpaid installments asked for more than the client owed; two of those clients owed nothing and still had an open installment. In DDT Pay, `AutopayService` read `Outstanding__c` but did not use it as a cap; the online pay page did. Required: every stored-card charge capped at what is owed, and a reconcile exception logged when the schedule asks for more.

## 3. Scope

**In scope**

- `AutopayService`: cap before the gateway call, exception, and a `Capped` autopay event.
- `Pay_Org_Config__mdt.Autopay_Cap_Disabled__c` (switch; unchecked means the cap is on).
- New picklist value `Capped` on `Pay_Autopay_Event__c.Action__c`.
- `Pay_Reconcile_Exception__c`: related list on `Pay_Invoice_Command_Page`, a tab in the DDT Payments app with Unresolved and All Exceptions list views, tab visibility in `Pay_Payments_Access`.
- Comment fixes in `PayProfileChargeService` and `PayStaffActionsController`.
- 6 new tests in `AutopayServiceTest`.

**Out of scope**

- Take a Payment (card on file and new card) and the online pay page: they already refused or clamped an over-collection, so they were verified, not changed.
- Repairing the schedule itself: that is DDT-1395 and DDT-1252 territory.

## 4. How it works (background)

Autopay runs unattended each morning and charges the stored card for installments due that day. A retry after a decline is not a separate path; it is the next morning's run through the same method. `Outstanding__c` on the invoice is what the client owes. The nightly reconcile already logs a **Balance Mismatch** exception when the schedule disagrees with the invoice.

## 5. Root cause

`AutopayService` selected `Outstanding__c` and never used it; it charged the sum of installment balances due today regardless of the real balance. An earlier validation rule (`Collected_Cannot_Exceed_Contracted`) tried to block over-collection, but it fired on the roll-up after the card had been charged. A card charge is a callout and does not roll back, so its best outcome was money taken and an invoice refusing to record it. It was removed on 2026-10-26 and the problem moved to this ticket.

## 6. Solution

- **Cap before the callout.** Each charge is capped at `Outstanding__c` before the card processor is called. If the client owes nothing, no call is made at all.
- **Exception, reusing Balance Mismatch.** A capped run writes the same exception type and uses the same de-duplication as the nightly check, because it is the same defect seen from two ends. A broken invoice carries one open exception whichever check finds it first.
- **Somewhere to see it.** On first build the exception was written where nobody could read it (no tab, no related list). The related list and tab were added, beyond the ticket's wording.
- **Inverted switch.** The switch is named "Cap Disabled" so a fresh deploy leaves the cap on with no post-deploy step. A switch named "enabled" would deploy unchecked and leave charges uncapped.
- **Staff paths refuse, autopay caps.** A person at a screen can investigate; autopay runs with nobody watching, so it takes what is owed and leaves a flag.
- During the merge with DDT-1315 (merchant routing) the two edits met on the same line of `AutopayService`; both were kept, the cap deciding how much and the routing deciding which merchant.

## 7. Testing

Break test on 2026-10-26 in the Partial Copy sandbox against the Authorize.Net sandbox, with a throwaway account and a sandbox test card. The full autopay batch was not run; only the test invoices were charged.

| Cap | Client owes | Due today | Charged | Exceptions |
|---|---|---|---|---|
| On | $0 | $500 | $0, no gateway call | 1 |
| On | $250 | $750 | $250 | 1 |
| Off | $0 | $500 | $500 | 0 |
| Off | $250 | $750 | $750 | 0 |

- The switch was set back right away and confirmed by query.
- Apex: `AutopayServiceTest` 14 of 14, `AutopayService` 96% coverage. New tests cover owes nothing, partial room, a healthy invoice not capped, cap off overcharging, no duplicate exception across runs, and 251 capped invoices with zero callouts.
- In the sandbox on the day, the defect was real in code but not firing: no autopay-live invoice had an overcharge due that morning. That is why the scenario was built rather than found.
- On 2026-10-28 a DDT-1315 sandbox deploy overwrote the class and dropped the cap in the sandbox; the merge restored it and it was re-verified.
- Not covered: production.

## 8. Step by step

1. Find every path that charges a stored card and check whether it caps or refuses.
2. In the unattended path, cap the amount at the balance before the gateway call.
3. Log a reconcile exception on the invoice, reusing the existing type and de-duplication.
4. Record a `Capped` event in the invoice's autopay history.
5. Make sure the exception is visible on the invoice page and in a working list.
6. Add a switch whose safe state is the default, and break-test with it both ways.

## 9. Rollback

Tick `Autopay_Cap_Disabled__c`. Autopay returns to charging the full amount due, and the exception and Capped event stop with it. Do not tick it outside a break test.

## 10. Known limits

- A capped invoice is not paused. It keeps being picked up each day, makes no gateway call, and de-dupes onto the same exception. Whether it should pause and show on a worklist was left for a decision.
- The same broken invoice behaves differently by path: a staff user gets a refusal, autopay takes a partial payment. Deliberate, awaiting confirmation.
- The cap stops the overcharge but does not repair the schedule.
- Day one in production will bring a backlog of exceptions. In the sandbox the nightly job had flagged 23 Balance Mismatch exceptions, matching the 23 of 53 invoices that did not tie out (the deploy doc gives 22 of 53).

## 11. Reuse for similar issues

- Sign of the same problem: an automated charge computed from a schedule, with the real balance read but never used.
- Put money checks before the callout. A validation rule after the charge cannot undo it.
- For a safety switch, make the safe state the default value.
- An exception nobody can see does nothing; add a tab or related list with it.
- Run the break test against the processor's sandbox with throwaway records, and only for those records.
