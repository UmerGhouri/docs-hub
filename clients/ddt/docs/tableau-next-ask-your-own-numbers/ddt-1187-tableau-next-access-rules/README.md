# Tableau Next access rules: who can see which numbers


| | |
|---|---|
| **Ticket** | DDT-1187 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Live since 2026-10-26 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tableau Next, Data Cloud, permission sets, semantic model) |
| **Area** | Self-serve reporting: access and sensitive data |
| **Related** | DDT-1183, DDT-1185, DDT-1186, DDT-1188 |
| **Code** | PR #804 and PR #822 in the ddt-salesforce repo (records of Setup changes; nothing deployed) |


---


## 1. Summary

Before the AI question tool went beyond the build team, the firm decided what each user may see and checked that it holds. Sensitive fields (trust, bank, card, client date of birth, treatment notes and similar) were hidden from the agent, card and bank fields were stopped at the data stream, and a sensitive identity field was removed from Data Cloud entirely. Testing as a non-admin user showed that the standard Data Cloud permission also opened a raw-data browser, so a reduced permission set was built that gives Tableau the data it needs without that browser. Rep, attorney and case manager names, and court appearances, were then added so staff can ask per-person questions.

## 2. What was reported

- 2026-10-02: test whether Salesforce record sharing carries into Tableau answers; confirm trust, payment and compensation data stay out of self-serve; decide whether reps see other reps' numbers; give view-only users access through a permission set. The test bank must include at least two questions a normal user is refused.

## 3. Scope

**In scope**

- Hiding 23 sensitive fields from the agent across Payment, PI Matter and Lead.
- Disabling card and bank fields on the fw1 Payment stream.
- Rebuilding the Account stream without a sensitive identity field.
- New permission set `DDT_Tableau_Data_Reader`, and additions to `DDT_Tableau_Agent_Access` (PR #804).
- Person names from User (name only) on Opportunity (Intake Owner, Primary Attorney, Court Case Manager) and Court Appearance (Assigned Attorney).
- A Court Appearance data stream with read on 4 fields only (PR #822).

**Out of scope**

- Changing record sharing in Salesforce (see section 5).
- A least-privilege trim of every Data Cloud definition object the reduced permission set still inherits (noted for later in the deploy doc).

## 4. How it works (background)

Data Cloud copies data out of Salesforce and does not apply Salesforce field-level security to it. Hiding a field in the semantic model only keeps the agent away from it; the data is still in Data Cloud. So the real control is who gets access to the model, the agent, and any Data Cloud tool that shows raw rows.

## 5. Root cause

Findings that shaped the rules:

- **Sharing:** Opportunity, Lead, Payment and Call Session are Public Read/Write org-wide and Matter follows its parent, so every internal user already sees every record in Salesforce. There is no record-level difference for Tableau to enforce. Firm leadership confirmed reps may see all reps' numbers.
- **Raw-data path:** a non-admin test user given the standard Data Cloud permission set could open Data Cloud's data browsing tool and see full rows, including fields hidden from the agent. Seven staff already held that permission from before this project.
- **Sensitive identity field:** it had come into Data Cloud through View All Fields on Account. Disabling it on the stream stopped new values but kept the ones already loaded.

## 6. Solution

1. Hid 23 sensitive fields from the agent; the agent refuses trust balance and salary questions.
2. Disabled card and bank fields on the Payment stream (production held 0 stored card security codes).
3. Deleted and rebuilt the Account stream without the sensitive field (2026-10-22): 35,903 records, matching Salesforce. Nothing depended on it.
4. Built `DDT_Tableau_Data_Reader`: a copy of the standard Data Cloud permission set with the raw-data browsing, profile, segment and activation tools switched off. After code review a second raw-row tool was also removed.
5. Moved the six staff who did not use the raw-data tool (checked against page-usage records) to the reduced permission set on 2026-10-23. Data Cloud raw-data access is now limited to firm leadership and the Data Cloud administrators.
6. Linked every person field staff ask about to User, bringing through the name only, and kept the record owner field hidden so it cannot be mistaken for the rep.
7. Added Court Appearance on 2026-10-26: read on 4 fields (court date, assigned attorney, court action, case type), a new stream of 27,325 records matching Salesforce, linked to the court case and to User.

What a Tableau user needs from here: the Tableau Next view-only licence and its unmetered permission set, `DDT_Tableau_Agent_Access`, `DDT_Tableau_Data_Reader`, and Viewer on the DDT Pilot model.

## 7. Testing

- Non-admin test (2026-10-21): August signed 267, open matters 152, September leads 1,627, September cash collected matching Salesforce, trust balance refused.
- With only the reduced permission set (2026-10-22): the agent answers 267 and the raw-data tool does not appear. With the standard permission set instead, raw rows were visible.
- Rep and role checks (2026-10-26): signed in August by rep totals 267; one rep, open cases for an attorney and for a case manager all match Salesforce. "My" questions make the agent ask the user's name and role first.
- Court appearances: next week 134, for one attorney 34, by attorney matches Salesforce row for row.
- Full test bank re-run after each re-index: all pass.

## 8. Step by step

1. Scan every field in the model for sensitive data and hide it from the agent.
2. Disable fields that analytics never needs at the stream itself.
3. Remove sensitive values already loaded by rebuilding the stream without the field.
4. Test as a real non-admin user with only the intended permissions.
5. Check every Data Cloud tool that user can open, not only the agent.
6. Give access through permission sets that leave out raw-data tools.
7. Bring person names in from User, name only.
8. Re-run the test bank after each change.

## 9. Rollback

Unassign and delete `DDT_Tableau_Data_Reader` and remove the two additions from `DDT_Tableau_Agent_Access`; users would then need the standard permission set again, which reopens the raw-data tool. Court Appearance: set its 4 fields back to no access for the connector and delete its stream and model objects. Disabled stream fields cannot be re-enabled.

## 10. Known limits

- The reduced permission set still inherits read on Data Cloud definition and settings objects (not record data). A tighter trim is possible later.
- Each piece of the access set was not tested on its own; the documented set is known to work, not proven minimal.

## 11. Reuse for similar issues

- Signs of the same problem: a field hidden from an AI agent is still visible somewhere else. Check every tool the user's Data Cloud permission opens.
- Hiding a field in a model is not access control. Stop sensitive fields at the stream.
- Test access as a real user with only the intended permissions, and record what they could and could not see.
