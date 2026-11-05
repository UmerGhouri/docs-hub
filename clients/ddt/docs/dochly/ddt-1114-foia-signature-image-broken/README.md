# FOIA letter shows a broken signature image

| | |
|---|---|
| **Ticket** | DDT-1114 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-24 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor) |
| **Area** | PI cases: FOIA requests to police departments for crash reports |
| **Related** | None recorded |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

The FOIA request letter generated from Dochly printed a generic broken-image icon (the "tree" placeholder) where the firm's signature should be. The reporter said the "tree" was *back*, so this was a recurrence of an earlier problem. The template was fixed in Dochly the same day.

## 2. What was reported

A generated FOIA letter showed:

- the broken-image placeholder in the signature block instead of the signature;
- a blank recipient block: "VIA CERTIFIED MAIL" followed by a lone comma, and a stray comma on the next line, with no agency name or address;
- "RE: FOIA Request - Crash Report #" with nothing after the `#`;
- "...near ." with the accident location missing from the narrative sentence.

Why it matters: FOIA letters go out by certified mail to police departments to obtain crash reports. A letter with no signature, no recipient, no report number and no location is not sendable, and every PI case that needs the police report waits on it.

The ticket names one affected letter. How many other letters were affected is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `FOIA` | Dochly template | Signature image restored |

**Out of scope**

- **The three blank merge fields.** The ticket asked whether the blank recipient, crash report number and location came from the same template problem or from missing data on the case. What was found, and whether anything was changed for them, is not recorded in the ticket.

## 4. How it works (background)

```
Staff open a PI case and generate the FOIA letter from Dochly
  -> Dochly merges case fields into the FOIA template
  -> the template's signature image is loaded into the document
  -> the PDF is printed and mailed by certified mail
```

If an image in a template cannot be loaded when the document is rendered, the PDF shows the viewer's broken-image icon in its place. The rest of the document still renders, so the problem is easy to miss until someone reads the printed letter.

## 5. Root cause

Not recorded in the ticket. The closing comment says only that it was fixed. The ticket also asked for a note on what had changed since the earlier fix, so the next recurrence could be caught faster; that note is not recorded either.

## 6. Solution

The FOIA template was corrected in Dochly's template editor so the signature image renders again. The exact change made to the image reference is not recorded in the ticket.

## 7. Testing

The fix was handed back to the reporter to check. Whether a letter was regenerated and inspected is not recorded in the ticket.

**Not covered:** there is no automated check on Dochly template output. A broken image is found only when someone reads a generated letter.

## 8. Step by step

1. **Generate the FOIA letter** on a PI case and confirm the broken-image icon appears in the signature block.
2. **Open the `FOIA` template** in Dochly's template editor and find the signature image.
3. **Repair the signature image** in the template. Exactly what was changed is not recorded in the ticket; see section 11 for what to check.
4. **Generate the letter again** and check the signature, the recipient block, the crash report number and the accident location.
5. **Write down what changed** on the ticket, so a recurrence can be compared against it.

## 9. Rollback

Not applicable: rolling back would bring the broken image back.

## 10. Known limits

- **This has happened before.** The reporter said the placeholder was "back". Without a recorded cause, the next recurrence starts from nothing.
- **The blank fields.** Whether the recipient, report number and location fields were fixed, or depend on data being filled in on the case, is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** a generated Dochly document shows a small broken-image icon where a logo or signature belongs, while the text around it renders normally.

- **Check the image reference first.** An image that a template loads from somewhere the generating user cannot reach renders as the broken-image icon, not as an error.
- **Generate as the person who will use it.** A template that works for an administrator can still show a broken image for someone else.
- **When a fix recurs, record the cause on the ticket.** This ticket asked for that note and did not get one, so the history of the earlier fix is lost.
- **Read the whole document, not just the reported part.** Here one report surfaced four separate defects on the same page.
