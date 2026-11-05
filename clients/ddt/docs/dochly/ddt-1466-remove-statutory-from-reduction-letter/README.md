# "Statutory" removed from the Medicaid/Medicare reduction letter

| | |
|---|---|
| **Ticket** | DDT-1466 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-28 |
| **Priority** | Medium |
| **Reported by** | Firm operations (PI team), 2026-10-28 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor) |
| **Area** | PI: lien reduction requests to Medicaid and Medicare |
| **Related** | DDT-1380 (earlier template text change), DDT-1094 (earlier Dochly template fixes) |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

A legal wording correction. The Medicaid/Medicare reduction letter asked for "a statutory 1/3 reduction", and the word "statutory" should not appear. It was removed. At the requester's follow-up request, the letter's date was also changed to the full written format.

## 2. What was reported

The sentence in the reduction letter read:

> Pursuant to the Common Fund Doctrine, we respectfully request a statutory 1/3 reduction.

It should read:

> Pursuant to the Common Fund Doctrine, we respectfully request a 1/3 reduction.

A follow-up from the requester on the ticket asked for the date on the letter to use the complete format, for example "October 28, 2026".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Medicaid/Medicare Reduction` | Dochly template | The word "statutory" removed from the reduction request sentence |
| Same | Dochly template | Date changed to the full "Month D, YYYY" format |

The ticket asked for every template containing "statutory 1/3 reduction" to be found. Which templates were checked besides this one is not recorded in the ticket. The `Medicaid/Medicare Reduction` template was last modified in production on 2026-10-28, a few minutes before the ticket was closed (read on 2026-11-05).

**Out of scope:** no other wording was to change.

## 4. How it works (background)

The sentence is plain text in the template body, so every letter generated from the template prints it exactly as typed.

## 5. Root cause

Not applicable: a wording change requested by the firm, not a fault.

## 6. Solution

In Dochly's template editor, the word "statutory" was deleted from the sentence, leaving the rest of the wording as it was. The date was changed to the full format. The exact date format token used is not recorded in the ticket.

## 7. Testing

The ticket asked for a test generation to confirm the new wording. Whether one was run is not recorded in the ticket. The requester thanked the team on the ticket after the change.

## 8. Step by step

1. **Search the template bodies** for "statutory 1/3 reduction" to find every template that uses the sentence.
2. **Open each match** in Dochly's template editor and delete only the word "statutory".
3. **Change the date** to the full "Month D, YYYY" format if asked.
4. **Search again.** No template should contain "statutory 1/3 reduction".
5. **Generate one letter** and read the sentence and the date.

## 9. Rollback

Not applicable: the old wording was the error.

## 10. Known limits

- **Copies of the letter** made before the change still carry the old wording. Production also holds a separate template named `Reduction`; whether it uses the same sentence is not recorded in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** a request to change a phrase "in the letter".

- **Search for the exact phrase across all template bodies**, not just the template named in the request.
- **Change only the words asked for.** The acceptance criteria here explicitly said no other wording should change.
- **Search again afterwards** and expect zero matches.
