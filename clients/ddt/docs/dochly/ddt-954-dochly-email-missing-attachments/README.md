# Dochly emails on the record show no attachments

| | |
|---|---|
| **Ticket** | DDT-954 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-09 |
| **Priority** | High |
| **Reported by** | Support team, 2026-09-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Dochly email delivery |
| **Area** | Logging Dochly document emails on the Court Appearance timeline |
| **Related** | None recorded |
| **Code** | PR #335 in the `ddt-salesforce` repo |

---

## 1. Summary

When a document was emailed from Dochly on a Court Appearance, the email shown in the record's Activity timeline listed **no attachments**, even though the client received them. Every Dochly send writes two email records: Dochly's own, which holds the files but is attached to no record, and a copy made by our flow, which shows on the timeline but had no files.

The fix extends the flow so it finds Dochly's original email and moves its file links onto the copy that shows on the record.

## 2. What was reported

Sending a document by email from Dochly on a Court Appearance, with attachments selected, produced an Activity entry with no attachments. The Task created alongside it read `Attachment: --none--`. That text is Salesforce's own summary of the flow's email record, and it was accurate: that record had no files.

**How many:** 87 emails since 2026-08-01 with the disposition-letter subject (`[Update] Court Case for%`) had `HasAttachment = true` and no related record (measured 2026-09-08).

**Nothing was lost.** Each file stayed linked to Dochly's original email and to the sending user's own Files. It was not linked to the Court Appearance, and Dochly's original shows on no page, so the files could not be reached from the case.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Dochly_Delivery_Log_Email_Activity` | Record-triggered flow | Version 3 recorded in the repo as it ran in production, then version 4 adds the attachment move |
| `docs/deploys/ddt-954-dochly-email-attachments.md` | Review doc | New |

Version 4 adds these elements, hung off the end of the existing chain so a failure here cannot affect the email or the Task:

| Element | Type | What it does |
|---|---|---|
| `Get_Source_Email_Message` | Get Records | Finds Dochly's original email |
| `Source_Email_Found` | Decision | No match: end quietly |
| `Get_Source_Attachments` | Get Records | Reads the original's file links |
| `Copy_Each_Attachment` | Loop | Iterates the links |
| `Stage_Attachment_Link` | Assignment | Builds a new file link pointing at the copy |
| `Any_Links_Staged` | Decision | Skips the insert when there is nothing to add |
| `Create_Attachment_Links` | Create Records | One bulk insert |
| `Delete_Source_Links` | Delete Records | Removes the links from the original, making it a move |
| `Suppress_Attachment_Fault` | Fault path | Swallows faults on the new path only |

**Out of scope**

- **The 87 emails already affected.** Not backfilled. The move applies to new sends only.
- **Signature-request emails.** These are audited elsewhere and have no delivery record.
- **The duplicate email record itself.** Two records per send is still the shape (see 5).
- **Linking the files to the Court Appearance's own Files list.** Deliberately left out to keep the change to what was agreed.

## 4. How it works (background)

```
User sends a document by email from Dochly
  -> Dochly sends the email and writes EmailMessage #1
       (holds the files, no related record, shows on no page)
  -> Dochly delivery record is set to Sent
  -> Dochly_Delivery_Log_Email_Activity (version 3) runs
  -> flow inserts EmailMessage #2 with the related record set
       (Salesforce auto-creates a Task, so it shows in Activity)
       (eleven fields copied; no files)
```

| | Dochly's original | The flow's copy |
|---|---|---|
| `HasAttachment` | true, carries the PDFs | false |
| `RelatedToId` | blank, shows on no page | the Court Appearance |
| `ActivityId` | blank | the auto-created Task |
| `MessageIdentifier` | set | blank |

Dochly cannot set the related record itself, because it stores a document's parent as two text fields (object name and record Id), not as a lookup.

## 5. Root cause

1. Version 3 of `Dochly_Delivery_Log_Email_Activity` creates the second email record. Its `Create_Email_Message` element sets eleven fields (Bcc, Cc, From address, From name, HTML body, Incoming, Message date, Related to, Status, Subject, To) and does nothing about attachments. There is no file-link step, so the copy is empty by construction.
2. **Timing confirms it.** Sends at 09:41, 09:43 and 09:46 on 2026-09-07 produced only Dochly's original. Version 3 was activated at 10:10 UTC that day, and the next send was the first with a copy.

**Also found:** the repo held an older version of this flow that created a Task with a different description. No such Task existed in production. Version 3 had never been committed, so the PR first recorded it as it ran in production, with no behaviour change, before changing anything.

**The cleaner fix is not possible.** Setting the related record on Dochly's original (one record, real message identifier, files already attached) was tried first. The field describes as updateable, but Salesforce rejects the write after insert:

```
INSUFFICIENT_ACCESS_OR_READONLY, You cannot edit this field: [RelatedToId]
```

Verified with anonymous Apex in a sandbox. Do not retry it on the strength of the field's describe.

## 6. Solution

### 6.1 Find Dochly's original

The flow matches on **Subject + To address, within 60 seconds either side of the sent date**, newest first, first record only. Two filters do the real work:

- the related record **is** blank, and
- the message identifier is **not** blank.

The flow's own copy can never meet both, because it sets the first and never the second, so the flow cannot match its own record. It is still a time-window match: during testing, same-subject sends to the same recipient happened 2 minutes 47 seconds apart, so the window is deliberately narrower than that.

