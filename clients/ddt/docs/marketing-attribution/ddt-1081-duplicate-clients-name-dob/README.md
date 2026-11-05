# Duplicate clients that share no phone or email


| | |
|---|---|
| **Ticket** | DDT-1081 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-25 |
| **Priority** | Low |
| **Reported by** | Support team (follow-up from DDT-951), 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Person Accounts, data merge) |
| **Area** | Client identity: duplicate client records |
| **Related** | DDT-951, DDT-825, DDT-826, DDT-1120 |
| **Code** | No code change |


---


## 1. Summary

DDT-951 merged duplicate client records that shared a phone number or email. Five more pairs were left over: the same person by name and date of birth, but with different phone numbers and different emails, so the phone match key could not find them. The list was regenerated from production, a sixth exact duplicate was found, and all six pairs were merged after confirmation, oldest record kept, with a full backup first. Name plus date of birth was not adopted as a permanent match key; the driver's licence number already on the record was found to be the better third key.

## 2. What was reported

- 2026-09-22: five pairs of ever-hired client records, each the same person by name and date of birth, with different phones and emails, left unmerged when DDT-951 closed. Regenerate the list, have a person confirm each pair, merge oldest-surviving in a watched batch outside a DDT-826 re-score window, and record whether name plus date of birth should become a permanent match key.

## 3. Scope

**In scope**

- Ever-hired Person Accounts grouped by first name, last name and `PersonBirthdate`, minus pairs that share `Phone_Key__c` or `PersonEmail`.
- Merging confirmed pairs, and the match-key decision.

**Out of scope**

- The Person Account duplicate rule that let a new exact duplicate through at intake: filed as DDT-1120.
- Re-scoring attribution on the merged records: left to DDT-826.

## 4. How it works (background)

A client is a Person Account. `Phone_Key__c` (DDT-825) is a stored, normalised phone key used to recognise the same person across records. Two records for one person split that person's history and payments, and a later case on the newer record cannot be recognised as a returning client.

## 5. Root cause

Regenerated from production on 2026-09-22: five pairs matched the earlier count exactly. Each pair had two different phone numbers and two different mailboxes, so `Phone_Key__c` could not see them by construction.

A sixth group was different: an exact duplicate with the same phone key and the same email, created by intake on 2026-09-21, five days after the DDT-951 clean-up. Nothing blocked or flagged it, which shows the duplicate population regrows at intake.

## 6. Solution

1. **Confirmation by query.** Every client record carries `Drivers_License_Number__c`. Identical licence numbers confirmed four of the six pairs (two also shared an address; the exact duplicate also shared phone and email).
2. **Four merged on 2026-09-23,** one at a time, oldest record kept, with a full-field backup of both sides and every child record taken first and kept outside the repo. 11 Court Cases and converted Leads were re-pointed to the surviving records. Each duplicate's phone number was kept on the surviving record as the mobile number so both numbers still match.
3. **Two needed a person.** In those pairs one side had no usable licence number. On 2026-09-25 each was confirmed from the sequence and type of cases on both records, matching email handles, and the date of birth, then merged the same way. Both surviving records keep every phone and email that matched before.
4. **Match-key decision:** name plus date of birth stays "no", as DDT-825 concluded; five pairs in the whole ever-hired base do not change that. The licence number is the better third key and is already on the record.

## 7. Testing

Not applicable as code testing. Each merge was preceded by a full backup, and child records were re-pointed and counted. Whether any automation fired on the surviving records during the merges is not recorded in the ticket.

## 8. Step by step

1. Query ever-hired Person Accounts grouped by name and date of birth having more than one row.
2. Drop pairs that already share a phone key or email (those belong to the earlier clean-up).
3. Compare `Drivers_License_Number__c`; identical numbers confirm the pair.
4. For pairs without a usable licence number, have a person confirm from case history and contact details.
5. Back up both records and all child records to a file.
6. Merge one pair at a time, oldest record surviving, outside any re-score window.
7. Keep the duplicate's phone and email on the survivor so future matching still works.

## 9. Rollback

A merge cannot be undone in place. The backups taken before each merge allow the losing record and its child links to be recreated by hand.

## 10. Known limits

- The later Court Case on each merged account may now qualify as a returning client under the credit rule; nothing re-scores it until DDT-826 runs.
- New duplicates keep arriving until the duplicate rule is fixed (DDT-1120).

## 11. Reuse for similar issues

- Signs of the same problem: two client records with the same name and date of birth but different contact details.
- Look for a hard identifier already on the record (licence number) before inventing a fuzzy match key.
- Always back up both sides and all child records before a merge, and merge in small, watched batches.
- After a clean-up, check whether new duplicates are still being created at intake.
