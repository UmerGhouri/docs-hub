# Export to Sheet on the PI Damages tab

| | |
|---|---|
| **Ticket** | DDT-569 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-18 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | PI matters: the Damages tab, and getting its figures into a spreadsheet |
| **Related** | DDT-724 (the date bug the filename date avoids) |
| **Code** | PR #181 in the ddt-salesforce repo |

---

## 1. Summary

A PI staff member keeps her own negotiation spreadsheet with formulas already in it, and was copying the Damages table into it line by line. She asked for an export. The Damages tab now has an **Export to CSV** button that downloads exactly the rows on the selected sub-tab plus the total row. It is built entirely in the browser from data the component already has, with amounts and dates written as plain values so a spreadsheet can sum and sort them.

## 2. What was reported

On 2026-07-28 the PI team asked for an "Export to Excel / Google Sheet" button on the Damages tab. It had to export the selected sub-tab only (All, Medical, Lost Wages, Personal Expense or Property), include every visible column and the total row, and produce at least an Excel file, with Google Sheets as a nice-to-have.

Asked why, the requester explained: she pastes the table into her own template, whose prefilled formulas work out the negotiation. Copying each line by hand took a good chunk of time per file, especially on cases with many providers and bills.

The request was approved on 2026-08-07, with the instruction to build a CSV download in the component rather than a Google Sheets export. Live 2026-08-18.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `damagesTable` | LWC | **Export to CSV** button next to *New Damages Record*; CSV built from the loaded rows |
| `damagesTable.test.js` | Jest test | New file, six tests (the bundle had none) |

**Out of scope**

- **A real Google Sheets export.** The firm's Drive connector is read-only, so writing a Sheet would mean going out through Zapier. That is real plumbing and a live dependency for a file the user edits on her own computer anyway. A `.csv` opens in Excel and imports into Sheets, which covers both asks.
- **An `.xlsx` file.** CSV opens directly in Excel without needing a library.
- **Apex, new queries, fields or permissions.** None were needed.

## 4. How it works (background)

```
PI Matter -> Damages tab (damagesTable LWC)
  -> the user picks a sub-tab
  -> DamagesTableController.getDamagesForMatter returns:
       rows       one page (25 rows) of Damages__c, already filtered by damage type on the server
       totals     computed on the server over the WHOLE filtered set
       totalCount how many rows match the filter in total
  -> the component formats the rows for display ($40,369.76, 8/10/26)
```

Because the server already filters by sub-tab, the rows the component holds **are** the rows on screen. An export built from them can't disagree with what the user is looking at.

## 5. Root cause

Not applicable: a feature request.

## 6. Solution

### 6.1 Export the raw values, not the display strings

The component formats amounts and dates for the screen. The export ignores those and reads the raw row data:

```
wrong:  "$40,369.76"   Excel reads it as text, and it won't sum
right:  40369.76
```

Dates go out as ISO calendar dates (`2026-08-10`), so they sort properly. If the export used the formatted strings, the requester's formulas wouldn't calculate, which would defeat the point.

### 6.2 Columns per sub-tab, matching the table

| Sub-tab | Columns |
|---|---|
| Medical | Provider, First Treatment, Last Treatment, Records Status, Records Received, Billed, Outstanding |
| Lost Wages | Employer, First Date Missed, Last Date Missed, Status, Claimed, Documented |
| Personal Expense | Vendor, Item, Date Incurred, Status, Claimed, Documented |
| Property | Vendor, Description, Date Incurred, Status, Claimed, Documented |
| All | Type, Provider / Employer / Vendor, Description, Claimed, Documented, Billed, Outstanding |

The last line of the file is the total row: the tab's total label in the first column and the matching total under each money column.

### 6.3 Quoting

Provider names often contain commas (a hospital name ending ", Inc."), so cells are quoted to the CSV standard (RFC 4180): any cell with a comma, quote or line break is wrapped in quotes, with embedded quotes doubled.

