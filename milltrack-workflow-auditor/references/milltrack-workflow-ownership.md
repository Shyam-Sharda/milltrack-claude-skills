# MillTrack Workflow Ownership

Use this to find who owns a transition today before deciding whether a change belongs.
The "owner" is the component that performs the state change now. Paths are under
`force-app/main/default/`.

## Contents
1. Service Manager workflows
2. Technician workflows
3. Customer workflows
4. System (record-triggered) automation
5. Overlapping owners: don't add another
6. Legacy Screen Flows and how they differ from the current owner
7. Read-only components (never workflow owners)
8. Quick lookup: change touches X → compare against Y

---

## 1. Service Manager workflows

Every manager wizard is hosted in `managerOperationsCenter` and launched from a queue
row's `actionName`. The controllers use USER_MODE.

| Workflow | LWC | Apex owner | From → To `Dispatch_Status__c` | Server-side start check | Major side effects |
|---|---|---|---|---|---|
| Review AI proposal | `managerReviewWorkOrderWizard` | `ManagerReviewWorkOrderController.approveWorkOrder` | `Pending` → `Parts to Order` (parts selected) or `Ready_to_Assign` (none) | **None**: relies on the queue offering Review only for `Pending` rows | Writes `Problem_Summary__c`, `Recommended_Solution__c`, `Recommended_Fix_Summary__c` (≤255), `Recommended_Tools__c`, `Recommended_Parts__c`, `Previous_Failure__c`, `Priority`; `Service_Manager__c` = current user; Case → `In Progress`; inserts WOP `Requested` (and rejects a duplicate active request for the same catalog part). Asset untouched |
| Follow-up after Temporary Fix | `managerFollowUpWorkOrderWizard` | `ManagerFollowUpWorkOrderController.approveFollowUp` | `Temporary Fix` → `Parts to Order` or `Ready_to_Assign` | Yes (`Temporary Fix`) | Reads the newest `Follow_Up_Required__c = true` Maintenance Log; updates `Recommended_Fix_Summary__c` and `Recommended_Tools__c`; `Service_Manager__c` = current user; Case → `In Progress`; inserts WOP `Requested`. **Asset untouched** (stays `Temporary Fix Applied`). Same Work Order, no new one |
| Order required parts | `managerOrderPartsWizard` (required mode) | `ManagerOrderPartsController.saveOrderingDecisions` | `Parts to Order` → `Parts to Order` (some still Requested) or `Waiting on Parts` (all ordered/cancelled, at least one ordered) or `Ready_to_Assign` (all cancelled) | Yes (`Parts to Order`) | WOP → `Ordered` (sets `Ordered_By__c`) or `Cancelled`; Case → `In Progress` / `Waiting on Parts` / `In Progress` respectively. Asset untouched |
| Order technician-requested part | `managerOrderPartsWizard` (technician mode) | `ManagerOrderPartsController.orderTechnicianRequestedPart` | stays `Waiting on Parts` | Yes (WOP `Requested` and WO `Waiting on Parts`) | WOP → `Ordered` plus required supplier name, phone and order URL |
| Receive parts | `managerReceivePartsWizard` | `ManagerReceivePartsController.receiveParts` | `Waiting on Parts` → `Ready_to_Assign` once no WOP is Requested/Ordered/Shipped; otherwise unchanged | Yes (`Waiting on Parts`; selected parts must be Ordered/Shipped) | Selected WOP → `Received`; Case → `In Progress` (all received) or `Waiting on Parts`. **Asset intentionally untouched** |
| Dispatch technician | `managerDispatchTechnicianWizard` | `ManagerDispatchTechnicianController.dispatchTechnician` | `Ready_to_Assign` → `Assigned` | Yes (`Ready_to_Assign`) | Rejects if any WOP is Requested/Ordered/Shipped; technician must be active with role `MillTrack_Technician`; visit ≥ now; no Assigned/In Progress visit for that technician within ±30 min. Sets `Suggested_Technician__c` and `OwnerId` = technician, `Scheduled_Visit__c`; Case → `Scheduled`. Asset untouched. No notification |

