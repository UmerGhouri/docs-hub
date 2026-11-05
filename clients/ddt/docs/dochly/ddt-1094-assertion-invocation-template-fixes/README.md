# Assertion of Rights missing officer details; Notice of Invocation printed empty Phone and Fax lines

| | |
|---|---|
| **Ticket** | DDT-1094 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-23 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation |
| **Area** | Court filings: Assertion of Rights and Notice of Invocation of Rights/Counsel |
| **Related** | ROS-5 (a contract printing a fee as "TBD.00" instead of "TBD") |
| **Code** | None. The fixes were made in the Dochly template editor. |

---

## 1. Summary

An attorney generated two Dochly documents and found three faults. The Assertion of Rights did not print the police officer's details. The Notice of Invocation of Rights/Counsel, a newly created template, printed "Phone:" and "Fax:" labels with nothing after them when the recipient had none, and centred the attorney's signature block instead of placing it at the lower right. All three were fixed in Dochly's template editor.

## 2. What was reported

From two documents generated for one client:

1. **Assertion of Rights:** the PO (police officer) information was not populated. The ticket's reading was that the PO merge fields were missing or not mapped.
2. **Notice of Invocation of Rights/Counsel:** when the recipient had no phone or fax, the document still printed empty "Phone:" and "Fax:" lines. Both are optional and should be hidden when blank.
3. **Notice of Invocation of Rights/Counsel:** the attorney's signature block was centred. It should be in the lower-right corner, right-aligned.

Each document needed manual cleanup before it could be sent.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| Assertion of Rights | Dochly template | PO information merges |
| Notice of Invocation of Rights/Counsel | Dochly template | Phone and Fax lines hidden when blank |
| Notice of Invocation of Rights/Counsel | Dochly template | Signature block right-aligned |

**Out of scope**

- Any other Dochly template.

## 4. How it works (background)

```
Attorney generates the document from the case
  -> Dochly merges the client, officer, recipient and attorney details
  -> optional lines print whether or not they have a value, unless the template hides them
  -> the PDF is generated
```

## 5. Root cause

Not recorded in the ticket. For the PO information the ticket suggested the merge fields were missing or not mapped; that was not confirmed on the ticket. The empty Phone and Fax lines are what a template prints when a label is written as fixed text with no condition on its value.

## 6. Solution

The completion note reads: "All the fixes have been completed." The merge fields and conditions used are not recorded.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked the attorney to regenerate both documents and confirm all three fixes; that confirmation is not recorded.

## 8. Step by step

1. In the Assertion of Rights template, add or correct the merge fields for the police officer's details, then generate it on a case that has an officer recorded.
2. In the Notice of Invocation of Rights/Counsel template, wrap the "Phone:" and "Fax:" lines in a condition so each prints only when its value is present.
3. Right-align the attorney's signature block and place it at the bottom right.
4. Generate the Notice once for a recipient with phone and fax, and once for one without, and check both.

## 9. Rollback

Restore the previous template versions in Dochly, if kept. That brings back the faults, so there is no reason to.

## 10. Known limits

- **The fixes are per template.** Other templates with optional contact lines or officer details need the same treatment on their own.

## 11. Reuse for similar issues

**How to recognise one:** a generated document has a label with nothing after it, or a section that should hold record data is empty.

- **Make every optional line conditional**, label and value together. A label written as fixed text prints even when its value is blank.
- **Test a new template with a sparse record** as well as a full one. Empty values are where most faults show.
- **Check layout against a filed copy.** Signature placement is a convention the template has to copy, not something it gets right by default.
- **When a whole group of fields is empty, check the mapping first.** One missing field is usually a typo; a whole block missing usually means the template points at the wrong record or relationship.
