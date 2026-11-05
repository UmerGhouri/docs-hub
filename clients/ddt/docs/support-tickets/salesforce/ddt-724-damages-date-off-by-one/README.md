# Damages treatment date shows one day early

| | |
|---|---|
| **Ticket** | DDT-724 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC), GitHub Actions CI |
| **Area** | PI Matter Damages tab, plus every LWC that displays or compares a Salesforce Date |
| **Related** | None recorded |
| **Code** | PR #163 in the ddt-salesforce repo |

---

## 1. Summary

On a PI Matter, a First Treatment Date entered as Aug 10 showed correctly in the edit modal but as 8/9/26 in the Damages list. The stored value was correct; the list's JavaScript parsed the date string as UTC midnight, which is the previous evening in Central Time. The fix moved every affected component onto a new shared date module, `c/sfDate`, and added a blocking CI check so the same bug cannot be reintroduced.

## 2. What was reported

A staff member entered a First Treatment Date of 8/10/2026 on a Medical damages record. The edit modal showed "Aug 10, 2026"; the Damages list showed 8/9/26. The ticket suspected a Date versus Datetime mismatch or a formula source.

The investigation found the scope was wider than the ticket:

- **All 7 date columns** on the Damages list were a day early: service date, records received date, date incurred, first and last treatment date, first and last date missed.
- **Three other live components** had the identical bug: the settlement statement (settlement reached date), the SLA timer (elapsed days started a day early, so SLAs read one day more overdue), and the matter task list (a task due today showed as overdue).
- **The mirror bug in 5 places:** date pickers pre-filled with `toISOString().slice(0,10)`, which gives tomorrow's date after 7pm Central.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `sfDate` | LWC service module (new) | Parse, format and emit Salesforce Dates with no timezone conversion; 16 Jest tests |
| `damagesTable` | LWC | `fmtDate` uses `formatSfDateShort`; 3 date-picker defaults fixed |
| `settlementStatement` | LWC | `settlementReachedDate` fixed; its hand-written month table moved into `sfDate` |
| `slaTimer` | LWC | Elapsed-days maths and displayed start date |
| `matterTasks` | LWC | Overdue comparison uses local midnight on both sides |
| `convertToHiredMatter` | LWC | No behaviour change; one line marked as a reviewed exception (see 6) |
| `scripts/date-lint.py`, `scripts/test_date_lint.py` | CI script and tests (new) | The check and 35 tests of it |
| `.github/workflows/ci.yml` | CI | New blocking `date-lint` leg |
| `jest.config.js` | Jest config | Pins the test timezone to `America/Chicago` |
| `CLAUDE.md` | Repo rules | The Salesforce Date rule |

**Out of scope:** the data. `Damages__c.First_Treatment_Date__c` and `Last_Treatment_Date__c` are `Date` fields and the stored values were correct, so no records needed repair. Datetime fields, which convert to the viewer's timezone correctly. Pre-existing lint errors in `damagesTable.js` and `slaTimer.js` were cleared only because they blocked any commit to those files; that part is mechanical.

## 4. How it works (background)

```
Damages__c.First_Treatment_Date__c  (Date, stored as 2026-08-10)
  -> Apex returns it to the LWC as the string "2026-08-10"
  -> damagesTable.fmtDate did new Date("2026-08-10")
  -> JavaScript reads a date-only ISO string as UTC: 2026-08-10T00:00:00Z
  -> displayed in Central Time: Aug 9, 7:00 pm  ->  "8/9/26"
```

The edit modal was correct because `lightning-input type="date"` handles the raw string itself and never went through `fmtDate`. Same record, same field, two rendering paths, one of them lossy.

A Salesforce Datetime arrives as a full instant (for example `"2026-08-10T14:23:00.000Z"`), and converting that to the viewer's timezone is correct. Only date-only values are affected.

## 5. Root cause

`damagesTable.js`'s `fmtDate` helper called `new Date(d)` on a Salesforce Date string. ECMA-262 parses the date-only ISO form as UTC. Every user of the org is in Central Time, which is behind UTC all year, so every date rendered as the previous day, for every user, every time.

The ticket's first guess (field type mismatch, or a formula or roll-up source) was wrong: both fields are `Date` and the list read the same field as the modal.

**Why it kept coming back.** This was at least the fourth occurrence. `settlementStatement` had already been fixed inline, with a comment explaining the trap, and the same file reintroduced the bug three getters further down. The same line of JavaScript is right or wrong depending on the type of a field declared in a different file and a different language, so a reviewer cannot tell by reading the component. `Date_of_Incident__c`, for example, is a `Date` on `Matter__c` and a `DateTime` on `Lead`, under the same API name.

## 6. Solution

**1. One shared module, `c/sfDate`.** It splits the `YYYY-MM-DD` string into year, month and day and never goes through a timezone:

