# MillTrack Case Data Model

Only the fields that matter when generating service records. Derived from
`force-app/main/default/objects/**` plus the Apex and Flows that write them.
If a field is not listed here, do not use it in a fixture unless the user
supplies it.

**Picklist rule:** every custom picklist below is `restricted=true`. Use the
API value exactly as written (left column of each value list). Labels that
differ from the API value are shown in brackets.

---

## Case (standard object)

Standard fields used by MillTrack automation. The Case object metadata and
its standard value sets were **not** in the supplied source, so the values
below are the ones MillTrack code actually writes, not a full picklist dump.

| Field | Type | Notes |
|---|---|---|
| `Subject` | Text(255) | Short symptom statement. Customer AI aims for 4–12 words. |
| `Description` | Long text | Customer-reported facts only (see `milltrack-scenario-rules.md`, section 2). |
| `Status` | Picklist | Values written by code: `New`, `In Progress`, `Scheduled`, `Waiting on Parts`, `Follow-Up Required`, `Closed`. |
| `Priority` | Picklist | Code normalizes to `Critical`, `High`, `Medium`, `Low` (default `Medium`). |
| `Origin` | Picklist | `Web` (portal Report an Issue) or `Customer AI` (Agentforce). |
| `AccountId` | Lookup(Account) | Customer account. |
| `ContactId` | Lookup(Contact) | Portal contact on that account. |
| `AssetId` | Lookup(Asset) | Machine. Must belong to the same Account (both create flows check this). |
| `OwnerId` | Lookup(User/Queue) | Set to `Service_Manager_Queue` on creation. |

Custom fields:

| Field | Type | Written by | Generate? |
|---|---|---|---|
| `Error_Code__c` | Text(50) | Customer create paths | Only when the user supplies a code. Store it verbatim. |
| `Technician_Requested__c` | Checkbox (default false) | Customer create paths | `true` only if the customer explicitly asked for a technician/onsite visit. |
| `Work_Order_Proposal_Created__c` | Checkbox (default false) | `Create_Work_Order_Proposal_From_Case` sets it `true` after creating the Work Order | `true` on any Case that already has its Work Order. This is also the flow's entry guard (see `milltrack-seed-output-rules.md`, section 3). |
| `Service_Request_Fingerprint__c` | Text(64) | `CUST_02` Customer AI path only (SHA-256 of Asset Id + original message) | Leave blank. Never fabricate a hash. |
| `Recommended_Solution__c` | Long text | **No current code writes it** | Leave blank unless explicitly requested. |
| `Recommended_Parts__c` | Long text | **No current code writes it** | Leave blank. |
| `AI_Troubleshooting_Summary__c` | Long text | **No current code writes it** (shown on Admin Case page only) | Leave blank. |

---

## WorkOrder (standard object)

