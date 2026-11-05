# Pacific Mortgage Group

Salesforce + Make + Retell + RingCentral automation for Pacific Mortgage Group: lead intake, first text, AI follow-up calls and Mia's text replies.

This folder has one sub-folder per resolved issue. Each one explains the problem, the scope, the root cause, exactly what was changed (with IDs), how it was tested, how to roll it back, and what to reuse for similar problems.

> **Before you change anything:** read [How the system fits together](#how-the-system-fits-together) and the issue closest to your problem. Always build in the sandbox (`[UAT]` scenarios, `pacific-uat` org) first.

---

## Issues

| Date | Issue | Systems | Status |
|---|---|---|---|
| 2026-10-05 | [AI calls repeating every 15 minutes](./2026-10-05-ai-calls-repeating-every-15-min/README.md) | Salesforce flow, Make | Resolved |
| 2026-10-12 | [UAT sandbox setup](./2026-10-12-uat-sandbox-setup/README.md) | Salesforce, Make | Done |
| 2026-10-16 | [RingCentral SMS tasks not auto-logging in the sandbox](./2026-10-16-ringcentral-sms-task-autolog/README.md) | RingCentral, Salesforce | Resolved |
| 2026-10-19 | [Welcome text replaces the AI welcome call](./2026-10-19-welcome-text-replaces-ai-welcome-call/README.md) | Salesforce flow, Make | Live |
| 2026-10-19 | [Mia replies to the first text](./2026-10-19-mia-replies-to-first-text/README.md) | Make (AI agent) | Live |
| 2026-10-19 | [Retell voice changed to Myra](./2026-10-19-retell-voice-myra/README.md) | Retell | Live |
| 2026-10-20 | [SMS Type field](./2026-10-20-sms-type-field-sandbox/README.md) | Salesforce, Make | Sandbox only |
| 2026-10-20 | [Text Messages panel on the lead page](./2026-10-20-text-messages-panel/README.md) | Salesforce (LWC + Apex), RingCentral | Live |
| 2026-10-21 | [Text Messages panel rate-limit fix](./2026-10-21-text-panel-rate-limit/README.md) | Salesforce (Apex), RingCentral | Live |
| 2026-10-23 | [Voicemail message on AI follow-up calls](./2026-10-23-retell-voicemail-message/README.md) | Retell | Live |
| 2026-10-29 | [Mia's memory moved to a Make Data Store](./2026-10-29-mia-memory-to-make-datastore/README.md) | Make | Live |
| 2026-11-02 | [Duplicate first text to the same number](./2026-11-02-duplicate-text-same-number/README.md) | Make, Salesforce | Live |

New issue? Copy `_TEMPLATE.md` (in this folder of the repo; not shown on the site) to `YYYY-MM-DD-short-name/README.md` and add a row above.

---

## How the system fits together

**Systems**
- **Salesforce** (org `[org]`, CLI alias `pacific`; sandbox `[org]--uat`, alias `pacific-uat`): system of record. Leads, tasks, flows.
- **Make** (us2 zone): integration layer. All Make writes to Salesforce run as **the client admin's** Salesforce user.
- **Retell**: AI voice agents ("Mia") for the follow-up calls; calls go out from **the AI calling number**.
- **RingCentral**: phones and SMS. First texts and Mia's replies go from **the client's main RingCentral line**. The sandbox uses **the developer's line**.
- **OpenAI** (inside Make): writes the after-hours text and Mia's replies.
- Also: Cal.com (booking), Pardot / Account Engagement (nurture emails), Gravity Forms + CallRail (lead capture), WebFX (marketing agency).

**Lead lifecycle (current)**

```
Web form -> new Lead (Status New, Web = Web)
  -> flow Lead_After_create_updated (v15)
       business hours -> outbound message Make_Welcome_SMS -> Make "New Lead >>> Welcome Sms"
       off hours      -> outbound message Make_SMS_v2      -> Make "Offbusiness Hours SMS v2"
  -> Make: Status = No Contact, Not Contacted = AI 1, assign to a loan officer, send first text,
           tick Welcome_SMS_Sent__c ("SMS Sent")
  -> flow Lead Before Update Automation (Lead_Automation v34) sets Outbound_Call_Date_and_time__c
  -> Make pollers every 15 min (weekdays) pick leads whose next-call time has passed:
       "Not Contact Welcome Retry"  (AI 1-3)                -> Send Welcome Retry Calls   -> Retell [Retry]      (days 3, 7, 14)
       "No Contact Retry to Que"    (Not Contacted AI 1-3)  -> Send No Contact Retry Calls -> Retell [No Contact] (day 21 + final)
  -> Retell posts the result -> "Post Call Analysis" / "Post Call Analysis No Contact"
       -> AI Welcome Call task, next Not_Contacted__c value, SMS; the last call closes the lead
Lead texts back -> RingCentral -> Make "Mia Text Agent [production]" -> Mia replies
                                   memory in Make Data Store "Mia Text Agent DB"
```

**Key rule:** Make moves the lead's stage (`Not_Contacted__c`); the Salesforce flow sets the **date** of the next call. If the flow has no rule for a stage, the date stays in the past and the 15-minute pollers keep calling the lead (see the 2026-10-05 issue).

**Sandbox counterparts**

| Production | Sandbox |
|---|---|
| New Lead >>> Welcome Sms | [UAT] Welcome Sms |
| Offbusiness Hours SMS v2 | [UAT] Offbusiness Hours SMS |
| Assign Leads | [UAT] Assign Leads |
| Mia Text Agent [production] | Mia Text Agent [sandbox] |
| Client's Salesforce connection | Developer's Salesforce connection (sandbox) |
| Client's RingCentral connection (main line) | Developer's RingCentral connection (sandbox line) |

---

## Conventions

- **Sandbox first.** Change the `[UAT]` scenario or the `pacific-uat` org, test with your own number, then copy to production.
- **Ask before you change.** Every Make or Salesforce change, sandbox included, is explained and approved first.
- **Log it.** Every change gets a row in `Pacific-Change-Log.xlsx` (what, before, after, why, rollback, approved by).
- **Back up first.** Save the scenario blueprint or metadata before editing, and note the backup path in the doc.
- **Screenshots** go in the issue's `images/` folder, named `01-...png`, `02-...png`. Blur lead phone numbers and emails before committing if the repo is shared outside Pacific.
- **No secrets in the repo.** API keys, JWTs and client secrets stay in Salesforce Setup / Make connections.
