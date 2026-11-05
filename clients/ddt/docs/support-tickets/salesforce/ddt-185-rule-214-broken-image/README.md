# Rule 214 template generating a broken image

| | |
|---|---|
| **Ticket** | DDT-185 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-18 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs document templates |
| **Area** | Generating the Rule 214 document for a court case |
| **Related** | None |
| **Code** | None. The fix was made in the org, not in the repo. |

---

## 1. Summary

The Rule 214 document generated through S-Docs showed two broken-image icons around its court heading instead of the intended graphic. The support team found the cause in the template and fixed it in production the same day, and the staff member who reported it confirmed a freshly generated document looked right. The exact cause and change are not recorded in the ticket.

## 2. What was reported

On 2026-06-17 a staff member who generates the Rule 214 document almost every day saw, for the first time, two broken-image icons in it: one near the top left and one in the body, around the court heading. A second staff member was seeing the same thing.

Why it matters: the Rule 214 document goes to courts and opposing counsel, so broken images look unprofessional and suggest the letterhead or seal isn't rendering.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Rule 214 | S-Docs template | The failing image reference was fixed. The exact change is not recorded in the ticket. |

**Out of scope**

- **Other S-Docs templates.** The ticket only named Rule 214.
- **The ddt-salesforce repo.** S-Docs templates are stored as records in the org, not as metadata, so there is nothing in git for this fix.

## 4. How it works (background)

```
Staff member opens the court case in Salesforce
  -> runs S-Docs and picks the Rule 214 template
  -> S-Docs merges the record's fields into the template
  -> images in the template are loaded from wherever the template points
  -> the finished document is produced
```

If an image the template points to can't be loaded when the document is built, the output shows a broken-image icon in its place. The rest of the document still generates, which is why the text looked fine.

## 5. Root cause

Not recorded in the ticket. The ticket listed the likely causes to check: a stale reference to a Salesforce Document or File, a broken image link, or a permissions problem on the source image. The support team's comment says only that the problem was found and fixed.

One useful clue from the report: the template had been working every day until 2026-06-17. Something the template depends on changed, rather than the template having always been wrong.

## 6. Solution

The support team fixed the image reference in the Rule 214 S-Docs template in production on 2026-06-17, then asked the reporter to generate a fresh Rule 214 on a test case. What exactly was changed is not recorded in the ticket.

## 7. Testing

**Live check (2026-06-17):** the reporter generated a fresh Rule 214 on a real case after the fix and replied "All good now!"

**Not covered:** no automated test exists for S-Docs template output. The repo was not changed, so there is no test to add.

## 8. Step by step

For a broken image in an S-Docs template:

1. **Reproduce it.** Generate the document on a case where it fails and note where the broken icons appear.
2. **Open the template** in S-Docs and find the image tags in the template body.
3. **Check each image source.** For each one, confirm the Document, File or static resource it points to still exists, still has the same reference, and can be read by the users who generate the document.
4. **Fix the reference**, or restore or re-share the source image.
5. **Generate the document again** and confirm the images render.
6. **Ask the person who reported it** to generate a fresh one and confirm, and record the reply on the ticket.

## 9. Rollback

Not applicable: the fix restored the template to a working state, so there is nothing useful to roll back to.

## 10. Known limits

- **The root cause is not on record,** so the same failure could come back without anyone knowing what to look for. If it does, write down which image was broken and why.
- **S-Docs data isn't readable from the support team's API access today** (queries on the template object return an invalid-type error), so the template can't be inspected by query.

## 11. Reuse for similar issues

**How to recognise one:** a generated document shows broken-image icons, but its text and merge fields are correct.

- **The template body is usually fine; the image source isn't.** Check that what each image points to still exists and is readable by the generating user.
- **"It worked yesterday" means something outside the template changed.** Look for a deleted or replaced file, a changed sharing setting or an updated logo.
- **Prefer image sources that don't depend on the generating user's access**, so the document renders the same for everyone.
- **Record the cause on the ticket** when you fix it in Setup or a vendor's screen. With no code change there is no other trace of what was done.
