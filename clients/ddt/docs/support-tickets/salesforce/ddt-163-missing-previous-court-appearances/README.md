# Previous court appearances missing on an account

| | |
|---|---|
| **Ticket** | DDT-163 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-15 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (archived court appearance records) |
| **Area** | Court appearance history on closed cases |
| **Related** | None recorded |
| **Code** | None. Answered with an explanation |

---

## 1. Summary

Staff reported that a closed case showed no previous court appearances, and then that many old cases did. Nothing was lost by mistake: by the firm's own requirement, court appearances are **archived, and removed from the case, once the case has been closed for more than a year**. The example case had been closed for over twelve months. No change was made; the firm noted that staff need a simple way to see when records are archived and how to unarchive them.

## 2. What was reported

On 2026-06-12 staff reported that a closed client account showed none of its previous court appearances, although seven dispositions had been entered on the case from the notes. A second report the same day said a lot of old cases were showing no court appearances even though the firm had been to court on them several times.

## 3. Scope

**In scope:** investigation and an explanation to staff.

**Out of scope**

- **Restoring the archived appearances.** They were archived as designed.
- **Changing the archive rule, or adding an on-page notice that records are archived.** Raised by the firm on 2026-06-15 as something to revisit; not done under this ticket.

## 4. How it works (background)

```
Case is closed
  -> more than 12 months pass
  -> its court appearance records are archived
  -> they no longer show on the case
```

How the archiving is configured is not recorded in the ticket.

## 5. Root cause

From the investigation on 2026-06-15: per the firm's requirement, court appearances are archived and removed from the case once the case has been closed for more than a year. The example case had been closed for over twelve months, so its appearances were archived as designed. That also explains the second report: older closed cases all pass the twelve-month line.

**The ticket's first guesses were all wrong.** It asked whether the records had been deleted, never synced, or filtered out of the related list. None applied. The records were archived by a rule the firm had asked for.

## 6. Solution

No change. The rule was explained to staff. The firm's follow-up on 2026-06-15: either revisit the rule, or at least give staff a simple way to see when records have been archived and how to unarchive them.

## 7. Testing

The example case's closed date was checked and found to be more than twelve months before the report. No query was recorded.

## 8. Step by step

1. **Check whether the case is closed, and when.** If it closed more than a year ago, its court appearances are expected to be archived.
2. **Check whether the appearances are archived** rather than deleted: look in Salesforce's archive tools for the court appearance records belonging to the case, and check the Recycle Bin to rule out a deletion.
3. **Explain the rule** to the person who reported it.
4. **If they need the history back for a specific case**, unarchive those records rather than re-entering them.
5. **If the request is to change the rule**, raise it as a new ticket for the firm to decide.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **Nothing on the case says its appearances were archived.** An empty related list looks exactly like missing data, so the same report will keep coming in.
- **Staff do not know how to unarchive**, per the firm's comment.

## 11. Reuse for similar issues

**How to recognise one:** related records "missing" only on old, closed records, with no sign of deletion.

- **Check the record's age against any archive rule first**, before looking for deletions or sync failures.
- **An empty list is not proof of lost data.** Check archive, Recycle Bin and sharing before concluding anything was lost.
- **If a rule hides data on purpose, staff need to be told on the page**, or the support team will answer the same question repeatedly.
