# Expense invoice attachments not syncing to Drive

| | |
|---|---|
| **Ticket** | DDT-775 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-21 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-19 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata), Google Drive |
| **Area** | Expenses: filing invoice attachments into the matter's Drive folder |
| **Related** | None recorded |
| **Code** | PR #223, PR #224 and PR #282 in the `ddt-salesforce` repo |

---

## 1. Summary

An invoice attached to an Expense record in Salesforce never appeared in the matter's "Expenses" folder in Google Drive. Nothing had failed: the Drive pipeline simply never handled files attached to `Expense__c`. PR #223 added Expense as a recognised parent and files the invoice into the matter's `Expenses/Invoices` folder (or the deepest part of that path that exists). PR #224 fixed a missing field in a query that stopped the first version from moving anything, and PR #282 fixed the same mistake for a second field in the same query.

## 2. What was reported

- **2026-08-19:** a staff member added an expense with a PDF invoice attached. The invoice showed in Salesforce under the Invoice column, but could not be found in the client's "Expenses" folder in Google Drive.
- The ticket asked for the root cause, whether it was a one-off, a backfill of the missing invoice, and confirmation that future invoices sync.

Why it matters: the Expenses folder in Drive is where staff pull invoices for reimbursements, settlements and case cost tracking. A missing invoice means re-requesting it from the vendor.

At the time, 218 files were attached to expense records. Only the reported one had been looked for in Drive.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Drive_Root__mdt.Expense_Invoice_Folder_Path__c` | Custom metadata field (new) | Slash-separated folder names, for example `Expenses/Invoices` |
| `Drive_Root.Matters_Aug2026` | Custom metadata record | Path set to `Expenses/Invoices` |
| `GoogleDriveService.practiceAreaFor` | Apex | `Expense__c` maps to PI |
| `GoogleDriveService.roots` | Apex | Query now selects `Expense_Invoice_Folder_Path__c` (PR #224) and `Quilia_Folder_Name__c` (PR #282) |
| `ContentVersionTriggerHandler` | Apex | Accepts an `Expense__c` parent |
| `DriveAttachmentSweep` | Apex | Accepts an `Expense__c` link |
| `DriveAttachmentRelay` | Apex | Resolves an Expense to its Matter's Drive folder; new `expenseInvoiceFolder` walks the path |
| `DriveAttachmentRelayTest`, `GoogleDriveServiceTest` | Apex tests | New tests, see section 7 |

**Out of scope**

- **Creating missing `Invoices` folders** on older matters. That is Drive housekeeping; the folder template already includes it for new matters.
- **Automatically moving historical invoices.** The sweep only looks back 3 hours, so deploying the change moves no existing file on its own. After the fix, the requester asked for the remaining expense invoices to be backfilled too, and the ticket records that as done. How that backfill was run is not recorded in the ticket.

## 4. How it works (background)

```
File attached to a record in Salesforce
  -> ContentVersionTriggerHandler (fast path)   accepted only Opportunity and Matter__c parents
  -> DriveAttachmentSweep (hourly catch-up)     accepted only Opportunity and Matter__c
  -> DriveAttachmentRelay                       copies the file into the record's Drive folder