| Instead of | Use |
|---|---|
| `new Date(row.firstTreatmentDate)` | `parseSfDate(...)`: local midnight, safe to compare |
| `new Date(d).toLocaleDateString()` | `formatSfDateShort` / `Medium` / `Long` / `Padded` |
| `new Date()` then `setHours(0,0,0,0)` | `todayLocal()` |
| `new Date().toISOString().slice(0, 10)` | `todaySfDateString()` |
| `someDate.toISOString().slice(0, 10)` | `toSfDateString(someDate)` |

All five components were moved onto it.

**2. A blocking CI check, `scripts/date-lint.py`.** A written rule had already been tried, in a code comment, in the file that then broke again. So the check does not rely on anyone remembering. It does not guess which names are dates: it builds the set from the two places the repo declares them (`@AuraEnabled public Date` in Apex and `<type>Date</type>` in field metadata), then flags `new Date()` on those names and `toISOString().slice(0,10)`. It hard-fails if that discovery returns nothing, so it cannot report clean on a scan it never ran. It was made blocking from day one because there was no legacy debt to grandfather: all 139 LWC JS files were clean.

**3. A reviewed exception, not a silent one.** `convertToHiredMatter` reads the Lead's `Date_of_Incident__c`, which is a `DateTime`, while the Matter field of the same API name is a `Date`. That line carries a `date-lint: ok - <reason>` marker; a marker without a reason does not suppress.

**4. Pin the test timezone in `jest.config.js`.** Jest runs in UTC in CI, where the buggy code gives the right answer, so a timezone test that does not pin Central Time proves nothing.

## 7. Testing

| Check | Result |
|---|---|
| `date-lint` | 139 of 139 LWC files clean |
| `test_date_lint` | 35 of 35, including that the lint still fails on the exact code that shipped this bug |
| LWC Jest | 127 of 127 pass, 16 of them new in `sfDate` |
| ESLint | Clean on all touched files |
| CI | All 6 checks green |

**CI caught a defect in the new tests.** The first version set `process.env.TZ` at the top of `sfDate.test.js`. That is silently a no-op: Node caches the timezone the first time a `Date` is created, before a test file's body runs. It passed on developer machines that were already on Central Time; the UTC CI runner failed 3 of 16, and they were exactly the tests asserting the trap is live. Under UTC the other 13 passed against the broken code. The pin was moved to `jest.config.js`, which runs before Jest starts its workers, and a guard assertion (`new Date('2026-08-10').getDate() === 9`) now fails loudly if the pin stops working. Verified with `TZ=UTC` and `TZ=America/Chicago`, 127 of 127 each.

**Not covered:** no test renders the component on a real Lightning page. Live verification on a fresh record by a Central Time user, including one entered after 7pm Central, was left to the reporter after the production deploy; its result is not recorded in the ticket.

## 8. Step by step

1. **Check the field's real type** in its metadata (`<type>Date</type>` or `DateTime`), not from the label or the API name.
2. **Find the rendering path** that differs. If the edit form is right and a list or summary is wrong, look for `new Date(...)` on the value in that component's JavaScript.
3. **Replace it with the matching `c/sfDate` function** from the table in section 6.
4. **Look for the mirror bug** in the same component: `toISOString().slice(0, 10)` used for a default date.
5. **Run `python scripts/date-lint.py` locally** (the repo's docs say `python3`; on some Windows machines that is `python`).
6. **Write Jest tests that pin Central Time** through `jest.config.js`, and include one assertion that fails if the pin is missing.
7. **Open the component on a real page** after deploying; CI does not render LWCs.

## 9. Rollback

Revert PR #163 and redeploy the previous versions of the five components. That brings the off-by-one dates back on every affected screen, so there is no reason to unless a component breaks. The CI leg can be removed from `ci.yml` independently. No data was changed, so nothing needs restoring.

## 10. Known limits

- `date-lint` only knows date-only names declared in this repo. A field returned under an alias, or a date computed and passed through without its declared name, is not discovered.
- A field name is not a type: the same API name can be a `Date` on one object and a `DateTime` on another. Each exception needs a reviewed `date-lint: ok - <reason>` marker.
- The check covers LWC JavaScript only, not Aura or Visualforce.

## 11. Reuse for similar issues

**How to recognise one:** a date that is right in an edit form and one day early in a list, summary, PDF or email, for Central Time users; or a default date that is tomorrow after 7pm.

- Ask whether the value is a Salesforce `Date` or `Datetime` before touching it in JavaScript. Only `Date` goes through `c/sfDate`.
- When a bug recurs after being written down, replace the rule with a shared helper and a check that blocks the merge.
- Make any check build its own list of what to look for from the repo, and fail when that list comes back empty.
- Any test about timezones must pin the timezone where it actually takes effect, and assert that the pin worked.
