# Cannot log in after email and name update

| | |
|---|---|
| **Ticket** | DDT-135 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-12 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (User record, login) |
| **Area** | Staff logins after a name or email change |
| **Related** | None recorded |
| **Code** | Not applicable: User record change in Setup, no code |

---

## 1. Summary

A staff member's Salesforce user was updated with a new name and a new email address. They then tried to log in with the new address and got an "account can't be found" error, because Salesforce logs in by **Username**, which is a separate field from Email. The support team updated the user so the new address could be used, and Salesforce sent a verification email that the user had to confirm before logging in.

## 2. What was reported

- **2026-06-09:** a staff member's user record had recently been changed to a new name and a new email address. Logging in with the new address returned "account can't be found," and they were locked out of Salesforce.
- **2026-06-10:** the support team updated the user and Salesforce sent a verification email. The requester thanked the team the same day.

One user was affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The affected staff member's User record | Salesforce User | Updated so the new address could be used to log in |

**Out of scope**

- The user's profile, permission sets and licences. Nothing about access changed, only the login identity.

## 4. How it works (background)

A Salesforce User has two fields that look alike:

| Field | What it is used for |
|---|---|
| **Username** | What the person types to log in. Unique across all Salesforce orgs. |
| **Email** | Where Salesforce sends notifications and verification emails. |

They often hold the same value, but changing one does not change the other. When an admin changes a user's **Email**, Salesforce sends a verification email to the new address, and the change completes once the user clicks the link.

## 5. Root cause

The user record's name and email were updated, but the person tried to log in with the new address as if it were their username. Login is tied to Username, not Email, so Salesforce found no account by that name.

The ticket's own analysis said the same: if the Username still held the old address, only the old address would work; if it had been changed, the new one would not work until the verification link was clicked. Which of the two was the case here is not recorded in the ticket.

## 6. Solution

The support team updated the user's email on 2026-06-10. Salesforce sent a verification email to the user, and the user had to confirm it before logging in again. Whether the Username was changed in the same edit is not recorded in the ticket.

## 7. Testing

- **Live check:** the requester acknowledged the fix on 2026-06-10. A confirmed successful login by the user is not recorded in the ticket.
- **Not covered:** nothing else to test; this is a single-record change.

## 8. Step by step

1. **Setup → Users → open the user.** Read **Username** and **Email** separately.
2. **If Username still holds the old address,** the quickest unblock is to have the person log in with the old address.
3. **To move them to the new address,** edit the user, set **Username** (and **Email**, if not already done) to the new address, and save.
4. **Tell the person a verification email is coming** to the new address and that they must click it.
5. **If it doesn't arrive,** resend it from the user record.
6. **Confirm with the person that they can log in** before closing the ticket.

## 9. Rollback

Set the Username and Email back to their previous values on the User record. Safe, though the person would then log in with the old address again.

## 10. Known limits

- **Usernames are unique across every Salesforce org**, not just this one. If the new address is already a username somewhere else, Salesforce will not accept it, and a different username is needed.
- **Renaming a user does not change their login.** The same confusion will happen for the next name change unless the Username is updated at the same time.

## 11. Reuse for similar issues

**How to recognise one:** "account can't be found" or "invalid username" right after a user's name or email was changed.

- **Check Username first, not Email.** They are different fields.
- **Change Username and Email together** whenever someone's address changes, and tell them about the verification email in the same message.
- **Give the person the old username as a stopgap** while the change is pending.
