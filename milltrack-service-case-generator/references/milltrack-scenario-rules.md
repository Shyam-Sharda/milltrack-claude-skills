# MillTrack Scenario Rules

Rules for making generated service scenarios realistic, coherent and honest.
Lifecycle facts are in `milltrack-case-lifecycle.md`; field facts are in
`milltrack-case-data-model.md`.

## 1. Synthetic, and obviously so

MillTrack runs in production for one real customer. Generated data must never
look like that customer's records.

- Use clearly fictional identities: "Northbay Dental Lab (Demo)", "Cedar Ridge
  Dental Studio (Demo)", "Alex Rivera (Demo)". Do not use real labs, clinics
  or people. Avoid names that read like a real business without a demo marker.
- Synthetic serial numbers should look synthetic: `DEMO-53DC-0001`, not a
  realistic manufacturer serial format. If the user wants a real Asset, use
  `<ASSET_ID>` and do not invent a serial.
- Mark batch output as synthetic once, at the top. Every record does not need
  the tag; the records themselves just must not imply a real event.
- Use production data only when the user supplies it for this task.

## 2. Customer report vs service solution

These are written by different personas, live in different fields, and must
stay separate.

| Customer writes (Case) | Manager/AI writes (WorkOrder) | Technician writes (visit/log) |
|---|---|---|
| `Subject`, `Description`: symptoms, when they occur, troubleshooting the customer already tried, exact error code text, explicit technician request, operational impact *only if stated* | `Problem_Summary__c`, `Recommended_Solution__c`, `Recommended_Fix_Summary__c`, `Recommended_Tools__c`, `Recommended_Parts__c`, `Previous_Failure__c` | `Technician_Notes__c`, log `Summary__c`, `Root_Cause__c`, `Parts_Used__c`, `Labor_Hours__c` |

Case Description style (from the live `Prepare_Customer_Service_Case_Payload` prompt):

- 1–4 concise factual sentences, written as service documentation.
- Keep the customer's troubleshooting. It is customer data.
- No diagnosis, failed component, root cause, repair step, appointment or outcome.
- **Omit missing facts.** Do not write "No error code provided." Leave the field blank.
- No pleasantries or conversational filler.
- Subject: about 4–12 words, symptom-focused (e.g. "DWX-53DC step-in cuts after basic troubleshooting").

Priority (same rubric the live prompt uses): `Critical` = safety risk, machine
fully down, or production stopped. `High` = severe or urgent degradation while
still partly usable. `Medium` = degraded, intermittent, or unclear (the default
when unsure). `Low` = minor, can wait. When the user states a priority, use it.

Leave `Case.Recommended_Solution__c`, `Case.Recommended_Parts__c` and
`Case.AI_Troubleshooting_Summary__c` blank. No current code writes them.

## 3. Machine-specific facts: plausible symptoms, no invented facts

The supplied source contains no manufacturer service manuals, so nothing in it
grounds machine-specific repair knowledge.

