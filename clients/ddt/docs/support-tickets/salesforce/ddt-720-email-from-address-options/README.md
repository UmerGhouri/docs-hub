# Email From address options

| | |
|---|---|
| **Ticket** | DDT-720, DDT-1259, DDT-1273, DDT-1276 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-12 |
| **Priority** | High (DDT-1259, DDT-1276), Medium (DDT-720, DDT-1273) |
| **Reported by** | Firm staff, 2026-08-12, 2026-10-06 and 2026-10-08 (two tickets) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Setup: Organization-Wide Addresses, queues, user email settings; Apex reading those settings) |
| **Area** | Sending email from Salesforce: the From dropdown in the email composer, and replies from the shared inbox |
| **Related** | DDT-712 (shared inbox composer) |
| **Code** | Not applicable: all four were fixed in Setup or in the user's own settings. No PR. |

---

## 1. Summary

Four tickets about which addresses Salesforce lets staff send from. They are not one bug. One was a profile missing from an org-wide address's access list; one was a user's email still connected to her old Google account after the PI team moved to a new email domain; one was a new org-wide address waiting on verification; and one was the shared inbox falling back to the Collections address because the PI Team queue's email no longer matched any org-wide address.

All four were fixed in Setup or in the user's settings. The common thread: the From dropdown only shows **verified** org-wide addresses the user's **profile** is allowed to use, plus the user's own **verified, connected** address.

## 2. What was reported

### DDT-720 (2026-08-12)
A PI team staff member composing an email from a client record saw only her personal address in the From dropdown. She needed the injury team's shared address. She could read the injury inbox but not send from it.

### DDT-1259 (2026-10-06)
After the PI team moved to a new email domain, the same staff member's From dropdown still showed her old personal address, and the injury team option still showed the old-domain address. After the first fix she reported that only her login had changed, not the email options.

### DDT-1273 (2026-10-08)
Another staff member could not find the new shared team address (on the new domain) in the From dropdown, from a Matter or anywhere else. A third staff member reported the same the same day: she needed that address for every process and saw only the main office address. The person with access to the team mailbox clicked the Salesforce verification link but was sent to a login page instead.

### DDT-1276 (2026-10-08)
Replies sent from the shared inbox on the PI queue went out from the **Collections** address, although the sender's own email settings showed her new address. Replies risked going to the wrong team. A second staff member reported the same.

## 3. Scope

**In scope (all Setup or user settings)**

| Item | Where | Change |
|---|---|---|
| `DDT Injury Team` org-wide address | Setup → Email → Organization-Wide Addresses | DDT-720: `DDT SOS` profile added to its allowed profiles. DDT-1259: address switched to the new-domain team address, then re-verified (DDT-1273) |
| User email (one staff member) | User record, and the user's Settings → Connected Accounts | DDT-1259: new address set and verified; Google account reconnected under the new address |
| Email signature (one staff member) | User email settings | DDT-1276 follow-up: updated to the new address at her request |
| `PI Team` queue | Setup → Queues | DDT-1276: queue email set to the new team address, which is now an org-wide address |

**Out of scope**

- **No code change.** `EmailReplyService` was not changed (see section 10 for its fallback).
- **Moving the staff member off the `DDT SOS` profile.** Raised on DDT-720 as the better long-term fix, since she was no longer on the SOS team. Not recorded as done.
- **An audit of every PI team user's new address.** DDT-1259 asked for it; other users were handled as they reported. Not recorded as a full audit.

## 4. How it works (background)

**The composer's From dropdown** lists:

1. The user's own email address, once the user has verified it (and, where a Google account is connected for sending, the address of that account).
2. Every **verified** org-wide address whose access is either "all profiles" or includes the user's profile.

**Shared inbox replies** do not use the dropdown. `EmailReplyService.pickOwaForInbox` picks the From address in code:

```
inbox (queue DeveloperName)
  -> queue's Email field
  -> org-wide address with that exact Address
  -> if none matches: the first org-wide address returned by an unordered query
```

Org-wide addresses as read from production on 2026-11-05 (display names only):

| Display name | All profiles |
|---|---|
| DDT Defense Team | yes |
| Collections | no |
| DDT Injury Team | no (profile list) |
| DDT Intake Team | yes |
| DDT SOS Team | yes |

## 5. Root cause

### DDT-720: profile not on the address's access list
`DDT Injury Team` is restricted to a list of profiles. The staff member's profile, `DDT SOS`, was not on it, so the address never appeared for her.

### DDT-1259: old Google connection and unverified new addresses
Two causes. First, the new personal address and the switched team address had been entered in Salesforce but not yet verified, and an unverified address does not appear. Second, after she verified, her Salesforce email was still connected to her **old** Google account under Connected Accounts, so that was the only personal address Salesforce would send from.

### DDT-1273: the new team address was not verified
The `DDT Injury Team` org-wide address now pointed at the new-domain team address, but nobody had completed its verification, so it was hidden from every dropdown. The first verification link sent the clicker to a login page rather than confirming.

