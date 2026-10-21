---
name: "milltrack-workflow-builder"
description: Designs and builds new or changed business workflows for MillTrack Intelligence, the Salesforce service app for dental CAD/CAM milling machines with Customer, Service Manager, and Field Technician personas. Finds the existing LWC, Apex, or Flow that already owns the step, keeps Dispatch_Status__c as the workflow status, and proposes the smallest change. Use whenever someone asks how to add, change, automate, or build something in MillTrack (a notification, reminder, queue, wizard step, status change, parts or dispatch behavior, temporary-fix follow-up, or a customer portal action), even if they don't say "workflow". Not for general Apex, Flow, or security reviews of existing code.
---

# MillTrack Workflow Builder

You are adding behaviour to a production Salesforce app that already has a clear
architecture. Most of the value you add comes from finding the component that already
owns the step and changing as little as possible. A generic Salesforce answer, such as
"create a record-triggered flow on WorkOrder.Status", is usually wrong for MillTrack.

## Load context first

Read the reference that matches the request before you reason about it. Each is short
and has a table of contents.

| Read | When |
|---|---|
| `references/milltrack-workflow-map.md` | Always. It shows which LWC, Apex method, or Flow owns each step and each `Dispatch_Status__c` transition |
| `references/milltrack-architecture.md` | Lifecycle, persona boundaries, the status model, AI boundaries, security summary, known drift |
| `references/milltrack-data-model.md` | Before naming any field or picklist value, or deciding a field is missing |
| `references/milltrack-development-rules.md` | Before generating code, or when choosing Flow vs Apex vs LWC |

These references are a snapshot of the source. If the MillTrack repo is available
(`force-app/main/default/...`), open the specific classes, LWCs, and flows you plan to
change and trust them over the references when they disagree. If they disagree, say so.

## Rules that matter most

1. **`WorkOrder.Dispatch_Status__c` is the workflow status.** Use it in every decision,
   queue, visibility rule, Flow entry condition, Apex check, and customer progress
   mapping. Standard `WorkOrder.Status` is not a MillTrack business-workflow field.
   Populate it only when Salesforce requires a valid standard Status for record creation
   or platform compatibility, and never use it in place of `Dispatch_Status__c` to
   represent MillTrack state. Compare against picklist
   API names, which include `Ready_to_Assign` with underscores. Don't add a new status
   value or a parallel status field unless the user explicitly approves it. A new value
   spreads to the Operations Center queues, technician home filters, the calendar,
   customer progress, and list views.
   `Case.Status`, `Asset.Machine_Status__c`, and `Work_Order_Part__c.Order_Status__c` are
   secondary statuses. Existing code updates them in the same transaction as the
   `Dispatch_Status__c` change, and so should new code.

2. **Extend the owner and don't duplicate it.** Current workflows run as an LWC wizard
   calling a persona-named Apex controller. Manager wizards are hosted in
   `managerOperationsCenter`, and technician wizards are WorkOrder quick actions. Several
   older Screen Flows that do the same job are still active, marked `ZZZ`/legacy in the
   map. Extend the LWC/Apex path, not the legacy flow. Don't add a second automation that
   performs a transition something already performs, and point out existing overlaps
   (for example, assignment) instead of adding to them.

3. **Keep personas separate.** Customers work in Experience Cloud, can only read Work
   Orders, and every query is scoped to their Account. Managers coordinate through the
   Operations Center. Technicians act only on Work Orders they own while the order is
   `In Progress` (Start Work needs `Assigned`). Don't expose manager or technician
   actions or data to customers, and don't hand technicians manager decisions.

4. **Don't invent.** Never make up object, field, or picklist API names, classes, LWCs,
   flows, Custom Permissions, permission sets, record types, notification types,
   relationships, endpoints, user assignments, or business rules. When something material
   isn't in the references or the source, write:
   **Cannot Verify:** <what is missing>. <What would settle it: which file, or which decision the user must make>.
   If a new field or value really is needed, propose it explicitly as new metadata and
   say why no existing field fits.
   A reference in source doesn't prove that the metadata it names is deployed. When Apex,
   a Flow, or another file references a Custom Notification Type, permission source,
   Custom Permission, Record Type, Flow, or other metadata whose definition wasn't
   supplied, say what the source expects and ask the user to confirm it's deployed.
   Example: "The source expects a Custom Notification Type with DeveloperName
   `Temporary_Repair_FollowUp`, but its metadata was not supplied. Confirm that it is
   deployed." If that matters to the build, list it under Cannot Verify.

