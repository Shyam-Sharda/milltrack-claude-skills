# MillTrack Architecture

Snapshot derived from the MillTrack SFDX source (sourceApiVersion 67.0) supplied on 2026-10-20.
If the live repo is available, prefer it over this file when the two disagree.

## Contents
1. Purpose
2. Personas and where they work
3. Implementation layers (what is current vs legacy)
4. Service lifecycle and the `Dispatch_Status__c` state machine
5. Secondary status fields (and why they are not the workflow status)
6. AI boundaries
7. Security model in one page
8. Known drift and overlaps

---

## 1. Purpose

MillTrack Intelligence is a Salesforce service-management app for dental CAD/CAM milling
machines (Roland/DGSHAPE, Straumann, Amann Girrbach, Ceramill). It moves a machine
problem from customer report → manager review → parts → technician visit → completion or
temporary fix → maintenance history, so each persona sees only the context and actions it
needs. One customer currently runs it in production.

## 2. Personas and where they work

| Persona | Surface | Entry points in source |
|---|---|---|
| Customer | Experience Cloud site `MillTrack Customer Portal` (network + site metadata); Customer AI (Agentforce) via `millTrackChatLauncher` / `millTrackMessagingHeader` | `customer*` LWCs + `Customer*Controller` Apex; `CUST_0x_*` flows used by Customer AI |
| Service Manager | Lightning app `MillTrack_Service_Manager` / `MillTrack_Intelligence`; Home page `Service_Manager_Home_Page` hosts `managerOperationsCenter` | `manager*` LWCs + `Manager*Controller` Apex |
| Field Technician | Lightning app `MillTrack_Technician`; `Technician_Work_Order_Record_Page` hosts `technicianCopilot` + `technicianAIChat`; WorkOrder quick actions launch technician wizards | `technician*` LWCs + `Technician*Controller` Apex; Technician AI (Agentforce) |
| Admin | `MillTrack_Admin_Home`, `adminHomeWorkspace` | `AdminHomeController` (read-only dashboard) |

Persona responsibilities (keep them separate):
- **Customer**: report issue, view own Assets / Service Requests (Cases) / Work Orders and
  their progress, service history, talk to Customer AI. Read-only on Work Orders today.
- **Service Manager**: review new Cases and AI Work Order proposals, decide parts, order and
  receive parts, dispatch technicians, review temporary-fix follow-ups, calendar, analytics.
- **Technician**: assigned Work Orders, Start Work, Copilot/briefing, Technician AI,
  Request Parts, Temporary Fix & Follow-Up, Complete Work Order.

## 3. Implementation layers

**Current pattern (prefer extending this): LWC wizard → `@AuraEnabled` Apex controller.**
- Manager actions are LWC wizards *hosted inside* `managerOperationsCenter` (the wizards
  are `isExposed=false`; the Operations Center opens them as modals based on the row's
  `actionName` and refreshes via `refreshApex` on `approved`/`complete`/`cancel` events).
- Technician actions are LWC `RecordAction` screen quick actions on WorkOrder
  (`Start_Work_LWC`, `Request_Parts_LWC`, `Temporary_Fix_Follow_Up`, `Complete_Work_Order_LWC`).
- Customer pages are LWCs on Experience Cloud pages calling `with sharing` controllers that
  re-derive the user's Contact/Account and filter by it.

**Flows that are still live and own behaviour:**
- Record-triggered on Case create: `Create_Work_Order_Proposal_From_Case` (creates the
  Work Order at `Pending`, AI proposal fields, owner = `Service_Manager_Queue`, sends
  `New_Service_Case` notification, sets Asset `Needs Service`, flags
  `Case.Work_Order_Proposal_Created__c`) and `SYS_01_Set_Asset_Service_Status`.
- Record-triggered on WorkOrder update: `Work_Order_Assignment_Sync` (see §8).
- Autolaunched helpers for AI: `Build_Technician_Dispatch_Context`, `Get_Machine_History`,
  `MGR_01_Regenerate_Work_Order_AI_Proposal`, `Get_Authorized_Customer_Assets_Agent`,
  `CUST_02/04/05/06_*` (Customer AI service-request and case-comment actions).
- Customer Screen Flow `Customer_Report_an_Issue` → subflow `Create_Customer_Service_Case_Core`.
- Manager Screen Flows still granted/launchable: `Order_Technician_Requested_Part`,
  `Dispatch_Technician` (quick action), `Generate_Work_Order_Proposal_From_Case`
  ("Review and Update Work Order"), `X13_Order_Parts_From_Work_Order`, `Receive_Parts_From_Work_Order`.

**Legacy Screen Flows superseded by LWC + Apex** (quick action labels prefixed `ZZZ`/`zzz`,
or no launcher in source): `Start_Work`, `Request_Parts`, `Complete_Work_Order`,
`Temporary_Fix_From_Work_Order`, `Resume_Work`. Do not extend these; extend the LWC/Apex
equivalent. They are still `Active`, so they are a duplication risk, not a free slot.

