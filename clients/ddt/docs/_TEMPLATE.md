# <Short title: the symptom, in plain words>

| | |
|---|---|
| **Ticket** | DDT-<number> |
| **Project** | <Dochly / Support tickets / ...> |
| **Status** | Live since <date> / Sandbox only / Done |
| **Priority** | <Urgent / High / Medium / Low> |
| **Reported by** | <role>, <date> |
| **Resolved by** | Support team |
| **Systems** | <Salesforce (Apex / Flow / LWC), Dochly, ...> |
| **Area** | <the part of the process> |
| **Related** | <other DDT tickets> |
| **Code** | <PR number in the ddt-salesforce repo> |

---

## 1. Summary
Two or three sentences: what went wrong (or what was asked for), why, and what we changed.

## 2. What was reported
What the user saw, and how many records or people it affected. Counts, not names.

## 3. Scope
**In scope:** every component touched (class, flow, field, template, permission set), by API name.
**Out of scope:** what this deliberately does not change, and why.

## 4. How it works (background)
A short diagram or list of steps showing the part of the system involved, before the change.

## 5. Root cause
Why it happened, with the evidence: the query you ran and what it returned.
If the ticket's first guess at the cause was wrong, say so. That's the most useful thing to learn from.

## 6. Solution
What changed, with the code or config. Explain *why* each choice was made, especially when a simpler option looks like it would work and doesn't.

## 7. Testing
The tests, what each one proves, and whether it fails without the fix.
How it was checked live, with the query and its result.
What is **not** covered.

## 8. Step by step
The exact steps, in order, so someone can repeat this fix.

1. <step>

Screenshots are optional. To add one later, save it in an `images/` folder next to this README and link it under its step:
`![What it shows](./images/02-step.png)`
Only link images that exist. The site build fails on a missing image.

## 9. Rollback
How to undo it, and whether undoing it is safe.

## 10. Known limits
Edge cases, and things that would break this fix later.

## 11. Reuse for similar issues
The patterns someone can copy for a related problem, and how to recognise one.
