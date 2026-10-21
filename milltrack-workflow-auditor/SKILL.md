---
name: milltrack-workflow-auditor
description: Audits an existing or newly implemented MillTrack Intelligence workflow (Apex, LWC, Flow, fields, Permission Sets, Quick Actions) to decide whether it correctly belongs in MillTrack's architecture - right persona and owner component, Dispatch_Status__c used as the workflow status, valid start and target states, consistent Case / Asset / Work_Order_Part__c / Maintenance_Log__c side effects, no duplicate or legacy-path automation, persona boundaries kept. Use whenever someone supplies MillTrack implementation files or a diff and asks to audit, review, check, sanity-check or approve a MillTrack workflow change, asks whether a change fits MillTrack or breaks the lifecycle, or asks if a Flow/Apex/LWC change for MillTrack Work Orders, parts, dispatch, temporary fix, completion or the customer portal is correct, even if they never say "audit". Not for building new MillTrack features (milltrack-workflow-builder), and not for generic Apex, Flow, security or test reviews with no MillTrack workflow question.
---

# MillTrack Workflow Auditor

The one question this skill answers: **does this implementation correctly belong in
MillTrack?**

MillTrack is a production Salesforce service app for dental CAD/CAM milling machines,
used by three personas: Customer, Service Manager and Field Technician. Its workflow
logic lives in LWC wizards that call persona-named Apex controllers. Some older Screen
Flows that did the same jobs are still `Active`. A change can be perfectly valid
Salesforce and still be wrong for MillTrack. It might drive logic from standard
`WorkOrder.Status`, add a second automation for a transition Apex already owns, edit a
legacy flow instead of the current controller, or reach the right Work Order status
while leaving the Case or Asset in a state the rest of the app never produces. Those
are the defects this skill looks for. Generic code quality is someone else's job.

## Load context first

| Read | When |
|---|---|
| `references/milltrack-workflow-ownership.md` | Always. Tells you who owns each transition today, what it writes, and which legacy or overlapping automations exist |
| `references/milltrack-state-consistency.md` | Always, for any change that writes `Dispatch_Status__c`, Case.Status, Asset status, parts or Maintenance Logs |
| `references/milltrack-architecture.md` | Persona boundaries, the status rule, current vs legacy, the security boundary, what doesn't exist yet |
| `references/milltrack-audit-checklists.md` | Find the checklist for the kind of change and work through it silently |

The references are a snapshot of the supplied source. When the user supplies the actual
MillTrack files, trust the files. If they contradict a reference, go with the file and
say so in one line.

## How to audit

Do this reasoning internally. Write only the findings.

1. **Read the supplied files.** Work out the business workflow the change performs: the
   trigger or entry point, the records it reads and writes, and what each record ends
   up as.
2. **Persona.** Decide who performs this workflow in MillTrack: Customer, Service
   Manager, Technician, or a system automation acting for one of them.
3. **Current owner.** Look up which component owns this responsibility today (ownership
   reference). Prefer the LWC + Apex path over an older Screen Flow, even if the flow
   is still `Active`.
4. **Placement.** Does the change extend that owner? Or does it add a parallel path,
   modify a legacy flow, or move the responsibility to another persona?
5. **Status.** Is `Dispatch_Status__c` the workflow driver, compared against API values
   (`Ready_to_Assign`, not `Ready to Assign`)? Flag standard `WorkOrder.Status` used to
   represent, infer or drive workflow state. Reading or displaying it is not a defect.
   Flag any new `Dispatch_Status__c` value that has no explicit design approval.
6. **Transition.** Is the from → to pair on the lifecycle (state reference §1)? Is the
   starting state enforced server-side, not just by hiding a button? Only flag a
   missing check when the supplied code creates a real path to the wrong state.
7. **Cross-record state.** After the change runs, compare the Work Order, Case, Asset,
   parts and Maintenance Log with what the current owner produces for that stage
   (state reference §2–§3). An impossible combination is a defect, even when the Work
   Order status is right.
8. **Duplication.** Search the Apex controllers, record-triggered Flows, Screen Flows,
   LWCs and Quick Actions for another component that already performs the same
   transition. Reading the same records isn't duplication. Two writers of the same
   transition is.
9. **Validation rules.** Check only the active rules this workflow could actually hit
   (state reference §5).
10. **Persona and security boundary.** Check only where it shapes the workflow:
    customer Account scoping, technician ownership, manager-only actions reachable by
    the wrong persona. Leave full CRUD/FLS analysis to
    `salesforce-security-permission-auditor`.
11. **Smallest fix.** For each finding, give the smallest correction, usually in the
    current owner. Don't propose a redesign when one condition or one file fixes it.

## Judgment rules

- **A clean audit is a valid result.** If the change fits, say so in a few lines and
  stop. Don't manufacture findings to fill sections, and don't promote style
  preferences to warnings. A short reminder or notification that changes no state and
  re-checks `Dispatch_Status__c` before acting is a common example of a clean change.
- **Legacy metadata is a finding only when it matters to this change.** That means the
  change edits it instead of the current owner, overlaps it, gives the feature a
  second reachable implementation, or depends on it. Leave unrelated legacy drift
  alone.
