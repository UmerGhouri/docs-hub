# Log a touch row for every new Lead


| | |
|---|---|
| **Ticket** | DDT-814 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-14 |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex tests), RingCentral (lead source) |
| **Area** | Attribution: Lead intake and the Inbound Contact touch log |
| **Related** | DDT-811, DDT-812, DDT-813, DDT-815, DDT-818, DDT-823, DDT-948, DDT-1027 |
| **Code** | PR #378 and #382 in the ddt-salesforce repo (activation state recorded in PR #381 and #387) |


---


## 1. Summary

A new after-save flow, `Lead_Create_Inbound_Contact_Touch`, writes one `Inbound_Contact__c` touch row for every new Lead, carrying phone, email, source, Google click id, channel and event time. It runs in system context because about four in five Leads are created by integration users, and it has a fault path so a logging failure can never block a Lead. Two problems appeared in production: the flow landed as Draft after a green deploy, and its channel derivation classified every phone Lead as Other. Both were fixed, 12 mislabelled rows were corrected, and the two attribution recompute flows were reactivated after a sandbox test. Complete on 2026-09-15.

## 2. What was reported

- 2026-08-24 (DDT-814): add a record-triggered flow on Lead (after save, on create) that creates one Inbound Contact with the listed field mapping. Build it in the sandbox first, only after DDT-811 and DDT-812 are live in production and DDT-813's fields exist. Run it in system context and test the guest-user path. Done when, in the sandbox, a returning client's second Lead with a captured referral converts and the Client's `Referral_Source__c` holds that referrer.
- 2026-09-10 (scope addition, from DDT-948): reactivating `Attribution_Recompute_On_Referral_Change_Lead` and `Attribution_Recompute_On_Referral_Change_Court_Case`, with a sandbox watch-it-run test first, is part of this ticket.

## 3. Scope

**In scope**

- New flow `Lead_Create_Inbound_Contact_Touch` and its `FlowDefinition`.
- `ReferralSourceServiceTest`, which the new flow broke.
- Correction of 12 rows written with the wrong channel.
- Reactivation of the two recompute flows.

**Out of scope**

- A conversion-time flow. DDT-815 confirmed conversion linking already lives in Apex, and the project rule is no new record-triggered flow on Lead that writes Inbound Contact during conversion.
- Calls, handled by the twin flow in DDT-818.

## 4. How it works (background)

Three Apex readers find "the Inbound Contact for this Lead" by taking the newest row by `Occurred_On__c`. Before DDT-811, a touch row stamped with a time would outrank every referral row, blanking the referral panel and carrying the wrong row into conversion. DDT-811 made all four read sites filter on `Referrer_Type__c != null`; touch rows carry no referrer type, so they never compete. That fix was confirmed in production, not in the PR, before this flow went on.

Volume measured on 2026-09-14 (last 30 days): 1,061 of 1,493 Leads came from the RingCentral site guest user and 167 from the website intake API user, so 82% of intake is not a licensed human.

## 5. Root cause

Not applicable as a fault: new automation. Two defects were found after go-live:

- **Draft after a green deploy.** The deploy reported 2/2 components and no errors, but the flow version was Draft and inactive. Read back from the flow version view; activated separately through its `FlowDefinition`. The same happened again with the fix deploy.
- **Every Lead classified Other.** `$Record.CreatedBy.Name` resolves to an empty string inside a record-triggered flow. This was proven in the sandbox by parking the resolved value in a text field: the org showed the user's name, the flow saw an empty string. So the Phone Call branch could never fire. The original sandbox test had used a web Lead, which returned on the first branch and never reached the comparison.

## 6. Solution

