# Invoice email template layout

| | |
|---|---|
| **Ticket** | DDT-119, DDT-515 |
| **Project** | Support tickets |
| **Status** | DDT-119 live since 2026-06-08; DDT-515 done 2026-07-27 (not found in production on 2026-11-05, see section 10) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-08 (DDT-119) and 2026-07-27 (DDT-515) |
| **Resolved by** | Support team |
| **Systems** | Salesforce, fw1 Payment Center (managed package): Visualforce email template, invoice PDF |
| **Area** | Billing: the invoice email and PDF sent to clients |
| **Related** | None recorded |
| **Code** | None. Both changes were made directly in the org; nothing was committed to the ddt-salesforce repo. |

---

## 1. Summary

The invoice that fw1 Payment Center emails to clients had layout problems: a font the PDF renderer could not load, centred and upper-cased column headers, the wrong field in the Item Description column, and no visible spacing between paragraphs of the terms. DDT-119 rebuilt the line-item table inside the invoice email template and fixed the font and spacing. DDT-515 then aligned the three PDF columns (Product, Item Description, Amount) with a CSS file wired into fw1's PDF stylesheet setting.

## 2. What was reported

- **DDT-119 (2026-06-08).** A request to clean up the classic invoice email template so client-facing invoices look on-brand and readable: layout, fonts, spacing, branding and merge fields. The ticket did not list specific defects; they were identified during the work (section 5).
- **DDT-515 (2026-07-27).** On the invoice PDF, the line-item table's three columns, **Product**, **Item Description** and **Amount**, did not line up: headers and values drifted out of sync.

Why it matters: the invoice goes straight to clients as a PDF attachment and inline email. A misaligned table makes it hard to tell which amount belongs to which line.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| fw1 invoice email template (Visualforce email template) | Email template | Montserrat font embedded; managed line-item component replaced with a custom table (DDT-119) |
| `fw1__Default_Terms_And_Conditions__c` on Payment Center Settings, and on existing invoices | Field values | Double line breaks between paragraphs (DDT-119) |
| `InvoicePdfCss` | Static resource (CSS) | Explicit text alignment per column (DDT-515) |
| `fw1__Invoice_Pdf_Css__c` on `fw1__Payment_Center_Setting__c` | Setting | The hook fw1 reads to load extra PDF CSS (DDT-515) |

**Out of scope**

- Moving to a Lightning email template. Both tickets raise it as a possible follow-up only.
- The fw1 package's own Visualforce pages (`EmailBatchInvoicePDF` and others). They are managed code and cannot be edited.

## 4. How it works (background)

```
Invoice is sent from fw1 Payment Center
  -> fw1 renders the invoice email template (Visualforce email template, fw1 namespace)
  -> the PDF attachment is rendered by fw1's managed page EmailBatchInvoicePDF
       -> which loads the CSS named in Payment Center Settings (fw1__Invoice_Pdf_Css__c)
  -> email and PDF go to the client
```

Two different surfaces control what the client sees:

- **The email template markup** controls the inline email body and the table structure.
- **The PDF page's CSS** controls how fw1's managed PDF page lays out its table. That page is managed code, so the only way to change its styling is the CSS hook.

## 5. Root cause

From the ticket comments:

1. **Font.** The template loaded Montserrat through a Google Fonts `@import`. Salesforce's PDF renderer does not support that, so the font never loaded.
2. **Header alignment.** The line items were drawn by the managed components `c:InvoiceLines` / `c:InvoiceLinesDynamic`, which force upper-case, centred headers. The template could not override them.
3. **Item Description.** The column did not show `fw1__Item_Description__c` per line.
4. **Paragraph spacing.** The default terms text used single line breaks, which the PDF collapses, so paragraphs ran together.
5. **PDF column drift (DDT-515).** The PDF is rendered by fw1's `EmailBatchInvoicePDF` page, whose layout is controlled by the CSS hook, **not** by the email template markup. Fixing the template alone could not align the PDF columns.

**The lesson.** The ticket's first guesses (missing `col` widths, mixed units) were about the template markup. The PDF was not drawn from that markup at all. Find out which component renders the output before editing anything.

## 6. Solution

### 6.1 Email template (DDT-119)

- **Font:** Montserrat Regular and Bold embedded directly in the template as base64-encoded WOFF data, uploaded through the Salesforce REST API, replacing the Google Fonts `@import`.
- **Line-item table:** the managed `c:InvoiceLines` / `c:InvoiceLinesDynamic` component was replaced with a custom `apex:repeat` table over the `fw1__Invoice_Lines__r` child relationship. In that table:
  - the **Product** header and the product name (`fw1__Product2__r.Name`) are left-aligned;
  - the **Item Description** header and cells are left-aligned and show `fw1__Item_Description__c` for each line.
