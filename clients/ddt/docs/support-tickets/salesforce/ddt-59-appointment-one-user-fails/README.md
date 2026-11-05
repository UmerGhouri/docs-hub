# Cannot create an appointment with one specific user

| | |
|---|---|
| **Ticket** | DDT-59 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (User record, appointments) |
| **Area** | Calendar: booking appointments with a staff member |
| **Related** | None recorded |
| **Code** | None. The fix was a change to one User record. |

---

## 1. Summary

A staff member could book appointments in Salesforce with every colleague except one. The cause was that colleague's **title** on their Salesforce User record. Once the title was corrected, appointments with them could be created normally.

## 2. What was reported

"I can't make an appt in Salesforce with [one colleague]. I can do it with everyone else, but not with [them]."

One user was affected as the person being booked. The ticket does not record the exact error, or which booking screen was in use.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The affected colleague's `User` record | Record data | `Title` corrected |

**Out of scope**

- **A setup checklist for new users.** The acceptance criteria asked for one if the cause was a user-record problem. It is not recorded as written.

## 4. How it works (background)

Appointments are created against Salesforce users. Some booking screens decide which users can be booked, or how a booking is routed, from fields on the User record rather than from permissions. When one user fails and every other user works, the difference is usually on that one User record.

Which booking screen was in use, and which logic reads the title, are **not recorded in the ticket.**

## 5. Root cause

Per the resolution note: "Root cause was [the colleague's] title in the org — once that was corrected, appointment creation worked normally."

The ticket's first guesses were an inactive or frozen user, a licence, calendar sharing, or a missing permission set. **None of those was the cause.** It was an ordinary data field.

What the title was before, what it was changed to, and why the booking depended on it are **not recorded in the ticket.**

## 6. Solution

The `Title` field on the colleague's User record was corrected in Setup (Setup → Users → the user → Edit → Title). The old and new values are not recorded.

## 7. Testing

**Live check:** appointment creation with the colleague "worked normally" after the change, per the resolution note.

**Not covered:** which booking screen depends on the title, and so which other users might have the same problem.

## 8. Step by step

1. **Get the exact symptom**: error toast, validation message, silent failure, or the user missing from a list.
2. **Compare the failing User record with a working one** side by side: active, profile, role, permission sets, time zone, email, and descriptive fields such as `Title` and `Department`.
3. **Check what the booking screen reads.** Look at the component, flow or Apex behind it for any filter on User fields.
4. **Correct the differing field** in Setup → Users.
5. **Retry the booking** as the reporting user.
6. **Record the old and new value on the ticket.**

## 9. Rollback

Set the title back to its previous value. That would break bookings with this user again, so there is no reason to.

## 10. Known limits

- **A descriptive field drives behaviour.** If titles are edited for HR reasons, bookings can break again without anyone touching Salesforce configuration.
- **The dependency is undocumented.** The ticket doesn't record which logic reads the title, so it can't be checked for other users.

## 11. Reuse for similar issues

**How to recognise one:** a process works for every user except one.

- **Diff the one User record against a working one** before investigating permissions or licences.
- **Check fields people think of as cosmetic**, such as title, department and role. Automation and components often filter on them.
- **Write down which field mattered and why**, so the next case is a two-minute fix.
