# Representation agreement not showing as sent or signed on the lead

| | |
|---|---|
| **Ticket** | DDT-491 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-23 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), DocuSign e-signature |
| **Area** | PI intake: converting a Lead to a hired client once the agreement is signed |
| **Related** | None recorded |
| **Code** | Commit `c66df3106` on `main` in the `ddt-salesforce` repo, 2026-07-23 (no PR number) |

---

## 1. Summary

A parent signed the representation agreement for a minor, but the Lead did not convert into a hired client, so staff thought the agreement had never been sent or signed. It had been sent and signed. The automatic conversion that runs on signing had failed silently, because the signer's name (the guardian, "as Personal Representative of" the minor) was longer than Salesforce's 40-character First Name field. The hire was completed by hand, and the conversion code was changed to take the client's name from the Lead instead of the signer field.

## 2. What was reported

On 2026-07-22 a staff member reported that an agreement sent to a parent on behalf of a minor did not show as sent or signed on the Lead, although the parent said she had signed. Later the same day they confirmed the agreement did show as signed, but the Lead had not converted into a hired PI client.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `HiredMatterConvertController` | Apex class | Person Account name now comes from the Lead, capped at 40 characters |
| `HiredMatterConvertControllerTest` | Apex test | New test `convertPiLead_OverlengthGuardianSignerName_SourcesNameFromLead` |
| The reported Lead | Data | Hire completed by hand: client, PI Matter, Lead converted |

**Out of scope**

- **Other places the signer name is used.** The form's name field still feeds the e-signature signer name; only the Person Account name was changed.

## 4. How it works (background)

```
Intake fills the convert form on the Lead
  (the form's name field doubles as the e-signature signer name)
  -> agreement sent for signature
  -> client signs
  -> HiredMatterConvertController.createPersonAccountFromForm builds the Person Account
  -> Matter created, Lead marked converted
```

For a minor or a represented party, the signer is the guardian, entered as "Name, as Personal Representative of Name, a minor". That text was also used as the client's name.

## 5. Root cause

1. `createPersonAccountFromForm` set the Person Account's First, Middle and Last name from the convert form's name fields.
2. Those fields double as the e-signature signer name, which for a minor is the guardian's long "as Personal Representative of" text.
3. The text was 44 characters, past the standard 40-character First Name limit, so the Account insert threw a "data value too large" error.
4. That rolled back the whole hire when the client signed. The only trace was a row in `API_Error_Log__c`; the user saw nothing.

**The ticket's first guess was wrong.** It asked whether the agreement had been sent at all. The executed PDF and its certificate were already in the Lead's Files.

## 6. Solution

### 6.1 Complete the stuck hire by hand

The client record and the PI Matter were created, the Lead was marked converted, and the conflict check came back clean.

### 6.2 Take the client's name from the Lead

```apex
private static final Integer PERSON_NAME_MAX = 40;

a.FirstName  = clientName(ld == null ? null : ld.FirstName,  c.firstName);
a.MiddleName = clientName(ld == null ? null : ld.MiddleName, c.middleName);
a.LastName   = clientName(ld == null ? null : ld.LastName,   c.lastName);

private static String clientName(String leadValue, String formValue) {
  String chosen = String.isNotBlank(leadValue) ? leadValue : formValue;
  if (String.isBlank(chosen)) {
    return chosen;
  }
  return chosen.length() > PERSON_NAME_MAX
    ? chosen.substring(0, PERSON_NAME_MAX)
    : chosen;
}
```

Why it's done this way:

- **The Lead is the client's identity**; the form's name is the signer's. Using the Lead puts the minor, not the guardian, on the client record.
- **Fall back to the form value** only when the Lead field is blank.
- **Hard-cap to 40** as a last backstop, so an over-long value can never again roll back a hire.

## 7. Testing

`convertPiLead_OverlengthGuardianSignerName_SourcesNameFromLead` fills the form with a 44-character guardian signer name and checks that the Person Account is created with the Lead's own name.

**Live check:** the manual hire was confirmed on the ticket on 2026-07-23. A query after a later minor hire is not recorded.

**Not covered:** whether the commit was deployed the same day is not recorded in the ticket.

## 8. Step by step

1. **Check the Lead's Files** for the executed agreement and certificate. If they are there, the agreement was signed.
2. **Check `API_Error_Log__c`** for a failed conversion on that Lead around the signing time.
3. **Complete the hire by hand** (client, Matter, convert the Lead) and run the conflict check.
4. **Change `createPersonAccountFromForm`** to use `clientName(...)` as in 6.2, and add the test.
5. **Validate, get approval, deploy.**

## 9. Rollback

Redeploy the previous `HiredMatterConvertController`. That brings back the silent failure for long signer names, so there is no reason to.

## 10. Known limits

- **A name over 40 characters on the Lead itself is cut off** at 40, not rejected.
- **Conversion failures are still only logged.** A failed automatic hire does not alert the user; it shows only in `API_Error_Log__c`.

## 11. Reuse for similar issues

**How to recognise one:** a signed agreement, but no converted Lead or new Matter, and no visible error.

- **Check `API_Error_Log__c` first.** Automated conversions log there and roll back silently.
- **Watch for fields that do two jobs.** A signer name is not a client name.
- **Look for length limits** when a value can carry extra wording, such as "as Personal Representative of".
- **Write a test with the exact failing value** from the error log.
