# Adding a user to a Heymarket inbox

| | |
|---|---|
| **Ticket** | DDT-122, DDT-195, DDT-631 |
| **Project** | Support tickets |
| **Status** | Done (DDT-122 2026-06-09, DDT-195 2026-06-18, DDT-631 2026-08-04) |
| **Priority** | Medium (DDT-122, DDT-631); High (DDT-195) |
| **Reported by** | Firm staff, 2026-06-08 to 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | Heymarket (texting), Heymarket panel inside Salesforce |
| **Area** | User access: texting clients from Salesforce records |
| **Related** | DDT-624 (Salesforce PI matter visibility), DDT-626 (phone group move), both part of the same PI onboarding as DDT-631 |
| **Code** | Not applicable: the change is made in Heymarket's admin settings |

---

## 1. Summary

Users who could open Salesforce could not text from it, because their Heymarket user was not a member of any inbox. The Heymarket panel says so: "Please contact a Heymarket administrator to be added to an inbox." The fix is to add the user to the right inbox in Heymarket; in one case the user also had to log out and back in.

## 2. What was reported

- **DDT-122 (2026-06-08).** An intake user saw "Please contact a Heymarket administrator to be added to an inbox" in the Heymarket panel on a Contact record. Needed: the Intake inbox.
- **DDT-195 (2026-06-18).** A user's Heymarket account existed but was in no inbox, with the same message. Asked which inbox, the user answered "intake".
- **DDT-631 (2026-08-03).** A staff member moving onto the Personal Injury team needed access to PI texts: past PI conversations and new ones going forward, at the same level as other PI team members.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Heymarket inbox membership | Heymarket admin setting | User added to the Intake inbox (DDT-122, DDT-195) or the PI inbox (DDT-631) |

**Out of scope**

- Salesforce permissions. Nothing in Salesforce changed. The Heymarket package licence in Salesforce is site-wide (read from production on 2026-11-05: unlimited licences), so no Salesforce licence needs assigning.
- The rest of the PI onboarding in DDT-631 (Salesforce matter visibility and the phone group move), handled in DDT-624 and DDT-626.

## 4. How it works (background)

```
User opens a Contact / Lead in Salesforce
  -> the Heymarket panel loads for that user's Heymarket account
  -> Heymarket shows the conversations of the inboxes the user is a member of
       no inbox -> "Please contact a Heymarket administrator to be added to an inbox"
```

Having a Heymarket user is not enough. What a user can read and send is decided by **inbox membership**, which is managed in Heymarket, not in Salesforce.

## 5. Root cause

Each user's Heymarket account was not a member of the inbox their job needed. The panel's message states this directly. For DDT-631 the user had no PI inbox membership because she had just moved team.

## 6. Solution

An administrator added each user to the inbox for their team in Heymarket:

- **DDT-122:** Intake inbox.
- **DDT-195:** Intake inbox. The user was then told to **log out and back in**, after which it worked.
- **DDT-631:** PI inbox, matching other PI team members' access.

The tickets do not record the exact Heymarket screens used.

## 7. Testing

The user's own confirmation was the check:

- **DDT-195:** the user confirmed "All good" after logging out and back in.
- **DDT-122:** the support team asked the user to check again; the reply is not recorded in the ticket.
- **DDT-631:** Not recorded in the ticket.

**Not covered:** whether a newly added PI member could see conversation history from before they joined. DDT-631 asked for it; the result is not recorded.

## 8. Step by step

1. **Find out which inbox the user needs.** Ask, or match a colleague in the same team (Intake, PI, and so on).
2. **In Heymarket, as an administrator,** open the inbox and add the user as a member, with the same access level as the rest of that team.
3. **Ask the user to log out and back in**, then open the Heymarket panel on a Salesforce record.
4. **Confirm** the panel shows the inbox and the user can send a message.

## 9. Rollback

Remove the user from the inbox in Heymarket. It affects only that user.

## 10. Known limits

- **It recurs for every new user or team move** until inbox membership is part of onboarding.
- **A session may hold the old membership.** If the message persists after adding the user, have them log out and back in.

## 11. Reuse for similar issues

**How to recognise one:** a user sees "contact a Heymarket administrator to be added to an inbox", or can text but cannot see a team's conversations.

- **It is a Heymarket setting, not a Salesforce one.** Check inbox membership first.
- **Team moves need inbox changes too.** When someone changes team, add the new team's inbox alongside the Salesforce and phone changes.
- **Sign out and back in** after any access change before deciding it did not work.
