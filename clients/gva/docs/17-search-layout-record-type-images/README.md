# Search Layout: Same-Size Record Type Bars for Household & Individual

| | |
|---|---|
| **Status** | Resolved; change set uploaded to Production 30 Oct 2026. Production deploy confirmation: TBD — not found in records (client asked about post-deploy steps on 2 Nov) |
| **Requested** | 28 Oct 2026 (follow-up 28 Oct, approval 30 Oct) |
| **Delivered** | 28 Oct 2026 (sandbox), 30 Oct 2026 (change set uploaded) |
| **Objects** | Account (labelled **Household & Individual**) |
| **Type** | Formula field change + 5 new static resource images. No Apex. |
| **Change set** | `Household Record Type Bars` |
| **Test classes** | None (no Apex) |

---

## 1. Summary

In global search results, both Financial Accounts and Households & Individuals show a **Record Type** column with a coloured bar image. The Financial Account bars are 190×50; most Household & Individual bars were shown at 190×80 because the Account formula forced that size. We made five new 190×50 bar images as static resources and pointed the Account `Record_Type_Color__c` formula at them, with no forced size. The Financial Account formula was not changed.

---

## 2. Client requests (as received)

**Original (28 Oct)**
> Could you please resize the Household & Individual record type color coded blocks on the search layout so that they are the same size as the Financial Accounts ones?

**Follow-up (28 Oct), with a Production screenshot**
> are these the same width as the ones in production for investments? the investments ones seem smaller in production in terms of width. Please see screenshot attached.

**Approval (30 Oct)**
> Understood, could you please prepare the changeset?

**Post-deploy question (2 Nov)**
> Is there anything post deployment that needs to be done or does it automatically update?

---

## 3. Scope

**In scope**
- `Account.Record_Type_Color__c` (label **Record Type**, Text formula using `IMAGE()`)
- Five record types: Household (GVA), Individual, Advisor, Vendor, Vendor Household
- Five new static resources (190×50 PNG)

**Out of scope**
- `Investments__c.Record_Type_Color__c` (Financial Account version — already 190×50, unchanged)
- Professional Relationship Household and Business Household (already 190×50, unchanged)
- The record page header banners (`recordHeaderColor`, `investment_Header_Recordtype`) — first suspected, but not what the client meant
- The Financial Account Annuity bar (`Orange_Bar`, 302×108) — off-size, but not part of this request

---

## 4. Background: what "search layout colour blocks" are

- The first guess was the coloured banners at the top of record pages. They are different components with different CSS, but they are not on the search layout.
- Both objects' **Search Layouts** (Setup → Object Manager → *object* → **Search Layouts**, not Page Layouts) include the formula field **Record Type** (`Record_Type_Color__c`). It uses `IMAGE("/resource/...")` per record type. Those images are the "colour coded blocks".
- Where users see them: type in the global search box, press **Enter**, then pick **Households & Individuals** or **Financial Accounts** on the left. They also show in the hover preview.

---

## 5. Root cause

The two formulas size the images differently.

**Financial Account** — no size, so each image shows at its real size (Green_Bar, Yellow_bar, Red = 190×50).

**Household & Individual — before:**

```
CASE(
    RecordType.Name,
    "Household (GVA)", IMAGE("/resource/Blue_bar", "Household",80,190),
    "Individual", IMAGE("/resource/Light_Grey_Bar", "Individual",80,190),
    "Advisor", IMAGE("/resource/Medium_Blue_bar", "Advisor",80,190),
     "Professional Relationship Household", IMAGE("/resource/Dark_Gray","Professional Relationship Household"),
     "Business Household", IMAGE("/resource/Light_Blue","Business Household"),
     "Vendor Household", IMAGE("/resource/Vendor","Vendor Household"),
     "Vendor", IMAGE("/resource/Grey_Bar","Vendor",80,190),
    RecordType.Name
)
```

| Record type | Old image | Real size | Shown at |
|---|---|---|---|
| Household (GVA) | `Blue_bar` | 192×95 | 190×80 (forced) |
| Individual | `Light_Grey_Bar` | 192×95 | 190×80 (forced) |
| Advisor | `Medium_Blue_bar` | 192×95 | 190×80 (forced) |
| Vendor | `Grey_Bar` | 262×109 | 190×80 (forced) |
| Vendor Household | `Vendor` | 190×80 | 190×80 |
| Professional Relationship / Business | `Dark_Gray` / `Light_Blue` | 190×50 | 190×50 (already right) |

---

## 6. Solution design

### Why new images instead of just changing `80,190` to `50,190`

The text is drawn into the images. Squashing a 192×95 image to 190×50 flattens the text (worst for "Individual", "Vendor" and the two-line "Vendor Household").

_Screenshot in the private copy: squashed-vs-new.png (Investment target, old image squashed to 50, new image)._

So we generated new 190×50 bars that keep each record type's colours and wording. A PowerShell `System.Drawing` script sampled the background and text colours from each old image and drew the label in Times New Roman, 19 px, to match the green Investment bar. The old images were left in the org untouched.

_Screenshot in the private copy: new-bars-vs-investment.png (New bars compared with the Investment bar (top))._

### Components

| Type | Name | Notes |
|---|---|---|
| Static Resource | `RT_Household_GVA` | 190×50 PNG, replaces `Blue_bar` |
| Static Resource | `RT_Individual` | 190×50 PNG, replaces `Light_Grey_Bar` |
| Static Resource | `RT_Advisor` | 190×50 PNG, replaces `Medium_Blue_bar` |
| Static Resource | `RT_Vendor` | 190×50 PNG, replaces `Grey_Bar` |
| Static Resource | `RT_Vendor_Household` | 190×50 PNG, replaces `Vendor` |
| Custom Field | `Account.Record_Type_Color__c` (existing, **edited**) | Points at the new images; forced size removed |

