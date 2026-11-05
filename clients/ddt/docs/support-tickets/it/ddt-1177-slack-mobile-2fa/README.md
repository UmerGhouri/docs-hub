# Cannot log into Slack mobile after 2FA setup

| | |
|---|---|
| **Ticket** | DDT-1177 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-08 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Slack (mobile app) |
| **Area** | Staff access to Slack on a phone |
| **Related** | None |
| **Code** | Not applicable: no code or configuration change |

---

## 1. Summary

A staff member's Slack mobile app logged him out and asked him to set up two-factor authentication. After he entered the correct security code, the app still would not let him sign in. He said he had force-quit the app on his phone. When the support team followed up on 2026-10-08, he confirmed it worked.

## 2. What was reported

On 2026-10-01 a staff member reported that the Slack mobile app had signed him out and prompted for two-factor setup, and that entering the correct code did not get him back in. He could not use Slack on his phone. One user was affected.

## 3. Scope

**In scope:** the one user's Slack mobile sign-in.

**Out of scope:** Slack workspace settings and two-factor policy. Nothing was changed.

## 4. How it works (background)

```
Slack mobile app signs the user out
  → prompts for two-factor setup
  → user enters the code from their authenticator
  → app should complete sign-in
```

## 5. Root cause

Not recorded in the ticket. The only step recorded is that the user force-quit the app; the ticket does not say whether that, or something else, cleared it.

## 6. Solution

The user force-quit the Slack app on his phone on the day of the report. On 2026-10-08 the support team followed up, and the user confirmed it worked. No other action is recorded.

## 7. Testing

The user confirmed on 2026-10-08 that he could sign in on mobile.

## 8. Step by step

1. Ask the user to force-quit the Slack mobile app, then reopen it.
2. Sign in again and complete the two-factor prompt.
3. Follow up with the user to confirm they can sign in.
4. If it still fails, the ticket records no further steps.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

The cause is unknown, so the same sign-in loop could happen again.

## 11. Reuse for similar issues

**How to recognise one:** a mobile app accepts a correct two-factor code but does not finish signing in.

- **Force-quit and reopen the app first.** It is quick, and it is the only step recorded before this one was resolved.
- **Follow up with the user** before closing; this ticket waited a week for confirmation.
