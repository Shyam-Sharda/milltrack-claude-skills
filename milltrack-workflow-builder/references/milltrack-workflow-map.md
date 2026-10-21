# MillTrack Workflow Map — who owns what

Use this to find the existing owner of a responsibility before proposing anything new.
"Owner" = the component that performs the state change today. Paths are under
`force-app/main/default/`.

## Contents
1. Manager Operations Center queues
2. Service Manager workflows
3. Technician workflows
4. Customer workflows
5. Automation inventory (Flows) — live, helper, legacy
6. Overlap / duplication warnings
7. Quick lookup: "I need to…" → start here

---

## 1. Manager Operations Center (`lwc/managerOperationsCenter`, `ManagerOperationsCenterController.getOperationsData`)

Read-only controller; one cacheable call returns every queue. The LWC opens a child wizard
by `actionName` and calls `refreshApex` after the wizard fires `approved`/`complete`.

| Queue (UI) | Source rows | Row action → wizard |
|---|---|---|
| New Cases | Case `Status = 'New'` | `VIEW_CASE` (navigate) |
| Requires Review | WO `Pending` | `REVIEW_WORK_ORDER` → `managerReviewWorkOrderWizard` |
| Follow-Up Required | WO `Temporary Fix` | `FOLLOW_UP_WORK_ORDER` → `managerFollowUpWorkOrderWizard` |
| Parts to Order | WO `Parts to Order` + WOP `Requested` on WOs `Waiting on Parts` | `ORDER_REQUIRED_PARTS` / `ORDER_TECHNICIAN_REQUESTED_PART` → `managerOrderPartsWizard` (mode) |
| Waiting on Parts | WOP `Ordered`/`Shipped` on WOs `Waiting on Parts` | `RECEIVE_PARTS` → `managerReceivePartsWizard` |
| Ready to Assign | WO `Ready_to_Assign` | `DISPATCH_TECHNICIAN` → `managerDispatchTechnicianWizard` |
| Assigned Visits | WO `Assigned` | navigate |
| Active Repairs | WO `In Progress` | navigate |

The LWC also has a generic Screen Flow modal (`lightning-flow` driven by
`QueueItem.flowApiName`, `handleFlowStatus`); the controller currently sets
`flowApiName = null` on every row, so it is an unused extension point.
`managerReviewWorkOrderWizard` embeds `MGR_01_Regenerate_Work_Order_AI_Proposal` as a hidden
`lightning-flow` to regenerate the AI proposal.

A new manager queue or stage = add a list + count to `OperationsData`, a branch in
`loadWorkOrderQueues`/`configureWorkOrderAction`, and the matching section/handler in the
LWC. Don't build a second dashboard.

`managerHomeWorkspace` composes `managerOperationsCenter`, `managerServiceCalendar`
(WO `Assigned`/`In Progress` by `Scheduled_Visit__c`), `managerServiceIntelligence`
(analytics over Maintenance Logs).

## 2. Service Manager workflows

| Workflow | LWC | Apex (methods) | From → To `Dispatch_Status__c` | Side effects |
|---|---|---|---|---|
| Review AI Work Order proposal | `managerReviewWorkOrderWizard` | `ManagerReviewWorkOrderController.getReviewData / approveWorkOrder / getToolCatalog`; `ManagerPartCatalogController.getCompatibleParts`; `GetRelevantHistoricalRepairsForCase.getHistoricalRepairsForLwc` | `Pending` → `Parts to Order` \| `Ready_to_Assign` | Sets plan fields, `Service_Manager__c` = me; Case → `In Progress`; inserts WOP `Requested` |
| Follow-up after temporary fix | `managerFollowUpWorkOrderWizard` | `ManagerFollowUpWorkOrderController.getFollowUpData / approveFollowUp / getToolCatalog` | `Temporary Fix` → `Parts to Order` \| `Ready_to_Assign` | Reads latest `Follow_Up_Required__c` Maintenance Log; updates fix summary/tools; `Service_Manager__c` = me; Case → `In Progress`; inserts WOP `Requested` |
| Order required parts | `managerOrderPartsWizard` (mode = required) | `ManagerOrderPartsController.getOrderPartsContext / saveOrderingDecisions` | `Parts to Order` → `Parts to Order` \| `Waiting on Parts` \| `Ready_to_Assign` | WOP → `Ordered`/`Cancelled`; Case aligned |
| Order technician-requested part | `managerOrderPartsWizard` (technician mode) | `ManagerOrderPartsController.getTechnicianRequestedPartContext / orderTechnicianRequestedPart` | stays `Waiting on Parts` | WOP `Requested` → `Ordered` (+ supplier fields) |
| Receive parts | `managerReceivePartsWizard` | `ManagerReceivePartsController.getReceivePartsContext / receiveParts` | `Waiting on Parts` → `Ready_to_Assign` when no WOP remains `Requested`/`Ordered`/`Shipped`; else unchanged | Selected WOP → `Received`; Case → `In Progress` (all received) or `Waiting on Parts`; Asset untouched |
| Dispatch technician | `managerDispatchTechnicianWizard` | `ManagerDispatchTechnicianController.getDispatchContext / getTechnicianSchedule / dispatchTechnician` | `Ready_to_Assign` → `Assigned` | `Suggested_Technician__c` + `OwnerId` = tech; `Scheduled_Visit__c`; Case → `Scheduled`; techs = role `MillTrack_Technician`; 30-min schedule buffer |
| Calendar | `managerServiceCalendar` | `ManagerServiceCalendarController.getCalendarData` | read-only | |
| Analytics | `managerServiceIntelligence` | `ManagerServiceIntelligenceController.getServiceIntelligence` | read-only | |
| Record-page summaries | `managerWorkOrderSummary`, `managerCaseSummary`, `maintenanceLogManagerContext`, `maintenanceLogServiceSummary`, `partCatalogRecordSummary`, `serviceManualRecordSummary`, `workOrderPartRecordSummary` | none (LDS / record data) | read-only | |