### DDT-1276: queue email matched no org-wide address
The `PI Team` queue's email was still the old-domain injury address. Once the `DDT Injury Team` org-wide address was switched to the new domain (DDT-1259), **no org-wide address matched the queue's email**. `pickOwaForInbox` then fell back to the first org-wide address its query returned, which was Collections.

**The lesson.** DDT-1276 was a side effect of the DDT-1259 change. Changing an org-wide address breaks every queue whose email pointed at the old value, and the code hides that by silently picking another address instead of failing.

Evidence, read from production on 2026-11-05: the `DDT Injury Team` org-wide address was created on 2025-06-11 and now holds the new-domain team address, and the `PI Team` queue's email is that same address.

```sql
SELECT DisplayName, IsAllowAllProfiles, CreatedDate FROM OrgWideEmailAddress ORDER BY CreatedDate LIMIT 50
SELECT DeveloperName, Email FROM Group WHERE Type = 'Queue' AND Email != null LIMIT 50
```

## 6. Solution

### DDT-720
Setup → Organization-Wide Addresses → `DDT Injury Team` → added `DDT SOS` to the allowed profiles (2026-08-13). The staff member refreshed and confirmed "DDT Injury Team" appeared.

### DDT-1259
- Set up the new personal address and switched `DDT Injury Team` to the new-domain team address (2026-10-07). Asked the user to click Salesforce's verification email, and the team mailbox's owner to click its own.
- When the options still showed the old domain, walked the user through reconnecting Google: her profile picture → Settings → Connected Accounts → on the Google row with the old address, Disconnect → Connect Google again with the new address → refresh. She confirmed it worked on 2026-10-09.

### DDT-1273
Resent the verification email for the team address (2026-10-08). The verification was completed the same afternoon and the staff member confirmed the address showed up. The ticket does not record the exact final step.

### DDT-1276
Set the `PI Team` queue's email to the new team address, which is a verified org-wide address, so `pickOwaForInbox` finds an exact match again. Shared inbox replies from the PI inbox have gone out as "DDT Injury Team" from that address since 2026-10-09. A second staff member with the same symptom was asked to verify her own new address.

Why the queue rather than the code: the code already did the right thing when the queue's email matched an org-wide address. Fixing the data restored that match without a deploy.

## 7. Testing

Not applicable: no code changed. Each ticket was confirmed by the reporter:

| Ticket | Confirmation |
|---|---|
| DDT-720 | Reporter saw "DDT Injury Team" in the From dropdown (2026-08-13) |
| DDT-1259 | Reporter completed the reconnect and confirmed (2026-10-09) |
| DDT-1273 | Reporter saw the team address in the dropdown (2026-10-08) |
| DDT-1276 | Replies from the PI inbox sent from the team address since 2026-10-09 |

A test send confirming that replies route back to the right inbox (asked for in DDT-720) is not recorded.

## 8. Step by step

When someone cannot send from an address they expect:

1. **Is it an org-wide address?** Setup → Organization-Wide Addresses. If it isn't listed, add it.
2. **Is it verified?** An unverified address never appears. Resend verification and ask whoever owns that mailbox to click the link (check spam).
3. **Is the user's profile allowed?** If the address is not "all profiles", add the user's profile to its list. Check the profile is the right one for the user's team.
4. **Is it the user's own address?** Check the user record's email is the new one and verified. Then check Settings → Connected Accounts: if Google is connected under an old address, disconnect and reconnect with the new one.
5. **Is it a shared inbox reply?** Check the queue's email field (Setup → Queues) exactly matches the address of a verified org-wide address.
6. **Ask the user to refresh** and open a new email from a record, then send a test.

## 9. Rollback

Each change can be reversed in the same Setup screen. Reverting is not useful: removing the profile hides the address again, and putting back the old queue email returns shared inbox replies to the Collections fallback.

## 10. Known limits

- **The fallback hides misconfiguration.** If a queue's email ever stops matching an org-wide address again (an address renamed, a queue edited), `pickOwaForInbox` will quietly send as whichever org-wide address comes first. Nothing alerts anyone. Changing that needs a code change and a separate ticket.
- **Access follows the profile, not the person.** DDT-720 was fixed by allowing a whole profile. If a user's team and profile drift apart again, the right fix is the user's profile, as noted on DDT-720.
- **Domain moves repeat this.** Each user moving to a new domain must verify the new address and reconnect Google themselves.

## 11. Reuse for similar issues

**How to recognise one:** "the address I need is not in From", or "it went out from the wrong address".

- Work through section 8 in order: exists → verified → profile allowed → user's own connection → queue email.
- **After changing an org-wide address, check every queue whose email used the old value.** That is what turned DDT-1259 into DDT-1276.
- **Shared inbox From comes from code, not the dropdown.** Read the queue's email, not the user's settings.
- **"Only my login changed"** after an email change means the Google connection still points at the old account.
