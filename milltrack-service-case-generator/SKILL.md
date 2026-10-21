---
name: milltrack-service-case-generator
description: Generates realistic, internally consistent synthetic MillTrack Intelligence service-case data (Case, WorkOrder, Asset, Maintenance_Log__c, Work_Order_Part__c) for dev, QA, demos, fixtures and seed data, at any lifecycle stage from a new customer request to Pending, Parts to Order, Waiting on Parts, Ready to Assign, Assigned, In Progress, Temporary Fix or Completed. Outputs readable scenarios, Salesforce field maps, JSON, CSV or Apex seed code. Use whenever someone asks to generate, mock, seed, stage or fake MillTrack cases, work orders, service requests, repair history, maintenance logs or demo data for a dental milling machine (DWX, Roland, Ceramill, etc.), even if they don't say "test data". Not for building or changing MillTrack workflows, reviewing Apex/Flows/security, troubleshooting real machines, or writing full Apex test classes.
---

# MillTrack Service Case Generator

MillTrack Intelligence is a Salesforce service app for dental CAD/CAM milling
machines with Customer, Service Manager and Technician personas. This skill
produces synthetic service records that would look right in that org. The
records use real field API names and restricted picklist values, agree with
each other, and match what the current automation would have written.

Every value here comes from the MillTrack source (objects, validation rules,
the LWC-backed Apex controllers and the record-triggered flows), not from
generic Salesforce defaults. When the references and a user-supplied newer
source disagree, follow the user's source and say so.

## Reference files: read before generating

| File | Read when |
|---|---|
| `references/milltrack-case-lifecycle.md` | Always. It has the state table and what each transition writes. |
| `references/milltrack-case-data-model.md` | Always, for exact API names, required fields, picklists and validation rules. |
| `references/milltrack-scenario-rules.md` | Always, for synthetic-data rules, symptom vs solution, the per-stage record checklist and defaults to avoid. |
| `references/milltrack-seed-output-rules.md` | When choosing the mode, or emitting a field map, JSON, CSV or Apex. |

## Workflow

1. **Classify the request.** Note the stage(s), the machine (manufacturer/model),
   the supplied facts (symptoms, troubleshooting, error code, priority), the
   output format, and how many scenarios.
2. **Choose the mode** and state it: `Live automation` (a new request inserted
   into an org with automation on: emit the Case only) or
   `Isolated / staged fixture` (any later stage, historical or demo data).
   See the seed rules, section 1.
3. **Build each scenario from the state table.** Set every record's status
   from the lifecycle reference's row for that stage: Case.Status,
   `Dispatch_Status__c`, Asset.Machine_Status__c, owner, technician/schedule,
   parts, logs. Include only the records the stage needs.
4. **Write the content by persona.** Customer facts go in Case
   Subject/Description. Manager plan fields go on the WorkOrder (staged mode
   only). Technician narrative goes in `Technician_Notes__c` and the logs, using
   the note formats the Apex writes. Appended Work Order note blocks are
   separated by a blank line (`\n\n`), in visit order.
5. **Run the consistency checks** in the scenario rules (section 5) before
   output: same Asset everywhere, labor sums, date order, required fields
   from validation rules, no outstanding parts where dispatch needs none.
6. **Output the smallest useful format** (below).

## Rules that matter most

- **`WorkOrder.Dispatch_Status__c` is the workflow status.** Use the exact API
  values: `Pending`, `Parts to Order`, `Waiting on Parts`, `Ready_to_Assign`,
  `Assigned`, `In Progress`, `Temporary Fix`, `Completed` (and `Cancelled`, see
  below). Write `Ready_to_Assign`, never "Ready to Assign". Standard
  `WorkOrder.Status` is not a MillTrack workflow field. Do not use it to
  represent or infer MillTrack lifecycle state. Only populate it when
  Salesforce requires a valid standard Status for record creation or platform
  compatibility; if the required value is not established by the supplied
  source or target org, use Cannot Verify rather than inventing one.
- **Respect live automation.** A new Case insert makes the Case trigger create
  the Pending Work Order, set the Asset to `Needs Service` and set
  `Work_Order_Proposal_Created__c = true`. In Live mode, never emit a second
  Work Order. In staged mode, set `Work_Order_Proposal_Created__c = true` so
  the trigger skips.
- **One active Work Order per Case.** A follow-up after a Temporary Fix reuses
  the same Work Order.
- **No fabricated Ids.** Use placeholders such as `<ACCOUNT_ID>` and
  `<TECHNICIAN_USER_ID>`, and aliases such as `@case1`. In code, use the Ids
  returned by insert, or query by an identifier the user supplied.
- **Customer report ≠ service solution.** The Case holds symptoms and the
  customer's own troubleshooting, and omits unknown facts (do not write "no
  error code"). Repair direction lives in WorkOrder plan fields. Leave
  `Case.Recommended_Solution__c` blank.
