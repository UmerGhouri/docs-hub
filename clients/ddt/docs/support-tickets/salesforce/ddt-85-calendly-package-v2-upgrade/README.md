# Calendly package upgrade from v1 to v2

| | |
|---|---|
| **Ticket** | DDT-85 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-04 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Calendly managed package) |
| **Area** | Calendly to Salesforce scheduling integration |
| **Related** | None recorded |
| **Code** | Not applicable: no change was needed. |

---

## 1. Summary

Calendly showed an in-app notice saying its v1 Salesforce package would be retired on 2026-08-31 and asking for an upgrade to v2. On checking, every DDT org was **already on Calendly v2.6**, so no upgrade or rebuild was needed.

## 2. What was reported

- **2026-06-01, DDT-85:** an in-app Calendly notice, "Action required: Update to Salesforce package Version 2". The ticket planned an audit of Calendly-dependent automation, a sandbox install, flow rebuilds and a production upgrade before the retirement date.

## 3. Scope

**In scope:** checking the installed Calendly package version in production and in each sandbox (three sandboxes).

**Out of scope:** the planned audit, rebuild and upgrade. Not needed once the version check showed v2 was already installed.

## 4. How it works (background)

Calendly connects to Salesforce through a managed package installed in each org. The package version is listed in Setup, Installed Packages.

## 5. Root cause

Not a defect. The notice asked for an upgrade the orgs had already received. Why Calendly showed the notice anyway was not recorded in the ticket.

## 6. Solution

No change. Every DDT org (production and the three sandboxes checked) was confirmed on Calendly **v2.6**.

Why check first: the planned work (audit, sandbox install, flow rebuild, production upgrade) was large. A version check takes a minute and showed none of it was needed.

## 7. Testing

The installed package version was checked in each org. No further testing was needed.

## 8. Step by step

1. In each org, open Setup, Installed Packages.
2. Find the Calendly package and read its **Version Number**.
3. If it is 2.x, no upgrade is needed. If it is 1.x, plan the upgrade in a sandbox first.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Confirmed on 2026-06-04. A sandbox refreshed or created later gets whatever version production has at that time.

## 11. Reuse for similar issues

**How to recognise one:** a vendor notice asking for a package upgrade or migration.

- Check the installed version in every org before planning any work. Vendor notices are often shown to everyone, including orgs already upgraded.
- Check sandboxes as well as production.
