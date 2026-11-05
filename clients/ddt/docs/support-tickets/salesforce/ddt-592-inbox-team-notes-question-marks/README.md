# Shared inbox Team Notes rendering as question marks

| | |
|---|---|
| **Ticket** | DDT-592 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-03 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), shared email inbox |
| **Area** | Shared inbox: the internal Team Notes panel on a Case |
| **Related** | DDT-632 (Team Note @mention emails), DDT-988 (Team Note reply notifications) |
| **Code** | No PR is tagged DDT-592 in the ddt-salesforce repo |

---

## 1. Summary

On the shared inbox, the Team Notes panel on a Case showed most of its entries as a blue "?" avatar with no author and no text. The entries were not lost notes: the panel was listing email and activity feed items alongside real notes, and those items have no note body or author to show. The fix made Team Notes show only real notes.

## 2. What was reported

- **2026-07-30:** on one active PI case, about 15 of roughly 19 Team Notes entries showed only a "?" icon and a timestamp, with no author name and no note text. Only a few notes rendered normally.
- It was reported as team-wide, not limited to one user or one case.
- The request was to find out whether the "?" meant a deactivated author, missing data or a display bug, and to restore the notes if they were real.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `InternalNoteService.getNotes` | Apex | Reads the Team Notes for a Case |
| `internalNotes` | LWC | Renders each note with an avatar, author and body |

**Out of scope**

- **The underlying feed data.** Nothing was deleted or repaired; the change only decides which feed items count as notes.
- **@mention notifications** on Team Notes. Those were separate work (PR #113, and DDT-632).

## 4. How it works (background)

Team Notes are Chatter feed posts (`FeedItem`) on the Case.

```
internalNotes LWC
  -> InternalNoteService.getNotes(caseId)
       -> reads FeedItem rows whose ParentId is the Case
       -> looks up each author's name and photo
  -> for each note, the avatar shows the author's initials
       (getInitials returns "?" when there is no author name)
```

A Case's feed holds more than notes. Emails and other system activity also create feed items on the Case, and those have no note body and no author the panel can name.

## 5. Root cause

The ticket's resolution comment records the cause: Team Notes was showing emails and activity as blank "?" rows. Every non-note feed item on the Case reached the panel, and with no author name and no body, the LWC fell back to a "?" avatar and an empty note.

The ticket's first guesses (a deactivated author, or note records that had lost their body) were not the cause. The real notes were intact; they were mixed in with feed items that were never notes.

**Evidence in the code.** `internalNotes.js` returns `'?'` from `getInitials` when the author name is empty, which is exactly the avatar the users saw.

## 6. Solution

Team Notes now lists only real notes. The version of `InternalNoteService.getNotes` on `main` does this in two places:

```apex
List<FeedItem> items = [
    SELECT Id, Body, Type, CreatedById, CreatedDate
    FROM FeedItem
    WHERE ParentId = :caseId AND Type = 'TextPost'
    ORDER BY CreatedDate ASC
    LIMIT 200
];
// ...
for (FeedItem fi : items) {
    if (String.isBlank(fi.Body))
        continue;   // FeedItem.Body cannot be filtered in SOQL, so blanks are skipped here
    // build the note
}
```

- **`Type = 'TextPost'`** keeps only posts a person typed, and drops email and tracked-change items.
- **The blank-body check** has to happen in Apex because `FeedItem.Body` is a long text area and cannot appear in a `WHERE` clause.

**How the change reached production is not recorded in the ticket.** No commit or PR in the repo is tagged DDT-592, and both filters were already in `main` before this ticket was opened. The resolution comment (2026-07-31) says only that it was fixed to show real notes.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for a check on the reported case and on two or more other cases; the ticket does not record that check or its result.

`InternalNoteServiceTest` covers the service in the repo, but no test is linked to this ticket.

## 8. Step by step

1. Open the Case and note which Team Notes entries show "?".
2. Query the Case's feed and compare types: `SELECT Id, Type, CreatedById, CreatedDate FROM FeedItem WHERE ParentId = '<case-id>' ORDER BY CreatedDate LIMIT 200`. Rows that are not `TextPost` are the ones rendering as "?".
3. Confirm the version of `InternalNoteService` running in production filters on `Type = 'TextPost'` and skips blank bodies. Retrieve it from production rather than trusting `main` (production can differ from `main`).
4. If production lacks the filter, deploy the `main` version with its test class, after reconciling it against production.
5. Reload the Case and confirm only real notes, each with an author, appear.

## 9. Rollback

Removing the filter would bring the "?" rows back. There is no reason to roll it back.

## 10. Known limits

- **Only typed posts count as notes.** If a future feature posts notes another way (for example a different feed item type), they will not show in Team Notes until the filter includes that type.
- **The 200-row limit** caps how many feed items are read per Case. On a Case with a very busy feed, older notes could fall outside it.
- **A note by a user whose name can't be read** still shows "?". The ticket asked for a "Former User" fallback; that is not recorded as built.

## 11. Reuse for similar issues

**How to recognise one:** a list built from a Chatter feed shows blank or nameless rows next to normal ones.

- **Group the feed by `Type`** before assuming data loss. Email, tracked changes and system posts all live in the same feed as user posts.
- **Filter by type in SOQL, and filter blank bodies in Apex**, since long text fields can't be filtered in SOQL.
- **Check what the UI shows for a missing value.** A "?" avatar here was a deliberate fallback, which pointed straight at rows with no author.