| Field | Type | Notes |
|---|---|---|
| `Dispatch_Status__c` | Picklist, restricted | **Canonical workflow status.** Values: `Pending`, `Parts to Order`, `Waiting on Parts`, `Ready_to_Assign` [Ready to Assign], `Assigned`, `In Progress`, `Temporary Fix`, `Completed`, `Cancelled`. |
| `Status` (standard) | Picklist | Not a MillTrack workflow field, and no MillTrack code writes it. Do not use it to represent or infer MillTrack lifecycle state. Only populate it when Salesforce requires a valid standard Status for record creation or platform compatibility; if the required value is not established by the supplied source or target org, use Cannot Verify rather than inventing one. |
| `CaseId`, `AccountId`, `ContactId`, `AssetId` | Lookups | Copied from the Case when automation creates the Work Order. |
| `Subject`, `Description` | Text | Copied from the Case on creation. |
| `Priority` | Picklist | Proposal AI sets it on creation; manager review sets `Low`/`Medium`/`High`/`Critical`. |
| `OwnerId` | User/Queue | Queue while managers own it; technician User when Assigned/In Progress; back to queue on parts request/Temporary Fix; `Service_Manager__c` user on Completed. |
| `Problem_Summary__c` | Long text | Manager-plan field. Normalized problem statement. Required at manager review. |
| `Recommended_Solution__c` | Long text | Manager-plan field. Required at manager review. |
| `Recommended_Fix_Summary__c` | TextArea, **≤255 chars** (enforced in Apex) | Manager-plan field. Required at follow-up approval. |
| `Recommended_Tools__c` | Long text | Manager-plan field (Tool Catalog names). |
| `Recommended_Parts__c` | Long text | Manager-plan free text. Not the same as Work_Order_Part__c records. |
| `Previous_Failure__c` | Checkbox | Manager/AI flag for a repeat failure. |
| `Service_Manager__c` | Lookup(User) | Set to the reviewing manager at review/follow-up approval. **Required before completion** (Apex check). |
| `Suggested_Technician__c` | Lookup(User) | Assigned technician (role `MillTrack_Technician`). Cleared on parts request, Temporary Fix and Completed. |
| `Scheduled_Visit__c` | DateTime | Set at dispatch. Cleared on parts request and Temporary Fix. Preserved on Completed. |
| `Work_Started_Date_Time__c` | DateTime | Set by Start Work. Never cleared. |
| `Work_Ended_Date_Time__c` | DateTime | Set on Completed only. |
| `Labor_Hours__c` | Number(5,2) | **Cumulative** across visits (each visit adds its hours). |
| `Technician_Notes__c` | Long text | Append-only visit history. **Required** for `Temporary Fix` and `Completed` (active validation rules). |
| `AI_Dispatch_Summary__c`, `Dispatch_Response__c` | Long text | AI dispatch briefing outputs. Leave blank in fixtures unless the user asks for one. |
| `Resolution_Status__c` | Picklist, restricted | Values: `Unresolved`, `Resolved`, `Temporary Fix`, `Escalated`, `Unable to Reproduce`, `Customer Declined`, `Machine Replaced`. **No current code writes it**, and both rules that would require it are inactive. Leave blank by default. |

---

## Asset (standard object)

| Field | Type | Notes |
|---|---|---|
| `Name` (standard) | Text | Required by the platform. |
| `AccountId` | Lookup | Customer account. Portal and agent flows only accept Assets on the customer's Account. |
| `Manufacturer__c` | Picklist, **required**, restricted | `Roland`, `Straumann`, `Ceramill`, `Amann Girrbach`, `Other`. |
| `Model__c` | Text(100), **required** | Model as supplied, e.g. `DWX-53DC`. |
| `Serial_Number__c` | Text(100) | Use an obviously synthetic value, or a placeholder. |
| `Machine_Status__c` | Picklist, restricted | `Active`, `Needs Service`, `Waiting on Parts`, `Ready to Assign`, `Temporary Fix Applied`, `Out of Service`, `Inactive`, `Retired`. |
| `Last_Service_Date__c` | Date | Set on Temporary Fix and Completed. |
| `Install_Date__c`, `Warranty_Start_Date__c`, `Warranty_End_Date__c` | Date | Optional context. |

Validation rule `Technician_Protect_Asset_Fields` (active): users with custom
permission `MillTrack_Technician_Restricted_Editing` cannot change `Name`,
`Manufacturer__c` or `Model__c` on an existing Asset. Seed code that edits
Assets should run as an admin or manager.

---

## Maintenance_Log__c (custom)

Name: AutoNumber `ML-{00000}`. Never set Name.

