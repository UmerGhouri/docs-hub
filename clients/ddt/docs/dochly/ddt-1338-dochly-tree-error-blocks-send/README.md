# "Tree" error came back when sending the PI engagement agreement

| | |
|---|---|
| **Ticket** | DDT-1338 |
| **Project** | Dochly |
| **Status** | Done 2026-10-15 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature |
| **Area** | Sending the PI engagement agreement for signature |
| **Related** | None recorded |
| **Code** | None |

---

## 1. Summary

Sending the PI engagement agreement to one client through Dochly showed the "tree", an error state staff had seen before and which had been fixed once already. The support team fixed it the next day. The ticket does not record what caused it or what was changed.

## 2. What was reported

- Staff tried to send the PI engagement agreement to one prospective PI client through Dochly, and the "tree" appeared, blocking the send.
- The reporter's wording, "the tree is **back**", marked it as a regression of something fixed before.
- Staff explained that the tree usually appears on a document **when a field is missing**, and that this time it also added an **extra page** to the document.
- Business impact, from the ticket: until the agreement is sent, the hand-off to a prospective client is delayed.

One client and one document were named. The ticket does not record whether other sends were affected.

## 3. Scope

**In scope:** the PI engagement agreement as sent through Dochly for the affected client. The specific template, field or record that was changed is not recorded in the ticket.

**Out of scope:** Not recorded in the ticket.

## 4. How it works (background)

```
Staff open the Dochly send for the client's record
  -> Dochly merges the record's fields into the engagement agreement template
  -> the generated document is reviewed and sent for signature
```

According to the reporting staff, when a field the template expects is missing, Dochly shows the "tree" in the document instead of a normal result.

## 5. Root cause

Not recorded in the ticket. The ticket asked for the regression's cause to be noted "so we can watch for it next time"; the closing comment says only that it was fixed. The only evidence on the ticket is the reporter's observation that the tree appears when a field is missing.

## 6. Solution

Not recorded in the ticket. The support team's closing comment on 2026-10-15 reads, in full, that it is fixed now. No commit or pull request in the `ddt-salesforce` repo references DDT-1338, so the fix was not made in that repo's metadata.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for the reporter to confirm the send went through; that confirmation is not on the ticket.

## 8. Step by step

The steps used are not recorded. For a repeat, the ticket's own requirements give the order:

1. Reproduce the tree on the affected record, or on the engagement agreement template.
2. Find which field the template expects and is missing (the reporter's lead).
3. Fix it so the agreement can be sent through Dochly.
4. Confirm with the reporter that the send goes through.
5. Write down what caused the regression on the ticket.

## 9. Rollback

Not applicable: the change made is not recorded, so there is nothing defined to undo.

## 10. Known limits

- **It has now happened at least twice.** Without a recorded cause, there is nothing to watch for, and it can return a third time.
- The ticket does not record whether the extra page the tree added was also fixed.

## 11. Reuse for similar issues

**How to recognise one:** a Dochly document shows the "tree" or gains an unexpected extra page, especially on a record with an empty field the template uses.

- **Check the merge fields first.** Compare the template's merge fields with the record and look for an empty one.
- **Write down the cause on the ticket** when you fix it, even in one line. A regression with no recorded cause has to be diagnosed from scratch every time.
