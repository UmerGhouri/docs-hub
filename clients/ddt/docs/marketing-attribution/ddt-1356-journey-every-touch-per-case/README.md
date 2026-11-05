# Let each case see every touch its person made


| | |
|---|---|
| **Ticket** | DDT-1356 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-20 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Attribution credit engine: which touches a case can see |
| **Related** | DDT-823, DDT-825, DDT-1286, DDT-1359 (parent DDT-1355) |
| **Code** | PR #759 in the ddt-salesforce repo |


---


## 1. Summary

The attribution credit engine could only weigh touches linked directly to a case (`Inbound_Contact__c.Opportunity__c`). Only 12% of new cases had any such touch, because most touches are linked to the person's Lead, Contact, Account or phone number instead. A new read-only resolver, `AttributionJourney`, finds every touch that belongs to a case's person inside the case's time window. It runs in shadow: the credit answer does not change yet, but the engine logs the journey size. In production, 95.8% of recent cases now have a journey touch.

## 2. What was reported

- 2026-10-16: step 1 of the attribution rebuild (parent DDT-1355). Measured in production: 89 of 753 cases created in the last 30 days had a touch linked to the case; 2,038 touch rows sat on converted Leads with no case link. Build a journey resolver that returns, for each case, every touch of its person within its window, in time order. Read-only; it must not write onto touch rows. Do not change who wins yet.

## 3. Scope

**In scope**

- `AttributionJourney` (new) and `AttributionJourneyTest`.
- `AttributionCreditRunner`: calls the resolver in shadow and logs the journey size.
- New field `Attribution_Config__mdt.Journey_Lookback_Days__c` (default 90).

**Out of scope**

- Changing who gets credit: step 4 (DDT-1359) does that. The selector still receives exactly the rows it did before.
- Back-filling `Opportunity__c` onto touch rows: a touch can belong to more than one case of the same person, and one lookup can hold only one, so the journey is computed, not stored.

## 4. How it works (background)

Touches (`Inbound_Contact__c`) link to a person in several ways: `Lead__c`, `Contact__c`, `Account__c`, `Opportunity__c`, or none, with a phone key. The firm's custom hire conversion (`HiredMatterConvertController`) does not use the standard Lead conversion: it creates the client Account and Matter directly, sets `Matter__c.Lead__c`, and marks the Lead "Converted" with `IsConverted` left false. So standard converted-Lead fields miss many of the links.

## 5. Root cause

The engine's query only matched `Opportunity__c = case`. Touches recorded before a case existed (calls, forms, the Lead's creation) stay on the Lead, Contact or Account and were invisible to it. In a 90-day sample, 52 of 74 Matters created from a Lead had `IsConverted = false`, which is why the custom link matters.

## 6. Solution

**Who is the person.** Touches matching any of these, de-duplicated by touch Id:

1. `Opportunity__c` = the case;
2. `Account__c` = the case's Account;
3. `Contact__c` = the Account's person contact;
4. `Lead__c` = any Lead that became this client: the standard converted-Account link plus `Matter__c.Lead__c` where the Matter's Account is the client;
5. the phone key from DDT-825, for calls that arrived before any Lead existed.

**Which window.** A touch counts if it occurred on or before the case's creation, and after the later of: creation minus `Journey_Lookback_Days__c` (default 90), or the creation of the same client's previous case. So a returning client's second case does not inherit the first case's journey. Touches after creation are service contact, not acquisition, and are never weighted.

**Cost and safety.** One query per key type for the whole batch (9 queries in all), each capped below the remaining row limit with a reserve, so an overflow throws a catchable error instead of an uncatchable limit. The runner skips the shadow unless enough queries, rows, CPU and heap remain, and swallows any shadow error as a warning, so the shadow cannot fail a credit stamp.

**Off switch, no deploy.** A `Triggers_Settings__c` row named `AttributionJourneyShadow` with Disabled checked.

## 7. Testing

- Apex tests cover each of the five person keys and both window edges, and fail when a key is removed.
- **Before deploy (offline replay of the rules over a production extract, 2026-10-19):** 673 of 719 recent cases (93.6%) had a journey touch, against 12.4%; 2,062 of 2,078 orphaned converted-Lead touches (99.2%) reached a journey; 0 touches after case creation; 0 touches carried over across 33 returning-client pairs.
- **After deploy (deployed code run read-only, 2026-10-20):** 688 of 718 recent cases (95.8%) had a journey touch; 0 touches after case creation; 0 carried over across 115 returning-client cases.
- The credit answer is unchanged by design (the selector gets the same rows). No before/after snapshot of `Attributed_Channel__c` was taken, and another attribution change (DDT-1286) went live the same day.

## 8. Step by step

1. Measure how many cases the engine can see touches for today, and how touches are linked to people.
2. Find every link a custom conversion leaves (here, the Matter's Lead lookup).
3. Write a resolver that collects touches by each person key, one bulk query per key, and filters by the case's window.
4. Run it in shadow from the engine, behind resource checks and an off switch, logging only the size.
5. Re-measure coverage from production after deploy.

## 9. Rollback

Create the `AttributionJourneyShadow` switch row with Disabled checked (no deploy), or redeploy `AttributionCreditRunner` from before PR #759. Nothing is written by the resolver, so there is no data to undo.

## 10. Known limits

- 26 of 1,390 journey touches (4 cases) were found by phone alone, and 356 phone keys are shared by more than one client Account (up to 27). Harmless in shadow, but to be weighed before the journey feeds credit.
- 16 orphaned touches reach no journey; 15 of them belong to a person with no case.

## 11. Reuse for similar issues

- Signs of the same problem: an engine that reads related records only through one direct lookup, while most records are linked through the person.
- Compute the relationship at read time when one record can belong to several parents; do not back-fill a single lookup.
- Ship big logic changes in shadow first, behind an off switch and resource guards, and measure coverage from production before letting them change answers.
