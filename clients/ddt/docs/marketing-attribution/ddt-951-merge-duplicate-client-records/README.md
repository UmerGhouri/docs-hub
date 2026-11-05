# Merge duplicate client records and clean invisible characters from phones


| | |
|---|---|
| **Ticket** | DDT-951 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Person Accounts, Leads, data operations) |
| **Area** | Client identity: duplicates and phone data quality |
| **Related** | DDT-825, DDT-826, DDT-967, DDT-822, DDT-1081, DDT-1120 |
| **Code** | No code change |


---


## 1. Summary

A measurement for the identity-key work (DDT-825) found ever-hired clients recorded as two or more Person Accounts sharing an exact phone or email. When a returning client is a second record, repeat revenue cannot flow back to the channel that first won them. This was data work, not code. Each candidate pair was reviewed by a person. 27 duplicate groups (28 records) were merged into the oldest record, and every remaining candidate group was documented with the reason it is not a duplicate. A second job stripped invisible characters (pasted zero-width spaces, iOS direction marks, non-breaking spaces) out of phone fields that no string match could ever find.

## 2. What was reported

- 2026-09-08: 191 duplicate Client records attached to 225 Court Cases share an exact phone or email with another ever-hired Client. Merge them by hand after a human confirms each pair, keep the oldest record, back everything up first, and do not run during the DDT-826 re-score window.
- 2026-09-08 (added): 15 Person Accounts and 22 Leads carry invisible or non-ASCII characters inside a phone field, plus one Lead with a sentence instead of a number. Strip them at the data.

## 3. Scope

**In scope**

- Person Accounts with ever-hired Court Cases that share an exact `Phone`, `PersonMobilePhone` or `PersonEmail` value.
- Standard Account merge, oldest record surviving.
- Phone values on Person Accounts and Leads containing invisible or non-ASCII characters.

**Out of scope**

- Duplicates that share no phone or email (found only by name plus date of birth). Five such pairs were listed and moved to DDT-1081.
- A validation rule or save-time strip to stop new pasted characters; that would be a code ticket.
- Preventing new duplicates at intake (see DDT-1120).

## 4. How it works (background)

Attribution decides whether a new case is a returning client by matching phone and email against earlier clients. Two records for one person break that match and split the client's history. Person Account merges are irreversible and carry Court Cases, Matters and payment history with them, and merging fires the surviving record's automation.

## 5. Root cause

Duplicates were never merged, although an exact match already found them. The original count was inflated. Re-checked during the work, many "groups" were not duplicates: the firm's own main line was stored on 17 unrelated clients, and other matches were intake placeholders or households sharing a number. After confirming each candidate against date of birth and driver's licence, 27 groups were true duplicates. The revenue figure on the ticket was overstated about 5.4 times for the same reason, which changed the priority comparison with DDT-825 from "five times the value" to roughly equal.

The invisible characters come from pasting: zero-width spaces from web pages, direction marks from iPhone copies, non-breaking spaces from Word or Outlook, and non-breaking hyphens or en dashes from autocorrect.

## 6. Solution

1. Back up the candidate records and their children before any change.
2. Regenerate the candidate list from production instead of trusting a stored extract.
3. Have a person confirm each pair; a shared phone is a strong signal but not proof.
4. Merge confirmed pairs in small batches with the standard Account merge, oldest surviving, so the original acquisition date and source are kept.
5. Write a reason for every candidate group left unmerged.
6. Strip anything that is not a digit, bracket, space or hyphen from the affected phone values, and blank the one non-number rather than inventing a value.

Why manual: 191 candidates is an afternoon of review, and a bulk merge on a shared phone would have merged spouses, parents and children, and every client tagged with the firm's own number.

## 7. Testing

- After the merges, no name-identical duplicate remained among ever-hired Clients sharing an exact phone or email. The 164 remaining candidate groups and 23 shared values each carry a recorded reason.
- Phone clean-up: 15 Person Accounts and 6 Leads fixed. 17 Leads remain: 16 are converted and locked by the platform, and 1 was blocked by a duplicate rule that correctly caught a real duplicate.
- No data lost, per the closing note.
- Not covered: merges were not rehearsed in a sandbox (not recorded in the ticket).

## 8. Step by step

1. Pull Person Accounts with ever-hired cases that share an exact phone or email, checking every phone field, not only the primary one.
2. Exclude known shared numbers (the firm's own lines, placeholders) before review.
3. Back up the records and their children.
4. Confirm each pair with a second identifier such as date of birth.
5. Merge in small batches, oldest surviving, and watch the first few for automation side effects.
6. Re-run the count and document each residual group.
7. For phone fields, search for non-ASCII characters and strip them in one small batch.

## 9. Rollback

Person Account merges cannot be undone in Salesforce. The only recovery is re-creating records from the pre-merge backup. Phone clean-up can be reversed from the same backup.

## 10. Known limits

- Five duplicate pairs share no phone or email and are only found by name plus date of birth (DDT-1081).
- 17 Lead phone values could not be edited (16 converted, 1 blocked by a duplicate rule).
- Nothing stops new pasted characters from arriving; where intake pastes from is still an open question.

## 11. Reuse for similar issues

- Signs: a returning client is counted as new, or a phone match fails on a number that looks correct on screen.
- Before treating shared phone or email as duplicates, remove the firm's own numbers, placeholders and households.
- Confirm with a second identifier and merge by hand; bulk merge on a key is unsafe and irreversible.
- Normalise phones by stripping non-digits rather than comparing formatted strings; invisible characters defeat every format match.
