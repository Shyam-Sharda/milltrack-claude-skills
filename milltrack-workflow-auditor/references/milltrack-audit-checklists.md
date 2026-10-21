# MillTrack Audit Checklists

These are reasoning aids. Pick the lists that match the change and work through them
silently. Report only the items that turn into real findings. Never print a checklist.

## Contents
1. Every audit (always)
2. New or changed manager workflow
3. Technician workflow
4. Customer workflow
5. Parts transition
6. Temporary Fix
7. Completion
8. Time-based automation
9. New status, field or schema proposal
10. Duplicate automation and legacy-path detection

---

## 1. Every audit

- [ ] What business step is this, and which persona does it belong to?
- [ ] Who owns that step today (ownership reference)? Does the change extend that
      owner?
- [ ] Is `Dispatch_Status__c` the driver, with exact API values? Is standard `Status`
      used for logic anywhere?
- [ ] Start → target state: is it on the transition table? Is the start enforced
      server-side?
- [ ] After it runs, do the WO, Case, Asset, parts and log match the stage's expected
      state?
- [ ] Is there already another writer of the same transition?
- [ ] Is any required metadata absent from what was supplied? Use Cannot Verify only
      if it blocks the audit.
- [ ] Is the response proportional to the change? A clean change gets a short "no
      meaningful issues".

## 2. New or changed manager workflow

- [ ] Launched from `managerOperationsCenter` (a queue row and `actionName`), not a new
      dashboard or a standalone record-page action
- [ ] New manager queue or stage added in `ManagerOperationsCenterController`, not in
      a second controller
- [ ] Apex validates the starting `Dispatch_Status__c`. Hidden buttons aren't enough.
      Note that `approveWorkOrder` already lacks this check.
- [ ] Writes are USER_MODE and the class is granted in `MillTrack_Service_Manager`
- [ ] Case.Status is aligned in the same transaction. The Asset is left alone unless
      the owner for that stage writes it.
- [ ] `Service_Manager__c` is set where the owner sets it (review, follow-up)
- [ ] Not reachable by technicians or customers

## 3. Technician workflow

- [ ] A WorkOrder `ScreenAction` LWC with a `Technician*Controller`; not an extension
      of the legacy `Start_Work`, `Request_Parts`, `Complete_Work_Order`,
      `Temporary_Fix_From_Work_Order` or `Resume_Work`
- [ ] Server checks the starting state **and** `OwnerId == UserInfo.getUserId()`,
      re-queried `FOR UPDATE`
- [ ] SYSTEM_MODE writes only after those checks. Technician FLS isn't widened to
      make it work.
- [ ] Visit outcomes return the WO to the right owner: the queue for parts and
      Temporary Fix, `Service_Manager__c` for completion. Technician and visit fields
      are cleared where the owner clears them.
- [ ] Labor accumulates on the WO and is visit-specific on the log; notes are
      appended, never overwritten
- [ ] No manager decisions: dispatching, approving follow-up, ordering or receiving
      parts
- [ ] `Technician_Protect_*` validation rules aren't tripped by updates to existing
      records

## 4. Customer workflow

- [ ] Lives in Experience Cloud (`customer*` LWC plus a `Customer*Controller`) or a
      `CUST_*` Customer AI action
- [ ] Account and Contact derived server-side from the running user. Client-supplied
      Account, Contact or Asset Ids are never trusted for authorization.
- [ ] Queries filtered by that Account; `with sharing` / USER_MODE
- [ ] Customer never sets `Dispatch_Status__c` or picks a workflow stage
- [ ] Any customer write follows the `CreateAuthorizedCaseComment` pattern (re-verify
      ownership in code, refuse closed Cases). WorkOrder Edit isn't widened for
      customers.
- [ ] Customer-visible wording comes from `buildProgress` / `getCustomerCaseStatus`,
      not from new status values
- [ ] No manager or technician controls or data exposed

## 5. Parts transition

- [ ] Uses `Order_Status__c` values exactly; outstanding = Requested/Ordered/Shipped
- [ ] Requested → Ordered goes through `ManagerOrderPartsController`; → Received goes
      through `ManagerReceivePartsController`
