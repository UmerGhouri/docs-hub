# Discovery Puller access in the CMT inbox

| | |
|---|---|
| **Ticket** | DDT-930 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-04 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission set, email inbox LWC) |
| **Area** | Pulling discovery from a court portal link in the CMT email inbox |
| **Related** | None recorded |
| **Code** | None. Permission set assignment in Setup |

---

## 1. Summary

A CMT staff member could not use the Discovery Puller in the CMT email inbox. Access to that feature comes from one permission set, `Discovery_Puller`, which they did not have. The support team assigned it, and after a hard refresh or a fresh login the **Pull Discovery** button appears on CMT inbox emails for them.

## 2. What was reported

- **2026-09-03:** a team lead asked, on behalf of one CMT staff member, for access to the Discovery Puller in the CMT inbox. Without it the staff member was blocked from part of their workflow.

One user affected.

## 3. Scope

**In scope:** assigning the `Discovery_Puller` permission set (label "Discovery Puller") to one user in production.

**Out of scope:** any change to the permission set itself, the inbox component or the discovery feature. The feature was working; only the user's access was missing.

## 4. How it works (background)

```
Email in the CMT inbox
  -> emailThreadView (LWC) renders the thread
  -> it embeds pullDiscovery (LWC)
  -> pullDiscovery asks DiscoveryPullController (Apex) whether the email holds a portal link
  -> if so, the "Pull Discovery" button is shown
```

The `Discovery_Puller` permission set grants access to the `DiscoveryPullController` Apex class and to the fields on `Discovery_Pull__c` that the feature reads and writes. Its own description in the repo says to assign it only to users who should see the **Pull Discovery** button. A user without it does not get a working button.

## 5. Root cause

The user did not hold the `Discovery_Puller` permission set. Access to this feature is granted per user through that permission set, not through the profile, so a new user of the feature has to be given it explicitly.

## 6. Solution

The support team assigned the `Discovery_Puller` permission set to the user in production on 2026-09-04, and told them that after a hard refresh or re-login, **Pull Discovery** would appear on any CMT inbox email.

Why a permission set and not a profile change: this feature is deliberately limited to the users who need it, and a profile change would give it to everyone on that profile.

## 7. Testing

- **Checked:** the assignment was made and the user was told to refresh.
- **Not covered:** the ticket's second acceptance criterion, the user confirming they could use it. Not recorded in the ticket.

A check you can run to confirm an assignment:

```sql
SELECT Assignee.Name, PermissionSet.Name
FROM PermissionSetAssignment
WHERE PermissionSet.Name = 'Discovery_Puller'
```

## 8. Step by step

1. Confirm which user needs access, by username, in production.
2. Check whether they already have it, with the query in section 7 filtered to that user.
3. In Setup, open **Users**, find the user, and under **Permission Set Assignments** click **Edit Assignments**.
4. Add **Discovery Puller** and save.
5. Ask the user to hard refresh the page, or log out and back in.
6. Ask them to open an email in the CMT inbox that holds a portal link and confirm **Pull Discovery** shows.

## 9. Rollback

Remove the permission set assignment from the user in Setup. Safe: it removes the button for that user and changes nothing else.

## 10. Known limits

- **The button only appears when the email contains a portal link** that the feature recognises. An email without one shows no button, even for a user with the permission set. That is easy to mistake for a missing permission.
- The page must be refreshed after the assignment. A page already open keeps the old access.

## 11. Reuse for similar issues

**How to recognise one:** a feature works for some users and is missing for one, and the feature's components live in a dedicated permission set.

- **Look for a feature permission set first.** Search the repo's `permissionsets` folder for the feature name; its description often says who should have it.
- **Assign, do not widen.** Add the one user to the existing permission set instead of editing a profile.
- **Tell the user to refresh**, and say what they should see and where, so they can confirm it in one look.
