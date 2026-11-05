# Send for Signature failed with an invalid templateId error

| | |
|---|---|
| **Ticket** | DDT-718 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-13 |
| **Priority** | High (the reporter asked for it to be treated as Urgent) |
| **Reported by** | Intake staff, 2026-08-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Dochly e-signature |
| **Area** | Intake: sending the representation agreement for signature |
| **Related** | DDT-711 (Matter Type picklist empty on Send for Signature), DDT-1070 (the same error text seen later, for a different reason) |
| **Code** | None recorded |

---

## 1. Summary

Intake staff could not send representation agreements. The Send for Signature action failed with `Value provided is invalid for action parameter 'templateId' of type 'Id'`. The ticket was resolved the next morning and the reporter confirmed sending worked again. **What was changed to fix it is not recorded in the ticket**, and no commit in the `ddt-salesforce` repo references this ticket as a fix.

## 2. What was reported

On the Send for Signature screen, the invitation email editor loaded normally (subject, body and merge fields all showed), but pressing **Send envelope** failed with:

```
Send failed — Value provided is invalid for action parameter 'templateId' of type 'Id'
```

It affected **3** client records over the afternoon, and the reporter said they had been unable to send agreements all day. A co-worker's workaround steps did not help. The reporter described it as blocking new client hires and asked for it to be treated as urgent.

## 3. Scope

**In scope:** the Send for Signature action and the Dochly template it passes. Which component was changed is not recorded in the ticket.

**Out of scope:** the user also mentioned that intake documents would not upload for one client. That was noted on the ticket but not investigated under it.

## 4. How it works (background)

```
User opens Send for Signature on a client record
  -> picks the document, edits the invitation email
  -> presses Send envelope
  -> a Dochly action is called with the template's Id as templateId
  -> Dochly generates the document and sends it for signature
```

`templateId` is typed as a Salesforce Id. If the value passed is empty, malformed, or the Id of a template that does not exist in that org, the call is rejected with this error, because the value does not satisfy the parameter's Id type.

## 5. Root cause

Not recorded in the ticket.

The ticket's first guess was that no template was being selected or resolved for the record's matter type, possibly a regression of DDT-711, which had fixed an empty Matter Type picklist on the same screen the day before. Whether that guess was right is not recorded.

For comparison only: the same error text appeared again a month later during DDT-1070, when a template Id that existed only in production was passed in the sandbox. That is a separate occurrence and is not evidence about this ticket's cause.

## 6. Solution

Not recorded in the ticket. The assignee asked the reporter to try again on the morning of 2026-08-13, and the reporter confirmed it was working.

## 7. Testing

**Live check:** the reporter retried in production on 2026-08-13 and confirmed sending worked.

**Not covered:** the ticket's acceptance criterion "no regression on other matter types" has no recorded check, and there is no automated test.

## 8. Step by step

The steps taken are not recorded in the ticket. To diagnose a recurrence:

1. **Reproduce on the affected record** and note which document or template was chosen.
2. **Check that the template exists and is active in this org.** A template is a record, so its Id differs between production and every sandbox.
3. **Check whatever chooses the template** (a matter-type mapping, custom metadata, a flow variable) returns a value for this record.
4. **Fix the source of the empty or wrong Id,** then retry on the same record.

## 9. Rollback

Not applicable: no recorded change to roll back.

## 10. Known limits

- **The cause is unknown,** so nothing prevents it from recurring.
- **The error message does not say which value was wrong.** An empty value, a malformed value and a valid Id from another org all produce the same message.

## 11. Reuse for similar issues

**How to recognise one:** an action fails with `Value provided is invalid for action parameter ... of type 'Id'`. The message is about the input value, not about what the action does with it.

- **Check that the Id exists in the org you are in.** Record Ids, including Dochly template Ids, are different in every sandbox.
- **Look upstream for the lookup that produced the Id.** Blank results from a mapping or picklist are a common source.
- **Write down the cause and the fix on the ticket,** even when the fix is quick. This ticket closed without either, so the next person starts from zero.