- **Paragraph spacing:** `fw1__Default_Terms_And_Conditions__c` on Payment Center Settings, and on existing invoice records, was rewritten with double line breaks between paragraphs.

Why a custom table: the managed component's styles cannot be overridden from the template, so the only way to control alignment was to stop using it.

### 6.2 PDF column alignment (DDT-515)

`InvoicePdfCss.css` was updated and connected through the existing `fw1__Invoice_Pdf_Css__c` hook:

```css
/* Product: first column, flush left */
text-align: left !important;

/* Item Description: centre only the second header cell */
.tableHeader + .tableHeader { text-align: center !important; }

/* Amount: last column, right-aligned (already in place, confirmed) */
text-align: right !important;
```

Only the selector for the second header is recorded in full in the ticket. The adjacent-sibling selector targets the second header cell without affecting the first. `!important` is needed to beat fw1's own styles.

Why CSS and not markup: the PDF page is managed code. The CSS hook is the only supported way to change it.

## 7. Testing

- **DDT-119:** Not recorded in the ticket. The ticket asked for test sends to several email clients; no results are recorded.
- **DDT-515:** Not recorded in the ticket beyond "confirmed correct" for the Amount column.
- **Production read, 2026-11-05:** of the five fw1 invoice email templates, only `Invoice Template - Outstanding` shows a last-modified date of 2026-06-08, the day of the DDT-119 change. The other four were last modified between 2020 and 2024.

**Not covered:** rendering across email clients, long item descriptions, many line items.

## 8. Step by step

**Template changes (DDT-119)**

1. Identify which fw1 invoice email template is actually being sent. Several exist (`Invoice Template`, `- Batch`, `- Outstanding`, `- PDF Preview`, `- Receipt`).
2. Convert the font to base64 WOFF and embed it in the template's styles in place of any external `@import`.
3. Replace the managed line-item component with an `apex:repeat` over `fw1__Invoice_Lines__r`, with left-aligned Product and Item Description columns showing `fw1__Product2__r.Name` and `fw1__Item_Description__c`.
4. In Payment Center Settings, edit `fw1__Default_Terms_And_Conditions__c` to put a blank line between paragraphs. Existing invoices carry their own copy, so update those too if they must change.
5. Send a test invoice and open both the email and the PDF.

**PDF alignment (DDT-515)**

1. Upload the CSS as a static resource.
2. Point `fw1__Invoice_Pdf_Css__c` on Payment Center Settings at it. The exact value entered is not recorded in the ticket.
3. Send a test invoice and check that Product is left-aligned, Item Description centred in the header, and Amount right-aligned.

## 9. Rollback

- **Template:** restore the previous template body. No copy of it was committed to the repo, so keep one before editing.
- **Terms text:** the rewritten text was not backed up per invoice; the change is cosmetic.
- **PDF CSS:** clear `fw1__Invoice_Pdf_Css__c`. It is blank in production today (section 10).

## 10. Known limits

- **The DDT-515 CSS is not active in production today.** A read on 2026-11-05 found `fw1__Invoice_Pdf_Css__c` blank and no static resource named `InvoicePdfCss`. The ticket comment says it was deployed to "production"; whether it was later removed or applied to a different org is not recorded. Re-apply it before assuming the PDF columns are aligned.
- **Nothing is in git.** Neither change was committed to the ddt-salesforce repo, so the repo cannot be used to restore or compare them.
- **fw1 is a managed package.** A package upgrade can change the PDF page and its class names (such as `.tableHeader`), which would silently break the CSS.
- **The DDT Payments package** was expected to replace fw1 Payment Center, which would retire these templates.

## 11. Reuse for similar issues

- **Find what renders the output first.** For fw1 invoices, the email body comes from the email template and the PDF comes from a managed Visualforce page. They are fixed in different places.
- **Managed components you cannot style:** replace them with your own `apex:repeat` over the same child relationship rather than fighting their CSS.
- **Managed pages you cannot edit:** look for a CSS or settings hook (for fw1, `fw1__Invoice_Pdf_Css__c`).
- **PDF fonts:** Salesforce's PDF renderer ignores external font imports. Embed the font.
- **Record the change in git the same day.** A fix made only in Setup can disappear without anyone noticing, as the DDT-515 CSS shows.