The Operations Center queues come from `ManagerOperationsCenterController`: New Cases
(Case `New`), Requires Review (`Pending`), Follow-Up Required (`Temporary Fix`), Parts
to Order (WO `Parts to Order` and WOP `Requested` on WOs `Waiting on Parts`), Waiting on
Parts (WOP `Ordered`/`Shipped` on WOs `Waiting on Parts`), Ready to Assign
(`Ready_to_Assign`), Assigned Visits (`Assigned`), Active Repairs (`In Progress`). If
duplicates exist, it shows the newest non-Completed/Cancelled Work Order per Case. One
active Work Order per Case is the working assumption.

## 2. Technician workflows

These run as WorkOrder quick actions. Each controller re-queries `FOR UPDATE`, checks
state and owner, and writes in SYSTEM_MODE.

| Workflow | LWC (quick action) | Apex owner | From → To | Start and owner check | Major side effects |
|---|---|---|---|---|---|
| Start work | `technicianStartWork` (`Start_Work_LWC`) | `TechnicianStartWorkController.startWork` | `Assigned` → `In Progress` | `Assigned` only. **No owner check** | `Work_Started_Date_Time__c` = now; Case → `In Progress` |
| Request parts (ends the visit) | `technicianRequestPartsWizard` (`Request_Parts_LWC`) | `TechnicianRequestPartsController.submitPartsRequestV2` (`submitPartsRequest` wraps it for one part) | `In Progress` → `Waiting on Parts` | `In Progress` and owner | Inserts WOP `Requested` with `Technician_Requested__c = true` (active catalog part matching the Asset's manufacturer, ≤25); visit Maintenance Log (`Temporary Repair` if `temporaryRepairApplied`, else `Inspection`; `Follow_Up_Required__c = true`, `Issue_Resolved__c = false`, visit labor, `Performed_By__c`); WO owner → `Service_Manager_Queue`, clears `Scheduled_Visit__c` and `Suggested_Technician__c`, adds labor, appends notes; Case → `Waiting on Parts`; `Parts_Request` notification to `Service_Manager__c`. Asset untouched |
| Temporary Fix | `technicianTemporaryFixWizard` (`Temporary_Fix_Follow_Up`) | `TechnicianTemporaryFixController.submitTemporaryFix` | `In Progress` → `Temporary Fix` | `In Progress` and owner; AssetId required | Maintenance Log `Temporary Repair` (`Issue_Resolved__c = false`, `Follow_Up_Required__c = true`, snapshots of Problem Summary, Recommended Fix and Tools, visit labor, `Performed_By__c`); Asset → `Temporary Fix Applied`, `Last_Service_Date__c` = today; WO owner → `Service_Manager_Queue`, clears technician and visit, adds labor, appends notes; Case → `Follow-Up Required`; `Temporary_Repair_FollowUp` notification to `Service_Manager__c` (a warning if null) |
| Complete | `technicianCompleteWorkOrderWizard` (`Complete_Work_Order_LWC`) | `TechnicianCompleteWorkOrderController.completeWorkOrder` | `In Progress` → `Completed` | `In Progress` and owner; AssetId and `Service_Manager__c` required | Repair summary, Root Cause and labor > 0 required; Maintenance Log `Repair` (`Issue_Resolved__c = true`, `Follow_Up_Required__c = false`, `Root_Cause__c`, `Parts_Used__c`, visit labor, `Performed_By__c`); Asset → `Active`, `Last_Service_Date__c` = today; Case → `Closed`; WO adds labor, appends notes, `Work_Ended_Date_Time__c` = now, owner → `Service_Manager__c`, clears `Suggested_Technician__c`, **keeps** `Scheduled_Visit__c`. WOP statuses untouched |

The technician home (`TechnicianHomeController`) treats `In Progress` as active and
leaves `In Progress`/`Completed`/`Waiting on Parts`/`Parts to Order`/`Temporary Fix`
out of "upcoming". It filters on `Suggested_Technician__c`.

## 3. Customer workflows (Experience Cloud and Customer AI)

| Workflow | Entry | Owner | Writes |
|---|---|---|---|
| Report an issue | `customerReportAnIssue` | `CustomerReportAnIssueController.submitServiceRequest` → subflow `Create_Customer_Service_Case_Core` | Case `New`, Origin `Web`, owner `Service_Manager_Queue`. Account, Contact and Asset are derived from the running user, and the Asset is verified against the Account |
| Report an issue (alternative) | Screen Flow `Customer_Report_an_Issue` | the same subflow | the same |
| Customer AI service request | `Customer_AI` agent | `CUST_04_Prepare_*` → `CUST_02_Create_Authorized_Service_Case` | Case `New`, Origin `Customer AI`, `Service_Request_Fingerprint__c` de-dup |
| Customer AI case comment | `Customer_AI` agent | `CUST_05_Add_Authorized_Case_Comment` → `CreateAuthorizedCaseComment` | Public CaseComment on an open, owned Case. `Customer_Case_Update` notification |
| Customer AI case lookup | `Customer_AI` agent | `CUST_06_Resolve_Authorized_Customer_Case` | none (read and verify) |
| Asset maintenance list | `customerAssetDetails` | `CustomerAssetDetailsController.saveMaintenanceList` | Asset `Customer_Maintenance_List__c` (USER_MODE) |
| Work Order progress | `customerWorkOrderDetails` | `CustomerWorkOrderDetailsController.getWorkOrderDetails` / `buildProgress` | none. Maps `Dispatch_Status__c` to customer stage text |
| Lists and details | `customerPortalHome`, `customerServiceRequestList`, `customerServiceRequestDetails`, `customerCaseDetails`, `customerAssetServiceHistory` | matching `Customer*Controller` | none |

Customers never set `Dispatch_Status__c`, and they never choose or skip a workflow
stage.

## 4. System (record-triggered) automation

| Flow | Trigger | Does |
|---|---|---|
| `Create_Work_Order_Proposal_From_Case` | Case after-insert, Status ≠ Closed, `Work_Order_Proposal_Created__c = false` | AI proposal (Prompt Builder + `ParseWorkOrderProposalFields`) → creates the WorkOrder at `Pending`, owner `Service_Manager_Queue`, linked to Case, Asset, Account and Contact; Asset → `Needs Service` (no guard); sets `Work_Order_Proposal_Created__c = true`; `New_Service_Case` notification to queue members |
| `SYS_01_Set_Asset_Service_Status` | Case after-insert, AssetId set, Status `New` | Asset → `Needs Service` unless Retired, Out of Service or Temporary Fix Applied |
| `Work_Order_Assignment_Sync` | WorkOrder after-update; entry: `Suggested_Technician__c` not null AND `Dispatch_Status__c` in (`Ready_to_Assign`, `Assigned`); decision: `Suggested_Technician__c` ≠ prior | Sets `Dispatch_Status__c = Assigned` and `OwnerId` = technician. **No** outstanding-parts check, **no** `Scheduled_Visit__c`, **no** Case `Scheduled` |
| `Work_Order_Created_or_Pending_Dispatch` | WorkOrder after-save on `Dispatch_Status__c = 'Manager Reviewed'` | **Dead**: that value doesn't exist in the restricted picklist |

Autolaunched helpers that own no transitions: `Build_Technician_Dispatch_Context`,
`Get_Machine_History`, `MGR_01_Regenerate_Work_Order_AI_Proposal`,
`Get_Authorized_Customer_Assets_Agent`. Drafts: `CUST_05_Add_Authorized_Case_Comment_V2`
and `Customer_Create_Case`.

## 5. Overlapping owners: don't add another

- **Assignment (`Ready_to_Assign` → `Assigned`)** has three writers:
  `ManagerDispatchTechnicianController.dispatchTechnician` (the owner),
  `Work_Order_Assignment_Sync` (it also re-fires right after the Apex dispatch,
  because the technician changed and the status is now `Assigned`), and the
  `Dispatch_Technician` Screen Flow. Any new automation that sets `Assigned`,
  `Suggested_Technician__c` or the technician `OwnerId` is a **fourth** writer. The fix
  is to route through `dispatchTechnician`. Don't extend the sync flow.
- **Asset `Needs Service` on Case create** has two writers:
  `Create_Work_Order_Proposal_From_Case` (unguarded) and `SYS_01` (guarded).
- **Temporary Fix manager notification** already fires immediately from
  `TechnicianTemporaryFixController`. A later *reminder* is not a duplicate. A second
  *immediate* notification on the same event would be.
- **Customer issue report** has two entry points (the LWC and the Screen Flow) sharing
  one subflow. That's the same writer, so it isn't a duplication finding.
- **Receive, order, start, request parts, complete and Temporary Fix** each have a
  legacy Screen Flow twin (§6). They're reachable only if their quick actions are on
  a layout, which is unverified.

## 6. Legacy Screen Flows and how they differ from the current owner

All are `Active`. Treat them as duplication risks, not as places to make changes. If a
change modifies one of these while the current owner holds that behavior, flag the
architecture mismatch and point the fix at the current owner.

| Legacy flow | Differs from the current owner by |
|---|---|
| `Dispatch_Technician` | Allows `Temporary Fix` → `Assigned` directly, skipping manager follow-up. Only asks the manager to confirm parts are ready (`partsReadyConfirmed`) and never counts outstanding WOPs. No ±30-min conflict check. Sets Asset → `Needs Service` (the Apex owner leaves the Asset alone) |
| `X13_Order_Parts_From_Work_Order` ("Add Required Parts") | Creates new WOPs straight at `Ordered` (quantity 1) instead of ordering existing `Requested` ones; WO → `Waiting on Parts`; Asset → `Waiting on Parts` (the current path never writes that value) |
| `Receive_Parts_From_Work_Order` | Asset → `Ready to Assign` (the current path leaves the Asset alone) |
| `Temporary_Fix_From_Work_Order` | Writes `Dispatch_Status__c = 'Temporary FIx'` (capital I), which a restricted picklist rejects at runtime; puts the reason in `Root_Cause__c` on an unresolved log |
| `Complete_Work_Order` ("ZZZ") | No `Dispatch_Status__c` eligibility decision in the supplied XML |
| `Resume_Work` | Sets `In Progress` with no starting-state check; no launcher in source |
| `Start_Work` ("ZZZ"), `Request_Parts` ("zzz") | Functionally close to the current owners; superseded by the `_LWC` actions |
| `Generate_Work_Order_Proposal_From_Case` ("Review and Update Work Order") | Sets `Parts to Order` / `Ready_to_Assign` and `Service_Manager__c` with no starting-state decision; doesn't create WOP records or update the Case |
| `Order_Technician_Requested_Part` | Same checks as `orderTechnicianRequestedPart` (WOP `Requested`, WO `Waiting on Parts`); granted to managers |

## 7. Read-only components (never workflow owners)

`managerOperationsCenter`, `managerServiceCalendar`, `managerServiceIntelligence`,
`managerHomeWorkspace`, `managerWorkOrderSummary`, `managerCaseSummary`,
`maintenanceLogManagerContext`, `maintenanceLogServiceSummary`,
`partCatalogRecordSummary`, `serviceManualRecordSummary`, `workOrderPartRecordSummary`,
`technicianHomeDashboard`, `technicianCopilot`, `technicianAIChat` (Technician AI is
read-only grounding), `technicianWorkOrderHeader`, `adminHomeWorkspace`. A change
that makes one of these *write* workflow state moves ownership into a display
component. That's a finding.

## 8. Quick lookup

| The change touches… | Compare it against |
|---|---|
| `Pending` handling, AI proposal approval | `ManagerReviewWorkOrderController.approveWorkOrder` |
| Anything after a Temporary Fix | `TechnicianTemporaryFixController` → `ManagerFollowUpWorkOrderController` |
| Parts Requested → Ordered → Received | review/follow-up (Requested) → `ManagerOrderPartsController` → `ManagerReceivePartsController` |
| Technician assignment, owner or schedule | `ManagerDispatchTechnicianController.dispatchTechnician` (plus §5 overlaps) |
| A visit outcome | `TechnicianCompleteWorkOrderController`, `TechnicianTemporaryFixController`, `TechnicianRequestPartsController` |
| What the customer sees | `CustomerWorkOrderDetailsController.buildProgress` / `getCustomerCaseStatus` |
| Case creation and the first Work Order | `Create_Work_Order_Proposal_From_Case`, `SYS_01` |
| Time-based reminders | Nothing exists yet. This would be the first scheduled automation |
