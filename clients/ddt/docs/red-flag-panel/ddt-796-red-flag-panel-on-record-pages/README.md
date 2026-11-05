# Put the Red Flag Panel on the desktop and phone matter pages


| | |
|---|---|
| **Ticket** | DDT-796, DDT-797 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-08 (reached production) |
| **Priority** | High (DDT-796), Medium (DDT-797) |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record pages) |
| **Area** | Matter red flags: record pages |
| **Related** | DDT-794, DDT-795, DDT-800 |
| **Code** | PR #656 and PR #657 in the ddt-salesforce repo, deployed together in one request |


---


## 1. Summary

The Red Flag Panel replaced the old red flag display on the two desktop Opportunity record pages, and was added to the phone page for hired matters, which had shown no red flags at all. Both PRs were deployed in one request. A field that only lived inside the old Red Flags section now has its own section.

## 2. What was reported

- 2026-08-21 (DDT-796): swap the Red Flags field section for the panel on the two desktop record pages, including a second copy said to be in the page subheader.
- 2026-08-21 (DDT-797): the phone page for hired matters references none of the red flag fields, so staff on a phone see no flags. Add the panel near the top.

## 3. Scope

**In scope**

- `Opportunity_Record_BB_Default_Hired` (main hired-matter page).
- `Opportunity_Record_Page`.
- `Opportunity_Hired_BB_Mobile_Only` (phone page).

**Out of scope**

- The `Opportunity.Red_Flags` quick action stayed for the moment; it was deleted in DDT-800.
- Two other Opportunity pages that never referenced red flags.

## 4. How it works (background)

A Lightning page that references a component depends on it, so the component had to be in production first (DDT-794), and non-admins needed class access (DDT-795) or they would see an access error. A component without the Small form factor is invisible on phone even if the page includes it.

## 5. Root cause

Not applicable: page change for a new feature. Two findings against the ticket, from production's copies:

1. The subheader never held a red flag field in production. All three references were inside the Red Flags section.
2. The Red Flags section also held the historical "How did you hear about us" field, which appears nowhere else on that page and is filled on 2,557 hired matters.

## 6. Solution

- Main hired page: the panel sits in the old section's slot above the tabs; the three old fields were removed; the section was kept and renamed "How Heard (Historical)", shown whenever that field has a value (before, only when a red flag fired).
- `Opportunity_Record_Page`: the Red Flags box was replaced by the panel; its desktop-only visibility rule was kept, so the page behaves the same on phone.
- Phone page: the panel was added after the path and before the tabs, the same spot as desktop. This is new behaviour, not a port.
- No page visibility rule on the panel: it renders nothing when there are no flags.
- All pages were built from production's current copies, not the repo's, because the repo copies were older. Two accordion properties that production returns on retrieve but rejects on deploy were removed.

## 7. Testing

- Check-only validation against production for each PR with the controller test, 20 of 20. It proved the pages deploy; it rendered nothing.
- On screen, 2026-10-08: as admin on the main hired page, correct flags, colours and groups, nothing on a clean matter, New Court Appearance opens, and inline saves confirmed on a test matter that was then deleted. As a case management user: panel and Fix buttons present. As a read-only user on the other page: panel present, Fix buttons hidden.
- Phone view, on a flagged matter: all 7 flags with correct colours and groups, buttons under each flag, Add Case Number and New Court Appearance open. A long message wrapped the icon onto its own line; fixed in PR #663 under DDT-794.

## 8. Step by step

1. Deploy the component and its access first.
2. Retrieve each page from production and keep the copy for rollback.
3. Replace the old field section with the component in the same slot; check the section for other fields before removing it.
4. Add the component to the phone page and confirm the component declares the Small form factor.
5. Validate, deploy the pages together, then open a flagged and a clean matter on desktop and phone, as admin and as a non-admin.

## 9. Rollback

Redeploy the production copies retrieved before the deploy, or edit the pages in App Builder.

## 10. Known limits

- The phone page applies to 8 profiles; other profiles get the main desktop page on their phone, where the panel also shows.
- New staff need both permission sets to see and fix flags.

## 11. Reuse for similar issues

- Build page changes from production's copy, not the repo's.
- Before removing a field section, check whether it holds anything shown nowhere else.
- A component that renders nothing when empty does not need a page visibility rule.
- For phone, check the component's form factors first; otherwise the page change silently does nothing.
