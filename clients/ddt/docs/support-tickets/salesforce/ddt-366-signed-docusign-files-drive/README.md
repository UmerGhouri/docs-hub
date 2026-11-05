# Signed DocuSign files saved to the Drive client folder

| | |
|---|---|
| **Ticket** | DDT-366, DDT-731 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-09 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex), DocuSign, Google Drive |
| **Area** | PI intake: the signed retainer after a Lead converts to a Matter |
| **Related** | DDT-867 (CloudFiles flows retired), PR #286 and PR #287 (routing moved from CloudFiles into Apex) |
| **Code** | PR #338 in the `ddt-salesforce` repo (DDT-731). DDT-366 has no PR recorded. |

---

## 1. Summary

When a PI client signs the retainer in DocuSign and the Lead converts to a Matter, the two signed PDFs (`Signed_Document.pdf` and `Certificate_Of_Completion.pdf`) landed on the Matter as ordinary Salesforce Files. Staff had to copy them into the Matter's Google Drive **Client Information** folder by hand and then delete the Salesforce copy.

The work was done in two halves. DDT-366 made the signed PDFs **copy** into Client Information automatically. DDT-731 made the overnight clean-up job **release the Salesforce copy** afterwards, which it had never once done for a signed document, because a link to the Lead made every signed PDF look as if it belonged to two cases.

## 2. What was reported

- **DDT-366 (2026-07-16):** a feature request. Signed retainers needed to be in the Matter's Client Information folder in Drive, and saving them there was a manual step. The ticket asked for: no duplicates on re-run, handling for a Matter whose Drive folder is not ready yet, and a visible failure path.
- **DDT-731 (2026-08-13):** on a live PI Matter signed the day before, both PDFs were still sitting on the Matter as Salesforce Files. The request: move them into Client Information and delete the Salesforce copies. It was raised as an enhancement to the new Drive integration that replaced CloudFiles, and was re-classed as a defect, since moving files out of Salesforce was the point of that work.

An early answer on DDT-366 (2026-07-16) explains one apparent "missing file": the signed PDF only reaches Salesforce once **both** parties have signed. On that record the client had signed and the firm's countersignature was still pending, so there was nothing to copy yet.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CloudFiles: Copy Files From Matter to GDrive` | Flow (CloudFiles actions) | DDT-366 behaviour: copies files named `Signed_Document` or `Certificate_Of_Completion` on a Matter into the Matter's Client Information folder. Later retired (see 4). |
| `DriveAttachmentRelay` | Apex class | PR #286: the same two-name routing, replicated natively before the CloudFiles flow was switched off |
| `Drive_Root__mdt.Signed_Docs_Folder_Name__c` | Custom metadata field | PR #286: the folder name the signed documents go to (`Client Information`) |
| `DriveReclaimBatch` | Apex class | PR #338: new `countsAsAnotherRecord` method decides which links mean "a second case" |
| `DriveReclaimBatchTest` | Apex test | PR #338: two new tests |

