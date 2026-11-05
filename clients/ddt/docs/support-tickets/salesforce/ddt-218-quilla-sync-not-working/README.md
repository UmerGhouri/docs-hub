# Quilla to Salesforce sync not working

| | |
|---|---|
| **Ticket** | DDT-218 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-24 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Quilla (external tool) |
| **Area** | Client records: information expected to arrive in Salesforce from another tool |
| **Related** | None recorded |
| **Code** | None. No commit or PR for this ticket exists in the `ddt-salesforce` repo |

---

## 1. Summary

A staff member reported that client logs and information from Quilla were not appearing in Salesforce, and was unsure whether they should be working in Quilla or only in Salesforce. The ticket was closed two days later with a note that the affected client's record had been fixed and that an implementation "for matters" was also done. What was wrong, and what was built, is not recorded.

## 2. What was reported

On 2026-06-22 a staff member, with a screen recording, reported that client logs and information from Quilla were not syncing to Salesforce. They also asked which tool the team should be working in. The ticket asked to confirm whether one user or everyone was affected. The answer is not recorded.

## 3. Scope

**In scope:** one client's record, which was fixed, and an implementation for Matters. The components involved are not recorded in the ticket.

**Out of scope:** Not recorded in the ticket.

## 4. How it works (background)

Not recorded in the ticket. Nothing named Quilla appears in the `ddt-salesforce` repo, and none of the package licences in production has a Quilla namespace (checked 2026-11-05). How Quilla data was meant to reach Salesforce is therefore not established.

## 5. Root cause

Not recorded in the ticket. The closing comment says only that the client's record was fixed and that the implementation for matters was done. Whether the record was missing data because a sync failed, or because the process had never covered Matters, is not stated.

## 6. Solution

Per the closing comment on 2026-06-24:

1. The affected client's record was corrected.
2. The same handling was implemented for Matters.

The exact change, and where it was made, is not recorded in the ticket. The guidance on whether staff should work in Quilla, Salesforce or both, which the ticket asked to be documented and shared, is also not recorded.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

The steps taken are not recorded. For a similar report, the ticket's own checklist is a reasonable start:

1. Confirm whether one user or everyone is affected.
2. Check the integration's status, recent sync logs, and any expired login or credential.
3. Compare one affected record with the source system to see exactly which fields are missing.
4. Fix the record, then fix whatever stopped it syncing, and check whether the same gap applies to other objects (here, Matters).
5. Write down which tool is the source of truth and share it with the team.

## 9. Rollback

Not applicable: the change itself is not recorded, so there is nothing specific to undo.

## 10. Known limits

- **The fix is undocumented.** If the problem returns, the investigation starts from scratch.
- **The workflow question was not answered on the record.** Staff asked whether to work in Quilla or Salesforce; any answer was given outside the ticket.

## 11. Reuse for similar issues

- **"Data from tool X is missing in Salesforce"** can mean a broken sync or a sync that never covered that object. Check which before fixing records by hand.
- **Fixing one record is not the fix.** Find out why the record was wrong, and whether other records or objects have the same gap.
- **Record the answer to the process question on the ticket.** "Which tool do we use?" is the part future staff will look for.
