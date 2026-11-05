# Clean up the phone tracking-number registry


| | |
|---|---|
| **Ticket** | DDT-816 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-14 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (records, custom metadata), RingCentral |
| **Area** | Attribution: which marketing channel a dialled number belongs to |
| **Related** | DDT-817, DDT-824, DDT-1082, DDT-1117, DDT-1135, DDT-1272 |
| **Code** | PR #380, #516 and #687 in the ddt-salesforce repo |


---


## 1. Summary

The attribution plan proposed a new phone-number registry object. One already existed: `Phone_Tracking_Number__c`, 48 rows, each linked to a Campaign whose name maps to a channel through `Channel_Mapping__mdt`. The work became registry clean-up instead: fix the vendor field, add two missing channel mapping records, deactivate an unused row, and decide which unmatched numbers belong in the registry at all. Measurement showed most unmatched numbers were staff or dialer lines that must not be registered. Follow-ups added the personal-injury website's number with its own Campaign, reactivated one live campaign number and reserved a row for a parked AI receptionist line.

## 2. What was reported

- 2026-08-24 (DDT-816): do not build `Phone_Registry__c`. Fill in the registry that exists: add the live numbers with no row, deactivate rows with no calls in 90 days, fix `Type__c`, and have a developer add two `Channel_Mapping__mdt` records so every campaign name resolves. Done when every `+`-prefixed inbound number in the last 90 days matches a registry row.
- 2026-09-10: a developer found that most missing numbers place outbound calls; firm leadership approved adding none of the remaining unidentified numbers and amended the done-when.
- 2026-09-25 to 2026-10-08: follow-up questions about the personal-injury website number, the AI receptionist line and one inactive campaign number.

## 3. Scope

**In scope**

- `Phone_Tracking_Number__c` records: `Type__c`, `Active__c`, new rows.
- `Channel_Mapping__mdt`: records for "Lake DUI Mailer" and "Video - Criminal (DUI) - Illinois" (PR #380), and one for the personal-injury website lead Campaign (PR #516).
- One new Campaign for the personal-injury website.

**Out of scope**

- A door-or-source flag on the registry. It waited on the door decision in DDT-824, which put the flag on `Channel_Mapping__mdt` instead.
- Writing the registry lookup onto each call (DDT-1082) and tying rows to ad campaigns by Id (DDT-1272).
- Changing call forwarding in RingCentral.

## 4. How it works (background)

A call's dialled number is looked up in `Phone_Tracking_Number__c`. The row's `Campaign__c` (labelled "Automated Source") names a Campaign; the Campaign name matches `Channel_Mapping__mdt.Source_Token__c`, which gives `Channel_Bucket__c`. `Practice_Area__c` holds Defense, PI or SOS, and an inactive row also skips practice-area routing (DDT-1135). When the ticket was written, 37 of the 39 campaign names on the registry already matched a mapping record.

## 5. Root cause

Not applicable as a single fault; this was data clean-up. Findings from production:

- `Type__c`: 27 of 48 rows said JustCall, a retired vendor whose picklist value is inactive; 20 were blank; one had a typo; RingCentral appeared zero times although it carried all the traffic.
- Over 90 days, inbound calls arrived on 104 distinct values in `Call_Session__c.Internal_Party_Number__c`; only 63 were dialable numbers. The rest were bare extensions and junk digits (2,112 calls, 12.5% of inbound), which are a call-sync matter (DDT-817), not registry rows.
- Of 17 numbers missing from the registry (30-day cut; 26 over 90 days), 12 placed outbound calls. Tracking numbers receive calls; they do not place them. One placed 1,804 outbound calls, three within the same second (a dialer); another was a staff member's direct line. Registering them would credit every inbound call to a channel.
- Seven more unmatched numbers sat in a RingCentral direct-dial block and took one or two stray calls each.

## 6. Solution

- **`Type__c` cleaned up** in list views, rows carrying RingCentral traffic set to RingCentral (approved 2026-09-10).
- **Two mapping records deployed** (PR #380) so all 39 campaign names resolve.
- **No staff or dialer lines added.** Of the 5 inbound-only unidentified numbers, firm leadership approved adding none: none was a call-tracking vendor number, together they took 5 inbound calls in 30 days (about 0.09% of volume), and a wrong registry row silently credits calls to the wrong channel forever. One was the AI receptionist pilot line, which carried only test calls.
- **Done-when amended:** exclude numbers with outbound traffic in 90 days and numbers in the direct-dial block.
- **One active row with no Campaign, practice area or type, and zero calls in 90 days,** was deactivated.
- **Personal-injury website number** (PR #516): it had no suitable Campaign; the defense website Campaign would merge two sites, and the Mainline PI Campaign is a door that never wins credit alone. A new Website Direct Campaign, a `Channel_Mapping__mdt` row (Website / Direct Forms, not a door) and a registry row (PI, RingCentral, active) were added, in that order, because the registry resolves through the mapping.
- **2026-10-12** (PR #687, data with a deploy note): an inactive number still printed on a live paid-search landing page was reactivated, so its calls route by practice area instead of landing unclassified; and the AI receptionist line got a reserved inactive row with no Campaign, so it stops showing as unmatched without changing how its calls are handled.

## 7. Testing

- Acceptance query re-run on 2026-09-28: 6 unmatched numbers (22 calls in 90 days), down from 7, then 5 once the AI receptionist line was reserved; the other 5 are unidentified or former staff lines with 11 calls between them.
- Before the follow-up, firm leadership pointed out that the ticket was Done while its acceptance query did not return zero; the re-run above answered that.
- After the reactivation, calls to that number route by the dialled number instead of being flagged unclassified (2 of its 3 Leads since mid-September had been flagged).

## 8. Step by step

1. Check for an existing object before building a new registry.
2. For each unmatched inbound number, count its outbound calls; any outbound traffic means a staff or dialer line, not a tracking number.
3. Only register a number when someone can name where it is published and its Campaign.
4. Create the `Channel_Mapping__mdt` row (deployed as metadata) before the Campaign and registry row that depend on it.
5. Fix vendor and active flags in list view; deactivate rows with no calls and no Campaign.
6. Re-run the acceptance query and record the remaining unmatched numbers and why.

## 9. Rollback

Registry edits are record changes and can be reversed in list view; the mapping records can be removed with a destructive deploy. A rollback was not recorded in the ticket.

## 10. Known limits

- The registry is matched by Campaign name, so renaming a Campaign silently breaks the chain (DDT-1272 tracks matching by Id instead).
- A handful of unidentified numbers remain unmatched by decision.
- Bare-extension calls cannot be matched to any registry row (see DDT-817).

## 11. Reuse for similar issues

- "Add every unmatched number" is the wrong rule: test for outbound traffic first.
- A wrong mapping is worse than a missing one; it looks correct forever.
- A Done ticket whose acceptance query no longer passes needs re-running at close, with the remaining exceptions written down.
- An inactive registry row can still take calls; check call volume before treating a row as retired.
