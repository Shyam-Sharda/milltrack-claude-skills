# MillTrack Case Lifecycle

How a service Case moves through Work Order states, and what each transition
writes to Case, WorkOrder, Asset, Maintenance_Log__c and Work_Order_Part__c.

**Current vs legacy:** the live UI runs through LWC wizards backed by Apex
controllers, launched from Quick Actions and the Manager Operations Center
(which sets `flowApiName = null`, so it launches no flows). Several older
screen flows are still Active in metadata, but their Quick Actions are
labelled `ZZZ` or `zzz`, or nothing in the UI launches them. Model fixtures on
the **current** path. Legacy behavior is listed at the end only so you can
recognize it.

## Contents
1. State table (quick reference)
2. Case creation and the live automation
3. Manager review
4. Parts ordering and receiving
5. Dispatch
6. Start work
7. Technician parts request
8. Temporary Fix and manager follow-up
9. Completion
10. Cancelled
11. Legacy automation (do not model)

---

## 1. State table

Values shown are what the current code leaves on each record **at rest** in
that stage. "—" = blank/none.

| Stage (`Dispatch_Status__c`) | Case.Status | Asset.Machine_Status__c | WO owner | Service_Manager__c | Suggested_Technician__c / Scheduled_Visit__c | Work_Order_Part__c | Maintenance_Log__c |
|---|---|---|---|---|---|---|---|
| `Pending` | `New` | `Needs Service` | Service Manager Queue | — | — / — | none | none |
| `Parts to Order` | `In Progress` | `Needs Service`* | Queue | manager | — / — | ≥1 `Requested` | none (first pass) |
| `Waiting on Parts` (manager ordered) | `Waiting on Parts` | `Needs Service`* | Queue | manager | — / — | ≥1 `Ordered`/`Shipped`, none `Requested` | none (first pass) |
| `Waiting on Parts` (technician requested) | `Waiting on Parts` | unchanged from visit (`Needs Service`*) | Queue | manager | — / — | ≥1 `Requested`, `Technician_Requested__c = true` | 1 visit log, `Follow_Up_Required__c = true` |
| `Ready_to_Assign` | `In Progress` | `Needs Service`* | Queue | manager | — / — | none outstanding (only `Received`/`Cancelled`, or none) | none, or prior visit logs |
| `Assigned` | `Scheduled` | `Needs Service`* | technician | manager | technician / future DateTime | none outstanding | none, or prior visit logs |
| `In Progress` | `In Progress` | `Needs Service`* | technician | manager | technician / set | none outstanding | none, or prior visit logs |
| `Temporary Fix` | `Follow-Up Required` | `Temporary Fix Applied` | Queue | manager | — / — | none outstanding | + 1 `Temporary Repair` log, unresolved |
| `Completed` | `Closed` | `Active` | `Service_Manager__c` user | manager | — / preserved | `Received`/`Cancelled` only | + 1 `Repair` log, resolved, Root Cause |

\* Current Apex never changes Asset status during review, ordering, receiving,
dispatch or start. The Asset keeps whatever it had: `Needs Service` from Case
creation, or `Temporary Fix Applied` on a follow-up cycle. The Asset values
`Waiting on Parts` and `Ready to Assign` are only written by legacy flows
(section 11). Do not set them in current-path fixtures.

**Technician note appending.** The parts-request, Temporary Fix and Complete
controllers each build a note block and append it to
`WorkOrder.Technician_Notes__c`. The code is `existing + '\n\n' + newBlock`
(one blank line between blocks). When the field is blank, it writes only the
new block. The Maintenance Log's own `Technician_Notes__c` holds just that
visit's block, with nothing appended. In staged data:

- A single-visit Work Order holds exactly one block.
- A multi-visit Work Order (for example, a Temporary Fix and then a Completed
  follow-up, or a parts-request visit and then a repair) holds the blocks in
  visit order, separated by `\n\n`.

The block formats are in sections 7–9.

---

## 2. Case creation and the live automation

**Customer entry points** (both create `Status = New`, `OwnerId = Service_Manager_Queue`):

