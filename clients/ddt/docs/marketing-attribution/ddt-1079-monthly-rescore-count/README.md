# Monthly count of re-scored attribution records in the Monday digest


| | |
|---|---|
| **Ticket** | DDT-1079 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-22 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership (asked on DDT-1026), 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, scheduled digest email) |
| **Area** | Marketing attribution: monitoring drift in credited channels |
| **Related** | DDT-1026, DDT-1078, DDT-822, DDT-826, DDT-947 |
| **Code** | PR #448 in the ddt-salesforce repo |


---


## 1. Summary

Under the v2 credit rules, records already stamped are re-scored one at a time whenever something reaches them, so channel totals can move with no deploy behind them. Firm leadership asked for a monthly count so that drift is observed rather than discovered. It was added as a new section of the existing Monday unmapped-token digest: no new object and no new scheduled job.

## 2. What was reported

- 2026-09-18 (on DDT-1026), filed 2026-09-22: a monthly count of stamped records re-scored under v2, split by rule version and object, ideally with how many changed channel or origin and the top channels gained and lost.

## 3. Scope

**In scope**

- `UnmappedTokenDigest` (section 3) and `UnmappedTokenDigestTest`.

**Out of scope**

- "How many changed" and "top five gained and lost": field history is not tracked on `Attributed_Channel__c` or `Attributed_Origin__c`, so there is no before value (see section 10).
- A new scheduled job: production was at 87 of 100 scheduled Apex jobs (DDT-947), so the digest's existing job was reused.

## 4. How it works (background)

Already-stamped records are re-scored by a new call touch (DDT-822), a referral capture, and since DDT-1078 a Campaign or how-heard change. Each re-score rewrites `Attributed_As_Of__c` and sets `Attribution_Rule_Version__c`. The `UnmappedTokenDigest` job already ran every Monday with recipients and a delivery path.

## 5. Root cause

Not applicable: new monitoring, not a defect.

## 6. Solution

The first digest of each calendar month (first Monday, Chicago time) carries section 3: records created before the previous month whose `Attributed_As_Of__c` was written during it, split by Opportunity and Lead and by `Attribution_Rule_Version__c`, with the v2 count in the subject line. Two aggregate queries over the roughly 34,000 stamped records, no row loop. The section re-checks field history tracking on each run and says in the digest text that it reports counts only, so the statement stays true if tracking is later turned on.

## 7. Testing

- `UnmappedTokenDigestTest` 42/42 in the sandbox and in the production validation; a real digest job ran clean in the sandbox.
- Smoke test in production: the September section rendered on demand read 73 records re-scored under v2 (60 Opportunity, 13 Lead). In total there were 1,907 engine writes to records that existed before September, 1,796 of them from the sweeper's backlog run on 2026-09-07 (blank version, meaning v1). 65 of the recompute stamps written on 2026-09-22 (the day of the DDT-1078 one-off re-score) were on records created before September and are in the count; the rest were created in September and are excluded by definition.
- The planned live check (the 2026-10-05 digest carrying the September section, the 2026-10-12 one not) is not recorded in the ticket.

## 8. Step by step

1. Define the population: written in the month, created before it.
2. Group by object and rule version with aggregate queries.
3. Check whether field history exists before promising "changed from / to" figures.
4. Add the section to an existing scheduled digest rather than a new job.
5. Render it on demand in production once as a smoke test.

## 9. Rollback

Redeploy the previous version of `UnmappedTokenDigest`. No data is written by this section.

## 10. Known limits

- Counts only: without field history on the attribution fields, the digest cannot say which channels gained or lost.
- Records created and re-scored in the same month are excluded by definition.

## 11. Reuse for similar issues

- When a number can move without a deploy, schedule a count of the movement and include the rule version.
- Reuse an existing scheduled job when the org is near its scheduled Apex limit.
- Have a report state its own blind spots in its text, and re-check them each run.
