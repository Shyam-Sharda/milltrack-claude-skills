# MillTrack Seed and Output Rules

How to turn a scenario into a field map, JSON, CSV or Apex without fabricating
Ids or duplicating live automation.

## 1. Pick and state the mode

Every output declares one mode on its first line of record data.

| Mode | Use when | What you emit |
|---|---|---|
| **Live automation** | The user wants a *new* service request that will be inserted into an org where MillTrack automation is active (portal-style or "create a case") | The Case only. Describe what the automation will create (the Pending WO, the Asset → Needs Service, `Work_Order_Proposal_Created__c = true`) instead of emitting those records. |
| **Isolated / staged fixture** | The user asks for a fixture, a demo dataset, historical records, a specific later lifecycle stage, or anything past `New` | Every record the stage needs (see the scenario rules, section 5). |

Defaults: "new request" with no stage → Live automation. Any named stage other
than New, or "historical", "demo dataset" or "fixture" → staged.

If the mode is genuinely ambiguous **and** it changes the records you would
emit, ask one short question, or pick the safer reading and say so under
Cannot Verify. The safer reading is Live for a single new request and staged
for everything else.

## 2. Never fabricate Salesforce Ids

Never invent Account, Contact, User, Queue, Asset, Case, WorkOrder,
Part_Catalog__c, Maintenance_Log__c or Work_Order_Part__c Ids. Never use
15/18-character strings that look like Ids.

Use symbolic placeholders for records that already exist:

`<ACCOUNT_ID>` `<CONTACT_ID>` `<ASSET_ID>` `<SERVICE_MANAGER_QUEUE_ID>`
`<SERVICE_MANAGER_USER_ID>` `<TECHNICIAN_USER_ID>` `<PART_CATALOG_ID>`

Use logical aliases for records created in the same fixture. Refer to them with
an `@` reference:

- `case1`, `wo1`, `log1`, `wop1`
- `"WorkOrder.CaseId": "@case1"`

Never set the autonumber Names (`ML-…`, `WOP-…`), `CaseNumber` or `WorkOrderNumber`.

## 3. Avoid duplicating the live Work Order

`Create_Work_Order_Proposal_From_Case` runs synchronously after a Case insert
when `Status != Closed AND Work_Order_Proposal_Created__c = false`.

- **Live mode:** insert the Case with `Status = New` and `Work_Order_Proposal_Created__c` unset. Do not insert a WorkOrder.
- **Staged mode, inserting into an org with automation on:** set `Case.Work_Order_Proposal_Created__c = true` on every Case you insert with its own WorkOrder. That flag is the flow's own "already has a Work Order" signal, and setting it skips the flow. A staged `Completed` Case inserted as `Closed` is also skipped.
  - `SYS_01_Set_Asset_Service_Status` still fires for `Status = New` Cases with an Asset and sets the Asset to `Needs Service`. That matches the Pending stage. For other stages, insert the Case at its target status.
  - **Cannot Verify** whether the target org disables automation for data loads (bypass custom settings or permissions were not in the source).
- Insert order for staged data: Account → Contact → Asset → (Part_Catalog__c if synthetic) → Case → WorkOrder → Work_Order_Part__c → Maintenance_Log__c → Asset status update.
- `Work_Order_Assignment_Sync` only fires on WorkOrder **update**. Inserting an `Assigned` WO with `Suggested_Technician__c` is safe. Updating a `Ready_to_Assign` WO that has `Suggested_Technician__c` set flips it to `Assigned`.

## 4. Field-map output (default for "Salesforce records" or "field map")

One block per record, with exact API names. Include only fields the stage
needs. Omit blanks.

```
### Case (alias: case1)
| Field | Value |
|---|---|
| AccountId | <ACCOUNT_ID> |
| Status | New |
...
```

For several scenarios, put the stage-defining fields in one compact table per
object, with one row per scenario.

## 5. JSON output

```json
{
  "mode": "Isolated / staged fixture",
  "synthetic": true,
  "records": [
    { "alias": "case1", "sobject": "Case", "fields": { "AccountId": "<ACCOUNT_ID>", "Status": "Follow-Up Required", "Work_Order_Proposal_Created__c": true } },
    { "alias": "wo1", "sobject": "WorkOrder", "fields": { "CaseId": "@case1", "Dispatch_Status__c": "Temporary Fix" } }
  ]
}
```

List records in insert order. Use real JSON types (booleans and numbers, not
strings). Use ISO 8601 for DateTime and `YYYY-MM-DD` for Date. Write
multi-line note fields with `\n` escapes, and separate appended
`WorkOrder.Technician_Notes__c` blocks with `\n\n`. In CSV, quote those
fields and keep the real line breaks. In Apex, build them with
`'\n\n'` concatenation.

## 6. CSV output

- One CSV per object, since rows have parent relationships. Give each file a heading like `WorkOrder.csv`.
- Add an `Alias` column, and use `@alias` (or an External Id column if the user names one) in lookup columns. Say that the loader must resolve aliases to Ids between object loads.
- Order the files by insert order.

## 7. Apex seed code (only when explicitly asked)

Write Apex only when the user asks for code **and** the target is clear.

- **Org seed script (Execute Anonymous):** query existing dependencies by a **user-supplied** stable identifier, e.g. `[SELECT Id FROM Asset WHERE Serial_Number__c = :serialFromUser LIMIT 1]`, and fail loudly if nothing matches. Never assume a Name or serial exists. Look up the queue by `DeveloperName = 'Service_Manager_Queue'` and users by a username or email the user supplies. Set `Work_Order_Proposal_Created__c = true` on staged Cases.
- **Test data inside a test class:** create the records in the test and use the Ids returned by `insert`. Users need role `MillTrack_Technician` for dispatch logic. Note the Case trigger calls Prompt Builder synchronously and has no fault path: **Cannot Verify** how it behaves in an Apex test context. Set `Work_Order_Proposal_Created__c = true` to keep test inserts deterministic. For in-depth test classes, point to `salesforce-apex-test-generator`.
- Use `Database.insert` with lists, and insert parents before children. No hard-coded Ids.

## 8. Cannot Verify wording

Use a `## Cannot Verify` section only for material unknowns, one line each:

- `**Cannot Verify:** the required Part Catalog record was not supplied. Use <PART_CATALOG_ID> for a catalog part whose Manufacturer__c matches the Asset.`
- `**Cannot Verify:** Maintenance_Log__c.Problem_Summary__c / Recommended_Fix_Snapshot__c / Tools_To_Bring_Snapshot__c are evidenced by current Apex references, but their field metadata was not supplied. Their exact type, length, defaults, validation and other metadata cannot be verified from this archive.`
  - Add "Included as copied by current Apex." or "Omitted." to say which choice the fixture made. Never say the fields themselves might not exist.
  - Mention this only in Temporary Fix, parts-request and Completed fixtures.
- `**Cannot Verify:** the current source does not establish a Cancelled workflow, reason, or side effects for Case, Asset or parts.`
  - Use this for any Cancelled Work Order request. Produce the WO status only, and state which side effects were *not* assumed.
- `**Cannot Verify:** the meaning of error code <code>; stored verbatim only.`
  - Use this only when the user asks what a code means.
- `**Cannot Verify:** the Case Status/Priority/Origin standard picklists were not supplied; values used are those the MillTrack code writes.`
  - Use this only when someone will load the data and it matters.
