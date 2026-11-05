# Hired matter case management: status workflow, soft close and notes

| | |
|---|---|
| **Ticket** | DDT-77 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Hired Matter case management: case lifecycle, court appearances, notes |
| **Related** | DDT-74 (removing Case Closed from the sub-status paths) |
| **Code** | Not recorded in the ticket. No PR is linked. |

---

## 1. Summary

The ticket bundled four enhancements for Hired Matters: a status workflow, a soft close, a court appearances view, and one place for notes. The work recorded as delivered is on the **Court Appearance Panel**: staff can now log the result of a past appearance from a modal, schedule the next appearance with the details pre-filled, and close the case from the result when it resolves the matter.

## 2. What was reported

- **2026-06-01, DDT-77:** a request to clean up the Hired Matter lifecycle, surface court activity, and reduce scattered notes. It listed four pieces: status workflow, soft close, court appearances, and notes consolidation. The ticket itself suggested splitting them into sub-tickets during scoping.

## 3. Scope

**In scope (recorded as delivered)**

| Component | Type | Change |
|---|---|---|
| Court Appearance Panel | LWC | Result entry for past appearances, next-appearance prefill, case close option, modal layout fix |

**Out of scope, or not recorded**

- **Status workflow** (allowed transitions between statuses). Not recorded as delivered.
- **Soft close** (a status or flag that hides a matter from active views but keeps it reachable). Not recorded as delivered.
- **Notes consolidation** (one canonical Notes surface on the Matter). Not recorded as delivered.

If you need any of those three, check for a later ticket before assuming they exist.

## 4. How it works (background)

```
Hired Matter page
  -> Court Appearance Panel lists the matter's appearances
       past appearance   -> log its result
       result entered    -> optionally schedule the next appearance
       result resolves the case -> optionally close the case
```

Before the change, logging an outcome meant navigating to the appearance record.

## 5. Root cause

Not a defect: an enhancement. Logging an appearance's result and booking the next court date took several separate steps on separate records.

## 6. Solution

From the resolution note, the Court Appearance Panel now has:

- **Result entry for past appearances.** Users log the outcome directly from a modal, without opening the record.
- **Prefill for the next court date.** From the result, the next court date and time, court action, courtroom and judge are filled in from the result data.
- **"Schedule Next Appearance".** After saving a result, the New Appearance modal reopens with those values pre-filled.
- **Case close option.** When the result resolves the case, the case is marked **Resolved** and a confirmation toast is shown.
- **Modal size and layout fixed** so it renders consistently across screen sizes.

Why prefill from the result: the next appearance is almost always in the same courtroom before the same judge, so retyping it is wasted effort and a source of typos.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

Not recorded in the ticket. To check the delivered behaviour:

1. Open a Hired Matter with a past court appearance.
2. In the Court Appearance Panel, log a result for the past appearance from the modal.
3. Choose Schedule Next Appearance and check the courtroom, judge, court action and date are pre-filled.
4. On a test matter, enter a result that resolves the case and check the case is marked Resolved with a confirmation toast.

## 9. Rollback

Not recorded in the ticket. The panel can be removed from the Matter page in Lightning App Builder.

## 10. Known limits

- Three of the four requested enhancements are not recorded as delivered (section 3).
- The case close option marks the case **Resolved**. Anything that expects a different closed value will not see these cases as closed.

## 11. Reuse for similar issues

**How to recognise one:** a ticket bundling several separate features.

- Split it before building, as the ticket itself suggested. One ticket per feature keeps it clear what shipped.
- Record which pieces were delivered and which were not when closing, so the rest is not assumed done.
- When one action usually leads to the next (log a result, book the next date), pre-fill the second from the first.