**What does not exist** (so a request needing it is net-new, not "extend"):
- MillTrack currently has no Apex triggers, asynchronous Apex
  (Queueable/Batch/Schedulable/@future), scheduled Flows, or scheduled paths. It does
  have live record-triggered Flows (listed above). Time-based follow-up is new capability.
- No Platform Events, no outbound integrations, no supplier API (README roadmap only).

## 4. Lifecycle and the `Dispatch_Status__c` state machine

`WorkOrder.Dispatch_Status__c` (restricted picklist) is the canonical MillTrack workflow
status. Values (API name → label): `Pending`, `Parts to Order`, `Waiting on Parts`,
`Ready_to_Assign` → "Ready to Assign", `Assigned`, `In Progress`, `Temporary Fix`,
`Completed`, `Cancelled`. Note `Ready_to_Assign` uses underscores — compare against the API name.

Transitions owned by current code:

```
Case created ──(Flow: Create_Work_Order_Proposal_From_Case)──► Pending
Pending ──(managerReviewWorkOrderWizard / ManagerReviewWorkOrderController.approveWorkOrder)──►
        Parts to Order (parts selected)  |  Ready_to_Assign (no parts)
Parts to Order ──(managerOrderPartsWizard / ManagerOrderPartsController.saveOrderingDecisions)──►
        Parts to Order (some still Requested) | Waiting on Parts | Ready_to_Assign (all cancelled)
Waiting on Parts ──(managerReceivePartsWizard / ManagerReceivePartsController.receiveParts)──►
        Ready_to_Assign when no part remains Requested/Ordered/Shipped (else stays)
Ready_to_Assign ──(managerDispatchTechnicianWizard / ManagerDispatchTechnicianController.dispatchTechnician)──► Assigned
Assigned ──(technicianStartWork / TechnicianStartWorkController.startWork)──► In Progress
In Progress ──(technicianRequestPartsWizard / TechnicianRequestPartsController)──► Waiting on Parts
In Progress ──(technicianTemporaryFixWizard / TechnicianTemporaryFixController.submitTemporaryFix)──► Temporary Fix
In Progress ──(technicianCompleteWorkOrderWizard / TechnicianCompleteWorkOrderController.completeWorkOrder)──► Completed
Temporary Fix ──(managerFollowUpWorkOrderWizard / ManagerFollowUpWorkOrderController.approveFollowUp)──►
        Parts to Order | Ready_to_Assign
```
`Cancelled` is a valid value and is read by queries/progress UI, but no current code path
sets it.

Transitions validate the *starting* `Dispatch_Status__c` and throw an
`AuraHandledException` if it does not match. Technician actions also require
`OwnerId == current user`. One exception: `ManagerReviewWorkOrderController.approveWorkOrder`
does not re-check for `Pending` (it relies on the Operations Center only offering Review for
`Pending` rows). New transitions should validate their starting state.

Notifications today (all `Messaging.CustomNotification` / Flow custom notification action):
temporary fix → `Temporary_Repair_FollowUp` to `Service_Manager__c`; technician parts
request → `Parts_Request` to `Service_Manager__c`; new Case → `New_Service_Case` to
`Service_Manager_Queue` members; customer case comment → `Customer_Case_Update`.
Dispatch, start, complete, review, order and receive send none.

When a Work Order goes back to the manager (Request Parts, Temporary Fix) the code also
sets `OwnerId` = `Service_Manager_Queue` and clears `Scheduled_Visit__c` and
`Suggested_Technician__c`. Dispatch sets `OwnerId` and `Suggested_Technician__c` to the
technician. Completion sets `OwnerId` back to `Service_Manager__c`.

## 5. Secondary status fields — kept aligned, never the driver

| Field | Meaning | Who sets it |
|---|---|---|
| `Case.Status` (standard) | Customer-facing request status. Values used in code: `New`, `In Progress`, `Scheduled`, `Waiting on Parts`, `Follow-Up Required`, `Closed`. Full picklist not in source. | Each WorkOrder transition updates the related Case alongside `Dispatch_Status__c`. |
| `Asset.Machine_Status__c` | Physical condition of the machine: `Active`, `Needs Service`, `Waiting on Parts`, `Ready to Assign`, `Temporary Fix Applied`, `Out of Service`, `Inactive`, `Retired`. | Case-create flows (`Needs Service`), temp fix (`Temporary Fix Applied`), completion (`Active`). `ManagerReceivePartsController` deliberately does **not** change it. |
| `Work_Order_Part__c.Order_Status__c` | Part lifecycle: `Requested`, `Ordered`, `Shipped`, `Received`, `Installed`, `Cancelled`. | Review/follow-up (Requested), technician request (Requested), order wizard (Ordered/Cancelled), receive wizard (Received). |
| `WorkOrder.Resolution_Status__c` | Outcome label. Exists; its validation rules are inactive; completion Apex does not set it. | Nobody in current code. |
| `WorkOrder.Status` (standard) | Platform status, not a MillTrack business-workflow field. Populate only when Salesforce requires a valid standard Status for record creation or platform compatibility; never use it instead of `Dispatch_Status__c` to represent MillTrack state. | Supplied code never reads or writes it. |

