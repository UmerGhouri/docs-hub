# Signed Proof of Representation PDF doesn't match the original layout

| | |
|---|---|
| **Ticket** | DDT-1509 |
| **Project** | Dochly |
| **Status** | Done 2026-11-05 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-11-04 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature (signed PDF, envelope email) |
| **Area** | The final signed PDF for the Medicare Proof of Representation |
| **Related** | None recorded |
| **Code** | None |

---

## 1. Summary

After every signer completed a Dochly envelope for the Medicare Proof of Representation, the final signed PDF no longer looked like the original: different fonts and spacing, misaligned fields, shifted signatures, and a garbled character in the page header. On the same ticket, staff asked how to put the client's full name in the envelope email instead of the Matter name; the answer was the `{{Account.Name}}` merge field. The ticket was closed on 2026-11-05 without recording what was changed for the layout.

## 2. What was reported

Staff compared the signed copy with the original, on a test envelope in production, and listed:

- **Fonts and spacing.** The page was re-rendered in a sans-serif form text instead of the original serif layout. Page breaks and spacing differed.
- **Fields misaligned.** Filled values (attorney, firm name, address, telephone) sat above or offset from their underlines, and one line was shifted.
- **Signatures and dates moved.** They sat on different lines from the original signature lines.
- **Garbled header.** Every page's header showed `Â·` where a `·` separator should be. The ticket suggests a UTF-8 encoding problem.
- **A possible data difference.** The address and telephone in the signed copy differed from the original screenshot, and the telephone was formatted differently. The ticket notes this may just be two different test records, and asked for the merge-field source to be checked.

Why it matters, from the ticket: the signed Proof of Representation is submitted to Medicare, and a malformed form risks rejection. The same problem might affect other Dochly templates.

**Follow-up, the same day.** Staff tried to change the envelope's email subject and body, but the merge they used gave the Matter name rather than the client's full name, and asked which field to use.

## 3. Scope

**In scope**

- The final signed PDF of the `Proof of Representation` Dochly template.
- The envelope email's subject and body, for the client name.

**Out of scope:** Not recorded in the ticket. It asked whether other templates were affected; the answer is not recorded.

## 4. How it works (background)

```
Dochly template (Proof of Representation)
  -> merged with the record's fields into the document
  -> sent for signature; each signer signs on Dochly's signing page
  -> when all have signed, Dochly re-renders the document with the
     signatures embedded and adds a header to each page
  -> final signed PDF
```

The signature request keeps a reference to its template because Dochly needs it to re-render the document with the signatures on completion. That re-rendering is where the signed copy can differ from what was sent.

## 5. Root cause

**Layout:** Not recorded in the ticket.

**Client name in the email:** the merge the requester used returned the Matter's name, not the client's. The support team's answer was the `{{Account.Name}}` merge field.

**One fact from production (2026-11-05):** the `Proof of Representation` template record's last-modified date is 2026-10-26, before this report. The ticket does not say where the layout fix, if any, was made.

## 6. Solution

**Client name in the email (recorded).** The support team told the requester to add `{{Account.Name}}` to the email subject and body for the client's name.

**Layout, header encoding and the address source:** not recorded in the ticket. No commit or pull request in the `ddt-salesforce` repo references DDT-1509.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for:

- the signed PDF to visually match the original (fonts, alignment, signature and date positions),
- no garbled characters in the header,
- the address and telephone values checked against their source,
- a re-test with the Proof of Representation template and one other template.

None of these results is recorded.

## 8. Step by step

**To show the client's name in a Dochly envelope email:**

1. Open the envelope's email subject or body.
2. Use `{{Account.Name}}` where the client's name should appear, not the Matter's name merge.
3. Send a test envelope and check the email.

**For the layout,** the steps used are not recorded. To check a template for the same problem, send a test envelope, sign it as every signer, and compare the final PDF with the original page by page: fonts, field alignment, signature and date positions, and the header text.

## 9. Rollback

Not applicable: the email change is a merge field in the email text, and the layout change is not recorded.

## 10. Known limits

- **The layout outcome is not recorded.** If the signed copy still differs from the original, this ticket gives nothing to go on beyond the list of symptoms in section 2.
- **Other templates** may be affected; the ticket did not record checking them.
- **`{{Account.Name}}` was given for this case only.** Whether it resolves to the client on every record a Dochly envelope can be sent from is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** a signed Dochly PDF looks different from the document that was sent, or shows odd characters such as `Â` in its header.

- **Compare the signed copy with the sent copy**, not with the template preview. Dochly re-renders the document when the last signer signs.
- **`Â` before a symbol** is the usual sign of UTF-8 text read as a single-byte encoding. Look at the header text and the separator character.
- **When a merge returns the Matter's name instead of the client's**, use `{{Account.Name}}`, and check the result on a test envelope.
- **Record what was changed** when closing a layout ticket, so the next one doesn't start from symptoms alone.