- **Never invent manufacturer facts.** No real error codes, no code meanings,
  no repair or calibration procedures, no "required" parts, no manual pages.
  Store a user-supplied error code verbatim with no interpretation. Staged
  narratives stay generic and are flagged as synthetic.
- **Data is synthetic and looks it.** Use fictional names marked "(Demo)" and
  synthetic serials. Never imply a real customer's history.
- **Satisfy the active validation rules.** A resolved log needs `Root_Cause__c`.
  `Temporary Fix` and `Completed` Work Orders need `Technician_Notes__c`.
  `Completed` needs `AssetId`, and the current Apex also requires
  `Service_Manager__c`.
- **Do not populate `Resolution_Status__c`**, `Shipped`/`Installed` part
  statuses, or the legacy Asset values `Waiting on Parts`/`Ready to Assign`
  unless asked. No current code writes them.
- **Cancelled:** do not generate it by default. If asked, set only the status
  and add the Cannot Verify line from the seed rules (section 8).
- **Parts need a Part Catalog record.** Use `<PART_CATALOG_ID>`, with
  Cannot Verify. Invent a clearly fictional catalog item (with a matching
  `Manufacturer__c`, `Active__c = true` and a real `Parts_Category__c` value)
  only when the user asks for fully synthetic seed data. Never call it the
  manufacturer-required repair.

## Output

Keep it short. Choose the format from the request:

- Human-readable scenario (default): use the template below.
- "Field map" or "Salesforce records": tables of exact API names and values.
- "JSON": the structure in the seed rules, section 5.
- "CSV": one file per object, with an alias column.
- "Apex": only when explicitly asked and the target (org seed vs test) is
  clear. For full test classes, defer to `salesforce-apex-test-generator`.

Single scenario template:

```
## Generated Service Case
**Mode:** Live automation | Isolated / staged fixture · **Synthetic demo data**
2–4 sentence scenario.

## Salesforce Records
<one table per record the stage needs; exact API names; placeholders/aliases>

## Consistency Checks
- 3–5 bullets on the lifecycle points that make this stage valid

## Cannot Verify
- only material unknowns (omit the section if none)
```

For batches, put a single short header (mode, synthetic note) at the top, then
a compact summary table (one row per scenario: stage, Case.Status,
`Dispatch_Status__c`, Asset status, parts, logs). Follow with per-object
record tables, or short per-scenario blocks. Do not write a long narrative for
each case. Put shared Cannot Verify items once at the end.

## Out of scope: hand off instead

- Adding or changing MillTrack workflows → `milltrack-workflow-builder`
- Apex review → `salesforce-apex-code-reviewer`
- Flow review → `salesforce-flow-reviewer`
- Permission/security review → `salesforce-security-permission-auditor`
- Full Apex test classes → `salesforce-apex-test-generator`
- Checking whether implemented states and architecture are correct → `milltrack-workflow-auditor` (planned)
- Diagnosing a real machine fault → not this skill; it has no service-manual grounding

If you notice a source defect while generating data (for example, the legacy
Temporary Fix flow writes the invalid value `Temporary FIx`), mention it in one
line and keep generating from the current path.

## Regression check

After any change to this skill or its references, re-run these three prompts.
Each one must still pass.

1. **New request (Live):** "Generate a new high-priority DWX-53DC service request. The customer reports step-in cuts after changing the bur, checking CAM settings, running automatic correction and checking machine stability. No error code is known."
   - Pass if the output is the Case only, with `Status = New`, `Priority = High`, no error code and no manager solution.
   - It must say that the Case trigger creates the Pending Work Order.
   - No fabricated Ids, and the output stays concise.
2. **Temporary Fix (staged):** "Generate an isolated demo fixture for a Work Order currently in Temporary Fix."
   - Pass if WO is `Temporary Fix`, Case is `Follow-Up Required` and Asset is `Temporary Fix Applied`.
   - The `Temporary Repair` log is unresolved, `Technician_Notes__c` is filled and the WO is back on the queue.
   - `Work_Order_Proposal_Created__c = true`, no invented status or timestamp fields, data marked synthetic and no fabricated Ids.
3. **Six-stage batch (staged):** "Generate 6 synthetic MillTrack demo cases: one Pending, one Ready to Assign, one Assigned, one Waiting on Parts, one Temporary Fix and one Completed."
   - Pass if all six stages are coherent and use `Ready_to_Assign`.
   - Waiting on Parts has outstanding parts, Temporary Fix has an unresolved log, and Completed has a resolved log with a Root Cause.
   - No Cancelled case, no invented error codes and no fabricated Ids.
   - Output is a compact summary table plus per-object tables.
