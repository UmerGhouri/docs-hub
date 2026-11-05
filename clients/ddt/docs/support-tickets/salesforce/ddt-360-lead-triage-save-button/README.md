# Lead Triage missing Save button on the Case-Related Party tab

| | |
|---|---|
| **Ticket** | DDT-360 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-17 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Lead Triage: Case-Related Party tab |
| **Related** | None recorded |
| **Code** | None. Answered as working by design |

---

## 1. Summary

On the Lead Triage screen, the **Case-Related Party** tab has no Save button, and staff thought they were stuck. There is no separate Save by design: on that tab, choosing a row action on an existing contact, or clicking **Create** in the new-party window, saves straight away. The support team explained the workflow and nothing was changed.

## 2. What was reported

A staff member triaging a call searched for an existing client, reviewed the client's court cases, and then found no button to save the triage outcome. The other Lead Triage tabs have their own submit actions, so the missing button looked like a bug. In this case the caller was a prosecutor who had emailed about a client, and that prosecutor was also listed on other clients' cases.

## 3. Scope

**In scope:** an explanation of how the tab saves. No component changed.

**Out of scope**

- **Adding a Save button.** Not done. It would duplicate the actions that already commit the record.

## 4. How it works (background)

```
leadTriage (LWC)
  tab "Case-Related Party" -> caseRelatedParty (LWC)

caseRelatedParty
  1. Search for the client (searchClients) and pick one
  2. Pick the court case or matter (getClientCasesAll)
  3. The related contacts on that case are listed (getRelatedContacts)
     a. Person already listed -> row action menu:
          Replace Primary Phone Number
          Add as Secondary Number
          Temporary / Do Not Save
        -> saveChanges, then linkLeadToContactAndDelete
     b. Person not listed -> "Create New Case-Related Party"
          -> window with Name, Phone, Email, Role (and optional business)
          -> Create -> createParticipant, then linkLeadToContactAndDelete
  4. The Lead is linked to the contact and deleted; the screen moves to the case
```

## 5. Root cause

Not a fault. The tab commits work through its actions rather than a Save button:

- Choosing a **row action** on an existing contact saves the change and links the Lead to that contact.
- Clicking **Create** in the new-party window creates the contact on the case with the chosen role, and links the Lead to it.

Either one ends the triage, so nothing else is needed. The screen gives no hint of this, which is why it looked broken.

## 6. Solution

The support team replied in the thread on 2026-07-16:

- If the prosecutor is already listed on the case, pick them and choose the row action.
- If not, click **Create New Case-Related Party**, fill in the role (Prosecutor), and click **Create**. That Create button is the save. Once done, the party is committed to the case.

## 7. Testing

Not applicable: no change was made. Whether the user confirmed the explanation is not recorded in the ticket.

## 8. Step by step

1. On the Lead, open Lead Triage and choose **Case-Related Party**.
2. Search for the client the caller is calling about, and select them.
3. Select the court case or matter the call relates to.
4. If the caller is already in the list, open the row's action menu and choose the action that fits their phone number.
5. If not, click **Create New Case-Related Party**, check the Name, Phone and Email copied from the Lead, choose a **Role**, and click **Create**.
6. Confirm the success message and that the screen moves to the case.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The tab never shows a Save button**, so new staff will ask the same question. A short hint on the screen would prevent it; none has been added.
- **Completing the tab deletes the Lead** once it is linked to the contact. Choose the right client and case before acting.
- **A row action fails if the Lead has no RingCentral call log.** The component shows "No RingCentral Call Logs were found for this Lead" and stops.

## 11. Reuse for similar issues

**How to recognise one:** "there is no Save button" on a custom screen where other similar screens have one.

- **Read the component before building anything.** Look for buttons and menu items that call Apex. One of them may already be the save.
- **Check what the action does after saving**, here linking and deleting the Lead, so the explanation to the user is complete.
- **Answer in the thread with the exact click path**, using the labels on screen.
