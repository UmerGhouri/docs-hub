# Record Header Colour Banner on Mobile (`recordHeaderColor` form factors)

| | |
|---|---|
| **Status** | Fixed and deployed in the sandbox on 2 Sep 2026. Production deploy: TBD (not found in records). Phone page activation: **not done** (left for the client to decide) |
| **Requested** | Late Aug 2026 (Slack). Follow-up received 2 Sep 2026 |
| **Delivered** | 2 Sep 2026 (sandbox) |
| **Objects** | Account (Household, Individual, Business, Advisor, Professional Relationship, Vendor pages), `Investments__c` (Financial Account pages) |
| **Type** | LWC metadata only (`js-meta.xml`). No JS, HTML or Apex change |
| **Change set** | TBD (not found in records) |
| **Test classes** | None. No Apex |

---

## 1. Summary

Two small LWCs show a coloured banner at the top of a record page, based on its record type:

- `recordHeaderColor` on the **Account** pages (Household, Individual and so on)
- `investment_Header_Recordtype` on the **Financial Account** pages

The client asked for the banner to work on mobile. An earlier change made the Financial Account banner phone-capable, but missed the Household/Individual one, which still said **"Supported form factors: desktop"** in Lightning App Builder. We added the same phone support to `recordHeaderColor`.

**Important:** this makes the component *allowed* on phone pages. The banner only shows in the Salesforce mobile app once the record pages are **activated for Phone**. They weren't, for either object (see section 10).

---

## 2. Client requests (as received)

**First ask (Slack):**
> Could you please make the record header color available for mobile use as well?

**After the first fix (Slack, with two App Builder screenshots):**
> It looks like the investment record types are available on both but the Household & Individual ones are desktop only. Could you please make sure that both are available on the desktop and mobile app?

The screenshots showed the App Builder palette tooltips:
- `investment_Header_Recordtype`: *Supported form factors: desktop and phone*
- `recordHeaderColor`: *Supported form factors: desktop*

_Screenshot in the private copy: 01-tooltip-desktop-only.png (Client screenshot: recordHeaderColor desktop only)._

---

## 3. Scope

**In scope**
- Add phone (`Small`) form-factor support to `recordHeaderColor`
- Get the earlier org-side fix to `investment_Header_Recordtype` into source

**Out of scope**
- Activating the Account and Financial Account record pages for **Phone**. This is an org-wide change for every mobile user, so we left the decision to the client.
- Banner colours and labels (JS, HTML, CSS unchanged)

---

## 4. Background: where "Supported form factors" comes from

The tooltip in App Builder comes from `<supportedFormFactors>` inside the bundle's `js-meta.xml`. When it's **omitted**, a `lightning__RecordPage` target defaults to **Large (desktop) only**. That was the symptom.

The repo was **behind the org**. The phone fix to `investment_Header_Recordtype` had been made directly in the sandbox (last modified 31 Aug 2026) and never retrieved. `supportedFormFactors` appeared 0 times in `force-app`. Editing from the stale repo and deploying would have **undone** that fix, so we retrieved first.

---

## 5. Root cause

| Component | Used on | Before | Cause |
|---|---|---|---|
| `investment_Header_Recordtype` | `Investment_Record_Page`, `Annuity_Page`, `EmployerPlan_Page` | Desktop + phone (fixed in the org) | n/a, already fixed |
| `recordHeaderColor` | `Client_Household_Record_Page_Household`, `_Individual`, `_Business`, `_Advisor`, `ProfessionalRelationship`, `Vendor`, `Vendor1` | **Desktop only** | No `targetConfigs` / `supportedFormFactors` in `recordHeaderColor.js-meta.xml` |

---

## 6. Solution design

One file changed: `recordHeaderColor.js-meta.xml`. We added the same block the investment component already had:

```xml
<targetConfigs>
    <targetConfig targets="lightning__RecordPage">
        <supportedFormFactors>
            <supportedFormFactor type="Large"/>
            <supportedFormFactor type="Small"/>
        </supportedFormFactors>
    </targetConfig>
</targetConfigs>
```

- The config is scoped to `lightning__RecordPage` only. `lightning__Inbox` stays in `<targets>` untouched, because it doesn't take a form-factor config. `Small` isn't valid on every target either, so putting it on the other targets could fail the deploy.
- The retrieve also removed a stale, wrongly-cased CSS file in the investment bundle (`investment_Header_RecordType.css`, capital T). LWC requires an exact name match, so that file was never loaded. It was fully commented out anyway.

