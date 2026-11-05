# Removing the redundant Violation of Sentence path

| | |
|---|---|
| **Ticket** | DDT-94 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-03 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Court Case record page, component visibility) |
| **Area** | Case management: stage paths on the Court Case page |
| **Related** | DDT-92 (Federal case status path) |
| **Code** | Not applicable: visibility change in the org, no pull request |

---

## 1. Summary

On Court Cases with Case Type "Violation of Sentence", a second path showed under the main one and read "No Stage Found". It belonged to the case plan builder, which showed for **any admin user** as well as for hired SOS cases. We removed the admin condition, so it now shows only for SOS cases with Hire Status "Hired".

## 2. What was reported

On 2026-06-02, a user opened a Court Case with Case Type "Violation of Sentence" and saw a second path, titled "Violation of Sentence", under the main path. It showed only "No Stage Found". It never resolved to a stage, so it added clutter and confused users. No user was blocked.

Acceptance criteria:

- The secondary "No Stage Found" path no longer appears on Violation of Sentence cases.
- No change to the main Intake → Case Manager → Attorney path.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Case plan builder path on the Court Case record page | Component visibility | Admin condition removed |

**Out of scope**

- **The main Intake → Case Manager → Attorney path.** Untouched.
- **Building real stages for Violation of Sentence.** The ticket allowed for that if the path was meant to drive a real stage flow. It wasn't; it was showing by mistake.

## 4. How it works (background)

The ticket named the object as `Court_Case__c`. In this org the Court Case is the standard Opportunity object, relabelled "Court Case", and that is where the record page lives.

The case-plan path looks up the stages for the record's practice area and Case Type. When there are no stages for that combination, it shows a single placeholder stage, "No Stage Found".

```
Court Case page loads
  -> visibility rule decides whether the case-plan path shows
  -> path looks up stages for the practice area + Case Type
       found     -> draws them
       not found -> draws one stage: "No Stage Found"
```

So the placeholder appears whenever the path is visible on a case type with no stages defined.

## 5. Root cause

From the fix note (2026-06-03): the path was visible when **either**

- the user was an admin, **or**
- Case Type was SOS **and** Hire Status was "Hired".

A Violation of Sentence case is not an SOS case, so the path had no stages for it. It showed only because the viewing user was an admin. Under that rule, a non-admin user would not have seen it on that case. That is why it looked like a "Violation of Sentence path": it was the general case-plan path, drawn on a case type it was never meant for.

**The lesson.** An "or admin" clause in a visibility rule makes admins see a different page from everyone else. Admins are the ones who report layout problems, so they end up reporting problems other users don't have.

## 6. Solution

The visibility rule now requires both:

- Case Type = SOS
- Hire Status = Hired

It no longer shows just because the user is an admin.

Why remove the clause rather than add a "Violation of Sentence" exclusion: an exclusion would fix this one case type and leave the placeholder showing for admins on every other case type with no stages. Removing the clause fixes all of them.

## 7. Testing

Not recorded in the ticket.

**Not covered:** no recorded check that the main path was unchanged, though the change touched only the case-plan path's visibility.

## 8. Step by step

1. **Open an affected record as an admin and as a non-admin.** If only the admin sees the extra path, the visibility rule has an admin clause.
2. **Open the Court Case record page** in Lightning App Builder and select the case-plan path component.
3. **Read its visibility rule**, including the filter logic (the AND / OR expression), not only the list of conditions.
4. **Remove the admin condition** and set the logic so the remaining conditions are all required.
5. **Save and activate** the page.
6. **Reopen the affected record as an admin** and confirm the extra path is gone; open a hired SOS case and confirm it still shows.

## 9. Rollback

Add the admin condition back to the visibility rule. Safe, but it brings the "No Stage Found" path back for admins on every non-SOS case.

## 10. Known limits

- **Admins lose a shortcut.** If admins used the path to inspect plans on non-SOS cases, they now can't from this page.
- **The placeholder still exists.** Any future visibility rule that shows the path on a case type with no stages brings "No Stage Found" back.

## 11. Reuse for similar issues

**How to recognise one:** a component shows an empty or placeholder state, and only some users report it.

- **Compare an admin view with a normal user's view** before changing anything.
- **Read the filter logic, not just the conditions.** A single OR can make a component appear far more widely than intended.
- **Avoid "or admin" in visibility rules.** It hides problems from regular users and shows problems only admins see.
- **Fix the rule, not the one symptom.** An exclusion for one case type leaves the same problem on the others.