5. **Use the smallest change that fully meets the requirement.** Check first whether
   MillTrack already does it, wholly or partly. If it already does, say so and stop, or
   ask what the user means beyond it. Don't refactor, rename, or fix unrelated drift. You
   can mention drift in one line when it affects the request.

## How to work through a request

Do this reasoning internally and keep the written answer short.

1. **Locate** the existing feature or closest workflow in the workflow map.
2. **Persona** – decide whose action or experience this is.
3. **States** – find the starting `Dispatch_Status__c`, the target status (if any), and
   which component owns that transition today.
4. **Data** – identify the existing objects and fields that express the need, using the
   data model and its "Concepts with NO field today" list.
5. **Mechanism** – choose LWC, Apex, or Flow using the guidance in development-rules §6.
   MillTrack currently has no Apex triggers, asynchronous Apex
   (Queueable/Batch/Schedulable/@future), scheduled Flows, or scheduled paths. It does
   have live record-triggered Flows. Time-based work is therefore new capability; say so
   when that applies. Scheduled paths run in system context. Don't infer end-user
   CRUD/FLS requirements from that alone, and don't describe the running user as the
   Automated Process user unless the supplied implementation establishes it. Focus on
   whether the automation itself needs new metadata or access, not on inventing a
   permission change for a persona.
6. **Access** – check whether the persona already has the object, FLS, class, and record
   access the change needs. If not, name the gap. If you can't tell, mark it Cannot
   Verify. Don't run a full audit. Tie every access change to a concrete behavior in
   the requested feature. If a persona gains no new read or write behavior, don't add
   permissions for that persona, even when a new field exists that it might want to see
   later. If the user later asks for that persona to see it, add the UI or read path
   and the FLS together then.
7. **Smallest change** – list exactly which files change, and mark each one new or modified.

## Plan or build?

- "How should we…", "Where would…", "What's the best way to…" → give the plan only.
  End with one line offering to generate the files.
- "Build…", "Create the files…", "Implement…", "Write the code…" → generate the files.
  - Give the complete file for each new file. For a modified file, give the complete
    file when it's short; otherwise give the complete changed methods or sections, with
    enough surrounding context that the change can be placed exactly.
  - Label each file **New** or **Modified**, with its path under `force-app/main/default/`.
  - Follow `milltrack-development-rules.md`: naming, the `with sharing` and DTO pattern,
    `handledException`, savepoints, status constants, the security mode for the persona,
    append-only notes, and notifications whose failure becomes a warning instead of a rollback.
  - Include `-meta.xml` files for new classes, LWCs, and flows. Update the LWC Jest file
    when changing an LWC.
  - Apex tests:
    - **New Apex class:** generate a matching new `*Test` class.
    - **Modified existing class, test class supplied:** update that test class as needed.
    - **Modified existing class, test class not supplied:** don't invent, replace, or
      duplicate a test class. Write: **Cannot Verify:** the existing tests that cover
      `[ClassName]` were not supplied. The matching test class should be inspected and
      updated for this change. When the test class's name is evidenced, list it under
      Files to Change, but don't make up its contents. If the user asks for the tests,
      recommend `salesforce-apex-test-generator`, given both the production class and
      the existing test class.
  - Don't change unrelated behaviour. If you have to touch a file that has known drift,
    leave the drift alone and mention it.

## Output format

Use only these default sections. For a plan-only request, one short closing sentence
offering to generate the implementation is allowed.

```
## Recommended Build
2–5 sentences: where this fits in MillTrack, which existing component owns it, and the mechanism.

## Files to Change
- `path or component` — New/Modified — one-line purpose
(only files actually involved)

## Important Logic
- the few rules that make this correct (status checks, timing source, side effects, access)

## Cannot Verify
- only if something material is missing; one line each with what would settle it
  (omit the section otherwise; never fill it with speculative feature ideas)
```

Don't add sections such as Architecture Review, Apex Review, Flow Review, Security Review,
Database Design, Testing Strategy, Deployment Strategy, or Overall Assessment unless the
user asks for them.

**Scale the answer to the request.** A small change gets a few lines: where it belongs,
which file, the key condition, and the fix. Then offer to build it. Don't turn "add one
manager notification when parts arrive" into a design document.

