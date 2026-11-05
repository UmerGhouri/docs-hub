# Voicemail message on AI follow-up calls

| | |
|---|---|
| **Status** | Live since 2026-10-23 |
| **Requested by** | Client admin, 2026-10-23 |
| **Done by** | Support team (Retell dashboard) |
| **Systems** | Retell |
| **Area** | Day 3 / 7 / 14 / 21 / final AI calls |
| **Change log rows** | Not logged (Retell dashboard change) |

---

## 1. Summary

When an AI follow-up call reached voicemail, the agent **hung up** without leaving a message. The client admin asked for a short message instead. We turned on Retell's **voicemail detection → leave a message (static text)** on both follow-up agents, with the lead's first name.

## 2. Client request

> "When the AI makes outbound calls and reaches voicemail, can we program her to leave a short message instead of hanging up? The message should mention that we're following up on their online inquiry, provide our phone number, and ask them to call us back."

## 3. Scope

| Agent | ID | Calls |
|---|---|---|
| Pacific Mortgage **[Retry]** | `[agent id]` | Days 3, 7, 14 (Make "Send Welcome Retry Calls") |
| Pacific Mortgage **[No Contact]** | `[agent id]` | Day 21 + final (Make "Send No Contact Retry Calls") |

_Screenshot in the private copy: 01-retry-agent-id.png (Retry agent ID)._
_Screenshot in the private copy: 02-no-contact-agent-id.png (No Contact agent ID)._

**Out of scope:** [Welcome] (inbound only now), chatbot agents, Make and Salesforce (no change; voicemail calls still end as "voicemail reached" and Post Call Analysis handles them as before).

## 4. How voicemail detection works

1. Make starts the call through Retell from **the AI calling number**, naming the agent (`override_agent_id`) and passing `first_name`, last name, email, lead ID and the last call summary.
2. If nobody answers, the carrier sends the call to voicemail. Retell **listens** during the first part of the call (up to 3 minutes) for a recorded greeting and beep.
3. When it detects voicemail, it applies the agent's voicemail setting: **hang up** (the old setting), **static text**, **prompt-generated message**, or **transfer**.

The message plays **only if the call actually reaches voicemail:**

| Lead's side | Retell result | Message plays? |
|---|---|---|
| Voicemail set up (most mobiles) | voicemail reached | Yes |
| Rings out, no voicemail | no answer | No |
| Declined | user declined, or voicemail if forwarded | Only if forwarded to voicemail |
| Unknown callers silenced / spam-filtered | voicemail or blocked | Only if it reaches voicemail |
| Busy / invalid number | busy / failed | No |

## 5. Solution

On **both** agents: **Agent → Call settings → Voicemail → Leave a message (static text)**, then **Publish**.

Message:
> Hi `{{first_name}}`, this is Mia from Pacific Mortgage Group following up on your online inquiry. Please call us back at (555) 010-0100. That's (555) 010-0100. Thank you.

- **Why `{{first_name}}`:** Make already sends it on every follow-up call. In production, **0 of 349** leads waiting for AI calls had a blank first name (29 had odd casing or a single letter, which is acceptable).
- **Why the number twice:** callers write it down on the second hearing.
- **Why "Mia":** both agents introduce themselves as Mia in live calls.

## 6. Testing

Tested on a **copy** of [Retry], so live calls weren't affected:
1. Duplicate [Retry], rename it "VM Test", **clear its webhook URL** (so the test call can't run Post Call Analysis on a real lead), add the voicemail message, and publish.
2. Phone Numbers → **the AI calling number** → set the outbound agent to the test copy → **Make an outbound call** to your own number with `first_name` set → don't answer.
3. **Immediately set the number's outbound agent back** to what it was (Pacific Mortgage [Welcome]).
4. Check Retell **Call History** (result "voicemail reached", transcript shows the message) and listen in your voicemail.

_Screenshot in the private copy: 03-test-call-number-settings.png (Test call number settings)._
_Screenshot in the private copy: 04-test-voicemail-transcript.png (Test voicemail transcript)._

Then the same setting was applied to [Retry] and [No Contact] and published, and both were tested.

## 7. Rollback
Set Voicemail back to **Hang up** on both agents and publish.

## 8. Known limits
- **Caller ID shows "CORONA CA"**: the AI calling number has no registered caller name (CNAM), so phones show the number's city. Registering a business name for the number is a separate request to the number provider.
- Calls that leave a message run a few seconds longer, so Retell's cost per call goes up slightly.
- **Clean-up:** check the outbound agent on the AI calling number is back to its original and delete the "VM Test" copy.

## 9. Reuse for similar issues
- **Test Retell changes on a duplicated agent with its webhook cleared.** A test call with the live webhook will update the lead with your phone number in Salesforce.
- **Agent settings must be on every agent that calls.** Make chooses the agent per call (`override_agent_id`).
- **Before adding a dynamic variable** to a message, check Make sends it and how often it's blank in Salesforce.
