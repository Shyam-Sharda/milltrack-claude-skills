# MillTrack Data Model (workflow-relevant only)

Only fields that exist in the supplied metadata or are provably used by source are listed.
"(no field metadata supplied)" means code or a permission set references the field, so the
source expects it, but its definition was not in the archive. That doesn't prove it is
deployed or show its type/values; confirm before relying on it.

## Contents
1. Object relationships
2. WorkOrder
3. Case
4. Asset
5. Maintenance_Log__c
6. Work_Order_Part__c
7. Part_Catalog__c, Tool_Catalog__c, Service_Manual__c, Manual_Visual__c
8. Validation rules that affect workflow
9. Concepts with NO field today

---

## 1. Relationships

```
Account ─< Contact
Account ─< Asset ─< Case ─< WorkOrder        (WorkOrder.CaseId, WorkOrder.AssetId, AccountId, ContactId)
WorkOrder ─< Maintenance_Log__c              (Work_Order__c, required, Restrict delete)
Asset     ─< Maintenance_Log__c              (Asset__c)
WorkOrder ─< Work_Order_Part__c              (Work_Order__c, required, Restrict)
Asset     ─< Work_Order_Part__c              (Machine__c)
Part_Catalog__c ─< Work_Order_Part__c        (Part_Catalog__c, required, Restrict)
Service_Manual__c ─< Manual_Visual__c
User ← WorkOrder.Suggested_Technician__c ("Assigned Technician"), WorkOrder.Service_Manager__c,
       Maintenance_Log__c.Performed_By__c, Work_Order_Part__c.Ordered_By__c
```
One active Work Order per Case is the working assumption (`ManagerOperationsCenterController`
picks the newest non-Completed/Cancelled one if duplicates exist).

## 2. WorkOrder

| Field | Type | Notes |
|---|---|---|
| `Dispatch_Status__c` | Picklist (restricted) | **Canonical workflow status.** `Pending`, `Parts to Order`, `Waiting on Parts`, `Ready_to_Assign` (label "Ready to Assign"), `Assigned`, `In Progress`, `Temporary Fix`, `Completed`, `Cancelled` |
| `Suggested_Technician__c` | Lookup(User) | Label "Assigned Technician". Set on dispatch, cleared when returned to manager |
| `Service_Manager__c` | Lookup(User) | Set to the approving manager by review/follow-up approval. Notification recipient. Can be null (code handles it with a warning) |
| `Scheduled_Visit__c` | DateTime | Set on dispatch; cleared on parts request / temp fix; preserved on completion |
| `Work_Started_Date_Time__c` | DateTime | Set by Start Work |
| `Work_Ended_Date_Time__c` | DateTime | Set by Complete Work Order only |
| `Labor_Hours__c` | Number | Cumulative; each visit adds to it |
| `Technician_Notes__c` | LongTextArea | Append-only pattern (existing + blank line + new note) |
| `Problem_Summary__c`, `Recommended_Solution__c` (label "AI Recommended Solution"), `Recommended_Fix_Summary__c` (TextArea, ≤255 enforced in Apex), `Recommended_Tools__c` (newline list validated against Tool_Catalog__c names), `Recommended_Parts__c`, `Previous_Failure__c` (Checkbox) | | AI proposal / manager plan fields |
| `AI_Dispatch_Summary__c`, `Dispatch_Response__c` | LongTextArea | AI briefing output |
| `Resolution_Status__c` | Picklist (restricted) | `Unresolved`, `Resolved`, `Temporary Fix`, `Escalated`, `Unable to Reproduce`, `Customer Declined`, `Machine Replaced`. Not set by current code |
| Standard used: `OwnerId`, `CaseId`, `AssetId`, `AccountId`, `ContactId`, `Subject`, `Description`, `Priority`, `WorkOrderNumber`, `CreatedDate`, `LastModifiedDate` | | Standard `Status` is not used by MillTrack code and is not a workflow field; populate only when Salesforce requires it (see architecture §5) |

## 3. Case

| Field | Type | Notes |
|---|---|---|
| `Status` (standard) | Picklist | Values used in code: `New`, `In Progress`, `Scheduled`, `Waiting on Parts`, `Follow-Up Required`, `Closed`. Full value set not supplied |
| `Origin` | standard | `Web` (portal form), `Customer AI` |
| `Priority` | standard | `Low`, `Medium`, `High`, `Critical` |
| `Error_Code__c` | Text | |
| `Technician_Requested__c` | Checkbox | Customer asked for an on-site technician |
| `Work_Order_Proposal_Created__c` | Checkbox | Guard for the Case-create Work Order flow |
| `Service_Request_Fingerprint__c` | Text | SHA-256 de-dup key for open customer requests |
| `AI_Troubleshooting_Summary__c`, `Recommended_Parts__c`, `Recommended_Solution__c` | LongTextArea | |
| `SLA_Due_Date__c` | (no field metadata supplied) | Read by `CustomerCaseDetailsController`, `managerCaseSummary` |

## 4. Asset (the "machine")

| Field | Type | Notes |
|---|---|---|
| `Machine_Status__c` | Picklist (restricted) | `Active`, `Needs Service`, `Waiting on Parts`, `Ready to Assign`, `Temporary Fix Applied`, `Out of Service`, `Inactive`, `Retired` |
| `Manufacturer__c` | Picklist (restricted, required) | `Roland`, `Straumann`, `Ceramill`, `Amann Girrbach`, `Other` |
| `Model__c` (required), `Serial_Number__c` | Text | Serial number is how `Get_Machine_History` finds the machine |
| `Install_Date__c`, `Last_Service_Date__c`, `Warranty_Start_Date__c`, `Warranty_End_Date__c` | Date | `Last_Service_Date__c` set on temp fix and completion |
| `Customer_Maintenance_List__c` | (no field metadata supplied) | Customer-editable via `CustomerAssetDetailsController.saveMaintenanceList` |
| `Portal_Image_Path__c`, `Portal_Manual_Path__c` | (no field metadata supplied) | Customer portal media |

