# Bank file deposits now fill the landed date in reconciliation step 3


| | |
|---|---|
| **Ticket** | DDT-898 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-10-12 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom object) |
| **Area** | Trust Tie-Out reconciliation screen, step 3 (deposits) |
| **Related** | DDT-897, DDT-1150, DDT-1312 |
| **Code** | PR #511 and #648 in the ddt-salesforce repo |


---


## 1. Summary

On the Trust Tie-Out screen, step 2 uploads the trust bank's CSV export and step 3 confirms that each waiting settlement deposit has landed. Payments were already matched from the file, but deposits were not: the browser read the credit rows and threw them away, so a person had to type each landed date that the file already held. Step 3 blocks money going out on those matters, so this stalled the whole reconciliation (DDT-1150). The fix matches credits to waiting deposits, proposes the landed date without saving it, and refuses to guess when two matters could own the same credit. A follow-up after the first live test separated deposits in transit from the difference, stopped false alarms on credits already accounted for, and added a written "Mark as explained" for credits no rule can close.

## 2. What was reported

- 2026-09-01: raised by the firm's finance staff in training and confirmed by firm leadership on the live screen the same day: an uploaded file contained a deposit, and step 3 still asked for its landed date. In the code, the parser comment said credits "are read but never sent", and the Apex matcher `matchBankFile` never looked at `Settlement_Recovery__c`.

## 3. Scope

**In scope**

- `TrustReconciliationController` (`matchBankFile`, deposit matching, `signOff`) and its test class.
- LWC `trustTieOut` (js, html, css, Jest tests).
- PR #648: new object `Trust_Credit_Explanation__c` (amount, bank date, reason, reconciliation) with permission set `Trust_Credit_Explanations`; new `Unconfirmed_Deposits__c` on the reconciliation.

**Out of scope**

- Deposits with no dates at all: no file can match them. The screen now says so above the table; finding out what happened to them is a business task, not code.
- Retiring the redundant `Date_Deposited_To_Bank__c` / `Date_Deposited__c` pair: to be checked separately before any delete.
- Payments the bank combined, bank holds and file age: carried into DDT-1312.

## 4. How it works (background)

A settlement payment received for a client is a `Settlement_Recovery__c`. Setting its landed date mirrors it into the trust ledger as a credit against the matter, so a wrong date puts one client's money on another client's ledger. A bank CSV credit row carries only an amount and a post date; the description is a generic label, not the payer. The screen finds columns by header name, not position.

## 5. Root cause

The deposit half of the bank file had never been used: the browser discarded credits after cancelling interest sweeps, and Apex never matched deposits. The ticket's own proposed anchor ("the date the deposit went to the bank") could not work either: that field was empty on every waiting deposit, and on the 12 records where it was filled it had never once differed from the landed date. `Date_Received__c` is the anchor instead.

After the first build, the live test on 2026-10-06/07 found three more problems in the code:

- The screen compared every credit in the file, back to the oldest, against only the deposits still waiting, so a credit already matched months earlier was called "no deposit waiting".
- The "two deposits add up to it" check ran before the "already settled" check, so it paired old credits with unrelated waiting deposits.
- The difference subtracted every waiting deposit, including ones handed to the bank but not yet posted, so a sign-off would store a large false difference.

## 6. Solution

**PR #511 (deployed 2026-09-28): read the deposits.**

- Leftover credits (after interest sweeps cancel out) are sent to Apex.
- The approved rule: when N credits of one amount share a post date and exactly N deposits of that amount are waiting, assign them in any order, because every pairing writes the same date. Anything else is reported for a person to decide, naming the candidate matters.
- Window: credits within 14 days of `Date_Received__c`, the generous end of the account's own history.
- Nothing is written. The file fills the box and shows the bank's date beside it; a person presses Save. Same contract as checks and payments.
- A credit matching no single deposit is checked against pairs of waiting deposits and reported, never matched (the bank sometimes combines what Salesforce splits). Pairs only; larger combinations show as unknown.

**PR #648 (deployed 2026-10-07): in transit, false alarms, explained credits.**

- The difference subtracts only waiting deposits the bank file shows. A deposit handed to the bank but not in the file is "in transit": shown as information and left out of the difference. Here a deposit reaches the ledger only once it has a landed date, so in transit is in neither the bank nor the Salesforce position; it must not be added anywhere.
- A credit already accounted for by a landed deposit near its date stays quiet, and that check runs before the pair check. Pairs are only offered from deposits received on or before the credit date.
- **Mark as explained:** a written reason, stored on `Trust_Credit_Explanation__c` with that day's sign-off and shown as explained on later uploads.
- `signOff` re-matches the file's credits on the server rather than trusting the figure the browser sent.

## 7. Testing

- PR #511: 12 new Apex tests, 54 of 54 in the production validation; 8 new Jest tests, 92 of 92. Two mutations checked by hand: not sending the credits fails two tests, not pre-filling the landed box fails one.
- PR #648: Apex and Jest tests for each rule; test counts are not recorded in the PR.
- Live: firm leadership and finance staff ran a real export on 2026-10-06 (found the false alarms) and again on 2026-10-08 after PR #648: the reconciliation tied and was signed. Seven follow-ups from that session moved to DDT-1312, and the ticket closed on 2026-10-12.
- No real export was ever attached to the ticket or the repo, by rule; only the header row shape was discussed.

## 8. Step by step

1. Send the file's credit rows (after removing those that cancel interest sweeps) to Apex alongside the debits.
2. Query waiting deposits; anchor on the received date, with a window that only looks forward from it.
3. Let landed deposits claim their credits first, so an accounted-for credit raises nothing.
4. Auto-propose only when every possible pairing writes the same date; otherwise list the candidates.
5. Check unmatched credits against pairs of waiting deposits and report them.
6. Keep in-transit deposits out of the difference.
7. Let a person write a reason for a credit no rule can close, stored with the sign-off.
8. Pre-fill; never save.

## 9. Rollback

Redeploy the previous `TrustReconciliationController` and `trustTieOut` bundle. PR #511 writes no data and adds no field. PR #648's explanation object can stay (additive); its rows are part of the reconciliation record.

## 10. Known limits

- Amount and date are the only key; two deposits of the same amount on different days still need a person.
- Combinations of three or more deposits in one credit are not detected.
- Deposits with no dates can never be matched from a file.

## 11. Reuse for similar issues

- Signs of the same problem: a screen asks a person to type a value that an uploaded file already contains. Check whether the parser drops those rows.
- Where a wrong match moves money between clients, propose and let a person save; refuse rather than guess.
- "Every pairing gives the same result" is a safe way to resolve ties without a decision.
- Check the "already accounted for" case before looking for new explanations of a bank line.
- Keep a written-reason escape for differences no rule can close, stored with the sign-off.
