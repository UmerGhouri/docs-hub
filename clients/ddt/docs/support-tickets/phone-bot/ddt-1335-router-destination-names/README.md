# Phone bot alert shows the queue name for the router destination

| | |
|---|---|
| **Ticket** | DDT-1335 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-19 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (Slack alerts), RingCentral |
| **Area** | The Router line on missed-call alerts |
| **Related** | None recorded |
| **Code** | PR #19 in the phone bot (`sf-ops-bot`) repo |

---

## 1. Summary

The phone bot's missed-call alerts showed where the call router sent a call as a bare extension number, so staff had to remember which queue each number was. The alert now shows the RingCentral name next to the number. Queues needed their own lookup, because the bot's existing list of extensions only held users.

## 2. What was reported

On 2026-10-14 firm staff asked for the Router line on missed-call alerts to name the destination. The line read like `tried ext <N> (<rep name>) — was unavailable — sent to <N>`: the first hop already showed a name, but the destination was just a number. Case Management staff use these alerts to decide whether to call a repeat caller back, and mapping numbers to teams in their heads was slow and error-prone.

The ticket asked for:

- every router hop to show `<Name> (ext <N>)`;
- queue extensions to resolve, not just user extensions;
- the alert still to render, with the raw number, if a lookup fails.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| `app/clients/router_db.py` in the phone bot repo | The Router line resolves each destination extension to its RingCentral display name |
| A new lookup across all RingCentral extension types | Added alongside the existing user-only registry |

**Out of scope**

- **The existing rep registry.** It was deliberately not widened to include queues (see 6).
- **Salesforce.** Nothing in Salesforce changed.

## 4. How it works (background)

```
Missed call
  -> phone bot builds the Router line: who was tried, and where the call went next
  -> posts the alert to Slack
```

Before the change, the bot named the first hop from its **rep registry**, which only loads RingCentral **user** extensions. Queue and department extensions were not in it, so the destination printed as a number.

## 5. Root cause

The only extension-to-name lookup the bot had was the rep registry, and it only covers user extensions. Destinations are usually queues, so there was no name to show.

## 6. Solution

Added a separate lookup across all RingCentral extension types, used only for the Router line's destination.

```
before:  sent to <N>
after:   sent to All Intake Members (ext <N>)
```

Why a separate lookup rather than widening the rep registry: the registry is also used to name the **rep who missed the call**. If it included queues, a queue could be named as the rep who missed a call.

The extension number stays visible so the raw routing can still be audited, and any failed lookup falls back to the number alone.

## 7. Testing

Verified live on 2026-10-19: three queue destinations resolved to their names, **CMT: Defense**, **All Intake Members** and **Intake + CMT**.

Automated tests in the bot's repo are not recorded in the ticket.

## 8. Step by step

1. Find where the bot builds the Router line (`app/clients/router_db.py`).
2. Confirm which lookup names the first hop, and whether it covers queues.
3. Add a lookup over all RingCentral extension types, keyed by extension number.
4. Use it only for the destination, formatting as `<Name> (ext <N>)`.
5. On a failed or empty lookup, print the number as before.
6. Deploy, and check a live alert whose call went to a queue.

## 9. Rollback

Revert PR #19 in the phone bot repo and redeploy. The alert goes back to showing bare numbers; nothing else depends on the change.

## 10. Known limits

- **Names come from RingCentral's display names**, so the alert reads whatever the queue is called there.
- **A failed lookup shows the number only**, by design.

## 11. Reuse for similar issues

**How to recognise one:** an alert or report shows an internal number that people have to translate in their heads.

- **Resolve to a name, keep the number.** The name helps the reader; the number keeps it auditable.
- **Don't widen a lookup that something else depends on.** Here, adding queues to the rep registry would have mislabelled who missed a call. Add a separate lookup instead.
- **Always degrade to the raw value** when the lookup fails, so the alert still posts.
