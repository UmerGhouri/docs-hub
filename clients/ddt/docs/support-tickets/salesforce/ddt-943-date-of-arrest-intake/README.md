# Date of Arrest on the intake screens

| | |
|---|---|
| **Ticket** | DDT-943 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-16 |
| **Priority** | Urgent |
| **Reported by** | Firm operations, 2026-09-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flows) |
| **Area** | Intake: creating a Potential Court Case and following up on it |
| **Related** | None |
| **Code** | PR #403 in the ddt-salesforce repo (builds on the fix in PR #252) |

---

## 1. Summary

The new Gameplan page, sent to people who have not hired the firm yet, counts down the days until they lose their driver's licence. That needs the arrest date, and Salesforce did not have it at the Potential stage: it was typed by the case manager on the Welcome Call, after the hire.

A "Date of Arrest" question was added to the two screens intake already uses, both writing the existing `Date_Time_of_Arrest__c` field. The change also closed a trap in one of the flows that would otherwise have wiped the arrest date on 901 existing records.

## 2. What was reported

Checked in production on 2026-09-06: of 197 Potential DUI and Felony DUI Court Cases created in the previous 90 days, 71 had `Date_Time_of_Arrest__c`, and all 71 were old. The newest Potential carrying it was created on 2026-07-27, when the old vendor tool that used to fill it stopped working. The field was still filled on hired cases, by the case manager. So the date was being typed, but too late for a page sent before the hire.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Case_Follow_Up_Attempt` | Screen Flow (the Follow Up card on the Potential Court Case page), v3 to v4 | Date of Arrest question on both input screens, written on both save paths; a sixth condition on `If_All_Fields_Are_Blank`; a Date of Arrest line on the summary screen; a "Follow Up Attempt" section header |
| `Intake_Court_Case_v2` | Screen Flow (the Court Case step of `DDT_Intake_Flow`), v26 to v27 | Date of Arrest question after Case Type, written on create and update; a missing fault path restored from production |
| `flowDefinition` files for both | Flow definitions | Pin the new active versions, 4 and 27 |

**Out of scope**

- **`Intake_Court_Case_for_Business_Accounts`.** A business does not get arrested.
- **`Intake_Court_Case`** (no `_v2`). Still Active but nothing calls it.
- **The Welcome Call screen.** Unchanged; the case manager now confirms a date instead of typing a new one.
- **New fields, required fields and the time of arrest.** None added. Only the date is asked.
- **`Arrest_Date__c`, `Date_Time_Pulled_Over_NOT_Arrest__c`, `Date_Time_of_Occurrence_NOT_Arrest__c`.** Not used. `Arrest_Date__c` has the obvious name and has never held a value in production.

## 4. How it works (background)

```
New Potential:   DDT_Intake_Flow -> Intake_Court_Case_v2 (Court Case screen) -> Intake_Citation
Follow-up call:  Potential Court Case page -> Follow Up card (Court_Case_Follow_Up_Attempt)
                   -> If_All_Fields_Are_Blank?
                        yes -> Initial screen (opens blank)
                        no  -> summary screen -> Initial2 (prefilled from the record)
```

The `If_All_Fields_Are_Blank` decision decides whether a record is a first entry or a repeat. It only checks the fields it lists. A record that has data in a field the decision does not check is treated as blank, opens an empty screen, and saving that screen writes empty values over the record.

## 5. Root cause

**Why the date was missing.** The field was only ever filled after the hire, so it could not be used before it. Not a bug: the question was simply not asked at intake.

**The trap the change had to avoid.** `If_All_Fields_Are_Blank` tested five follow-up fields. If the new question were added without adding the arrest date to that decision, a Court Case with an arrest date but no follow-up history would take the blank path and the save would write null over the arrest date. Measured in production on 2026-09-16, that was **901 of the 3,119** Court Cases carrying an arrest date (28.9%): arrest date set and all five follow-up fields blank.

This had happened before on the same decision. PR #252 fixed the same failure for the client concerns field, after it had wiped concerns on 403 of 1,798 records.

## 6. Solution

**Both screens write the same existing field**, `Date_Time_of_Arrest__c`. The question is a Date and not required.

**The noon rule.** The field is a DateTime and the question is a Date. Letting Flow convert a Date gives midnight GMT, which shows as the previous day to a Chicago user. Noon GMT shows as the right day on both sides of the daylight-saving switch:

| Stored | Shown to a Chicago user |
|---|---|
| 2026-03-03 12:00 GMT | 2026-03-03 06:00 |
| 2026-03-03 00:00 GMT | 2026-03-02 18:00 |

**The write is guarded** so it never destroys a value:

```
IF( ISBLANK(Date_of_Arrest),
    <the value already on the record>,
    IF( Date_of_Arrest = DATEVALUE(<the value already on the record>),
        <the value already on the record>,
        DATETIMEVALUE(TEXT(Date_of_Arrest) & " 12:00:00") ) )
