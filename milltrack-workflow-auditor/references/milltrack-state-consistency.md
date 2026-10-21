# MillTrack Cross-Record State Consistency

A workflow isn't correct just because the Work Order reaches the right
`Dispatch_Status__c`. This file describes what the **current owners** leave behind on
the Work Order, Case, Asset, `Work_Order_Part__c` and `Maintenance_Log__c` at each
stage. When an audited change produces a combination these owners never produce,
treat it as a finding.

## Contents
1. Allowed transitions
2. Expected state by stage
3. Maintenance Log (service history) requirements
4. Parts lifecycle rules
5. Validation rules that can block a workflow write
6. Combinations that are always wrong

---

## 1. Allowed transitions

| From | To | Owner | Condition |
|---|---|---|---|
| (Case created) | `Pending` | `Create_Work_Order_Proposal_From_Case` | Case not Closed; proposal flag false |
| `Pending` | `Parts to Order` / `Ready_to_Assign` | `ManagerReviewWorkOrderController.approveWorkOrder` | parts selected / none |
| `Parts to Order` | `Parts to Order` / `Waiting on Parts` / `Ready_to_Assign` | `ManagerOrderPartsController.saveOrderingDecisions` | some Requested remain / at least one Ordered and none Requested / all Cancelled |
| `Waiting on Parts` | `Ready_to_Assign` | `ManagerReceivePartsController.receiveParts` | no WOP Requested/Ordered/Shipped |
| `Ready_to_Assign` | `Assigned` | `ManagerDispatchTechnicianController.dispatchTechnician` | no outstanding WOP; valid technician; future visit; no ±30-min conflict |
| `Assigned` | `In Progress` | `TechnicianStartWorkController.startWork` | |
| `In Progress` | `Waiting on Parts` | `TechnicianRequestPartsController.submitPartsRequestV2` | owner = current technician |
| `In Progress` | `Temporary Fix` | `TechnicianTemporaryFixController.submitTemporaryFix` | owner = current technician |
| `In Progress` | `Completed` | `TechnicianCompleteWorkOrderController.completeWorkOrder` | owner = current technician |
| `Temporary Fix` | `Parts to Order` / `Ready_to_Assign` | `ManagerFollowUpWorkOrderController.approveFollowUp` | manager follow-up review |

**Not allowed** (examples; anything off the table above needs an explicit design
reason):
- `Temporary Fix` → `Assigned` or `In Progress`. This skips manager follow-up. Only
  the legacy `Dispatch_Technician` flow allows it.
- `Waiting on Parts` or `Parts to Order` → `Assigned` while any WOP is
  Requested/Ordered/Shipped
- `Pending` → `Assigned`, `In Progress` or `Completed`. This skips review.
- `Assigned` → `Completed`, or any → `Completed` without an `In Progress` visit by the
  owning technician
- `Completed` → anything. There's no reopen path in the source.
- Any transition driven by the customer, or by a technician outside their own
  `In Progress` (or `Assigned` for start) Work Order
- Any transition into `Cancelled`. The source defines no workflow for it, so ask for
  the design. Don't judge the side effects.

Both picklist values existing doesn't make a transition valid.

## 2. Expected state by stage

"Unchanged" means the current owner deliberately leaves the field as it was. A change
that *does* write it needs a reason.

### Pending
- **WO:** owner `Service_Manager_Queue`; AI proposal fields; linked to Case, Asset,
  Account and Contact. `Service_Manager__c`, `Suggested_Technician__c` and
  `Scheduled_Visit__c` are empty.
- **Case:** `New` (both customer creation paths set `New`); owner
  `Service_Manager_Queue`; `Work_Order_Proposal_Created__c = true`.
- **Asset:** `Needs Service` (set by Case-create automation).
- **Parts:** none. **Maintenance Log:** none.

### Parts to Order
- **WO:** `Service_Manager__c` = approving manager; plan fields set; still owned by
  the queue; no technician or visit.
- **Case:** `In Progress`.
- **Asset:** unchanged. It's `Needs Service` after review, or `Temporary Fix Applied`
  after a follow-up.
