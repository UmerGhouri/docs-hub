# Cook County rescission order document not generating

| | |
|---|---|
| **Ticket** | DDT-56 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs document generation |
| **Area** | Court documents: generating the Cook County rescission order |
| **Related** | DDT-57 (wrong office address on the same template, raised right after this was fixed) |
| **Code** | None. The fix was applied in the org, not in the repo. |

---

## 1. Summary

A staff member could not generate the **Cook County rescission order** from S-Docs, and needed it urgently. The support team found which template was failing, applied a fix the same evening, and the user confirmed it worked about an hour after first reporting it.

The ticket does not record what the root cause was or what the fix changed.

## 2. What was reported

The user reported, marked urgent, that S-Docs was not generating for them and they needed to produce the order. A screenshot was attached in the original Slack thread.

Timeline from the thread (times only; the ticket was logged on 2026-05-26):

| Time | Event |
|---|---|
| 8:11 PM | User reports S-Docs is not generating |
| 8:12 PM | Support starts investigating |
| 8:29 PM | Support asks which template; the user names the Cook County rescission order |
| 8:48 PM | Still not working |
| 9:05 PM | Support applies a fix |
| 9:11 PM | User confirms it works |

Only one template was reported. Whether any other template was affected is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Cook County rescission order | S-Docs template | Fixed so it generates. What was changed is not recorded in the ticket. |

**Out of scope**

- **The office address on the same template.** The user noticed it was wrong once the document generated. That became DDT-57.

## 4. How it works (background)

```
User opens a record and starts S-Docs
  -> picks a template (here: the Cook County rescission order)
  -> S-Docs merges record fields into the template body
  -> the generated document is returned to the user
```

S-Docs templates are **records** of the managed-package object `SDOC__SDTemplate__c`, not metadata. Editing one in the org changes it immediately, and the change cannot be committed to the repo the way a class or a flow can.

## 5. Root cause

Not recorded in the ticket. The resolution note says only that the order "now generates cleanly in S-Docs" for the reporting user and others.

The ticket's acceptance criteria asked for the cause to be documented and for a check that no other templates were affected. Neither is recorded.

## 6. Solution

A fix was applied by the support team at 9:05 PM. What was changed (template body, a merge field, a permission, or something else) is **not recorded in the ticket.**

## 7. Testing

**Live check:** the reporting user generated the order successfully at 9:11 PM, and the resolution note says it now generates for the reporting user and for other users.

**Not covered:** the root cause, and whether any other S-Docs templates had the same problem.

## 8. Step by step

The exact fix is not recorded, so these are the investigation steps that fit this symptom:

1. **Get the template name first.** In this ticket it took 17 minutes to learn which template was failing. Ask for it in the first reply.
2. **Reproduce as an administrator** on the same record. If it works for you but not the user, suspect permissions or licences; if it fails for everyone, suspect the template.
3. **Check the user's S-Docs access**: the package licence and the S-Docs permission set.
4. **Open the template record** and check for a merge field that refers to a field which was renamed, deleted or is hidden from the user.
5. **Fix, then have the user retry** on the same record.
6. **Write down what you changed** on the ticket. This ticket did not, which is why this page cannot say.

## 9. Rollback

Not applicable: the change made is not recorded, so there is nothing specific to undo.

## 10. Known limits

- **S-Docs template changes are not version-controlled.** Templates are records, so a fix leaves no history in the repo. Record the change on the ticket instead.
- **The cause is unknown**, so the same failure could recur without warning on this or another template.

## 11. Reuse for similar issues

**How to recognise one:** a user says S-Docs "isn't generating" for one document.

- **Ask which template and which record** in the first reply.
- **Compare working and failing users** to split a permission problem from a template problem.
- **Write the root cause and the change on the ticket** before closing it. Template fixes leave no trace anywhere else.
- **Once it generates, check the output.** On this ticket the next problem (the wrong office address, DDT-57) was only visible once the document generated.
