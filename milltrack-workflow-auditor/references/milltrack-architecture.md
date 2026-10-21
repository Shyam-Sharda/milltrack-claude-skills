# MillTrack Architecture (audit view)

This is a snapshot taken from the MillTrack SFDX source (sourceApiVersion 67.0)
supplied on 2026-10-20. Paths are under `force-app/main/default/`. If live source is
supplied and disagrees with this file, trust the source.

## Contents
1. Purpose and personas
2. Current implementation pattern
3. The `Dispatch_Status__c` rule
4. Current vs legacy automation
5. Security boundary (only the workflow-relevant parts)
6. What does not exist yet
7. Metadata the source references but did not supply

---

## 1. Purpose and personas

MillTrack Intelligence moves a milling-machine problem through these steps: customer
report → manager review → parts → technician visit → completion or temporary fix →
maintenance history. One customer runs it in production.

| Persona | Surface | Owns |
|---|---|---|
| Customer | Experience Cloud site `MillTrack Customer Portal`; Customer AI (`aiAuthoringBundles/Customer_AI`) | Reporting an issue; viewing their own Account's Assets, Cases, Work Orders, progress and service history; adding comments to their own open Cases through Customer AI. Read-only on Work Orders |
| Service Manager | Lightning apps `MillTrack_Service_Manager` / `MillTrack_Intelligence`; the `managerOperationsCenter` home workspace | Review, parts decisions, ordering and receiving, dispatch, follow-up after Temporary Fix, operations monitoring, calendar, analytics |
| Technician | Lightning app `MillTrack_Technician`; WorkOrder quick actions; `technicianCopilot` and `technicianAIChat` on the record page | Start work, request parts during a visit, Temporary Fix, completion |
| Admin | `adminHomeWorkspace` / `AdminHomeController` | Read-only dashboard. No workflow ownership |

A responsibility that moves to the wrong persona is a finding. Examples: a customer
choosing a workflow state, a technician approving follow-up or dispatching themselves,
or a manager action reachable from the portal.

## 2. Current implementation pattern

**LWC wizard → `@AuraEnabled` Apex controller.** Extend this pattern; don't route
around it.

- **Manager wizards** have `isExposed=false` and are opened as modals by
  `managerOperationsCenter`, based on the queue row's `actionName`. They fire
  `approved`/`complete`/`cancel` events, and the host refreshes with `refreshApex`.
  `ManagerOperationsCenterController` is read-only and builds every queue from
  `Dispatch_Status__c` (and from `Order_Status__c` for parts rows).
- **Technician wizards** are WorkOrder `ScreenAction` quick actions: `Start_Work_LWC`,
  `Request_Parts_LWC`, `Temporary_Fix_Follow_Up`, `Complete_Work_Order_LWC`.
  Controllers re-query with `FOR UPDATE`, check the starting `Dispatch_Status__c` and
  `OwnerId == UserInfo.getUserId()`, then write with `AccessLevel.SYSTEM_MODE`.
  Technicians have no FLS edit on `Dispatch_Status__c`, so they rely on that pattern.
- **Manager controllers** query `WITH USER_MODE` and write with `AccessLevel.USER_MODE`.
- **Customer controllers** are `with sharing`. They derive Contact and Account from the
  running user (`User.ContactId` → `Contact.AccountId`) and filter every query by that
  Account.
- Each write method wraps the Work Order, Case, Asset, parts and Maintenance Log DML in
  one `Savepoint` and rolls back on error. Related-record side effects belong in the
  **same transaction** as the `Dispatch_Status__c` change.
- Notifications use `Messaging.CustomNotification`. A failed notification becomes a
  `warningMessage`, never a rollback.

Live record-triggered Flows (each owns real behavior):
- `Create_Work_Order_Proposal_From_Case`: Case after-insert. Creates the WorkOrder at
  `Pending`.
- `SYS_01_Set_Asset_Service_Status`: Case after-insert. Sets Asset `Needs Service`.
- `Work_Order_Assignment_Sync`: WorkOrder after-update. Overlaps dispatch (see the
  ownership reference, §5).
- `Work_Order_Created_or_Pending_Dispatch`: dead. It enters on `Manager Reviewed`,
  which is not a picklist value.

## 3. The `Dispatch_Status__c` rule

