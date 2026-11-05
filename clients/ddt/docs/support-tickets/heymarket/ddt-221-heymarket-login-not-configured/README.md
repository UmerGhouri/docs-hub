# Heymarket says login is not configured

| | |
|---|---|
| **Ticket** | DDT-221, DDT-474, DDT-516, DDT-734, DDT-1028 |
| **Project** | Support tickets |
| **Status** | Done (DDT-221 2026-06-24, DDT-474 2026-07-21, DDT-516 2026-07-29, DDT-734 2026-08-14, DDT-1028 2026-09-17) |
| **Priority** | High (DDT-221, DDT-474, DDT-734, DDT-1028); Medium (DDT-516) |
| **Reported by** | Attorney, Case Manager, Intake team and Firm staff, 2026-06-23 to 2026-09-16 |
| **Resolved by** | Support team |
| **Systems** | Heymarket (texting), Heymarket panel inside Salesforce; Conga Composer (DDT-516 only) |
| **Area** | User access: texting clients and leads from Salesforce records |
| **Related** | DDT-122, DDT-195, DDT-631 (a different Heymarket message: user not in any inbox) |
| **Code** | Not applicable: the fix is a Heymarket user mapping record, created through Heymarket's sync or by hand |

---

## 1. Summary

Users opening the Heymarket panel on a Salesforce record saw **"Your login is not configured to use Heymarket, please contact Admin!"** and could not text. The message does not mean a missing licence or permission set. It means there is no **Heymarket Team Mapping** record linking that person's Salesforce user to their Heymarket user. The fix in each case was to create that link.

## 2. What was reported

- **DDT-221 (2026-06-23).** A newly set-up attorney could not log in to Heymarket: "login is not configured to use Heymarket, please contact admin."
- **DDT-474 (2026-07-21).** A case manager saw the same message in the Heymarket component in Salesforce. A colleague reported it on his behalf.
- **DDT-516 (2026-07-27).** A staff member reported access problems with both Heymarket and Conga Composer, with no error captured at first. The Heymarket part turned out to be the same message.
- **DDT-734 (2026-08-13).** A staff member saw the same message in the Heymarket section inside Salesforce.
- **DDT-1028 (2026-09-16).** An intake user saw the same message on Lead records, which blocked texting new leads.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `heymarket__Team_Mapping__c` | Heymarket managed-package object | One record per affected user, linking their Salesforce user to their Heymarket user |
| Conga Composer package licence (DDT-516 only) | Managed package licence | One seat freed from an inactive user and held by the affected user |

**Out of scope**

- **Heymarket permission sets.** In DDT-516 the user already had the Heymarket permission set and was an active Heymarket user. Neither was the problem.
- **Heymarket inbox membership.** A user with a mapping but no inbox sees a different message ("Please contact a Heymarket administrator to be added to an inbox"). That is covered by DDT-122, DDT-195 and DDT-631.
- **The audit DDT-516 asked for** (other users in the same role missing access) and its proposed onboarding runbook. Neither is recorded as done.

## 4. How it works (background)

```
User opens a Lead / Contact / other record in Salesforce
  -> the Heymarket panel looks for a heymarket__Team_Mapping__c
     whose Salesforce User is the person viewing
       none found  -> "Your login is not configured to use Heymarket, please contact Admin!"
       found       -> panel loads as that Heymarket user
                      -> shows the inboxes that Heymarket user belongs to
                           no inbox -> "Please contact a Heymarket administrator to be added to an inbox"
```

`heymarket__Team_Mapping__c` fields, read from production on 2026-11-05:

| Field | Type | Holds |
|---|---|---|
| `heymarket__Team__c` | Master-Detail (Team) | The Heymarket team the mapping belongs to |
| `heymarket__Salesforce_User__c` | Lookup (User) | The Salesforce user |
| `heymarket__Heymarket_User_Id__c` | Number | The matching Heymarket user's id |
| `heymarket__Heymarket_User_Name__c` | Text | The Heymarket user's name |
| `heymarket__Default_Inbox_Id__c` | Number | The user's default inbox |
| `heymarket__Default_Inbox_Number__c` | Text | That inbox's number |

## 5. Root cause

### Heymarket (all five tickets)

The affected user had **no Team Mapping record** linking their Salesforce user to their Heymarket user. DDT-516 recorded the finding: "This error checks only for that mapping; the permission set and Heymarket account are not what it validates."

**The ticket's first guess was wrong, and that is the useful lesson.** DDT-516 was written as a "licence vs permission set" problem and laid out how to diagnose each. The user had both. The panel's error is about the mapping, so checking licences and permission sets finds nothing.

How the mapping goes missing is not recorded in the tickets. DDT-516's fix note says the auto-sync can skip a user, and to check that the user's **email in Heymarket matches their email in Salesforce** when it does. At least two of the users (DDT-221, DDT-474) had been set up as new Salesforce users shortly before, which fits a mapping that was never created.

