# LA Sales Pipeline of Potentials report access

| | |
|---|---|
| **Ticket** | DDT-1147, DDT-1433 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-29 (DDT-1147) and 2026-10-30 (DDT-1433) |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-09-28 (DDT-1147); Firm staff, 2026-10-26 (DDT-1433) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports and report folders) |
| **Area** | Sales reporting: the LA and Non-LA pipeline of potentials reports |
| **Related** | None |
| **Code** | Not applicable: report changes made in Salesforce, no code |

---

## 1. Summary

Two intake staff members, a month apart, needed to see their own potentials on the **LA Sales Pipeline of Potentials** report. The report does not decide who is "LA" from a user field, role or group. Both the LA report and its **Non-LA** twin filter **Intake Owner** against a fixed list of names. Adding the person's name to that list on both reports was the fix.

## 2. What was reported

- **DDT-1147, 2026-09-28:** an intake team member asked to see her potentials on the LA report. She needed access to the report, and her name to appear as an Intake Owner row.
- **DDT-1433, 2026-10-26:** a second staff member was to be added to the LA report and counted as LA, not Non-LA. The ticket noted that no User field labelled "LA" existed, so the split had to be driven by something else.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| LA Sales Pipeline of Potentials | Report (Sales Reports folder) | Person added to the Intake Owner "equals" filter list |
| Non-LA Sales Pipeline of Potentials | Report (Sales Reports folder) | Person added to the Intake Owner "not equal to" filter list (DDT-1433) |

**Out of scope**

- **Folder sharing.** For DDT-1433 no change was needed (section 5). For DDT-1147 any folder change is not recorded in the ticket.
- **Opportunity ownership.** No potentials were reassigned.
- **A copy of the LA report** that DDT-1433 found in the same folder. The fix names only the two reports above.

## 4. How it works (background)

```
LA Sales Pipeline of Potentials      filter: Intake Owner equals     [fixed list of names]
Non-LA Sales Pipeline of Potentials  filter: Intake Owner not equal  [the same list]
Both reports group rows by Intake Owner.
```

So a person is "LA" only if their name is typed into the filter list. Being assigned as Intake Owner on potentials is not enough.

## 5. Root cause

The LA/Non-LA split is a hand-maintained name list inside two report filters. When a new intake person starts, nobody adds them, so they fall into the Non-LA report and do not appear on the LA one.

DDT-1433 recorded this explicitly after the ticket's first guess (a user attribute, role, queue or group) was checked and ruled out: there is no user field, role or group behind it.

Report access was not the problem in DDT-1433. The person's role is All Employees, and the Sales Reports folder is already shared with that role at View.

## 6. Solution

- **DDT-1147:** the requester's name now shows under Intake Owner on the LA report. The ticket records the outcome only, not the exact edit.
- **DDT-1433:** the person was added to the Intake Owner filter on **both** reports, so she counts as LA and is excluded from Non-LA.

Why both reports: the Non-LA report uses the same list with "not equal to". Editing only the LA report would count the person in both reports.

## 7. Testing

- **DDT-1147:** the person was confirmed to appear under Intake Owner on the LA report.
- **DDT-1433:** the LA report showed the person with 5 open potentials, and a login as that user confirmed she could open the report and the Sales Reports folder.

**Not covered:** the requester's own confirmation is not recorded on either ticket.

## 8. Step by step

1. Open the LA Sales Pipeline of Potentials report in the Sales Reports folder and edit it.
2. Open the Intake Owner filter ("equals") and add the person's full name exactly as it appears on their User record. Save.
3. Open the Non-LA Sales Pipeline of Potentials report, open its Intake Owner filter ("not equal to") and add the same name. Save.
4. Check the person's role, then check the Sales Reports folder's sharing. If the folder is not shared with that role, share it at View.
5. Run the LA report and confirm the person has a row. Run the Non-LA report and confirm they do not.
6. Log in as the user (Setup → Users → Login) and confirm they can open the folder and the report.

## 9. Rollback

Remove the name from both filters. Safe: reports hold no data.

## 10. Known limits

- The list is manual. Every new LA intake person needs the same two edits, and someone leaving still sits in the list.
- A misspelt name silently matches nothing and puts the person in Non-LA.
- If a third report copies the filter, it needs the same edit.

## 11. Reuse for similar issues

**How to recognise one:** "I can't see my row on report X" or "put me in team A, not team B", where no field on the user explains the split.

- **Open the report's filters first.** Team splits are often a typed list of names, not a field.
- **Look for a mirror report** with the inverse filter, and edit both.
- **Separate access from content.** Check folder sharing against the user's role before changing anything; often access is already there.
- **Verify as the user** with Login As, not as an admin.
