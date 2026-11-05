# Lien Manager component and lien-damages link

| | |
|---|---|
| **Ticket** | DDT-87 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom object, permission set) |
| **Area** | PI liens and medical damages on the Matter page |
| **Related** | DDT-86 (Expense Manager, the same panel pattern) |
| **Code** | PR #22 in the `ddt-salesforce` repo, with the deploy doc `docs/deploys/lien-manager-lwc.md` |

---

## 1. Summary

The ticket was a review request for PR #22: a new **Lien Manager** panel on the Matter page, and a new junction object, `Lien_Damages__c`, so one lien can cover several Damages records and one Damages record can carry several liens. A one-time script then created a junction row for every existing lien. It shipped to production on 2026-06-02, and the backfill created **122** junction rows.

## 2. What was reported

- **2026-06-02, DDT-87:** review PR #22 before promotion. A lien could only point at one Damages record through a single lookup, which does not match how liens work in practice:
  - **Several Damages to one lien:** one provider, several visits, one consolidated lien.
  - **One Damages to several liens:** an ER visit billed separately by the facility, radiology and individual physicians.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Lien_Damages__c` | Custom object | New junction. Master-Detail to `Lien__c` (primary) and to `Damages__c` |
| `lienManager` | LWC | New panel on `Matter_Record_Page`, above the existing liens related list |
| `LienManagerController` | Apex class | New: `getLiensForMatter`, `getLienDamagesDefaults`, `addDamagesLink`, `removeDamagesLink` |
| `LienManagerControllerTest` | Apex test | New, 11 tests |
| `DamagesTableController` | Apex class | `createLienForDamage` also inserts the junction row |
| `Original_Lien_Amount__c`, `Final_Negotiated_Amount__c`, `Effective_Deduction__c`, `Proposed_Reduction__c` | Fields on `Lien__c` | Help text only |
| `Matter_Access` | Permission set | Class access, and access to `Lien_Damages__c` |
| Backfill | Anonymous Apex | One-time: a junction row for every existing lien |

**Out of scope** (from the deploy doc)

- **Removing the single `Lien__c.Damages__c` lookup.** It stays as the "primary" Damages link, because the settlement statement and the Damages table still read it.
- **An allocation amount per junction row** (how much of a lien belongs to each visit).
- **Removing the standard liens related list**, and mounting the panel on the Damages page.

## 4. How it works (background)

```
Before:  Lien__c.Damages__c  (one lookup)  -> one Damages record per lien

After:   Lien__c.Damages__c  (kept, primary link)
         Lien_Damages__c     (junction)      -> any number of Damages per lien,
                                                any number of liens per Damages
```

Matter page: `lienManager` lists the Matter's liens with totals (Original, Negotiated, Savings). The Edit form has a **Linked Damages** section to add or remove junction rows. A `+N more` badge shows when a lien has several links.

## 5. Root cause

Not a defect: a data model change. A single lookup can only express one-to-one, and real liens are many-to-many with Damages.

## 6. Solution

### 6.1 Junction object

`Lien_Damages__c` has two Master-Detail fields, so its sharing follows its parents and its rows are deleted with them. The existing lookup was kept rather than replaced.

Why keep the lookup: the settlement statement and `createLienForDamage` read it. Replacing it would have meant changing those in the same PR. The junction adds links alongside it, and the lookup can be retired once the settlement statement reads the junction.

### 6.2 Dual write

When a lien is created from the Damages table, `createLienForDamage` inserts the lien, then the junction row, in the same transaction. If the junction insert fails, the lien insert rolls back with it.

### 6.3 Duplicate links

`addDamagesLink` checks for an existing link before inserting and returns a friendly error for a duplicate.

### 6.4 Backfill of existing liens

From the deploy doc. Run once, after deploy:

```apex
Set<Id> liensWithJunction = new Set<Id>();
for (Lien_Damages__c ld : [SELECT Lien__c FROM Lien_Damages__c LIMIT 50000]) {
    liensWithJunction.add(ld.Lien__c);
}
List<Lien_Damages__c> toInsert = new List<Lien_Damages__c>();
for (Lien__c l : [SELECT Id, Damages__c FROM Lien__c WHERE Damages__c != null LIMIT 50000]) {
    if (liensWithJunction.contains(l.Id)) continue;
    toInsert.add(new Lien_Damages__c(Lien__c = l.Id, Damages__c = l.Damages__c));
}
insert toInsert;
```

Why it is written this way: it skips any lien that already has a junction row, so it is safe to run again.

## 7. Testing

From the PR and its deploy doc, run in the sandbox:

| Test | Result |
|---|---|
| `LienManagerControllerTest` | 11 of 11 |
| `DamagesTableControllerTest` (regression on `createLienForDamage`) | 54 of 54 |
| `SettlementStatementControllerTest` | 20 of 20 |
| `ExpenseManagerControllerTest` | 8 of 8 |
| Jest, `lienManager` (empty state, rows and totals, load error) | 3 of 3 |

Manual checks in the sandbox: liens render with totals; the Edit form shows the backfilled link; adding a second link works and a duplicate is refused; removing a link works; the Matter's total liens roll-up is unchanged; help text shows on the four money fields.

**Production:** the backfill created **122** junction rows (from the resolution note).

**Not covered:** the settlement statement still reads only the primary lookup, so additional junction links do not appear on it.

## 8. Step by step

1. Before deploying, count liens that have a Damages link: `SELECT COUNT() FROM Lien__c WHERE Damages__c != null`.
2. Deploy the object, Apex, LWC, flexipage, help text and permission set together, running the specified tests:

   ```
   sf project deploy start -o <prod-alias> \
     --test-level RunSpecifiedTests \
     --tests LienManagerControllerTest \
     --tests DamagesTableControllerTest
   ```

3. Run the backfill in 6.4 as anonymous Apex.
4. Check `SELECT COUNT() FROM Lien_Damages__c` matches the count from step 1.
5. Confirm users hold `Matter_Access`, then open a Matter with liens and check the panel.

## 9. Rollback

From the deploy doc:

1. **Junction insert failures:** revert the junction lines in `createLienForDamage`. Existing liens are unaffected.
2. **Remove the panel:** take `lienManager` off the Matter page. The standard related list still shows.
3. **Remove the object:** a destructive deploy of `Lien_Damages__c`. Its rows are Master-Detail children and go with it, with no orphans.
4. **Permission set:** remove the new class and object access.

## 10. Known limits

- **Two ways to link a lien to Damages.** The primary lookup and the junction both exist. Anything new should read the junction.
- **No allocation per link.** The junction records that a lien covers a Damages record, not how much of it.

## 11. Reuse for similar issues

**How to recognise one:** staff work around a single lookup by creating duplicate records, or cannot record a real relationship because the lookup only allows one parent.

- Add a junction object with two Master-Detail fields, and keep the old lookup until every reader has moved.
- Write both in the same transaction so they cannot drift apart on create.
- Backfill with a script that skips rows already done, and count before and after.
