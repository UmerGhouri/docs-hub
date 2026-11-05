# Suppressing countersign reminders for test records

| | |
|---|---|
| **Ticket** | DDT-1304 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-13 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature, Salesforce |
| **Area** | Representation Agreement countersignature reminders |
| **Related** | None recorded |
| **Code** | None. Done in Dochly; no PR |

---

## 1. Summary

A firm administrator kept getting reminder emails asking them to countersign Representation Agreements that belonged to test records. The reminders came from Dochly, because five test contracts were still waiting on the administrator's countersignature. Voiding those five contracts in Dochly stopped the reminders.

Voiding them had a side effect: two test Matters were re-created. Those were removed by hand.

## 2. What was reported

On 2026-10-12 the administrator reported repeated emails with the subject pattern `Reminder: "<Rep Agreement Name>" - please review and sign`, all for test records. They asked for the reminders to stop, and ideally for test records never to trigger them.

Later the same day, after the fix, they reported that the test cases had reappeared as hired Matters.

## 3. Scope

**In scope**

| Item | Where | Change |
|---|---|---|
| 5 test contracts awaiting countersignature | Dochly signature requests | Voided |
| 2 test Matters re-created by the voiding | Salesforce `Matter__c` | Removed |

**Out of scope**

- **A permanent "test record" opt-out.** The ticket asked for this as an ideal. Nothing was built; voiding the pending requests was enough to stop the emails.
- **Changing what happens when a contract is voided.** The support team said it would fix this so voiding no longer re-creates Matters. That follow-up is not recorded in this ticket.

## 4. How it works (background)

```
Representation Agreement sent through Dochly
  -> client signs
  -> request waits for the firm's countersignature
  -> Dochly sends "please review and sign" reminders to the countersigner until it is signed or voided
```

The ticket first guessed the reminders came from a Salesforce flow or email alert. They came from Dochly itself.

## 5. Root cause

The five test contracts were real Dochly signature requests that had never been finished. Each was still waiting on the administrator's countersignature, so Dochly kept reminding them, as it would for a real contract. Nothing marks a contract as "test", so Dochly treats both the same way.

**The side effect.** Voiding the five contracts re-created two test Matters as hired Matters. Why voiding does this is not recorded in the ticket.

## 6. Solution

1. Found the pending signature requests where the administrator was the outstanding countersigner. There were 5, all test contracts.
2. Voided all 5 in Dochly. Reminders stop once a request is no longer pending.
3. Removed the 2 test Matters the voiding had re-created.

Why voiding rather than a filter: the reminders come from Dochly, not from Salesforce automation, so there was no Salesforce condition to add. Voiding ends the request at its source.

## 7. Testing

- The administrator acknowledged the fix on 2026-10-12.
- No automated test applies; this was a data action in Dochly.
- **Not covered:** whether voiding a contract still re-creates Matters. Not recorded in the ticket.

## 8. Step by step

1. Get the subject line or record names of the reminder emails from the person receiving them.
2. In Dochly, list signature requests still waiting on that person's countersignature.
3. Confirm with the reporter that each one is a test record.
4. Void each test request.
5. Check Salesforce for Matters created or re-created at the time of the voiding, and remove any that belong to test records.
6. Ask the reporter to confirm the reminders have stopped.

## 9. Rollback

Not applicable: a voided Dochly request cannot be reinstated, and these were test contracts with no reason to restore them.

## 10. Known limits

- **Every new test contract can do this again.** Nothing stops a test contract from sending reminders. Finish or void test requests when the test is done.
- **Voiding can re-create Matters.** Until that is fixed, check for re-created Matters after voiding any contract.

## 11. Reuse for similar issues

**How to recognise one:** reminder emails about a document nobody intends to sign, usually from testing.

- **Check the e-signature vendor first.** Reminder emails about pending signatures normally come from the vendor's own reminder schedule, not from Salesforce automation.
- **End the request at the source** by voiding it, rather than trying to filter emails.
- **Look for side effects** after voiding: anything that listens to signature status changes may run.
