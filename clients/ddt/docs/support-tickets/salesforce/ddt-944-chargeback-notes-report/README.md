# Report of accounts with chargeback notes

| | |
|---|---|
| **Ticket** | DDT-944 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Files, invoice notes) |
| **Area** | Billing: reviewing chargebacks |
| **Related** | None |
| **Code** | Not applicable: a one-off data pull, no code change |

---

## 1. Summary

The firm saw a rise in chargebacks and asked for one list of every account with chargeback notes, so the team could look for patterns. The ticket assumed chargebacks were tracked in a "Chargeback" folder in Salesforce Files. In fact that folder existed for only 9 of the 30 affected accounts, so the list was built from the invoice notes instead and delivered as a PDF register.

## 2. What was reported

Billing staff wanted every account with chargeback-related notes, showing the client, the date, the amount and the reason, to understand what was driving a recent increase in chargebacks. The request was for a Salesforce report if possible, or an export if the data did not sit in a reportable object.

## 3. Scope

**In scope:** a one-time register of every account with a chargeback, back to July 2024, with client, date, amount and reason.

**Out of scope:** a scheduled or repeatable report. Whether one was wanted is not recorded in the ticket. No Salesforce metadata was changed.

## 4. How it works (background)

```
A client disputes a payment with their card issuer (a chargeback)
  -> staff record the details (reason, actions taken) in the invoice notes
  -> sometimes, supporting files are also saved in a "Chargeback" folder in Files
```

Standard report types do not cover Files well, which is why the ticket allowed for an export instead of a report.

## 5. Root cause

Not applicable: this was a request for a report, not a fault.

**The ticket's first assumption was wrong.** It said chargebacks are tracked in a "Chargeback" Files folder. Only 9 of the 30 accounts with a chargeback had that folder. The invoice notes were the complete source, so building the list from the folder would have missed two-thirds of the cases.

## 6. Solution

The chargeback register was built from the invoice notes rather than the Chargeback folders. It covers all 30 accounts back to July 2024, with the client, the date, the amount and the reasons, and was sent to the requester as a PDF attached to the ticket on 2026-09-09.

The exact query used to find the notes is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The cross-check that is recorded: the register's 30 accounts were compared with the Chargeback folders, which exist for 9 of them.

## 8. Step by step

To produce the register again:

1. Find every invoice whose notes mention a chargeback, and the account it belongs to.
2. For each, take the client, the chargeback date, the amount and the reason from the notes.
3. Cross-check against the "Chargeback" folders in Files, but do not rely on them: most chargebacks have no folder.
4. Export the list and deliver it privately. It names clients, so it must not be posted anywhere public.

## 9. Rollback

Not applicable: nothing in Salesforce was changed.

## 10. Known limits

- **The register is a snapshot** as of 2026-09-09. New chargebacks are not added to it.
- **It depends on staff writing chargebacks into the invoice notes.** A chargeback recorded only somewhere else would be missed.
- **The Chargeback folders are incomplete**, so they cannot be used as the source for a repeatable report.

## 11. Reuse for similar issues

**How to recognise one:** a request for "all records of type X" that names where X is stored.

- **Check that the named location is complete** before building from it. Count it against another source first.
- **Notes and free text are often the only complete record** of a manual process. Expect to read them rather than report on a field.
- **Ask whether the request is one-off or repeatable.** A repeatable report needs the data in a reportable field, which may mean adding one.