| Field | Type | Notes |
|---|---|---|
| `Work_Order__c` | Lookup(WorkOrder), **required**, delete=Restrict | Every log belongs to a Work Order. |
| `Asset__c` | Lookup(Asset) | Same machine as the Work Order. |
| `Maintenance_Date__c` | Date, **required** | Visit date. |
| `Maintenance_Type__c` | Picklist, **required**, restricted | `Installation_And_Calibration` [Installation And Calibration], `Inspection`, `Preventive Maintenance`, `Repair`, `Temporary Repair`, `Emergency Service`. |
| `Summary__c` | Long text | Visit or repair summary. |
| `Technician_Notes__c` | Long text | Structured note built by Apex (see `milltrack-case-lifecycle.md`, sections 7–9). |
| `Root_Cause__c` | Long text | **Required when `Issue_Resolved__c = true`** (active rule `Require_Root_Cause`). |
| `Parts_Used__c` | Long text | Optional free text at completion. |
| `Labor_Hours__c` | Number(5,2) | **This visit only** (not cumulative). Apex requires >0 for Temporary Fix and Completed. |
| `Issue_Resolved__c` | Checkbox | `true` only on completion logs. |
| `Follow_Up_Required__c` | Checkbox | `true` on Temporary Repair and parts-request logs. |
| `Performed_By__c` | Lookup(User) | The technician. |
| `OwnerId` | User | Technician (set by Apex). |

**Apex-evidenced snapshot fields:** `Problem_Summary__c`,
`Recommended_Fix_Snapshot__c` and `Tools_To_Bring_Snapshot__c`. Current Apex
references and writes these fields, which is evidence that they exist in the
source org. Their field metadata was not supplied, so their exact type,
length, defaults, validation and other metadata cannot be verified from this
archive.

What current Apex copies into them (Work Order value → log field):

| Controller | `Problem_Summary__c` | `Recommended_Fix_Snapshot__c` | `Tools_To_Bring_Snapshot__c` |
|---|---|---|---|
| `TechnicianTemporaryFixController` | `WorkOrder.Problem_Summary__c` | `WorkOrder.Recommended_Fix_Summary__c` | `WorkOrder.Recommended_Tools__c` |
| `TechnicianRequestPartsController` | `WorkOrder.Problem_Summary__c` | not written | not written |
| `TechnicianCompleteWorkOrderController` | `WorkOrder.Problem_Summary__c` | not written | not written |

Rules for fixtures:

- These fields **may** be included when the fixture reproduces what the
  current Apex writes.
- Include only the fields that controller writes, each holding the copied Work
  Order value (blank if the source Work Order field is blank).
- Do not state or imply a type, length, default or other metadata for them.
- Omitting them is also acceptable, since they are snapshots and nothing in
  the supplied source requires them.
- In either case, add the one-line metadata note from
  `milltrack-seed-output-rules.md`, section 8.

Validation rule `Technician_Protect_MaintenanceLog_Fields` (active): restricted
technicians cannot change `Work_Order__c`, `Maintenance_Date__c`,
`Maintenance_Type__c` or `Labor_Hours__c` after insert.

Sharing: Service Managers see Maintenance Logs through owner-based sharing
rules, from the `MillTrack_Technician` and `MillTrack_Service_Manager` roles to
the `MillTrack_Service_Managers` group. The follow-up wizard reads the log
`WITH USER_MODE`, so a seeded log owned by an admin with neither role may be
invisible to the manager. The wizard then fails with "No follow-up
Maintenance Log could be found". In seed code, set `OwnerId` to a technician
user.

Logs with `Issue_Resolved__c = true AND Follow_Up_Required__c = false` are
what historical-repair precedent queries return
(`GetRelevantHistoricalRepairs*`, `TechnicianCopilotController`). That makes
completed fixtures useful as AI grounding data.

---

## Work_Order_Part__c (custom)

Name: AutoNumber `WOP-{00000}`. Never set Name.