`dochly__Attachment_Names__c` was not used. It holds file names, not Ids, as newline-separated text, and Flow has no split function.

### 6.2 Move the file links, don't re-point them

The ticket first proposed a copy. The design changed to a **move** after testing:

- **Re-pointing an existing file link fails silently.** `LinkedEntityId` is creatable but not updateable. `Database.update` returns success with zero errors while the link stays where it was. A flow built that way would run green and change nothing.
- **Creating a new link on an existing email works, and deleting a link works.** Together, that is a move.

`Delete_Source_Links` hangs off the **success** connector of `Create_Attachment_Links`. Links are never removed unless they were first added, and any fault on either step goes to `Suppress_Attachment_Fault` and leaves both sides untouched. The files themselves are never at risk: a file survives its links, and each is also linked to the sending user.

### 6.3 Why each choice was made

- **The copy is kept, not replaced.** Inserting an email with the related record set is the only thing that creates the Task, and the Task is what Activity shows.
- **A separate fault path.** Reusing the existing one would have recorded an attachment failure as a Task-creation failure.
- **`Any_Links_Staged` guards the insert.** An email with no attachments is normal.
- **One bulk insert, not one per file.** A batch can send many deliveries in one transaction, and per-file inserts would multiply against the DML limit.
- **Failures are silent**, consistent with the rest of this flow.

### 6.4 The trade-off of a move

Dochly's original is the record of what was actually emailed. After the move it no longer lists the files that went out, and its `HasAttachment` stays true (the flag is not writable), so it claims attachments it cannot show. The additive alternative (leave the original's links in place) was considered and rejected on 2026-09-08. Reverting to it is deleting `Delete_Source_Links` and its connector.

A later investigation (2026-09-09) found that no custom component can sit on the email page, so only the standard Attachments list can ever show these files. The review doc records that this removes the original reason for the move while its cost is unchanged, and that a reviewer would be right to ask for the delete step to come out. The PR shipped with it.

## 7. Testing

**Validation.** Check-only deploy of the flow against production: succeeded, 8 of 8 named Apex tests passing. A sandbox could not be used: it had no Dochly objects at the time. This proved the flow **deploys**. It ran no flow interview, so it proved nothing about the match, the loop, the insert or the delete.

**Rendering, checked by opening the screen on 2026-09-09.** The open question was whether the email page shows linked files when `HasAttachment` is false on the copy. It does: the email record showed **Attachments (2)** listing both PDFs with their sizes, with `HasAttachment = false`. The flag drives the paperclip icon, not the related list.

**Live state (2026-11-05):** version 4 of `Dochly_Delivery_Log_Email_Activity` is the Active version in production.

**Not covered:**

- No automated test of the flow's match against a real pair of emails.
- A count of moved files after go-live is not recorded in the ticket.

## 8. Step by step

1. **Confirm the split.** Find a recent Dochly send and its two email records. One has `HasAttachment = true` and no related record; the other has the related record and no files.
2. **Record the running flow first.** Retrieve `Dochly_Delivery_Log_Email_Activity` from production and commit it as-is if the repo copy is older.
3. **Add the elements in section 3** after the existing chain, entered from the two points where the chain previously ended (`Auto_Task_Exists`'s default connector and `Update_Auto_Task`'s success connector).
4. **Validate check-only against production:**

    ```
    sf project deploy validate \
      --source-dir force-app/main/default/flows/Dochly_Delivery_Log_Email_Activity.flow-meta.xml \
      --target-org <prod-alias> \
      --test-level RunSpecifiedTests --tests <named test class>
    ```

5. **Get the PR approved, then deploy the flow as an inactive (Draft) version** so it can be activated deliberately. Confirm the new version is Draft and the old version is still Active.
6. **Activate the new version.** It applies to every Dochly send at once; it cannot be scoped to one record.
7. **Send one Dochly email with an attachment on a test Court Appearance**, open the Activity entry, and check the attachments list.

## 9. Rollback

- **Whole change:** activate version 3 again. Sends go back to producing an empty copy on the timeline.
- **Just the move:** delete `Delete_Source_Links` and its connector. The flow then copies the links and leaves Dochly's original untouched.

## 10. Known limits

- **A wrong time-window match now does harm.** If the match ever picks the wrong email, the move strips that message's attachments, rather than only adding a stray link.
- **Dochly's original no longer lists its files** but still says `HasAttachment = true` (see 6.4).
- **`HasAttachment` stays false on the copy**, so it shows no paperclip icon, even though the Attachments list shows the files.
- **Two email records per send remain.** Removing the duplicate needs the related record to be writable, and it is not.
- **Older sends are not repaired.**

## 11. Reuse for similar issues

**How to recognise one:** a managed package writes a standard record (email, Task, file) without linking it to the business record, and an automation makes a second, linked copy that is missing something.

- **Compare the two records side by side**, field by field. The difference is usually the answer.
- **Use timing to pin the source.** Find the moment the problem started and compare it against flow version activation times.
- **Don't trust a field's describe for writability.** Try the write in a sandbox. Some fields reject it, and some, like `LinkedEntityId`, report success and change nothing.
- **Check the result by reading the record back**, not by the save result.
- **To re-home a file, create a new link and then delete the old one**, in that order, with the delete on the success path only.
- **Deploy a flow as Draft first** when activating it affects every record at once.
