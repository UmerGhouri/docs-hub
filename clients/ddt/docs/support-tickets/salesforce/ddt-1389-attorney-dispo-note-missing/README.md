# Attorney disposition note missing from Notes

| | |
|---|---|
| **Ticket** | DDT-1389 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (court appearance disposition screen, Notes) |
| **Area** | Court cases: recording a hearing's disposition and the attorney's note |
| **Related** | None recorded |
| **Code** | Not applicable: investigation only, no change made |

---

## 1. Summary

A staff member could not find the attorney's disposition note on one matter, although the Court Appearance history showed the disposition had been entered in April 2026. The note was not deleted. It was never saved: at that time the disposition screen skipped the note whenever the notes box was left empty, so only the result was recorded. The disposition screen in use since July 2026 always saves the note, so it cannot happen again in the same way.

## 2. What was reported

On 2026-10-21 a staff member reported that on one court case the attorney's disposition note did not appear in the Notes section. The Court Appearance history showed:

- The appearance created and an attorney assigned in March 2026.
- The Disposition field set by the attorney in April 2026.
- The disposition letter marked as sent about a week later.

There was no matching note. They asked whether it could be restored, since a deleted note would mean lost data. One record was affected.

## 3. Scope

**In scope:** finding out whether the note had ever existed, and telling the requester.

**Out of scope:** any code or configuration change. The behaviour that caused it had already been replaced by the July 2026 disposition screen.

## 4. How it works (background)

```
Attorney opens the disposition screen on a Court Appearance
  -> enters the result (Disposition) and, optionally, notes
  -> the result is saved on the Court Appearance (field history records the change)
  -> the note is saved to the matter's Notes
```

Before July 2026, the last step ran only when the notes box had text in it.

## 5. Root cause

The note was never created. In April 2026 the disposition screen skipped creating a note whenever the notes box was left empty, so only the Disposition value was saved. That is why field history shows the disposition but the Notes section has nothing for it.

The ticket's first worry was a deleted note. The investigation found it was not deleted but never saved. The queries used to establish this are not recorded in the ticket.

## 6. Solution

Nothing to restore. The requester was told on 2026-10-22:

- the note was never saved, so there is nothing in the Recycle Bin or elsewhere to recover;
- if the attorney had comments beyond the court results already posted on the record, they need to add them again;
- the disposition screen in use since July 2026 always saves the note, so this will not recur.

## 7. Testing

Not applicable: no change was made. The finding was given to the requester on the ticket.

## 8. Step by step

To investigate a "missing note" like this:

1. **Read the field history** on the Court Appearance to get the exact date and time the disposition was entered.
2. **Check the matter's Notes** (Notes and Attachments, and Notes on the related list) for anything near that time.
3. **Check the Recycle Bin** for a deleted note on that record.
4. **Check whether the note could be on another record**, such as the client's Account.
5. **Check how the disposition screen behaved on that date.** If the note was optional then and the box was empty, no note was ever created.
6. **Tell the requester** whether it can be restored, and if not, that the attorney should re-enter it.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **Older dispositions.** Any disposition entered before July 2026 with an empty notes box has no note. That cannot be recovered.
- **Notes are only as complete as what was typed.** The current screen always saves a note, but the content is still whatever the attorney entered.

## 11. Reuse for similar issues

**How to recognise one:** a user says data was "deleted" but field history shows the action happened and no trace of the expected record exists.

- **Ask "was it ever saved?" before "who deleted it?"** Check the Recycle Bin, then check whether the process that should have created it ran at all on that date.
- **Find out what version of the process was live on the date in question.** A screen or flow that has since been rebuilt may have behaved differently.
- **Say plainly when something cannot be recovered**, and who needs to re-enter it.
