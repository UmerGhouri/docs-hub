# Liens report type and Financials tab access

| | |
|---|---|
| **Ticket** | DDT-280 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report types, Apex class access, Lightning record page) |
| **Area** | PI matters: tracking lien negotiations |
| **Related** | None recorded |
| **Code** | Not applicable: configured in Setup, no PR |

---

## 1. Summary

A staff member tracking lien negotiations on PI matters had two blockers. The only report type she could find, "Activities with Liens", returned no records and did not offer the lien fields she needed. And opening the Financials tab on a Matter showed a "You do not have access to Apex class..." popup. The team granted her access to the Apex class and created a new **Matters with Liens** custom report type.

## 2. What was reported

- **Reporting:** the "Activities with Liens" report type returned zero records and did not expose the Client Matter, Lien Holder, Treatment Provider or Negotiated Amount fields.
- **Access:** on a sample Matter, the Financials tab showed "You do not have access to Apex class..." so she could not use it.

Why it matters: the lien negotiation work happens on the Financials tab, and the firm needed a report to follow it.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Matters with Liens | Custom report type (category: Other Reports) | New. Matter with its `Lien__c` records, exposing client / Matter, lien holder, treatment provider and negotiated amount |
| The Apex class behind the Financials tab | Apex class access | Granted to the requester. Which class, and through which profile or permission set, is not recorded in the ticket |

**Out of scope**

- **Building her report.** The team offered to build it once she sent the columns, grouping, filters and totals she wanted.
- **The hire-status filter.** On 2026-07-23 she asked to be able to filter the report by hire status, to leave out inactive cases. What was done about that is not recorded in the ticket.

## 4. How it works (background)

```
Matter__c  (PI matter)
  -> Lien__c  (one per lien; master-detail to Matter__c)
       Lienholder__c, Treatment_Provider__c,
       Original_Lien_Amount__c, Final_Negotiated_Amount__c,
       Negotiation_Status__c, ...
```

- **A report can only use the objects and fields its report type includes.** "Activities with Liens" is built on activities, so it returns rows only where liens have activities, and it doesn't carry the lien's own fields.
- **A Lightning component that calls Apex** needs the user to have access to that Apex class, through their profile or a permission set. Without it the component shows the "You do not have access to Apex class" error.

## 5. Root cause

- **Reporting:** no report type joined Matters to their Liens. The one that existed was activity-based, which is why it returned nothing useful.
- **Access:** the requester's profile and permission sets did not include the Apex class the Financials tab calls. Which class it was is not recorded in the ticket.

## 6. Solution

### 6.1 Report type

Created **Matters with Liens** in Setup → Report Types, filed under *Other Reports*. It makes these available to a report:

| Needed in the report | Field on `Lien__c` |
|---|---|
| Client / Matter | `Matter__c` and the Matter's fields |
| Lien Holder | `Lienholder__c` |
| Treatment Provider | `Treatment_Provider__c` |
| Negotiated Amount | `Final_Negotiated_Amount__c` |

The ticket names the four fields by label only. The API names above are the closest-matching fields on `Lien__c` in the repo, so check them against the report type itself before relying on them.

### 6.2 Apex class access

Access to the Financials tab's Apex class was granted, and the requester was told to log out and back in (or hard-refresh) and reopen the tab.

## 7. Testing

Not recorded in the ticket beyond the resolution note. The ticket's acceptance criteria were: the Financials tab opens with no popup, a report type exposes the four fields, and the requester builds and saves her report.

**Not covered:** whether other users on the same profile hit the same popup. The ticket asked for this check; its result is not recorded.

## 8. Step by step

**For the "no access to Apex class" popup:**

1. **Read the class name in the popup.** It names the class the component tried to call.
2. **Find what grants it.** Setup → Apex Classes → the class → Security shows which profiles have it. Permission sets list it under Apex Class Access.
3. **Grant it through an existing permission set** that the user's role already uses, rather than to one user's profile, so everyone in the same role gets it.
4. **Ask the user to log out and back in**, then reopen the tab.
5. **Check other users with the same profile**, since they will hit the same popup.

**For the report type:**

1. Setup → Report Types → New Custom Report Type.
2. Primary object: Matters. Category: Other Reports. Label: Matters with Liens.
3. Add Liens as the related object.
4. Under Edit Layout, add the lien fields the requester needs (holder, treatment provider, negotiated amount).
5. Set the deployment status to Deployed, so users can see it.
6. Build a test report and confirm the fields appear and rows return.

## 9. Rollback

- **Apex class access:** remove the class from the permission set or profile. The popup comes back.
- **Report type:** it can only be deleted once no reports use it.

## 10. Known limits

- **Neither change is in git.** The report type was created in Setup and does not appear in the ddt-salesforce repo, so a check of the repo will not show it exists.
- **The class name was not recorded**, so if the popup returns, start again from the class named in the popup.

## 11. Reuse for similar issues

**How to recognise one:** "You do not have access to Apex class" on a tab or panel, or "my report returns nothing" from someone using an activity-based report type.

- **The popup names the class.** Grant that class, through a permission set, and check everyone else on the same profile.
- **"Report returns zero rows" is often the wrong report type**, not missing data. Check what objects the report type joins.
- **Record the class and the permission set in the ticket**, so the next fix doesn't start from scratch.
