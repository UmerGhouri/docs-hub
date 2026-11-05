# Trust Tie-Out: seven fixes from the first live sign-off


| | |
|---|---|
| **Ticket** | DDT-1312 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-10-26 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom fields, permission sets, data backfill) |
| **Area** | Trust Tie-Out reconciliation screen |
| **Related** | DDT-863, DDT-898, DDT-1150, DDT-1314, DDT-1416 |
| **Code** | PR #707, #714 and #762 in the ddt-salesforce repo |


---


## 1. Summary

Firm leadership tested the deposit matching from DDT-898 live with a real trust bank file on 2026-10-08. It tied and was signed, but the session produced seven follow-ups where the screen said something was wrong when nothing was, or counted something twice. All seven were fixed in PR #707 (live 2026-10-13). A second pass (PR #714) stopped step 3 from requiring a click on a false statement. The last fix, marking three pre-trust-bank entries as belonging to the old trust account, needed a guarded data backfill that ran on 2026-10-20 after approval. Firm leadership confirmed the final fix on the screen on 2026-10-25.

## 2. What was reported

- 2026-10-12: split from DDT-898. In priority order: (1) money the bank is holding shows as a red difference; (2) a deposit was refused as ambiguous because of older same-amount credits; (3) after Save, saved deposits are counted twice; (4) payments the bank combined into one debit show as "no record in Salesforce"; (5) "bank file is N days old" measures the last confirmed ACH, not the file; (6) entries from the old trust account can never match the current bank's file; (7) after sign-off, the page reloads looking like an unworked day.
- 2026-10-13 (comment): after a real run on the new screen, step 3 could only be passed by clicking "None of these have landed yet", which was false for credits just explained.

## 3. Scope

**In scope**

- `TrustReconciliationController`, `TrustReconciliationControllerTest`, `TrustLedgerEntryTriggerHandler` and its test, LWC `trustTieOut`.
- New fields: `Trust_Reconciliation__c.Bank_Holds__c`, `Trust_Reconciliation__c.Bank_File_Through__c`, `Trust_Ledger_Entry__c.Bank_Account__c`; `Difference__c` formula updated.
- Permission sets `Trust_Ledger_Admin`, `Trust_Ledger_User`, `Trust_Tie_Out`.
- Script `ddt1312_old_trust_account_backfill.apex` (guarded, dry run by default).

**Out of scope**

- Turning a late deposit red: raised during this ticket and moved to its own ticket, DDT-1416.
- Voiding or deleting old-account entries: they are part of the seven-year client ledger record.

## 4. How it works (background)

The reconciler types the bank's current and available balances in step 1 and uploads the bank CSV in step 2. The screen matches checks, payments and deposits against the file, and the difference is the bank's available balance less the Salesforce position, less payments not yet at the bank, less deposits at the bank but not yet confirmed. Sign-off stores the figures on `Trust_Reconciliation__c`.

## 5. Root cause

From the live session and the code:

1. Available balance excludes deposited checks the bank is holding, so held money looked missing.
2. Deposit eligibility looked both ways in the 14-day window, so a credit posted before a deposit was received counted as a candidate.
3. A credit that any waiting deposit could own skipped the "already landed" check, so a saved deposit's credit kept counting as waiting money at the bank and was offered to another matter.
4. The bank sometimes takes two same-client, same-day payments as one ACH; the matcher only compared single entries.
5. File age came from `MAX(Bank_File_Seen_On__c)` on non-check debits, which is the last ACH the bank confirmed.
6. Three February entries went through the previous trust account before the current one opened in April 2026, so no current bank file could ever contain them.
7. The page rebuilt from zero on load instead of reading a signed day back from its record.

## 6. Solution

**PR #707 (deployed 2026-10-13).**

