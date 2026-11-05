# Welcome text replaces the AI welcome call

| | |
|---|---|
| **Status** | Live since 2026-10-19 |
| **Requested by** | Client admin, 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (flow, field, outbound messages, Lead page), Make |
| **Area** | First contact with new web leads |
| **Change log rows** | 5–6, 8–12, 16–19, 23–38, 40, 42–48 |

---

## 1. Summary

New web leads used to get an **AI welcome call** within minutes (or the next morning if they came in after hours). Pacific wanted a **text first** instead, with the day 3 / 7 / 14 / 21 AI calls kept.

Now every new web lead gets one first text:
- **Business hours:** a fixed welcome text asking if they're free for a quick call with a loan specialist.
- **After hours and weekends:** a text from Mia (written by OpenAI from approved templates) confirming the inquiry was received and the team will reach out during business hours.

The lead is then moved to **No Contact / AI 1**, assigned to a loan officer, and marked **SMS Sent** so it can never get the first text twice.

## 2. Client request

> "Assign the lead to a loan officer after the SMS is sent" / "Can we first build the new automation and test it in a sandbox before implementing?"

_Screenshot in the private copy: 01-client-request.png (Client request)._

## 3. Scope

**In scope**
- Flow `Lead_After_create_updated`: production v12 (welcome call) → **v15** (welcome text).
- New field `Lead.Welcome_SMS_Sent__c` (label **SMS Sent** in the sandbox, "Welcome SMS Sent" in production).
- Outbound messages `Make_Welcome_SMS`, `Make_SMS_v2` (new).
- Make `New Lead >>> Welcome Sms` and the new `Offbusiness Hours SMS v2`.
- Make `Working hours automation`: turned **off**.
- Lead page (`Lead_record_page`): "Welcome AI Call" replaced by "Welcome SMS Sent".

**Out of scope**
- Day 3–21 AI calls (unchanged; they still run from AI 1).
- The old `Offbusiness Hours SMS` and outbound message `Make_SMS`: left untouched as a rollback path.

## 4. How it worked before

```
New web lead -> Lead_After_create_updated v12
   business hours -> Make_call -> "New Lead >>> Welcome Call" -> Retell call -> Post Call Analysis -> AI 1, assign
   off hours      -> Make_SMS  -> "Offbusiness Hours SMS" (text) ... next morning "Working hours automation"
                                   (polls Status = New AND Welcome_AI_Call__c != true every 15 min) -> AI call
```
Assignment happened in **Post Call Analysis**, so it depended on the AI call.

## 5. Problems found while building it

| Problem | Cause | Fix |
|---|---|---|
| **Same lead texted three times** (Oct 12) | `Lead_Automation`'s "Change to new" step resets `Welcome_AI_Call__c` to false whenever Status goes back to New, so the old guard let the welcome path fire again | v15 fires the welcome paths **only when the lead is new** (`$Record__Prior.Id` is null) **or `Web__c` just changed**, with "only when record is changed to meet criteria" **off**; plus a new lock field `Welcome_SMS_Sent__c` that **nothing ever resets** |
| No assignment without the AI call | Assignment lived in Post Call Analysis | The text scenarios call the Assign Leads webhook themselves |
| Off-hours leads stayed New and were AI-called next morning | `Working hours automation` | Turned **off** |

_Screenshot in the private copy: 02-same-lead-texted-three-times.png (Same lead texted three times)._

## 6. Solution

### 6.1 Salesforce
- **Field** `Welcome_SMS_Sent__c` (checkbox, default false, never reset). Admin can edit; User and Super User profiles read-only.
- **Outbound messages** `Make_Welcome_SMS` and `Make_SMS_v2` include `Welcome_SMS_Sent__c` and `Do_Not_Text__c`.
- **Flow v15** (`Lead_After_create_updated`): both welcome outcomes need `Web = Web`, the new-or-Web-changed rule, `Welcome_AI_Call__c = false` and `Welcome_SMS_Sent__c = false`. Business hours → `Make_Welcome_SMS`; off hours → `Make_SMS_v2`.