## 3. Technician workflows

All launched as WorkOrder LWC quick actions; each controller checks starting status and
`OwnerId == UserInfo.getUserId()` (Start Work checks status only), then writes in `SYSTEM_MODE`.

| Workflow | LWC (quick action) | Apex | From → To | Side effects |
|---|---|---|---|---|
| Home / my work | `technicianHomeDashboard` | `TechnicianHomeController.getHomeData` | read-only | Active = `In Progress`; upcoming excludes In Progress/Completed/Waiting on Parts/Parts to Order/Temporary Fix |
| Start work | `technicianStartWork` (`Start_Work_LWC`) | `TechnicianStartWorkController.startWork` | `Assigned` → `In Progress` | `Work_Started_Date_Time__c`; Case → `In Progress` |
| Request parts (ends visit) | `technicianRequestPartsWizard` (`Request_Parts_LWC`) | `TechnicianRequestPartsController.getRequestContext / getManufacturerParts / submitPartsRequestV2` (`submitPartsRequest` kept for single-part compatibility) | `In Progress` → `Waiting on Parts` | Inserts WOP `Requested` + `Technician_Requested__c`; visit Maintenance Log (`Temporary Repair` or `Inspection`, follow-up true); owner → `Service_Manager_Queue`; clears tech/visit; Case → `Waiting on Parts`; `Parts_Request` notification |
| Temporary fix & follow-up | `technicianTemporaryFixWizard` (`Temporary_Fix_Follow_Up`) | `TechnicianTemporaryFixController.getTemporaryFixContext / submitTemporaryFix` | `In Progress` → `Temporary Fix` | Maintenance Log `Temporary Repair` (follow-up true, resolved false, plan snapshots); Asset → `Temporary Fix Applied`; owner → `Service_Manager_Queue`; clears tech/visit; adds labor; appends notes; Case → `Follow-Up Required`; `Temporary_Repair_FollowUp` notification to `Service_Manager__c` (warning if null) |
| Complete work order | `technicianCompleteWorkOrderWizard` (`Complete_Work_Order_LWC`) | `TechnicianCompleteWorkOrderController.getCompletionContext / completeWorkOrder` | `In Progress` → `Completed` | Maintenance Log `Repair` (resolved true); Asset → `Active`; Case → `Closed`; `Work_Ended_Date_Time__c`; owner → `Service_Manager__c`; clears tech |
| Copilot / briefing | `technicianCopilot` (record page) | `TechnicianCopilotController.getCopilotContext` | read-only | |
| Technician AI | `technicianAIChat` (record page) | `TechnicianAIChatController.sendMessage` → agent `MillTrack_Technician_Agent` | read-only | |
| Header | `technicianWorkOrderHeader` | none | read-only | |

## 4. Customer workflows (Experience Cloud)

Controllers are `with sharing`, derive Contact/Account from the running user, and filter
every query by that Account (`WITH USER_MODE`).

| Workflow | LWC | Apex | Writes |
|---|---|---|---|
| Portal home | `customerPortalHome` | `CustomerPortalHomeController.getHomeData` | — |
| Report an issue | `customerReportAnIssue` | `CustomerReportAnIssueController.getReportIssueData / submitServiceRequest` | Creates Case (→ triggers WO creation flow) |
| Report an issue (Screen Flow alt.) | Flow `Customer_Report_an_Issue` | subflow `Create_Customer_Service_Case_Core` + `GetServiceManagerQueue` | Creates Case, Origin `Web` |
| Asset details | `customerAssetDetails` | `CustomerAssetDetailsController.getAssetDetails / saveMaintenanceList` | Asset `Customer_Maintenance_List__c` (USER_MODE) |
| Asset service history | `customerAssetServiceHistory` | `CustomerAssetServiceHistoryController.getServiceHistory` | — |
| Service request list / details | `customerServiceRequestList`, `customerServiceRequestDetails`, `customerCaseDetails` | `CustomerServiceRequestListController`, `CustomerServiceRequestDetailsController`, `CustomerCaseDetailsController` | — |
| Work Order details + progress | `customerWorkOrderDetails` | `CustomerWorkOrderDetailsController.getWorkOrderDetails` | — (read-only). `buildProgress` maps `Dispatch_Status__c` → customer stage text; `getCustomerCaseStatus` maps Case.Status |
| Customer AI | `millTrackChatLauncher`, `millTrackMessagingHeader` | Agent `Customer_AI` → `CUST_02` (create Case), `CUST_05` (comment; open Case only), `CUST_06` (resolve Case by number), `Get_Authorized_Customer_Assets_Agent`, `ResolveAuthorizedAssetSelection` | Case, CaseComment |
| Shared chrome | `millTrackHeader`, `millTrackPageHero`, `customerBackButton` | — | — |

