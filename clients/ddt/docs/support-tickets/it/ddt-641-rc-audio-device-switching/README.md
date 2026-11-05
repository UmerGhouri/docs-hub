# RingCentral keeps switching audio devices

| | |
|---|---|
| **Ticket** | DDT-641 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-04 |
| **Resolved by** | Support team |
| **Systems** | RingCentral desktop app, computer sound settings |
| **Area** | Taking calls: microphone and speaker selection |
| **Related** | None recorded |
| **Code** | None |

---

## 1. Summary

A staff member reported that the RingCentral desktop app kept changing her microphone and speaker on its own, so on calls neither side could hear the other. The ticket was closed on 2026-09-16. The sources do not record what was found or changed.

## 2. What was reported

On 2026-08-04 one staff member reported that RingCentral kept switching her audio input and output devices by itself. When calls came in, callers could not hear her and she could not hear them. She confirmed in the thread that this was the **RingCentral desktop app**, not the RingCentral agent tool used inside Salesforce. It blocked her day-to-day call work. One user was affected.

## 3. Scope

**In scope:** audio device settings in the RingCentral desktop app and in the computer's sound settings, on one user's machine.

**Out of scope:** RingCentral call routing and the Salesforce telephony integration. Nothing in the report pointed at either.

## 4. How it works (background)

```
Computer sound settings: default microphone and speaker
  -> RingCentral desktop app: Settings -> Audio
       either follows the system default
       or uses a device picked in the app
  -> call audio
```

If the app follows the system default, anything that changes the system default (a Bluetooth headset switching, another app taking the device) changes RingCentral's devices too, mid-call or between calls.

## 5. Root cause

Not recorded in the ticket. The ticket listed likely causes to check: the computer's default audio device changing, a Bluetooth headset switching automatically, or the app set to follow the system default.

## 6. Solution

Not recorded in the ticket. The plan written into the ticket was:

1. In the RingCentral desktop app, open Settings, then Audio, and pick the exact microphone and speaker instead of following the system default.
2. In the computer's sound settings, set the same devices as the default and make sure no other app is taking them.
3. Make an inbound and an outbound test call.

## 7. Testing

Not recorded in the ticket. The acceptance test was that the chosen devices stay selected across calls and restarts, and that a test call has two-way audio.

## 8. Step by step

The steps planned in the ticket, for repeating the check:

1. Confirm which app the problem is in: the RingCentral desktop app or the agent tool inside Salesforce.
2. In the desktop app, open **Settings**, then **Audio**. Choose the headset's microphone and speaker by name.
3. Open the computer's sound settings. Set the same headset as the default input and output.
4. If a Bluetooth headset is used, check whether it is switching devices automatically, one of the causes the ticket listed.
5. Restart the app and make an inbound and an outbound test call.

## 9. Rollback

Not applicable: these are per-user device settings with nothing to undo.

## 10. Known limits

The cause and the fix were not written down, so a repeat of this problem has no confirmed answer to start from.

## 11. Reuse for similar issues

**How to recognise one:** one user, one-way or no audio, and devices that "change on their own".

- **Ask which app first.** The desktop app and the agent tool inside Salesforce have separate audio settings.
- **Pin devices by name** in the app rather than following the system default.
- **Record what fixed it** in the ticket before closing, so the next case starts from an answer.
