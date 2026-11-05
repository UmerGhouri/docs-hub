# Assertion of Rights printed the same client and police department on every document

| | |
|---|---|
| **Ticket** | DDT-1181 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor) |
| **Area** | Criminal and traffic matters: the client's assertion of rights notice |
| **Related** | None recorded |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

The Dochly template "Assertion of Rights" had a client's name and a police department typed into its main paragraph as plain text. Every document generated from it named that same client and that same department, whoever it was generated for. Both were replaced with merge fields.

## 2. What was reported

A document generated for one client still named a different client in the sentence "...I, [name],..." and still named one specific police department in "...any member or agent of the [department]...". The same two values appeared on every document from this template.

Why it matters: this is a client-facing legal notice. A wrong client name or a wrong arresting agency makes the document unusable and is a professional risk.

How many documents had gone out with the wrong values is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Assertion of Rights` | Dochly template | Hardcoded client name replaced with a merge field for the client's full name |
| `Assertion of Rights` | Dochly template | Hardcoded police department replaced with a merge field for the arresting agency |

**Out of scope:** nothing else in the template was reported or changed.

## 4. How it works (background)

```
Staff generate "Assertion of Rights" from the case
  -> Dochly copies the template body
  -> merge tokens are replaced with the record's values
  -> any plain text in the body is copied exactly as typed
```

A merge token changes per record. Plain text never does.

## 5. Root cause

The client name and the police department were typed into the template as plain text instead of merge fields, so they were copied unchanged into every document. How they got there is not recorded in the ticket.

## 6. Solution

In Dochly's template editor, the two plain-text values in the paragraph were replaced with merge fields: one for the client's full name, one for the case's arresting agency. The exact merge-field API names used are not recorded in the ticket.

## 7. Testing

The ticket's acceptance criteria asked for the document to be verified on at least two test cases with different clients and different arresting agencies. Whether that was done is not recorded in the ticket; the closing comment says only that it was fixed.

**Not covered:** documents generated before the fix still carry the wrong values.

## 8. Step by step

1. **Generate the template** on two cases with different clients and different arresting agencies, and confirm both documents show the same hardcoded values.
2. **Open "Assertion of Rights"** in Dochly's template editor.
3. **Replace the typed client name** with the merge field for the client's full name.
4. **Replace the typed police department** with the merge field for the arresting agency.
5. **Save**, then generate on the same two cases and confirm each shows its own client and agency.

## 9. Rollback

Not applicable: reverting would put the wrong client and agency back into every document.

## 10. Known limits

- **A case with no arresting agency filled in** will print a blank where the agency belongs.
- **Already-generated documents** are not corrected by a template fix. They have to be regenerated.

## 11. Reuse for similar issues

**How to recognise one:** two documents from the same template, for different records, show the same name, place or number.

- **Generate for two different records and compare them.** Any value that is the same on both and should not be is hardcoded.
- **When building a template from a real document**, search the body for the original client's name, case number, agency and dates before publishing it.
- **Prefer a merge field over typed text** for anything that belongs to the case.
