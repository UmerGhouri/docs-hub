# Registering a phone number for lead creation

| | |
|---|---|
| **Ticket** | DDT-1334 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-20 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Campaign, Phone Tracking Number, Apex), RingCentral |
| **Area** | Inbound calls creating PI Leads, by the number the caller dialled |
| **Related** | None recorded |
| **Code** | None. Two records created in Salesforce; no PR |

---

## 1. Summary

The firm started a new Meta ad campaign, "Meta PI Cold", with its own RingCentral tracking number. Calls to it reached intake, but no Salesforce Lead was created and no alert fired, because the number was not registered in Salesforce. The support team created a Campaign and a Phone Tracking Number record for it, the same way the main PI line is registered. A test call from a new phone number then created a PI Lead with the right source.

## 2. What was reported

On 2026-10-14 firm staff reported that calls to the new tracking number forwarded to intake correctly, but created no Lead, so nothing appeared in the lead alert channel. They asked for the number to create PI Leads with `Automated_Source__c` = "Meta PI Cold", mirroring the main PI line.

Testing then ran into two things that looked like failures and were not:

- **2026-10-16 and 2026-10-19.** Test calls created no Lead. Both test phones already had a record in Salesforce, and the system skips callers it already knows, so it does not make duplicates.
- **2026-10-20.** A call from a new phone created the PI Lead correctly, but no alert posted. The call was answered, and the alert channel only posts calls nobody picks up (missed or voicemail). The main PI line behaves the same way.

## 3. Scope

**In scope**

| Record | Object | Values |
|---|---|---|
| Meta PI Cold | `Campaign` | New |
| The campaign's tracking number | `Phone_Tracking_Number__c` | New: Campaign = Meta PI Cold, Type = RingCentral, Active, Practice Area = PI |

**Out of scope**

- **The advertising and tag-manager side of the campaign.** Handled by the requester.
- **Code changes.** None were needed; the existing call handler already reads this registry.

## 4. How it works (background)

```
Inbound call to a tracking number
  -> RingCentral notifies Salesforce
  -> Apex looks up the dialled number in Phone_Tracking_Number__c
       Active + Practice Area (Defense / PI / SOS)  -> Lead record type and queue
       otherwise                                    -> falls back to the answering extension
  -> skips the caller if their number already matches an Account, Contact or Lead
  -> skips if the tracking row has Do_not_Create_Lead__c checked
  -> creates the Lead
       Campaign__c          = the tracking row's Campaign
       Automated_Source__c  = the Campaign's Name
       Phone_Number_Name__c = the tracking row's name
```

So the Lead's source comes from the **Campaign's name**, not from a field on the tracking number. Registering a number means creating both records.

## 5. Root cause

The new tracking number had no `Phone_Tracking_Number__c` row, so Salesforce could not tie calls to it to a campaign or a practice area, and had no source to set. Exactly why the first calls created no Lead at all is not recorded in the ticket. The later tests showed one reason a call creates nothing: the caller is already known to Salesforce.

## 6. Solution

1. Created the Campaign "Meta PI Cold". Its name becomes `Automated_Source__c` on each new Lead.
2. Created the `Phone_Tracking_Number__c` row for the new number: Campaign = Meta PI Cold, Type = RingCentral, Active, Practice Area = PI.

Why both: the record type and queue come from the tracking row's Practice Area, and the source text comes from the Campaign's name. Either alone gives an incomplete Lead.

## 7. Testing

- **2026-10-20:** a test call from a phone number never seen before created a PI Lead with source "Meta PI Cold". The requester accepted the result.
- **Not covered:** the missed-call alert for this number. The test call was answered, so no alert was expected. The requester was told how to see it: call once more and let it ring out or go to voicemail. Whether that was done is not recorded.

## 8. Step by step

1. Look at how an existing number on the same practice area is registered (here, the main PI line) and copy its pattern.
2. Create a Campaign whose name is the exact text you want in `Automated_Source__c`.
3. Create a `Phone_Tracking_Number__c` row for the new number: link the Campaign, Type = RingCentral, Active checked, Practice Area set, `Do_not_Create_Lead__c` unchecked.
4. Test from a phone number that has **never** called the firm and has no record in Salesforce.
5. Check the new Lead's record type and `Automated_Source__c`.
6. To test the alert, let the test call ring out or go to voicemail.

## 9. Rollback

Set the tracking row to inactive, or delete it and the Campaign. Calls to the number then stop being tied to the campaign.

## 10. Known limits

- **Existing callers never create a new Lead.** A known caller dialling a new campaign number is not attributed to that campaign.
- **The alert only covers unanswered calls.** An answered call creates the Lead silently.
- **The match is on the last 10 digits of the dialled number.** Enter the number in a form that contains them.

## 11. Reuse for similar issues

**How to recognise one:** a new tracking number rings through to staff, but no Lead appears.

- **Check `Phone_Tracking_Number__c` for the number first.** No row means no campaign and no source.
- **The source comes from the Campaign name.** Name the Campaign exactly as the source should read.
- **Test from a brand-new phone number.** Testing from a known number creates nothing and looks like a failure.
- **Let the test call go unanswered** if you need to see the alert.