Business logic, queues, visibility and progress all key off `Dispatch_Status__c`. Keep
Case/Asset updates as side effects of the same transaction, the way existing controllers do.

## 6. AI boundaries

- **Customer AI** (`aiAuthoringBundles/Customer_AI`): troubleshooting, manual search,
  error codes, and service requests. Writes only through `CUST_02_Create_Authorized_Service_Case`
  (Case create with fingerprint de-dup) and `CUST_05_Add_Authorized_Case_Comment` (public
  CaseComment on an open, owned Case). Identity comes from `VerifiedCustomerId`, never from typed IDs.
- **Technician AI** (`MillTrack_Technician_Agent`): read-only grounding via
  `Build_Technician_Dispatch_Context`, `GetRelevantHistoricalRepairs`, knowledge search.
  Embedded on the technician Work Order page through `technicianAIChat` → `TechnicianAIChatController`.
- **Manager AI**: Prompt Builder templates `Generate_Work_Order_Proposal_Fields`,
  `Generate_Dispatch_Briefing`, `Prepare_Customer_Service_Case_Payload`, parsed by
  `ParseWorkOrderProposalFields`; history from `GetRelevantHistoricalRepairsForCase`.
Workflow changes rarely need to touch AI. Do not route state transitions through an agent.

## 7. Security model (summary — full audits belong to `salesforce-security-permission-auditor`)

- Permission sets: `MillTrack_Service_Manager`, `MillTrack_Technician`,
  `MillTrack_Customer_Portal_Access` (Customer Community Login), `MillTrack_Admin`, plus
  agent permission sets.
- Roles: `MillTrack_Administrator` > `MillTrack_Service_Manager` > `MillTrack_Technician`.
  Dispatch picks technicians by `UserRole.DeveloperName = 'MillTrack_Technician'`.
- Public group `MillTrack_Service_Managers` receives Edit via owner-based sharing rules on
  Case, WorkOrder, Maintenance_Log__c owned by manager/technician roles.
- Queue `Service_Manager_Queue` (Case, WorkOrder) — the manager-side owner.
- Customers: sharing set `MillTrack_Customer_Account_Access` gives **Read** on Account,
  Asset, Case, WorkOrder (by Account) and Maintenance_Log__c (by `Asset__c.Account`).
  Portal permission set: WorkOrder R, Case CR. Customers cannot edit Work Orders.
- Technicians: WorkOrder RU but FLS-editable only `Labor_Hours__c`, `Technician_Notes__c`;
  `Dispatch_Status__c` is read-only for them. Technician controllers therefore validate in
  code (status + `OwnerId == current user`) and then write with `AccessLevel.SYSTEM_MODE`.
  Custom Permission `MillTrack_Technician_Restricted_Editing` drives "protect fields"
  validation rules on Asset, Maintenance_Log__c, Work_Order_Part__c. The `MillTrack_Technician`
  permission set enables it, but its CustomPermission definition was not supplied.
- Managers: controllers query `WITH USER_MODE` and write `AccessLevel.USER_MODE`; the
  manager permission set has FLS edit on the workflow fields. Maintenance_Log__c is Read-only for managers.

## 8. Known drift and overlaps (do not build on these blindly)

- `Temporary_Fix_From_Work_Order` (legacy flow) writes `Dispatch_Status__c = 'Temporary FIx'`
  (capital I), and nothing in source launches it. `TechnicianTemporaryFixController` is the live path.
- `Work_Order_Created_or_Pending_Dispatch` filters on `Dispatch_Status__c = 'Manager Reviewed'`,
  which is not a picklist value — this flow cannot fire as written.
- `Work_Order_Assignment_Sync` (after-update: technician changed while `Ready_to_Assign`
  or `Assigned` → set `Assigned` + owner) overlaps `ManagerDispatchTechnicianController.dispatchTechnician`,
  which already sets both. Don't add a third assignment automation.
- `Dispatch_Technician` flow allows `Temporary Fix` → `Assigned` directly; the LWC/Apex
  dispatch path requires `Ready_to_Assign` (follow-up review first).
- `Receive_Parts_From_Work_Order` flow sets Asset `Ready to Assign`; the Apex receive path
  intentionally leaves Asset unchanged.
- Both `Create_Work_Order_Proposal_From_Case` and `SYS_01_Set_Asset_Service_Status` set Asset
  `Needs Service` on Case create; only SYS_01 guards Retired/Out of Service/Temporary Fix Applied.
- `Work_Order_Part__c` list view `Parts_To_Order` filters `Order_Status__c = 'Pending'` (not a value).
- README lists `customerWorkOrderProgress` and `technicianWorkOrderDetails`; neither exists
  in source. Progress lives in `customerWorkOrderDetails`.
- Page layouts and Experience Cloud page metadata were not supplied, so which legacy flow
  quick actions are still on layouts, and where customer LWCs sit in the site, are unverified.
