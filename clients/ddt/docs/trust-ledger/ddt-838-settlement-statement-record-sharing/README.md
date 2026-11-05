# Settlement statement access follows matter access again


| | |
|---|---|
| **Ticket** | DDT-838 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-09-28 (reached production) |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-08-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Settlement statement access control |
| **Related** | DDT-835, DDT-836, DDT-842, DDT-843 |
| **Code** | PR #509 in the ddt-salesforce repo |


---


## 1. Summary

An audit found that the Apex behind the settlement statement had, since May 2026, stopped applying record-level sharing, and the repo agreed with production so no drift check could see it. The team measured the real reach before changing anything, confirmed with Firm leadership that the change had not been intended, and restored the original access rule: if a user can reach the Matter, they can reach its settlement statement. A non-admin user was then checked on a real statement and saw the same figures as an admin.

## 2. What was reported

- 2026-08-25: the audit of a May 2026 bulk production-to-git sync flagged that the settlement statement's access setting had been loosened, and that nobody owned the question. The ticket deliberately said not to change it on its own strength: first decide whether the change was intended, then find out who can reach the screen.

## 3. Scope

**In scope**

- The settlement statement's Apex controller: its record access setting and the class header comment, which now states the access rule.
- Its test class: new tests that run as a non-admin user.
- Deploy and history docs that described the old setting now carry dated corrections.

**Out of scope**

- The existing field-level check on the deposit-date field stays as it is; it exists for a separate reason.
- No change to org-wide defaults, profiles or permission sets.

## 4. How it works (background)

Apex can either respect the running user's record sharing or ignore it. The settlement statement reads Matter, lien, expense and payment figures, which are trust-account data. Every other screen in the org answers "can this user see this matter" through record sharing. A controller that ignores sharing answers that question by itself, and will silently disagree with the rest of the system the day sharing is narrowed.

## 5. Root cause

- The class was created on 2026-05-12 respecting sharing. Setup audit trail shows one change on 2026-05-21. On 2026-05-22 a bulk retrieve copied production into git, and the copied version already ignored sharing. The audit trail does not show the code itself, so the 2026-05-21 change is the only identified candidate, not a proven one.
- Firm leadership, who wrote the class, confirmed the change was not deliberate, and the developer found nothing in the class that needed it.
- Why it went unseen for four months: once a bulk sync copies a production mistake into git, git and production agree, so no drift detector can flag it.

Measured exposure (production, 2026-09-24): 37 active internal users can reach the screen. Account is Public Read/Write and the matter and settlement records are controlled by their parent, so every internal user could already see every matter's figures. The loose setting gave nobody extra access that day.

## 6. Solution

- The controller respects record sharing again (PR #509).
- The header comment now carries the rule in plain words: "If a user can reach the Matter, they can reach its settlement statement."

Why revert when there was no exposure today: the protection was dormant, not absent. If the firm ever narrows access (for example practice-area matters to one team), every other screen would respect it and this one would not.

Why this test and not the one the ticket asked for: the ticket asked for a test that a user without matter access cannot read the figures. Because Account is Public Read/Write, no such user exists. The test that matters is the opposite: a real non-admin still gets the figures after the change.

## 7. Testing

- New tests: a matter-management user holding only the settlement statement permission set gets the real, non-zero figures, and can write through the panel (create a recovery, waive an expense).
- Production validation before deploy: 126 of 126 tests across the four test classes that call this controller.
- Before deploying, production's classes were confirmed identical to the PR base; after deploying, production's controller was retrieved and matches `main`.
- Live check, 2026-09-28: logged in as a non-admin matter-management user and opened a real settlement statement. Every figure matched the admin view; no zeros and no load error. Read only.

## 8. Step by step

1. Measure who can reach the screen: profiles and permission sets with access, and whether any are limited to a subset of matters.
2. Check whether anything in the class actually needs to ignore sharing.
3. Get the owner's answer on intent.
4. Restore the sharing setting and write the access rule in the class header.
5. Add a test that runs as a non-admin with only the feature's permission set and asserts real figures come back.
6. Validate with every test class that calls the controller, deploy, then open a real statement as a non-admin and read the numbers.

## 9. Rollback

Revert PR #509 and redeploy the controller. Because sharing is currently wide open to internal users, rollback changes nothing anyone can observe today.

## 10. Known limits

- The audit trail cannot prove which change in May loosened the setting.
- The new test proves real users still see the figures; it cannot prove a restricted user is blocked, because no restricted user exists under today's sharing model.

## 11. Reuse for similar issues

- Signs of the same problem: an Apex access setting that differs from what its author intended, while git and production agree. A drift check will never catch it; read the code.
- Measure the real exposure before changing an access setting; it tells you both the severity and whether the change is safe.
- On a money screen, a green deploy is not proof. Log in as a real non-admin and read the figures; a failed load on this component used to show zeros (DDT-835).
- Write the access rule in the class header in one sentence so the next reader knows the intent.
