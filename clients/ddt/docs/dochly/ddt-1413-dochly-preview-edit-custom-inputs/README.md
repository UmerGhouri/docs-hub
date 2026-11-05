# Letter custom inputs couldn't be previewed or edited before routing

| | |
|---|---|
| **Ticket** | DDT-1413 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (templates and preview) |
| **Area** | Generating letters from Dochly templates |
| **Related** | DDT-993, DDT-1408 (earlier Dochly template work) |
| **Code** | None. Changes made in Dochly |

---

## 1. Summary

When staff generated letters in Dochly, they couldn't see the document or change its custom input fields before the final routing page, so one wrong field meant starting over. Two more problems were reported on the same ticket: editing the second of two documents generated together reset the first, and the templates still showed an old email address. The support team fixed all three in Dochly, with no code in the `ddt-salesforce` repo.

## 2. What was reported

**The original request.** When generating a letter, the only place to review the document was after the routing page. If a single custom input field needed correcting, the user lost their work and had to start again. This cost time on every letter that needed a correction. The reporter attached a screen recording of the problem.

**Two follow-ups, the same day:**

1. **Edits on one document reset another.** When several documents are generated in one session (for example a Release of Lien and a Lien Reduction), saving edits to the second document reverted the first back to its original template state.
2. **Template details.** The header was too big, and the email address in the templates was out of date and needed changing to the team's current shared address.

How many letters or templates were affected is not recorded in the ticket.

## 3. Scope

**In scope**

- Previewing and editing custom input fields before the routing page.
- Editing several documents generated in one session without one resetting another.
- The email address shown in the templates.

**Out of scope:** Not recorded in the ticket. The header size raised in the follow-up is not mentioned in the closing comment, so whether it was changed is not recorded.

## 4. How it works (background)

```
Staff start a Dochly letter from a record
  -> fill in the template's custom input fields
  -> (one or more documents are generated in the session)
  -> routing page: choose what happens next
```

Before the fix, the document could only be reviewed at the routing step, after the custom inputs had already been committed.

## 5. Root cause

Not recorded in the ticket. The ticket asked whether this could be done through Dochly's configuration or would need a custom screen; the closing comment records the outcome but not which it was, or why one document's edits reset another.

## 6. Solution

From the support team's closing comment on 2026-10-26:

| Problem | Outcome |
|---|---|
| No preview or edit before routing | Custom inputs can now be **viewed and edited in the preview** |
| Editing one document resets another in the same session | **Fixed** |
| Out-of-date email in the templates | **Updated in the templates** |
| Header too big | Not recorded |

Where each change was made (template settings, a Dochly package setting, or Dochly support) is not recorded in the ticket. No commit or pull request in the `ddt-salesforce` repo references DDT-1413.

## 7. Testing

Not recorded in the ticket. The acceptance criteria were that the user can preview the document before the routing page and correct a custom input field without restarting; the ticket doesn't record a check against either.

**Not covered:** the header size, and the multi-document case with more than two documents.

## 8. Step by step

The exact steps are not recorded. To check the result:

1. Generate a letter with custom input fields from a record.
2. Before the routing page, open the preview and change one custom input. The change should show without losing the other inputs.
3. Generate two documents in one session, edit and save the second, and confirm the first keeps its edits.
4. Check the email address shown in the generated letter is the team's current one.

## 9. Rollback

Not applicable: the changes were made in Dochly and the ticket doesn't record what they were, so there is no defined undo. Reverting the email address would reintroduce the out-of-date one.

## 10. Known limits

- **The email address is in the templates.** If it changes again, each template has to be updated, and new templates copied from old ones carry the old address.
- **The header-size request** has no recorded outcome.

## 11. Reuse for similar issues

**How to recognise one:** staff lose work in a Dochly generation session, either by having to restart for one wrong field or by one document's edits undoing another's.

- **Ask for a screen recording** of the session, as the reporter gave here. It shows the exact step where work is lost.
- **Test with two documents in one session**, not one. The reset only showed up there.
- **Treat contact details in templates as data that goes stale.** Check every template that shows an address or email when one changes.
