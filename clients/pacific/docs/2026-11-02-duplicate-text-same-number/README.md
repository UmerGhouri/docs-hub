# Duplicate first text to the same phone number

| | |
|---|---|
| **Status** | Resolved, live in production since 2026-11-02 |
| **Reported by** | Client admin, 2026-11-02 |
| **Resolved by** | Support team |
| **Systems** | Make, Salesforce |
| **Area** | First text to new web leads (welcome / after-hours SMS) |
| **Change log rows** | 82–85 in `Pacific-Change-Log.xlsx` |

---

## 1. Summary

A lead filled out two website forms 1½ minutes apart and received the same text twice. Each form creates its own Lead record, and the "don't text twice" check only looked at the current record, not the phone number.

We added a check before the text is sent: if another lead with the same phone number already got a first text in the last 24 hours, skip the text. The new lead still gets its status, assignment and follow-up AI calls.

## 2. Client report

> "Why did two SMS messages go out back to back to the same lead with the same information?"

_Screenshot in the private copy: 01-client-report.png (Client report: two identical texts one minute apart)._

## 3. Scope

**In scope**
- The two Make scenarios that send the **first** text to a new web lead:
  - `New Lead >>> Welcome Sms`: business hours.
  - `Offbusiness Hours SMS v2`: after hours and weekends.
- Their sandbox copies: `[UAT] Welcome Sms` and `[UAT] Offbusiness Hours SMS`.

**Out of scope**
- Mia's replies (`Mia Text Agent [production]`). Those are answers to the lead, not first texts.
- Merging duplicate leads. The second lead is still created; we only stop the second text.
- AI calls. Both leads still get their day 3–21 calls.

## 4. How the flow works (background)

```
Web form -> new Lead in Salesforce
  -> Flow "Lead_After_create_updated" (v15)
       business hours -> outbound message Make_Welcome_SMS -> Make "New Lead >>> Welcome Sms"
       off-hours      -> outbound message Make_SMS_v2      -> Make "Offbusiness Hours SMS v2"
  -> Make: set Status = No Contact, Not Contacted = AI 1, Welcome AI Call = true
           assign lead (Assign Leads hook)
           send text from the client's main RingCentral line
           tick Welcome_SMS_Sent__c ("SMS Sent")
```

Before this fix, the text step only checked `Do_Not_Text__c` and `Welcome_SMS_Sent__c` **on the same lead**.

## 5. Root cause

- Two form submissions create two Lead records with the same `MobilePhone`.
- Each record runs its own Make scenario, and on its own record `Welcome_SMS_Sent__c` is still false.
- So both pass the "Can text" filter and both send.

How often: in the 30 days before the fix, this was the only real case out of 236 web leads.

## 6. Solution

### 6.1 Matching the phone number

Phones are saved in different formats, e.g. `(555) 123-4567` and `555-123-4567`. We match on the existing formula field `Lead.Normalized_Phone__c`, which holds the last 10 digits of `MobilePhone`.

In Make, the parsed number `{{e164}}` (e.g. `+15551234567`) becomes 10 digits with `replace(e164; "+1"; emptystring)`.

### 6.2 The lookup (new module #60, both scenarios)

Module: **Salesforce > Make an API Call** (`salesforce:makeApiCall`), on the same Salesforce connection as the scenario.

- URL: `/v59.0/query`
- Method: `GET`
- Query string `q`:

```sql
SELECT COUNT() FROM Lead
WHERE Normalized_Phone__c = '{{replace(10.e164; "+1"; emptystring)}}'
  AND Welcome_SMS_Sent__c = true
  AND Id != '{{9.lead_id}}'
  AND CreatedDate >= {{formatDate(addHours(now; -24); "YYYY-MM-DDTHH:mm:ss"; "UTC")}}Z
```

Result: `{{60.body.totalSize}}`: 0 means no recent text to this number, 1 or more means a duplicate.

