# Defense team sign-off on their migrated Dochly templates

| | |
|---|---|
| **Ticket** | DDT-488 |
| **Project** | Dochly Template Migration |
| **Status** | Done 2026-09-07 |
| **Priority** | None set |
| **Reported by** | Support team (migration plan), 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Dochly templates |
| **Area** | Team review before the S-Docs templates are retired |
| **Related** | DDT-487 (SOS team sign-off), DDT-489 (PI team sign-off), DDT-484 (template migration) |
| **Code** | Not applicable: a review step, no code |

---

## 1. Summary

Before the Defense team's S-Docs templates were retired, the team was to review their Dochly replacements against real records and give written sign-off. The ticket was closed on 2026-09-07 with one comment: "Approved no change required till now".

## 2. What was reported

Not a fault report. A planned approval gate in the S-Docs to Dochly migration.

## 3. Scope

**In scope:** every Defense template migrated from S-Docs to Dochly. The ticket does not list them by name.

**Out of scope:** building the templates (DDT-484) and the other teams' reviews (DDT-487, DDT-489).

## 4. How it works (background)

```
Template rebuilt in Dochly (DDT-484)
  -> team generates it against real records of their own
  -> checks layout, merged data and output
  -> corrections made and re-checked
  -> written sign-off
  -> the S-Docs version can be retired
```

## 5. Root cause

Not applicable: this was an approval step, not a fault.

## 6. Solution

The approval was recorded as a comment on the ticket on 2026-09-07: "Approved no change required till now". No corrections were raised.

The comment was posted by a member of the Support team. Which member of the Defense team gave the approval, and whether a separate written sign-off exists, is not recorded in the ticket.

## 7. Testing

Which templates were generated, and against how many records, is not recorded in the ticket.

## 8. Step by step

1. Walk the team through each of their migrated templates in Dochly.
2. Have them generate each one against real records of their own and check layout, merged data and output.
3. Make any corrections and have the team re-check them.
4. Record the sign-off on the ticket before the S-Docs version is retired.

## 9. Rollback

Not applicable: an approval changes nothing in the org. A template found wrong later is fixed in Dochly's template editor.

## 10. Known limits

- **"Till now" is the scope of the approval.** It covers the templates as they stood on 2026-09-07. A template edited later is not covered by it.
- **The S-Docs licence was already off.** It was switched off on 2026-09-01, six days before this sign-off, according to the DDT-1437 PR message.

## 11. Reuse for similar issues

- **Record who approved, not only that approval happened.** A one-line comment from the Support team is hard to trace back to the team that reviewed.
- **List the templates covered by the sign-off,** so a later change to one of them can be checked against what was approved.