**When MillTrack already does what was asked**, keep the answer very short:
1. Say that first.
2. Name the current owner (LWC and Apex method, or Flow).
3. Describe the existing behavior in a sentence or two.
4. Under Files to Change, write `None`.
5. Mention one overlap or drift item only if it materially affects the request.
6. Ask for clarification only if the user might mean a different behavior. Don't label
   unspecified alternatives as small changes unless the source proves it. Use neutral
   wording: "If you mean a different behavior, such as X, Y, or Z, clarify which one and
   I can evaluate the smallest correct change."
Omit Cannot Verify unless something material is unknown.

## Out of scope: send these to other skills

This skill designs and builds MillTrack workflows. For reviews, use:
- `salesforce-apex-code-reviewer` – quality review of existing Apex
- `salesforce-flow-reviewer` – quality review of existing Flow metadata
- `salesforce-security-permission-auditor` – full permission or access audits
- `salesforce-apex-test-generator` – in-depth Apex test generation
- `milltrack-workflow-auditor` (planned) – architecture-aware review of completed MillTrack workflows

If the request is "review this" rather than "add or change this", say which skill fits
instead of running a review here.

## Example requests

**1. "Alert the Service Manager if a technician records a temporary fix and the Work Order is still unresolved 3 days later."**
Find the temporary-fix owner (`technicianTemporaryFixWizard` → `TechnicianTemporaryFixController`)
and the follow-up owner (`managerFollowUpWorkOrderWizard`, Operations Center "Follow-Up
Required" queue). "Unresolved" means `Dispatch_Status__c` is still `Temporary Fix`. Prefer
existing timing evidence over a new timestamp field when it meets the requirement. The
Temporary Repair Maintenance Log `CreatedDate` provides an existing precise timestamp
associated with the temporary-fix transaction, but don't claim it is identical to the
moment the Work Order's status changed. The time the record entered `Temporary Fix`
(the scheduled-path trigger time) is the other option. A scheduled path or scheduled Flow
fits, and it would be MillTrack's first scheduled automation. Note that Apex already sends
an immediate notification, so this is a reminder and not a new follow-up process.

**2. "When parts are received, automatically move the Work Order back into the manager dispatch queue."**
Find `managerReceivePartsWizard` → `ManagerReceivePartsController.receiveParts`. It already
moves `Waiting on Parts` → `Ready_to_Assign` (the dispatch queue) once no part is Requested,
Ordered, or Shipped. Say that it already exists, with Files to Change: `None`. If the user
might mean something else, use the neutral clarification wording, for example: partial
receipt returning to dispatch, a notification, or reassigning the original technician.
Don't predict how big any of those would be, and don't add a parallel flow.

**3. "Add a customer-facing button that lets the customer confirm the machine is working after service."**
This belongs in the Experience Cloud layer, most likely `customerWorkOrderDetails` plus a new
method on `CustomerWorkOrderDetailsController`. Customers can only read WorkOrder, and no
confirmation field exists, so the state to record and the write path are open decisions.
Raise them as Cannot Verify and propose options. Re-verify Account ownership on the server,
and don't show technician or manager controls. Don't widen customer WorkOrder Edit access to
support this one controlled write. Don't add manager FLS for any new confirmation field
unless the request also asks managers to see the confirmation.

## Testing plan

After changing this skill, run these three fixtures and check the pass criteria.

**Fixture 1: extend an existing workflow.** "When the technician applies a temporary fix, create a manager follow-up after 3 days."
Pass if it finds and reuses the temporary-fix and follow-up architecture, uses
`Dispatch_Status__c`, doesn't create a parallel process, flags missing date or timestamp
information only if it is really absent, and stays concise.

**Fixture 2: new workflow with an architecture choice.** "Let customers confirm that the repair is successful from the portal."
Pass if it puts the interaction in the customer Experience Cloud layer, extends an existing
customer Work Order component where that's appropriate, identifies the server-side and state
change needed, respects persona boundaries, doesn't expose technician or manager
functionality, and doesn't invent fields.

**Fixture 3: mostly reuse.** "When received parts are logged, return the Work Order to dispatch."
Pass if it recognizes the existing parts-receiving and dispatch components, proposes the
smallest change (here, none or only a clarification), doesn't create a new parts system or
an unnecessary Apex service, and stays short.
