# Salesforce

Fixes and changes inside Salesforce: Apex, flows, LWC, fields, page layouts, permissions, reports and data.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-36, DDT-39 | [Call Session records: warm-transfer linking and session fixes](./ddt-36-call-session-linking-fixes/README.md) | Live since 2026-06-02 |
| DDT-37, DDT-40 | [No-invoice warning on criminal cases and billing gap report](./ddt-37-no-invoice-warning-billing-gap/README.md) | Done 2026-05-26 |
| DDT-38 | [Client label missing on auto-created matter accounts](./ddt-38-client-label-matter-accounts/README.md) | Done 2026-05-26 |
| DDT-41, DDT-112, DDT-164, DDT-175, DDT-242, DDT-256, DDT-290, DDT-310, DDT-529, DDT-623, DDT-700, DDT-865, DDT-885, DDT-915, DDT-937, DDT-939, DDT-1190, DDT-1279 | [Changing primary attorney and case manager assignments by courthouse](./ddt-41-courthouse-attorney-case-manager-assignments/README.md) | Live since 2026-10-09 (each ticket went live on its own completed date, listed in section 2) |
| DDT-42, DDT-49 | [AI Summary component: missing Generate button and summary errors](./ddt-42-ai-summary-generate-errors/README.md) | Live since 2026-05-26 |
| DDT-43, DDT-44, DDT-45, DDT-46 | [Shared Inbox New button regression and deploy guards](./ddt-43-shared-inbox-new-button-regression/README.md) | Done 2026-05-26 |
| DDT-47 | [Generating AI summaries for hired, open cases](./ddt-47-ai-summaries-open-cases/README.md) | Live since 2026-05-26 |
| DDT-48 | [Case Plan Builder permission sets by practice area](./ddt-48-plan-builder-permission-sets/README.md) | Live since 2026-05-26 |
| DDT-50, DDT-51, DDT-107, DDT-108, DDT-156, DDT-157 | [Merging open pull requests and syncing main with production](./ddt-50-merge-open-prs-sync-main/README.md) | Done 2026-06-11 |
| DDT-52, DDT-90 | [Settlement statement finalize, KPI capture and fee math](./ddt-52-settlement-statement-finalize-fee-math/README.md) | Live since 2026-06-02 |
| DDT-53, DDT-284, DDT-738 | [Stop Collections flag not suppressing invoices and reminders](./ddt-53-stop-collections-still-invoiced/README.md) | Live since 2026-08-17 |
| DDT-54 | [Calendar event hover shows client, hearing type, judge and location](./ddt-54-calendar-hover-hearing-details/README.md) | Live since 2026-06-01 |
| DDT-55 | [Full calendar access for managing attorneys](./ddt-55-managing-attorney-calendar-access/README.md) | Live since 2026-05-26 |
| DDT-56 | [Cook County rescission order document not generating](./ddt-56-rescission-order-not-generating/README.md) | Live since 2026-05-26 |
| DDT-57, DDT-166 | [Cook County document templates show the old office address](./ddt-57-cook-templates-old-address/README.md) | Live since 2026-06-15 |
| DDT-58 | [Send Zoom Info flow fails with an unhandled fault](./ddt-58-send-zoom-info-flow-fault/README.md) | Live since 2026-05-26 |
| DDT-59 | [Cannot create an appointment with one specific user](./ddt-59-appointment-one-user-fails/README.md) | Live since 2026-05-26 |
| DDT-60, DDT-61 | [Adding a field to a layout when the user lacks access](./ddt-60-layout-field-user-lacks-access/README.md) | Live since 2026-05-26 |
| DDT-62 | [Case Plan Builder prerequisite list shows duplicate task names](./ddt-62-plan-builder-duplicate-prerequisites/README.md) | Live since 2026-05-26 |
| DDT-67, DDT-143, DDT-176 | [Renaming a matter stage orphans records back to stage 1](./ddt-67-stage-rename-orphans-records/README.md) | Live since 2026-06-11 |
| DDT-68 | [Letting intake merge duplicate person clients](./ddt-68-duplicate-merge-access-intake/README.md) | Live since 2026-06-01 |
| DDT-69 | [Matter stage path order wrong: Treatment before Case Opening](./ddt-69-matter-stage-path-order/README.md) | Live since 2026-06-02 |
| DDT-70 | [Installment amounts stored as fractional pennies trigger false past-due emails](./ddt-70-installment-fractional-pennies/README.md) | Live since 2026-06-01 |
| DDT-71, DDT-190, DDT-191, DDT-192, DDT-193, DDT-194 | [Send for Signature through DocuSign: send, sign, convert, void](./ddt-71-docusign-send-for-signature/README.md) | Live since 2026-06-17. Retired 2026-10-19 (DDT-1071), replaced by Dochly (DDT-1070) |
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
| DDT-114, DDT-926 | [Payment reminders show amounts not yet due](./ddt-114-reminders-amounts-not-due/README.md) | Live since 2026-09-08 (DDT-114 live since 2026-06-08) |
| DDT-115, DDT-117, DDT-204, DDT-266, DDT-267 | [Creating a new Salesforce user by cloning a profile](./ddt-115-new-user-clone-profile/README.md) | Done 2026-07-07 |
| DDT-116, DDT-151, DDT-997, DDT-1163 | [Refunding a payment fails with Too many SOQL queries](./ddt-116-refund-too-many-soql/README.md) | Done 2026-10-16 |
| DDT-118 | [Enabling Blueback for Aggravated Speeding cases](./ddt-118-blueback-aggravated-speeding/README.md) | Live since 2026-06-08 |
| DDT-119, DDT-515 | [Invoice email template layout](./ddt-119-invoice-email-template/README.md) | DDT-119 live since 2026-06-08; DDT-515 done 2026-07-27 (not found in production on 2026-11-05, see section 10) |
| DDT-120, DDT-188 | [Invoice balance and installment out of sync after payment](./ddt-120-invoice-installment-balance-mismatch/README.md) | Done 2026-06-22 (no system change; data explained and corrected) |
| DDT-121, DDT-133, DDT-281, DDT-579 | [Assigning CloudFiles licence and permission set](./ddt-121-cloudfiles-permission-assignment/README.md) | Done (DDT-121 2026-06-09, DDT-133 2026-06-12, DDT-281 2026-07-13, DDT-579 2026-07-30) |
| DDT-124 | [Checking checkbox on Opportunity](./ddt-124-checking-checkbox-opportunity/README.md) | Done 2026-06-09 (the field is no longer in production as of 2026-11-05, see section 10) |
| DDT-128 | [Shared inbox send fails on LOB heap limit](./ddt-128-inbox-send-heap-limit/README.md) | Done 2026-07-13 |
| DDT-134 | [Lead Source not carried to Court Case on conversion](./ddt-134-lead-source-not-carried/README.md) | Live since 2026-06-11 |
| DDT-135 | [Cannot log in after email and name update](./ddt-135-login-fails-username-change/README.md) | Done 2026-06-12 |
| DDT-136 | [Recreating a Marketing Cloud audience as an Account report](./ddt-136-marketing-audience-account-report/README.md) | Done 2026-06-16 |
| DDT-137 | [Opening the converted record after lead conversion](./ddt-137-open-record-after-conversion/README.md) | Live since 2026-06-11 |
| DDT-139, DDT-514 | [Email to Legal POD flow: welcome email, tasks and sender](./ddt-139-legal-pod-email-flow/README.md) | Live since 2026-07-27 |
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
| DDT-168 | [Migrating Opportunity Products to Matter Cost records](./ddt-168-migrate-products-matter-costs/README.md) | Done 2026-06-15 |
| DDT-169 | [Invoice line consolidation loses costs that share a product](./ddt-169-invoice-line-consolidation-loss/README.md) | Done 2026-06-15 |
| DDT-170 | [Verifying the firm email domain for user email sending](./ddt-170-verify-email-domain-sending/README.md) | Done 2026-06-16 |
| DDT-172, DDT-1231 | [Convert to Hired Case flow fails with an unhandled fault](./ddt-172-convert-hired-case-flow-fault/README.md) | Live since 2026-10-08 |
| DDT-173 | [Cook County Appearance (Long) template error](./ddt-173-cook-appearance-long-error/README.md) | Live since 2026-06-16 |
| DDT-174, DDT-224 | [Attorney bonus button and per-attorney bonus reports](./ddt-174-attorney-bonus-button-reports/README.md) | Live since 2026-06-24 |
| DDT-177 | [SOS team dashboard](./ddt-177-sos-team-dashboard/README.md) | Live since 2026-06-19 |
| DDT-178, DDT-1127 | [Calendly scheduling: a user cannot be selected](./ddt-178-calendly-user-not-selectable/README.md) | Live since 2026-09-28 |
| DDT-185 | [Rule 214 template generating a broken image](./ddt-185-rule-214-broken-image/README.md) | Live since 2026-06-18 |
| DDT-186, DDT-568, DDT-642, DDT-1153, DDT-1173, DDT-1227, DDT-1267 | [Adding, removing or correcting citations on a case](./ddt-186-correct-citations-on-case/README.md) | Live since 2026-10-08 |
| DDT-189 | [Reminder field on the New Task layout](./ddt-189-task-reminder-field-layout/README.md) | Live since 2026-06-22 |
| DDT-196 | [Payment Plan Fee report counting credit memos as payments](./ddt-196-ppf-report-credit-memos/README.md) | Live since 2026-06-22 |
| DDT-197, DDT-858, DDT-1410 | [Granting Payment Center licence and payment access](./ddt-197-payment-center-access/README.md) | Live since 2026-10-26 |
| DDT-198, DDT-246, DDT-247, DDT-269, DDT-299 | [Disposition entry tool builds and adoption tracking](./ddt-198-disposition-entry-tool/README.md) | Live since 2026-07-10 |
| DDT-201, DDT-273, DDT-522, DDT-656, DDT-748, DDT-766, DDT-809, DDT-875, DDT-950, DDT-1062, DDT-1149, DDT-1373 | [Days Since Last Outreach not updating correctly](./ddt-201-days-since-last-outreach/README.md) | Live since 2026-10-23 |
| DDT-202 | [No Pending Case option when closing a potential case](./ddt-202-no-pending-case-close-reason/README.md) | Live since 2026-06-19 |
| DDT-203, DDT-776, DDT-1012 | [Payments not linked to the opportunity or court case](./ddt-203-payment-not-linked-case/README.md) | Done 2026-09-15 |
| DDT-207 | [Auto-BillPay did not trigger for a scheduled installment](./ddt-207-autobillpay-not-triggered/README.md) | Done 2026-06-29 |
| DDT-208, DDT-209 | [Two-tier close reason on Matter and migration](./ddt-208-matter-two-tier-close-reason/README.md) | Done 2026-06-22 |
| DDT-210 | [Pre-hire fields on hired matters and urgent flags on assignment email](./ddt-210-pre-hire-fields-hired-matter/README.md) | Done 2026-06-22 |
| DDT-211 | [Court Appearance module migration and cleanup](./ddt-211-court-appearance-module-migration/README.md) | Done 2026-06-22 |
| DDT-212 | [Acquired and merger cases record type and migration](./ddt-212-acquired-merger-record-type/README.md) | Done 2026-06-22 |
| DDT-213 | [Duplicate Schedule Follow-Up Court Date to-dos](./ddt-213-duplicate-follow-up-todos/README.md) | Done 2026-06-22 |
| DDT-214 | [Matter restructure: stage fields, validation rules and path components](./ddt-214-matter-restructure-stage-fields/README.md) | Done 2026-07-06 |
| DDT-215, DDT-1044 | [Income Today report classification and case status](./ddt-215-income-today-report/README.md) | Live since 2026-06-26 (DDT-215) and 2026-09-18 (DDT-1044) |
| DDT-216, DDT-245, DDT-270, DDT-301, DDT-318 | [Daily Docket court calendar app builds](./ddt-216-daily-docket-calendar-app/README.md) | Live since 2026-07-10 (DDT-216, DDT-245, DDT-270, DDT-301). DDT-318 Done 2026-09-21, built in a sandbox |
| DDT-217, DDT-772 | [Welcome text after hire](./ddt-217-welcome-text-after-hire/README.md) | Live since 2026-06-24 (DDT-217) and 2026-09-01 (DDT-772) |
| DDT-218 | [Quilla to Salesforce sync not working](./ddt-218-quilla-sync-not-working/README.md) | Done 2026-06-24 |
| DDT-222 | [CloudFiles Google Drive connection fails with Unauthorized 401](./ddt-222-cloudfiles-drive-connection-401/README.md) | Done 2026-06-24 |
| DDT-223, DDT-520 | [Create Court Case flow fails with an unhandled fault](./ddt-223-create-court-case-flow-fault/README.md) | DDT-223: Done 2026-06-24. DDT-520: Live since 2026-08-10 |
| DDT-225 | [CloudFiles flow moving native files to Google Drive](./ddt-225-cloudfiles-move-files-drive/README.md) | Live since 2026-06-24 (the flow was later retired under DDT-867) |
| DDT-226, DDT-276, DDT-1030, DDT-1369 | [Adding a new sales rep to intake reports and dashboards](./ddt-226-add-rep-sales-reports/README.md) | DDT-226: Live since 2026-06-24. DDT-276: Done 2026-07-08. DDT-1030: Live since 2026-09-17. DDT-1369: Done 2026-10-20 |
| DDT-275, DDT-473, DDT-506, DDT-567, DDT-767, DDT-1328, DDT-1412 | [Merging duplicate leads, accounts and contacts](./ddt-275-merge-duplicate-leads-accounts/README.md) | Done 2026-10-26 (last of the seven; each was a one-off data fix in production) |
| DDT-280 | [Liens report type and Financials tab access](./ddt-280-liens-report-type-access/README.md) | Live since 2026-07-27 |
| DDT-285 | [Shared inbox Reply All missing recipients and wrong signature address](./ddt-285-inbox-reply-all-recipients/README.md) | Live since 2026-07-27 |
| DDT-286, DDT-469 | [Missing disposition history on an opportunity](./ddt-286-missing-disposition-history/README.md) | Done 2026-07-27 (DDT-286 done 2026-07-10, DDT-469 done 2026-07-27) |
| DDT-287, DDT-314, DDT-1015, DDT-1109 | [Disposition email blank or missing disposition data](./ddt-287-disposition-email-blank/README.md) | Live since 2026-09-24 (first fix live 2026-07-10) |
| DDT-288 | [Confirm NCD button not sending the disposition letter](./ddt-288-confirm-ncd-letter-not-sent/README.md) | Live since 2026-07-27 |
| DDT-289, DDT-375, DDT-1161 | [Calendly bookings not creating Salesforce leads](./ddt-289-calendly-bookings-no-lead/README.md) | Live since 2026-10-01 |
| DDT-291 | [Adding Will County courthouses and arresting agencies](./ddt-291-will-county-courthouses-agencies/README.md) | Live since 2026-07-09 |
| DDT-293, DDT-297, DDT-673, DDT-882, DDT-938 | [Adding or removing a user from a shared inbox](./ddt-293-shared-inbox-membership/README.md) | Live since 2026-09-07 |
| DDT-294 | [Intake team access to PI case files](./ddt-294-intake-access-pi-files/README.md) | Live since 2026-07-10 |
| DDT-295, DDT-632 | [Shared inbox tag notifications not received](./ddt-295-inbox-tag-notifications/README.md) | Live since 2026-08-11 |
| DDT-296, DDT-327, DDT-995, DDT-1305, DDT-1500 | [Legal Growth Engine (Martindale-Avvo, Nolo) lead integration](./ddt-296-legal-growth-engine-leads/README.md) | Live since 2026-11-05 |
| DDT-300 | [Read-only Court Appearances panel on Opportunity](./ddt-300-court-appearances-panel-opportunity/README.md) | Live since 2026-10-01 (merged to `main` 2026-07-10) |
| DDT-302, DDT-303 | [Coverage panel Not Available status guard](./ddt-302-coverage-not-available-guard/README.md) | Live since 2026-07-10 |
| DDT-316 | [Merge Cases quick action to deduplicate opportunities](./ddt-316-merge-cases-quick-action/README.md) | Done 2026-07-13 |
| DDT-317 | [Assign Lead chat button in case conversion](./ddt-317-assign-lead-chat-button/README.md) | Done 2026-07-13 |
| DDT-319 | [Court Appearance result modal, path and panel enhancements](./ddt-319-court-appearance-result-modal/README.md) | Done 2026-07-13 |
| DDT-322, DDT-323 | [Contract initial deposit and flat-fee discount logic](./ddt-322-contract-deposit-flat-fee/README.md) | Done 2026-07-13 |
| DDT-324 | [Card declines labeled as Issuer in the payment processor](./ddt-324-card-declines-labeled-issuer/README.md) | Done 2026-07-13 |
| DDT-325, DDT-1230 | [Replies to resolved email threads not reopening or notifying](./ddt-325-replies-resolved-threads/README.md) | Live since 2026-10-05 |
| DDT-351, DDT-1194, DDT-1488 | [Matter Current Stage drifting from the real stage](./ddt-351-stale-current-stage/README.md) | Live since 2026-11-02 |
| DDT-378, DDT-1469 | [Granting access to a report or dashboard](./ddt-378-report-dashboard-access/README.md) | Live since 2026-10-30 |
| DDT-467, DDT-637, DDT-760, DDT-1495, DDT-1515 | [Updating email signatures and sender details](./ddt-467-email-signature-updates/README.md) | Live since 2026-11-05 (latest change) |
| DDT-470, DDT-764, DDT-851, DDT-918, DDT-1001, DDT-1095, DDT-1103, DDT-1106, DDT-1228, DDT-1307, DDT-1321, DDT-1434, DDT-1470, DDT-1474, DDT-1502 | [Adding a citation type](./ddt-470-add-citation-type/README.md) | Live since 2026-07-21 (first request); latest addition live since 2026-11-05 |
| DDT-471, DDT-1003 | [Lead conversion fails: unable to find default record type](./ddt-471-lead-convert-default-record-type/README.md) | Live since 2026-09-15 |
| DDT-472, DDT-604 | [Payment plan capped at 20 installments](./ddt-472-payment-plan-installment-cap/README.md) | Done 2026-08-04 |
| DDT-475, DDT-761 | [Calendly meetings not syncing to Salesforce](./ddt-475-calendly-meetings-not-syncing/README.md) | Live since 2026-08-21 |
| DDT-476 | [Will County case search link in Clerk Links](./ddt-476-will-case-search-clerk-link/README.md) | Live since 2026-07-22 |
| DDT-478 | [Duplicate leads from the retargeting landing page](./ddt-478-duplicate-leads-landing-page/README.md) | Done 2026-07-27 |
| DDT-490 | [Settlement statement incident date wrong and PDF download for non-attorneys](./ddt-490-settlement-statement-incident-date/README.md) | Live since 2026-07-27 |
| DDT-491 | [Representation agreement not showing as sent or signed on the lead](./ddt-491-agreement-status-on-lead/README.md) | Done 2026-07-23 |
| DDT-493 | [Microsoft Teams option for attorney appearance](./ddt-493-teams-appearance-option/README.md) | Live since 2026-07-23 |
| DDT-494 | [Demands tab unavailable on companion matters](./ddt-494-demands-tab-companion-matters/README.md) | Live since 2026-09-11 |
| DDT-496 | [Hire Status filter on the Pending and Requested Liens report](./ddt-496-liens-report-hire-status/README.md) | Live since 2026-07-27 |
| DDT-497 | [Retiring duplicate Welcome Call fields on Opportunity](./ddt-497-retire-welcome-call-fields/README.md) | Live since 2026-07-24 |
| DDT-500, DDT-831, DDT-1501 | [Wrong saved card used as the default payment profile](./ddt-500-default-payment-card-wrong/README.md) | Done 2026-11-04 (last of the three) |
| DDT-508, DDT-513 | [Opening documents fail on a field that no longer exists](./ddt-508-opening-documents-missing-field/README.md) | Live since 2026-07-28 (DDT-508 closed 2026-07-27, DDT-513 closed 2026-07-28) |
| DDT-510 | [Production deploy and merge policy](./ddt-510-deploy-merge-policy/README.md) | Done 2026-07-28 (merged to `main`) |
| DDT-511, DDT-721 | [Matter Intake Owner stamped with the wrong user](./ddt-511-matter-intake-owner-wrong/README.md) | Live since 2026-08-13 (DDT-721). DDT-511 closed 2026-07-28. |
| DDT-517 | [Auto-resolving Seal and Expunge matters when the certified order arrives](./ddt-517-seal-expunge-auto-resolve/README.md) | Done 2026-07-29 |
| DDT-518 | [Bulk-closing overdue tasks in a list view](./ddt-518-bulk-close-overdue-tasks/README.md) | Done 2026-07-29 |
| DDT-519 | [Operations Admin role on PI matters and default case manager](./ddt-519-pi-operations-admin-role/README.md) | Live since 2026-07-28 |
| DDT-521 | [Assign to team member button on the injury email inbox](./ddt-521-injury-inbox-assign-button/README.md) | Live since 2026-07-28 (ticket closed 2026-07-29) |
| DDT-523 | [Compliance Completed checkbox on the Court Appearance layout](./ddt-523-compliance-completed-checkbox/README.md) | Live since 2026-07-28 (ticket closed 2026-07-29) |
