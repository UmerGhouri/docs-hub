# Call transfer link fields were unreadable for call-logging users


| | |
|---|---|
| **Ticket** | DDT-868 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-01 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission sets, custom fields), RingCentral call data |
| **Area** | Call logging: transfer linkage and reporting |
| **Related** | DDT-817, DDT-818, DDT-819 |
| **Code** | PR #379, PR #581 and PR #587 in the ddt-salesforce repo |


---


## 1. Summary

The call logging integration writes six fields on `Call_Session__c` that link a transferred call leg to its parent and peer sessions. Almost nobody could read them, so transfer legs could not be reported on. Read access was added to the permission set that call-logging users already hold, one field that existed in production but not in git was added to the repo, and a small extra permission set held by two administrators was folded in and retired. Call-logging users can now query and report on the transfer links.

## 2. What was reported

- 2026-08-29: six live linkage fields on call sessions are invisible to everyone except a single holder of a narrow permission set. A query for one of them returns "No such column" even for an administrator, so a missing permission looks exactly like a field that was never built. One of the six also exists in production and in no branch of the repo.

## 3. Scope

**In scope**

- Read access on `Parent_Session__c`, `Parent_Telephony_Session_Id__c`, `Is_Transfer_Continuation__c`, `Conversation_Master__c`, `Peer_Session__c` and `Peer_Telephony_Session_Id__c` through the call-logging permission set, in production and in the sandbox.
- `Parent_Telephony_Session_Id__c` added to `force-app/main/default/objects/Call_Session__c/fields/`.
- Four further call-related fields that only the narrow permission set granted, folded into the call-logging permission set; the narrow set then deleted.

**Out of scope**

- Page layouts. The four folded fields are not on any record page, so they show in reports and queries only.
- Inactive assignments on the call-logging permission set, noted as a separate small clean-up.

## 4. How it works (background)

The call logging code builds one `Call_Session__c` per call and links blind-transfer legs to the original call through parent, peer and conversation-master fields. Field-level access in this org is granted through permission sets. A field with no read grant is not just hidden on the page: queries treat it as if it does not exist.

## 5. Root cause

A permission that was never granted. Read access on the six fields lived only in one narrow permission set (held by two people) and a vendor package set. Evidence: the ticket's query returned `No such column 'Parent_Session__c'` while the field was present in `FieldDefinition`; 43 custom fields existed on the object and an administrator describe returned 37. Separately, `Parent_Telephony_Session_Id__c` was absent from the repo, so a full redeploy of the object from git would not have recreated it.

## 6. Solution

1. Add read grants for the six fields to the call-logging permission set (PR #379). Additive only, so no existing grant could be removed.
2. Add `Parent_Telephony_Session_Id__c` to git so the repo matches production.
3. Deploy the same permission set to the sandbox, where the six grants were also missing (56 grants there against 62 in production).
4. Firm leadership decided to fold the narrow permission set's four extra grants into the call-logging set (PR #581) and retire the narrow set (PR #587), so the firm does not carry a two-person permission set. A backup of the set and its assignments was kept.

## 7. Testing

- Done-when check, run as a non-administrator who holds the call-logging set but never held the narrow set: the 90-day parent-session count returned 4,459 rows instead of an error, and all six fields read without error. A user without the call-logging set was refused.
- After the fold, the same non-administrator could read all four folded fields.
- Sandbox: all six fields read back as readable after the deploy.
- Not covered: no automated test; these are permission checks run against the org.

## 8. Step by step

1. Compare the object's fields in `FieldDefinition` with what a describe returns for a normal user; the missing ones are the unreadable ones.
2. Find which permission sets grant read on them.
3. Add read grants to the permission set the intended users already hold. Keep the change additive.
4. Compare the production and sandbox copies of that permission set before deploying, so no grant is silently removed.
5. Check the result as a non-administrator who does not hold any other set granting the fields.
6. Add any production-only field to git.

## 9. Rollback

Remove the added grants from the call-logging permission set and redeploy. The retired permission set can be restored from the kept backup and reassigned.

## 10. Known limits

- The two peer fields held no data in the 90 days measured, so the reporting value is in the other four.
- The fix was first marked done against production only; the sandbox gap was found and closed a day later.

## 11. Reuse for similar issues

- Signs: a query says "No such column" for a field you can see in Setup. Check field access before concluding the field is missing.
- Test access as a user who holds only the intended permission set. An administrator, or someone holding an extra set, can pass a check that real users fail.
- A permission fix is not done until the sandbox has it too, because the sandbox is where the next person builds.
- Diff production against git for fields the code writes; a production-only field breaks the next full deploy.
