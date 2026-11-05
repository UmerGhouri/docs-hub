# Data & File Storage

Production was over its file storage limit (108%) and close to its data storage limit (93%). Over the limit, Salesforce can block new uploads and record inserts. This project measured what was using the space, decided what to move to Google Drive, archive or delete, ran the clean-up with backups, and added guards so usage stays under the limits.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-1385 | [File storage over the limit: analysis and move to Google Drive](./ddt-1385-file-storage-over-limit/README.md) | Live since 2026-11-05 |
| DDT-1386 | [Data storage almost full: analysis and retention rules](./ddt-1386-data-storage-retention-rules/README.md) | Live since 2026-11-05 |
| DDT-1407 | [Review the three daily reporting snapshots](./ddt-1407-daily-reporting-snapshot-retention/README.md) | Live since 2026-11-04 |
| DDT-1473, DDT-1480, DDT-1481 | [Move shared-inbox email attachments to Google Drive](./ddt-1473-inbox-attachments-to-google-drive/README.md) | Live since 2026-10-30 |
| DDT-1483 | [Move no-Case email attachments and Court Case videos to Google Drive](./ddt-1483-no-case-email-files-and-videos/README.md) | Live since 2026-10-30 |
| DDT-1489 | [Stop MassMailer saving an email and a task per newsletter recipient](./ddt-1489-massmailer-activity-logging-off/README.md) | Live since 2026-11-02 |
| DDT-1490 | [Log cleanup: log failures only and purge old log rows](./ddt-1490-log-cleanup-failures-only/README.md) | Live since 2026-11-02 |
| DDT-1491 | [Remove leftover JustCall records](./ddt-1491-remove-justcall-data/README.md) | Done 2026-11-02 |
| DDT-1492 | [Drive Activity Log: 90-day retention and daily purge](./ddt-1492-drive-activity-log-retention/README.md) | Live since 2026-11-04 |
