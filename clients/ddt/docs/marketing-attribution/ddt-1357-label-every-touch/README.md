# Label every touch: kind, source system and a duplicate-proof key


| | |
|---|---|
| **Ticket** | DDT-1357 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-22 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, trigger, Custom Metadata, custom fields) |
| **Area** | Attribution touch log (`Inbound_Contact__c`) |
| **Related** | DDT-561, DDT-821, DDT-1358, DDT-1359, DDT-1362 (parent DDT-1355) |
| **Code** | PR #758, #764, #811 and #836 in the ddt-salesforce repo |


---


## 1. Summary

Every touch row now says what kind of evidence it is (`Touch_Kind__c`), which system or party produced it (`Source_System__c`), and carries a unique key (`Touch_Key__c`) so the same fact cannot be written twice. Before, rows had only a medium label (Phone Call, Web Form and so on), so a Google LSA lead and a Legal Growth Engine delivery looked the same, and a retry could duplicate a row. The writers now label rows as they are created, a one-time backfill labelled the existing rows, and a follow-up keys referral rows as soon as they are written.

## 2. What was reported

- 2026-10-16: step 2 of the attribution rebuild (parent DDT-1355). `Channel_Type__c` describes only the medium; `Vendor Feed` had never been used; the vendor appears only as free text in `Automated_Source__c`; and nothing stops duplicates. Add the three fields, set them in the existing writers, derive the source system from the existing dictionary, and backfill the existing rows touching only the three new fields.

## 3. Scope

**In scope**

- Deploy A (PR #758): `Touch_Kind__c` (restricted, 7 values: Interaction, Lead Delivery, Web Visit, Ad Click, Self-Reported, Referral, Campaign Assignment), `Source_System__c` (restricted, 14 values), the description of the existing `Touch_Key__c`, a new `Channel_Mapping__mdt.Source_System__c` column set on 83 dictionary rows, and field access on the existing attribution permission sets.
- Deploy B (PR #764): `TouchLabelService` (the one place kind, system and key are decided), the `Lead_Create_Inbound_Contact_Touch` and `Call_Session_Create_Inbound_Contact` flows, `WebsiteTouchService`, a before-insert default in `InboundContactTrigger`, and the one-time `InboundContactTouchBackfillBatch`.
- Follow-up (PR #811): referral rows get their key when written. Test fix (PR #836).

**Out of scope**

- `Channel_Type__c` stays as the medium.
- Who gets credit: nothing here re-scores a record.
- No new record-triggered flow on Lead; only the existing one changed.

## 4. How it works (background)

Touch rows are written by a Lead-creation flow, a call-session flow, the website endpoint (DDT-821) and the referral panel. Lead sources arrive as tokens in `Automated_Source__c`, and `Channel_Mapping__mdt` is the dictionary that maps tokens to channels, with a named owner and a weekly unmapped-token digest (DDT-561).

## 5. Root cause

Not a fault but a missing structure: the touch log had no field for the kind of evidence or the delivering party, and no unique key. Measured on 2026-10-16 over 30 days: LSA, Legal Growth Engine, FindLaw and Meta leads were labelled only "Phone Call" or "Other".

## 6. Solution

- **Kinds and systems.** The source system comes from the dictionary through the new column; unmapped tokens get `Other` and show up in the existing digest. A Lead whose system is a third party (Google LSA, Legal Growth Engine, FindLaw, Nolo, Martindale-Avvo, LegalMatch, Meta) gets `Lead Delivery`. Calls are `Interaction` from RingCentral. Website rows are `Web Visit`, or `Ad Click` when a click id is present.
- **Keys.** Deterministic formats such as `CALL:` plus the call session, `LEAD:` plus the Lead, `WEB:` plus Lead, time and source, and `REF:` for referrals. The website writer upserts on the key; the two flows insert and the unique index rejects a second write of the same fact. A second creation row for the same Lead gets a `:DUP:` suffix and is counted, never deleted.
- **Default on insert.** The trigger fills whichever label a writer left blank, never overwrites, and never fails a save.
- **Two deploys.** A restricted picklist value is not live inside the deploy that adds it, so fields shipped first (A) and writers second (B). The flows landed as Draft and were activated with FlowDefinition-only deploys.
- **Backfill.** A one-time batch updated only the three fields, oldest first, with the touch trigger handler bypassed so nothing re-scored. Rows it could not decide got `Interaction` / `Other` and were counted.
- **Referral keys (PR #811).** The referral panel and conversion wrote no key, so only the backfill keyed referral rows. An after-insert step now keys each new referral row in a separate update, so a refused key leaves the row for the backfill and never fails the user's save. An off switch (`ReferralTouchKeys`) needs no deploy. 11 old referral rows had no date; by decision (2026-10-22) they were dated from their creation time, then labelled and keyed.

## 7. Testing

- Production check-only gates before each deploy; a test asserts the two system value sets (field and dictionary) stay identical.
- **Backfill (2026-10-19, after a backup):** 27,926 of 27,937 rows labelled, 140 of 140 batches, 0 errors, no Lead or Court Case modified. (The ticket estimated 27,547 rows; the PR measured 27,935 on the day.) The 11 skipped were the undated referral rows.
- **Done-when, read from production on 2026-10-22:**
  1. 0 new touches with a blank kind on 2026-10-19, 10-20 and 10-21.
  2. 470 of 471 third-party Leads from the last 30 days have a Lead Delivery touch (LSA 177/177, Meta 93/93, FindLaw 35/35, Nolo 14/14, Legal Growth Engine 151/152). The one exception was created by hand and re-tagged later, so its creation touch records what was known then.
  3. A real re-run of the backfill keyed only 8 new hand-entered referral rows; row count unchanged at 31,902.
  4. Every touch keyed except the 11 undated rows; 420 rows were `Other`, with the top unmapped tokens listed.
  5. No change to `Attributed_Channel__c`: the backfill bypassed re-scoring.
- After PR #811 (2026-10-23): every touch in production has a kind and a key.
- PR #836 fixed a website test that expected a referral row's key to stay blank; it failed in production after #811 and passes in production, the full sandbox and the partial-copy sandbox.

## 8. Step by step

1. Add the new restricted fields and the dictionary column in one deploy.
2. Put all labelling rules in one service; call it from each existing writer.
3. Add a before-insert default that fills blanks and never fails a save.
4. Deploy the writers, then activate the flow versions separately.
5. Back up, dry-run, then run the backfill with re-scoring bypassed; re-run it to prove it changes nothing.
6. Check new rows for blanks three days running.

## 9. Rollback

- Referral keys: create the `ReferralTouchKeys` switch row (no deploy).
- Writers: redeploy the previous classes and reactivate the previous flow versions.
- Fields: nothing reads them for credit yet; a destructive deploy removes them. Keys already written stay valid.

## 10. Known limits

- 420 rows are `Other` (mostly dictionary rows that name no system, plus a few blank or unusual tokens). A handful of dictionary choices were left for the owner to confirm.
- The two flows cannot upsert, so they rely on the unique index rejecting a duplicate.

## 11. Reuse for similar issues

- Signs of the same problem: a log table where one label mixes medium, vendor and evidence type, and backfills or retries create duplicates.
- Separate "what kind", "from where" and "how" into distinct fields, derive "from where" from the one existing dictionary, and add a deterministic unique key.
- Set unique keys after insert in a separate update when a user's save must never fail on a race.
- Ship restricted picklist values one deploy before the code that writes them.
