# Demands tab unavailable on companion matters

| | |
|---|---|
| **Ticket** | DDT-494 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, validation rule) |
| **Area** | PI settlements: logging demands and offers per client on companion matters |
| **Related** | None recorded |
| **Code** | Commit `26da4565c` on `main` (2026-07-24, no PR number) and PR #353 in the `ddt-salesforce` repo |

---

## 1. Summary

Companion matters are several clients from the same crash who share one At-Fault insurance policy. On a companion matter the Demands tab showed "No coverage available" and offered no way to create a demand. A first fix in July made the shared coverage visible and allowed the demand. In September it turned out that a demand created on a passenger's matter was still being filed on the driver's matter. The second fix makes the Demands component always send the matter it was opened from.

## 2. What was reported

- **2026-07-23.** On a primary matter the Demands tab worked. On its companion matters, which share the same At-Fault policy, the tab showed "No coverage available" and had no Create New Demand option. Each client gets their own settlement offer, so each matter needs its own demand tracker.
- **2026-09-10.** After the July fix, creating a bodily injury (BI) demand on a passenger's matter saved it to the driver's matter instead. That mixed the clients' settlement numbers, distorted team KPIs, and left the passenger's matter with no negotiation history.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DemandController.getAvailableCoverages` | Apex | Also returns coverage shared through the matter's Incident (July) |
| `DemandController.saveDemand` | Apex | Accepts a demand on a companion matter when the coverage shares its Incident (July) |
| `Coverage_Matter_Match` | Validation rule on `Demand__c` | Relaxed to allow same-Incident companion coverage (July) |
| `DemandControllerTest` | Apex test | Two companion tests (July) |
| `demandTracker` | LWC | New demand always carries `Matter__c` = the matter it was opened from (PR #353) |
| `demandTracker.test.js` | Jest test | Two companion tests, and corrected mocks (PR #353) |
| Demands wrongly filed on a primary matter | Data | Cleared (September) |

**Out of scope**

- **Who owns the shared Coverage record.** It stays on the primary matter.

## 4. How it works (background)

```
One crash = one PI_Incident__c
  -> several Matter__c (driver = primary, passengers = companions), each with Incident__c set
  -> one At-Fault Coverage__c, owned by the primary matter (Coverage__c.Matter__c)
     and shared through Coverage__c.Incident__c

Demands tab (demandTracker LWC)
  -> getAvailableCoverages(matterId)   lists coverage for the picker
  -> saveDemand(record)                 saves Demand__c
       if record.Matter__c is blank -> uses Coverage__c.Matter__c
