# Re-score a record when its Campaign or how-heard answer arrives late


| | |
|---|---|
| **Ticket** | DDT-1078 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-22 (reached production) |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow / Apex) |
| **Area** | Marketing attribution: keeping a record's credit current |
| **Related** | DDT-574, DDT-822, DDT-826, DDT-948, DDT-561, DDT-1079, DDT-995 |
| **Code** | PR #434 in the ddt-salesforce repo |


---


## 1. Summary

A new Court Case or Lead is credited to a channel within minutes, often before intake has finished filling it in. When the Campaign or the how-heard answer arrived after that first stamp, nothing re-scored the record, so it stayed Unknown for good. The two existing recompute flows were widened to also fire when those fields get a value. 76 records already stuck on Unknown were re-scored once by hand.

## 2. What was reported

- 2026-09-22: found while reading the first weekly unmapped-token digest. Two records from the digest window were stamped Unknown and would now resolve to a real channel (one to LSA, one to Online - Unspecified) because their Campaign or how-heard answer arrived after the stamp.

## 3. Scope

**In scope**

- Flows `Attribution_Recompute_On_Referral_Change_Court_Case` and `Attribution_Recompute_On_Referral_Change_Lead`, and their flow definitions.
- A one-off recompute of records already stale.

**Out of scope**

- Re-scoring the whole stamped history: DDT-826.
- The digest's own defect: DDT-561.

## 4. How it works (background)

Records are stamped fast: by the hourly sweeper (DDT-574) or by the touch recompute within seconds of a call (DDT-822). Intake then completes the record: the Campaign lands from the campaign processor or the Court Case after-save flow, or a staff member picks the how-heard answer. The sweeper only picks up records with `Attributed_As_Of__c` empty, so a stamped record is outside its reach by design.

## 5. Root cause

The two recompute flows entered only on `ISCHANGED(Referral_Source_Type__c)`. A Campaign or how-heard change on a stamped record triggered nothing. A design comment on DDT-574 (2026-08-01) had asked for the Campaign to be in the entry criteria, but it never made the acceptance criteria.

Sizing (read-only, production, 2026-09-22): every stamped record with `Attribution_Basis__c = 'None'` was run through `AttributionTokenSelector.choose` and `AttributionCreditService.lookup`. 63 Court Cases created since 2024 would now resolve to a real channel (8 of them since the sweeper went live on 2026-09-07), plus 1 unconverted Lead. Referral-tier answers were skipped, so this undercounts.

## 6. Solution

1. Widened the entry criteria, keeping the existing referral-type leg:
   - Court Case: `ISCHANGED(CampaignId)` or `ISCHANGED(How_did_you_hear_on_Creation__c)`, non-blank after the change.
   - Lead: `ISCHANGED(Campaign__c)` or `ISCHANGED(How_did_you_hear_about_us__c)`, non-blank after.
   Why it cannot loop: `AttributionCreditRunner` writes only the attribution fields, the rule version and the two prior-engagement fields, none of which are in the entry criteria. Why non-blank: clearing a field should not re-score.
2. Deployed in two steps: the flows first (they land as Draft), then the flow definitions to activate the new version. Sending both in one package reported success and created no new version.
3. One-off fix of history: the stale set was re-captured (63 Court Cases and 13 unconverted Leads) and `AttributionCreditRunner.recompute` was run over them in two chunks.

## 7. Testing

- Sandbox, on real records: attaching a Campaign, setting how-heard on a Lead, and setting how-heard on a Court Case each re-scored within seconds with a v2 stamp; clearing how-heard did not re-run. Three recompute jobs for three changes, none for the clear.
- Production: both flows read back active on the new version through the Tooling API, not from the deploy result.
- The 76 re-scored records: 76 of 76 re-stamped with v2, 0 still Unknown, basis Dictionary Token on every one. Court Cases moved to Online - Unspecified 31, Lead Vendors / Directories 19, Paid Social 8, LSA 2, Direct / Phone 2, Google Business Profile 1; Leads to Lead Vendors 12 and Online - Unspecified 1.
- One-week check (2026-09-30): of records created since go-live, 2 Court Cases and 77 Leads were Unknown and none would resolve from current fields, so the sizing query stayed at zero. The nightly campaign processor run fired the re-score 1 to 2 times a night with no failures; 656 re-score jobs ran with 0 errors.

## 8. Step by step

1. Find stamped records whose current fields would now resolve to a real channel (read-only).
2. List every field the resolver reads as a token, and check each one is in some recompute trigger.
3. Add the missing fields to the flow entry criteria, non-blank after the change.
4. Check the writer cannot re-trigger the flow.
5. Deploy flows, then flow definitions, and read the active version back.
6. Re-score the existing stale set once, then re-run the sizing query a week later.

## 9. Rollback

Set each flow definition back to the previous active version. The one-off recompute is not reversed; records would simply stop re-scoring on these fields.

## 10. Known limits

- Bulk updates (the nightly campaign processor can set Campaign on many Leads) queue one recompute per batch of up to 200 records; volume was watched and was low.
- Records whose fields change to blank keep their last answer.

## 11. Reuse for similar issues

- Signs of the same problem: a record shows Unknown attribution but has a Campaign or how-heard answer that maps to a real channel. Compare its stamp time with when those fields were filled.
- Any field the resolver reads must be in a recompute trigger, or late data never counts.
- Flow and FlowDefinition in the same deploy package can silently create no new version; deploy them separately and read back.
