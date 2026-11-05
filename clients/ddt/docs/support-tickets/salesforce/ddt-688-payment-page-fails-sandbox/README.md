# Payment page failing to load

| | |
|---|---|
| **Ticket** | DDT-688 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-17 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Visualforce page in the managed billing package, `fw1` namespace) |
| **Area** | Taking a client payment |
| **Related** | None |
| **Code** | None. No change was made in Salesforce. |

---

## 1. Summary

A staff member could not process a client payment: after adding the payment and clicking Process, the `Payment` page showed a browser error saying the page might be down or might have moved. The ticket first assumed the page was pointing at a sandbox called "fw1".

`fw1` is the namespace of the managed billing package, not a sandbox, and the `Payment` page belongs to that package. Support concluded the problem was in the managed package, not in Salesforce configuration we control. No change was made, and the ticket was closed asking the reporter to say if it happened again.

## 2. What was reported

On 2026-08-10, a member of the intake team added a payment for one client and clicked Process. Instead of the payment page, the browser showed: "The webpage ... might be temporarily down or it may have moved permanently to a new web address." The page address was the `Payment` Visualforce page on a host containing `fw1`.

It was urgent: the payment was needed that same day.

## 3. Scope

**In scope:** the `Payment` Visualforce page the billing package opens when processing a payment.

**Out of scope:** any change to the package, its page, or Salesforce configuration. None was made.

## 4. How it works (background)

```
Staff member adds a payment and clicks Process
  -> Salesforce opens the billing package's Payment Visualforce page
       (served from a host that includes the package namespace, fw1)
  -> the package takes the payment
```

The billing package is installed as a managed package. Its objects carry the `fw1__` prefix in this org (for example `fw1__Payment__c` and `fw1__Invoice__c`). Its `Payment` page is not in the `ddt-salesforce` repo, because managed-package code belongs to the vendor and cannot be retrieved or edited.

## 5. Root cause

**The ticket's first guess was wrong.** It read `fw1` in the page address as the name of a sandbox, and suggested a button was pointing at a sandbox instead of production. But `fw1` is the managed billing package's namespace, which is why it appears in the address of that package's pages. The page was being opened from production as normal.

Support's conclusion, recorded on 2026-08-17, was that the failure came from the managed package and not from the Salesforce side. What exactly failed inside the package is not recorded in the ticket.

## 6. Solution

No change. Support closed the ticket on 2026-08-17 as a managed-package issue and asked the reporter to say if it was still happening. Whether the original payment was processed, and how, is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. The acceptance criteria (page loads for the reporter and one other intake user, payment processed) are not marked as checked.

## 8. Step by step

1. **Read the page address before assuming a sandbox.** A host of the form `<org>--<namespace>` for a Visualforce page is a managed package's page, not a sandbox. Compare the part after `--` with the package prefixes on the org's objects (`fw1__` here).
2. **Check whether the page is ours.** If it is not in the repo's `pages` folder and its namespace belongs to a managed package, it is the vendor's.
3. **Try it again, and as a second user.** Note whether it fails for everyone or one person, and on every payment or one.
4. **Check the user's access to the package**, such as a package licence, if it fails for one user only.
5. **If it is the package, raise it with the vendor** with the time, the steps, and a screenshot of the error.
6. **Follow up with the reporter** to confirm whether it is still happening.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **The underlying cause was not identified**, so the same error could come back.
- **Fixes inside a managed package are the vendor's.** We cannot change its page or code.

## 11. Reuse for similar issues

**How to recognise one:** a Visualforce page fails to load and its address contains an unfamiliar word after `--`.

- **That word is usually a managed package namespace**, not a sandbox name. Match it against the object prefixes in the org.
- **Pages that aren't in the repo aren't ours.** Check the repo before trying to debug the page.
- **For package issues, collect evidence and go to the vendor**, and keep the reporter updated.
