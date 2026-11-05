# How-heard list: missing vendor options, and every option labelled by type


| | |
|---|---|
| **Ticket** | DDT-1180 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-02 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (global value set, record types) |
| **Area** | Intake: "How did you hear about us" |
| **Related** | DDT-557, DDT-902, DDT-961, DDT-1117, DDT-1178 |
| **Code** | PR #601 in the ddt-salesforce repo |


---


## 1. Summary

The "How did you hear about us" list offered 21 options and none of them was Legal Growth Engine, the firm's largest lead-vendor line. A hand-made lead from that vendor was saved as "Google / Online Search" and credited Online - Unspecified. The value existed but was inactive, and Nolo DUI had the same gap. Both were activated and added to every record type that shows the question, and the 23 options were relabelled and reordered so each label says its type ("Vendor: ...", "Referral: ..."). Only labels changed; every stored value stayed the same.

## 2. What was reported

- 2026-10-01: creating a PI lead by hand for a Legal Growth Engine lead, firm leadership found no correct option and picked Google. The lead converted and was credited as online search. Asked: activate Legal Growth Engine and Nolo DUI on every record type that shows how-heard; keep the vendor's sub-products and other retired options off; group the whole list by type in one list (no second "which vendor" picklist), changing labels only; confirm both values resolve in the channel dictionary; fix the affected lead and matter.

## 3. Scope

**In scope**

- Global value set `Client_Advertising_Source`: 2 activations, 10 relabels (5 vendors as "Vendor: ...", 5 referral answers as "Referral: ..."), order of the 23 active values.
- Record types: Lead Defense, PI and SOS; Court Case (Opportunity) Hired and Potential; Matter; Person Account. Both values added to how-heard on each.
- Data: how-heard set on one lead and its matter.

**Out of scope**

- The vendor's sub-products, FindLaw and Abogado options stay inactive (decision in DDT-1117 and DDT-961: one label for the whole vendor package).
- No change to `Channel_Mapping__mdt`: both values already mapped to Lead Vendors / Directories.

## 4. How it works (background)

How-heard is a restricted picklist backed by one global value set, shared by Lead, Matter, Court Case and Account, so a label change shows everywhere. A restricted picklist rejects a value through the API unless it is active and assigned to the record's record type. Vendor leads normally arrive through a feed with `Automated_Source__c` set, so the hidden value only mattered when someone created a vendor lead by hand.

## 5. Root cause

`Legal Growth Engine` and `Nolo DUI` had been inactive in `Client_Advertising_Source` since before the 2026-04-01 snapshot, probably because their leads usually came through the feed. With no correct option, staff picked the closest wrong one.

## 6. Solution

1. **Activate both values** and add them to how-heard on all seven record types. Why: until a value is active and on the record type, it cannot be saved.
2. **Relabel by type, keep stored values.** Groups in order: found us (search, LSA, social, video, reviews, billboard, walk-in, hired in court), vendors, referrals, already knows us (returning, existing, former lead, other, unknown). Why: Salesforce picklists cannot show headings, so the label and order carry the grouping; keeping API values means no report, rule or code breaks.
3. **Built on production's copy.** A value-set deploy writes the whole set and a record-type deploy rewrites its picklist assignments, so the files were compared with a full production retrieve first (they matched, compact layouts included).
4. **Checked nothing matches on the label.** Apex, flows, LWCs and formulas compare stored values only; screens read labels for display.
5. **Clean-up.** The affected lead's how-heard was set and it re-stamped through the credit engine to Lead Vendors / Directories, rather than editing the derived field by hand. The matter's sales-comp source moved to Paid Ads on the next nightly run.

## 7. Testing

- Read back after deploy: 23 active values in order, 12 changes against production (2 activations, 10 labels); all seven record types showed 23 options with the sub-products hidden, and no compact-layout resets.
- Screen check in production (2026-10-02): the new Lead form (PI, Defense, SOS), the PI hire wizard, the Defense and SOS conversions, and Court Case and Matter edit all showed the new labels in order. Old records showed the new labels with stored values unchanged.
- 2026-10-12: the matter's comp source read Paid Ads, set by the attribution job.
- No Apex tests were involved; this was a configuration change.

## 8. Step by step

1. Retrieve the value set and every affected record type from production; diff against git.
2. Search code, flows, LWCs and formulas for comparisons against labels.
3. Activate the values, relabel, and set the order; turn off alphabetical sorting if it is on.
4. Add the values to each record type.
5. Deploy, then check Setup Audit Trail for compact-layout resets.
6. Check the real screens, not only Setup.
7. Correct affected records through the normal process, not by typing derived fields.

## 9. Rollback

Redeploy the value set and record types from before PR #601. Records saved with the two values keep them; once the values are inactive again, those records cannot be re-saved with them through the API.

## 10. Known limits

- Labels show everywhere the value set is used, including reports; this is intended.
- A record type added later must be given both values, or the question will again lack them there.

## 11. Reuse for similar issues

- Signs of the same problem: a lead source credited to the wrong channel because staff picked the nearest option. Check for an inactive picklist value.
- Group long picklists with label prefixes and order, and leave API values alone.
- Always build value-set and record-type deploys from production's current copy.