```

The separate coverage panel already understood coverage shared through the Incident. The demand path did not.

## 5. Root cause

### July: no coverage and no Create button

`getAvailableCoverages` only returned coverage whose `Matter__c` was the current matter. A companion matter owns no coverage of its own, so the list was empty. `saveDemand` and the `Coverage_Matter_Match` validation rule also required the demand and coverage to be on the same matter, so a companion demand would have been rejected anyway.

### September: demand filed on the wrong matter

The July fix only covered the read side. The new-demand payload sent from `demandTracker` left `Matter__c` blank, so `saveDemand` filled it from `Coverage__c.Matter__c`:

```apex
if (record.Matter__c == null) {
    record.Matter__c = cov.Matter__c;   // the primary matter, for shared coverage
}
```

For shared coverage that is always the primary matter, so every companion demand landed on the driver's file.

**The first fix looked complete.** The button appeared and saving worked, so it was verified as fixed in July. Nobody checked which matter the saved demand belonged to.

## 6. Solution

### 6.1 Read side (commit `26da4565c`, 2026-07-24)

- `getAvailableCoverages` now returns coverage where `Matter__c = :matterId` **or** `Incident__c = :incidentId` (the matter's Incident), mirroring the coverage panel.
- `saveDemand` allows a demand whose matter differs from the coverage's matter when both share the same Incident. Otherwise it still throws "Coverage and Demand must belong to the same Matter or companion group."
- `Coverage_Matter_Match` only fires when the matters differ **and** the Incidents do not match:

```
AND(
  NOT(ISBLANK(Coverage__c)),
  NOT(ISBLANK(Matter__c)),
  Coverage__r.Matter__c <> Matter__c,
  OR(
    ISBLANK(Coverage__r.Incident__c),
    ISBLANK(Matter__r.Incident__c),
    Coverage__r.Incident__c <> Matter__r.Incident__c
  )
)
```

### 6.2 Write side (PR #353, 2026-09-11)

```js
// Always stamp the Matter we were opened from. A companion matter's
// At-Fault Coverage is owned by the primary matter, so leaving this
// blank makes Apex fall back to Coverage__c.Matter__c and the demand
// lands on the primary's file instead of this client's. (DDT-494)
this.newDemand = {
    Matter__c: matterId,
    ...
};
```

No Apex change was needed: after 6.1, Apex already accepts a companion demand when the coverage shares the Incident.

### 6.3 Data

The demands that had been wrongly filed on the primary matter were cleared.

## 7. Testing

| Test | What it proves |
|---|---|
| `DemandControllerTest.getAvailableCoverages_companionMatter_seesIncidentSharedCoverage` | A companion matter sees the shared coverage |
| `DemandControllerTest.saveDemand_companionMatter_sharedCoverage_succeeds` | A demand can be saved on a companion matter with shared coverage |
| `demandTracker.test.js`: "files the demand on the companion matter, not the matter that owns the coverage" | The LWC sends the companion's matter |
| `demandTracker.test.js`: "asks for coverages using the companion matter id" | The coverage list is requested for the right matter |

PR #353 also fixed the Jest mocks: `getAvailableCoverages` is called imperatively, not wired, and `jest.mock` only hoists above the imports when the module path is a literal.

**Live check:** in July, `getAvailableCoverages` returned the shared coverage on both reported companion matters (empty before). In September, a demand created on a passenger's matter filed on that matter.

**Not covered:** the July tests prove a companion demand *can* be saved, not that the component sends the right matter. That gap is why the September bug reached users.

## 8. Step by step

1. **Confirm the matters are companions.** They share `Incident__c`, and the At-Fault Coverage is owned by one of them.
2. **Read side:** change `getAvailableCoverages` and `saveDemand`, and relax `Coverage_Matter_Match`, as in 6.1. Add the two Apex tests.
3. **Write side:** in `demandTracker.js`, set `Matter__c` to the current matter on every new demand, as in 6.2. Add the two Jest tests.
4. **Validate against production, get approval, deploy.**
5. **Users must hard-refresh** (Ctrl+Shift+R) or close and reopen their Salesforce tabs. The old component stays cached and will keep misfiling until they do.
6. **Find and clear demands** that were filed on the primary matter for a companion client.
7. **Create a demand on a companion matter** and check its `Matter__c`.

## 9. Rollback

- **LWC:** redeploy the previous `demandTracker`. Companion demands would land on the primary matter again.
- **Apex and validation rule:** redeploy the previous versions. Companion matters would show "No coverage available" again.

## 10. Known limits

- **Companion grouping depends on `Incident__c`.** A companion matter with no Incident, or a different one, will not see the shared coverage.
- **Cached components.** Any change to `demandTracker` needs users to refresh before it takes effect.

## 11. Reuse for similar issues

**How to recognise one:** a record created from one page ends up attached to a different parent.

- **Look for a server-side fallback** that fills a blank parent from a related record. It hides a missing value in the client.
- **Always send the parent you were opened from** rather than relying on a fallback.
- **When verifying a fix, check where the saved record landed**, not only that the save succeeded.
- **Test both sides**: Apex that it accepts the record, and the component that it sends the right values.
- **Shared records across a group** (one coverage, several matters): match on the group key (`Incident__c`), not on the owner.
