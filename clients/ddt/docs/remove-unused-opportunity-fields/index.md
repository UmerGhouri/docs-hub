# Remove Unused Opportunity Fields

The Opportunity object was at about 490 of its 500 custom fields, so the next feature needing a field could not have one. This project measured every field's use across all Opportunities, deleted the fields holding no data or belonging to dead processes, and set a template for how Opportunity is used from here.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-1063 | [Delete the Opportunity fields that hold no data](./ddt-1063-delete-empty-opportunity-fields/README.md) | Live since 2026-09-28 |
| DDT-1065, DDT-1415 | [Full review of the Opportunity fields, and deleting the 35 it approved](./ddt-1065-opportunity-field-review-and-deletions/README.md) | Live since 2026-10-29 |
| DDT-1066 | [Measure 28 suspect Opportunity fields and delete the dead ones](./ddt-1066-retire-named-dead-opportunity-fields/README.md) | Live since 2026-10-01 |
| DDT-1067 | [Review requests were recorded twice; keep only the Review Request object](./ddt-1067-single-record-for-review-requests/README.md) | Live since 2026-09-29 |
| DDT-1069 | [Retire the old DUI intake questionnaire fields on Opportunity](./ddt-1069-retire-old-dui-questionnaire-fields/README.md) | Live since 2026-09-28 |
| DDT-1152 | [Stop the intake flow writing an unused checkbox, then delete it](./ddt-1152-remove-unused-intake-checkbox/README.md) | Live since 2026-09-30 |