- [ ] No move to `Ready_to_Assign` or `Assigned` while outstanding parts remain
- [ ] Waiting on Parts is released only when nothing is outstanding. A partial receipt
      leaves the stage unchanged.
- [ ] Receiving doesn't change the Asset (that was the legacy behavior)
- [ ] No duplicate active request for the same catalog part on one WO
- [ ] Technician requests use an active catalog part matching the Asset's manufacturer
- [ ] Case.Status aligned: `Waiting on Parts` while waiting, `In Progress` once
      released

## 6. Temporary Fix

- [ ] Starts only from `In Progress`, done by the owning technician
- [ ] Ends at `Temporary Fix` (exact spelling), with Asset `Temporary Fix Applied`,
      Case `Follow-Up Required`, an unresolved `Temporary Repair` log with follow-up
      true, owner `Service_Manager_Queue`, and technician and visit cleared
- [ ] The next step is `managerFollowUpWorkOrderWizard` on the same WO. No path to
      `Assigned` or `In Progress` without follow-up, and no second active WO.
- [ ] Notes not blank (validation)
- [ ] An extra immediate manager notification duplicates the existing one. A later
      reminder doesn't.

## 7. Completion

- [ ] Starts only from `In Progress`, done by the owning technician; AssetId and
      `Service_Manager__c` present
- [ ] One `Repair` log: resolved true, follow-up false, Root Cause non-blank, summary,
      visit labor, `Performed_By__c`
- [ ] Asset → `Active`; Case → `Closed`; WO owner → `Service_Manager__c`; technician
      cleared; `Work_Ended_Date_Time__c` set; notes appended
- [ ] All of it in one transaction with a savepoint. The WO isn't finalized before
      the evidence exists.
- [ ] Implemented in `TechnicianCompleteWorkOrderController`, not in a parallel
      trigger, Flow or the legacy `Complete_Work_Order` flow

## 8. Time-based automation

- [ ] Recognized as MillTrack's first scheduled automation. That's new capability,
      not a defect.
- [ ] Entry and run-time conditions use `Dispatch_Status__c`, and the status is
      **re-checked when the scheduled path runs**
- [ ] Timing comes from existing evidence: the scheduled-path trigger time, the
      Temporary Repair log `CreatedDate`, or `Maintenance_Date__c`. Don't demand a new
      timestamp field unless none of those fits.
- [ ] Reminder or notification only: it changes no workflow state, and doesn't
      reassign, re-dispatch or create a second follow-up process
- [ ] If it does change state, it goes through the transition's owner and doesn't add
      a parallel writer
- [ ] Recipient and notification type exist in the source or are explicitly proposed.
      A reused CustomNotificationType's metadata wasn't supplied, so note it as Cannot
      Verify only if the audit depends on it.
- [ ] Scheduled paths run in system context. Don't invent persona permission changes
      for that.

## 9. New status, field or schema proposal

- [ ] New `Dispatch_Status__c` value: needs explicit design approval. List its impact
      on the Operations Center queues, technician home filters, the calendar, customer
      progress, list views, Flows and Apex.
- [ ] A parallel status field next to `Dispatch_Status__c` is drift. Flag it.
- [ ] New field: could an existing field or record (log, WOP, `LastModifiedDate`,
      notes) express it?
- [ ] New picklist value on Case.Status, `Machine_Status__c` or `Order_Status__c`: is
      every reader of that field updated?
- [ ] Names referenced but not defined in the supplied metadata → Cannot Verify. Never
      assume they're deployed.

## 10. Duplicate automation and legacy-path detection

- [ ] Search for other writers of the same transition: Apex controllers,
      record-triggered Flows, Screen Flows, LWCs, Quick Actions
- [ ] Assignment already has three writers. A new one is always a finding.
- [ ] Does the change edit a legacy flow (ownership reference §6) while the LWC/Apex
      owner holds that behavior? Flag it and point the fix at the current owner. Don't
      recommend editing both.
- [ ] Does the new path re-trigger an existing record-triggered Flow?
      `Work_Order_Assignment_Sync` fires on any technician change while
      `Ready_to_Assign` or `Assigned`.
- [ ] Reading the same records isn't duplication. Only flag duplication that can cause
      conflicting or redundant writes.
- [ ] Unrelated legacy drift that the change doesn't touch: leave it out.