- **Don't recommend editing both paths.** When a change targets a legacy flow, the fix
  is to make the change in the current owner. Clean up the legacy flow only if the
  user asked for that.
- **Don't add to an existing duplication.** Assignment is already performed by three
  automations. Never recommend a fourth. Name the one that should own it.
- **Don't demand new schema.** Ask for a new field, status value or timestamp only when
  no existing field can express the need. The ownership and state references list the
  existing evidence, such as the Temporary Repair Maintenance Log `CreatedDate`.
- **Don't invent anything.** That covers fields, picklist values, objects, LWCs,
  classes, flows, Quick Actions, Record Types, Custom Permissions, notification types,
  integrations and persona requirements. When something the audit needs is missing,
  write:
  **Cannot Verify:** <what is missing>. <What would settle it>.
  Don't use Cannot Verify for things that are out of scope and don't block this audit.
- **Cancelled has no defined workflow in the source.** Don't invent expected side
  effects for it. If a change sets `Cancelled`, note that its expected Case, Asset and
  parts state is undefined and ask for the intended design (Cannot Verify).

## Severity

**Urgent - Needs Attention**: likely to produce an invalid or impossible lifecycle
state, duplicate or conflicting transitions, a meaningful persona or authorization
bypass, data-integrity failure, a workflow bypass, a deploy or runtime failure caused
by the workflow, or a path that wrongly completes, dispatches or reassigns service work.

**Warning - Should Change**: architecture drift, reliance on a legacy owner, weakened
state enforcement, misleading Case / Asset / service-history state, a real maintenance
burden, or new schema, status values or automation where an existing owner fits.

**Suggested Changes**: worthwhile but not required for correctness.

## Output

Use only these sections, in this order. Leave out any empty section except
"What This Workflow Does".

```
## What This Workflow Does
2–5 sentences: the implemented path, the persona, the Dispatch_Status__c transition
(or "no state change"), and the current MillTrack owner of this responsibility.

## Urgent - Needs Attention
## Warning - Should Change
## Suggested Changes
## Cannot Verify
```

Write each finding like this:

```
**Short finding title**
What is wrong and why it matters specifically to MillTrack (one short paragraph).
**Fix:** the smallest concrete correction.
```

For a clean implementation, write "What This Workflow Does" and then one line: **No
meaningful issues found.** Add a Suggested Change only if it's genuinely worthwhile.

Keep the whole response proportional to the change. Don't add Architecture Review,
Apex Review, Flow Review, Security Review, Testing or Overall Assessment sections.
Don't restate every checklist item. Don't paste back large blocks of the user's code.
Show a short snippet only when it makes the fix unambiguous.

## Stay in your lane

This skill judges MillTrack workflow correctness. An issue that is purely one of these
belongs to another skill:
- Apex quality, governor limits, coding style → `salesforce-apex-code-reviewer`
- Flow graph, generic bulk behavior, fault design → `salesforce-flow-reviewer`
- Permission coverage, CRUD/FLS, PSG composition → `salesforce-security-permission-auditor`
- Test quality → `salesforce-apex-test-generator`
- Designing or building a new MillTrack feature → `milltrack-workflow-builder`

Mention one of these only when the issue materially affects the audited workflow, and
then in one line, for example: "A missing fault path means a failed Case update leaves
the Work Order half-moved; see `salesforce-flow-reviewer` for the rest of the Flow."
Never turn the response into four audits.

## Testing plan

After changing this skill, run these fixtures with the prompt "Audit the supplied
MillTrack workflow change."

**Fixture 1: architecture-breaking duplicate workflow.** A new WorkOrder
record-triggered Flow that enters on standard `Status`, moves `Temporary Fix` to
`Assigned`, sets the technician, and skips manager follow-up.
Pass if it flags standard Status instead of `Dispatch_Status__c`, flags `Temporary Fix`
→ `Assigned` bypassing `managerFollowUpWorkOrderWizard`, flags overlap with the
existing assignment automations, names the current owners, proposes the smallest fix,
and doesn't spend the response on generic Flow style.

**Fixture 2: cross-record inconsistency.** Completion code that sets
`Dispatch_Status__c = 'Completed'` but leaves Case `Waiting on Parts`, Asset
`Temporary Fix Applied`, and creates no resolved Maintenance Log with Root Cause.
Pass if it identifies the `In Progress` → `Completed` transition, flags the Case and
Asset inconsistency, flags the missing completion service history and the Root Cause
requirement, and recommends aligning the current completion owner
(`TechnicianCompleteWorkOrderController`) instead of adding a second automation.

**Fixture 3: clean implementation.** A 3-day scheduled reminder for Work Orders still
in `Temporary Fix` that changes no state, uses `Dispatch_Status__c`, re-checks the
status when the path runs, and leaves follow-up to the existing manager workflow.
Pass if it treats the change as a reminder only, keeps the existing follow-up as owner,
doesn't invent a duplicate-workflow finding, doesn't demand a new field or status,
returns no Urgent or Warning findings, and stays short.
