# Merging duplicate leads, accounts and contacts

| | |
|---|---|
| **Ticket** | DDT-275, DDT-473, DDT-506, DDT-567, DDT-767, DDT-1328, DDT-1412 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-26 (last of the seven; each was a one-off data fix in production) |
| **Priority** | Medium |
| **Reported by** | Intake team and Firm staff, 2026-07-07 to 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lead, Person Account, Contact) |
| **Area** | Data cleanup: duplicate prospect and client records |
| **Related** | DDT-1120 (stops lead conversions creating duplicate clients) |
| **Code** | Not applicable: data-only fixes, no code or PR |

---

## 1. Summary

Staff kept finding the same person stored two to five times: as duplicate Leads, duplicate Person Accounts, or a duplicate Contact. Each request was handled as a merge in production by an admin, after confirming with the requester which record should survive. Two of the seven tickets named the wrong surviving record, and one "delete this duplicate" request was solved by relinking the record instead, because deleting it would have destroyed call history.

## 2. What was reported

- **DDT-275 (2026-07-07):** two duplicate Person Accounts for one client. Done 2026-07-08.
- **DDT-473 (2026-07-20):** five Leads for one person. Four were `New`; one was `Disqualified` with a lost reason of "Cannot Afford It". Done 2026-07-21.
- **DDT-506 (2026-07-24):** an intake user got "You don't have permission to merge this record" when merging two duplicate Leads. Done 2026-07-29.
- **DDT-567 (2026-07-28):** two Leads for one person; the requester wanted to keep the SOS lead, not the Defense lead. Done 2026-07-29.
- **DDT-767 (2026-08-18):** four Leads sharing one email address. Three were `Disqualified`, one `New`, and the phone numbers did not all match. Done 2026-08-19.
- **DDT-1328 (2026-10-13):** a staff member created a new Person Account for someone who already had one from 2021. Done 2026-10-14.
- **DDT-1412 (2026-10-23):** a staff member could not delete a duplicate Contact ("no permission") and asked the team to delete or merge it. Done 2026-10-26.

Why it matters: duplicates split activity history across records, let two reps work the same person, and make reports count one person several times.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Lead` | Standard object | Duplicates merged into one surviving Lead (DDT-473, DDT-506, DDT-567, DDT-767) |
| `Account` (Person Account) | Standard object | Duplicates merged into one surviving account (DDT-275, DDT-1328) |
| `Contact` | Standard object | Duplicate relinked to the right Account rather than deleted (DDT-1412) |

**Out of scope**

- **Users' merge and delete permissions.** The requesters in DDT-506 and DDT-1412 lacked the access to do this themselves. Whether that access should change is not recorded in the tickets; the merges were done by an admin instead.
- **Preventing new duplicates.** DDT-767 asked whether duplicate rules should be tightened. That was not done under these tickets. DDT-1120 is the separate work on duplicate clients created by lead conversion.

## 4. How it works (background)

```
Salesforce merge (Lead, Account or Contact)
  -> you pick one master record and up to two others
  -> for each field, you choose which record's value survives
  -> related records (Tasks, Opportunities, Campaign Members, ...) move to the master
  -> the other records are deleted, with MasterRecordId pointing at the master
