# Attorney signature image on generated documents

| | |
|---|---|
| **Ticket** | DDT-230, DDT-258, DDT-499, DDT-629 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-04 |
| **Priority** | High |
| **Reported by** | Firm staff and an attorney, first on 2026-06-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (static resources, User field, formula field), document templates (S-Docs at the time) |
| **Area** | The attorney signature printed on court filings and other generated documents |
| **Related** | DDT-185 (another template image problem), DDT-1439 (S-Docs replaced by Dochly and uninstalled) |
| **Code** | None. Every fix was made in Setup in production, not in the repo. |

---

## 1. Summary

Four tickets, one mechanism. An attorney's signature reaches a generated document as an image file stored in Salesforce as a static resource, pointed to from a field on that attorney's user record. Each ticket was a break somewhere in that chain: a signature that was missing, one that showed the wrong image, one that had never been added for a new attorney, and one that printed far too large.

The first three were fixed by uploading or replacing the static resource and setting the user's **Signature URL** field. How the fourth (the oversized signature) was fixed is not recorded in the ticket.

## 2. What was reported

- **DDT-230, 2026-06-24.** One attorney's **Signature URL** field was blank, so documents produced for him went out without a signature. The firm had supplied his signature earlier.
- **DDT-258, 2026-07-01.** The same attorney reported that his signature block was showing the wrong image. A correct image was supplied in the Slack thread.
- **DDT-499, 2026-07-23.** A newly started attorney, the primary attorney for one county, had no signature in Salesforce at all, so that county's filings went out unsigned. The requester also asked for the attorney's bar registration number to appear on the paperwork. The firm asked that in future a new attorney's signature be added as soon as they start.
- **DDT-629, 2026-08-03.** On the **auto-generated** Appearance form, one attorney's signature rendered much larger than on the **manually generated** Appearance for the same attorney, filling most of the signature block. Both PDFs were attached for comparison.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Signature<FirstName>` static resources | Static resource (PNG, cache control Public) | One created (DDT-230), the same one replaced (DDT-258), one created (DDT-499) |
| `User.Signature_URL__c` | Text field on User | Set to the static resource path for the two attorneys concerned |
| Appearance document output | Document template / generation | Signature size corrected (DDT-629); what was changed is not recorded |

**Out of scope**

- **The formula and the templates' design.** No ticket changed how the signature is pulled into documents; each fix was to the data the chain reads.
- **Where the bar registration number was stored (DDT-499).** The ticket closed after the requester supplied it and the support team said "its done", but the field or template that now carries it is not recorded.

## 4. How it works (background)

```
Signature image (PNG)
  -> Setup > Static Resources: Signature<FirstName>, cache control Public
  -> User.Signature_URL__c = /resource/Signature<FirstName>
  -> Opportunity (Court Case).Signature__c, a formula on the case's Primary Attorney:
       IF(ISBLANK(Primary_Attorney__r.Signature_URL__c), '',
          IMAGE(Primary_Attorney__r.Signature_URL__c, FirstName + ' ' + LastName))
  -> document templates that print the primary attorney's signature
```

Two ways the Appearance was produced at the time:

- **Manually**, by a staff member generating the document from the record.
- **Automatically**, by the record-triggered flow `Ready_to_Create_Paper_Appeareance`, which created an S-Docs job on the Court Case and picked the template by county and courthouse (DuPage, Lake, Cook short form, Cook long form).

The templates themselves were records inside the document-generation package, not metadata in the repo, so which field each one merged for the signature is not recorded. `TrustMonthlyStatementPdfController` is a second, separate consumer: it prints the **running user's** `Signature_URL__c` on trust statements.

## 5. Root cause

### DDT-230: signature missing

The field was blank. Why it went blank is **not recorded in the ticket**. Production shows the attorney's current static resource was **created on 2026-06-25 at 11:01 UTC**, 22 minutes before the ticket closed, so the fix was a new upload plus setting the field, not a restore of an older file.

### DDT-258: wrong image

The field pointed at the right static resource; the image inside it was wrong. Production shows the same static resource **last modified on 2026-07-02 at 08:55 UTC**, ten minutes before the ticket closed. The file was replaced in place and the field did not need to change.

### DDT-499: new attorney never added

Nothing had been set up for the new attorney. There was no step in onboarding that added a signature when an attorney joined. A static resource for him was **created on 2026-07-24 at 08:21 UTC**.

### DDT-629: oversized signature

**Not recorded in the ticket.** One fact narrows it: that attorney's static resource was **last modified on 2026-04-14**, months before the fix on 2026-08-04, so the fix did **not** replace the image file. Both paths read the same image file, so the difference most likely lay in how each template sized it. The ticket does not confirm this.