> **Why an API call and not "Search Records"?** "Search Records" returns **zero bundles** when nothing matches, and that stops the rest of the route, so normal leads would never get their text. `SELECT COUNT()` through "Make an API Call" always returns exactly one bundle with `totalSize`.

### 6.3 Business-hours scenario (`New Lead >>> Welcome Sms`)

The status update (#27) and assignment (#51) already run **before** the text, so only the text step changes.

| Step | Change |
|---|---|
| #60 | New lookup, placed after #51 (assignment) and before #40 (send SMS) |
| #40 "Can text" filter | Added condition `{{60.body.totalSize}}` = 0 (number) |

Order after the change: `Webhook -> Parse -> Variables -> Parse phone -> #27 Update lead -> #51 Assign -> #60 Lookup -> #40 Send SMS -> #52 Tick SMS Sent`

### 6.4 After-hours scenario (`Offbusiness Hours SMS v2`)

Here the status update and assignment come **after** the text. Adding the condition to the existing filter would also skip the status and assignment, so a router was added.

| Step | Change |
|---|---|
| #60 | New lookup, after #17 (Get lead) |
| #61 | New router |
| Route 1 (first module #2) | Existing path unchanged: GPT text -> send SMS -> update lead (+ SMS Sent) -> assign. Filter "Can text" + `totalSize = 0` |
| Route 2 (new #62, #63) | **Duplicate only:** same conditions but `totalSize > 0`. Updates the lead (No Contact / AI 1 / Welcome AI Call, **no** SMS Sent) and calls the Assign Leads hook. No text. |

Route 2 repeats the "Can text" conditions (not Do Not Text, name doesn't contain "test", SMS Sent not ticked). Without them, leads that are skipped today for other reasons would start being updated.

## 7. Testing

**Sandbox, 2026-11-02, business hours.** Two leads with the same mobile number, created about 1 minute apart, without "test" in the name.

| Run | Operations | Result |
|---|---|---|
| Lead 1 | 10 | Text sent, SMS Sent ticked, No Contact / AI 1, assigned |
| Lead 2 | 8 | **No text**, SMS Sent not ticked, No Contact / AI 1, assigned |

The 2 fewer operations are the skipped "send SMS" and "tick SMS Sent" steps.

The query was also run against production data, and it finds the client-reported lead's first record when checked from the second one.

**Not yet covered:** route 2 of the after-hours scenario. The test ran in business hours. Check the first real after-hours duplicate in Make history.

> Screenshots to add: `02-sandbox-history.png` (the 10 vs 8 operations runs), `03-welcome-sms-after.png`, `04-offhours-router.png`.

## 8. Rollout

1. Built and tested in the sandbox scenarios.
2. Same change applied to production on the client's Salesforce and RingCentral connections. The main line and the production Assign Leads hook unchanged.
3. Both scenarios confirmed active after the update.
4. Logged in `Pacific-Change-Log.xlsx` (rows 82–85).

## 9. Rollback

- **Welcome Sms:** delete #60 and remove the `totalSize` condition from the #40 filter.
- **Offbusiness v2:** delete #60, #61 (router) and route 2 (#62, #63), and remove the `totalSize` condition from the #2 filter.

## 10. Known limits

- **Forms seconds apart:** if two leads arrive within a few seconds, the first may not have SMS Sent ticked yet, so both can still get a text. Minutes apart (the real case) is caught.
- **US numbers only:** the `+1` strip assumes US numbers. A non-US number won't match, so it behaves as before (text sent).
- **24-hour window** is based on the other lead's `CreatedDate`, which is close to when its first text was sent.
- **Cost:** +1 Make operation per new lead (about 240 a month).

## 11. Reuse for similar issues

- **"Same person, two records" problems:** match on `Normalized_Phone__c`, not the raw phone fields.
- **Checking whether something exists in Make without stopping the flow:** use `SELECT COUNT()` through "Make an API Call" and filter on `body.totalSize`.
- **Skipping one step without skipping the steps after it:** if later steps must still run, use a router with a second route rather than a filter on the shared path.
- **Always change the `[UAT]` scenario first** and test with your own number before touching production.
