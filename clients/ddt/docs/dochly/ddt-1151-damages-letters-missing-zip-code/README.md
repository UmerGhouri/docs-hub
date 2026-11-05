# Damages letters print the state twice and no ZIP code

| | |
|---|---|
| **Ticket** | DDT-1151 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-30 |
| **Priority** | Medium |
| **Reported by** | Support team, 2026-09-29 (found during DDT-993) |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor) |
| **Area** | PI: records and bills requests to medical providers |
| **Related** | DDT-993 (provider department contacts, where this was found) |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

All six active Dochly templates on `Damages__c` built the provider's address with the state code where the ZIP code belongs, so the letters were expected to print the state twice and no ZIP. One merge field was changed in each of the six templates.

## 2. What was reported

The problem was found by reading the template bodies during DDT-993, not from a user report. The address line in every affected template was:

```
{{Provider__r.BillingStreet}}
{{Provider__r.BillingCity}}, {{Provider__r.BillingState}} {{Provider__r.BillingStateCode}}
```

| | Address line |
|---|---|
| Expected | `Chicago, Illinois 60601` |
| Likely actual | `Chicago, Illinois IL` |

The ticket says this was **not yet confirmed by generating a letter**. It was based on the merge tokens: none of the six template bodies (`dochly__DH_Template__c.dochly__Body__c`) contained `BillingPostalCode`.

The bug already existed before DDT-993 and was not caused by it, so it got its own ticket instead of going into the DDT-993 pull request.

## 3. Scope

**In scope**

| Template | Object | Change |
|---|---|---|
| Records only | `Damages__c` | `BillingStateCode` replaced with `BillingPostalCode` |
| Bills only | `Damages__c` | Same |
| Hospital Specific | `Damages__c` | Same |
| Incomplete notice | `Damages__c` | Same |
| Follow-up | `Damages__c` | Same |
| R&B Requests | `Damages__c` | Same |

**Out of scope**

- **"PI: Follow up" on `Record_Request__c`.** It merges `{{Treatment_Provider__r.ShippingPostalcode}}` with a lower-case "c", while the field is `ShippingPostalCode`. Whether Dochly resolves the token regardless of case was unverified, and whether it was changed is not recorded in the ticket.

## 4. How it works (background)

```
Staff generate a request letter from a Damages record
  -> Dochly reads the template body
  -> each merge token is replaced with the matching field from the record
     (Provider__r.* reads the provider Account's billing address)
  -> the letter is produced
```

A merge token that names a real field always resolves. So a token pointing at the *wrong* real field produces no error and no blank: it prints a plausible-looking wrong value.

## 5. Root cause

The last token on the city line was `{{Provider__r.BillingStateCode}}` instead of `{{Provider__r.BillingPostalCode}}`. `BillingState` already prints the state name, so the state appeared twice and the ZIP never appeared.

**Evidence.** None of the six template bodies contained the text `BillingPostalCode` when read on 2026-09-29.

How the wrong token got into all six templates is not recorded in the ticket.

## 6. Solution

In Dochly's template editor, one token was changed in each of the six templates:

```
Before:  {{Provider__r.BillingCity}}, {{Provider__r.BillingState}} {{Provider__r.BillingStateCode}}
After:   {{Provider__r.BillingCity}}, {{Provider__r.BillingState}} {{Provider__r.BillingPostalCode}}
```

There is no metadata and no deploy for a Dochly template edit, and nothing shows up in git. That is why the ticket asked for the list of changed templates to be recorded on it.

## 7. Testing

The ticket's plan was to generate one letter **before and after** the fix, for a provider with a ZIP code on file, and attach both. Whether that was done is not recorded in the ticket; the closing comment says only that all six templates were fixed.

**Not covered:** the "PI: Follow up" case-sensitivity question (section 3).

## 8. Step by step

1. **Generate one letter** from any affected Damages template for a provider with a ZIP code on file. Confirm the state prints twice.
2. **Open each of the six templates** in Dochly's template editor.
3. **Find the city line** and replace `{{Provider__r.BillingStateCode}}` with `{{Provider__r.BillingPostalCode}}`.
4. **Save** each template.
5. **Generate the same letter again** and confirm the ZIP code prints.
6. **Record the list of templates changed** on the ticket, since nothing appears in git.

## 9. Rollback

Not applicable: reverting would restore the wrong token.

## 10. Known limits

- **A provider with no ZIP on file** still prints nothing after the state.
- **Other templates** that build an address by hand may have the same mistake. Only the six `Damages__c` templates were checked.

## 11. Reuse for similar issues

**How to recognise one:** a generated document shows a value that is real but in the wrong place, such as the state where the ZIP should be.

- **Search the template bodies, not the output.** Reading `dochly__Body__c` for a field name finds every template that does or does not merge it, without generating anything.
- **Copy-paste spreads mistakes.** When one template has a wrong token, check every template on the same object.
- **Check token spelling against the field's API name exactly**, including case, when the merge engine's behaviour on case is unknown.
- **Record template edits on the ticket.** They leave no trace in git.
