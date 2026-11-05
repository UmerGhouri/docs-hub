# Five Will County court filing templates in S-Docs and Dochly

| | |
|---|---|
| **Ticket** | DDT-598 |
| **Project** | Dochly |
| **Status** | Live since 2026-08-13 |
| **Priority** | Medium |
| **Reported by** | Support team (planned work), 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | S-Docs, Dochly document generation |
| **Area** | Court filings: generating standard opening documents for Will County cases |
| **Related** | None recorded |
| **Code** | None. The templates were built in the S-Docs and Dochly template editors; no commit in the `ddt-salesforce` repo references this ticket. |

---

## 1. Summary

Attorneys needed five standard Will County court filings that they could generate from a case record instead of drafting by hand. Five templates were built, in both S-Docs and Dochly: Petition to Rescind, Rule 214, Motion for Discovery, Notice of Filing and Appearance by Attorney.

## 2. What was reported

This was planned work, not a fault. The ticket asked for five Will County filings, each with the county's caption, court name, judge, case number and certificate-of-service conventions merged in from Salesforce:

1. **PTR**: Petition to Rescind a statutory summary suspension, common in Illinois DUI cases. The ticket notes these have hard statutory deadlines, so a faster template lowers the risk of missing one.
2. **Rule 214**: Illinois Supreme Court Rule 214 request to produce documents.
3. **Motion for Discovery**: formal request for discovery from the State.
4. **Notice of Filing**: the cover notice that goes with a court filing.
5. **Appearance by Attorney**: the attorney's formal entry of appearance.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| PTR, Rule 214, Motion for Discovery, Notice of Filing, Appearance by Attorney | S-Docs | Five new templates |
| The same five | Dochly | Five new templates |

**Out of scope**

- **Other counties.** The ticket asked whether Cook, DuPage, Lake, Kane and others need the same set. That was left as an open question; this ticket covers Will County only.
- **Repo documentation.** The ticket asked for a `docs/templates/will-county-filings.md` page with the merge-field map and sample PDFs. That file does not exist on `main` in the `ddt-salesforce` repo.

## 4. How it works (background)

```
User opens a Court Case record
  -> picks a template (S-Docs or Dochly)
  -> the template merges record fields: client, case number, charges, court date, judge, attorney, firm details
  -> the generated PDF is saved against the record
```

The Will County details the ticket asked to build in were the court (Twelfth Judicial Circuit, County of Will), the county's caption style, the assigned judge merged from the record, the county's case-number format (year, case-type code, number), the signature block and the State's Attorney's Office for certificates of service.

## 5. Root cause

Not applicable: this was new work, not a fault.

## 6. Solution

The completion note on the ticket reads: "5 Will County court filing templates are created in both SDocs and Dochly."

Which merge fields each template uses, the conditional sections, and the button or action that launches each one are not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for a test generation against a real Will County matter and sign-off from an attorney who files there regularly; the ticket does not record either.

## 8. Step by step

The ticket records only the outcome. The plan it set out, for someone building the same set for another county:

1. Take a recently filed pleading from that county and copy its caption format exactly.
2. Build the template in the document tool with the county header, caption block and body.
3. Map merge fields from Salesforce: client name, case number, charges, court date, judge, attorney and firm details.
4. Generate each template against a real case in that county and check it is court-ready.
5. Make it available from the Court Case record.

## 9. Rollback

Deactivate or delete the templates in S-Docs and Dochly. No data depends on them, so removing them affects only future generations.

## 10. Known limits

- **The same five templates exist in two systems.** The ticket itself flagged that building them twice is wasted effort unless there is a reason to prefer one system, and asked for that reason to be written down. It is not recorded.
- **No merge-field map was written down**, so a later change to a source field has no record of which templates depend on it.

## 11. Reuse for similar issues

- **Copy a real filing first.** County caption and case-number conventions are easiest to get right by matching a pleading the clerk has already accepted.
- **Merge the judge and case number from the record**, never type them into the template.
- **One template per county per filing** is the pattern used here. The ticket raised a single template with county-conditional sections as the alternative to weigh before building the next county.
- **Write the merge-field map down** when you build a template, so a field change can be traced to the templates it affects.