```js
function csvCell(value) {
    if (value === null || value === undefined) return '';
    const s = String(value);
    return /[",\n\r]/.test(s) ? '"' + s.replace(/"/g, '""') + '"' : s;
}
```

### 6.4 The filename

```
Damages-<Tab>-<matter record id>-<today>.csv
```

It carries the sub-tab and the matter so a folder of these is easy to navigate, and **deliberately not the client's name**: this file is meant to go to opposing counsel and into demand letters, and a filename is the part nobody checks.

Today's date comes from `todaySfDateString()` in `c/sfDate`, not `new Date().toISOString().slice(0, 10)`. The second form converts to UTC first, so after 7pm Central it gives tomorrow's date. That is the mirror of the DDT-724 bug, and the repo's `date-lint` check fails the build on it.

### 6.5 A guard the ticket didn't ask for

The component holds one page of 25 rows, but the totals are computed over the **whole** filtered set. On a tab with more than 25 rows, an export would ship rows that don't add up to its own total row, and that total would look authoritative. So when `totalCount` is larger than the rows loaded, the export **refuses** with a message instead of producing a partial file.

On 2026-08-17 the largest matter in production had 16 damages rows on All and 15 on Medical, so this doesn't fire today.

## 7. Testing

Six Jest tests in `damagesTable/__tests__/damagesTable.test.js`:

| Test | Proves |
|---|---|
| Amounts export as `40369.76`, and the file contains no `$` | 6.1 |
| Dates export as `2026-08-10` | 6.1 |
| A provider name with a comma is quoted | 6.3 |
| The total row is present with the tab's label and its sum | 6.2 |
| The filename carries the tab and no client name | 6.4 |
| The export refuses when `totalCount` is larger than the loaded page | 6.5 |

The full LWC suite passed (143 of 143), with ESLint and Prettier clean.

- **Live check:** the support team confirmed the button was live and asked the requester to try it on a real file and check the columns. Her confirmation is not recorded on the ticket.
- **Not covered:** opening the downloaded file in Excel or Sheets is a manual check, assigned to the requester.

## 8. Step by step

1. **Confirm the component already holds the filtered rows.** Here `getDamagesForMatter` filters by damage type on the server, so the rows in memory are the rows on screen.
2. **Define the columns per tab** with a function that reads the raw row value, never the display getter.
3. **Quote every cell** with the RFC 4180 rule in 6.3.
4. **Append the total row** from the same totals the table shows.
5. **Build the filename** from the tab, the record key and `todaySfDateString()`. Leave the client's name out.
6. **Refuse when the data in memory is not the whole set** (`totalCount` larger than the loaded rows).
7. **Write Jest tests** for raw amounts, raw dates, quoting, the total row, the filename and the refusal.
8. **Deploy the whole `damagesTable` bundle**, after checking its other files match production, since a bundle deploys as a unit.
9. **Ask the requester** to export the Medical tab and confirm the Billed column sums in Excel.

## 9. Rollback

Revert the `damagesTable` bundle. The button disappears, and nothing else in the component depends on the new code.

## 10. Known limits

- **More than 25 rows can't be exported.** The export refuses rather than truncating. If a matter ever passes 25 rows on a tab, the export needs to load every page first.
- **CSV only.** No formatting, no `.xlsx`, no direct Google Sheets.
- **The filename uses the matter's record id** because that is the only case key the component holds.

## 11. Reuse for similar issues

**How to recognise one:** "can I get this table into a spreadsheet?" for a table rendered by one of the firm's own components.

- **Export what's on screen** from the data the component already has. A separate Apex or report-based export filters again and eventually disagrees with the screen.
- **Write numbers and dates raw.** Formatted currency becomes text in Excel.
- **Quote every cell** to RFC 4180; real names contain commas.
- **Keep client names out of filenames** for anything that may leave the firm.
- **Check that the rows add up to the total** before exporting a page of a paged list.
- **Never take today's date from `toISOString()`** in LWC; use `c/sfDate`.
