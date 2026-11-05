# Setting up a Calendly account for a new user

| | |
|---|---|
| **Ticket** | DDT-679, DDT-1156 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Calendly (organisation directory, user invitations) |
| **Area** | Staff onboarding: scheduling links for client meetings |
| **Related** | None |
| **Code** | None. Done in Calendly's admin screens. |

---

## 1. Summary

Two requests about getting a staff member onto the firm's Calendly organisation. The first was a plain new account. The second looked like a broken booking link, but the cause was an invitation that had not been accepted: after the user's email address was changed, Calendly re-issued the invitation and her seat stayed **Pending** until she accepted it, so her booking pages did not exist. Once she accepted and set up again, the link worked.

## 2. What was reported

- **DDT-679, 2026-08-07.** A staff member had no Calendly account and could not share a booking link for client meetings. A colleague asked for one to be set up under the firm's Calendly team. The admin replied the same day that she would complete it; the ticket closed on 2026-08-21.
- **DDT-1156, 2026-09-29.** A staff member could not book a case-update call on an attorney's Calendly. The booking link showed "This Calendly URL is not valid." In the organisation directory the attorney's seat was on the **Pending** tab, with an invitation sent the day before.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Calendly organisation directory | Vendor admin screen | Invite the user (DDT-679); have the user accept the re-issued invitation (DDT-1156) |

**Out of scope**

- **Moving old meetings to the new account.** After accepting, the attorney found no upcoming events and asked whether there was a migrate option. The sources do not record an answer. See 10.

## 4. How it works (background)

```
Admin invites a user's email to the Calendly organisation
  -> seat shows as Pending in the organisation directory
  -> user opens the invitation email, accepts, signs in, sets up calendar and event types
  -> seat becomes Active
  -> the user's booking pages (their scheduling links) go live
```

Until the invitation is accepted, the user's booking links do not resolve. Calendly shows "This Calendly URL is not valid", which looks like a broken link rather than an account that is not finished.

## 5. Root cause

- **DDT-679:** not a fault. The user had no account yet.
- **DDT-1156:** the attorney's email address had been updated in Calendly the day before. That sent her a new invitation, and her seat sat in **Pending** until she accepted it. A pending seat has no live booking pages, so the link failed. This was the cause the ticket suggested, and the fix confirmed it.

## 6. Solution

- **DDT-679:** the Calendly admin invited the user to the firm's Calendly organisation. The exact seat and event-type settings are not recorded in the ticket.
- **DDT-1156:** the Calendly admin asked the attorney to accept the updated invitation. She accepted it and went through Calendly's setup again with the new email address. The booking link then worked.

## 7. Testing

- **DDT-1156:** the staff member who reported it confirmed the same day: "It is working now."
- **DDT-679:** the user's confirmation is not recorded in the ticket.

**Not covered:** whether existing booked meetings carried across to the re-created account.

## 8. Step by step

**New user**

1. Sign in to Calendly as an organisation admin and open the organisation's people / directory page.
2. Invite the user's firm email address and put them in the right group (for example, Attorneys).
3. Ask the user to open the invitation email, accept it, sign in, and connect their calendar.
4. Check the user now shows as Active, not Pending.
5. Ask the user to open their booking link, or open it yourself, and confirm it loads.

**Booking link says "This Calendly URL is not valid"**

1. Look the user up in the organisation directory. If they are on the **Pending** tab, the invitation has not been accepted.
2. Resend the invitation if needed, and ask the user to accept it and finish setup.
3. Once the seat is Active, open the link again.

## 9. Rollback

Not applicable: nothing was changed that would need undoing. A seat can be removed from the organisation in Calendly's admin screens if it was added by mistake.

## 10. Known limits

- **Changing a user's email re-issues the invitation.** Their links stop working until they accept it again. Warn the user before changing it.
- **Re-setting up may leave old events behind.** The attorney saw no upcoming events after accepting. Plan to re-book or check with Calendly support before changing an email for someone with meetings booked.

## 11. Reuse for similar issues

**How to recognise one:** a Calendly link that says "This Calendly URL is not valid" for a staff member, especially soon after onboarding or an email change.

- **Check the organisation directory's Pending tab first.** A pending invitation explains a dead link.
- **The fix is on the user's side:** they must accept the invitation and complete setup. The admin can only resend it.
- **Confirm by opening the link**, not by the seat status alone.
