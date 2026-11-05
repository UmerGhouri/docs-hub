# One attribution answer per case: the credit hierarchy and channel dictionary


| | |
|---|---|
| **Ticket** | DDT-555 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-08-20 (reached production) |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / Flow / custom metadata / permission set) |
| **Area** | Marketing attribution on Opportunity (Court Case) and Lead |
| **Related** | DDT-509, DDT-556, DDT-557, DDT-558, DDT-561, DDT-574, DDT-811 |
| **Code** | PR #99 in the ddt-salesforce repo (git record of work deployed to production on 2026-07-29) |


---


## 1. Summary

The firm could not say reliably where a case came from: about 99% of records had no usable origin. This ticket built a derived credit engine that gives every Court Case (Opportunity) and Lead one answer. It checks, in order, whether the client is returning, whether someone referred them, and otherwise which marketing channel they came through. Matching uses a governed dictionary (`Channel_Mapping__mdt`) that can be edited in Setup without a deploy. Eight fields on Opportunity and six on Lead hold the result. It was deployed on 2026-07-29 and populated by the backfill in DDT-556.

## 2. What was reported

- 2026-07-28: build the resolver, the fields and the channel dictionary on Opportunity and Lead, in a fresh branch off `main` so the referral capture deploy (DDT-509) is not disturbed. Validate in the partial-copy sandbox, deploy to production with specified tests.

## 3. Scope

**In scope**

- Fields on both Opportunity and Lead: `Attributed_Origin__c`, `Attributed_Channel__c`, `Attributed_Source__c`, `Attribution_Basis__c`, `Attributed_As_Of__c`, `Attribution_Conflict__c`; on Opportunity also `Ever_Hired__c` and `Total_Paid_Historical__c`.
- Custom metadata `Channel_Mapping__mdt` (169 rows at deploy) and `Attribution_Config__mdt`.
- Apex: `AttributionCreditService` (the pure resolver), `AttributionCreditRunner` (queries and writes), `AttributionCreditBatch` (chunked backfill), `AttributionCreditServiceTest`.
- Two record-triggered recompute flows, one on Court Case and one on Lead, calling `AttributionCreditRunner.recompute` as an invocable method.
- Permission set `Attribution_Credit_Hierarchy` (read access to the 14 fields only).

**Out of scope**

- `Matter__c`: few rows, most with no source token, and nothing reads it.
- Running the backfill (DDT-556) and the flow suppression it needed (DDT-558).
- Stamping brand-new records: found during review and filed as DDT-574.

## 4. How it works (background)

Each record has up to three raw source tokens. The resolver reads the first one present: the Campaign name, else `Automated_Source__c`, else `How_did_you_hear_on_Creation__c`. The third is filled on about 91.7% of records and is a controlled picklist, so it joins the dictionary by exact match.

The hierarchy, first match wins:

1. **Returning**: a normalised phone or email matches an earlier ever-hired engagement created at least 30 days before.
2. **Referral**: five sub-tiers. The structured referral type from DDT-509, the client's first referral, the Lead's referring attorney, a self-reported referral answer, then a dictionary or pattern match. The tier is gated on the referral type, not the referrer's name, so a known but unnamed referrer still counts as a referral.
3. **Channel**: the dictionary bucket for the token.

`Attributed_Channel__c` is never blank: an unmatched record gets the literal value `Unknown`.

## 5. Root cause

Not applicable: new feature. Before it, origin had to be guessed from several partly filled fields with no precedence.

## 6. Solution

Built and validated in the partial-copy sandbox, then validated against production (check-only) and deployed on 2026-07-29. Key choices:

- **One resolver for every path.** The batch and the record-triggered recompute call the same entry points, so the backfill and live updates cannot drift.
- **No write guard and no "only when blank" filter.** Precedence alone decides. A guard would make a mis-clicked referral impossible to correct.
- **Never blank.** The external reporting dashboard filters out blank channels, so a blank would silently remove returning clients and referrals from channel reports.
- **"Channel" is not a referral type.** A review found that a referral type of `Channel` (meaning "not a referral") fell through to "Referral - Outside Attorney". A `TYPE_NOT_A_REFERRAL` constant now gates both referral sub-tiers. New tests failed against the old code with `Expected: Channel, Actual: Referral - Outside Attorney`.
- **Recompute runs off the save transaction.** It runs as a queued job so a bulk edit does not hit CPU limits, and it honours the `AttributionCreditService.bypass` switch.
- **Two buckets added**: `Broadcast / Outdoor` and `Online - Unspecified`, for billboard, radio, TV and generic internet answers that had nowhere to go. They were added to the dictionary's restricted bucket field and to `Attributed_Channel__c`.
- **Flows left Draft until after the backfill**, then activated (DDT-556).

## 7. Testing

- Partial-copy sandbox: 36 of 36 tests; coverage service 97%, runner 92%, batch 88%.
- First production check-only run failed with 9 errors that the sandbox did not show: 7 dictionary rows used the two new bucket values, which existed in the sandbox only as Setup changes; 2 came from a flow setting invalid at the API version. Both fixed. The re-run passed: 200 components, 36 of 36 tests, 0 errors.
- Performance in the sandbox: resolver alone 500 records in 70 ms CPU and 0 queries; with writes, 25 records took 1,811 ms CPU and 19 queries, and 200 records exceeded the CPU limit. All the cost came from the Opportunity's record-triggered flows. This made DDT-558 a hard blocker.
- After deploy: fields, classes, 169 dictionary rows and the permission set were present. The permission set had been assigned to nobody, so the fields were invisible even to administrators. It was assigned on 2026-07-29 to all 40 active full-licence users, with 0 failures, and both Opportunity and Lead field queries then ran cleanly.
- Not covered: "identical resolver in both paths" was proven by structure, not end to end. Test Opportunities could not be inserted without a record-type-aware data factory, so it was checked on the DDT-556 canary.

## 8. Step by step

1. Create the fields, the two custom metadata types, then their records (types before records).
2. Deploy the Apex and the permission set; deploy specific field files, not whole folders, to avoid dragging in drift.
3. Add every new bucket value to the field metadata, not only in Setup.
4. Run a production check-only deploy with the named tests.
5. Deploy, then assign the permission set right away and confirm the fields can be queried.
6. Leave the recompute flows Draft until the backfill is done; activate them through FlowDefinition.

## 9. Rollback

Deactivate the two recompute flows and stop the batch. The fields, metadata and classes are additive and can stay; stamped values can be cleared by a batch if needed. Removing the fields is destructive and was not planned.

## 10. Known limits

- The recompute flows only fire when `Referral_Source_Type__c` changes, so they never stamp a new record (fixed by DDT-574). They were later deactivated (DDT-948).
- The pattern fallback reaches only 6 of the channel buckets; unmapped tokens need the weekly digest (DDT-561).
- `Attributed_Channel__c` is not a restricted picklist, so a missing value in its definition fails silently (it just cannot be filtered on).
- Converted Leads are not stamped by decision; channel conversion rates must be read from the Opportunity.

## 11. Reuse for similar issues

- Keep the decision logic pure (no queries) and put queries and writes in a separate runner. The pure part is cheap to test and fast to run.
- Write a literal `Unknown`, never blank, when a downstream filter drops nulls.
- Before a mass update, measure the cost per record with writes on. Here the automation, not the logic, set the limit.
- A green sandbox does not prove production. Run a check-only deploy against production before the real one.