## 5. Maintenance_Log__c (visit / service history)

One record per technician visit outcome. This is where visit evidence lives.

| Field | Type | Notes |
|---|---|---|
| `Work_Order__c` (required), `Asset__c` | Lookup | |
| `Maintenance_Date__c` | Date (required) | Visit date (`Date.today()`) — Date only, no time |
| `Maintenance_Type__c` | Picklist (restricted, required) | `Installation_And_Calibration`, `Inspection`, `Preventive Maintenance`, `Repair`, `Temporary Repair`, `Emergency Service` |
| `Follow_Up_Required__c` | Checkbox | `true` for temporary fix and parts-request visits |
| `Issue_Resolved__c` | Checkbox | `true` only on completion (`Require_Root_Cause` validation then needs `Root_Cause__c`) |
| `Summary__c`, `Technician_Notes__c`, `Root_Cause__c`, `Parts_Used__c` | LongTextArea | |
| `Labor_Hours__c` | Number | This visit's hours |
| `Performed_By__c` | Lookup(User) | The technician |
| `Problem_Summary__c`, `Recommended_Fix_Snapshot__c`, `Tools_To_Bring_Snapshot__c` | (no field metadata supplied) | Snapshots of the WO plan at visit time; written by temp-fix Apex, read by follow-up Apex. Permission sets spell the last one `Tools_to_Bring_Snapshot__c` (API names are case-insensitive) |

The temporary-fix moment is recorded as: Maintenance_Log__c with
`Maintenance_Type__c = 'Temporary Repair'`, `Follow_Up_Required__c = true`,
`Issue_Resolved__c = false` (its `CreatedDate` provides an existing precise timestamp
associated with the temporary-fix transaction, not guaranteed identical to the exact
Work Order status-transition time), plus the
WorkOrder moving to `Temporary Fix`. `ManagerFollowUpWorkOrderController` loads the newest
log with `Follow_Up_Required__c = true` for the Work Order.

## 6. Work_Order_Part__c

| Field | Type | Notes |
|---|---|---|
| `Work_Order__c` (required), `Machine__c` (Asset), `Part_Catalog__c` (required) | Lookup | |
| `Order_Status__c` | Picklist (restricted) | `Requested`, `Ordered`, `Shipped`, `Received`, `Installed`, `Cancelled`. Outstanding = Requested/Ordered/Shipped; receivable = Ordered/Shipped |
| `Quantity__c` | Number | The quantity used by current code. `Quantity_Needed__c`, `Quantity_Used__c` also exist |
| `Technician_Requested__c` | Checkbox | Distinguishes technician requests from manager-required parts |
| `Supplier_Name__c`, `Supplier_Phone__c`, `Order_URL__c`, `Ordered_By__c`, `Notes__c` | | Captured when ordering |

## 7. Catalog / documentation objects

- `Part_Catalog__c`: `Part_Number__c` (required), `Manufacturer__c`, `Compatible_Model__c`,
  `Active__c` (only active parts may be selected), `Stock_Status__c`, `Parts_Category__c`,
  `Preferred_Supplier__c`, `Supplier_Name__c`, `Supplier_Phone__c`, `Order_URL__c`,
  `Cost__c`, `Keyword_Aliases__c`, `Notes__c`.
- `Tool_Catalog__c`: no custom field metadata supplied; code uses only `Id`, `Name`.
- `Service_Manual__c`: `Manufacturer__c`, `Model__c`, `Document_Type__c`, `Version__c`, `Notes__c`.
- `Manual_Visual__c`: manual figures (`Service_Manual__c`, `Audience__c` Customer/Technician/Both, `Category__c`, `Image_URL__c`, `Page_Number__c`, …).

## 8. Validation rules that constrain workflow writes

| Object | Rule | Active | Effect |
|---|---|---|---|
| WorkOrder | `Require_completion_notes_for_Completed` | yes | `Completed` needs `Technician_Notes__c` |
| WorkOrder | `Require_notes_before_Temporary_Fix` | yes | `Temporary Fix` needs `Technician_Notes__c` |
| WorkOrder | `Require_Asset_On_Completion` | yes | `Completed` needs `AssetId` |
| WorkOrder | `Require_Resolution_Status_on_Completed` | no | |
| WorkOrder | `Resolution_Status_only_after_completion` | no | |
| Maintenance_Log__c | `Require_Root_Cause` | yes | `Issue_Resolved__c` needs `Root_Cause__c` |
| Asset / Maintenance_Log__c / Work_Order_Part__c | `Technician_Protect_*` | yes | Users with Custom Permission `MillTrack_Technician_Restricted_Editing` cannot change key fields after create |

## 9. Concepts that have NO field today

Do not assume these exist; raise **Cannot Verify** or propose the field explicitly.
- A dedicated "temporary fix recorded at" timestamp on WorkOrder (use the Temporary Repair
  Maintenance_Log__c `CreatedDate`/`Maintenance_Date__c`, or the flow trigger time).
- Any customer confirmation / sign-off / satisfaction field on WorkOrder, Case or Maintenance_Log__c.
- A follow-up due date, reminder count, or escalation flag.
- A "parts received at" date on Work_Order_Part__c (only `LastModifiedDate`).
- A `Cancelled` reason or who-cancelled field.
- CustomNotificationType metadata: the source expects types with DeveloperName
  `Temporary_Repair_FollowUp`, `Parts_Request`, `New_Service_Case`, `Customer_Case_Update`,
  but their metadata was not supplied. Confirm they are deployed before relying on them.