```

- A blank answer never clears a date someone already typed. These screens are re-entered, and the intake question is hidden once the case is Hired, which makes it blank in the flow.
- An unchanged answer keeps the exact original value, including a real time a case manager typed. Rewriting it to noon could move a late-evening arrest onto the next day.

**Follow Up Attempt.** The question is on `Initial` and `Initial2`; `Initial2` prefills from the record. `Date_Time_of_Arrest__c IsNull` is added as a sixth condition on `If_All_Fields_Are_Blank`, so records with an arrest date take the prefill path. The summary screen in front of `Initial2` now shows a Date of Arrest line first; without it, a record sent there because of its arrest date showed five empty labels and nothing else. All three screens got a "Follow Up Attempt" section header, at the requester's request, because the card had no title.

**Intake Court Case v2.** The question sits right after Case Type, prefilled from the existing record and kept across the County/City loop-back, and hidden once Hire Status is Hired. The branch also restored a fault path on `Create_Court_Case` that existed in production but not in git; deploying git's copy as-is would have removed it.

**`DDT_Intake_Flow` version 66.** The ticket thought it was an unreleased draft. It is Obsolete: it was activated and rolled back to version 65 a minute later. Nothing to bury or switch on. `DDT_Intake_Flow` itself was not changed.

## 7. Testing

All seven screen tests from the ticket were clicked through by hand in the sandbox on 2026-09-16 and passed:

| # | Test | Result |
|---|---|---|
| 1 | New value | Saved as noon GMT, not midnight |
| 2 | Existing value prefills | Card opened on the summary screen with the date prefilled |
| 3 | Existing value survives an unchanged save | The original value was kept exactly |
| 4 | Right day | 03/03/2026 read as March 3, not March 2 |
| 5 | New Potential through the intake flow | Date stored on the day typed |
| 6 | Left empty | Flow finished with no error |
| 7 | Business account | Finished with no error and never asked for the date |

Test 1 is what exposed the empty summary screen, fixed before deploy.

Both flows were validated check-only against production. After deploy, the question was confirmed visible on a real Potential in production, with `Court_Case_Follow_Up_Attempt` on v4 and `Intake_Court_Case_v2` on v27.

**Not covered:** the negative control for test 3 (remove the sixth condition, confirm test 3 then fails) was listed as outstanding in the deploy doc. The test record had the at-risk profile and opened on the summary screen, which is the rerouted behaviour, but that is inference rather than the control itself.

## 8. Step by step

1. **Confirm which flows the screens use.** Open `DDT_Intake_Flow` in production and check the subflow its Court Case step calls.
2. **Retrieve both flows from production** and compare with git. Keep anything prod has that git does not (here, the fault path).
3. **Add the Date question** to each screen and write it with the guarded noon formula in section 6.
4. **Add the new field to `If_All_Fields_Are_Blank`.** Do this in the same change, not as a follow-up.
5. **Add the field to the summary screen** so a record routed there shows why.
6. **Click through all seven tests in a sandbox.** A clean deploy does not open a screen.
7. **Run the negative control:** remove the new decision condition, confirm test 3 fails, then restore it.
8. **Pin the versions.** Production deploys flows as Draft, so the `flowDefinition` files must name the new version. Confirm the current active versions just before deploying and bump the pin if they have moved.
9. **Deploy and read back** that the newest version of each flow is Active.
10. **Open a real Potential in production** and confirm the question shows on both screens.

## 9. Rollback

Reactivate the previous version of each flow (v3 and v26) in Setup. Arrest dates already typed stay on the records, which is safe because it is the same field the case manager uses. If the follow-up flow is rolled back, the old `If_All_Fields_Are_Blank` decision returns, so records with an arrest date and no follow-up history are exposed to the wipe again.

## 10. Known limits

- **Any new field added to these screens must also be added to `If_All_Fields_Are_Blank`.** Missing it is what wiped data in PR #252, and it would have here.
- **The version pins go stale** the moment anyone else deploys either flow.
- **The Gameplan countdown needs a few weeks of typed dates** before it is useful; it was not switched on as part of this ticket.
- **The negative control on test 3** was not recorded as run.

## 11. Reuse for similar issues

**How to recognise one:** a value is needed earlier in the process than it is currently entered.

- **Ask for it on a screen people already use**, writing the existing field. Don't add a second field for the same fact.
- **Date into DateTime: store noon GMT**, and guard the write so blank never clears and unchanged keeps the original.
- **In any flow that decides "first entry or repeat" by checking fields, add every new field to that check.** Otherwise the save writes blanks over real data.
- **Prove a protective test can fail** by removing the fix and running it again.
- **Retrieve flows from production before editing.** Git can be behind, and deploying git's copy reverts prod's changes.