**Out of scope** (recorded in the PR #338 deploy doc as deliberately left for separate work)

- **One batch chunk failing every night** with "Cannot locate Apex Type for ID", which means some documents are never examined while the job still reports Completed. It predates this change.
- **Signed documents uploaded in August, before the relay stamped each document's Id onto it.** The clean-up job only looks at stamped documents, so those were to be moved and deleted by hand.
- **Envelopes that never completed** because the firm had not countersigned. That is operational, not code.

## 4. How it works (background)

```
Client signs the PI retainer in DocuSign; the firm countersigns
  -> envelope reaches "completed"
  -> DocuSignPDFDownloader saves the signed PDFs as Salesforce Files
       first on the Lead, then links them to the new Matter
  -> COPY  (instant): the file is copied into the Matter's Drive folder
       signed documents go into the "Client Information" subfolder
  -> RELEASE (overnight): DriveReclaimBatch checks Drive really holds an
       identical copy, then deletes the Salesforce copy to the Recycle Bin
```

**The copy half.** At the time of DDT-366, the copy was done by the CloudFiles flow above. It fired when a file was attached to a Matter, filtered on the file name containing `Signed_Document` or `Certificate_Of_Completion`, and copied it into the folder named `Client Information`. It was copy-only. On 2026-09-01 the same rule was rebuilt in `DriveAttachmentRelay` (PR #286), the flow was captured into git (PR #287), and the CloudFiles flows were then removed (DDT-867).

**The release half.** `DriveReclaimBatch` runs nightly and deletes a Salesforce copy only if all of these hold: the document is recent, it is on a Court Case, Matter or Expense, it is linked to **exactly one record**, Drive holds a matching untrashed file, and the byte size matches. The "exactly one record" rule exists because deleting a Salesforce file removes it from every record it is attached to.

## 5. Root cause

DDT-366 was a missing feature, not a fault.

DDT-731's cause was in the "exactly one record" rule. It counted every link on the document except the uploading user, and two kinds of link that are not cases at all were always present on a signed PDF:

1. **The Lead.** `DocuSignPDFDownloader` publishes the signed PDF to the Lead first and then links the Matter. So every DocuSign retainer carried two record links from the moment it was created.
2. **E-signature junction records.** Dochly-signed documents carry two link rows on Dochly's own objects.

Either one alone kept the count above 1 for good, so the job **never released any signed document**. Measured in production on 2026-09-09: release log entries naming `Signed_Document.pdf` or `Certificate_Of_Completion.pdf`, all time, were **zero**. It affected both practice areas, so it was not specific to DocuSign.

**Evidence.** The links on the 26 documents one nightly run kept, counted on 2026-09-09:

| Link type | Count | A second case? |
|---|---|---|
| `User` | 26 | no, already ignored |
| `Opportunity` (Court Case) | 18 | yes |
| `dochly__DH_Signature_Request__c` | 16 | no, e-sign junction |
| `Matter__c` | 8 | yes |
| `Lead` | 8 | no, same client before conversion |
| `dochly__DH_Document__c` | 8 | no, e-sign junction |
| `Account` | 2 | yes |

The check to run for this symptom (zero before the fix):

```sql
SELECT Status__c, COUNT(Id)
FROM Drive_Activity_Log__c
WHERE Action__c = 'Delete Local Copy'
  AND (Resource_Name__c LIKE '%Signed_Document%'
       OR Resource_Name__c LIKE '%Certificate_Of_Completion%')
GROUP BY Status__c
```

## 6. Solution

### 6.1 Copy the signed PDFs into Client Information (DDT-366)

The behaviour live in production was the CloudFiles flow described in section 4: on a file attached to a Matter, if the name contains `Signed_Document` or `Certificate_Of_Completion`, find the Matter's connected Drive folder, look for a subfolder named `Client Information`, and copy the file there. The ticket does not record who built the flow or exactly when; the closing note on 2026-07-20 confirms that once both parties sign, the signed document and the certificate copy into Client Information, verified in production.

When the CloudFiles integration was replaced, PR #286 rebuilt the rule in `DriveAttachmentRelay` rather than reinterpreting it:

- **Name match, case-insensitive, "contains".** A second copy arrives as `Signed_Document (1).pdf`, so an exact match would miss it.
- **A separate config field**, `Signed_Docs_Folder_Name__c`, rather than pointing every PI document at Client Information. The general folder setting applies to every document the relay copies, which would have relocated all PI documents.
- **A missing folder is logged, not created.** Not every Matter folder has a Client Information subfolder; inventing one would leave staff hunting for it.

### 6.2 Release the Salesforce copy (DDT-731, PR #338)

The inline "skip User links" check was replaced with one method that asks the question the rule actually means: would a second case lose this document?

```apex
private static final String ESIGN_JUNCTION_PREFIX = 'dochly__';

@TestVisible
private static Boolean countsAsAnotherRecord(Id linkedId) {
    SObjectType t = linkedId.getSobjectType();
    if (t == User.SObjectType || t == Lead.SObjectType) {
        return false;
    }
    return !String.valueOf(t).startsWith(ESIGN_JUNCTION_PREFIX);
}
```

Why it's done this way:

- **The rule is narrowed, not removed.** `Account`, Court Case, Matter and everything else still count. The 2 kept documents with an Account link stay protected.
- **Deleting the guard was measured and rejected.** Of the 36 documents in one run, 26 sat on exactly one case and 0 on two or more. Dropping the guard would release the same files today and lose the protection the first time a document really is attached to two Matters.
- **The Dochly namespace is matched as text**, not by naming the type. The sandbox had no Dochly package at the time, and naming the type would stop the class compiling there.

**What staff see:** the signed contract disappears from the Lead page. One file on two records can't be freed from Salesforce and still shown on the Lead; the only copy is then in the Matter's Client Information folder. This was a sign-off step in the runbook before deploying.

## 7. Testing

| Test | What it proves |
|---|---|
| `releasesADocumentThatIsAlsoLinkedToItsLead` (new) | The DocuSign shape, Matter plus Lead, is released |
| `countsOnlyLinksThatAreRealCaseRecords` (new) | Court Case and Account count; Lead and User do not |
| `keepsADocumentThatIsAlsoLinkedToAnotherRecord` (existing) | An Account link still protects the document |

Validated check-only against production, 15 of 15 tests passing.

**Not covered:** the `dochly__` branch has no test and cannot have one, because the sandbox had no Dochly package to build a record from when it was written. It was verified against production only.

**Live check (2026-11-05):** the query in section 5 returns **87** `Success` rows, the first at 2026-09-09 13:21 UTC, against zero before the fix.

## 8. Step by step

1. **Confirm the symptom** with the query in section 5. Zero (or no recent) rows for signed documents means nothing is being released.
2. **Inspect the links on a kept document**: query `ContentDocumentLink` for it and group by the linked record's object type. Look for Lead and e-signature junction links.
3. **Change `countsAsAnotherRecord`** so only real case records count, and add the tests in section 7.
4. **Validate check-only against production** with the class and its test. At the time the sandbox could not validate this: it lacked the Drive configuration, the activity log object and the Dochly package.
5. **Get sign-off** that intake does not rely on the Lead page to find the signed retainer.
6. **Deploy.**
7. **Set the `DriveReclaimLocalCopies` custom label to `dryrun` for one night** and read what it would release, without deleting.
8. **Set it back to `true`.**
9. **Next morning**, open a newly signed Matter: both PDFs should be gone from Salesforce Files and present in Drive under Client Information.

## 9. Rollback

- **Stop deletions immediately:** set the `DriveReclaimLocalCopies` label to `false` in Setup. No deploy needed.
- **Code:** redeploy the previous `DriveReclaimBatch`. Signed documents go back to never being released, which is safe but brings the storage problem back.
- **A wrongly released file** goes to the Recycle Bin, never a hard delete, so it can be restored for 15 days.

## 10. Known limits

- **The signed contract is no longer visible on the Lead** once released.
- **A new e-signature vendor would need adding.** Its junction objects would count as "another record" and block release again, the same way Dochly's did.
- **Files only reach Salesforce after both signatures.** If the firm doesn't countersign, nothing is filed at all.
- **No Client Information subfolder, no filing there.** The relay logs it and leaves the file in the Matter folder.
- **The other limits listed in section 3** (the failing batch chunk, unstamped August files) were outside this fix.

## 11. Reuse for similar issues

**How to recognise one:** a clean-up or "move" job runs every night, reports success, and a particular kind of file is never touched.

- **Look at what the job counts, not just whether it runs.** Group the links on a kept file by object type. Integration plumbing (Leads before conversion, e-sign junctions, uploader User links) often looks like "shared with another record".
- **Narrow a safety rule; don't delete it.** Measure how many files the narrower rule releases against removing the guard. If the numbers are the same, keep the guard.
- **Copy a live behaviour exactly before retiring it.** When the CloudFiles flow was replaced, its rule was treated as the specification and captured into git first.
- **Give a destructive job a dry-run switch** that can be flipped from Setup without a deploy.