Customer write precedent: `CreateAuthorizedCaseComment` (`without sharing`, invocable)
re-verifies Contact ↔ Case ↔ Account ownership in code before writing in system mode, and
refuses closed Cases. A completed Work Order closes its Case, so post-completion customer
actions cannot reuse the case-comment path as-is.

## 5. Automation inventory (Flows)

**Record-triggered (live):**
| Flow | Trigger | Does |
|---|---|---|
| `Create_Work_Order_Proposal_From_Case` | Case after-insert, Status ≠ Closed, proposal flag false | AI proposal → WorkOrder `Pending` owned by `Service_Manager_Queue`; Asset `Needs Service`; flag Case; `New_Service_Case` notification to queue members |
| `SYS_01_Set_Asset_Service_Status` | Case after-insert, AssetId set, Status New | Asset `Needs Service` unless Retired/Out of Service/Temporary Fix Applied |
| `Work_Order_Assignment_Sync` | WorkOrder after-update, technician set, status `Ready_to_Assign` or `Assigned`, technician changed | `Dispatch_Status__c` → `Assigned`, Owner → technician |
| `Work_Order_Created_or_Pending_Dispatch` | WorkOrder after-save on `Manager Reviewed` | **Dead** — value doesn't exist |

**Autolaunched helpers (called by AI/flows):** `Build_Technician_Dispatch_Context`,
`Get_Machine_History`, `MGR_01_Regenerate_Work_Order_AI_Proposal`,
`Get_Authorized_Customer_Assets_Agent`, `Create_Customer_Service_Case_Core`, `CUST_02/04/05/06_*`.
Drafts: `CUST_05_Add_Authorized_Case_Comment_V2`, `Customer_Create_Case`.

**Screen Flows still active but superseded or secondary:** `Start_Work`, `Request_Parts`,
`Complete_Work_Order` (quick actions labelled `ZZZ`/`zzz`), `Temporary_Fix_From_Work_Order`,
`Resume_Work` (no launcher in source), `Dispatch_Technician` (quick action, active label),
`Generate_Work_Order_Proposal_From_Case`, `X13_Order_Parts_From_Work_Order`,
`Receive_Parts_From_Work_Order`, `Order_Technician_Requested_Part` (granted to managers).
Treat the LWC/Apex path as the owner; touching these flows needs explicit confirmation.

**None exist:** scheduled flows, scheduled paths, Apex triggers, Schedulable/Batch/Queueable.

## 6. Overlap / duplication warnings

- Assignment: `ManagerDispatchTechnicianController` + `Work_Order_Assignment_Sync` +
  `Dispatch_Technician` flow all set `Assigned`. Don't add a fourth.
- Receive parts: `ManagerReceivePartsController` (current, Asset untouched) vs
  `Receive_Parts_From_Work_Order` (legacy, sets Asset `Ready to Assign`).
- Temporary fix: `TechnicianTemporaryFixController` (current) vs `Temporary_Fix_From_Work_Order`
  (legacy, misspelled `Temporary FIx`).
- Case-create Asset update: two flows both set `Needs Service`.
- Manager notification on temporary fix already fires immediately from Apex.

## 7. Quick lookup

| Request touches… | Start at |
|---|---|
| Something a manager does on a queue row | `managerOperationsCenter` + the wizard for that `actionName` |
| Temporary fix / follow-up | `TechnicianTemporaryFixController` → `ManagerFollowUpWorkOrderController` |
| Parts lifecycle | `ManagerReviewWorkOrderController`/`ManagerFollowUpWorkOrderController` (Requested) → `ManagerOrderPartsController` (Ordered) → `ManagerReceivePartsController` (Received) |
| Dispatch / scheduling | `ManagerDispatchTechnicianController`, `managerServiceCalendar` |
| Technician visit outcome | `TechnicianCompleteWorkOrderController`, `TechnicianTemporaryFixController`, `TechnicianRequestPartsController` |
| What the customer sees of progress | `CustomerWorkOrderDetailsController.buildProgress` / `getCustomerCaseStatus` |
| New customer action | `customerWorkOrderDetails` / `customerServiceRequestDetails` + a new method on the matching `Customer*Controller` |
| Time-based reminders | Nothing exists — first scheduled automation in MillTrack |
| Initial Case → Work Order | `Create_Work_Order_Proposal_From_Case` |
