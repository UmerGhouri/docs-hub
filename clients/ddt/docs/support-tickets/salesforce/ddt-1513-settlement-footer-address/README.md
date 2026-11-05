# Settlement statement footer address and website

| | |
|---|---|
| **Ticket** | DDT-1513 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-05 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-11-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Visualforce page, LWC) |
| **Area** | PI settlement statements sent to clients |
| **Related** | None recorded |
| **Code** | PR #894 in the `ddt-salesforce` repo |

---

## 1. Summary

Settlement statements were going out with the old office address and the old driver-defense website in the footer. The footer text is hard-coded in two places: the on-screen preview (an LWC) and the Visualforce page that renders the PDF the client actually receives. Both copies were updated to the new office address and the injury website, with the phone label and separator changed to match the requested text.

## 2. What was reported

- **2026-11-04:** the settlement statement footer still showed the previous office address and the previous website. The requester gave the exact new footer text: the new office address and the injury website on the first line, and the phone (labelled "P:") and email, separated by a pipe, on the second. Website and email were to be links if the template supports them.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `SettlementStatementPDF` | Visualforce page | Footer copy updated; this is the PDF the client receives |
| `settlementStatement` | LWC | Footer copy updated; this is the on-screen preview |

**Out of scope**

- **Collections demand-letter templates.** They also carry the old address, but they are a different document and belong in their own ticket.
- **Phone number and email address.** Kept as they were; only the phone label and separator changed.

## 4. How it works (background)

```
Settlement statement
  -> settlementStatement LWC        on-screen preview, with its own footer copy
  -> SettlementStatementPDF page    renders the PDF sent to the client, with its own footer copy
```

The footer is plain text in each file, not a shared setting, so the two copies must be changed together.

## 5. Root cause

The footer text was hard-coded with the old office address and website, in two separate files.

**The first fix missed the one that mattered.** The first commit changed only the LWC footer. Review of PR #894 caught that the LWC is only the on-screen preview; the statement that reaches the client is rendered by the Visualforce page, which carried its own copy of the footer and still showed the old details. That PDF was the actual failure the ticket was raised for.

## 6. Solution

Both footers now read: the new office address, a pipe, the injury website; then "P:" with the phone number, a pipe, "E:" and the email address.

In the LWC:

```html
<div class="ddt-letter-footer">
    NEW OFFICE ADDRESS |
    <a href="WEBSITE" target="_blank" rel="noopener">WEBSITE</a><br />
    P: PHONE | E:
    <a href="mailto:EMAIL">EMAIL</a>
</div>
```

In the Visualforce page, the same text with plain links (`a href`) for the website and the `mailto:` email.

Why it's done this way:

- **The website link in the LWC opens in a new tab** (`target="_blank" rel="noopener"`), so clicking it does not navigate the Lightning app away from the record.
- **The "E:" label was kept** to match the requested text exactly.
- **Both files were run through prettier** so the format check passes.

## 7. Testing

- Deployed to production on 2026-11-05: 2 of 2 components (the Visualforce page and the LWC), with `RunSpecifiedTests` passing 106 of 106 and 0 failures.
- After deploy, both footer copies were reported to show the new address and website (clickable), the "P:" label and the email link.

**Not covered:** no automated test checks footer text, and the Apex tests run during the deploy do not render the page or the component. Which tests were named is not recorded in the ticket.

## 8. Step by step

1. Find every copy of the old text in the repo, not just the first one:
   ```bash
   git grep -n "<old address or website text>" -- force-app
   ```
2. Work out which copy the client actually receives. Here, the Visualforce page renders the PDF; the LWC is only a preview.
3. Update every copy with the exact requested text.
4. Run prettier on the changed files.
5. Deploy both components together to a sandbox, generate a statement and open the PDF, then deploy to production.
6. Generate a new statement in production and check both the preview and the PDF.

## 9. Rollback

Revert the two files and redeploy them. Text only; no data is written, so it is safe. Statements already generated keep whatever footer they were generated with.

## 10. Known limits

- **The footer is still hard-coded in two places.** The next change of address has to touch both, and other documents (such as the Collections demand letters) have their own copies.
- **Already-generated statements are not changed.** Only newly generated statements show the new footer.
- **PR #894 was still in review** when the change was deployed to production.

## 11. Reuse for similar issues

**How to recognise one:** a request to change fixed text (address, phone, website, signature) on a document.

- **Search the whole repo for the old text** before changing anything. Documents often have a preview copy and a rendered copy.
- **Fix the copy the client receives first.** Check which component produces the PDF or email that actually goes out.
- **Check the generated output**, not just the screen. A preview can look right while the PDF is still wrong.
- **List other documents with the same text** and raise them separately rather than widening the ticket.
