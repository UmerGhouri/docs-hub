# Calendly scheduling: a user cannot be selected

| | |
|---|---|
| **Ticket** | DDT-178, DDT-1127 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-16 and 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flow), Calendly |
| **Area** | Intake: booking a consultation with a specific staff member |
| **Related** | DDT-1309 (adding a host option to Schedule Calendly Meeting) |
| **Code** | Not recorded in the ticket |

---

## 1. Summary

Twice, intake could not book a Calendly meeting with one particular staff member while booking with everyone else worked. In June, picking that user in Salesforce's **Schedule Calendly Meeting** modal loaded nothing. In September, that person's tile on the Calendly consultation page could not be clicked. Both were fixed and confirmed working; neither ticket records the exact change. The flow behind the modal finds a user's Calendly link only for a fixed set of job titles, which is the first thing to check when this recurs.

## 2. What was reported

- **DDT-178, 2026-06-16.** On a Lead, an intake user opened Schedule Calendly Meeting, chose **Host = Other** and selected himself as the User. No scheduling page, no time slots, no error: the modal stayed on the user selection screen. Selecting other users from the same modal worked. Fixed 2026-06-17; the user was told to hard-refresh and try again, and that his Calendly link would now resolve.
- **DDT-1127, 2026-09-25.** On the Calendly consultation scheduling page, one intake staff member appeared in the list of people to book with, but her tile did nothing when clicked; it did not advance to time selection. Other staff on the same page worked. A new lead's consultation could not be booked with her. Fixed 2026-09-28: "Calendly consultation buttons now show in production."

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Calendly_Schedule_Calendly_Meeting_v2_Lead` ("[Calendly] Schedule Calendly Meeting v2") | Screen Flow on Lead | How it finds the chosen user's Calendly link (section 4) |
| The affected users' Calendly links and event types | Calendly / User records | The fix location for each ticket is not recorded |

**Out of scope:** the Court Case and Client versions of the scheduling flow; neither ticket involved them.

## 4. How it works (background)

The Lead flow turns the host the user picks into a Calendly link:

```
Host = Lead Owner → the Lead's Lead_Owner_Calendly_Link__c
Host = Other      → Get_Legal_Assistant: active users whose Title is
                    Legal Assistant, Intake Attorney or Intake Specialist
                  → loop: keep up to 7 of them (legal_assistant1 … legal_assistant7)
                  → AssignLA formula: match the chosen user's name against those slots
                    and take that user's Calendly__CalendlyLink__c
                  → free_consultation = link + "/free-consultation?sfid=…&name=…"
```

The formula, as held in the repo:

```
IF({!Host} = {!Lead_Owner}, {!recordId.Lead_Owner_Calendly_Link__c},
IF({!Name_potential.recordName}= {!legal_assistant1.Name}, {!legal_assistant1.Calendly__CalendlyLink__c},
...
IF({!Name_potential.recordName}= {!legal_assistant6.Name}, {!legal_assistant6.Calendly__CalendlyLink__c},
"This Lead's owner is the " & {!recordId.Owner_Name__c} )))))))
```

If the chosen user is **not** among the users the lookup returned, or their Calendly link field is blank, the formula falls through to a sentence instead of a link, and nothing schedulable loads. No error is shown.

## 5. Root cause

**DDT-178:** Not recorded in the ticket beyond "your Calendly link will resolve now". What git does show: the production copy of this flow retrieved on 2026-07-03 had one change against the 2026-05-22 copy. The user lookup's title filter gained **Intake Specialist**:

```
- <filterLogic>(1 OR 2) AND 3</filterLogic>
+ <filterLogic>(1 OR 2 OR 3) AND 4</filterLogic>
+ Title EqualTo "Intake Specialist"
```

A user whose title is outside the list is never loaded, so their link can never be found, which matches the reported symptom exactly. The ticket does not say this was the change made for DDT-178, so treat it as the likely fix, not a confirmed one.

**DDT-1127:** Not recorded in the ticket. The tile that would not click was on Calendly's own consultation page, so the cause was on the Calendly side or in the links it was given (the ticket's guesses were the person's event type, routing or availability).

## 6. Solution

- **DDT-178:** the chosen user's Calendly link was made to resolve, effective 2026-06-17. See section 5 for the flow change that falls in this window.
- **DDT-1127:** fixed so the consultation buttons show in production, effective 2026-09-28. The exact change is not recorded.

## 7. Testing

- **DDT-178:** the user was asked to hard-refresh and select himself again. A confirmation is not recorded.
- **DDT-1127:** the fix was checked in production ("consultation buttons now show"). A test booking end to end is not recorded.

## 8. Step by step

When one person cannot be chosen as a Calendly host:

1. **Check the user's Title.** For the Lead flow it must be Legal Assistant, Intake Attorney or Intake Specialist, and the user must be active:
   ```sql
   SELECT Name, Title, IsActive, Calendly__CalendlyLink__c
   FROM User
   WHERE Title IN ('Legal Assistant', 'Intake Attorney', 'Intake Specialist') AND IsActive = true
   ```
2. **Check the user's Calendly link field is filled** and opens their Calendly page.
3. **Count the rows.** The formula only matches the first six users; a seventh is loaded but never matched (section 10).
4. **If the user's title is legitimately different**, add it to the `Get_Legal_Assistant` filter in the flow (edit production's current version, save as a new version, activate).
5. **If the link resolves but Calendly's page misbehaves** (a tile that will not click), check the person's event types, availability and team membership in Calendly's admin screen.
6. **Ask the requester to hard-refresh** and book a test meeting with that person.

## 9. Rollback

For a flow change, reactivate the previous flow version in Setup → Flows. For a Calendly-side change, undo it in Calendly's admin screen. Neither touches Salesforce records.

## 10. Known limits

- **Only six "Other" hosts can ever resolve.** The loop fills seven slots, but the `AssignLA` formula checks only `legal_assistant1` to `legal_assistant6`. A seventh qualifying user is silently never matched; an eighth is not even loaded.
- **Matching is by name.** Two users with the same full name would collide.
- **Eligibility is by job title.** Renaming a title (for example "Intake Specialist" to something else) removes everyone with it from the list, with no error.
- **Neither fix is recorded in detail**, and the flow version in git is a production snapshot, not a reviewed change.

## 11. Reuse for similar issues

**How to recognise one:** a scheduling or lookup feature works for everyone except one person, and shows no error for that person.

- Find the filter that decides who is eligible (title, role, active flag) and check the person against it.
- Look for fixed-size slots in a flow (variables numbered 1 to N); they cap how many people it can serve.
- A formula that falls back to text instead of failing hides the problem. Prefer an explicit error.
- Write the actual change on the ticket, so the next occurrence does not start from scratch.
