# Locating a signed S-Sign representation agreement

| | |
|---|---|
| **Ticket** | DDT-239 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-30 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Sign e-signature |
| **Area** | Retrieving signed client agreements |
| **Related** | None recorded |
| **Code** | Not applicable: no code or configuration changed |

---

## 1. Summary

A staff member could see that a client's S-Sign envelope was **Completed** but could not find the signed representation agreement from the envelope's related list. The signed PDF had been saved, but on the **client record and the court case**, not on the envelope. Nothing was broken and no change was needed: the staff member was pointed to the file, which they already had access to.

## 2. What was reported

- **2026-06-29:** one client signed a representation agreement through S-Sign. The S-Sign envelope (`SSign__SSEnvelope__c`) showed **Sign Status: Completed**, but the staff member could not open the signed document from the envelope's related list and could not move the case forward without it.

One envelope and one client were affected. The ticket did not report other cases.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The client's signed agreement file | Salesforce File | None. Located and shared with the requester |

**Out of scope**

- **Changing where S-Sign saves signed documents.** The file was saved where it should be. The confusion was about where to look.
- **Permissions.** The ticket asked whether access needed fixing. It did not: the requester already had access to the file.

## 4. How it works (background)

```
Staff send the agreement from S-Sign
  -> client signs
  -> S-Sign envelope moves to "Completed"
  -> the signed PDF is saved as a Salesforce File named "SIGNED_<document name> - <client> - <date>.pdf"
  -> the file is attached to the client record and the court case (Files related list)
```

The envelope record tracks the signing status. It is not where the signed file is kept.

## 5. Root cause

Not a fault. The signed agreement existed as a Salesforce File on the **client's record and on his court case**, under the **Files** related list. It was not on the S-Sign envelope, which is why it looked stuck.

The ticket listed three possible causes: a permission issue, a missing attachment, or an S-Sign sync gap. It was none of them. The file was present and the requester could already open it.

## 6. Solution

The support team found the file and told the requester where it was: the client's page → **Files**, file name starting with `SIGNED_Representation Agreement`. The same file also appears under the court case's Files.

No configuration, permission or data change was made.

## 7. Testing

Not applicable: no change was made. The ticket records only that the requester was given the location and asked to confirm it opened.

## 8. Step by step

To find a signed S-Sign document when the envelope shows **Completed**:

1. Open the **client record** (Account), not the S-Sign envelope.
2. Scroll to the **Files** related list.
3. Look for a file whose name starts with `SIGNED_`, followed by the document name, the client's name and the signing date.
4. If it is not on the client record, open the client's **court case** and check its Files related list.
5. Open the file to confirm it is the signed copy.
6. Only if the file is on neither record, investigate S-Sign itself (envelope status, sync errors).

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The S-Sign envelope's own related list does not show the signed PDF, so staff who start from the envelope will keep hitting this. Nothing was changed to address that.

## 11. Reuse for similar issues

**How to recognise one:** a document process reports success (Completed, Signed, Sent), but staff "can't find the file".

- **Check the parent records first.** Signed and generated files are usually attached to the client or the case, not to the tracking record (envelope, request, job).
- **Search by file name pattern** (`SIGNED_...`) before assuming the file was lost.
- **Confirm access before changing permissions.** If the file exists and the user can open it from the right record, there is nothing to fix.
