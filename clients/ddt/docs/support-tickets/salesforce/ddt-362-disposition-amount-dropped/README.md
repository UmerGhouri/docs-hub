# Disposition amount dropped on save and Chatter double-post

| | |
|---|---|
| **Ticket** | DDT-362 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), Chatter, email template |
| **Area** | Court-case disposition entry: the Chatter summary and the client disposition email |
| **Related** | None recorded |
| **Code** | Direct commit to `main` on 2026-07-14 in the ddt-salesforce repo (no PR number): "Stop duplicate `[DISPOSITION ENTERED]` Chatter post on disposition edits" |

---

## 1. Summary

A staff member saved a disposition on a court case, found the amount due had not been stored, and saved it again to add the amount. The second save posted a second full `[DISPOSITION ENTERED]` Chatter post, so the case feed showed two identical-looking disposition posts. The client email that went out also showed the due date but not the amount or how to pay.

The code change fixed the double post: a save on a disposition that already exists is now headed `[DISPOSITION UPDATED]`. The sources do not record a code change for the dropped amount or for the email template.

## 2. What was reported

The ticket was logged 2026-07-16 and closed the same day; the code fix is dated 2026-07-14, so the ticket records work already done. It concerned one client's court case (a negotiated plea with one charge dismissed and one on supervision, with an amount due and a compliance date). The ticket raised three related problems:

1. **Amount dropped on first save.** The disposition was saved and the amount due was not stored. The staff member had to come back and add it in a second edit.
2. **Duplicate Chatter post.** The second edit published a brand-new `[DISPOSITION ENTERED]` post on the court case (Opportunity) feed, as if the whole disposition had been entered from scratch.
3. **Client email incomplete.** The disposition email to the client ("Update Court Case" template) showed the due date but not the amount due, and carried no payment instructions.

The ticket also asked for a corrected email to that client and for the duplicate post to be removed. Whether either was done is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DispositionEntryController` | Apex class | `saveDisposition` detects an edit; `buildDispositionSummary` takes an `isEdit` flag and picks the header |

**Out of scope**

- **Issue 1 (amount dropped on first save).** No change for it appears in the repo or the ticket. Not recorded in the ticket.
- **Issue 3 (amount and payment instructions missing from the email).** No change to the email template appears in the repo or the ticket. Not recorded in the ticket.
- **Editing or deleting the earlier Chatter post.** The fix labels the second post honestly; it does not merge the two posts.

## 4. How it works (background)

```
Staff open the Disposition Entry screen (dispositionEntry LWC) on a court appearance
  -> Save calls DispositionEntryController.saveDisposition
  -> buildDispositionSummary builds the text: header line + labeled lines
  -> the text is written to Court_Appearance__c.Disposition_Summary__c
  -> the same text is posted to the court case (Opportunity) Chatter feed
```

This auto-Chatter summary was added in an earlier disposition build (see `docs/deploys/disposition-feedback-chatter.md` in the repo). Every save posted the summary, always headed `[DISPOSITION ENTERED]`.

## 5. Root cause

**Duplicate post.** `buildDispositionSummary` always returned `'[DISPOSITION ENTERED]\n' + ...`, and `saveDisposition` posted it on every save. Nothing distinguished a first entry from an edit, so re-saving an existing disposition to add the amount published a second post that read like a new entry.

**Dropped amount and incomplete email.** Not recorded in the ticket. The ticket's own guess for the email was that it fired at the first save, when the amount was still blank, and was not sent again after the edit. That guess was not confirmed in any source.

## 6. Solution

An edit is recognised by the court appearance already carrying a `Disposition_Summary__c`, which is written on every first save. The query that loads the appearance now also reads that field:

```apex
List<Court_Appearance__c> ref = [
  SELECT Opportunity__c, Case_Type__c, Opportunity__r.Case_Type__c,
         Disposition_Summary__c
  FROM Court_Appearance__c
  WHERE Id = :input.courtAppearanceId
  LIMIT 1
];
// A disposition already on file (summary present) means THIS save is an EDIT.
Boolean isDispositionEdit =
  !ref.isEmpty() && String.isNotBlank(ref[0].Disposition_Summary__c);

String summary = buildDispositionSummary(input, sentCfg, isDispositionEdit);
```

And the header is chosen from that flag:

```apex
String header = isEdit == true
  ? '[DISPOSITION UPDATED]'
  : '[DISPOSITION ENTERED]';
return header + '\n' + String.join(lines, '\n');
```

Why it is done this way:

- **The stored summary is the marker.** It already exists on every saved disposition, so no new field was needed to tell a first save from an edit.
- **Re-label rather than suppress.** An edit still posts, so the feed records that the disposition changed and what it now says, but it no longer reads as a second entry.
- **Stacked-sibling saves are unchanged.** When one disposition is fanned out to stacked appearances, those saves are always first entries and keep the `[DISPOSITION ENTERED]` header.

## 7. Testing

The commit changes only `DispositionEntryController` and adds no test. Not recorded in the ticket how the change was checked.

**Not covered:** an automated test that an edit produces `[DISPOSITION UPDATED]`; anything about the amount or the email.

## 8. Step by step

1. In `DispositionEntryController.saveDisposition`, add `Disposition_Summary__c` to the query that loads the court appearance.
2. Set `isDispositionEdit` when that field is not blank.
3. Pass the flag to `buildDispositionSummary` and choose the header from it.
4. Validate against production with `DispositionEntryControllerTest`, get approval, deploy.
5. Check: save a new disposition (one `[DISPOSITION ENTERED]` post), then edit it (one `[DISPOSITION UPDATED]` post).

## 9. Rollback

Redeploy the previous `DispositionEntryController`. Edits go back to posting `[DISPOSITION ENTERED]`. No data changes, so rollback is safe.

## 10. Known limits

- **An edit still adds a post.** The feed shows the original entry plus one update post per edit. The ticket's other options (edit the existing post, or post only the changed fields) were not built.
- **Clearing a disposition resets the marker.** Clearing sets `Disposition_Summary__c` to blank, so the next save is treated as a first entry again.
- **The amount and email problems are not addressed by this change.** See section 3.

## 11. Reuse for similar issues

**How to recognise one:** an automation fires "on save" and its output reads the same whether the record was just created or merely edited.

- Find a field that is always written on the first save and use it to tell a first save from an edit.
- Prefer labelling an edit honestly over silently skipping it, so the audit trail stays complete.
- When a ticket bundles several problems, check each one has its own change before closing it.