You may write plausible **synthetic symptoms and visit narratives** for demo
data ("step-in cuts on the restoration margin", "intermittent spindle noise at
startup"). You may not present any of these as manufacturer fact:

- real manufacturer error codes, or the meaning of any code
- exact repair or calibration procedures
- required replacement parts for a symptom
- service-manual instructions or page numbers

When the user supplies an error code, store it verbatim in `Error_Code__c` and
say nothing about what it means. When they supply none, leave the field blank.
Do not invent one, even for realism.

Manager-plan and technician fields in **staged** fixtures need content. Keep it
generic and clearly a demo narrative: "Inspect spindle and tool path behavior
per manufacturer documentation; verify calibration", not "Replace spindle
bearing P/N 12345 and run the X-axis calibration in menu 4.2". Root causes in
completed fixtures should read as plausible demo findings ("Worn collet
causing tool runout"). The output must say once that these are synthetic and
not manufacturer guidance.

Do not hand-author `Problem_Summary__c` or `Recommended_*` for a **live**
new-Case request. The proposal flow writes those.

## 4. Persona boundaries

- **Customer** creates the Case (Asset, Subject, Description, Priority, Error Code, Technician Requested). Customers never set Work Order fields, statuses or parts.
- **Service Manager** owns review, parts selection and ordering, receiving, dispatch and follow-up. They set `Service_Manager__c`, the plan fields, `Ordered_By__c` and supplier snapshots.
- **Technician** owns start, parts request, Temporary Fix and completion. They are `Performed_By__c` on logs and the Work Order owner while it is `Assigned` or `In Progress`.

Use the right placeholder for each role:

- `<SERVICE_MANAGER_USER_ID>` for `Service_Manager__c` and `Ordered_By__c`
- `<TECHNICIAN_USER_ID>` for `Suggested_Technician__c`, `Performed_By__c` and the owner while Assigned or In Progress
- `<SERVICE_MANAGER_QUEUE_ID>` for queue-owned records

## 5. Lifecycle consistency: records each stage needs

Check every scenario against this list and the state table in the lifecycle
reference before you output it.

| Stage | Must include | Must not include |
|---|---|---|
| New request (live) | Case only (`Status = New`) | Any WorkOrder, Work_Order_Part__c or Maintenance_Log__c inserted by you |
| `Pending` (staged) | Case `New`, `Work_Order_Proposal_Created__c = true`; WO `Pending` on the queue; Asset `Needs Service` | `Service_Manager__c`, technician, schedule, parts, logs |
| `Parts to Order` | WO with `Service_Manager__c` and plan fields; ≥1 WOP `Requested` (`Technician_Requested__c = false` on a first pass); Case `In Progress` | Technician, schedule, Ordered parts with nothing left Requested |
| `Waiting on Parts` | ≥1 outstanding WOP. Manager-ordered: `Ordered` + `Ordered_By__c` + supplier snapshot. Technician-requested: `Requested`, `Technician_Requested__c = true`, `Notes__c`, plus the visit Maintenance Log (`Inspection` or `Temporary Repair`, unresolved, follow-up) and `Labor_Hours__c`/`Work_Started_Date_Time__c` on the WO. Case `Waiting on Parts` | Technician or schedule on the WO; Asset `Waiting on Parts` (legacy value) |
| `Ready_to_Assign` | `Service_Manager__c`, plan fields; Case `In Progress` | Any `Requested`/`Ordered`/`Shipped` part; `Suggested_Technician__c` (the sync flow would flip it to Assigned) |
| `Assigned` | `Suggested_Technician__c` = owner = technician; future `Scheduled_Visit__c`; Case `Scheduled` | Outstanding parts; `Work_Started_Date_Time__c` |
| `In Progress` | Assigned fields + `Work_Started_Date_Time__c`; Case `In Progress` | Outstanding parts; `Work_Ended_Date_Time__c` |
| `Temporary Fix` | `Temporary Repair` log (unresolved, `Follow_Up_Required__c = true`, labor >0); WO `Technician_Notes__c` with a TEMPORARY REPAIR block, cumulative labor, `Work_Started_Date_Time__c` kept, owner = queue, technician and schedule cleared; Asset `Temporary Fix Applied` with `Last_Service_Date__c`; Case `Follow-Up Required` | `Issue_Resolved__c = true`; a new Dispatch Status; `Resolution_Status__c`; a second active WO |
| `Completed` | `Repair` log with `Issue_Resolved__c = true`, `Follow_Up_Required__c = false`, **non-blank `Root_Cause__c`**, labor >0; WO `Technician_Notes__c` with a COMPLETED REPAIR block, `AssetId`, `Service_Manager__c`, owner = manager, `Work_Started_Date_Time__c` < `Work_Ended_Date_Time__c`, technician cleared; Asset `Active` with `Last_Service_Date__c` = log date; Case `Closed` | Outstanding parts; `Resolution_Status__c` (unless asked) |

Cross-record checks:

- The same `AssetId` everywhere: Case, WO, `Asset__c` on logs, `Machine__c` on parts.
- `Case.AccountId = Asset.AccountId`. The Contact belongs to that Account.
- Work Order `Labor_Hours__c` = sum of its logs' `Labor_Hours__c`. Each log holds only its own visit's hours.
- Dates in order: Case created ≤ visit dates ≤ completion; `Maintenance_Date__c` = date of the visit that wrote it; `Last_Service_Date__c` = date of the most recent Temporary Fix or Repair log.
- Every Technician_Notes block that Apex would have appended is present, in visit order, and the blocks are separated by a blank line (`\n\n`), as the controllers write them. Each log's `Technician_Notes__c` holds only its own visit's block.
- A completed follow-up cycle has two logs on one WO: `Temporary Repair` (unresolved) then `Repair` (resolved).

## 6. One active Work Order per Case

The normal model is one Work Order per Case, created by the Case trigger and
carried through every visit, including follow-ups after a Temporary Fix.

Do not create a second non-terminal Work Order for a Case. Historical
`Completed` Work Orders may each have their own historical Case. A follow-up is
**not** a new Work Order.

## 7. Defaults to avoid

- No `Cancelled` scenarios unless asked. If asked, see the seed rules for the Cannot Verify wording.
- No `Shipped` or `Installed` parts unless asked (no workflow sets them). If asked, note that they are valid values with no current workflow behind them.
- No `Resolution_Status__c`.
- No standard `WorkOrder.Status` as a workflow signal. It is not a MillTrack workflow field. Only populate it when Salesforce requires a valid standard Status for record creation or platform compatibility. If the source or target org doesn't establish the required value, use Cannot Verify rather than inventing one.
- No invented status values (e.g. "Manager Reviewed", "Scheduled" on a WO, "Ready to Assign" with spaces).
- No `Emergency Service`, `Preventive Maintenance` or `Installation_And_Calibration` logs on the service-case path. Current code only writes `Inspection`, `Temporary Repair` and `Repair`. Use the other types only for explicitly requested non-workflow history, and say so.
