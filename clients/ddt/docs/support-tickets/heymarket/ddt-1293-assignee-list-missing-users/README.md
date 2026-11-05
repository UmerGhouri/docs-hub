# Heymarket chat assignee list missing users

| | |
|---|---|
| **Ticket** | DDT-1293 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-09 |
| **Resolved by** | Support team |
| **Systems** | Heymarket (Salesforce integration, user mapping) |
| **Area** | Client text messaging: assigning chats to teammates |
| **Related** | None recorded |
| **Code** | Not applicable: configuration fix in the Heymarket user mapping |

---

## 1. Summary

A staff member who monitors client text messages could only assign Heymarket chats to herself and one other entry. The rest of the team was missing from the assignee picker, and she could not tag anyone in comments either. Her Heymarket settings had not changed. Redoing her **Heymarket user mapping** in Salesforce restored the full list.

## 2. What was reported

- **2026-10-09:** the staff member reported that the assignee picker on a chat offered only herself and one other entry, so she could not route chats to the right teammates. She asked how to check whether she was an admin.
- **2026-10-09:** an administrator confirmed she had exactly the same settings as before. She added that she also could not tag anyone in comments.
- **2026-10-09:** the PI attorney asked for it to be fixed because she and one colleague are primarily responsible for monitoring.
- **2026-10-12:** after the fix she confirmed "its all set now".

One user was reported as affected. Whether anyone else saw the same is not recorded in the ticket.

## 3. Scope

**In scope**

- The affected user's Heymarket user mapping in Salesforce.

**Out of scope**

- **Her Heymarket role and team membership.** Confirmed unchanged, so not edited.
- **Other users' mappings.** Not reported as affected.

## 4. How it works (background)

```
Heymarket, used from inside Salesforce
  -> each Salesforce user has a Heymarket user mapping
  -> the user assigns chats and tags teammates in comments from Salesforce
```

How Heymarket builds the assignee list from the mapping is not recorded in the ticket. What the ticket shows is that the user's own mapping affected both the assignee list and comment tagging.

## 5. Root cause

Her Heymarket role and settings were unchanged, which ruled out a permission change. Redoing her user mapping fixed both the assignee list and comment tagging, so the mapping for her user was the problem.

What exactly was wrong with the mapping, and how it got that way, is **not recorded in the ticket**.

## 6. Solution

Her Heymarket user mapping was redone in Salesforce. After that the assignee picker showed the full team in Salesforce.

## 7. Testing

- The fix was checked in Salesforce by the Support team: the full assignee list showed.
- **Live check:** the user confirmed on 2026-10-12 that it was all set.

**Not covered:** whether comment tagging was checked separately is not recorded.

## 8. Step by step

1. **Confirm the scope.** Ask whether the user also cannot tag people in comments, and whether anyone else sees a short list.
2. **Check the user's Heymarket role and team.** If they are unchanged, do not change them.
3. **Open the Heymarket user mapping** for that user in Salesforce.
4. **Redo the mapping** for that user.
5. **Open a chat as that user** (or have them do it) and check the assignee picker shows the full team.
6. **Ask the user to confirm,** including comment tagging.

## 9. Rollback

Not applicable: the mapping was restored to a working state, not changed to something new.

## 10. Known limits

- The cause of the broken mapping is unknown, so it may recur for this or another user.

## 11. Reuse for similar issues

**How to recognise one:** one user's Heymarket picker or tagging shows far fewer people than it should, while their role and settings look unchanged.

- **Rule out settings first,** then go straight to that user's Heymarket user mapping in Salesforce.
- **Here comment tagging and the assignee list failed together,** and one mapping fix cured both. Either symptom is worth checking against the mapping.
- **Record what the mapping looked like before you redo it,** so the cause can be found if it happens again.
