# Salesforce

Fixes and changes inside Salesforce: Apex, flows, LWC, fields, page layouts, permissions, reports and data.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-36, DDT-39 | [Call Session records: warm-transfer linking and session fixes](./ddt-36-call-session-linking-fixes/README.md) | Live since 2026-06-02 |
| DDT-37, DDT-40 | [No-invoice warning on criminal cases and billing gap report](./ddt-37-no-invoice-warning-billing-gap/README.md) | Done 2026-05-26 |
| DDT-38 | [Client label missing on auto-created matter accounts](./ddt-38-client-label-matter-accounts/README.md) | Done 2026-05-26 |
| DDT-41, DDT-112, DDT-164, DDT-175, DDT-242, DDT-256, DDT-290, DDT-310, DDT-529, DDT-623, DDT-700, DDT-865, DDT-885, DDT-915, DDT-937, DDT-939, DDT-1190, DDT-1279 | [Changing primary attorney and case manager assignments by courthouse](./ddt-41-courthouse-attorney-case-manager-assignments/README.md) | Live since 2026-10-09 (each ticket went live on its own completed date, listed in section 2) |
| DDT-50, DDT-51, DDT-107, DDT-108, DDT-156, DDT-157 | [Merging open pull requests and syncing main with production](./ddt-50-merge-open-prs-sync-main/README.md) | Done 2026-06-11 |
| DDT-52, DDT-90 | [Settlement statement finalize, KPI capture and fee math](./ddt-52-settlement-statement-finalize-fee-math/README.md) | Live since 2026-06-02 |
| DDT-53, DDT-284, DDT-738 | [Stop Collections flag not suppressing invoices and reminders](./ddt-53-stop-collections-still-invoiced/README.md) | Live since 2026-08-17 |
| DDT-54 | [Calendar event hover shows client, hearing type, judge and location](./ddt-54-calendar-hover-hearing-details/README.md) | Live since 2026-06-01 |
| DDT-72 | [Every new lead flagged as a duplicate account](./ddt-72-duplicate-account-flag-new-leads/README.md) | Done 2026-06-01 |
| DDT-73 | [Hiding the old app and setting the main app as default](./ddt-73-hide-old-app-set-default/README.md) | Done 2026-06-01 |
| DDT-74 | [Removing Case Closed from criminal case status paths](./ddt-74-remove-case-closed-paths/README.md) | Done 2026-06-01 |
| DDT-75 | [Single video link field on events](./ddt-75-event-single-video-link/README.md) | Done 2026-06-01 |
| DDT-76 | [Invoice discount amount not linking to the contract](./ddt-76-invoice-discount-amount-contract/README.md) | Done 2026-06-08 |
| DDT-77 | [Hired matter case management: status workflow, soft close and notes](./ddt-77-hired-matter-case-management/README.md) | Done 2026-07-13 |
| DDT-84 | [County column on the Payments report](./ddt-84-county-payments-report/README.md) | Live since 2026-06-02 |
| DDT-85 | [Calendly package upgrade from v1 to v2](./ddt-85-calendly-package-v2-upgrade/README.md) | Done 2026-06-04 |
| DDT-86 | [Expense Manager component and payment tracking fields](./ddt-86-expense-manager-payment-tracking/README.md) | Live since 2026-06-02 |
| DDT-87 | [Lien Manager component and lien-damages link](./ddt-87-lien-manager-component/README.md) | Live since 2026-06-02 |
| DDT-88 | [Repository deploy pull request review](./ddt-88-repository-deploy-pr-review/README.md) | Live since 2026-06-02 |
| DDT-89 | [Treatment call linker and transcript component](./ddt-89-treatment-call-linker/README.md) | Live since 2026-06-02 |
| DDT-91, DDT-179 | [Torq integration connected app and read-only access](./ddt-91-torq-connected-app-access/README.md) | Done 2026-06-17 |
| DDT-92 | [Federal case status path](./ddt-92-federal-case-status-path/README.md) | Done 2026-06-04 |
| DDT-93 | [Separating person fields from matter fields on client accounts](./ddt-93-client-account-field-cleanup/README.md) | Done 2026-06-04 |
| DDT-94 | [Removing the redundant Violation of Sentence path](./ddt-94-remove-violation-sentence-path/README.md) | Live since 2026-06-03 |
| DDT-96, DDT-113, DDT-138, DDT-252, DDT-1068, DDT-1372 | [Lead conversion fails on a restricted picklist value](./ddt-96-lead-convert-restricted-picklist/README.md) | Live since 2026-10-23 (the full fix, DDT-1372, deployed 2026-10-20; earlier tickets were one-value fixes or workarounds) |
| DDT-98 | [Disabling the post-close case survey automation](./ddt-98-disable-post-close-survey/README.md) | Live since 2026-06-03 |
| DDT-101, DDT-229, DDT-237, DDT-264, DDT-279, DDT-578, DDT-654, DDT-774, DDT-1048, DDT-1296 | [User gets a no-access-to-Apex-class error](./ddt-101-no-access-apex-class/README.md) | Live since 2026-10-12 (each fix was made in production on the day its ticket closed; DDT-1296 was the last) |
| DDT-102, DDT-638, DDT-685, DDT-732 | [Adding an arresting agency picklist value](./ddt-102-arresting-agency-picklist-values/README.md) | Live since 2026-08-14 (DDT-102 closed 2026-06-08 with no change; DDT-638 live 2026-08-04; DDT-685 live 2026-08-10) |
| DDT-104, DDT-105, DDT-127, DDT-199, DDT-200, DDT-236, DDT-259, DDT-283, DDT-305, DDT-765 | [S-Docs and S-Sign licence assignment and access errors](./ddt-104-sdocs-ssign-licence-assignment/README.md) | Live since 2026-06-08 (first ticket closed); last ticket closed 2026-08-19 |
| DDT-106 | [Matter gap checklist component](./ddt-106-matter-gap-checklist/README.md) | Live since 2026-06-05 |
| DDT-111 | [Duplicate cases being created automatically](./ddt-111-duplicate-cases-auto-created/README.md) | Live since 2026-06-29 |
| DDT-119, DDT-515 | [Invoice email template layout](./ddt-119-invoice-email-template/README.md) | DDT-119 live since 2026-06-08; DDT-515 done 2026-07-27 (not found in production on 2026-11-05, see section 10) |
| DDT-120, DDT-188 | [Invoice balance and installment out of sync after payment](./ddt-120-invoice-installment-balance-mismatch/README.md) | Done 2026-06-22 (no system change; data explained and corrected) |
| DDT-121, DDT-133, DDT-281, DDT-579 | [Assigning CloudFiles licence and permission set](./ddt-121-cloudfiles-permission-assignment/README.md) | Done (DDT-121 2026-06-09, DDT-133 2026-06-12, DDT-281 2026-07-13, DDT-579 2026-07-30) |
| DDT-124 | [Checking checkbox on Opportunity](./ddt-124-checking-checkbox-opportunity/README.md) | Done 2026-06-09 (the field is no longer in production as of 2026-11-05, see section 10) |
| DDT-140, DDT-141, DDT-167 | [Onboarding stage automation from task completion](./ddt-140-onboarding-stage-automation/README.md) | Live since 2026-06-10 (DDT-140, DDT-141), extended 2026-06-15 (DDT-167) |
| DDT-142 | [Auto-deleting open tasks when a case reaches a closing stage](./ddt-142-delete-tasks-on-case-close/README.md) | Live since 2026-06-11 |
| DDT-144, DDT-145 | [Lead conversion fails with RECORD_IN_USE_BY_WORKFLOW](./ddt-144-lead-convert-record-in-use/README.md) | Done 2026-06-11 (no system change; both Leads converted) |
| DDT-146 | [Treatment Calls Log report for personal injury](./ddt-146-treatment-calls-log-report/README.md) | Live since 2026-06-12 |
| DDT-148 | [Invoice rollup flow failure blocks hired case conversion](./ddt-148-invoice-rollup-flow-fails/README.md) | Done 2026-06-12 (no system change; the conversion had already succeeded on a retry) |
| DDT-149, DDT-154 | [Invoice stuck in Payment Pending after client paid](./ddt-149-payment-stuck-pending/README.md) | Live since 2026-06-19 |
| DDT-152 | [Cancel Hired Case flow fails with an unhandled fault](./ddt-152-cancel-hired-case-flow-fault/README.md) | Live since 2026-06-12 |
| DDT-155 | [SOS cases missing the Blueback approval-stage ribbon](./ddt-155-sos-blueback-ribbon-missing/README.md) | Done 2026-06-16 |
| DDT-158 | [SOS stage path not syncing to Case Information](./ddt-158-sos-stages-path-sync/README.md) | Live since 2026-06-11 |
| DDT-159 | [Evaluator Name field in the SOS Documents section](./ddt-159-sos-evaluator-name-field/README.md) | Live since 2026-06-24 |
| DDT-160 | [Removing Is Mitigation Needed from SOS layouts](./ddt-160-remove-mitigation-field-sos/README.md) | Live since 2026-06-11 |
| DDT-161, DDT-1032 | [Answered calls create Missed Call leads](./ddt-161-answered-calls-missed-call-leads/README.md) | Live since 2026-09-18 |
| DDT-162, DDT-187, DDT-504 | [MassMailer install and user access](./ddt-162-massmailer-install-access/README.md) | Live since 2026-07-24 |
| DDT-163 | [Previous court appearances missing on an account](./ddt-163-missing-previous-court-appearances/README.md) | Done 2026-06-15 |
| DDT-165 | [Case Type reverting after lead conversion](./ddt-165-case-type-reverts-conversion/README.md) | Done 2026-07-22 |
| DDT-185 | [Rule 214 template generating a broken image](./ddt-185-rule-214-broken-image/README.md) | Live since 2026-06-18 |
| DDT-186, DDT-568, DDT-642, DDT-1153, DDT-1173, DDT-1227, DDT-1267 | [Adding, removing or correcting citations on a case](./ddt-186-correct-citations-on-case/README.md) | Live since 2026-10-08 |
| DDT-189 | [Reminder field on the New Task layout](./ddt-189-task-reminder-field-layout/README.md) | Live since 2026-06-22 |
| DDT-196 | [Payment Plan Fee report counting credit memos as payments](./ddt-196-ppf-report-credit-memos/README.md) | Live since 2026-06-22 |
| DDT-197, DDT-858, DDT-1410 | [Granting Payment Center licence and payment access](./ddt-197-payment-center-access/README.md) | Live since 2026-10-26 |
| DDT-198, DDT-246, DDT-247, DDT-269, DDT-299 | [Disposition entry tool builds and adoption tracking](./ddt-198-disposition-entry-tool/README.md) | Live since 2026-07-10 |
| DDT-201, DDT-273, DDT-522, DDT-656, DDT-748, DDT-766, DDT-809, DDT-875, DDT-950, DDT-1062, DDT-1149, DDT-1373 | [Days Since Last Outreach not updating correctly](./ddt-201-days-since-last-outreach/README.md) | Live since 2026-10-23 |