`WorkOrder.Dispatch_Status__c` is MillTrack's canonical workflow status. It is a
**restricted** picklist with these API values:
`Pending`, `Parts to Order`, `Waiting on Parts`, `Ready_to_Assign` (label "Ready to
Assign"), `Assigned`, `In Progress`, `Temporary Fix`, `Completed`, `Cancelled`.

- Compare against API values. `'Ready to Assign'` never matches. A misspelled value
  written to a restricted picklist fails at runtime (for example the legacy
  `'Temporary FIx'`).
- Standard `WorkOrder.Status` is not a MillTrack workflow field. Populate it only when
  the platform needs a valid Status for record creation or compatibility. Using it to
  represent, infer or drive workflow state is a finding: entry criteria, decisions,
  queue filters, Apex conditions, customer progress. Existing code only *displays* it
  in one place (`TechnicianHomeController` puts `Status` in a display DTO). Reading it
  for display isn't a defect.
- A new `Dispatch_Status__c` value is an architectural change. It affects the Operations
  Center queues (`ManagerOperationsCenterController.queueStatuses`), the technician
  home filters (`TechnicianHomeController.EXCLUDED_SCHEDULED_STATUSES`), the Service
  Calendar (`Assigned`/`In Progress`), customer progress
  (`CustomerWorkOrderDetailsController.buildProgress` / `getCustomerCaseStatus`), list
  views, Flows and Apex conditions. Flag it unless explicit design approval is shown.
- Secondary statuses follow the Work Order; they never drive it. These are
  `Case.Status`, `Asset.Machine_Status__c`, and `Work_Order_Part__c.Order_Status__c`.

## 4. Current vs legacy automation

| Responsibility | Current owner | Legacy / overlapping path still `Active` |
|---|---|---|
| Review AI proposal | `managerReviewWorkOrderWizard` → `ManagerReviewWorkOrderController` | `Generate_Work_Order_Proposal_From_Case` (Screen Flow labelled "Review and Update Work Order"; `Work_Order_Record_Page1` references a quick action of that name whose metadata wasn't supplied) |
| Order required parts | `managerOrderPartsWizard` → `ManagerOrderPartsController.saveOrderingDecisions` | `X13_Order_Parts_From_Work_Order` |
| Order technician-requested part | `managerOrderPartsWizard` (technician mode) → `ManagerOrderPartsController.orderTechnicianRequestedPart` | `Order_Technician_Requested_Part` (granted in `MillTrack_Service_Manager`) |
| Receive parts | `managerReceivePartsWizard` → `ManagerReceivePartsController.receiveParts` | `Receive_Parts_From_Work_Order` |
| Dispatch | `managerDispatchTechnicianWizard` → `ManagerDispatchTechnicianController.dispatchTechnician` | `Dispatch_Technician` (quick action labelled "Dispatch Technician") **and** record-triggered `Work_Order_Assignment_Sync` |
| Start work | `technicianStartWork` (`Start_Work_LWC`) → `TechnicianStartWorkController.startWork` | `Start_Work` (quick action "ZZZ Start Work"); `Resume_Work` (no launcher in source) |
| Request parts | `technicianRequestPartsWizard` (`Request_Parts_LWC`) → `TechnicianRequestPartsController.submitPartsRequestV2` | `Request_Parts` (quick action "zzz Request Parts") |
| Temporary Fix | `technicianTemporaryFixWizard` (`Temporary_Fix_Follow_Up`) → `TechnicianTemporaryFixController.submitTemporaryFix` | `Temporary_Fix_From_Work_Order` (no quick action metadata in source) |
| Complete | `technicianCompleteWorkOrderWizard` (`Complete_Work_Order_LWC`) → `TechnicianCompleteWorkOrderController.completeWorkOrder` | `Complete_Work_Order` (quick action "ZZZ Complete Work Order") |
| Follow-up after Temporary Fix | `managerFollowUpWorkOrderWizard` → `ManagerFollowUpWorkOrderController.approveFollowUp` | none |
| Customer report issue | `customerReportAnIssue` → `CustomerReportAnIssueController.submitServiceRequest` (runs subflow `Create_Customer_Service_Case_Core`) | Screen Flow `Customer_Report_an_Issue` (the same subflow, so it's a second entry point and not a conflicting writer) |

The `ZZZ`/`zzz` labels mark actions that have been superseded. How each legacy path
differs in behavior is listed in the ownership reference, §6.

## 5. Security boundary (only the workflow-relevant parts)

Run a full permission audit only when permissions break the workflow directly. In that
case, defer to `salesforce-security-permission-auditor`.

- **Customer.** The sharing set `MillTrack_Customer_Account_Access` grants **Read** on
  Account, Asset, Case and WorkOrder (by Account), and on Maintenance_Log__c (by
  `Asset__c.Account`). `MillTrack_Customer_Portal_Access` grants WorkOrder Read only and
  Case Create/Read. A customer path that writes Work Orders, picks a workflow state,
  or trusts a client-supplied Account or Contact Id is a finding. The established
  write precedent is `CreateAuthorizedCaseComment` (`without sharing`), which
  re-verifies Contact ↔ Case ↔ Account in code and refuses closed Cases. Customer AI
  identity comes from `VerifiedCustomerId`, never from typed IDs.
- **Technician.** WorkOrder Read/Edit; FLS edit only on `Labor_Hours__c` and
  `Technician_Notes__c`. The Custom Permission
  `MillTrack_Technician_Restricted_Editing` drives the `Technician_Protect_*`
  validation rules. A technician action is expected to check that the Work Order is in
  the right starting state **and** is owned by the current user before writing in
  system mode.
- **Manager.** The `MillTrack_Service_Manager` permission set grants every
  `Manager*Controller` class, WorkOrder CRUD and FLS edit on the workflow fields. The
  public group `MillTrack_Service_Managers` gets Edit through owner-based sharing
  rules. `Service_Manager_Queue` is the manager-side owner of Cases and Work Orders.
- Dispatch picks technicians by `UserRole.DeveloperName = 'MillTrack_Technician'`.

Known gaps that already exist (mention them only if the audited change touches or
relies on them):
- `ManagerReviewWorkOrderController.approveWorkOrder` does **not** re-check `Pending`
  on the server. It relies on the Operations Center offering Review only for `Pending`
  rows.
- `TechnicianStartWorkController.startWork` checks for `Assigned` but **not** the
  owner. All the other technician controllers check both.
- The supplied permission sets don't grant Apex class access for most
  `Technician*Controller` / `Customer*Controller` classes. The granting source (a
  Profile or another assignment) wasn't supplied, so mark it Cannot Verify. Don't call
  it missing.

## 6. What does not exist yet

- No Apex triggers, no Queueable/Batch/Schedulable/`@future`, no scheduled Flows, no
  scheduled paths. A time-based change is MillTrack's **first** scheduled automation.
  That isn't a defect in itself, but a scheduled path must re-check
  `Dispatch_Status__c` when it runs, because the record can move on in the meantime.
- No Platform Events, outbound integrations or supplier API.
- None of these fields exist: a "temporary fix recorded at" field on WorkOrder, a
  follow-up due date or reminder count, a customer sign-off field, a "parts received
  at" date, or a cancellation reason. Existing evidence often covers the need: the
  Temporary Repair Maintenance Log `CreatedDate` or `Maintenance_Date__c`, the
  scheduled-path trigger time, `LastModifiedDate`.
- No current code sets `Cancelled`.

## 7. Metadata the source references but did not supply

Use Cannot Verify only if the audited change depends on one of these.
- CustomNotificationType DeveloperNames `Temporary_Repair_FollowUp`, `Parts_Request`,
  `New_Service_Case`, `Customer_Case_Update`
- Custom Permission `MillTrack_Technician_Restricted_Editing` (its definition)
- Maintenance_Log__c fields `Problem_Summary__c`, `Recommended_Fix_Snapshot__c`,
  `Tools_To_Bring_Snapshot__c`; Case `SLA_Due_Date__c`; Asset
  `Customer_Maintenance_List__c`
- Quick actions `WorkOrder.Order_Parts`, `Receive_Parts`, `Review_and_Update_Work_Order`,
  `Temporary_Fix`, `Temporary_Repair_Follow_Up` (referenced by
  `Work_Order_Record_Page1`)
- Page layouts and FlexiPage activation assignments. Whether a legacy quick action is
  still on a layout users see is unverified.
- Apex test classes, the full Case.Status value set, and Tool_Catalog__c fields
