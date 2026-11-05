# Cannot sign in to Shadow Agent

| | |
|---|---|
| **Ticket** | DDT-262 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-02 |
| **Resolved by** | Support team |
| **Systems** | RingCentral, Shadow Agent (signs in with a RingCentral account) |
| **Area** | Staff sign-in to a RingCentral-connected app |
| **Related** | None recorded |
| **Code** | Not applicable: nothing was changed in Salesforce or the repo |

---

## 1. Summary

A staff member could not sign in to Shadow Agent, an app that uses RingCentral sign-in. RingCentral showed a "service temporarily unavailable" warning page. A second staff member reported the same problem soon after. Support asked the first user to try again and send back the full address of the warning page; on that retry the sign-in worked. No root cause was recorded.

## 2. What was reported

On 2026-07-02 a staff member reported that signing in to Shadow Agent stopped on a RingCentral warning page:

> "The service is temporarily unavailable. Please try again later. If the problem persists, please contact RingCentral support."

The page carried a reference code. About ten minutes after the ticket was opened, a second staff member reported the same issue with a screenshot. The ticket notes that the firm's RingCentral account was healthy at the platform level.

## 3. Scope

**In scope:** the RingCentral sign-in step for Shadow Agent, for the affected users.

**Out of scope:** no Salesforce component, RingCentral setting or app authorisation was changed. The re-authorisation and RingCentral support contact the ticket listed as options were not needed.

## 4. How it works (background)

```
User opens Shadow Agent
  -> Shadow Agent sends the user to RingCentral to sign in
  -> RingCentral checks the user and the app's access
  -> RingCentral returns the user to Shadow Agent, signed in
```

The warning appeared at the RingCentral step, before the user was returned to Shadow Agent.

## 5. Root cause

**Not recorded in the ticket.** Two users hit the same RingCentral warning within minutes of each other, and the first user's retry under two hours later succeeded, with no change recorded on the firm's side. The ticket does not say whether the cause was on RingCentral's side, Shadow Agent's side, or with the users' sessions.

## 6. Solution

No configuration change. Support asked the first user to sign in again and, if the RingCentral warning appeared, to copy the full web address from the browser's address bar and send it back, so the exact failing request could be inspected. On that retry the user signed in successfully and confirmed it.

Why capture the address rather than just a screenshot: the warning page's address carries the details of the sign-in request that failed, which a screenshot of the page text does not show.

## 7. Testing

Live check: the affected user confirmed they could sign in to Shadow Agent on 2026-07-02.

**Not covered:** the ticket does not record whether the second user who reported the same problem was confirmed working.

## 8. Step by step

1. **Get the exact warning text and reference code** from the user, and ask whether anyone else is affected. Several users at once points to the service rather than one account.
2. **Check that the user's RingCentral account is active** and has whatever access Shadow Agent needs.
3. **Ask the user to retry.** If the warning appears again, have them copy the **full address from the address bar** and send it.
4. **If it still fails**, re-authorise Shadow Agent's connection to RingCentral, or contact RingCentral support with the reference code and the captured address.
5. **Confirm with the user** that sign-in works.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The cause is unknown, so the same warning may come back.
- The ticket gives no confirmation for the second affected user.

## 11. Reuse for similar issues

**How to recognise one:** a sign-in to a RingCentral-connected app stops on a RingCentral-branded error page, not the app's own.

- **Count the affected users first.** One user suggests an account or permission problem; several at once suggests the service.
- **A "temporarily unavailable" page is worth one retry** before changing anything.
- **Capture the full address of the error page**, not only a screenshot, and keep the reference code for RingCentral support.
