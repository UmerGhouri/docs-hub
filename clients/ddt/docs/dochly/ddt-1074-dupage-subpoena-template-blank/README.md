# DuPage Subpoena Duces Tecum template opened blank and generated without its body

| | |
|---|---|
| **Ticket** | DDT-1074 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-22 |
| **Priority** | High |
| **Reported by** | Case staff, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation |
| **Area** | Court filings: subpoenas to police departments in DuPage County cases |
| **Related** | DDT-887 (generated files downloaded from the DDT Generated Documents folder) |
| **Code** | None. The fix was made in Dochly. |

---

## 1. Summary

The Dochly template "Subpoena Duces Tecum - Dupage" could not be edited: the editor sat on "Loading…" with a blank page. Documents generated from it held only the values typed into its prompt screen, with none of the subpoena body. The template was restored so it opens and edits normally, and a prompt for the recipient was added and merged into the "TO:" line. A second problem the reporter raised afterwards, the download window closing almost immediately, was reported fixed the next morning.

## 2. What was reported

- **Editor blank.** The edit button opened the editor with its toolbar but a blank white page stuck on "Loading…". No edits were possible.
- **Generated document missing its body.** Output contained only the prompt values, the officer's name and address and the court time, and none of the subpoena text.

The ticket's view was that both had one cause: the template's body was not loading or not stored. The subpoena could not be used at all.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| DuPage Subpoena Duces Tecum | Dochly template | Body restored and editable; recipient prompt merged into the "TO:" line |
| Download window after generation | Dochly | Reported fixed; the change is not recorded |

**Out of scope**

- Other subpoena templates in the same list, such as the Cook County and Kane County ones.

## 4. How it works (background)

```
User selects the subpoena template on a case
  -> Dochly shows "Fill in the details" prompts:
       what time is court (required)
       who the subpoena is being sent to (optional)
       the specific materials requested (optional)
  -> the template body merges the record fields and the prompt answers
  -> the PDF is generated and saved
```

The template's header is the State of Illinois, County of DuPage, in the Circuit Court of the Eighteenth Judicial Circuit, with a People of the State of Illinois v. Defendant caption and the case number.

## 5. Root cause

Not recorded in the ticket. The ticket's hypothesis, that the body was not loading or not stored, matches both symptoms, but it was not confirmed on the ticket.

## 6. Solution

**The template.** The support team's note on 2026-09-21 reads: "The template is editable, and the prefilled field has also been added."

The confirmation screenshots show:

- The template now listed as "DuPage - Subpoena Duces Tecum".
- The editor loading the full document: the DuPage court header, the caption, and the "SUBPOENA DUCES TECUM" title.
- The "TO:" line filled from the recipient prompt:

```
TO: {{prompt:Addressed_To}}
```

`{{prompt:Addressed_To}}` is a Dochly prompt merge field. It prints whatever the user typed into the "Who is this being sent to?" prompt.

How the body was restored is not recorded.

**The download window.** After confirming the template was fixed, the reporter said the download window appeared for about a second and closed by itself, on two cases, so the document had to be downloaded from the DDT Generated Documents folder. The support team replied "It's fixed now" on 2026-09-22 and the ticket closed. What was changed is not recorded.

## 7. Testing

The support team generated the subpoena on a test record and attached screenshots of the prompt screen and the filled document. The reporter confirmed the template was fixed the same day. A confirmation of the download window fix is not recorded. No other test is recorded.

## 8. Step by step

1. Open the template from Dochly's Templates tab and check whether the editor loads its body.
2. If it stays blank, restore the body from a working copy or a previously generated document.
3. Check every prompt on the "Fill in the details" screen has a matching merge field in the body, for example `{{prompt:Addressed_To}}` on the "TO:" line.
4. Generate it on a test record, fill in every prompt, and check the full subpoena text prints.

## 9. Rollback

Not applicable: the previous version could not be opened or used.

## 10. Known limits

- **The cause is not recorded**, so another template could lose its body the same way. A template that opens blank in the editor is the early sign.
- **The download window fix is unconfirmed.** If it closes early again, the DDT Generated Documents folder is the workaround.

## 11. Reuse for similar issues

**How to recognise one:** a Dochly template opens blank in the editor, or generates a document holding only the prompt answers.

- **A blank editor and a body-less document together mean the template body is the problem**, not the merge fields.
- **Keep a copy of every court template's body** outside the template, so it can be restored.
- **Name prompts clearly and reference them by name** in the body, such as `{{prompt:Addressed_To}}`, and check each prompt shows up in the output.
- **If the download fails, the file is still saved** in the case's DDT Generated Documents folder.