| Field | Type | Notes |
|---|---|---|
| `Work_Order__c` | Lookup(WorkOrder), **required**, delete=Restrict | |
| `Part_Catalog__c` | Lookup(Part_Catalog__c), **required**, delete=Restrict | Must reference a real or explicitly synthetic catalog record. |
| `Machine__c` | Lookup(Asset), delete=Restrict | The Work Order's Asset. |
| `Order_Status__c` | Picklist, restricted | `Requested`, `Ordered`, `Shipped`, `Received`, `Installed`, `Cancelled`. |
| `Quantity__c` | Number(18,0) | Quantity requested (>0). |
| `Quantity_Needed__c` | Number(18,0) | Not written by current code. Leave blank. |
| `Quantity_Used__c` | Number(8,2) | Only the legacy X13 flow writes it (0). Leave blank. |
| `Technician_Requested__c` | Checkbox | `true` when a technician requested the part during a visit. `false` for manager-selected parts. |
| `Notes__c` | Long text | Technician's request reason. |
| `Ordered_By__c` | Lookup(User) | Manager who ordered it. Cleared on Cancelled. |
| `Supplier_Name__c` | Text(50) | Snapshot at order time. |
| `Supplier_Phone__c` | Phone | Snapshot at order time. |
| `Order_URL__c` | URL | Snapshot at order time. |

**Outstanding** = `Requested`, `Ordered` or `Shipped`. Outstanding parts block
`Ready_to_Assign` and dispatch.

No current code sets `Shipped` or `Installed`. Received parts stay `Received`
after completion.

Validation rule `Technician_Protect_Work_Order_Part_Field` (active): restricted
technicians cannot change `Work_Order__c` or `Part_Catalog__c` after insert.

---

## Part_Catalog__c (custom, only when a fully synthetic catalog is requested)

| Field | Type | Notes |
|---|---|---|
| `Name` | Text | Part name. |
| `Part_Number__c` | Text(50), **required** | |
| `Manufacturer__c` | Picklist, **required**, restricted | Same values as Asset. **Must equal the Asset's manufacturer** for technician part requests (Apex check). |
| `Active__c` | Checkbox (default false) | Must be `true` for technician requests. |
| `Compatible_Model__c` | Text(100) | |
| `Parts_Category__c` | Picklist, restricted | `Spindle`, `Collet`, `Tool Changer`, `Filter`, `Limit Switch`, `USB Cable`, `X Ball Screws`, `Y Ball Screws`, `Z Ball Screws`, `Motors`, `Motherboard`, `Cutting Tools`, `Motors & Drives`, `Toolholders & Collets`, `Filters & Maintenance`, `Workholding & Fixtures`, `Coolant System`, `Calibration & Sensors`, `Electronics & Controls`, `Seals & Gaskets`. |
| `Stock_Status__c` | Picklist, restricted | `In Stock`, `Low Stock`, `Out of Stock`, `Special Order`. |
| `Preferred_Supplier__c` | Picklist, restricted | `Roland`, `Straumann`, `Amann Girrbach`, `Henry Schein`, `Other`. |
| `Supplier_Name__c` (Text 100), `Supplier_Phone__c`, `Order_URL__c`, `Cost__c` (Currency), `Keyword_Aliases__c`, `Notes__c` | | Optional. |

`Tool_Catalog__c` exists (Name only in supplied metadata). Manager review and
follow-up split `Recommended_Tools__c` on newlines, commas or semicolons, and
check each entry against `Tool_Catalog__c.Name`. No Tool Catalog records were
supplied, so synthetic tool names in staged fixtures are illustrative only.
Flag them once under **Cannot Verify** if the data will be loaded through the
wizards. Direct inserts bypass that check.

---

## Users, queue and roles (never fabricate their Ids)

| Concept | Source | Placeholder |
|---|---|---|
| Service Manager Queue | Queue DeveloperName `Service_Manager_Queue` (Case + WorkOrder) | `<SERVICE_MANAGER_QUEUE_ID>` |
| Service Manager | User on `WorkOrder.Service_Manager__c`. Role `MillTrack_Service_Manager` | `<SERVICE_MANAGER_USER_ID>` |
| Technician | User with role DeveloperName `MillTrack_Technician` (dispatch requires it, active) | `<TECHNICIAN_USER_ID>` |
| Customer | Portal Contact on the Account (`MillTrack Customer Portal`) | `<CONTACT_ID>`, `<ACCOUNT_ID>` |
