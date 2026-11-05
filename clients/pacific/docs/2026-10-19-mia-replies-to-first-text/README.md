# Mia replies to the first text

| | |
|---|---|
| **Status** | Live since 2026-10-19 |
| **Reported by** | Client admin (test on 2026-10-14) |
| **Resolved by** | Support team |
| **Systems** | Make (AI agent), Salesforce |
| **Area** | Mia's text replies |
| **Change log rows** | 20–22, 39 |

---

## 1. Summary

When a lead answered the welcome text with "Now" ("are you free for a quick call right now?"), Mia replied with a generic opener as if the conversation had just started. Mia had no idea a welcome text had been sent, when the lead came in, or who their loan officer was.

We now look up the lead in Salesforce before Mia replies, pass the lead's context into Mia's instructions, and add rules for replies to our first text.

## 2. Client report

The client admin replied "Now" to the welcome text, and Mia asked "Are you looking to buy a home, refinance, or just have some questions about financing?"

_Screenshot in the private copy: 01-generic-reply-to-now.png (Generic reply to "Now")._

## 3. Scope

**In scope:** `Mia Text Agent [production]`, modules 41, 44, 45, 47, 49, 50 only; sandbox clone `Mia Text Agent [sandbox]`.

**Out of scope:** Mia's original instructions (kept word for word, with a section appended), booking (Mia only sends the Cal.com link), `Mia Stop Mia Text`.

## 4. How Mia works (background)

```
Lead texts the main line -> RingCentral webhook -> Mia Text Agent [production]
   -> find this phone's conversation in Mia's memory (Data Store; Google Sheet before 2026-10-29)
   -> router: three routes, each with its own AI agent module (#47, #49 = thread already exists, #50)
   -> reply by SMS from the main line, save thread id
```

## 5. Root cause

The AI agent modules only received the incoming message and the thread. No lead data, so "Now" had no meaning.

## 6. Solution

| Module | Change |
|---|---|
| #41 | **SOQL lookup:** newest **unconverted** lead whose mobile matches the sender |
| #44 / #45 | Build the context: `welcome_sms_sent`, `came_in` (business hours / after hours), `lead_created`, `loan_officer`, `lead_status` |
| #47 / #49 / #50 | A **"Lead context"** block added to each agent's input, and a **"Replies to our first text"** section appended to the original prompt |

**Rules in "Replies to our first text"**
- "Now" / "yes" in business hours → a loan officer will call shortly (name used here only).
- "Now" after hours → the team is offline; send the booking link `cal.com/team/pacific-mortgage-group/consultation`; no staff name.
- A time ("2pm EST") → convert to Pacific time and offer the booking link.
- "Who is this?" → explain Pacific Mortgage Group and the inquiry they made.
- "Not now" / not interested / wrong number → polite close.

**Loan officer name:** left blank while the lead is still owned by the client admin (production) or the developer (sandbox), because that means it isn't assigned yet. In testing, Mia wrote "our team (including the developer's name)" in an after-hours reply; that's why this rule exists.

_Screenshot in the private copy: 02-staff-name-in-offhours-reply.png (Staff name in an after-hours reply (sandbox, fixed))._

## 7. Testing (sandbox, 2026-10-15)

| Reply | Result |
|---|---|
| "Now" after hours | Team offline + booking link, no staff name: Pass |
| "Not now", "Who is this?", "2pm EST" (→ 11 AM PT), FHA rate question, wrong number / not interested | Pass |
| "Stop" | Do Not Text + Not Interested, no reply (6 operations): Pass |

Sandbox test without a real phone: POST a RingCentral-format inbound SMS JSON to the clone's webhook.

## 8. Rollout
Production changed only in modules 41, 44, 45, 47, 49, 50, verified against the tested sandbox build. Backup: `mia_prod_backup_2026-10-19.json`.

## 9. Rollback
Restore the backup blueprint into the scenario (Make → scenario → ⋮ → Import blueprint).

## 10. Known limits
- Mia only sends the booking link; she can't book. A "Meeting Booked … by AI Agent" task comes from **any** Cal.com booking, including the lead booking it themselves.
- If two open leads share a number, Mia uses the newest one.

## 11. Reuse for similar issues
- **AI gives generic answers?** Give it the record's context (status, owner, what we already sent) in the input, not just the message.
- **Append to a client-approved prompt rather than rewriting it,** and keep a backup of the original.
- **Hide staff names until the record is really assigned.** "Owned by the admin" means not assigned yet.
