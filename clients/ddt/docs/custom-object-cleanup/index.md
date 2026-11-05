# Custom Object Cleanup

Production was at its custom-object limit, which blocked new builds. This project found the firm's own custom objects and custom settings that are no longer used, removed what depended on them, backed up their data and deleted them to free slots. Managed-package objects were out of scope.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-1390 | [Find the custom objects that are safe to delete](./ddt-1390-find-unused-custom-objects/README.md) | Done 2026-10-28 |
| DDT-1391, DDT-1392, DDT-1393, DDT-1472 | [Back up, unwire and delete 39 unused custom objects](./ddt-1391-delete-unused-custom-objects/README.md) | Live since 2026-10-30 |
