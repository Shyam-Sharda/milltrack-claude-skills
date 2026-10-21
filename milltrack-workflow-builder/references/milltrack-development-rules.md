# MillTrack Development Rules

Conventions observed in the supplied source. Follow them for new or changed files so the
result looks like the rest of MillTrack.

## Contents
1. Naming
2. Apex controller pattern
3. Security mode pattern by persona
4. LWC patterns
5. Flow conventions
6. Choosing Flow vs Apex vs LWC in MillTrack
7. Smallest-change checklist
8. Testing conventions

---

## 1. Naming

| Artifact | Convention | Examples |
|---|---|---|
| LWC | `<persona><Thing>[Wizard]` camelCase; persona prefix `customer`, `manager`, `technician`, `admin`, shared `millTrack` | `technicianTemporaryFixWizard`, `managerReceivePartsWizard`, `customerWorkOrderDetails` |
| Apex controller for an LWC | `<Persona><Thing>Controller` | `TechnicianTemporaryFixController`, `ManagerFollowUpWorkOrderController` |
| Invocable Apex (Flow/agent action) | Verb-first, no suffix | `GetServiceManagerQueue`, `CreateAuthorizedCaseComment`, `ParseWorkOrderProposalFields` |
| Newer Flows | Prefix by audience: `CUST_0n_`, `MGR_0n_`, `SYS_0n_` + Title_Case description | `CUST_06_Resolve_Authorized_Customer_Case`, `SYS_01_Set_Asset_Service_Status` |
| Older Flows | Plain Title_Case (`Start_Work`, `Request_Parts`) — don't copy for new flows |
| Quick actions | `WorkOrder.<Action>_LWC` for LWC screen actions | `WorkOrder.Complete_Work_Order_LWC` |
| Status constants | `private static final String STATUS_…`/`REQUIRED_…` at class top | `STATUS_READY_TO_ASSIGN = 'Ready_to_Assign'` |
| Custom notification types | `DeveloperName` constants, e.g. `Temporary_Repair_FollowUp`, `Parts_Request` | |

Status values are compared against picklist **API names** (`Ready_to_Assign`, not "Ready to Assign").

## 2. Apex controller pattern

Copy the shape of `TechnicianTemporaryFixController` / `ManagerFollowUpWorkOrderController`:
- `public with sharing class`. (Only `CreateAuthorizedCaseComment` is `without sharing`, and
  it re-verifies ownership manually.)
- Read method: `@AuraEnabled(cacheable=true) get<Thing>Context|Data(Id workOrderId)` returning
  an inner DTO class with `@AuraEnabled public … { get; set; }` properties.
- Write method: `@AuraEnabled submit…/approve…/receive…(Id workOrderId, …)` that:
  1. null-checks and trims inputs; throws on blanks,
  2. re-queries the record (`FOR UPDATE` on technician actions),
  3. validates starting `Dispatch_Status__c` (and owner for technician actions),
  4. `Savepoint` → DML for Maintenance Log / Asset / WorkOrder / Case together → rollback on error,
  5. returns a `SubmitResult`/`ApprovalResult` DTO with `success`, `message`, optional `warningMessage`.
- Errors: a private `handledException(String)` that builds `AuraHandledException` and calls
  `setMessage` so the LWC sees the text. User-facing, plain-English messages.
- Notifications: `Messaging.CustomNotification` looked up by `CustomNotificationType.DeveloperName`,
  wrapped in try/catch; failure becomes `warningMessage`, never a rollback.
- Queue lookup: `Group` where `Type = 'Queue' AND DeveloperName = 'Service_Manager_Queue'`.
- Notes are appended, never overwritten (`existing + '\n\n' + new`). Labor hours accumulate.
- Keep Case.Status (and Asset status where relevant) aligned in the same transaction.

## 3. Security mode by persona

| Persona | Pattern |
|---|---|
| Manager | Queries `WITH USER_MODE`; DML `Database.update(x, AccessLevel.USER_MODE)`. Relies on `MillTrack_Service_Manager` FLS. Add FLS there only for fields that the requested feature makes managers read or write |
| Technician | Validate status + `OwnerId == UserInfo.getUserId()` in code, then write with `AccessLevel.SYSTEM_MODE` (technicians lack FLS edit on `Dispatch_Status__c`). Don't widen technician FLS to make a feature work |
| Customer | `with sharing` + derive Contact/Account from `UserInfo` → `User.ContactId` → `Contact.AccountId`; filter by Account; never trust a client-supplied account/contact. Customers have Read only on WorkOrder (sharing set + permission set), so any customer write needs an explicit server-side ownership check and a deliberate access decision. Don't widen customer WorkOrder Edit access just to support one controlled write |
| AI agents | Read through flows/invocables; writes only via the dedicated `CUST_*` actions |

The supplied permission sets don't establish Apex class access for several existing
Technician and Customer controllers. Most `Technician*Controller` / `Customer*Controller`
classes are not listed in `MillTrack_Technician` or `MillTrack_Customer_Portal_Access`.
The granting source was not supplied and could be a Profile or another assigned
permission source. Use **Cannot Verify** rather than assuming which security source
grants the class access.

