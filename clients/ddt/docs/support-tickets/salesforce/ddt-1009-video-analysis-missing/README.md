# Video analysis missing after completion

| | |
|---|---|
| **Ticket** | DDT-1009 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Court Case record, Chatter) |
| **Area** | Case work: video analysis on a court case |
| **Related** | None recorded |
| **Code** | Not applicable: no code change |

---

## 1. Summary

A staff member reported that a completed video analysis had disappeared from a court case. Investigation showed nothing was deleted: the Video Analysis field on that case had never been filled in. The review notes had been written in a Chatter post on the case instead, and the videos were still in Google Drive. No change was made, and the reporter closed the ticket.

## 2. What was reported

- **2026-09-14:** a staff member said a video analysis for one client's court case had been completed and was visible the week before, but now showed as missing. They were not sure when it had been completed.
- **2026-09-15:** asked where they had seen it, they recalled checking their notes the previous week and thought they had filled the field in on 7/6.

One court case was involved.

## 3. Scope

**In scope:** the Video Analysis field on one Court Case record (Opportunity, relabelled "Court Case" in this org), its change history, and the Chatter feed on the same record.

**Out of scope:** any code, layout or permission change. Nothing was found to fix.

## 4. How it works (background)

- The court case record has a Video Analysis field where the written analysis is kept.
- Changes to the record are tracked in a change log on the record, so edits to tracked fields leave a dated entry with who made them.
- The record also has a Chatter feed, where staff can post free-text updates. A Chatter post is not the same as filling in a field: the field stays blank.
- Case videos are stored in Google Drive.

## 5. Root cause

Not a defect. The field was never filled in, so there was nothing to lose.

Evidence recorded on the ticket:

- **No change to the Video Analysis field** was ever logged on that record.
- **The log was working on that record and that day.** It captured four other edits by the same user on 7/6 (including a Video Analysis entry on a different case that morning, and discovery and evaluation updates in the afternoon), and three other field changes on this case over its life. So a missing entry is real evidence that the field was not changed, not a gap in the log.
- **The write-up existed elsewhere.** On 7/6 the same user posted the video review as a Chatter post on this case. That is most likely what they remembered seeing.
- **The source videos were still in Drive.**

The ticket's suggested causes (deleted, failed to save, display or permissions) did not apply. The work was saved in a different place from the one the user later looked.

## 6. Solution

No system change.

- The reporter was told the notes were safe in the Chatter post on the case and the videos were still in Drive.
- The support team offered to copy the Chatter write-up into the Video Analysis field. The reporter said it was not needed and asked for the ticket to be closed.

## 7. Testing

Not applicable: nothing was changed. The investigation steps are in section 8.

## 8. Step by step

1. Open the court case and confirm the Video Analysis field is blank.
2. Check the record's change history for any entry on the Video Analysis field. Check the Recycle Bin if a whole record is said to be missing.
3. Prove the history is reliable for that record: find other tracked edits by the same user on the same day, and other changes to this record. If those are logged, an absent entry means the field was not changed.
4. Look in the record's Chatter feed around the date the user remembers. Free-text updates are often posted there instead of in a field.
5. Check the source material (here, the Drive video folder) still exists.
6. Report back with where the work actually is, and offer to move it into the field.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The analysis still sits in a Chatter post, not in the Video Analysis field, so anything that reads that field (reports, completion flags) will treat this case as not analysed.
- The change history only covers tracked fields. This approach does not work for a field that is not tracked.

## 11. Reuse for similar issues

**How to recognise one:** "I filled it in and now it is gone", with no clear date and no error.

- Check the field's history before assuming a deletion or a bug.
- Before trusting an empty history, show the history captured other edits by the same person on the same day. That turns "no entry" into evidence.
- Check Chatter, emails and notes on the record. Work saved in the wrong place looks exactly like lost work.
- Offer to move the content into the right field, but let the user decide.