### 6.2 Make: `New Lead >>> Welcome Sms`
`Webhook → Respond (SOAP ack) → Parse XML → Set variables → Parse phone → #27 Update lead (Status = No Contact, Not Contacted = AI 1, Welcome AI Call = true) → #51 HTTP to Assign Leads → #40 RingCentral send SMS from the client's main RingCentral line (filter: not Do Not Text, not already SMS Sent) → #52 tick Welcome SMS Sent`

Error handler on #40: if RingCentral rejects the number, set `Invalid_Phone_Number__c = true`, then Break with retry.

### 6.3 Make: `Offbusiness Hours SMS v2`
`Webhook → Respond → Parse XML → Set variables → Parse phone → #17 Get lead → #2 OpenAI "Generate SMS" (filter: not Do Not Text, name doesn't contain "test", not SMS Sent) → #24 send SMS from the main line → #50 Update lead (No Contact / AI 1 / Welcome AI Call / Welcome SMS Sent) → #51 Assign Leads`

The OpenAI prompt holds four approved templates (before 9 AM, weekday after 5:30 PM, Saturday after 1 PM / Sunday, business hours) and picks one from the current time.

> The Make "Respond" module returns the SOAP `<Ack>true</Ack>` Salesforce outbound messages expect. Without it, Salesforce retries the message.

### 6.4 The "stay New" detour (same day)
On the morning of 2026-10-19, the client admin asked for texted leads to **stay New** so the team would call them first; #27 / #50 stopped setting the status and a "New - Welcome Text Sent" list view was added. That afternoon the request was reversed it, so the automation can't be forgotten: #27 / #50 set **No Contact / AI 1** again and the list view was deleted.

_Screenshot in the private copy: 03-client-question-status.png (Client question about New vs No Contact)._

### 6.5 Data clean-up
`Welcome_SMS_Sent__c` was ticked on the 21 weekend leads (Oct 17–19) that had already received the old after-hours text, verified 1-to-1 against Make run times. A Do Not Text lead was excluded. Backup: `weekend_leads_backup_2026-10-19.csv`.

## 7. Testing

| Where | Test | Result |
|---|---|---|
| Sandbox | Off-hours lead → one Mia text | Pass |
| Sandbox | Business-hours lead → one welcome text | Pass |
| Sandbox | Lead set back to New → no second text | Pass |
| Sandbox | Assigned from the client admin to a logged-in loan officer | Pass |
| Production (test flow limited to the tester's own number, then reverted) | Test-number lead gets the text and is assigned | Pass |
| Production go-live, 9:07 AM PT | Business-hours web lead | Text sent (9 ops), assigned to a loan officer in 6 s, SMS task logged 20 s later |

## 8. Rollout (2026-10-19)
1. Deploy v15 **inactive**, plus the field, outbound messages and Lead page.
2. Update both Make scenarios (they stay idle until the flow fires).
3. Turn **off** `Working hours automation`.
4. **Activate v15.**
5. Tick SMS Sent on the weekend leads.

## 9. Rollback
Activate **v12** and turn `Working hours automation` back **on**. Expect a burst of AI calls to leads still at New when it starts.

## 10. Known limits
- **Assignment only picks logged-in loan officers** (Assign Leads checks AuthSession). Leads that arrive at night stay with the client admin until the first call; assignment then happens from Post Call Analysis.
- **After-hours wording uses Make's `now`** (UTC), not Pacific time, to choose the template. Not fixed yet.
- **Saturday 9–1 is treated as business hours in the prompt,** but the flow's business-hours check decides which scenario runs.

## 11. Reuse for similar issues
- **Need a "do it once" guard?** Use a checkbox that no other automation resets. Check `Lead_Automation` before relying on an existing field; it resets several on "back to New".
- **Flow should fire only on create (or one field change)?** Use `$Record__Prior.Id` is null, OR `ISCHANGED(field)`, with "only when record is changed to meet criteria" off.
- **Changing a live path?** Build a v2 scenario and outbound message next to the old ones, so rollback is just reactivating the old flow version.
