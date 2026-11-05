# Incoming ringtone won't stop during an active call

| | |
|---|---|
| **Ticket** | DDT-828 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-01 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | RingCentral desktop app |
| **Area** | Taking back-to-back calls: call waiting while already on a call |
| **Related** | None |
| **Code** | None. Guidance given to the user; no configuration change recorded |

---

## 1. Summary

A rep on an active RingCentral call heard a second incoming call ring loudly and continuously, drowning out the caller they were talking to. Support gave the rep app settings to quiet or move the ringer, and the two ways to handle a second call. Turning Call Waiting off was offered but held back, because it changes where unanswered calls go and needed the firm's decision.

## 2. What was reported

While the rep was on a call, a second call came in (the app showed two calls: one active, one incoming). The incoming ringtone did not stop, and it was loud enough that the rep could not hear or continue the active call. Reps take back-to-back calls, so this affected call quality and the client's experience.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| RingCentral desktop app, Settings, Audio | Per-user app setting | Guidance only: ringtone volume and ringer device |

**Out of scope**

- **Turning off Call Waiting for the user.** Offered, but not done without the firm's approval (section 6.3). Whether it was later turned off is not recorded in the ticket.
- **Account-wide call-waiting settings.** Not changed.

## 4. How it works (background)

With **Call Waiting on**, a RingCentral user who is already on a call still gets a second incoming call: the app shows it and plays the ringtone through the configured **ringer device**. If the ringer device is the same headset the user is talking on, the ringtone plays in their ear over the conversation.

With **Call Waiting off**, a second call does not ring the busy user at all. A direct call goes to their voicemail; a queue call goes to another member of the queue.

## 5. Root cause

Not recorded in the ticket. The ticket asked whether this was an app setting, an account setting or a RingCentral bug; the reply addressed the symptom (ringer volume and device) without stating which.

## 6. Solution

### 6.1 Quiet the ringer, or move it off the headset

In the RingCentral desktop app:

1. Click the gear icon (Settings).
2. Go to **Audio**.
3. Lower **Ringtone volume**, or change **Ringer device** to the computer's speakers instead of the headset.

This stops a second call's ringtone from playing loudly in the ear during an active call. The rep's app was in Spanish, so the reply also gave the Spanish labels for each setting.

### 6.2 Handle the second call

When a second call comes in, either:

- put the current call on **Hold** and answer the new one, or
- **Decline** the incoming call. A queue call goes back to the queue and another member picks it up.

### 6.3 Turning Call Waiting off (held back)

Support offered to turn Call Waiting off for the user, but did not do it without the firm's approval. With it off, a direct call to a busy rep goes to **voicemail** instead of ringing. That is worse for clients than a queue call rolling to another rep, so it is a business decision, not a support fix.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for the rep to confirm the fix resolved the interference; that confirmation is not recorded.

## 8. Step by step

1. Ask which device the rep is talking on and which device the ringer is set to.
2. In the RingCentral desktop app, open Settings, then Audio.
3. Lower **Ringtone volume**, or set **Ringer device** to the speakers so the ring is no longer in the headset.
4. Remind the rep of Hold and Decline for a second call.
5. Only with the firm's approval, turn off Call Waiting for that user, and tell them that direct calls will then go to voicemail while they are on a call.
6. Ask the rep to confirm on their next overlapping call.

## 9. Rollback

Set the ringtone volume or ringer device back in the same Audio settings. Nothing account-level was changed.

## 10. Known limits

- **Moving the ringer to speakers** means the rep may miss the second call if they are away from the desk or the speakers are muted.
- **The underlying cause was not established.** If the ringtone genuinely never times out, that is still unexplained.
- **Call Waiting off** sends direct calls to voicemail while the rep is busy.

## 11. Reuse for similar issues

**How to recognise one:** a rep complains about sound from the app during a call (a ring, a notification) rather than about the call audio itself.

- **Check the ringer device first.** If it is the same headset as the call audio, every incoming ring plays in the rep's ear.
- **Separate the ringer from the call audio** before changing any account-level setting.
- **Treat Call Waiting as a routing decision.** Turning it off changes where clients' calls go, so it needs the firm's sign-off.
- **Give instructions in the language the user's app is in.**
