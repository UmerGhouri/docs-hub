# Dochly "Download All PDFs" downloaded 1 of 6 documents

| | |
|---|---|
| **Ticket** | DDT-887 |
| **Project** | Dochly |
| **Status** | Done 2026-09-01 |
| **Priority** | Medium |
| **Reported by** | Case staff, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation, the case Files tab |
| **Area** | Case setup: downloading the opening document package |
| **Related** | DDT-1074 (the download window closing early, with the same workaround) |
| **Code** | None |

---

## 1. Summary

A staff member generated six opening documents for one client in a single Dochly run and clicked "Download All PDFs". Dochly downloaded only one and warned that the rest were skipped, even though it listed all six as generated. The ticket was closed by pointing the user to where all six were already saved: the **DDT Generated Documents** folder on the case's Files tab. The download behaviour itself was not changed.

## 2. What was reported

One case, six documents. Dochly showed: **"Some files skipped. 1 of 6 documents could be downloaded."** The same screen listed all six documents under "PDF generated successfully":

- Appearance - Lake County
- Lake PTRSSS (Custom)
- Motion For Discovery (DUI)
- Rule 214 Request to Produce (DUI)
- Rule 237 Notice to Produce
- Notice of Filing

The user had to download each PDF separately, which slowed down case setup.

## 3. Scope

**In scope**

- Telling the user where the generated PDFs are stored and how to get them.

**Out of scope**

- **Changing the "Download All PDFs" behaviour.** Nothing in the ticket records a change to it.

## 4. How it works (background)

```
User selects several templates on a Court Case and generates them in one run
  -> Dochly generates each PDF
  -> each PDF is saved to the case's "DDT Generated Documents" folder (Files tab)
  -> "Download All PDFs" offers them for download in the browser
```

The screenshot attached to the resolution shows the case's Files tab with the **DDT Generated Documents** folder holding the generated PDFs.

## 5. Root cause

Not recorded in the ticket. The ticket asked whether the cause was the Dochly integration, permissions on specific templates, or the browser or file size; none of these was confirmed.

## 6. Solution

The resolution on the ticket: generate the documents again, or open them from the **DDT Generated Documents** folder under the **Files** tab. The generated PDFs are saved there, separately from the browser download.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

When "Download All PDFs" skips files:

1. Open the Court Case record.
2. Go to the **Files** tab.
3. Open the **DDT Generated Documents** folder.
4. Download the PDFs from there.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The bulk download can still skip files.** The cause was not found, so the workaround is the fix.
- A similar symptom came up three weeks later in DDT-1074, where the download window closed after a second and the user again downloaded from the DDT Generated Documents folder. That one was reported fixed on 2026-09-22.

## 11. Reuse for similar issues

**How to recognise one:** Dochly says a document was generated, but the browser download is missing, partial or closes too fast.

- **Check the DDT Generated Documents folder first.** Generation and download are separate steps; if generation succeeded, the file is there.
- **Separate "was it generated" from "did the browser download it"** before investigating. The first is a Dochly or template problem; the second is a browser or download problem.
