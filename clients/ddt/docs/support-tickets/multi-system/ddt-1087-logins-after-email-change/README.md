# Logins break after a staff email address change

| | |
|---|---|
| **Ticket** | DDT-1087, DDT-1232, DDT-1253, DDT-1254, DDT-1263 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-08 |
| **Priority** | High (DDT-1232 was Urgent) |
| **Reported by** | Firm staff, 2026-09-22 to 2026-10-07 |
| **Resolved by** | Support team |
| **Systems** | Google Workspace sign-in, Heymarket, RingCentral, Shadow Agent, Trainual |
| **Area** | Staff accounts: moving to a new firm email domain |
| **Related** | None recorded |
| **Code** | Not applicable: account changes made in each vendor's admin screen |

---

## 1. Summary

Staff were moved to email addresses on a new firm domain. Afterwards, signing in with Google to several third-party apps failed with "account not found", because each app looks up the user by the exact email address Google sends, and Google now sent the new one. Each app's account had to be moved to the new address, and in Heymarket the user's inbox access had to be granted again.

## 2. What was reported

- **2026-09-22, DDT-1087:** a staff member could not sign in to Heymarket. Google sign-in with the new address returned "Account not found — No account exists for the SSO account selected". The same person reported the same problem with Shadow Agent.
- **2026-10-05, DDT-1232 (Urgent):** a second staff member was locked out of Heymarket. Google sign-in said no account existed; signing in with the old address sent a verification code to the old mailbox, which did not arrive.
- **2026-10-06, DDT-1253:** a third staff member could not sign in to RingCentral or Shadow Agent with the new address: "We didn't find any matched account."
- **2026-10-06, DDT-1254:** the DDT-1232 staff member, now signed in to a new Heymarket account, had no inboxes and asked whether the old account's history could be moved over.
- **2026-10-07, DDT-1263:** the DDT-1253 staff member could not sign in to Trainual with the new address.

Why it matters: each person was blocked from texting clients (Heymarket), taking calls (RingCentral), or training (Trainual) until it was fixed.

## 3. Scope

**In scope**

| System | Change |
|---|---|
| Heymarket | New user created on the new address; linked to the old account and the old one removed; inbox membership granted |
| RingCentral | User's login moved to the new address |
| Shadow Agent | User's login moved to the new address |
| Trainual | User's login moved to the new address, then done for everyone on the PI team |

**Out of scope**

- The email change itself in Google Workspace. That was a deliberate firm decision; these tickets dealt with its effect on other apps.

## 4. How it works (background)

```
User clicks "Sign in with Google" in a third-party app
  -> Google authenticates the user
  -> Google tells the app: "this user's email is <primary address>"
  -> the app looks for an account with exactly that address
  -> found: signed in       not found: "account not found"
```

A firm administrator explained the key point in the DDT-1087 thread: even if the old address is kept as an alias in Google Workspace, third-party apps receive only the **primary** address during Google sign-in.

## 5. Root cause

When a user's primary Google address changes, every app that matches accounts by email still holds the old address. Google sends the new one, so the app finds no account. One team member noted in the thread that this had been warned about beforehand: these apps would still need a name-and-password login, or an account change, after the switch.

For DDT-1232 the email-code fallback also failed, because the code went to the old mailbox and did not arrive there in time to be used.

DDT-1254 was a knock-on effect: a new Heymarket user starts with no inbox membership, so the new account worked but could see nothing.

## 6. Solution

### 6.1 Heymarket (DDT-1087, DDT-1232, DDT-1254)

1. A firm administrator sent a Heymarket invitation to the user's **new** address. The first invitation in DDT-1087 did not arrive and was re-sent.
2. The user created the new Heymarket account from the invitation.
3. The administrator linked the new account to the old one and removed the old one.
4. The user could sign in but was not a member of any inbox. Inbox membership was then granted. For DDT-1254 the user confirmed their privileges and work had transferred to the new address.
5. In DDT-1254 the user then reported that Heymarket worked in the app and browser but not inside Salesforce. A support team member reported it fixed the same evening. What was changed is not recorded in the ticket.

### 6.2 RingCentral and Shadow Agent (DDT-1253)

The RingCentral desktop app started working first, while the RingCentral phone app and Shadow Agent still failed. Firm leadership then passed it to an administrator, who made a further change, and the user confirmed it now worked. The exact steps in each admin console are not recorded in the ticket.

### 6.3 Trainual (DDT-1263)

An administrator updated the user's Trainual login, and the user confirmed it worked. Firm leadership then asked for the same update for everyone on the PI team, and the administrator confirmed it had already been done.

## 7. Testing

Each ticket closed on the user's own confirmation in the support Slack thread that they could sign in. For DDT-1254, a support team member also signed in through the user's profile and confirmed inboxes were visible and messages could be sent. No other testing is recorded.

## 8. Step by step

For each staff member whose primary email changes:

1. List every app they sign in to with Google: at least Heymarket, RingCentral (desktop **and** phone app), Shadow Agent and Trainual.
2. **Heymarket:** invite the new address, have the user accept, link the new account to the old one, remove the old one, then add the user to the same inboxes they had before.
3. **RingCentral and Shadow Agent:** change the user's login email to the new address in each admin console, then have the user sign in on every device.
4. **Trainual:** change the user's email to the new address.
5. **Heymarket inside Salesforce:** have the user confirm the Heymarket panel in Salesforce works, not only the app and browser.
6. Ask the user to confirm each app before closing.

Better: do steps 2 to 4 **before** the primary address changes, so nobody is locked out.

## 9. Rollback

Not applicable: reverting would mean changing each app's account back to the old address, which no longer matches the user's Google identity.

## 10. Known limits

- Each app is fixed separately and by hand. Any other app that uses Google sign-in will fail the same way for a renamed user.
- Heymarket inbox membership does not follow the user to a new account automatically.
- What fixed Heymarket inside Salesforce in DDT-1254 is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** "account not found" or "no matched account" right after someone's email address changed, with the right password or a working Google sign-in.

- **Third-party apps match on the exact primary email.** An alias in Google does not help.
- **Plan the change per app, before the switch.** Here the same problem arrived as five tickets over two weeks.
- **After a new account is created, check its access**, not only its login: inboxes, queues, licences, and any embedded view in Salesforce.
- **Do it for the whole team in one pass** when a domain change is rolled out, as was done for Trainual.
