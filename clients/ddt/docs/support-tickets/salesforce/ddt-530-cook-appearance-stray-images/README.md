# Stray image artifacts on the generated Cook County Appearance

| | |
|---|---|
| **Ticket** | DDT-530 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, document generation template (S-Docs suspected in the ticket) |
| **Area** | Generating the Cook County Appearance form for a court case |
| **Related** | None recorded on the ticket |
| **Code** | None. The fix was made in the org, not in the repo. |

---

## 1. Summary

The Cook County Appearance document (form CCCR 0114) came out of Salesforce with two stray images on it: one at the top left above the "APPEARANCE" heading and one at the bottom left below the clerk's name line. The ticket called them "green image artifacts", but the green was the reporter's own highlighting, as her second screenshot showed. Regenerating didn't help. The support team fixed the template in production the next day. The exact cause and change are not recorded in the ticket.

## 2. What was reported

On 2026-07-27 a staff member sent a generated Appearance for one client showing two unwanted images, one top left and one bottom left. In the first screenshot she had highlighted them in green; a second screenshot showed the same document without the highlight. The images came back every time the document was regenerated.

Why it matters: the Appearance is filed with the Circuit Court of Cook County. Stray marks on a court filing look unprofessional and can draw questions from the clerk or a rejection.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Cook County Appearance | Document generation template | The two stray images were removed. The exact change is not recorded in the ticket. |

**Out of scope**

- **The ddt-salesforce repo.** Document templates are stored as records in the org, not as metadata, so there is nothing in git for this fix.
- **Other templates.** The ticket only named the Cook County Appearance.

## 4. How it works (background)

```
Staff member opens the court case in Salesforce
  -> picks the Cook County Appearance template
  -> the template's layout, images and merge fields are filled from the record
  -> a PDF is produced, ready to file
```

Anything placed in the template, including an image, appears on every document generated from it. That is why regenerating the document didn't remove the images: they were in the template, not in the case data.

## 5. Root cause

Not recorded in the ticket. The ticket asked for the stray images in the template's header and footer areas to be found and removed. The support team's only comment is that it was fixed.

## 6. Solution

The support team removed the two stray images from the Cook County Appearance template in production on 2026-07-28. The exact change is not recorded in the ticket.

## 7. Testing

- **Live check:** not recorded in the ticket. The ticket's acceptance criteria were a clean newly generated document and a clean regenerated document on an existing case.
- **Not covered:** no confirmation from the reporter is recorded.

## 8. Step by step

The ticket does not record the exact fix, so these are the steps to repeat it.

1. **Get a copy of the bad output** and note exactly where the stray marks sit.
2. **Open the template** in the document generation tool and look in its header, footer and top-of-body areas for image elements at those positions.
3. **Remove the stray images**, keeping any image that belongs on the form (a seal or logo).
4. **Generate a new document** on a test case, and **regenerate** one on an existing case.
5. **Ask the reporter to confirm** both look clean before closing.

## 9. Rollback

Not applicable: the change removed images that should never have been on the form.

## 10. Known limits

- **The change isn't recorded**, so if stray images return there is no record of what was removed last time.
- **S-Docs has since been retired** in favour of Dochly, so this template may no longer be the one in use.

## 11. Reuse for similar issues

**How to recognise one:** a mark, image or block that appears on every document generated from one template, whatever the case, and survives regenerating.

- **If it survives regenerating, it is in the template.** Case data changes per record; the template doesn't.
- **Ask for an unhighlighted copy.** The reporter's own highlighting can be mistaken for the problem, as the second screenshot here made clear.
- **Write down what you removed.** Template fixes leave no trace in git, so the ticket comment is the only record.
