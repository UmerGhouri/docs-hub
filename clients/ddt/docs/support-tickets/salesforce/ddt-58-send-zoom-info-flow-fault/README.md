# Send Zoom Info flow fails with an unhandled fault

| | |
|---|---|
| **Ticket** | DDT-58 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flow, quick action), Heymarket text messaging |
| **Area** | Court Cases: sending a client the Zoom details for a court appearance |
| **Related** | None recorded |
| **Code** | None. The flow was not changed. |

---

## 1. Summary

Clicking **Send Zoom Info** on one Court Case always ended in Salesforce's generic "An unhandled fault has occurred in this flow" message, so the client could not be sent the Zoom details for their hearing.

The resolution note gives the cause as "a date issue". The flow itself was not modified, so the fix was made outside the flow. What exactly was changed is not recorded in the ticket.

## 2. What was reported

On one Court Case, every attempt to use **Send Zoom Info** failed with:

> An unhandled fault has occurred in this flow. An unhandled fault has occurred while processing the flow. Please contact your system administrator for more information.

It was recurring on that record. The ticket asked whether other Court Cases were affected; the answer is not recorded. Only one record is named.

Why it matters: the action is how staff send a client the Zoom details for an upcoming court appearance. While it fails, the client has to be sent them by hand.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Send_Zoom_Meeting_Info` | Quick action (label "Send Zoom Info") | None |
| `Court_Appearance_Assignments_Send_Zoom_Codes` | Screen flow | None. Production shows it last modified on 2026-05-21, before this ticket. |
| The date behind the failure | Record data | Corrected. Which record and field were changed is not recorded. |

**Out of scope**

- **Adding fault handling to the flow.** The acceptance criteria asked for a clear error message instead of the generic fault. That was not done (see section 10).

## 4. How it works (background)

Court Case is the relabelled `Opportunity` object. The **Send Zoom Info** button is a quick action that launches the screen flow **Court Appearance Assignments - Send Zoom Codes**:

```
Send Zoom Info (quick action on the Court Case)
  -> Get the Case Appearances: first Court_Appearance__c for this case where
       Court_Date_Today__c = true OR Future_Court_Date__c = true,
       sorted by Court_Date__c ascending
  -> Is the Zoom Info populated?   (Zoom_Meeting_Id__c not blank)
       yes -> Send screen -> "Proceed?"
                -> text the client (Heymarket action)
                -> Check Client Email -> email the client
                -> post "Zoom codes sent" to the case's Chatter
                -> Sent screen
       no  -> Check the Courthouse (County__c, Courtroom__c)
                -> DuPage / Lake / no appearance found / no code screens
```

The flow picks the appearance by its **date**: `Court_Date_Today__c` and `Future_Court_Date__c` are checkbox formulas on `Court_Appearance__c` calculated from `Court_Date__c`. The message it sends also prints the court date and time from the appearance's text date fields.

**The flow has no fault paths.** None of its elements has a fault connector, so any error inside it, from any element, ends in the same generic "unhandled fault" screen.

## 5. Root cause

Per the resolution note: "There was a date issue due to which user wasn't able to 'Send Zoom Info'."

Which date, on which record, and why it caused a fault are **not recorded in the ticket.** What can be established:

- The flow's active version was last modified in production on 2026-05-21, five days before the ticket, so the fix was **not** a flow change. Query used:

  ```sql
  SELECT ApiName, LastModifiedDate
  FROM FlowDefinitionView
  WHERE ApiName = 'Court_Appearance_Assignments_Send_Zoom_Codes'
  ```

- The flow's behaviour depends on the appearance's `Court_Date__c`, both to choose the appearance and to build the message.

## 6. Solution

The support team corrected the date problem on 2026-05-26 and closed the ticket. The record, field and value changed are **not recorded in the ticket.**

## 7. Testing

The acceptance criteria asked for Send Zoom Info to work end to end on the reported record and on a fresh test case. The ticket was closed as done; the test results are **not recorded in the ticket.**

**Not covered:** whether other Court Cases had the same bad date, and the flow's generic error message.

## 8. Step by step

The exact fix is not recorded, so these are the steps that diagnose this symptom:

1. **Get the real error.** The screen message hides it. Use the flow error email sent to the fault recipient, or Setup → Paused and Failed Flow Interviews, or a debug log for the user while they click the button.
2. **Find the element that failed.** The error email names it.
3. **Check the case's court appearances.** Look at `Court_Date__c` on each, and the formula checkboxes `Court_Date_Today__c` and `Future_Court_Date__c`. The flow only picks an appearance whose checkboxes are true.
4. **Check the chosen appearance's other inputs**: `Zoom_Meeting_Id__c`, `County__c`, `Courtroom__c`, and the client's phone and email on the case.
5. **Correct the bad value** on the record.
6. **Retry Send Zoom Info** on the same case and confirm the Chatter post "Zoom codes sent" appears.
7. **Write the field and value you changed on the ticket.**

## 9. Rollback

Not applicable: the change was a data correction on one record, and the previous value was the cause of the failure.

## 10. Known limits

- **The flow still has no fault handling.** Any future error in it, a bad date or anything else, will show the same generic message and need an administrator to read the error email.
- **The cause is not recorded**, so if another record hits the same date problem, nothing in the ticket says what to look for beyond "a date issue".

## 11. Reuse for similar issues

**How to recognise one:** a screen flow fails with "An unhandled fault has occurred in this flow" on one record but works on others.

- **The message is generic, not the cause.** Always read the flow error email or the failed interview before changing anything.
- **Record-specific failures are usually data.** Compare the failing record with one that works, field by field, starting with the fields the flow filters or formats.
- **Check whether the flow changed** (`FlowDefinitionView.LastModifiedDate`) before assuming the fix belongs in the flow.
- **Add fault paths** to user-facing flows, with a screen that says what is missing, so the next user can fix their own data.
