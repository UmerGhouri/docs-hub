# Clean up Manage and Unmanaged Package

This project reduced the number of packages installed in the production org. It removed the ones nobody used and documented the ones the firm depends on. Packages use custom objects, fields, Apex and storage, some are paid, and old ones bring outdated code, connected apps, scheduled jobs and integration users. The work ran as analysis first, then dependency removal, backup, and one uninstall at a time with sign-off.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-1428, DDT-1440 | [Inventory of every installed package: keep or remove](./ddt-1428-prod-package-inventory-keep-remove/README.md) | Done 2026-10-29 |
| DDT-1437 | [Uninstall nine unused packages and remove dead S-Docs buttons](./ddt-1437-uninstall-nine-unused-packages/README.md) | Live since 2026-10-29 |
| DDT-1438 | [Remove Talkdesk, the replaced phone system](./ddt-1438-decouple-and-uninstall-talkdesk/README.md) | Live since 2026-10-30 |
| DDT-1439 | [Retire S-Docs and SSign after the move to Dochly](./ddt-1439-retire-sdocs-and-ssign/README.md) | Live since 2026-10-28 |
| DDT-1477 | [Uninstall the Graphics Pack image library](./ddt-1477-uninstall-graphics-pack/README.md) | Live since 2026-10-30 |
