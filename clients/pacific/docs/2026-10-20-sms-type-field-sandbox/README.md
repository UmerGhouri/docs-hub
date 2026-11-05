# SMS Type field (welcome vs after-hours)

| | |
|---|---|
| **Status** | Sandbox only (2026-10-20). Production waiting for the client's OK |
| **Reported by** | Client admin, 2026-10-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, Make |
| **Change log rows** | 49–54 |

---

## 1. Summary

The checkbox was labelled "Welcome SMS Sent", so after-hours leads looked as if they had received the welcome text. In the sandbox we:
- renamed the checkbox label to **SMS Sent** (API name unchanged); and
- added a picklist **SMS Type** with the values **Welcome SMS** and **After Hours SMS**, set by Make.

## 2. Client report

> "This one shows Welcome SMS Sent after hours, this should be after hours sms not welcome sms."

_Screenshot in the private copy: 01-client-report.png (Client report)._

## 3. Scope

**In scope (sandbox):** `Lead.Welcome_SMS_Sent__c` (label only), new `Lead.First_SMS_Type__c`, `Lead_record_page`, `[UAT] Welcome Sms` step #52, `[UAT] Offbusiness Hours SMS` step #50.

**Out of scope:** production, until approved; the welcome-text logic itself.

## 4. Solution

| Item | Change |
|---|---|
| `Welcome_SMS_Sent__c` | Label → **SMS Sent**. Help text: ticks when the first text (welcome or after-hours) goes out. **API name unchanged**, so flows, outbound messages and Make keep working |
| `First_SMS_Type__c` (new) | Restricted picklist: `Welcome SMS`, `After Hours SMS`. Read-only for User and Super User; editable for Admin. Label "SMS Type" (first created as "First SMS Type") |
| Lead page | SMS Type placed directly below SMS Sent (AI Call section) |
| `[UAT] Welcome Sms` #52 | Also sets `first_SMS_Type__c = "Welcome SMS"` |
| `[UAT] Offbusiness Hours SMS` #50 | Also sets `first_SMS_Type__c = "After Hours SMS"` |

_Screenshot in the private copy: 02-sms-sent-and-type-fields.png (SMS Sent and SMS Type on the lead)._

## 5. Go-live steps (when approved)
1. Deploy the field (+ field-level security) and the label change to production.
2. Add SMS Type to the production Lead page under SMS Sent.
3. Production Make: add `first_SMS_Type__c` to `New Lead >>> Welcome Sms` #52 and `Offbusiness Hours SMS v2` #50.
4. Optional: back-fill existing texted leads from the Make run history.

> **Order matters:** deploy the field **before** adding it to Make, or the Make update step fails with "No such column".

## 6. Rollback
Remove `first_SMS_Type__c` from the two Make steps, then delete the field; set the label back to "Welcome SMS Sent".

## 7. Reuse for similar issues
- **Rename the label, never the API name,** when other automations use the field.
- **Picklists written by Make should be restricted,** so a typo in Make fails visibly instead of creating a new value.
