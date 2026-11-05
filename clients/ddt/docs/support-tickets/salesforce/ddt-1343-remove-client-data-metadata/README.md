# Removing client data from metadata and code

| | |
|---|---|
| **Ticket** | DDT-1343 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-16 |
| **Priority** | Low |
| **Reported by** | Firm staff, 2026-10-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom metadata, Apex, Apex tests), repo documentation |
| **Area** | Data hygiene: keeping real client data out of configuration and code |
| **Related** | Follow-up to PR #723 and PR #724 in the `ddt-salesforce` repo |
| **Code** | PR #746 in the `ddt-salesforce` repo |

---

## 1. Summary

Earlier clean-up work (PR #723 and PR #724) removed client names and details from docs, scripts, test data and comments. This ticket covered what was left: real client folder names used as examples in Google Drive configuration records, and a real court-case key in Drive code and tests. All of them were replaced with fictional values in the same format. One client-named custom metadata record was deliberately left alone.

## 2. What was reported

Three remaining items held real client data:

- Three `Drive_Root__mdt` records used real client folder names as examples.
- A real court-case key appeared in `GoogleDriveService`, `GoogleDriveServiceTest` and `DriveFolderControllerTest`.
- One `Channel_Mapping` custom metadata record was named after a client (developer name, label and value), and two manifests listed it.

The ticket's definition of done: none of these items contains client data in production or in `main`, and every class or test touched passes the production check-only gate.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Drive_Root.Cloudfiles_Client_Folders` | Custom metadata record | `Notes__c`: the folder-name example is fictional, same format |
| `Drive_Root.Files_Client_Folders` | Custom metadata record | Same |
| `Drive_Root.Files_Opportunity` | Custom metadata record | Same |
| `GoogleDriveService` | Apex class | One doc comment: the example court-case key is now `1234567` |
| `GoogleDriveServiceTest` | Apex test | Key `1234567` (7 digits, same length as before) in the mocked folder names and the assertions |
| `DriveFolderControllerTest` | Apex test | Same key in one fixture folder name and its assertion |
| `docs/deploys/native-drive-integration.md` | Repo doc | The same real names and keys in its naming examples replaced with the fictional ones |

**Out of scope**

- **The client-named `Channel_Mapping` record.** It was left as it is: renaming it would change live attribution matching for little gain. It is still listed in two manifests.

## 4. How it works (background)

`Drive_Root__mdt` records describe where each kind of Google Drive folder lives and how it is named. Their `Notes__c` field holds a human-readable example of a folder name. `GoogleDriveService.roots()` selects `Notes__c`, but no code uses its value, so the examples are documentation only.

`Channel_Mapping` records are used for marketing attribution: incoming values are matched against the record's value, which is why renaming one is a behaviour change and not just a label change.

## 5. Root cause

Not applicable: this was clean-up, not a bug. Real examples had been used in configuration notes and test fixtures; how they got there is not recorded in the ticket.

## 6. Solution

**Replace, don't delete.** Each real value was replaced with a fictional one **in the same format**: same separators, same date style, same 6- and 7-digit key lengths. That keeps the examples useful and keeps the tests checking the same thing.

- In the tests the key appears in **both** the mocked folder name and the assertion, so both were changed together.
- Only `Notes__c` changed on the three Drive records. A custom metadata deploy rewrites the whole record, so a read-back guard (`scripts/cmt-readback-guard.py`) was run before deploying. It reported no collateral overwrite: only `Notes__c` changes on each record.
- **No behaviour change.** Nothing reads `Notes__c`, and the Apex change in `GoogleDriveService` is a comment.

**Why the `Channel_Mapping` record stayed.** Its value is what attribution matches on. Renaming it would change how live leads are attributed, which is a business risk out of proportion to a name in configuration.

## 7. Testing

Validated with `GoogleDriveServiceTest` and `DriveFolderControllerTest`. These classes don't compile in the partial sandbox (the Drive integration exists only in production), so the production check-only run was the test run. The deploy reached production on 2026-10-16.

**Not covered:** the `Channel_Mapping` record still carries client data in production and in git.

## 8. Step by step

1. **Find the remaining occurrences.** Search the repo for the real names and keys, including custom metadata, test classes, comments and docs.
2. **Choose fictional values in the same format** (same separators, date style and lengths).
3. **Replace them everywhere they appear together**, especially a test fixture and its assertion.
4. **For custom metadata, check what else the deploy will overwrite.** A custom metadata deploy rewrites the whole record. Run the read-back guard against production and confirm only the intended field changes.
5. **Before renaming any record whose value something matches on**, check what matches on it. If it is live matching, decide separately.
6. **Validate check-only against production** with the affected test classes, then deploy after approval.
7. **Re-run the read-back guard after deploying** to confirm production matches the repo.

## 9. Rollback

Not advisable: rolling back would put real client data back into configuration. The change has no behaviour effect, so there is nothing functional to roll back.

## 10. Known limits

- **One client-named `Channel_Mapping` record remains**, by decision.
- **New client data can creep back in** whenever someone copies a real example into a fixture or a note. Nothing blocks it automatically.

## 11. Reuse for similar issues

- **Replace with fictional values in the same shape**, never with obviously fake placeholders that change length or format. Parsers and tests often depend on the format.
- **Change a fixture and its assertion together** so the test still proves the same thing.
- **Custom metadata deploys overwrite the whole record.** Check production's current values first.
- **Separate documentation fields from matched values.** A note can be changed freely; a value something matches on is behaviour.
