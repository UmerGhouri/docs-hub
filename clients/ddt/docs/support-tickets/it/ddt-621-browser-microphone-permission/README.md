# Browser microphone permission blocked for the RingCentral dialer

| | |
|---|---|
| **Ticket** | DDT-621 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-03 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | RingCentral for Salesforce (embedded dialer), Chrome browser, RingCentral desktop app |
| **Area** | Placing outbound calls from the dialer inside Salesforce |
| **Related** | DDT-622 (same user, transfer problem reported the same day) |
| **Code** | Not applicable: a dialer setting, no code |

---

## 1. Summary

One user could not dial out from the RingCentral dialer embedded in Salesforce, because the browser would not let RingCentral use the microphone and the user could not add RingCentral to the browser's microphone allow list. Rather than fixing the browser permission, the dialer was set to place calls through the RingCentral desktop app installed on the computer, which already worked.

## 2. What was reported

On 2026-08-03 the dialer showed: *"To use this function, go to your browser settings to allow RingCentral to access microphone."* One user was affected.

Troubleshooting before the ticket was filed found:

- Salesforce **was** on the browser's microphone allow list.
- RingCentral **was not**.
- When the user tried to add RingCentral (or any site) on Chrome's microphone settings page, the browser gave no option to add one. The ticket suggested the setting might be locked by a managed policy or a restricted Chrome profile.
- The RingCentral desktop app worked normally, so the problem was limited to the browser.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| RingCentral dialer in Salesforce, for the affected user | Dialer setting | Calls placed through the installed RingCentral app instead of the browser |

**Out of scope**

- **The browser's microphone policy.** Whether Chrome was under a managed policy was not confirmed, and the allow list was not changed.

## 4. How it works (background)

The RingCentral dialer in Salesforce can place a call in more than one way. The two that matter here:

```
Browser:  dialer in Salesforce -> call runs in the browser
          (the dialer's error asks for RingCentral to be allowed the microphone)
App:      dialer in Salesforce -> call is placed through the RingCentral app
          installed on the computer
```

Before the fix the user's dialer was using the browser, which is why the browser's microphone permission mattered.

## 5. Root cause

The browser had not granted RingCentral microphone access, and the user could not grant it from the browser's settings. Why the allow list could not be edited (a managed policy, a restricted profile, or something else) is not recorded in the ticket.

## 6. Solution

During a screen-share with the user, the RingCentral dialer in Salesforce was configured to **place calls through the RingCentral app installed on the computer**. The desktop app already worked, so outbound calling from Salesforce worked once calls were routed through it.

Why this route: it avoided changing browser permissions that the user could not change, and it reused a part of the setup that was already known to work.

## 7. Testing

The fix was reported as working in the ticket on 2026-08-03. No further test is recorded.

**Not covered:** the browser-based calling mode for this user still lacks microphone access.

## 8. Step by step

1. Confirm the error text in the dialer is the microphone-permission message.
2. Check the browser's microphone settings. Note whether RingCentral is allowed and whether the user can add a site.
3. Confirm the RingCentral desktop app is installed and can place calls.
4. In the RingCentral dialer inside Salesforce, open its settings and set calls to be placed with the RingCentral app on the computer rather than the browser.
5. Have the user place a test outbound call from Salesforce.

## 9. Rollback

Switch the dialer's calling setting back to the browser. This brings back the original error unless the browser's microphone permission is fixed first.

## 10. Known limits

- The user's calls now depend on the desktop app being installed, signed in and running.
- The browser permission issue itself was not resolved, so switching back to browser mode will fail the same way.

## 11. Reuse for similar issues

**How to recognise one:** the dialer in Salesforce asks for microphone access, but the RingCentral desktop app works fine on the same computer.

- Check the browser's microphone allow list for both Salesforce and RingCentral.
- If the user cannot add a site, the browser may be under a managed policy. Fixing that needs whoever controls the policy.
- The quickest workaround is to have the dialer place calls through the installed RingCentral app.
