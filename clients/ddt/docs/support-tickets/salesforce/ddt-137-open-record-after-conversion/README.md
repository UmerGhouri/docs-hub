# Opening the converted record after lead conversion

| | |
|---|---|
| **Ticket** | DDT-137 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Lead conversion wizard on the Lead page |
| **Related** | DDT-134 (same conversion method, fixed the same day) |
| **Code** | No PR. The change reached `main` in the 2026-07-03 production backfill |

---

## 1. Summary

After converting a Lead, staff had to search for the new record by hand to keep working on it. The conversion wizard now opens the new record automatically: the Court Case (Opportunity) for a Court Case conversion, or the Matter for a PI conversion. The Apex already returned the new record's Id; the wizard simply hadn't used it.

## 2. What was reported

- **2026-06-09:** a staff member asked for the converted record to open automatically after conversion, because staff convert Leads often and lose time searching for the new record every time.
- **2026-06-10:** the support team deployed the change to production. The requester confirmed it worked the same day.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `newPotentialClient` | LWC | After `convertLead` returns, navigates to the new record; success message updated |

**Out of scope**

- **`NewPotentialClientController`.** Its `convertLead` method already returned the new record's Id, so no Apex change was needed.
- **Salesforce's standard Convert button.** The ticket asked for the behaviour on every conversion path; the change is in the custom wizard only.

## 4. How it works (background)

```
User fills in the conversion wizard on the Lead page (newPotentialClient LWC)
  -> calls NewPotentialClientController.convertLead(wrapper)
  -> Apex creates the Court Case (Opportunity) or the Matter
  -> Apex returns the new record's Id
  -> before: the wizard showed "Your Conversion process to ... In Progress" and stayed on the Lead
```

## 5. Root cause

Not a defect: a request for new behaviour. The wizard discarded the Id that `convertLead` returned, so there was nothing to navigate to.

## 6. Solution

The wizard keeps the returned Id and navigates to it with `NavigationMixin`:

```javascript
const typeLabel = this.selectedConversionType === 'CourtCase' ? 'Court Case' : 'Matter';
const targetObjectApiName = this.selectedConversionType === 'CourtCase' ? 'Opportunity' : 'Matter__c';

const resultId = await convertLead({ wrapper: { /* ... */ } });

this.dispatchEvent(new ShowToastEvent({
    title: 'Success!',
    message: `Your Conversion to ${typeLabel} is complete — opening the new record.`,
    variant: 'success'
}));

// Auto-navigate to the converted record (Opportunity for CourtCase, Matter for PI).
// In a console app this opens as a subtab; in standard apps it replaces the page.
if (resultId) {
    this[NavigationMixin.Navigate]({
        type: 'standard__recordPage',
        attributes: { recordId: resultId, objectApiName: targetObjectApiName, actionName: 'view' }
    });
}
```

Why it's done this way:

- **Navigate in the LWC, not in Apex.** Apex can't move the user's browser; the component that started the conversion is the right place.
- **`standard__recordPage`** lets Salesforce decide how to open it: a subtab in a console app, the same page in a standard app. The ticket asked whether it should be a new tab or the current one; this follows the app the user is in.
- **Only when `resultId` is set**, so a conversion that returns nothing leaves the user where they are instead of navigating to an empty page.
- **The success message changed** from "In Progress" to "complete — opening the new record", so the user knows why the page changes.

**How it reached git.** No commit or pull request references DDT-137. The navigation first appears in git in the 2026-07-03 commit that backfilled production's state into `main`; the version in the 2026-05-22 sync does not have it.

## 7. Testing

- **Live check:** the requester confirmed on 2026-06-10 that it works.
- **Jest test:** none. The `newPotentialClient` jest tests do not assert on navigation.
- **Not covered:** the standard Convert button, and conversions where `convertLead` returns no Id.

## 8. Step by step

1. **Find the component that starts the conversion** (here `newPotentialClient`) and the Apex method it calls.
2. **Check what the Apex returns.** If it doesn't return the new record's Id, change it to.
3. **Extend the component with `NavigationMixin`** if it isn't already, and keep the result of the Apex call.
4. **Navigate with `standard__recordPage`** to that Id, using the right object API name for each conversion type.
5. **Update the success message** so it says the new record is opening.
6. **Test both conversion types** in a sandbox, in the console app and a standard app.
7. **Get the change approved, deploy, and ask the requester to confirm.**

## 9. Rollback

Redeploy the previous version of `newPotentialClient`. Users go back to searching for the converted record. Nothing else depends on the navigation.

## 10. Known limits

- **Custom wizard only.** Any other conversion path needs the same change.
- **It replaces the Lead page in a standard (non-console) app**, so the user leaves the Lead behind.
- **No test protects it**, so a later edit to the component could remove it without anything failing.

## 11. Reuse for similar issues

- **"After I do X, I have to go and find the new record"** is usually a component throwing away an Id its Apex already returns. Check the return value first.
- **Use `NavigationMixin` with `standard__recordPage`** rather than building a URL, so console and standard apps both behave correctly.
- **Ship it through a PR**, so the change is in git as soon as it is in production.