Background that matters for anyone chasing this again: the `Signature__c` formula calls `IMAGE()` **without** a height or width, so wherever it is shown without a size set by the template, the image appears at its own pixel size. The signature images in production are not normalised. On 2026-11-05 there were 22 static resources named `Signature...`, ranging from about 5 KB to about 1.1 MB.

**Evidence queries (run 2026-11-05):**

```sql
SELECT Name, ContentType, BodyLength, CacheControl, CreatedDate, LastModifiedDate
FROM StaticResource
WHERE Name LIKE 'Signature%'
ORDER BY LastModifiedDate DESC
```

```sql
SELECT Signature_URL__c, IsActive
FROM User
WHERE Signature_URL__c != null
```

The second returned 26 users, 10 of them active. Every active user's URL pointed at a static resource that exists, though one uses a different naming pattern. Six inactive users pointed at static resources that no longer exist.

## 6. Solution

### 6.1 Missing signature (DDT-230) and new attorney (DDT-499)

1. Upload the signature image as a new static resource named `Signature<FirstName>`, cache control **Public**.
2. On the attorney's user record, set **Signature URL** to `/resource/Signature<FirstName>`.

This is the same pattern every working signature in the org follows.

### 6.2 Wrong image (DDT-258)

Edit the existing static resource and upload the correct file. Keep the name so the user field and every formula keep pointing at it.

### 6.3 Oversized signature (DDT-629)

The support team reported "its fixed" on 2026-08-04. What was changed is **not recorded in the ticket**.

### 6.4 Static resources are not in git

The repo's `staticresources/` folder is a snapshot from 2026-05-22, and the 2026-07-03 backfill excluded static resources on purpose. None of the signatures added by these tickets exists in the repo.

## 7. Testing

- **DDT-230:** a screenshot posted on the ticket showed the field populated, and the requester acknowledged it.
- **DDT-258:** marked done; the requester's confirmation is not recorded.
- **DDT-499:** the requester confirmed the signature now appears on the paperwork, noting it is "a little on the small end".
- **DDT-629:** marked fixed; a check on a fresh auto-generated Appearance is not recorded.

**Not covered:** no automated test exists for any part of this chain, and no ticket records a spot check of every attorney's signature, which DDT-230 asked for. The query in section 5 is that check.

## 8. Step by step

To add or fix an attorney's signature:

1. **Get a clean image** of the signature: PNG, cropped tight to the ink, at the size it should print. Compare its dimensions with an attorney whose signature already prints correctly.
2. **Setup → Static Resources.** To add one: **New**, name `Signature<FirstName>`, upload the file, set **Cache Control** to **Public**, save. To replace one: open it, **Edit**, upload the new file, save.
3. **Setup → Users →** the attorney **→ Edit.** Set **Signature URL** to `/resource/Signature<FirstName>`. Skip this step when only replacing the image.
4. **Check the formula.** Open a Court Case where this attorney is the Primary Attorney. The **Signature** field should show the image.
5. **Generate a test document** through both the manual path and any automated path that produces the same document, and compare the signature size.
6. **Run the spot-check query** from section 5 to confirm every active attorney's URL points at an existing static resource.

## 9. Rollback

- **Field:** clear or restore the previous value of **Signature URL**. Safe and immediate.
- **Static resource:** Salesforce keeps no version history for a static resource. **Download the current file before replacing it**, or there is nothing to roll back to.

## 10. Known limits

- **Onboarding has no signature step.** DDT-499 happened because nothing adds a signature when an attorney starts. Until that is part of onboarding, the next new attorney will hit it too.
- **Image size is not controlled in one place.** The formula sets no size, and the images in production range from about 5 KB to about 1.1 MB. A large or uncropped image can print oversized on any template that does not constrain it.
- **Not in git.** The signature images added since 2026-05-22 exist only in the org.
- **The document stack has changed since.** S-Docs was replaced by Dochly and uninstalled under DDT-1439, so the S-Docs templates these tickets were tested against no longer exist. The static resources and the user field are still in use.

## 11. Reuse for similar issues

**How to recognise one:** a generated document shows a missing, wrong or oddly sized signature for one attorney while others look fine.

- **Walk the chain from the end that is broken:** static resource exists, then user field points at it, then the case's Primary Attorney is the right person, then the template.
- **Read `CreatedDate` and `LastModifiedDate` on the static resource.** They tell you whether a fix replaced the file or changed something else, which is how this page establishes what was done when the tickets do not say.
- **Compare the two generation paths** when manual and automated output differ. They read the same image, so look at the template first.
- **Back up the file before replacing a static resource.**
- **Add the signature when an attorney is onboarded,** not when the first filing goes out unsigned.