```

`Expense__c` has no Drive folder field of its own. Its folder lives on the parent Matter.

## 5. Root cause

**The ticket's premise was wrong.** It asked why the sync "failed". There was no expense-invoice sync. `Expense__c` was excluded at all three layers above, so the fast path skipped the file, the sweep skipped it, and the relay had no folder to send it to. The invoice was never lost; it stayed on the Expense record the whole time.

**The destination also had to be measured.** Listing 22 matters with expenses on 2026-08-20:

```
"Expenses"                  20 of 22
"Expenses/Invoices"         10 of 22
neither                      2 of 22
```

Two folder structures are live, split by the 2026-07-30 move away from the CloudFiles package. No single destination exists on every matter.

**A second bug after the first deploy (PR #224).** The new field was read by the relay but was never added to the `SELECT` in `GoogleDriveService.roots()`. Custom metadata does not fail at compile time for this. The row loads and the read throws "row was retrieved via SOQL without querying the requested field" at runtime. Three things hid it:

1. The relay catches the exception and logs it as Failed in `Drive_Activity_Log__c`.
2. The Queueable still reported Completed with 0 errors.
3. Every test seeded the metadata cache with `new Drive_Root__mdt(...)`, so none of them ever ran the real query.

So the first deploy was green, the backfill reported success, and the invoice did not move. The Failed row in `Drive_Activity_Log__c` named the missing field.

**The same mistake again (PR #282, 2026-09-01).** A later change read `Quilia_Folder_Name__c` and also left it out of the same query. A read-only probe against production caught it before any real file hit that path.

## 6. Solution

### 6.1 Treat Expense as a Drive parent (PR #223)

- The trigger handler and the sweep accept an `Expense__c` parent. The handler passes the Expense Id through as-is rather than looking up its Matter, because the handler runs on every file inserted anywhere in the org and is designed to spend no queries. The relay already queries per record, so the Matter lookup lives there.
- `practiceAreaFor` names `Expense__c` explicitly. Letting it fall through to the "all areas" default would resolve the wrong root and could file a PI invoice into the Defense shared drive. Those drives are kept separate on purpose.

### 6.2 The destination is a path, walked one folder at a time

`Expense_Invoice_Folder_Path__c` holds folder **names**, not Ids, because each case folder is a copy of a template with its own fresh Ids. The relay walks the path and stops at the deepest folder that exists:

| What exists on the matter | Where the invoice lands | Logged |
|---|---|---|
| `Expenses/Invoices` | `Invoices` | no |
| `Expenses` only | `Expenses` | yes |
| neither | the matter folder | yes |
| path not configured | the matter folder | no |

```apex
String current = caseFolderId;
for (String segment : path.split('/')) {
    String wanted = segment.trim();
    if (String.isBlank(wanted)) continue;
    GoogleDriveService.DriveResource found =
        GoogleDriveService.findChildFolder(current, wanted, subject);
    if (found == null) {
        // log "This matter has no <wanted> folder ... filed one level up"
        return current;
    }
    current = found.id;
}
return current;
```

Why this way:

- **A missing folder is never created.** Folder names come from the template, so a missing one means someone renamed or deleted it. Creating a replacement leaves staff with two folders to search.
- **A matter that later gains the subfolder starts using it** with no code change.
- **The invoice gets its own setting** instead of reusing the generated-documents folder name, because that setting applies to every document the relay copies for the practice area. Pointing it at "Expenses" would have filed signed retainers there too.

### 6.3 Add every field the code reads to the query (PR #224, PR #282)

`Expense_Invoice_Folder_Path__c`, and later `Quilia_Folder_Name__c`, were added to the `SELECT` in `GoogleDriveService.roots()`, with a comment above the list saying that every field read anywhere must be named there. PR #282 deliberately did not wrap the new read in a broad `catch`: a swallowed exception is what kept this hidden.

### 6.4 Backfill the reported invoice

The reported invoice was backfilled by re-running the real relay for that one Expense rather than uploading it by hand, so the backfill also proved the deploy worked:

```apex
DriveAttachmentRelay.enqueue(new Set<Id>{ '<expense record Id>' });
```

The relay skips files by name, so running it twice cannot create a duplicate.

## 7. Testing

**PR #223:** six tests in `DriveAttachmentRelayTest`.

| Test | Proves |
|---|---|
| `...GoesToTheInvoicesSubfolderWhenTheWholePathExists` | full path goes to `Invoices` |
| `...StopsAtExpensesWhenInvoicesIsMissing` | the "Expenses only" shape files correctly |
| `...FallsBackToTheMatterFolderWhenNeitherExists` | a matter with neither folder still gets the file |
| `...GoesToTheMatterFolderWhenNoPathIsConfigured` | blank config keeps the old behaviour |
| `anExpenseWhoseMatterHasNoDriveFolderIsSkipped` | no destination means no upload |
| `practiceAreaForAnExpenseIsPI` | the invoice cannot reach the Defense drive |

The test Drive mock gained an opt-in folder-by-name lookup so it could express "the first folder exists, the second does not". Left empty it behaves as before, and the 86 existing tests passed unchanged. Validated against production: 92 tests, 0 failed.

**PR #224:** two tests in `GoogleDriveServiceTest`.

- `rootsQueryReturnsEveryFieldTheCodeReads` runs the real query and reads every field. It is the only shape of test that fails when a field is left out.
- `thePIRootCarriesTheExpenseInvoicePath` checks the PI root record actually has the path configured. A complete query over an unconfigured record would quietly file every invoice in the matter folder root.

Both failed against the old query with the exact production error, and passed with the field restored (79 tests, 0 failed).

**PR #282:** `Quilia_Folder_Name__c` added to the guard test, plus `thePIRootCarriesTheQuiliaFolderName`. Validated against production: 284 tests, 0 failures.

**Live check:** on 2026-08-21 the reported invoice was confirmed in the matter's Drive folder by listing the folder.

**Not covered:** whether every historical expense invoice reached Drive in the later backfill is not recorded in the ticket.

## 8. Step by step

1. **Confirm the file exists on the Expense** and is missing from Drive. It is not lost.
2. **Measure the destination folders** on a sample of matters with expenses before choosing a target.
3. **Add the custom metadata field and set it on the PI root.** Deploy the field and the metadata record in the same deploy as the Apex, or before it. The first validation failed because the field's description was over 1,000 characters, so the field did not deploy and everything that referenced it failed with it.
4. **Change the trigger handler, sweep, relay and `practiceAreaFor`** as in section 6.
5. **Add the new field to the `SELECT` in `GoogleDriveService.roots()`** and to `rootsQueryReturnsEveryFieldTheCodeReads`.
6. **Validate against production** with `DriveAttachmentRelayTest`, `ContentVersionTriggerHandlerTest`, `DriveAttachmentSweepTest` and `GoogleDriveServiceTest`.
7. **Get the PR approved and deploy.**
8. **Backfill the reported Expense** with `DriveAttachmentRelay.enqueue(...)`.
9. **Check `Drive_Activity_Log__c`** for a Failed row, then list the matter's Drive folder. Do not trust the Queueable's Completed status.
10. **Attach an invoice to a new expense** and confirm it reaches Drive.

## 9. Rollback

Revert the four Apex classes. The custom metadata field and value can stay; with no Apex reading them they do nothing. Invoices already copied to Drive stay there, and the Salesforce copies are never removed.

## 10. Known limits

- **About half the matters had no `Invoices` subfolder** at the time. Their invoices land in `Expenses` (or the matter folder), with a log note.
- **Historical files do not move on their own.** The sweep looks back only 3 hours.
- **Any new field read from `Drive_Root__mdt` must be added to `roots()` and to the guard test.** Tests that build the metadata record in memory cannot catch an omission. PR #282 is the proof that this happens again.

## 11. Reuse for similar issues

**How to recognise one:** a file "didn't sync", but nothing errored. Check whether the parent object was ever in the pipeline's scope before looking for a failure.

- **Trace every layer** (trigger, sweep, relay) for the object type before assuming one of them broke.
- **Measure the real destination** on a sample of records; folder structures differ between older and newer records.
- **Fall back one level, log it, never invent folders.**
- **When custom metadata is read through a hand-written query**, every field must be in the `SELECT`. Keep a test that runs the real query and reads every field.
- **A swallowed exception plus a green status hides failures.** Check the activity log and the destination, not the job status.
