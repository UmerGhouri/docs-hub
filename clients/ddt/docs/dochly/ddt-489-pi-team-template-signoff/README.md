# PI team sign-off on their migrated Dochly templates

| | |
|---|---|
| **Ticket** | DDT-489 |
| **Project** | Dochly Template Migration |
| **Status** | Done 2026-10-01 |
| **Priority** | None set |
| **Reported by** | Support team (migration plan), 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Dochly templates |
| **Area** | Team review before the S-Docs templates are retired |
| **Related** | DDT-487 (SOS team sign-off), DDT-488 (Defense team sign-off), DDT-484 (template migration) |
| **Code** | Not applicable: a review step, no code |

---

## 1. Summary

Before the PI (personal injury) team's S-Docs templates were retired, the team was to review their Dochly replacements against real PI records and Matters and give written sign-off. The ticket was closed on 2026-10-01. Unlike the SOS and Defense tickets, it carries no comment, so the sign-off itself is not recorded on it.

## 2. What was reported

Not a fault report. A planned approval gate in the S-Docs to Dochly migration.

## 3. Scope

**In scope:** every PI template migrated from S-Docs to Dochly. The ticket does not list them by name.

**Out of scope:** building the templates (DDT-484) and the other teams' reviews (DDT-487, DDT-488).

## 4. How it works (background)

```
Template rebuilt in Dochly (DDT-484)
  -> team generates it against real PI records and Matters
  -> checks layout, merged data and output
  -> corrections made and re-checked
  -> written sign-off
  -> the S-Docs version can be retired
```

## 5. Root cause

Not applicable: this was an approval step, not a fault.

## 6. Solution

The ticket was moved to Done on 2026-10-01. Who approved, which templates were reviewed and whether any corrections were made are not recorded in the ticket.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

1. Walk the team through each of their migrated templates in Dochly.
2. Have them generate each one against real PI records and Matters and check layout, merged data and output.
3. Make any corrections and have the team re-check them.
4. Record the sign-off on the ticket before the S-Docs version is retired.

## 9. Rollback

Not applicable: an approval changes nothing in the org. A template found wrong later is fixed in Dochly's template editor.

## 10. Known limits

- **No written sign-off is attached to the ticket,** although the ticket's own acceptance criteria asked for one.
- **PI Dochly work was still moving during the review.** DDT-1070 moved the PI contract onto Dochly in late September 2026, days before this ticket closed. An approval at one date does not cover templates edited later.

## 11. Reuse for similar issues

- **Post the sign-off on the ticket before closing it,** with the approver's role and the list of templates covered. A ticket closed with no comment leaves no record that the gate was passed.
