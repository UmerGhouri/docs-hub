# Client label missing on auto-created matter accounts

| | |
|---|---|
| **Ticket** | DDT-38 |
| **Project** | Support tickets |
| **Status** | Done 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex trigger) |
| **Area** | Matter accounts and their primary contact |
| **Related** | DDT-37, DDT-40 (closed the same day by the same resolver) |
| **Code** | None in the ddt-salesforce repo |

---

## 1. Summary

The ticket reported that accounts created automatically for matters were not tagged as client accounts, which made them hard to tell apart from other accounts in views, reports and automations.

The resolution recorded on the ticket describes a **different change**: setting the matter's chosen primary contact as the primary Opportunity Contact Role. Neither the field it relies on nor a "Client Account" record type exists in the DDT production org (section 5), so this page records what was asked, what was written down, and the gap between them.

## 2. What was reported

On 2026-05-26: when matter-related accounts were auto-created, the "Client Account" label was not applied. The ticket asked for:

- new auto-created matter accounts to carry the "Client Account" label or record type at creation,
- a one-time backfill of existing accounts missing it,
- an audit showing none are missing it going forward.

Number of affected accounts: not recorded in the ticket.

## 3. Scope

**In scope**, as recorded in the ticket comment:

| Component | Type | Change |
|---|---|---|
| `OpportunityTrigger` | Apex trigger | Entry point for the change |
| `OpportunityTriggerHandler` | Apex class | When `Is_PNC_Primary_Contact__c` is "No" on a matter, sets the contact in the Primary Contact field as the primary Opportunity Contact Role (`IsPrimary = true`) |
| Existing matters with `Is_PNC_Primary_Contact__c` = "No" | Data | Updated the same way, once |

**Out of scope:** not recorded in the ticket. Nothing recorded touches the account label or record type the ticket asked about.

## 4. How it works (background)

```
A matter (Opportunity) is created for a client
  -> an account is created for it automatically
  -> the account should be marked as a client account          (what the ticket asked)
  -> the matter's primary contact should be the primary contact role (what the comment describes)
```

## 5. Root cause

Not recorded in the ticket.

**The recorded fix does not match the ticket.** The title and description are about an account label; the resolution comment is about Opportunity Contact Roles. Checked on 2026-11-05:

| Check | Result |
|---|---|
| Account record types in DDT production | Business Account, Person Account, Insurance Carrier, Medical Providers, Vendor, plus two inactive. **No "Client Account" record type** |
| `FieldDefinition` for `%PNC%` on `Matter__c` | 0 rows |
| `FieldDefinition` for `%PNC%` on `Opportunity` | One unrelated field. **No `Is_PNC_Primary_Contact__c`** |
| `Is_PNC_Primary_Contact` anywhere in the ddt-salesforce repo history | Not found |
| `OpportunityTriggerHandler` | Exists in DDT production and in the repo; no version in the repo history references that field |

The ticket was closed in the same minute as DDT-37 and DDT-40, by the same resolver, and the DDT-40 comment links to a report in a different Salesforce org. The most likely explanation is that this work was done in that other org. That is an inference, not something the ticket states.

## 6. Solution

As recorded by the resolver:

- **Going forward:** when `Is_PNC_Primary_Contact__c` = "No" is selected on a matter, `OpportunityTriggerHandler` sets the contact in the Primary Contact field as the primary Opportunity Contact Role (`IsPrimary = true`).
- **Existing data:** matters already set to "No" were updated the same way.

What the ticket asked for (a client label on auto-created accounts, plus a backfill and an audit) is not recorded as done.

## 7. Testing

Not recorded in the ticket: no test class, no test results and no audit query.

**Not covered:** the ticket's own acceptance criteria (new accounts labelled, reports accurate, audit clean).

## 8. Step by step

To pick this up in DDT production:

1. **Decide what "Client Account" means here.** DDT production has no record type by that name; it may be a field value, a record type to create, or a request from another org.
2. **Find the code that auto-creates matter accounts** and set the label there at creation.
3. **Write the audit query** for auto-created matter accounts without the label, and run it to get the backfill count.
4. **Backfill** with `Database.update(rows, false)`, reporting each failure, and run the audit again afterwards.
5. **Record the fix on the ticket** with the query and its before and after counts.

## 9. Rollback

Not applicable: no change matching this ticket is present in DDT production or in the repo.

## 10. Known limits

- **The ticket's request is not shown as met** in DDT production (section 5).
- **The recorded change targets a field DDT production does not have.** If it was applied elsewhere, it has no effect here.
- **Same-day closures.** DDT-37, DDT-38 and DDT-40 closed within a minute of each other; see the DDT-37 page for the same pattern.

## 11. Reuse for similar issues

**How to recognise one:** a ticket closed quickly with a resolution that describes a different feature from its title.

- **Compare the resolution against the acceptance criteria**, line by line, before closing or documenting a ticket.
- **Confirm the named fields and classes exist in the target org** with `FieldDefinition` and `ApexClass` queries. A comment is not proof the change is live.
- **Label at creation, backfill once, audit after.** For any "missing label" ticket, all three are needed, and the audit query is what proves it.