- **Parts:** at least one WOP `Requested` (manager-required, so
  `Technician_Requested__c` false).
- **Maintenance Log:** nothing new.

### Waiting on Parts
There are two valid ways in, and the state differs between them:
- **Manager ordering (from `Parts to Order`):** no WOP `Requested`; at least one
  `Ordered`; Case `Waiting on Parts`; Asset unchanged; queue owner; no technician.
- **Technician request (from `In Progress`):** new WOP `Requested` with
  `Technician_Requested__c = true`; a visit Maintenance Log (`Temporary Repair` or
  `Inspection`, follow-up true, unresolved); owner `Service_Manager_Queue`; technician
  and visit cleared; Case `Waiting on Parts`; Asset unchanged.
- Either way: `Requested` parts may exist in this stage **only** if they're
  technician-requested and waiting to be ordered.
- Receiving some parts leaves the stage unchanged, with Case `Waiting on Parts`.

### Ready_to_Assign
- **Parts:** none Requested, Ordered or Shipped. Received or Cancelled are fine.
- **Case:** `In Progress`.
- **WO:** queue owner; `Service_Manager__c` set; no technician or visit.
- **Asset:** unchanged (`Needs Service`, or `Temporary Fix Applied` after a follow-up).
  The current path never writes Asset `Ready to Assign`; only the legacy receive flow
  does.

