# MillTrack Claude Skills

A collection of Claude Skills built specifically for **MillTrack Intelligence**, a Salesforce service-management application for dental CAD/CAM milling systems.

These skills give Claude application-specific knowledge of MillTrack's Salesforce architecture, service lifecycle, data model, workflow ownership, and Customer / Service Manager / Field Technician responsibilities.

Rather than relying on generic Salesforce prompts, each skill is designed to understand how MillTrack actually works and make decisions that fit the existing application.

## About MillTrack Intelligence

MillTrack Intelligence is a custom Salesforce service-management application designed for organizations that support dental CAD/CAM milling equipment.

The application coordinates the full service lifecycle across three primary personas:

```text
Customer
   ↓
Service Request / Case
   ↓
Service Manager Review
   ↓
Work Order / Parts / Dispatch
   ↓
Field Technician
   ↓
Repair / Temporary Fix / Follow-Up
   ↓
Service History
```

MillTrack is built using technologies including:

- Apex
- Lightning Web Components
- Salesforce Flow
- Experience Cloud
- Agentforce
- Prompt Builder
- Salesforce security and sharing
- custom service-management objects and metadata

A core architectural rule throughout MillTrack is that:

```text
WorkOrder.Dispatch_Status__c
```

is the canonical service-workflow status.

Standard `WorkOrder.Status` is not used to represent MillTrack's business lifecycle.

---

# Skills

## 1. MillTrack Workflow Builder

Designs new MillTrack workflows and feature changes while preserving the existing application architecture.

The skill first determines which existing component already owns the requested behavior and then recommends the smallest appropriate change.

It understands:

- Customer, Service Manager and Field Technician responsibilities
- the MillTrack Work Order lifecycle
- `Dispatch_Status__c` transitions
- existing manager and technician wizards
- LWC / Apex / Flow ownership
- Case and Asset side effects
- parts workflows
- Temporary Fix and follow-up behavior
- Experience Cloud customer workflows
- current versus legacy automation
- when a requested feature already exists

Example:

```text
/milltrack-workflow-builder

When a technician applies a temporary fix, create a manager follow-up after 3 days.
```

Instead of creating a second follow-up process, the skill recognizes that MillTrack already has a Temporary Fix → Manager Follow-Up workflow and determines what additional reminder behavior is actually required.

Folder:

```text
milltrack-workflow-builder/
```

---

## 2. MillTrack Service Case Generator

Generates realistic, internally consistent synthetic MillTrack service data for development, QA, demonstrations and test fixtures.

It can generate scenarios at different stages of the service lifecycle while keeping related Salesforce records consistent.

Supported scenarios can include:

- new customer service requests
- Pending Work Orders
- parts-required scenarios
- Ready to Assign Work Orders
- dispatched technician visits
- Waiting on Parts
- Temporary Fix scenarios
- completed repairs
- historical service records
- multi-case demo datasets

The skill understands relationships between:

```text
Case
WorkOrder
Asset
Maintenance_Log__c
Work_Order_Part__c
Part_Catalog__c
```

It also distinguishes between:

**Live automation**

```text
New Case
→ existing MillTrack automation creates the Pending Work Order
```

and:

**Isolated / staged fixtures**

```text
Case
+ WorkOrder
+ Asset state
+ Maintenance Logs
+ Parts
```

This prevents generated seed data from accidentally duplicating records that MillTrack's live automation already creates.

The generator also avoids:

- fabricated Salesforce record IDs
- invented error codes
- unsupported manufacturer repair procedures
- invalid `Dispatch_Status__c` values
- inconsistent Case / Work Order / Asset states
- unsupported cancellation behavior
- treating synthetic service history as real customer data

Example:

```text
/milltrack-service-case-generator

Generate 6 synthetic MillTrack demo cases:
one Pending, one Ready to Assign, one Assigned,
one Waiting on Parts, one Temporary Fix and one Completed.
```

Folder:

```text
milltrack-service-case-generator/
```

---

## 3. MillTrack Workflow Auditor

**Planned**

The Workflow Auditor will review implemented MillTrack changes and determine whether they correctly fit the application's architecture.

Its primary question will be:

> Does this implementation correctly belong in MillTrack?

It will check areas such as:

- workflow ownership
- valid `Dispatch_Status__c` transitions
- duplicate automation
- current versus legacy implementation paths
- Case / Work Order / Asset consistency
- parts lifecycle
- Maintenance Log requirements
- Customer / Manager / Technician boundaries
- architecture-breaking state transitions

This complements the Workflow Builder:

```text
Workflow Builder
→ designs the change

Workflow Auditor
→ verifies the implementation
```

Planned folder:

```text
milltrack-workflow-auditor/
```

---

# Repository Structure

```text
milltrack-claude-skills/
│
├── milltrack-workflow-builder/
│   ├── SKILL.md
│   └── references/
│
├── milltrack-service-case-generator/
│   ├── SKILL.md
│   └── references/
│
└── milltrack-workflow-auditor/       # planned
    ├── SKILL.md
    └── references/
```

Each skill is self-contained.

`SKILL.md` defines the skill's purpose, workflow, reasoning rules, scope and expected output.

`references/` contains focused MillTrack architecture and data-model knowledge derived from the Salesforce source.

---

# Design Principles

The MillTrack skills follow several shared principles.

### Understand the existing application first

Before proposing or generating anything, identify the existing workflow owner and relevant Salesforce components.

### Use the existing architecture

Extend existing LWCs, Apex controllers and Flows rather than creating parallel implementations.

### Preserve the workflow state model

`WorkOrder.Dispatch_Status__c` remains the canonical MillTrack workflow status.

### Make the smallest correct change

Do not redesign an entire service process to satisfy a small requirement.

### Keep personas separate

Customer, Service Manager and Field Technician responsibilities remain distinct.

### Do not invent metadata

Fields, statuses, objects, classes, Flows, permissions and relationships must come from supplied source or be clearly identified as proposed new metadata.

### Keep related records consistent

A valid MillTrack state often requires multiple records to agree:

```text
WorkOrder
Case
Asset
Maintenance Log
Parts
```

### Clean results are valid

The skills should recognize when MillTrack already performs the requested behavior or when an implementation has no meaningful issue.

---

# Relationship to the General Salesforce Skills

These MillTrack-specific skills are intentionally separate from my general Salesforce Claude Skills.

The general-purpose toolkit handles tasks such as:

```text
Apex code review
Flow review
Apex test generation
Debug log analysis
Salesforce security auditing
```

The MillTrack skills operate at the application-architecture level and use knowledge of the actual MillTrack service lifecycle.

For example:

```text
"Is this Apex technically correct?"
→ Salesforce Apex Code Reviewer

"Does this Apex implement the correct MillTrack workflow?"
→ MillTrack Workflow Auditor
```

---

# Testing

Each skill is tested against deliberately different scenarios.

Tests include:

- valid implementations
- deliberately incorrect lifecycle states
- missing dependencies
- duplicate workflow risks
- existing behavior that should not be rebuilt
- synthetic service datasets at multiple lifecycle stages

A skill is expected to recognize both genuine problems and clean implementations.

---

# Project Goal

The goal of this project is to explore how application-specific Claude Skills can provide a reusable AI development layer on top of an existing Salesforce application.

Instead of repeatedly explaining MillTrack's architecture in prompts, the skills capture its durable rules so Claude can reason consistently about:

```text
Architecture
Workflow design
Service data
Lifecycle state
Implementation ownership
```

This repository focuses specifically on **MillTrack-aware development workflows**.
