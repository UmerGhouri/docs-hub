# Repository deploy pull request review

| | |
|---|---|
| **Ticket** | DDT-88 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom object and fields, validation rules, flow, Lightning pages, permission set) |
| **Area** | PI demands: tracking a demand from preparation to outcome |
| **Related** | DDT-86 (Expense Manager, whose PR was based on this one) |
| **Code** | PR #20 in the `ddt-salesforce` repo, with the deploy doc `docs/deploys/demand-tracker.md` |

---

## 1. Summary

The ticket was a review request for PR #20, the **Demand Tracker**: a component that tracks each PI demand through preparation, sending, offers and counter-offers, client authorization and outcome, with every demand tied to a `Coverage__c` record. The review found one blocker: an **active screen flow** in production still created demands without a Coverage, and the PR's new `Coverage_Required` validation rule would have made that flow fail. The PR was changed to replace the flow on the Matter page with the new component and deactivate the flow. It shipped on 2026-06-02.

## 2. What was reported

- **2026-06-02, DDT-88:** review PR #20 before promotion. The ticket listed 9 commits, 43 of 43 tests passing, and a deploy doc with its design decisions.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `demandTracker` | LWC | New. Works on the Matter page (all demands, New Demand) and the Demand page (one demand) |
| `DemandController` | Apex class | New. Reads demands, saves them, moves phases, records offers and outcomes |
| `DemandControllerTest` | Apex test | New, 43 tests |
| `Offer__c` | Custom object | New. One record per offer or counter-offer, Master-Detail to `Demand__c` |
| `Offer__c` validation rules | Validation rules | Counter-offers need client authorization; round 0 must come from the carrier; amount must be positive |
| `OfferTrigger`, `OfferTriggerHandler` | Apex trigger and class | Roll the latest offer and activity date up to the Demand |
| 24 new fields on `Demand__c` | Fields | Coverage link and line, phase, approval and send audit, prep checklist, outcome, activity dates |
| `Coverage_Required` | Validation rule on `Demand__c` | A demand must have a Coverage |
| `Matter_Record_Page` | Lightning page | The old demand screen flow replaced by `demandTracker` |
| `Demand_Record_Page` | Lightning page | New, hosts `demandTracker` |
| `Demands_Screen_Flow` | Flow | **Deactivated** |
| `Demand_Tracker` | Permission set | Access to the new object, fields and controller |

**Out of scope** (from the deploy doc)

- **Deleting the old carrier, adjuster, claim number, defendant and demand type fields on `Demand__c`.** Kept, made read-only through the permission set, and left for a later PR once existing demands are migrated.
- **Other demand flows** (`Demands_Flow`, `Demands_Datatable`, `Create_Demands_Screen`, `Email_Demand_Letters`). Unchanged; none of them create demands without a Coverage.
- **Generating the demand letter.** The tracker stores the final PDF only.

## 4. How it works (background)

```
New Demand (Matter page)
  -> pick a Coverage, then a Coverage line (liability or UM/UIM)
  -> carrier / adjuster / claim / limit are read live from the Coverage, never retyped
Phase: Preparation -> Drafting -> ... -> Sent -> Negotiations -> outcome
  Drafting needs all 6 prep boxes; Sent needs an approver; Negotiations needs at least one offer
Offers: unlimited rounds; a DDT counter-offer needs client authorization
Outcome: Accepted / Rejected / Moved to Litigation / Withdrawn
```

Before this PR, the Matter page created demands through `Demands_Screen_Flow`, which built them from the older insurance policy data, with no Coverage.

## 5. Root cause

**What the review found.** `Demands_Screen_Flow` was still active in production and created demands with no Coverage. Once `Coverage_Required` was switched on, every demand that flow created would have been rejected.

**The review also tested the data migration** against existing demands. Most matched a Coverage. One was ambiguous and looked like UM/UIM, and four looked like old test or incomplete data that would not match automatically.

The lesson: a new validation rule applies to **every** path that writes the object, not just the new component. List the existing writers before switching one on.

## 6. Solution

The review offered three options: deactivate the flow, update it, or ship the rule inactive. The PR took the first, with a replacement.

- **`Matter_Record_Page`:** the flow's component was replaced by `demandTracker` in Matter mode, which becomes the way to create a demand.
- **`Demands_Screen_Flow`:** deactivated.
- **`Coverage_Required`:** left active. The new component always sets a Coverage, so nothing legitimate is blocked.

Why the rule could stay active: once the flow was gone, the only way to create a demand from the Matter page was the new component, and it always sets a Coverage. So the rule had nothing left to block.

**Migration decisions** recorded in the deploy doc:

- The 4 test or incomplete demands: delete.
- The 1 ambiguous demand: treat as UM/UIM and link it to the matter's UM/UIM Coverage line, creating the Coverage first if missing.
- Run the migration straight after deploy, so existing demands have a Coverage before anyone edits them under the active rule.

## 7. Testing

From the PR and deploy doc:

| Test | Result |
|---|---|
| `DemandControllerTest` | 43 of 43 in the sandbox |
| Bulk offer insert (30 offers in one transaction) | Covered by a test in `DemandControllerTest` |

The deploy doc's smoke test for production: create a demand from a real Coverage; tick the prep boxes and move through to Sent; record a carrier offer and several counter-offers; confirm a counter without authorization is blocked and a final offer cannot be deleted; close as Accepted. It also lists three demand reports to check for breakage.

Production: the ticket records only that it was already deployed. The smoke test results and the migration run are not recorded in the ticket.

## 8. Step by step

1. Before adding a validation rule, list every flow, component and integration that creates or edits the object. Check each one sets the new required value.
2. Validate against production with the specified test:

   ```
   sf project deploy validate --target-org <prod-alias> \
     --source-dir <files> --test-level RunSpecifiedTests --tests DemandControllerTest
   ```

3. Deploy with the same command, using `start` instead of `validate`.
4. Assign `Demand_Tracker` to the PI team (the deploy doc names attorneys, case managers and paralegals).
5. Confirm `Demands_Screen_Flow` is inactive and the Matter page shows `demandTracker`.
6. Run the demand migration, applying the decisions in section 6.
7. Run the smoke test in section 7.

## 9. Rollback

From the deploy doc:

1. **Soft:** remove `Demand_Tracker` from users. The new fields become invisible and the component errors gracefully. The standard Demand layout still works on the old fields.
2. **Reports:** the old offer fields stay populated, so existing reports keep working.
3. **Full:** revert the PR. All new metadata is additive. To restore the old create path, reactivate `Demands_Screen_Flow` and put its component back on the Matter page, but only with `Coverage_Required` inactive, or every demand it creates will fail.

## 10. Known limits

- **Old demand fields still exist** until the follow-up PR deletes them.
- **Coverage data is read live**, by design. Editing a Coverage changes every demand on that line.
- **Offers marked final cannot be deleted** from the component.

## 11. Reuse for similar issues

**How to recognise one:** a new required field or validation rule on an object that other automation already writes.

- Before switching on a rule, find every writer of the object: active flows, other components, integrations, Apex.
- For each writer, either make it set the value, replace it, or ship the rule inactive and switch it on later.
- Test a data migration against real records before deploy and decide each unmatched record up front.
