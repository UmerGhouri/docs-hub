# UAT sandbox setup

| | |
|---|---|
| **Status** | Done 2026-10-12 – 2026-10-16, in use |
| **Requested by** | Client admin, 2026-10-12 ("can we first build the new automation and test it in a sandbox before implementing?") |
| **Done by** | Support team |
| **Systems** | Salesforce, Make, RingCentral |
| **Area** | Test environment |
| **Change log rows** | 1–4, 7, 13–20 |

---

## 1. Summary

Pacific had no sandbox (the old UAT, pmguat and UATPMG sandboxes were deleted in 2025). We created a **UAT Partial Copy** sandbox and sandbox copies of the Make scenarios, wired so the sandbox can **never** trigger production automations, call real leads or text from the client admin's line.

## 2. Scope

**In scope**
- Salesforce sandbox `[org]--uat` (CLI alias `pacific-uat`).
- Sandbox outbound messages, call center, test user.
- Make `[UAT]` scenario copies and sandbox connections.

**Out of scope**
- Retell (one account; the sandbox doesn't place AI calls).
- RingCentral account (one account; the sandbox texts from the developer's line).

## 3. The problem a sandbox copy creates

A sandbox copies production metadata, **including endpoints**. Out of the box, a sandbox lead would:
- fire outbound messages to **production** Make webhooks, causing real AI calls, real texts from the main line and real assignments;
- log RingCentral activity to the **production** CTI adapter.

## 4. Solution

### 4.1 Salesforce sandbox

| Item | Change | Why |
|---|---|---|
| Outbound message `Make_Welcome_SMS` | Endpoint → `[UAT] Welcome Sms` webhook | Business-hours first text goes to the sandbox scenario |
| Outbound message `Make_SMS` | Endpoint → `[UAT] Offbusiness Hours SMS` webhook | Off-hours text goes to the sandbox scenario |
| **All other** Lead/Event outbound messages | Endpoint → `https://example.com/uat-disabled` | Nothing else can reach production Make (AI calls, alerts). Backups in `uat/backup` |
| Developer's user | Loan Officer role removed | Test leads aren't assigned to the developer as a loan officer |
| Developer's user | Added to the RingCentral call center | RingCentral panel and SMS auto-logging in the sandbox |
| RingCentral call center (CTI adapter URL) | `https://[org]--rcsfl.vf.force.com/...` → `https://[sandbox-org]--rcsfl.sandbox.vf.force.com/apex/OpenCTIIndex` | The copy pointed at production, so the panel would log to production |

### 4.2 Make

| Sandbox scenario | Copy of | Differences |
|---|---|---|
| `[UAT] Welcome Sms` | New Lead >>> Welcome Sms | Sandbox Salesforce connection; texts from the **sandbox line** on the developer's sandbox RingCentral connection |
| `[UAT] Offbusiness Hours SMS` | Offbusiness Hours SMS | Same connections as above |
| `[UAT] Assign Leads` | Assign Leads | Its own AI-agent thread; output type Text; logged-in loan officer check kept |
| `Mia Text Agent [sandbox]` | Mia Text Agent [production] | Listens on the sandbox line, sandbox Salesforce, replies from the sandbox line |

To clone a scenario: **Make → scenario → ⋮ → Clone**, choose "create a new webhook" for the trigger (never reuse the production hook), then swap every connection to the sandbox one.

## 5. Testing

- Off-hours lead → one Mia text; business-hours lead → one welcome text (8 operations); lead set back to New → no second text; lead assigned from the client admin to a logged-in loan officer. All passed 2026-10-13.

## 6. Rollback

Not applicable. To return a sandbox endpoint to production, use the original URLs in the change log (rows 2–4). Never do this in a sandbox that real users test in.

## 7. Known limits

- **One RingCentral account and one Retell account** serve both orgs. The sandbox shares RingCentral API rate limits with production (see the panel rate-limit issue).
- **Sandbox data is a copy from 2026-10-12.** SMS tasks after that date are only in production.
- **Login:** `test.salesforce.com`, production username + `.uat`, password as of the refresh date.
- **An older Make RingCentral connection named after the developer is not logged in as the developer.** Sending from the sandbox line through it fails with `403 MSG-304`. Use the sandbox RingCentral connection.

## 8. Reuse for similar issues

- **After any sandbox refresh, repeat section 4.1**, or the sandbox will hit production webhooks.
- **Every new automation gets a `[UAT]` twin** with its own webhook and sandbox connections, and the sandbox is changed first.
- **Test with your own phone number** (or the sandbox line), never a real lead's.
