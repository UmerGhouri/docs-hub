# Lead Source not carried to Court Case on conversion

| | |
|---|---|
| **Ticket** | DDT-134 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Converting a Lead into a Court Case (Opportunity) |
| **Related** | None recorded |
| **Code** | No PR. Commit `fd2b2169a` in the ddt-salesforce repo; the change reached `main` in the 2026-07-03 production backfill |

---

## 1. Summary

When a Lead was converted into a Court Case, the Court Case's **Lead Source** was blank even though the Lead had a value. The custom conversion code never read `Lead.LeadSource` and never set it on the new Opportunity. The fix adds it to the query and copies it across after conversion.

## 2. What was reported

- **2026-06-09:** a staff member reported that Court Cases created by converting a Lead showed Lead Source as blank ("--None--"), with a screenshot of the Court Case layout. Lead source drives marketing and referral reporting, so losing it meant the firm could not trace which channels produced cases.
- **2026-06-10:** the support team asked for an example Lead and its Court Case, and the fix was deployed to production the same day.

How many Court Cases were created with a blank Lead Source is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `NewPotentialClientController` | Apex class | `convertLead` reads `Lead.LeadSource` and sets `Opportunity.LeadSource` after conversion (Court Case path) |

The same commit also copied `Automated_Source__c` from the Lead onto the new client Account, and wrote the wizard's "how did you hear" choice onto the Lead before conversion.

**Out of scope**

- **Lead field mapping in Setup.** The ticket listed it as a possible cause. The fix was made in code, not in the mapping.
- **Existing Court Cases** already created with a blank Lead Source. No backfill is recorded.
- **The Matter (PI) conversion path** of the same method. The change targets the Court Case path.

## 4. How it works (background)

Conversion from the Lead page runs through the `newPotentialClient` LWC, which calls `NewPotentialClientController.convertLead`:

```
User picks "Court Case" in the conversion wizard
  -> convertLead re-queries the Lead
  -> creates the client Account
  -> Database.convertLead(...) creates the Opportunity
  -> a follow-up update sets fields on the Opportunity
       (Hire_Status__c, How_did_you_hear_on_Creation__c, Case_Description__c, ...)
```

The follow-up update is where the code decides which Lead values reach the Court Case. Anything not set there and not mapped in Setup is left blank.

## 5. Root cause

`convertLead` never queried `Lead.LeadSource` and never assigned it in the follow-up Opportunity update. The wizard's "how did you hear" answer went to `How_did_you_hear_on_Creation__c` only, so `Opportunity.LeadSource` stayed blank.

**The first diagnosis was different.** On 2026-06-10 the support team's first reply said the "How did you hear about us?" field on the layout was a custom field set by a dropdown in the conversion wizard, not standard Lead Source, and that standard Lead Source did carry over. The commit made the same day says the opposite: that `Opportunity.LeadSource` is the field shown as "How did you hear about us?" on the layout, and that the code never set it. The code change is the second reading. When a layout label is involved, check which field sits behind it before deciding which fix applies.

## 6. Solution

Two lines in the Court Case path of `convertLead`:

```apex
Lead updatedLead = [
    SELECT FirstName, LastName, Email, Description, Automated_Source__c,
           How_did_you_hear_about_us__c, LeadSource
    FROM Lead
    WHERE Id = :leadId
    LIMIT 1
];
// ...
Opportunity opp = new Opportunity(Id = lcr.getOpportunityId());
opp.Hire_Status__c = wrapper.get('hireStatus');
opp.How_did_you_hear_on_Creation__c = wrapper.get('howHear');
opp.LeadSource = updatedLead.LeadSource;
opp.Case_Description__c = updatedLead.Description;
update opp;
```

Why it's done this way:

- **Set after `Database.convertLead`, in the same follow-up update** that already sets the other Court Case fields, so all the copied values are in one place.
- **Read from the Lead record, not from the wizard**, because Lead Source is already on the Lead and the user is not asked for it during conversion.

The current version of the class on `main` still sets `opp.LeadSource` from the Lead, now read from the Lead queried at the top of the method.

**How it reached git.** The commit sits on a feature branch, and no pull request references DDT-134. The fix was deployed to production on 2026-06-10, and the line arrived on `main` only with the 2026-07-03 commit that backfilled production's state into git.

## 7. Testing

- **Apex test:** none was added. `NewPotentialClientControllerTest` does not assert on `LeadSource`.
- **Live check:** not recorded in the ticket beyond the deploy on 2026-06-10.
- **Not covered:** nothing would fail if the line were removed.

## 8. Step by step

1. **Find which field the user means.** Open the layout and confirm the API name behind the label (here, standard `LeadSource` versus the custom how-heard fields).
2. **Convert a test Lead with that field filled** and check the new Court Case.
3. **Find the conversion code.** For the Lead page wizard it is `NewPotentialClientController.convertLead`.
4. **Add the field to the Lead query** and assign it in the post-conversion Opportunity update.
5. **Add a test** that converts a Lead with a Lead Source and asserts the Opportunity has it. (This was not done here.)
6. **Validate against production** with the class's test, get the change approved, deploy, then repeat step 2 in production.

## 9. Rollback

Redeploy the previous version of `NewPotentialClientController`. That brings the blank Lead Source back. No data is changed by the fix, so nothing else needs undoing.

## 10. Known limits

- **Only conversions after the deploy are fixed.** Court Cases converted before 2026-06-10 keep a blank Lead Source unless backfilled.
- **Only this conversion path is fixed.** Any other code that converts Leads into Court Cases must copy the field itself.
- **No test protects it**, so a later edit to `convertLead` could drop the line without anything failing.

## 11. Reuse for similar issues

**How to recognise one:** a value is on the Lead and blank on the record the Lead was converted into.

- **Check whether conversion is custom.** With custom Apex, the Setup field mapping is only part of the story; the code's follow-up update decides the rest.
- **Confirm the field behind the label** before choosing a fix. Here the first answer and the fix disagreed on which field "How did you hear about us?" was.
- **Add the test** with the fix. A missing field copy is invisible until someone runs a report.
- **Ship it through a PR**, so the fix is in git when it is in production.