1. **Held by the bank** = current minus available minus the file's pending debits. Its own tile, stored on `Bank_Holds__c`, added back in all three places the difference is computed (the LWC, `signOff` and the `Difference__c` formula). Zero when no file was read, because without pending debits a hold cannot be told from a payment going out; never negative.
2. Deposit eligibility is directional: received on or before the credit's post date, within the window.
3. Landed deposits claim their credits first, but only when the file covers their landed date. If the re-read after Save fails, the file's deposit figure is cleared rather than counted twice. This reverses a rule from PR #648; a wrong hand-typed landed date now shows loudly as an unplaced deposit instead of a hidden double count.
4. A bank debit equal to one matter's same-day payments is recognised: not raised again if already confirmed, otherwise offered for confirmation together.
5. File age comes from the file's latest post date, stored on `Bank_File_Through__c`.
6. New `Bank_Account__c` picklist on ledger entries (current trust bank by default, plus "Old trust account"). Old-account entries are left out of bank-file matching, the covered window, `paymentsNotExamined` and the in-flight figure, but stay on the client ledger and in the position. The field is locked after creation, like Amount, because flipping a payment to the old account would hide it.
7. A signed day opens as done: signed figures read back from the record, "Signed off by X at time" and "Re-open to redo" (no re-open on a finalized day).

**PR #714 (2026-10-14): step 3 settles by explaining.** Once a file is read, step 3 is done when every raised credit has a non-blank reason and no waiting deposit the file shows is left unsaved. Ambiguous or combined credits still need a pick and Save. "None of these deposits have landed yet" only appears before a file is read, beside the deposits table. Each reason box says "Saved when you sign off."

**Fix 6 backfill (run 2026-10-20).** Backup taken, dry run found exactly the three expected entries on one matter, then the live run on firm leadership's approval. The backup showed the three rows had a blank `Bank_Account__c`, not the default (a picklist default applies only to rows created after the field existed), so PR #762 corrected the undo instruction to "clear the field" rather than "write the default". Exactly three rows were modified; a second accidental run found nothing to change and refused.

## 7. Testing

- PR #707: Jest 123 of 123 (16 new, asserting on rendered text; three mutations caught). Apex: 20 new tests; production check-only validation 197 of 197 across five trust test classes. Production pre-checks confirmed the new fields did not exist and exactly three live entries pre-dated the current bank.
- PR #714: Jest 131 of 131 (8 new), four mutations caught. LWC only.
- PR #762: comments only; nothing deployed.
- Live: firm leadership and finance staff ran a real file on 2026-10-13 and it tied; the developer's 2026-10-19 comment records the six code fixes as confirmed from that run. Fix 6 was confirmed on 2026-10-25: the "2 older payments predate every bank file" note was gone and the entries were still on the matter.
- Rule from DDT-863: a green pipeline on a screen change proves nothing; done meant a real file through the screen.

## 8. Step by step

1. Collect issues from a real sign-off with a real file, in priority order.
2. Add the hold figure from data the reconciler already types, and add it back wherever the difference is computed.
3. Make matching windows directional and let already-landed items claim their bank lines first.
4. Recognise same-matter, same-day sums on the payments side.
5. Take file age from the file itself and store it.
6. For entries that can never match, add an attribute that excludes them from matching but keeps them on the ledger; backfill with backup, dry run, approval, live run.
7. Reload a signed day from its record.
8. Never require a click on a statement that is false to advance a step.

## 9. Rollback

Redeploy the previous controller, trigger handler and `trustTieOut` bundle; the new fields can stay. Fix 6 is undone by clearing `Bank_Account__c` on the three rows, using the backup taken at run time, not by writing the default.

## 10. Known limits

- Bank holds are only computed when a file with the pending status column is read.
- Combined payments are recognised only for one matter on one day.
- Reasons typed on a day are not reloaded if the same day is signed again; they must be retyped.

## 11. Reuse for similar issues

- Signs of the same problem: a reconciliation screen shows a red difference that matches a held, pending or in-transit amount to the penny. Model that amount explicitly.
- A control that only moves on when someone clicks a false statement teaches people to click without reading.
- Picklist defaults do not backfill; check the real value before writing an undo instruction.
- Guarded backfills: refuse unless the exact expected rows are found, dry run first, back up at run time.
