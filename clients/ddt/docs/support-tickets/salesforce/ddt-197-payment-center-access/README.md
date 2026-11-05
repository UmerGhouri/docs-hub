# Granting Payment Center licence and payment access

| | |
|---|---|
| **Ticket** | DDT-197, DDT-858, DDT-1410 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-26 |
| **Priority** | High (DDT-197), Medium (DDT-858, DDT-1410) |
| **Reported by** | Case Manager, Intake team and Firm staff, 2026-06-18 to 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Payment Center managed package, package licences, permission sets) |
| **Area** | Payments: viewing and taking client payments |
| **Related** | None |
| **Code** | None. User access changes made in Setup. |

---

## 1. Summary

Payments in Salesforce run on the Payment Center managed package (namespace `fw1`). A user needs two things to use it: a **package licence** (a per-user seat) and **permission sets** that grant access to its objects. Three staff members were missing access. In the two cases where the fix is recorded, the missing piece was the package licence, which was assigned from Setup.

## 2. What was reported

- **DDT-197 (2026-06-18).** A Case Manager could not use **Pay Invoice** to send a payment. Opening the payment page showed "License Required: The Visualforce Page Payment is part of the AppExchange Package Payment Center, and requires a license to use." The licence was assigned within minutes; after a hard refresh the user confirmed it worked.
- **DDT-858 (2026-08-27).** An intake team member saw the **Payments** related list on a Court Case as blank and asked for access to view payments. Access was granted on 2026-08-28 and the user was asked to log out and back in.
- **DDT-1410 (2026-10-23).** A manager asked for a staff member to be able to view and process payments, without more access than needed. On 2026-10-26 the staff member was assigned a Payment Center licence and could then view and process payments.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Payment Center (`fw1`) package licence | Managed package licence | Assigned to the user (DDT-197, DDT-1410) |
| User access to Payments | Licence and/or permission set | Granted (DDT-858; method not recorded) |

The package's permission sets, for reference (labels and API names):

| Label | API name | Description in the org |
|---|---|---|
| Payment Center Read Only | `fw1__Payment_Center_Read_Only` | None (package permission set) |
| Payment Center Edit Access | `Payment_Center_Edit_Access` | "Installments and Payments edit access only." (firm-built, not part of the package) |
| Payment Center All Access | `fw1__Payment_Center_All_Access` | None (package permission set) |
| Payment Center Upgrade Permissions | `fw1__Payment_Center_Upgrade_Permissions` | Permissions so that upgrading users do not hit permission errors |
| Access Merchant Payments | `AccessMerchantPayments` | Salesforce's standard merchant-payment features, not part of the package |

**Out of scope**

- **Payments Administrator or All Access for new users.** DDT-1410 asked explicitly not to grant these unless needed.
- **Buying more seats.** DDT-197 noted that if all seats were used, an inactive user's seat could be reassigned or another bought. Neither was needed.

## 4. How it works (background)

```
User opens a payment page or the Payments related list
  -> Salesforce checks: does this user hold a Payment Center package licence?
       no  -> package Visualforce pages show "License Required";
              package objects behave as if they do not exist for this user
  -> then checks permission sets: can this user read / edit the package objects?
       no  -> related lists and tabs show nothing
```

Two layers, and they fail differently:

- **No licence:** the package's own pages show the "License Required" error, and the user cannot query or see the package's records at all, whatever their permission sets say.
- **Licence but no permission set:** pages open, but objects and fields the user is not granted are hidden.

Licences are a fixed number of seats. On 2026-11-05 the package had **33 seats, 32 in use**.

## 5. Root cause

- **DDT-197 and DDT-1410:** the user had no Payment Center package licence. In both cases the user already held the Payment Center permission sets from an earlier date (read from production on 2026-11-05: assigned 2026-03-31 for the DDT-197 user and 2026-10-12 for the DDT-1410 user, both before the ticket). The licence was the missing piece. The "License Required" error in DDT-197 names it directly.
- **DDT-858:** not recorded in the ticket. A blank Payments related list fits either a missing licence or a missing permission set, and the ticket does not say which was granted.

## 6. Solution

The package licence was assigned in Setup:

**Setup → Installed Packages → Payment Center → Manage Licenses → Add Users**, then select the user.

For DDT-858 the support team recorded only that access was granted; whether that was a licence, a permission set or both is not recorded. DDT-1410's requester suggested `Payment_Center_Edit_Access` and possibly `AccessMerchantPayments`; the recorded fix was the licence alone, because the user already held the package permission sets.

## 7. Testing

- **DDT-197:** the user did a hard refresh and confirmed it was fixed.
- **DDT-858:** the user was asked to log out and back in and check the Payments tab on the example case; they replied with thanks. Whether they confirmed they could see the payment is not recorded.
- **DDT-1410:** production on 2026-11-05 shows a Payment Center package licence for the user, created 2026-10-26, the day the ticket closed.

**Not covered:** no recorded test of the user actually completing a payment.

## 8. Step by step

1. **Read the error.** "License Required" on a Payment Center page means the package licence. A blank related list with no error may be either layer.
2. **Check the licence.** Setup → Installed Packages → Payment Center → Manage Licenses, and look for the user. Or query:

    ```sql
    SELECT User.Name, CreatedDate
    FROM UserPackageLicense
    WHERE PackageLicense.NamespacePrefix = 'fw1'
    ```

3. **Check seats.** If none are free, ask who approves a reassignment from an inactive user or a purchase:

    ```sql
    SELECT AllowedLicenses, UsedLicenses
    FROM PackageLicense
    WHERE NamespacePrefix = 'fw1'
    ```

4. **Check permission sets.** Confirm the user holds the read or edit permission set that matches what they need. Grant the least access that does the job, as DDT-1410 asked.
5. **Assign what is missing**: Manage Licenses → Add Users for the licence; the user's Permission Set Assignments for permission sets.
6. **Ask the user to hard-refresh, or log out and back in**, then open the payment page or Payments related list that failed.
7. **Record in the ticket exactly what was assigned.**

## 9. Rollback

Remove the user from Manage Licenses, or remove the permission set assignment. Safe: no data changes. Note that removing the licence also removes the user's access to every Payment Center record.

## 10. Known limits

- **Seats run out.** With 32 of 33 seats used, the next request may need a seat freed or bought.
- **DDT-858's change cannot be read back now.** That user's current access no longer shows the grant, so what was assigned is known only from the ticket.

## 11. Reuse for similar issues

**How to recognise one:** any "License Required" message naming an AppExchange package, or a managed package's tab or related list that is empty for one user and full for another.

- **Managed packages have two layers: licence, then permissions.** Check the licence first; permission sets do nothing without it.
- **Check what the user already has** before granting more. In two of these three tickets the permission sets were already in place.
- **Grant the least access** that meets the need, and say in the ticket what you granted.
- **Ask for a hard refresh or a fresh login** after the change, since cached sessions can keep showing the old access.