---

## 7. How to test in the sandbox

**Test A: the component (pass/fail for this fix)**
1. Setup → Object Manager → **Account** → Lightning Record Pages → *Client / Household Record Page - Household* → **Edit**.
2. In the left palette under **Custom**, hover `recordHeaderColor`. The tooltip reads **"Supported form factors: desktop and phone"**.
3. Switch the App Builder preview from **Desktop** to **Phone**. The banner renders and isn't greyed out.
4. Repeat on *Client / Household Record Page - Individual*. Spot-check *Investment Record Page* to confirm both match.

_Screenshot in the private copy: 02-tooltip-desktop-and-phone.png (Tooltip after the fix)._
_Screenshot in the private copy: 03-phone-preview.png (Phone preview in App Builder)._

**Test B: the real mobile app (only after Phone activation, see section 10)**
1. In the Salesforce mobile app, log in to the sandbox via **Use Custom Domain**.
2. Open a Household and an Individual. Check the banner colour: blue `#2196f3` for `Household_GVA`, light grey `#d3d3d3` for `Individual`, dark navy `#001f3f` for `Advisor`. Red `#b40d00` means the record type isn't in the component's list.
3. Browser device emulation is only a rough check. It isn't a true phone form factor.

_Screenshot in the private copy: 04-mobile-app-banner.png (Banner in the mobile app)._

---

## 8. Test classes and coverage

Not applicable. This is metadata only, with no Apex.

---

## 9. Deployment

Sandbox (2 Sep 2026): a dry-run deploy succeeded, then the real deploy succeeded. A retrieve afterwards confirmed the org copy contains the new block.

**Production change set name:** TBD (not found in records).

| Component Type | Name |
|---|---|
| Lightning Web Component Bundle | `recordHeaderColor` |
| Lightning Web Component Bundle | `investment_Header_Recordtype` (only if Production doesn't already have its phone fix) |

No test classes are needed.

**Rollback:** delete the `<targetConfigs>` block and redeploy.

---

## 10. Post-deployment: activate the pages for Phone

When we checked on 2 Sep 2026, the `View` override for the **Small** form factor was `Default` on both `Account` and `Investments__c`. The apps `Great_Valley_Advisors_FA_Lightning` and `GVA_System_Admin` had **no Phone assignments** for these pages, only Desktop ones. So the mobile app showed Salesforce's default page, with **no banner on either object**, including Financial Accounts.

To make the banner appear on phones:
1. Lightning App Builder → open the record page → **Activation…** → **Phone** tab
2. Assign it as org default, or app default for the relevant apps → **Save**
3. Repeat for the Household, Individual and Investment pages as needed

> The repo's app metadata can be stale. Confirm the current assignments in App Builder → Activation before relying on the counts above.

_Screenshot in the private copy: 05-phone-activation.png (Phone activation)._

Status of this step: TBD (not found in records). On 2 Sep we drafted a reply saying one more setting was needed and asking the client to confirm. Whether it was sent, and whether the pages were activated, isn't recorded.

---

## 11. Lessons and gotchas

- **"Available on phone" in App Builder ≠ "shows in the mobile app".** The component needs `Small` support **and** the page needs a Phone activation.
- **Retrieve before you edit.** Someone had fixed the sibling component directly in the org. Deploying from the stale repo would have reverted it.
- **Scope `targetConfigs` to the target you mean.** Not every target accepts `Small` or form factors (Home Page, Inbox).
- **Check case-sensitive file names in LWC bundles.** A mis-cased CSS file is silently ignored.

---

## 12. Seeing a similar issue?

- App Builder tooltip says *desktop* only → add a `targetConfig` for `lightning__RecordPage` with `Large` + `Small`.
- The component is phone-capable but doesn't appear in the mobile app → the record page isn't activated for Phone (App Builder → Activation → Phone).
- Related: the New Investment Task button on mobile is a different mobile issue (quick action / URL button), covered in [05](../05-new-investment-task-mobile/README.md).

## Files

- `recordHeaderColor.js-meta.xml` (the change)
- `recordHeaderColor.js` (colours and labels per record type)
- `investment_Header_Recordtype.js-meta.xml` (the reference fix)
- Pages: `Client_Household_Record_Page_Household.flexipage-meta.xml`, `Client_Household_Record_Page_Individual.flexipage-meta.xml`, `Investment_Record_Page.flexipage-meta.xml`