When proposing a new controller:
- say that the intended persona needs Apex class access to it;
- modify a supplied permission source only when the architecture or evidence establishes
  that it is the correct source (for example, every `Manager*Controller` is granted in
  `MillTrack_Service_Manager`);
- otherwise use Cannot Verify.
Don't run a full security audit.

## 4. LWC patterns

- Manager wizards: `isExposed=false`, `@api recordId`, hosted in `managerOperationsCenter`'s
  modal sections; fire `CustomEvent('approved'|'complete'|'cancel')`; show `ShowToastEvent`.
  The host refreshes with `refreshApex(this.wiredOperationsResult)`.
- Technician wizards: `isExposed=true`, target `lightning__RecordAction` (ScreenAction),
  `@api recordId`, close with `CloseActionScreenEvent`, may navigate to the created
  Maintenance Log with `NavigationMixin`.
- Customer components: targets `lightningCommunity__Page`/`Default`; read `recordId` from
  `@api` or `CurrentPageReference`; images from static resource `MillTrack_Machine_Media`.
- Imperative Apex for writes; cacheable Apex for reads. SLDS + `lightning-*` base components.
- Every LWC folder has a `__tests__/<name>.test.js` (currently CLI-generated placeholders).

## 5. Flow conventions

- Record-triggered flows used today are after-save on Case create and WorkOrder update.
  Entry criteria use `Dispatch_Status__c`, never standard `Status`.
- Fault paths are rare in the older flows; new flows should have them.
- Autolaunched flows that serve AI return `success` / `statusCode` / `statusMessage` outputs.
- Don't add logic to legacy Screen Flows (`Start_Work`, `Request_Parts`, `Complete_Work_Order`,
  `Temporary_Fix_From_Work_Order`, `Resume_Work`); the LWC/Apex versions own those steps.

## 6. Choosing Flow vs Apex vs LWC — in MillTrack terms

- **A step a persona clicks through** → extend the existing wizard LWC + its controller.
  Only create a new wizard when no existing wizard owns that `Dispatch_Status__c` step.
- **A state transition** → the Apex controller that already owns the neighbouring
  transition. Don't add a record-triggered flow that also writes `Dispatch_Status__c`
  for a transition Apex already performs.
- **Time-based follow-up / reminder** → record-triggered Flow with a scheduled path (or a
  scheduled flow) is the natural fit, since MillTrack has no async Apex; it would be the
  first scheduled automation, so call that out. Re-check `Dispatch_Status__c` when the path runs.
  Scheduled paths execute in system context. Don't infer end-user CRUD/FLS requirements from
  that alone, and don't describe the running-user association as the Automated Process user
  unless the supplied implementation establishes that. Plan for whatever metadata or access
  the automation itself needs, not for a persona permission change.
- **Notification only** → reuse an existing CustomNotificationType where it fits
  (`Temporary_Repair_FollowUp`, `Parts_Request`, …); new type = new metadata. The source
  expects these types, but their metadata wasn't supplied, so ask the user to confirm the
  one you reuse is deployed.
- **Customer-visible wording** → `CustomerWorkOrderDetailsController.buildProgress` /
  `getCustomerCaseStatus`, not new status values.
- **Complex validation, multi-object transactional writes, ownership checks** → Apex.
- **Simple field update on a record event** with no existing owner → record-triggered Flow.

## 7. Smallest-change checklist

1. Is the behaviour already there? (Receiving all parts already returns the WO to
   `Ready_to_Assign`; temp fix already notifies the manager and lands in Follow-Up Required.)
2. Which component owns the starting status? Extend it.
3. Can existing fields express it? (Temp-fix time = Temporary Repair Maintenance Log; "unresolved" = still `Temporary Fix`.)
4. Does a new value belong in `Dispatch_Status__c`? Almost never — adding a value touches
   the Operations Center queues, technician home filters, calendar, customer progress map
   and list views. Get explicit approval first.
5. Which persona's access changes? Name the permission set / FLS / class access, or Cannot Verify.
   Every access change must be tied to a concrete behavior in the requested feature; a persona
   with no new read/write behavior gets no new permissions.
6. Leave unrelated code, legacy flows and known drift alone unless asked; mention them if they interfere.

## 8. Testing conventions

- Apex test classes exist in the full repo but were intentionally not supplied (for example,
  `ParseWorkOrderProposalFieldsTest` is referenced by `MillTrack_Admin`). Existing production
  controllers already have test classes.
  - **New Apex class:** generate a matching new `*Test` class covering the allowed starting
    status, a wrong status, a wrong owner (technician actions), and the Case/Asset side effects.
  - **Existing class modified, test class supplied:** update that test class as needed.
  - **Existing class modified, test class not supplied:** don't invent, replace, or duplicate
    a test class. Write **Cannot Verify:** the existing tests that cover `[ClassName]` were not
    supplied. The matching test class should be inspected and updated for this change.
    When the test class's name is evidenced, list it as the file likely to change, but don't
    make up its contents.
  - If the user asks for the tests, recommend `salesforce-apex-test-generator`, given both the
    production class and the existing test class.
- LWC Jest files exist per component; update the matching `__tests__` file when changing an LWC.