All five static resources use **Cache Control = Public**.

Images: Household (GVA) · Individual · Advisor · Vendor · Vendor Household · reference Green_Bar

### Formula after

```
CASE(
    RecordType.Name,
    "Household (GVA)", IMAGE("/resource/RT_Household_GVA", "Household"),
    "Individual", IMAGE("/resource/RT_Individual", "Individual"),
    "Advisor", IMAGE("/resource/RT_Advisor", "Advisor"),
     "Professional Relationship Household", IMAGE("/resource/Dark_Gray","Professional Relationship Household"),
     "Business Household", IMAGE("/resource/Light_Blue","Business Household"),
     "Vendor Household", IMAGE("/resource/RT_Vendor_Household","Vendor Household"),
     "Vendor", IMAGE("/resource/RT_Vendor","Vendor"),
    RecordType.Name
)
```

### Visibility
Static resources have no profile settings; any logged-in user can load them. Who sees the bar is still controlled by the field's FLS, which was not changed.

---

## 7. The "Investments look narrower in Production" follow-up

The client's Production screenshot showed the green Investment bar narrower than the new Household bars. Cause: **same image, narrower column.** Both are 190×50, but the search results table shrinks images to fit the column, keeping their shape. The Financial Accounts results table has more columns (10 vs 9), so its **Record Type** column is narrower.

| Results table | Record Type column | Bar rendered at (sandbox screenshot) |
|---|---|---|
| Households & Individuals (9 columns) | wider | about 115×30 |
| Financial Accounts (10 columns) | narrower | about 83×21 |

Widening the Financial Accounts Record Type column makes them match. In the hover preview both show at full 190×50. Message sent to the client:

> They're the same size image (190×50). The Investment bar only looks narrower because the Financial Accounts results table has more columns, so Salesforce shrinks the image to fit its narrower "Record Type" column. Widening that column makes it the same as the Household bars

The alternative (fixed size like `IMAGE(..., 26, 100)` in **both** formulas) was offered but not needed; the client accepted the explanation and asked for the change set.

---

## 8. How to test in the sandbox

1. Hard-refresh (Ctrl+F5) so the browser doesn't show cached images.
2. Type a search term in the global search box and press **Enter**.
3. Open **Households & Individuals** results and **Financial Accounts** results. Compare the **Record Type** bars, and hover over a Household and a Financial Account record to compare the previews.
4. Check each record type if possible: Household (GVA), Individual, Advisor, Vendor, Vendor Household.

Sandbox records used: a test Household, an Individual, and a test Financial Account.

_Screenshot in the private copy: 01-household-search-results.png (Household search results with new bars)._
_Screenshot in the private copy: 02-financial-account-search-results.png (Financial Account search results for comparison)._

---

## 9. Deployment (change set)

**Change set name:** `Household Record Type Bars`

| Component Type | Name |
|---|---|
| Static Resource | `RT_Household_GVA` |
| Static Resource | `RT_Individual` |
| Static Resource | `RT_Advisor` |
| Static Resource | `RT_Vendor` |
| Static Resource | `RT_Vendor_Household` |
| Custom Field | Household & Individual › **Record Type** (`Account.Record_Type_Color__c`) |

- In the change set picker there are two fields labelled **Record Type**. Pick the one whose Parent Object is **Household & Individual**, not **Financial Account**. (The picker shows the parent as `Household &amp;amp; Individual` — a display bug, it's the right one.)
- The field is listed under **R** (its label), not H.
- Images and field must go in the **same** change set. If the images are missing, the formula saves but shows broken images.
- Do **not** use *View/Add Dependencies*.
- No Apex, so no tests to run.

**In Production:** Inbound Change Sets → **Validate** → **Deploy**. If Production isn't in the upload list: Production → Setup → **Deployment Settings** → (the sandbox) → **Allow Inbound Changes**.

_Screenshot in the private copy: 03-change-set-components.png (Change set components)._

---

## 10. Post-deployment

Nothing to do. Answer given to the client on 2 Nov:

> Nothing else is needed after the deployment. It updates automatically on all existing records. If you still see the old bars, refresh the page (Ctrl+F5).

`Record_Type_Color__c` is a formula, so every record shows the new image as soon as the deploy finishes. No data update and no permission changes.

---

## 11. Lessons and gotchas

- **"Search layout" is literal.** Search Layouts are under Object Manager → *object* → Search Layouts, not Page Layouts. Confirm with a screenshot of where the client is looking before touching anything.
- **Don't resize images that contain text** through `IMAGE()` height/width. Make a correctly sized image instead.
- **Image size in a results table depends on column width**, not only on the image. Two identical images can look different in two tables.
- **Duplicate field labels across objects** (`Record Type` on Account and on Investments) make change set selection error-prone. Check the Parent Object column.
- **Formula fields need no backfill**; they re-evaluate on read.

---

## 12. Seeing a similar issue?

- Coloured record type bars look different sizes → compare the `IMAGE()` calls in each object's `Record_Type_Color__c` and the real pixel size of each static resource.
- Bar looks smaller only in the results table → it's the column width; drag the column or set a fixed size in both formulas.
- Need a new bar → generate a 190×50 PNG in the same style and add it as a Public-cache static resource.

## Files

- `Account/fields/Record_Type_Color__c.field-meta.xml`
- `Investments__c/fields/Record_Type_Color__c.field-meta.xml` (reference, unchanged)
- Static resources: `RT_Household_GVA`, `RT_Individual`, `RT_Advisor`, `RT_Vendor`, `RT_Vendor_Household`