### Conga Composer (DDT-516 only)

A separate problem with the same user. She already had the Conga **Composer User** permission set, and Conga templates are shared Public Read/Write internally, so neither sharing nor the permission set was blocking her. The Conga package licence pool was **fully allocated, 22 of 22 seats used**, and several seats were held by inactive users.

## 6. Solution

### Heymarket: create the mapping

Create the Team Mapping for the user, either:

- **Heymarket → Sync Users**, the option named in DDT-516, or
- **a manual `heymarket__Team_Mapping__c` record**, with the Salesforce user, the Heymarket user's id and name, and the team.

If sync skips the user, check that their email is the same in both systems first.

Recorded per ticket:

- **DDT-516:** mapping created as above.
- **DDT-734:** "I linked her Heymarket and Salesforce users." The user was told to refresh, and then had access.
- **DDT-1028:** "It's done ... it's connected."
- **DDT-221, DDT-474:** marked resolved; the exact action is not recorded in the ticket.

Why a mapping and not a permission set: the panel decides who the viewer is in Heymarket from the mapping alone. A permission set grants access to the package's objects and pages, but it cannot tell the panel which Heymarket user to act as.

### Conga Composer (DDT-516)

An inactive user's Conga licence was removed to free a seat, and the affected user was confirmed to hold a Conga package licence and read access to the template object.

## 7. Testing

The check in each case was the user opening the panel again:

| Ticket | Result |
|---|---|
| DDT-221 | Support asked the user to check; the reply is not recorded in the ticket. |
| DDT-474 | Fixed about eight minutes after the ticket opened; the user confirmed through a colleague, and a screenshot of the working panel was posted. |
| DDT-516 | Mapping created, Conga seat and template access confirmed. The user's own confirmation is not recorded. |
| DDT-734 | After a refresh: "I have access now." |
| DDT-1028 | Not recorded in the ticket beyond "it's connected". |

**This query finds active users who hold a Heymarket permission set but have no mapping.** It was run against production on 2026-11-05 and returned **4**. Those users were not looked at; some may not text at all.

```sql
SELECT COUNT_DISTINCT(AssigneeId)
FROM PermissionSetAssignment
WHERE PermissionSet.NamespacePrefix = 'heymarket'
  AND Assignee.IsActive = true
  AND AssigneeId NOT IN (SELECT heymarket__Salesforce_User__c FROM heymarket__Team_Mapping__c)
```

**Not covered:** a test text sent and received after the fix, which every ticket's acceptance criteria asked for. None records one.

## 8. Step by step

1. **Confirm the exact message.** "Your login is not configured to use Heymarket" is this page. "Please contact a Heymarket administrator to be added to an inbox" is the inbox problem in DDT-122.
2. **Check whether the user has a mapping.** Query `heymarket__Team_Mapping__c` filtered on `heymarket__Salesforce_User__c` for the user. No row means this is the cause.
3. **Check the user exists in Heymarket**, and that their Heymarket email matches their Salesforce email.
4. **Run Heymarket → Sync Users.** Re-run the query in step 2. If a row now exists, go to step 6.
5. **If sync skipped them, create the mapping by hand**: a `heymarket__Team_Mapping__c` record with the Team, the Salesforce user, and the Heymarket user's id and name. Copy the team and default inbox from a working colleague's mapping.
6. **Ask the user to refresh the record page** and open the Heymarket panel.
7. **If the panel now says they are not in an inbox**, add them to their team's inbox in Heymarket (see DDT-122).
8. **Ask for a test text** to and from a staff phone, and record it on the ticket.

## 9. Rollback

Delete the mapping record to remove the link. There is no reason to: without it the user cannot text from Salesforce. For the Conga change in DDT-516, the freed seat belonged to an inactive user, so nothing is lost by leaving it.

## 10. Known limits

- **Sync can skip users.** An email that differs between Heymarket and Salesforce is the recorded reason to check; other reasons are not recorded.
- **New starters can hit this.** At least two of the five users were newly set up. DDT-516 called for an onboarding runbook that covers package access; no runbook is recorded.
- **Conga seats are a fixed pool.** It was full at 22 of 22 in DDT-516. Freeing a seat from an inactive user works only while inactive users still hold seats.
- **Active users without a mapping** were found by the query in section 7 on 2026-11-05 and not investigated.

## 11. Reuse for similar issues

**How to recognise one:** a managed-package panel loads but says the user is "not configured", "not activated" or "not linked", while their permission set looks right.

- **Read the error literally.** A "not configured" message usually points at a user link record, not a permission.
- **Find the package's mapping object** and look for the user's row before touching licences or permission sets.
- **Compare with a working colleague's records**: mapping, licence, permission sets, inbox.
- **Licences are a separate question.** When a package pool is full, look for seats held by inactive users first (DDT-516).
- **Add the mapping step to new-user setup**, so it stops arriving as a ticket.