| Path | Owner of logic | Origin | Notes |
|---|---|---|---|
| Portal "Report an Issue" | `customerReportAnIssue` LWC → `CustomerReportAnIssueController` → `Create_Customer_Service_Case_Core` flow | `Web` | Inputs: Asset (must be on the customer's Account), Subject (≤255), Description (required), Priority (Critical/High/Medium/Low, default Medium), Error Code (optional), Technician Requested. |
| MillTrack Customer AI (Agentforce) | `CUST_04_Prepare…` (Prompt Builder payload) → `CUST_02_Create_Authorized_Service_Case` | `Customer AI` | Also sets `Service_Request_Fingerprint__c` and blocks a duplicate open Case with the same fingerprint for the same Contact/Account/Asset. |

**Record-triggered automation on Case insert** (after-save, synchronous):

1. `Create_Work_Order_Proposal_From_Case`. Entry: `Status != Closed AND Work_Order_Proposal_Created__c = false`.
   - Calls `GetRelevantHistoricalRepairsForCase`, then Prompt Builder `Generate_Work_Order_Proposal_Fields`, then `ParseWorkOrderProposalFields`.
   - **Creates one WorkOrder:** `Dispatch_Status__c = Pending`, `OwnerId = Service_Manager_Queue`, `CaseId`, `AccountId`, `AssetId`, `ContactId`, `Subject`, `Description` copied from the Case, plus AI-drafted `Problem_Summary__c`, `Recommended_Solution__c`, `Recommended_Parts__c`, `Recommended_Tools__c`, `Previous_Failure__c`, `Priority`.
   - Updates the Asset to `Machine_Status__c = Needs Service`.
   - Sets `Case.Work_Order_Proposal_Created__c = true`.
   - Sends a "new service case" custom notification to queue members.
   - Has **no fault connectors**. If the prompt or parse step fails, the whole Case insert fails.
2. `SYS_01_Set_Asset_Service_Status`. Entry: `AssetId != null AND Status = New`. Sets the Asset to `Needs Service`.

So in an org with automation active, inserting a `New` Case produces the
Pending Work Order by itself. A generator must not insert a second one.

The Pending Work Order's AI fields are drafts produced by the live system.
Do not hand-author them in a live-automation fixture.

---

## 3. Manager review: `Pending` → `Parts to Order` or `Ready_to_Assign`

Owner: `managerReviewWorkOrderWizard` → `ManagerReviewWorkOrderController.approveWorkOrder`.

- Required inputs: `Problem_Summary__c`, `Recommended_Solution__c`, `Priority` ∈ {Low, Medium, High, Critical}. `Recommended_Fix_Summary__c` is optional here and must be ≤255 characters.
- Writes: `Problem_Summary__c`, `Recommended_Solution__c`, `Recommended_Fix_Summary__c`, `Recommended_Tools__c` (validated against Tool Catalog), `Recommended_Parts__c`, `Previous_Failure__c`, `Priority`, `Service_Manager__c = reviewing manager`.
- If the manager selected catalog parts: `Dispatch_Status__c = Parts to Order`, and one `Work_Order_Part__c` per part (`Order_Status__c = Requested`, `Technician_Requested__c = false`, `Machine__c = Asset`, `Quantity__c`). Otherwise: `Ready_to_Assign`.
- Case → `In Progress`. Asset unchanged.

## 4. Parts ordering and receiving

**Order parts** (`managerOrderPartsWizard` → `ManagerOrderPartsController`, from `Parts to Order`):

- For each `Requested` part, the manager either removes it (`Cancelled`, `Ordered_By__c` cleared) or confirms the order (`Ordered`, `Ordered_By__c = manager`, supplier name/phone/URL snapshot from the Part Catalog). A part can also be left `Requested`.
- Next state, computed over **all** parts on the Work Order:
  - any `Requested` left → stays `Parts to Order`, Case `In Progress`
  - else any non-Cancelled part → `Waiting on Parts`, Case `Waiting on Parts`
  - else (all cancelled) → `Ready_to_Assign`, Case `In Progress`
- Asset unchanged.

**Order a technician-requested part** (`ManagerOrderPartsController.orderTechnicianRequestedPart`): only while the Work Order is `Waiting on Parts` and the part is `Requested`. Requires supplier name, phone and order URL. Sets `Ordered`. The Work Order stays `Waiting on Parts`.

**Receive parts** (`managerReceivePartsWizard` → `ManagerReceivePartsController`, from `Waiting on Parts`):

- Receivable: `Ordered`, `Shipped`. Receiving sets `Received`.
- If no outstanding parts remain (`Requested`/`Ordered`/`Shipped`): Work Order → `Ready_to_Assign`, Case → `In Progress`. Otherwise the Work Order stays and the Case stays `Waiting on Parts`.
- Asset unchanged (the code comment says explicitly: "Receiving parts must not change it").

## 5. Dispatch: `Ready_to_Assign` → `Assigned`

Owner: `managerDispatchTechnicianWizard` → `ManagerDispatchTechnicianController`. A `Dispatch Technician` Flow quick action also exists (see legacy).

- Preconditions: status is `Ready_to_Assign`, **zero outstanding parts**, the technician is an active User with role `MillTrack_Technician`, `Scheduled_Visit__c` is now or in the future, and the technician has no overlapping visit (30-minute buffer).
- Writes: `Suggested_Technician__c = tech`, `OwnerId = tech`, `Scheduled_Visit__c`, `Dispatch_Status__c = Assigned`. Case → `Scheduled`. Asset unchanged.

**Assignment sync trigger** `Work_Order_Assignment_Sync` (after **update** only):
if `Suggested_Technician__c` is set and the status is `Ready_to_Assign` or
`Assigned`, it forces `Assigned` and `OwnerId = Suggested_Technician__c`.
So a `Ready_to_Assign` record must keep `Suggested_Technician__c` blank, or
the next update will flip it to Assigned.

## 6. Start work: `Assigned` → `In Progress`

Owner: `technicianStartWork` → `TechnicianStartWorkController`. Requires `Assigned`.
Writes `Dispatch_Status__c = In Progress`, `Work_Started_Date_Time__c = now`.
Case → `In Progress`. The owner stays the technician.

## 7. Technician parts request: `In Progress` → `Waiting on Parts`

Owner: `technicianRequestPartsWizard` → `TechnicianRequestPartsController.submitPartsRequestV2`.

- Preconditions: `In Progress`, the caller owns the Work Order, the Asset has a manufacturer, and each part is an active catalog item **with the same manufacturer as the Asset**, no duplicates, quantity >0, at most 25 parts. Reason, visit summary and labor (≥0) are required.
- Creates `Work_Order_Part__c` per part: `Requested`, `Technician_Requested__c = true`, `Notes__c = reason`, `Machine__c`, `Quantity__c`.
- Creates one `Maintenance_Log__c`: `Maintenance_Type__c = Temporary Repair` if a temporary repair was applied, else `Inspection`. Also `Summary__c = visit summary`, `Technician_Notes__c` = "PARTS REQUEST …" block, `Labor_Hours__c = visit labor`, `Issue_Resolved__c = false`, `Follow_Up_Required__c = true`, `Performed_By__c = tech`, and a `Problem_Summary__c` snapshot.
- Work Order: `Waiting on Parts`, `OwnerId = Service_Manager_Queue`, `Scheduled_Visit__c` and `Suggested_Technician__c` cleared, `Labor_Hours__c += visit labor`, a "PARTS REQUEST VISIT" note appended to `Technician_Notes__c`.
- Case → `Waiting on Parts`. **Asset unchanged.**
- Next: the manager orders the part (section 4), then receives it → `Ready_to_Assign` → re-dispatch.

## 8. Temporary Fix and manager follow-up

**Record Temporary Fix** (`technicianTemporaryFixWizard` via Quick Action `Temporary_Fix_Follow_Up` → `TechnicianTemporaryFixController`):

- Preconditions: `In Progress`, the caller owns the Work Order, and an Asset is linked. Visit summary required; labor >0.
- Creates `Maintenance_Log__c`: `Maintenance_Type__c = Temporary Repair`, `Maintenance_Date__c = today`, `Summary__c = visit summary`, `Technician_Notes__c = "TEMPORARY REPAIR / FOLLOW-UP\n\nVisit Summary:\n…\n\nLabor Hours:\n…"`, `Labor_Hours__c = visit labor`, `Issue_Resolved__c = false`, `Follow_Up_Required__c = true`, `Performed_By__c` and `OwnerId` = tech. It also writes the `Problem_Summary__c`, `Recommended_Fix_Snapshot__c` and `Tools_To_Bring_Snapshot__c` snapshots, copied from the Work Order's `Problem_Summary__c`, `Recommended_Fix_Summary__c` and `Recommended_Tools__c`. The Apex evidences these fields, but their metadata was not supplied (see `milltrack-case-data-model.md`). **Root_Cause__c is not set** by the current controller.
- Asset → `Temporary Fix Applied`, `Last_Service_Date__c = today`.
- Work Order → `Temporary Fix`, `OwnerId = Service_Manager_Queue`, `Scheduled_Visit__c` and `Suggested_Technician__c` cleared, `Labor_Hours__c += visit labor`, note appended to `Technician_Notes__c` (required by the active rule `Require_notes_before_Temporary_Fix`). `Work_Started_Date_Time__c` is kept.
- Case → `Follow-Up Required`. A "Temporary Repair Follow-Up" notification goes to `Service_Manager__c`.

**Manager follow-up** (`managerFollowUpWorkOrderWizard` → `ManagerFollowUpWorkOrderController`, from `Temporary Fix`):

- Required: `Recommended_Fix_Summary__c` (≤255). Optional tools and catalog parts.
- Writes `Recommended_Fix_Summary__c`, `Recommended_Tools__c`, `Service_Manager__c`. Then `Parts to Order` (with new `Requested` parts) or `Ready_to_Assign`.
- Case → `In Progress`. Asset stays `Temporary Fix Applied` until completion.
- The **same Work Order** carries the follow-up visit. No new Work Order is created.

## 9. Completion: `In Progress` → `Completed`

Owner: `technicianCompleteWorkOrderWizard` → `TechnicianCompleteWorkOrderController`.

- Preconditions: `In Progress`, the caller owns the Work Order, an Asset is linked, and `Service_Manager__c` is set. Required: repair summary, **root cause**, labor >0. Parts used is optional.
- Creates `Maintenance_Log__c`: `Maintenance_Type__c = Repair`, `Maintenance_Date__c = today`, `Summary__c`, `Root_Cause__c`, `Parts_Used__c`, `Technician_Notes__c = "COMPLETED REPAIR\n\nRepair Summary:\n…\n\nRoot Cause:\n…\n\nParts Used:\n… (or 'None recorded')\n\nLabor Hours:\n…"`, `Labor_Hours__c = visit labor`, `Issue_Resolved__c = true`, `Follow_Up_Required__c = false`, `Performed_By__c` and `OwnerId` = tech, plus a `Problem_Summary__c` snapshot.
- Asset → `Active`, `Last_Service_Date__c = today`.
- Case → `Closed`.
- Work Order (written last) → `Completed`, `Labor_Hours__c += visit labor`, note appended, `Work_Ended_Date_Time__c = now`, `OwnerId = Service_Manager__c`, `Suggested_Technician__c` cleared. `Work_Started_Date_Time__c` and `Scheduled_Visit__c` are preserved.
- Active validation rules: `Technician_Notes__c` not blank, `AssetId` not blank.
- Work Order Parts are not touched. `Received` stays `Received` (nothing sets `Installed`).
- `Resolution_Status__c` is **not** set.

## 10. Cancelled

`Cancelled` is a valid `Dispatch_Status__c`. Customer-facing controllers
display it, and the Operations Center treats it as terminal. **No current
workflow sets it**, and the source defines no cancellation reason or side
effects for Case, Asset or parts. Part-level `Cancelled` (manager removes a
requested part) is a different thing and is fully supported.

## 11. Legacy automation (recognize it, do not model it)

| Item | Why it is legacy | Behavior that differs from current |
|---|---|---|
| `Temporary_Fix_From_Work_Order` flow | Not wired to any Quick Action (the `Temporary_Fix_Follow_Up` action uses the LWC) | Writes `Dispatch_Status__c = 'Temporary FIx'` (typo, invalid for the restricted picklist) and puts the reason in `Root_Cause__c`. |
| `Complete_Work_Order` flow | Quick Action labelled "ZZZ Complete Work Order" | Same outcome as current, but no `Service_Manager__c` precondition. |
| `Request_Parts` flow | "zzz Request Parts" | Similar outcome to current. |
| `Start_Work` flow | "ZZZ Start Work" | Same outcome. |
| `X13_Order_Parts_From_Work_Order` flow | Not launched by the current UI | Creates parts directly as `Ordered` with `Quantity_Used__c = 0` and sets **Asset → `Waiting on Parts`**. |
| `Receive_Parts_From_Work_Order` flow | Not launched by the current UI | Sets **Asset → `Ready to Assign`**. |
| `Generate_Work_Order_Proposal_From_Case` screen flow | Superseded by the `managerReviewWorkOrderWizard` LWC | Same status outcome. |
| `Work_Order_Created_or_Pending_Dispatch` flow | Entry criteria use `Dispatch_Status__c = 'Manager Reviewed'`, which is not a picklist value, so it effectively never fires | — |
| `Dispatch_Technician` flow (Quick Action still labelled normally) | Parallel to the LWC dispatch wizard | Also sets Asset → `Needs Service`. Otherwise the same. |
| `Resume_Work` flow | Its UI entry point was not found in the supplied source | Sets `In Progress`. |
| `Customer_Create_Case`, `CUST_05_…_V2` | Draft | — |
