# Error opening a lead from an email link

| | |
|---|---|
| **Ticket** | DDT-778 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-21 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-19 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lead, Chatter email notifications), RingCentral |
| **Area** | Opening records from Salesforce notification emails |
| **Related** | None recorded |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

A staff member clicked a link in a Salesforce Chatter notification email and got "We couldn't find the record you're trying to access". The link pointed to a Lead that had been deleted after the email was sent. It was not an access problem. No change was needed; the user was given the existing client record the Lead duplicated.

## 2. What was reported

- **2026-08-19:** the user clicked a link in a Salesforce email and saw:

  > We couldn't find the record you're trying to access. It may have been deleted by another user, or there may have been a system error.

- She described the record as a "case", but the link was a Lead link (the Id starts with `00Q`) from a `chatterPostNotification` email.
- Asked whether it happened elsewhere, she said it was one record, a RingCentral lead.

## 3. Scope

**In scope:** investigating one Lead link from a Chatter notification email.

**Out of scope:** any change to notifications, sharing or the RingCentral lead creation. None was needed.

## 4. How it works (background)

```
Inbound call from an unknown number
  -> RingCentral integration auto-creates a "RingCentral Caller" Lead
  -> a colleague @mentions a user on that Lead in Chatter
  -> Salesforce emails the user a link to the Lead
  -> later, someone deletes the Lead as a duplicate
  -> the link in the already-sent email now points at nothing
```

Salesforce shows the same "couldn't find the record" page for a deleted record as it does for some access problems, so the message alone does not tell you which it is.

## 5. Root cause

The Lead no longer existed. Investigation found:

1. It was an auto-generated "RingCentral Caller" Lead created on 2026-08-10.
2. The reporting user **owned** it, so sharing was not the issue.
3. A colleague @mentioned her on it that evening, which sent the email.
4. Another staff member deleted it the next morning as a duplicate. The caller's number was already on file under an existing client with an open case.

The email went out before the deletion, which is why the link failed.

The ticket's guesses (deleted, converted, merged, or no sharing access) included the right one; the investigation ruled out the access explanation.

## 6. Solution

No configuration or code change. The user was told the Lead had been deleted as a duplicate and was given the existing client Account and open Court Case to work from instead.

## 7. Testing

Not applicable: nothing was changed. The findings in section 5 (owner, creation date, the Chatter mention and the deletion) are recorded on the ticket; the exact queries used are not recorded in the ticket.

## 8. Step by step

1. **Take the record Id from the link.** The prefix tells you the object (`00Q` is Lead).
2. **Check whether the record exists**, including the Recycle Bin. A deleted record can be queried with `ALL ROWS` in Apex or with the API's query-all option.
3. **If it is deleted, find who deleted it and when**, and compare with the email's send time.
4. **If it exists, check ownership and sharing** for the user.
5. **For an auto-created RingCentral lead, search by the phone number** to find the existing client or matter it duplicated.
6. **Reply to the user** with the record they should use instead.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- **Links in already-sent emails cannot be updated.** Any notification about a record that is later deleted will show this error.
- **Auto-created RingCentral leads for existing clients** are deleted as duplicates, so this can recur with mentions on such leads.

## 11. Reuse for similar issues

**How to recognise one:** "couldn't find the record" from an email link, especially for a single record.

- **Check existence before sharing.** A deleted record and an inaccessible record show the same page.
- **Compare the email time with the deletion time.**
- **Look for a duplicate.** Auto-created leads from phone calls are often deleted because the caller is already a client.
