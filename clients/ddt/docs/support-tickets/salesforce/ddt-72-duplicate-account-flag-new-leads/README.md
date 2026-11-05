# Every new lead flagged as a duplicate account

| | |
|---|---|
| **Ticket** | DDT-72 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Lead intake: duplicate account check when a new lead is created |
| **Related** | None recorded |
| **Code** | Not recorded in the ticket. The change was made to a flow; no PR is linked. |

---

## 1. Summary

Every new Lead was getting the duplicate Account flag, even when no matching Account existed. The duplicate matching in the `Create_New_Lead` flow ran even when the Lead had no email, phone or contact to match on, and those blank values produced a false match. The fix added null checks so the matching only runs when there is real identifying information.

## 2. What was reported

- **2026-06-01, DDT-72:** every new Lead fired the duplicate Account flag, including leads with no real matching Account. This added friction to intake. The ticket did not record how many users or leads were affected, or when it started.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Create_New_Lead` | Flow | Null checks on lead email, phone number and contact before the duplicate Account matching logic |

**Out of scope**

- **Salesforce Duplicate Rules and Matching Rules.** The ticket listed them as likely causes to investigate. The fix was made in the flow, and the ticket does not record any change to the rules.

## 4. How it works (background)

```
New Lead is created
  -> Create_New_Lead flow runs
  -> duplicate Account matching logic compares the Lead's email / phone / contact
  -> if a match is found, the Lead is flagged as a duplicate account
```

## 5. Root cause

From the resolution note: when the Lead's email, phone number or contact was missing, the matching logic still ran on the blank values, and that caused every new lead to be flagged.

The ticket's first guesses (an over-broad Duplicate Rule, a loose Matching Rule) were not where the fix landed. The cause was in the flow's own matching step.

The query or record evidence was not recorded in the ticket.

## 6. Solution

The `Create_New_Lead` flow now checks that lead email, phone number and contact are not null **before** the duplicate Account matching runs. The deduplication logic only executes when valid identifying information is present.

Why this way: matching on an empty value is the classic false-positive. Two blanks compare as equal, so every lead without that value "matches" every account without it. Guarding the input removes the false match without loosening the check for leads that do have the data.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for two checks, which are the right ones to repeat:

- A new Lead with no real duplicate shows **no** flag.
- A new Lead that genuinely matches an existing Account **still** shows the flag.

## 8. Step by step

1. Open Setup, Flows, and open `Create_New_Lead`.
2. Find the step that runs the duplicate Account matching.
3. Before it, add a Decision that checks the lead email, phone number and contact are each not null (the values the match uses).
4. Route to the matching step only when the identifying values are present. Otherwise skip it.
5. Save as a new version and activate.
6. Create a test Lead with no real duplicate and confirm no flag appears.
7. Create a test Lead that matches a real Account and confirm the flag still appears.

## 9. Rollback

Reactivate the previous version of `Create_New_Lead` in Setup, Flows. That brings the false flags back, so only do it if the new version itself fails.

## 10. Known limits

- The guard only stops matches on **missing** values. A lead with a placeholder value (for example a dummy phone number) would still match other records holding the same placeholder.
- The ticket's open questions (whether to flag across Lead, Person Account and Contact) were not answered in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** a duplicate or match check that fires on almost every record, regardless of the data.

- Check what the match does with **blank** inputs first. Blank equals blank is the most common cause.
- Guard every match input with a not-null check before the comparison runs.
- Test both directions: no false flag on a clean record, and the flag still fires on a real duplicate.
