# Disabling the post-close case survey automation

| | |
|---|---|
| **Ticket** | DDT-98 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-03 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-06-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Salesforce Surveys, configured in production) |
| **Area** | Admin notifications: daily Salesforce Surveys failure emails |
| **Related** | None |
| **Code** | No pull request. The change was made in production, not in code |

---

## 1. Summary

An administrator was getting about one email a day from Salesforce saying survey emails "can't be sent" because no default community was selected for Surveys. The firm was not trying to send surveys at all. The source was two unused **sample surveys** (CSAT and NPS) that were active in production. Deactivating them stopped what was trying to send.

## 2. What was reported

About one email per day from Salesforce Surveys, subject "Salesforce survey emails can't be sent in Driver Defense Team", explaining that no default community was selected for Surveys. The ticket believed a survey was being sent automatically when a Case closed, and asked for that automation to be found and switched off.

No users were blocked. It was a recurring nuisance email.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| CSAT sample survey | Salesforce Survey | Deactivated in production |
| NPS sample survey | Salesforce Survey | Deactivated in production |

**Out of scope**

- **Choosing a default community for Surveys.** That is what the email asks for, but it would have made the surveys work rather than stop. The firm did not want surveys sent.
- **Case automation.** No flow or process was changed. The resolution recorded in the ticket names the sample surveys, not a Case flow.

## 4. How it works (background)

```
An active survey is set up to send invitations
  Salesforce Surveys tries to send a survey email
  no default community is selected for Surveys, so the send fails
  Salesforce emails the admin that survey emails can't be sent
```

As the failure email itself says, Salesforce Surveys needs a default community selected before it can send survey emails. Without one, every attempted send fails and generates a notice to an administrator.

## 5. Root cause

Two sample surveys that ship with Salesforce Surveys, a CSAT survey and an NPS survey, were active in production and unused. They were what kept attempting to send.

**The ticket's first guess was a Case-close flow or process.** The resolution recorded in the ticket does not name any Case automation: deactivating the two surveys alone was what "removes the trigger". The query or screen used to trace the sends back to these surveys is not recorded in the ticket.

## 6. Solution

The two sample surveys were deactivated in production. No code or metadata change was made.

Why this, and not the alternatives:

- **Deactivate, not delete.** Deactivating stops sends and keeps the surveys, so the change can be reversed if the firm ever wants surveys.
- **Not setting a default community.** That would have turned failing sends into successful ones, emailing surveys to Case contacts, which the firm did not want.

## 7. Testing

Not applicable: no code changed. The ticket's acceptance check was that the daily failure email stops within one to two days. Whether that was confirmed afterwards is not recorded in the ticket.

## 8. Step by step

1. **Read the failure email.** If it says survey emails can't be sent because no default community is selected, an active survey is trying to send.
2. **List the surveys.** In the Surveys app (or Setup, Surveys), list all surveys and note which are active.
3. **Identify the unused ones.** Sample surveys (CSAT, NPS) are the usual suspects when nobody at the firm set up surveys.
4. **Deactivate them.** Do not delete them.
5. **Confirm no process depends on them** before closing, as the ticket required. How this was checked is not recorded in the ticket.
6. **Watch for the email for one to two days.** It should stop.

## 9. Rollback

Reactivate the surveys. The failure emails return unless a default community is also selected for Surveys. Safe to do, but there is no reason to unless the firm decides to send surveys.

## 10. Known limits

- **Any survey activated later will start the same failure emails** until a default community is selected for Surveys.
- **The repo carries Survey objects, layouts and two sample survey email templates** (`Embed_a_Survey_Link`, `Embed_a_Survey_Question`). These are standard metadata, not evidence that surveys are in use.

## 11. Reuse for similar issues

**How to recognise one:** a recurring Salesforce system email about a feature nobody at the firm uses.

- **Look for sample or default content that shipped active.** Feature setups often switch on sample records (surveys, templates, processes) that run by themselves.
- **Turn off the source, not the symptom.** Configuring the missing setting the email asks for makes the unwanted thing work.
- **Deactivate rather than delete**, so the change can be reversed.
- **Check the ticket's guess.** Here the ticket suspected a Case-close automation; the cause was the surveys themselves.