- **Mapping:** `Captured_Phone__c`, `Captured_Email__c`, `Lead__c`, `Automated_Source__c`, `GCLID__c`, `Channel_Type__c` (Phone Call, Web Form or Other), `Occurred_On__c` = flow time, `Linked_Party_Type__c = Lead`.
- **System context** (`runInMode` system), because in user context the create fails for integration-user Leads while the flow still reports success.
- **Fault connector on the create**, because a failed DML in an after-save flow rolls back the Lead that triggered it. The trade-off was written into the flow description: a handled fault ends quietly and sends no error email, so a logging outage is silent. Detection is DDT-823's heartbeat, not extra logging inside the flow.
- **PR #382, channel fix:** a Get Records on User by `$Record.CreatedById`, and the formula reads the name from the queried record. Still matched by name, not Id, because user Ids differ between orgs. The lookup has its own fault connector, and a missing user yields Other.
- **Data correction:** the 12 guest-user rows written as Other between activation and the fix were updated from an explicit Id list, with a guard that refused to write unless all 12 were still as measured. 12 updated, 0 failed; the 12 parent Leads were snapshotted before and after and no field moved.
- **Recompute flows:** both were watched in the sandbox re-stamping an already-attributed Lead and Court Case when their referral type changed, with a negative control (clearing the type did not re-run the resolver). Then reactivated in production. The gap query (stamped, carrying a referral type, basis None) returned 0 Leads and 0 Opportunities, so nothing needed re-scoring.
- **Git:** PR #387 recorded the active versions, because git held version 1 (the broken derivation) and had no `FlowDefinition` at all for the two recompute flows.

## 7. Testing

- Sandbox done-when: a touch row and a referral row coexisted on the same Lead, and on conversion the referral, not the newer touch, landed on the Client's referral fields.
- Fault path by negative control: an always-fail validation rule blocked the touch row, and the Lead still saved. The rule was removed afterwards.
- Activating the flow broke 3 of 18 tests in `ReferralSourceServiceTest`, because Lead inserts in tests now create a touch row. Fixed so the count assertion counts the referral row only and still fails under the DDT-811 bug; 18 of 18, later 19 of 19, with the flow active.
- An early validation used a single unrelated test, which could not have caught this; the gate was re-run with the full local suite.
- Production, 2026-09-15: 39 of 39 guest-user rows since the fix were Phone Call; 0 rows whose channel contradicts their creator; 66 Leads since activation and 67 rows (one Lead was deleted and its row survived, handed to DDT-823); 11 referral rows untouched.

## 8. Step by step

1. Confirm in production that the referral readers filter out touch rows and that change detection (DDT-812) is live.
2. Build the flow in the sandbox: after save, on create, system context, one Create Records with a fault connector.
3. Derive the creator's name with a Get Records on User, not from `$Record.CreatedBy`.
4. Test each channel branch, the fault path (temporary failing validation rule) and the referral survival on conversion.
5. Deploy, then read the active version back from the flow version view; activate through `FlowDefinition` if it landed as Draft.
6. Read field values on the first production rows, not just the count.
7. Commit the production activation state to git.

## 9. Rollback

Deactivate `Lead_Create_Inbound_Contact_Touch` (set its `FlowDefinition` active version to 0). Lead creation does not depend on it. Rows already written stay in the touch log. The specific rollback was not recorded beyond this.

## 10. Known limits

- The Phone Call branch matches the guest user's display name. Renaming that user silently reclassifies phone Leads to Other with no error.
- The guest user does not exist in the sandbox, so that branch was proven only by pointing the literal at a sandbox user, then confirmed on production rows.
- A failed touch write is swallowed by design and only the daily heartbeat (DDT-823) would notice.
- The 2026-09-14 merge closed the ticket automatically while work was open; this recurred on other tickets and became DDT-1027.

## 11. Reuse for similar issues

- In record-triggered flows, do not trust cross-object references on `$Record` (such as `CreatedBy.Name`); query the related record.
- Every Create in an after-save flow needs a fault connector if the triggering record must survive.
- After any flow deploy, read the active version back; a green deploy can leave the flow as Draft.
- A classification bug that writes a valid picklist value looks healthy to every row count; check derived values on real rows.
- Put a guard on data corrections: an explicit Id list plus "refuse unless all rows still match what was measured".
