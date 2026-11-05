# Hiding the old app and setting the main app as default

| | |
|---|---|
| **Ticket** | DDT-73 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Setup: profiles, app settings) |
| **Area** | Which Lightning app users land in when they sign in |
| **Related** | None recorded |
| **Code** | Not applicable: configured in Setup, no PR is linked. |

---

## 1. Summary

Users were still opening the legacy **RLF (OLD)** app by mistake. The fix hid RLF (OLD) and made **RLF (Main)** the default app on every listed user profile, so users land in the current app when they sign in.

## 2. What was reported

- **2026-06-01, DDT-73:** users were landing on or opening the old RLF app by mistake, causing confusion about which system to work in. The number of users affected was not recorded.

## 3. Scope

**In scope:** app visibility and default app assignment on these 14 profiles.

| Profile | | |
|---|---|---|
| Admin | Leadership | System Administrator |
| AffiniPay Profile | Analytics Cloud Integration User | Analytics Cloud Security User |
| API_ONLY | Billing | Lawyers |
| RLF Intern User | RLF Sys Admin | RLF User |
| Sales Insights Integration User | Staff | |

On each: RLF (OLD) hidden, RLF (Main) set as the default.

**Out of scope**

- **Deleting RLF (OLD).** The ticket recommended keeping it hidden rather than deleting it, because deletion is irreversible. It was hidden, not deleted.
- **Permission sets.** The resolution note lists profiles only.

## 4. How it works (background)

Each profile has **App Settings**: a list of apps with a **Visible** checkbox and a **Default** radio button. The default app is the one a user lands in at sign-in. An app that is not visible on the user's profile (and not granted by a permission set) does not appear in the App Launcher.

## 5. Root cause

RLF (OLD) was still visible, and in some cases the default, on user profiles. Not recorded in the ticket which profiles had which setting before the change.

## 6. Solution

On each of the 14 profiles in section 3:

- **RLF (OLD):** Visible unchecked.
- **RLF (Main):** Visible checked and Default selected.

Why hide instead of delete: hiding is reversible. If someone later needs RLF (OLD) for historical lookups, it can be made visible again on one profile or through a permission set.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for a spot-check with 2 to 3 users on different profiles, confirming RLF (OLD) is gone from the App Launcher and RLF (Main) loads at sign-in.

## 8. Step by step

1. Setup, Profiles, open the first profile in section 3.
2. Go to **Assigned Apps** (or App Settings in the enhanced profile UI) and click Edit.
3. Uncheck **Visible** for RLF (OLD).
4. Check **Visible** for RLF (Main) and select it as **Default**.
5. Save, and repeat for each profile in the list.
6. Log in as (or ask) users on two or three different profiles and confirm RLF (Main) opens and RLF (OLD) is not in the App Launcher.
7. Tell staff the old app is gone and the default has changed.

## 9. Rollback

Re-check Visible for RLF (OLD) on any profile that needs it. Safe: nothing was deleted.

## 10. Known limits

- A **permission set** that grants RLF (OLD) would still make it visible to its holders. The resolution only changed profiles.
- A profile created later starts with its own app settings and must be set the same way.

## 11. Reuse for similar issues

**How to recognise one:** users opening the wrong app, or a retired app still showing in the App Launcher.

- Change app visibility and default on **every** profile in use, not just the common ones. Integration and API profiles are in the list too.
- Hide before you delete. Deleting an app cannot be undone.
- Check permission sets as well as profiles when an app still shows up.
