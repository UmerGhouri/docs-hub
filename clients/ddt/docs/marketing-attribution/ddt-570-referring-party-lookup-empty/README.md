# "Referring Party" search on the Lead returns no results


| | |
|---|---|
| **Ticket** | DDT-570 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-08-04 (analysis; nothing deployed) |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Intake: Referral Source panel on the Lead |
| **Related** | DDT-509, DDT-810, DDT-811, DDT-815, DDT-1022 |
| **Code** | No code change |


---


## 1. Summary

An intake rep chose Referral Type "Business Partner" in the Referral Source panel on a Lead, typed part of a partner's name, and got no results. The search was working. It only lists business accounts of the chosen type, and the partner the rep meant had never been created as a Business Partner account. The only match on that name was a client, which the panel hides on purpose. The ticket was closed with that answer. A later review found that the panel was barely used and that most saved referrals had a type but no party.

## 2. What was reported

- 2026-07-28: on the Lead's Referral Source section, the Referring Party search returns nothing (for example a partial partner name with Referral Type = Business Partner), so intake cannot record who referred the lead. A metadata check had found no "Referring Party" or "Referral Type" field on Lead.

## 3. Scope

**In scope**

- The `inboundContactReferrerPicker` panel and its `referralPicker` search, built in DDT-509.

**Out of scope**

- Creating the missing partner account: that is a data-entry step for intake or operations.
- Changing the picker so a referrer can be created during intake: not decided in this ticket.

## 4. How it works (background)

The panel is a custom component, not a field on Lead, which is why a field search on Lead found nothing. Behind it, the referral is stored on a hidden `Inbound_Contact__c` record. The search is scoped by the chosen Referral Type: a Business Partner search only lists business accounts of that type, and clients are excluded.

## 5. Root cause

No defect. A search for a known Business Partner (a partial name of an existing partner) returned it. The name the rep searched for did not exist as a Business Partner; the only record with that name was a client, excluded by design.

## 6. Solution

No change. The rep was told the partner must be created as a Business Partner account before it can be picked.

## 7. Testing

- One lookup against one existing Business Partner returned the expected account.
- Not covered: a full capture-and-save on a real Lead ending with a referring party set. On 2026-08-24, 27 days after go-live, the referral object held 10 rows in total; 8 had a referral type but no party; the structured referral type was set on 0 Opportunities and 2 Leads; all 1,413 referral credits in the org came from free-text matching. Three causes fit equally: the picker, the need for an existing account, or staff not being told to use it.

## 8. Step by step

1. Confirm what the "field" really is: here a custom component on the page, not a field on the object.
2. Find what the search queries and how it filters by type.
3. Search for a known existing record of that type to prove the search works.
4. Look up the name the user typed across all record types to see why it is excluded.
5. Before closing, check whether real users complete the whole capture, not just the search.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- A referrer who is not already set up as the right kind of account cannot be picked during a call.
- A referral saved with a type but no party derived nothing onto the parent records until DDT-811 widened that rule.
- Flow-based conversion paths did not capture the referring party at the time (later DDT-1022 and DDT-1083).

## 11. Reuse for similar issues

- "Lookup returns nothing" is often a scope filter working as designed. Test with a record you know should match.
- A check on one record cannot show whether staff use a feature. Measure completed saves before closing a capture-path ticket.
- If users choose a type and stop, the design (not the search) may be the friction.