### Assigned
- **WO:** `Suggested_Technician__c` = technician; `OwnerId` = the same technician;
  `Scheduled_Visit__c` set (it was ≥ now at dispatch and doesn't clash within ±30 min
  with that technician's Assigned/In Progress visits).
- **Case:** `Scheduled`.
- **Parts:** no outstanding parts at dispatch time.
- **Asset:** unchanged. The current dispatch doesn't touch it; the legacy
  `Dispatch_Technician` resets it to `Needs Service`.

### In Progress
- **WO:** `Work_Started_Date_Time__c` set; owner and technician still the dispatched
  technician.
- **Case:** `In Progress`. **Asset:** unchanged.

### Temporary Fix
- **WO:** `Temporary Fix`; owner `Service_Manager_Queue`; `Suggested_Technician__c` and
  `Scheduled_Visit__c` cleared; visit labor added to `Labor_Hours__c`; temporary-fix
  note appended to `Technician_Notes__c`.
- **Case:** `Follow-Up Required`.
- **Asset:** `Temporary Fix Applied`; `Last_Service_Date__c` = visit date.
- **Maintenance Log:** `Temporary Repair`, `Issue_Resolved__c = false`,
  `Follow_Up_Required__c = true`, visit labor, `Performed_By__c` = technician, linked to
  the WO and Asset, with plan snapshots.
- **Notification:** `Temporary_Repair_FollowUp` to `Service_Manager__c`.
- **Next step:** manager follow-up on the **same** Work Order. No second active Work
  Order.

### Completed
- **WO:** `Completed`; owner → `Service_Manager__c`; `Suggested_Technician__c`
  cleared; `Scheduled_Visit__c` kept as history; `Work_Ended_Date_Time__c` set; labor
  accumulated; completion notes appended (`Technician_Notes__c` not blank); `AssetId`
  set.
- **Case:** `Closed`.
- **Asset:** `Active`; `Last_Service_Date__c` = visit date.
- **Maintenance Log:** `Repair`, `Issue_Resolved__c = true`,
  `Follow_Up_Required__c = false`, non-blank `Root_Cause__c`, `Summary__c`, visit
  labor, `Performed_By__c` = technician.
- **Parts:** the current owner doesn't change WOP statuses at completion, so don't
  demand `Installed`.

### Cancelled
No current code sets it, and the source defines no Case, Asset or parts behavior for
it. Raise Cannot Verify and ask for the intended design. Don't invent expectations.

## 3. Maintenance Log (service history) requirements

| Visit outcome | Required by the current owner | Type | Resolved | Follow-up | Root Cause |
|---|---|---|---|---|---|
| Completion | yes, one per completion | `Repair` | true | false | required (validation `Require_Root_Cause`) |
| Temporary Fix | yes, one per temporary fix | `Temporary Repair` | false | true | not required |
| Technician parts request | yes, one per request visit | `Temporary Repair` if a temporary repair was applied, else `Inspection` | false | true | not required |
| Start work, dispatch, review, order, receive, follow-up | **none** | | | | |

Every visit log carries `Work_Order__c`, `Asset__c`, `Maintenance_Date__c` (today),
visit-specific `Labor_Hours__c` and `Performed_By__c`. Flag a missing log only for the
three outcomes above. "A log would be useful here" isn't a finding.
`ManagerFollowUpWorkOrderController` loads the newest `Follow_Up_Required__c = true`
log. A Temporary Fix path that skips the log breaks follow-up: it throws "No follow-up
Maintenance Log could be found".

## 4. Parts lifecycle rules

`Order_Status__c` (restricted): `Requested`, `Ordered`, `Shipped`, `Received`,
`Installed`, `Cancelled`.
- **Outstanding** = Requested, Ordered, Shipped. **Receivable** = Ordered, Shipped.
- Current code writes Requested (review, follow-up, technician request), Ordered and
  Cancelled (ordering), and Received (receiving). Nothing current writes `Shipped` or
  `Installed`.
- A Work Order must not move to `Ready_to_Assign` or `Assigned` while outstanding
  parts remain. The source has no exception.
- The same catalog part can't have two active (non-Cancelled) requests on one Work
  Order (review and follow-up enforce this).
- Technician requests must use an active `Part_Catalog__c` that matches the Asset's
  `Manufacturer__c`.
- Don't invent part records or catalog entries during an audit.

## 5. Validation rules that can block a workflow write

| Rule | Active | Blocks |
|---|---|---|
| WorkOrder `Require_completion_notes_for_Completed` | yes | `Completed` with blank `Technician_Notes__c` |
| WorkOrder `Require_notes_before_Temporary_Fix` | yes | `Temporary Fix` with blank `Technician_Notes__c` (any existing note satisfies it, because notes are append-only) |
| WorkOrder `Require_Asset_On_Completion` | yes | `Completed` with no `AssetId` |
| Maintenance_Log__c `Require_Root_Cause` | yes | `Issue_Resolved__c = true` with blank `Root_Cause__c` |
| `Technician_Protect_Asset_Fields` | yes | users with `MillTrack_Technician_Restricted_Editing` changing Asset Name, Manufacturer or Model after create |
| `Technician_Protect_MaintenanceLog_Fields` | yes | the same users changing a log's Work Order, Date, Type or Labor after create |
| `Technician_Protect_Work_Order_Part_Field` | yes | the same users changing a WOP's Part Catalog or Work Order after create |
| WorkOrder `Require_Resolution_Status_on_Completed`, `Resolution_Status_only_after_completion` | **no** | nothing. Don't cite them |

Validation rules run even in SYSTEM_MODE DML. A technician-context automation that
updates protected fields on existing records will fail.

## 6. Combinations that are always wrong

- WO `Completed` with Case not `Closed`, Asset not `Active`, or no resolved `Repair`
  log with Root Cause
- WO `Temporary Fix` without Asset `Temporary Fix Applied`, Case `Follow-Up Required`,
  an unresolved follow-up log, or a queue owner. Or with a technician or visit still
  set.
- WO `Assigned` with an outstanding WOP, no `Scheduled_Visit__c`, `OwnerId` ≠
  `Suggested_Technician__c`, or Case not `Scheduled`
- WO `Ready_to_Assign` or `Assigned` with any WOP Requested, Ordered or Shipped
- WO `Waiting on Parts` with no outstanding WOP at all (nothing would ever release it
  to `Ready_to_Assign`)
- Two active (non-Completed/Cancelled) Work Orders on one Case
- A technician still owning a Work Order that has returned to the manager
  (`Waiting on Parts`, `Temporary Fix`, `Parts to Order`, `Ready_to_Assign`)
- Any `Dispatch_Status__c` literal not among the nine API values (for example
  `Ready to Assign` or `Temporary FIx`)