```

- **Up to three records per merge.** Five duplicates (DDT-473) need more than one pass.
- **The losing records are deleted.** That is why a user without delete access on the object gets "You don't have permission to merge this record".
- **Merging is not the same as deleting.** A plain delete removes the record's child activities with it. A merge moves them to the master first.

## 5. Root cause

The duplicates were created by people, not by a defect found under these tickets. The sources show three ways it happened:

- **A search by one field missed the existing record.** In DDT-767 the requester searched by phone number, and the existing Leads had a different number, so they were not found until she searched by name.
- **A new record was created for a returning person.** In DDT-1328 a new Person Account was created for someone who already had a 2021 "Lost Potential Client" account. In DDT-767 the person also already existed as a lost potential client account.
- **Repeat enquiries arrived as new Leads.** DDT-473 and DDT-767 each had several Leads for the same person with the same email.

**What the tickets got wrong, and why you confirm before merging:**

- **DDT-567:** the ticket's record Ids were reversed. The record it labelled "SOS, Phone and Mobile both filled" was in fact the Defense lead with Mobile empty. The merge followed the requester's actual instruction, "keep the original file that is marked SOS".
- **DDT-1328:** the ticket's "Master (keep)" link pointed to the old 2021 record. The requester's message and screenshot both said to keep the new one, so the new one was kept.
- **DDT-506:** the ticket said two duplicates. A third Lead with the same phone and a differently written name turned up during the merge, and was merged in after the requester confirmed.

## 6. Solution

### 6.1 Leads (DDT-473, DDT-506, DDT-567, DDT-767)

- **DDT-473:** all five Leads merged into one. The requester chose to reopen it as `New` rather than keep `Disqualified`, and it was assigned to her for follow-up. The earlier note about the person being unable to afford it stayed on the record.
- **DDT-506:** the two Leads merged first, then the third after confirmation. Result: one Lead, status `Working`, with all 10 activities and the campaign member preserved.
- **DDT-567:** kept the SOS Lead with both Phone and Mobile filled; the Defense duplicate merged in. Result: one Lead categorised as SOS, all 6 activities preserved.
- **DDT-767:** the requester chose to keep the record carrying the phone number from her most recent consultation request. All four merged into it. She was told the person also exists as a lost potential client account, so no new account should be created for him.

### 6.2 Person Accounts (DDT-275, DDT-1328)

- **DDT-275:** the two Person Accounts merged into one.
- **DDT-1328:** kept the newer account (with the current email and phone) and merged the 2021 account into it. Field choices made deliberately:
  - **Carried over from the old record:** Commercial License and State Issued, which were empty on the new one.
  - **Left empty on purpose:** Case Manager (the old value came from the 2021 lost lead) and Heymarket ID (tied to the old phone number, so it would have linked texts to the wrong number).
  - A CSV backup of both records was taken before the merge.

### 6.3 Contact (DDT-1412)

The request was to delete the duplicate Contact. It was **not deleted.** Deleting it would have removed 29 call Tasks and unlinked 24 call sessions and a Matter. It was linked to the correct Account instead, which removed the duplication without losing any history.

## 7. Testing

Not applicable as automated tests: these are data fixes. Each was checked in production after the merge:

| Ticket | Check after the fix |
|---|---|
| DDT-506 | One Lead remains, with 10 activities and 1 campaign member |
| DDT-567 | One Lead remains, categorised SOS, with 6 activities |
| DDT-1328 | Both Opportunities and all 5 text Tasks are on the surviving account; the old account is deleted with `MasterRecordId` set to the survivor |
| DDT-473, DDT-767, DDT-1328 | The requester confirmed the result |

For DDT-275 and DDT-1412 the post-merge check is not recorded in the ticket beyond the resolution note.

## 8. Step by step

1. **Find every copy.** Search by name, phone and email separately. DDT-767 shows that one search field misses records.
2. **Confirm the survivor with the requester in writing.** Do not trust record Ids or links in the ticket on their own: two of these seven had them wrong. Check the Id against what the requester described (status, record type, which fields are filled).
3. **Ask which status the survivor should carry** when the duplicates disagree (for example `New` against `Disqualified`).
4. **Back up first.** Export the records you are about to merge to CSV.
5. **Merge.** Lead: Lead record → Related → Duplicate Records, or the Merge Leads action. Person Account: Account → Merge Accounts. Pick the master, then go field by field.
6. **Choose field values deliberately.** Keep current contact details from the newest record. Carry over values only the old record has. Leave values empty when the old value is stale and would mislead (an old case manager, an id tied to an old phone number).
7. **More than three records:** merge in passes of three.
8. **Before a delete, count what the record owns.** If it has Tasks, call sessions or Matters, relink or merge instead.
9. **Verify** that related records are on the survivor and that the losers carry `MasterRecordId`.
10. **Tell the requester** and ask them to confirm.

The check in step 9 for an account:

```sql
SELECT Id, IsDeleted, MasterRecordId
FROM Account
WHERE Id IN ('<loser-id>')
ALL ROWS
```

## 9. Rollback

- **A merge can't be cleanly undone.** The losing records can be restored from the Recycle Bin for a limited time, but the related records that moved to the master stay there and have to be moved back by hand. That is why step 4 takes a backup and step 2 confirms the survivor before anything is changed.
- **The DDT-1412 relink** can be undone by changing the Contact's Account back.

## 10. Known limits

- **Duplicates will keep arriving.** Nothing in these tickets stops a new Lead or account being created for someone who already exists. Duplicate prevention for clients created through lead conversion is DDT-1120.
- **Some users can't merge their own duplicates.** Without delete access they need an admin each time.

## 11. Reuse for similar issues

**How to recognise one:** a request to "merge", "combine" or "delete a duplicate", or a user hitting "You don't have permission to merge this record".

- **Confirm the survivor against the requester's words**, not the ticket's Ids.
- **Search for more copies than reported.** DDT-506 had three, not two.
- **Prefer merge or relink over delete.** Delete takes the history with it; merge moves it.
- **Back up, then verify with `MasterRecordId`.**
- **Choose every field on purpose**, and write down which values you carried over and which you left empty, as in DDT-1328.
