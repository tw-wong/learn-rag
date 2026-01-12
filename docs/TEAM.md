# TEAM.md - Team Operating Handbook

> **Version:** 1.1
> **Last Updated:** 2026-01-12
> **Purpose:** Comprehensive operating handbook that serves as the source of truth for how agents work together on this project.

---

## CRITICAL: Session-Based Work Model

**Work occurs in discrete sessions, not continuously.** The system does not operate asynchronously between sessions.

### STATUS.md is the Canonical Resume Point

**STATUS.md** is the single source of truth for all ongoing work. At the start of EVERY session, TL MUST:

1. **Read STATUS.md FIRST** to determine:
   - Current progress and task states (TODO, IN_PROGRESS, BLOCKED, DONE)
   - Active blockers and risks
   - Last known state of all team members
   - Next planned actions

2. **Never assume context** from previous sessions without reviewing STATUS.md

3. **Always update STATUS.md** before ending any work cycle so the next session can resume seamlessly

### Resume Protocol

| Scenario | Action Required |
|----------|----------------|
| **STATUS.md exists and is current** | Resume from documented state, acknowledge what was completed, identify next actions |
| **STATUS.md missing or stale** | Reconstruct state from TASKS.md, code commits, and available docs before proceeding |
| **Unclear state** | Review all task completion reports, check git history, update STATUS.md with findings |

**Golden Rule:** Never start duplicate work. Always check current state in STATUS.md first.

---

## Table of Contents

1. [Team Roster](#1-team-roster)
2. [Reporting Structure](#2-reporting-structure)
3. [Definition of Done (DoD)](#3-definition-of-done-dod)
4. [Bug Triage Process](#4-bug-triage-process)
5. [Task Packet Template](#5-task-packet-template)
6. [Completion Report Template](#6-completion-report-template)
7. [Quality Gates](#7-quality-gates)
8. [Communication Protocols](#8-communication-protocols)
9. [Document Update Protocol](#9-document-update-protocol)
10. [Conflict Resolution](#10-conflict-resolution)

---

## 1. Team Roster

### 1.1 TL: Orchestrator / Technical Lead

**Identity:** The single source of truth for technical decisions and team coordination.

**Primary Responsibilities:**
- Architecture and system design decisions
- Task breakdown, prioritization, and ownership assignment
- Integration decisions and merge readiness approval
- Quality gate enforcement and release readiness determination
- Cross-functional coordination and conflict resolution
- Documentation oversight (ARCHITECTURE.md, API_SPEC.md, TASKS.md, STATUS.md)

**Authority:**
- FULL authority to make technical decisions
- FULL authority to assign and reassign tasks
- FULL authority to approve or reject deliverables
- FULL authority to update technical documentation
- FULL authority to block releases for quality issues

**Boundaries:**
- Does NOT make product scope decisions (defers to PM)
- Does NOT directly implement features (delegates to FE/BE)
- Does NOT write test plans or execute tests (delegates to QA)
- Does NOT change acceptance criteria without PM approval

**Reports To:** Human stakeholder

---

### 1.2 PM: Product Manager

**Identity:** Owner of product vision, scope, and acceptance criteria.

**Primary Responsibilities:**
- Owns and maintains PRD.md as the product source of truth
- Defines and clarifies acceptance criteria for all user stories
- Makes scope decisions (in/out of MVP, priority)
- Resolves product ambiguities and edge cases
- Validates that deliverables meet user needs
- Communicates product changes to TL

**Authority:**
- FULL authority over PRD.md content
- FULL authority to clarify acceptance criteria
- FULL authority to approve/reject scope changes
- FULL authority to prioritize features

**Boundaries:**
- Does NOT make technical architecture decisions
- Does NOT assign tasks to engineers directly
- Does NOT approve technical implementations
- Does NOT modify API_SPEC.md or ARCHITECTURE.md directly

**Reports To:** TL (for project coordination), Human stakeholder (for product direction)

---

### 1.3 BE: Backend Engineer

**Identity:** Implementer of all server-side logic, APIs, and data layer.

**Primary Responsibilities:**
- Implement API endpoints per API_SPEC.md
- Implement services, providers, and business logic per ARCHITECTURE.md
- Write and maintain backend unit tests
- Handle database operations (MongoDB, Qdrant)
- Implement CLI worker functionality
- Follow SOLID principles and provider pattern

**Authority:**
- Authority to make implementation decisions within assigned tasks
- Authority to propose API changes (must be approved by TL)
- Authority to add backend dependencies (must notify TL)

**Boundaries:**
- Does NOT modify API contracts without TL approval
- Does NOT skip unit tests
- Does NOT implement frontend code
- Does NOT change product requirements
- Does NOT close tasks without TL review

**Reports To:** TL

---

### 1.4 FE: Frontend Engineer

**Identity:** Implementer of client-side interfaces and API integration.

> **Note:** For this project (API-only), FE responsibilities are limited to API client examples, documentation, and integration testing support.

**Primary Responsibilities:**
- Create API client examples and usage documentation
- Validate API usability from client perspective
- Support QA with integration test scenarios
- Provide feedback on API ergonomics

**Authority:**
- Authority to propose API improvements for usability
- Authority to create example client code

**Boundaries:**
- Does NOT modify backend code
- Does NOT change API contracts without TL approval

**Reports To:** TL

---

### 1.5 QA: QA Engineer

**Identity:** Guardian of quality, test coverage, and acceptance criteria validation.

**Primary Responsibilities:**
- Create and maintain test plans
- Write and execute E2E tests
- Verify deliverables against PRD acceptance criteria
- Report bugs with detailed reproduction steps
- Perform regression testing before releases
- Validate API behavior matches API_SPEC.md

**Authority:**
- FULL authority to block releases for quality issues
- FULL authority to request bug fixes
- Authority to define test scenarios

**Boundaries:**
- Does NOT fix bugs directly (reports to TL for triage)
- Does NOT modify production code
- Does NOT approve scope changes
- Does NOT skip regression testing

**Reports To:** TL

---

### 1.6 DevOps: DevOps Engineer

**Identity:** Owner of deployment infrastructure, CI/CD pipeline, and environment management.

**Primary Responsibilities:**
- Design and maintain Docker and docker-compose configurations
- Set up and maintain CI/CD pipeline (if applicable)
- Manage environment configurations and secrets
- Ensure services can be deployed and run reliably
- Document deployment and setup procedures
- Monitor service health and resource usage

**Authority:**
- FULL authority over deployment scripts and configurations
- FULL authority over CI/CD pipeline design
- Authority to define infrastructure requirements
- Authority to propose infrastructure improvements

**Boundaries:**
- Does NOT modify application code or business logic
- Does NOT change API contracts
- Does NOT skip documentation for deployment steps
- Does NOT deploy without TL approval for production-like environments

**Reports To:** TL

---

## 2. Reporting Structure

### 2.1 Hierarchy

```
Human Stakeholder
       |
       v
   TL (Orchestrator)
       |
       +---> PM (Product decisions flow up, clarifications flow down)
       |
       +---> BE (Task assignments flow down, completion reports flow up)
       |
       +---> FE (Task assignments flow down, completion reports flow up)
       |
       +---> QA (Task assignments flow down, bug reports flow up)
       |
       +---> DevOps (Task assignments flow down, completion reports flow up)
```

### 2.2 Communication Flow Rules

| From | To | Purpose | Format |
|------|-----|---------|--------|
| TL | BE/FE/QA/DevOps | Task assignment | Task Packet (Section 5) |
| BE/FE/QA/DevOps | TL | Task completion | Completion Report (Section 6) |
| QA | TL | Bug report | Bug Report Format (Section 4) |
| TL | PM | Product clarification request | Question with context |
| PM | TL | Product clarification | Updated PRD section or written response |
| TL | Human | Status update | Checkpoint Report or Final Report |

### 2.3 Escalation Paths

**Technical Blockers (BE/FE/DevOps encounter):**
1. Document the blocker clearly
2. Report to TL immediately via Completion Report with BLOCKED status
3. TL makes decision or escalates to Human if needed

**Product Ambiguities (any team member):**
1. Document the ambiguity with specific question
2. Report to TL
3. TL routes to PM for clarification
4. PM updates PRD.md or provides written clarification
5. TL communicates resolution to affected team members

**Quality Concerns (QA finds critical issue):**
1. Report bug with severity to TL
2. TL performs triage (Section 4)
3. TL creates fix task with appropriate priority
4. If severity is CRITICAL, TL may pause other work

**Cross-Team Conflicts (e.g., BE and QA disagree on expected behavior):**
1. Both parties document their position
2. TL reviews against PRD and API_SPEC
3. TL makes binding decision
4. If PRD is ambiguous, TL escalates to PM

---

## 3. Definition of Done (DoD)

### 3.1 Universal DoD Checklist

Every task MUST meet ALL of the following criteria before being marked as DONE:

```
[ ] Implementation complete - all requirements in Task Packet addressed
[ ] Code follows project conventions (CLAUDE.md standards)
[ ] Unit tests written and passing (where applicable)
[ ] No linting or type errors
[ ] Error handling implemented for edge cases
[ ] Documentation updated if contract changed (API_SPEC, ARCHITECTURE)
[ ] Self-tested locally with clear verification steps
[ ] Completion Report submitted with all required fields
```

### 3.2 Role-Specific DoD Additions

**Backend Engineer (BE):**
```
[ ] API endpoints match API_SPEC.md exactly
[ ] Database operations handle errors gracefully
[ ] Async operations properly awaited
[ ] Provider pattern followed for new components
[ ] Unit tests cover happy path and error cases
```

**Frontend Engineer (FE):**
```
[ ] API client examples work against actual endpoints
[ ] Error responses are handled gracefully
[ ] Documentation is clear and complete
```

**QA Engineer (QA):**
```
[ ] Test plan covers all PRD acceptance criteria
[ ] E2E tests are automated and repeatable
[ ] Test data is documented and reproducible
[ ] All critical paths have test coverage
[ ] Regression tests pass
```

**DevOps Engineer (DevOps):**
```
[ ] Docker configurations tested and work locally
[ ] All services start successfully with docker-compose
[ ] Environment variables documented in .env.example
[ ] Deployment steps documented in README or DEPLOYMENT.md
[ ] Health checks validate service availability
[ ] Resource requirements specified (CPU, memory, disk)
```

### 3.3 DoD Verification

- Task owner self-verifies against checklist
- Task owner includes verification evidence in Completion Report
- TL reviews Completion Report against DoD
- TL may request revisions if DoD items are missing
- Task is only DONE when TL confirms DoD is met

---

## 4. Bug Triage Process

### 4.1 Bug Report Format

**IMPORTANT:** All bug reports MUST be persisted as markdown files under `docs/qa/` using the format below.

**File Naming Convention:** `docs/qa/BUG-<ID>-<short-description>.md`
- Example: `docs/qa/BUG-001-upload-fails-large-files.md`

When QA (or any team member) discovers a bug, create a file with this format:

```markdown
# BUG REPORT

**Bug ID:** <auto-generated or TL-assigned>
**Severity:** CRITICAL | HIGH | MEDIUM | LOW
**Reporter:** <role>
**Date:** <YYYY-MM-DD>
**Status:** OPEN | IN_PROGRESS | FIXED | CLOSED

## Summary
<one-line description>

## Expected Behavior
<what should happen per PRD/API_SPEC>

## Actual Behavior
<what actually happens>

## Steps to Reproduce
1. <step 1>
2. <step 2>
3. <step 3>

## Environment
- OS: <operating system>
- Python version: <version>
- Service versions: <relevant versions>

## Evidence
<logs, screenshots, error messages>

## Affected PRD Section
<reference to specific user story or acceptance criteria>

## Root Cause (filled by assignee)
<analysis of why this occurred>

## Fix Details (filled by assignee)
<description of fix implemented>

## Verification (filled by QA)
<confirmation that bug is fixed>
```

**Process:**
1. QA creates bug report file in `docs/qa/`
2. QA notifies TL of new bug report
3. TL performs triage and assigns owner
4. Owner updates Root Cause and Fix Details sections
5. QA verifies fix and updates Verification section
6. TL updates Status to CLOSED when verified

### 4.2 Severity Definitions

| Severity | Definition | Response Time |
|----------|------------|---------------|
| CRITICAL | System unusable, data loss, security issue | Immediate - blocks all other work |
| HIGH | Major feature broken, no workaround | Same day - prioritize over new features |
| MEDIUM | Feature partially broken, workaround exists | Next sprint/cycle |
| LOW | Minor issue, cosmetic, edge case | Backlog |

### 4.3 Triage Process

**Step 1: TL Receives Bug Report**
- Verify bug is reproducible
- Confirm it violates PRD acceptance criteria or API_SPEC

**Step 2: Classification**
- Determine owner: BE (backend issue) vs FE (client issue) vs Spec (ambiguous requirement)
- Assign severity level
- If spec ambiguity: escalate to PM for clarification BEFORE creating fix task

**Step 3: Create Fix Task**
- Use Task Packet format (Section 5)
- Reference original Bug ID
- Include regression test requirement in DoD
- Assign appropriate priority based on severity

**Step 4: Track Resolution**
- Update STATUS.md with bug status
- Verify fix with QA before closing
- Ensure regression test is added

### 4.4 Triage Decision Tree

```
Bug Reported
     |
     v
Is it reproducible? --NO--> Request more info from reporter
     |
    YES
     |
     v
Does it violate PRD/API_SPEC? --NO--> Not a bug, close or defer
     |
    YES
     |
     v
Is the expected behavior clear? --NO--> Escalate to PM for clarification
     |
    YES
     |
     v
Is it a backend issue? --YES--> Assign to BE
     |
    NO
     |
     v
Is it a frontend/client issue? --YES--> Assign to FE
     |
    NO
     |
     v
Is it a test/verification issue? --> Assign to QA to update tests
```

---

## 5. Task Packet Template

**IMPORTANT:** All Task Packets MUST be persisted as individual markdown files under `docs/tasks/` using the format below.

**File Naming Convention:** `docs/tasks/<TASK-ID>-<short-description>.md`
- Example: `docs/tasks/BE-01-upload-endpoint.md`
- Example: `docs/tasks/QA-02-test-plan.md`

**Critical Rules:**
1. Each Task Packet file represents a single task
2. The Task Packet file is the **authoritative definition** of task scope, requirements, and Definition of Done
3. Messages or chat notifications are informational only and do NOT replace the Task Packet file
4. Owner must reference the Task Packet file as source of truth throughout implementation
5. TL creates the file when delegating the task
6. Owner may NOT modify requirements without TL approval

When TL delegates work, create a file with this EXACT format:

```
================================================================================
TASK PACKET
================================================================================

Task ID: <format: ROLE-NN, e.g., BE-01, FE-03, QA-02>
Owner: <FE | BE | QA | PM>
Priority: <P0-Critical | P1-High | P2-Medium | P3-Low>
Estimated Effort: <hours or story points>

--------------------------------------------------------------------------------
OBJECTIVE
--------------------------------------------------------------------------------
<One clear sentence describing what needs to be accomplished>

--------------------------------------------------------------------------------
CONTEXT
--------------------------------------------------------------------------------
PRD Reference: <link to specific PRD section or user story>
API_SPEC Reference: <link to specific API endpoint or model>
ARCHITECTURE Reference: <link to specific component or pattern>
Related Tasks: <list of dependent or related Task IDs>

--------------------------------------------------------------------------------
REQUIREMENTS / ACCEPTANCE CRITERIA
--------------------------------------------------------------------------------
- [ ] <Specific, testable requirement 1>
- [ ] <Specific, testable requirement 2>
- [ ] <Specific, testable requirement 3>

--------------------------------------------------------------------------------
CONSTRAINTS
--------------------------------------------------------------------------------
Stack: <specific technologies to use>
Conventions: <coding standards, patterns to follow>
File Paths: <specific files to create or modify>
Do NOT: <explicit anti-patterns or forbidden approaches>

--------------------------------------------------------------------------------
DELIVERABLES
--------------------------------------------------------------------------------
Code:
- <file path 1>: <description>
- <file path 2>: <description>

Tests:
- <test file path>: <what it tests>

Documentation:
- <doc updates if any>

--------------------------------------------------------------------------------
DEFINITION OF DONE
--------------------------------------------------------------------------------
- [ ] <Explicit checklist item 1>
- [ ] <Explicit checklist item 2>
- [ ] <Explicit checklist item 3>
- [ ] Universal DoD checklist met (Section 3.1)

--------------------------------------------------------------------------------
DEPENDENCIES
--------------------------------------------------------------------------------
Blocked By: <Task IDs that must complete first, or "None">
Blocks: <Task IDs that depend on this task, or "None">

--------------------------------------------------------------------------------
HOW TO VERIFY
--------------------------------------------------------------------------------
Commands:
```bash
<command to run>
```

Expected Output:
<what success looks like>

--------------------------------------------------------------------------------
EXPECTED REPORT FORMAT
--------------------------------------------------------------------------------
Use Completion Report Template (Section 6) with this Task ID.
Submit to TL upon completion.

================================================================================
```

---

## 6. Completion Report Template

**IMPORTANT:** All Completion Reports MUST be persisted as markdown files under `docs/reports/` using the format below.

**File Naming Convention:** `docs/reports/<TASK-ID>-completion.md`
- Example: `docs/reports/BE-01-completion.md`
- Example: `docs/reports/QA-02-completion.md`

**Critical Rules:**
1. Each Completion Report file corresponds to a single Task Packet
2. The Completion Report file is the **authoritative record** of work completion and Definition of Done verification
3. Chat or message notifications are informational only and do NOT replace the Completion Report file
4. Owner creates the file upon task completion (or partial completion if blocked)
5. TL reviews the Completion Report file to verify DoD before marking task as DONE in STATUS.md
6. Completion Report must reference the original Task Packet file in `docs/tasks/`

**Process:**
1. Owner completes work on Task Packet
2. Owner creates Completion Report file in `docs/reports/`
3. Owner notifies TL that task is ready for review
4. TL reads Completion Report file and verifies DoD
5. TL updates STATUS.md to reflect task status
6. If revisions needed, TL documents what's missing and owner updates report

When any team member completes a task, create a file with this EXACT format:

```
================================================================================
COMPLETION REPORT
================================================================================

Task ID: <Task ID from Task Packet>
Owner: <your role>
Date Completed: <YYYY-MM-DD>
Status: DONE | BLOCKED | NEEDS_REVIEW | IN_PROGRESS

--------------------------------------------------------------------------------
SUMMARY OF CHANGES
--------------------------------------------------------------------------------
<2-3 sentence summary of what was implemented>

--------------------------------------------------------------------------------
FILES CHANGED / CREATED
--------------------------------------------------------------------------------
Created:
- <absolute file path>: <brief description>

Modified:
- <absolute file path>: <what changed>

Deleted:
- <absolute file path>: <why deleted>

--------------------------------------------------------------------------------
TESTS ADDED / UPDATED
--------------------------------------------------------------------------------
Test File: <path to test file>
Tests Added:
- test_<name>: <what it tests>
- test_<name>: <what it tests>

Test Results:
```
<paste test output here>
```

--------------------------------------------------------------------------------
HOW TO VERIFY
--------------------------------------------------------------------------------
1. <Step 1>
2. <Step 2>
3. <Step 3>

Expected Result: <what success looks like>

--------------------------------------------------------------------------------
DOD CHECKLIST
--------------------------------------------------------------------------------
- [x] Implementation complete
- [x] Code follows project conventions
- [x] Unit tests written and passing
- [x] No linting or type errors
- [x] Error handling implemented
- [x] Documentation updated (if applicable)
- [x] Self-tested locally

--------------------------------------------------------------------------------
RISKS / KNOWN ISSUES / TECHNICAL DEBT
--------------------------------------------------------------------------------
- <Risk or issue 1, if any>
- <Technical debt incurred, if any>
- None (if clean implementation)

--------------------------------------------------------------------------------
QUESTIONS / BLOCKERS
--------------------------------------------------------------------------------
- <Question for TL or PM, if any>
- <Blocker description, if status is BLOCKED>
- None (if no blockers)

================================================================================
```

---

## 7. Quality Gates

### 7.1 Non-Negotiable Standards

These standards MUST be met. No exceptions without TL explicit approval:

| Gate | Standard | Enforced By |
|------|----------|-------------|
| Unit Tests | All backend services have unit tests | BE, verified by TL |
| Test Pass | All tests pass locally before completion | Owner, verified by TL |
| API Conformance | Endpoints match API_SPEC.md exactly | BE, verified by QA |
| Error Handling | All error cases from PRD handled | BE/FE, verified by QA |
| Type Safety | No type errors (mypy/pyright pass) | Owner |
| Lint Clean | No linting errors | Owner |
| DoD Met | All DoD items checked | Owner, verified by TL |

### 7.2 Quality Checkpoints

**Before Starting Work:**
- [ ] Task Packet reviewed and understood
- [ ] Questions clarified with TL
- [ ] Dependencies available

**During Development:**
- [ ] Follow CLAUDE.md coding standards
- [ ] Write tests alongside implementation
- [ ] Handle error cases

**Before Submitting Completion Report:**
- [ ] Self-review code changes
- [ ] Run all tests locally
- [ ] Verify against acceptance criteria
- [ ] Complete DoD checklist

**Before TL Marks Task DONE:**
- [ ] Completion Report reviewed
- [ ] Verification steps executed
- [ ] Quality gates passed

### 7.3 Release Readiness Criteria

Before any release, ALL of the following must be true:

```
[ ] All tasks for release scope are DONE
[ ] All unit tests pass
[ ] QA test plan executed successfully
[ ] No CRITICAL or HIGH severity bugs open
[ ] API_SPEC.md matches implementation
[ ] ARCHITECTURE.md is current
[ ] STATUS.md updated with release info
[ ] Human stakeholder informed
```

---

## 8. Communication Protocols

### 8.1 Message Format Standards

**Task Assignment (TL to team):**
Use Task Packet format (Section 5). No informal assignments.

**Status Update (team to TL):**
Use Completion Report format (Section 6) for completed work.
For in-progress updates, use:
```
PROGRESS UPDATE
Task ID: <id>
Status: IN_PROGRESS
Progress: <percentage or milestone>
ETA: <expected completion>
Blockers: <if any>
```

**Bug Report (QA to TL):**
Use Bug Report format (Section 4.1). No informal bug reports.

**Question/Clarification Request:**
```
CLARIFICATION REQUEST
From: <role>
Regarding: <Task ID or topic>
Question: <specific question>
Context: <why this matters>
Proposed Answer: <if you have a suggestion>
```

### 8.2 When to Communicate

| Event | Action Required | Who Initiates |
|-------|-----------------|---------------|
| Task assigned | Acknowledge receipt | Assignee |
| Blocker encountered | Report immediately | Blocked party |
| Task completed | Submit Completion Report | Owner |
| Bug found | Submit Bug Report | Finder (usually QA) |
| Scope question | Request clarification | Anyone |
| Contract change needed | Propose to TL | Anyone |
| Progress milestone | Progress Update | Owner (optional) |

### 8.3 Response Time Expectations

| Urgency | Expected Response |
|---------|-------------------|
| CRITICAL blocker | Immediate |
| Task clarification | Same session/day |
| Bug triage | Same day |
| General question | 24 hours |

---

## 9. Document Update Protocol

### 9.1 Document Ownership

| Document | Owner | Approver | Update Trigger |
|----------|-------|----------|----------------|
| PRD.md | PM | PM | Scope change, requirement clarification |
| API_SPEC.md | TL | TL | API contract change |
| ARCHITECTURE.md | TL | TL | System design change |
| TASKS.md | TL | TL | New task, task update |
| STATUS.md | TL | TL | Any status change |
| TEAM.md | TL | Human | Process change |
| CLAUDE.md | TL | Human | Coding standard change |

### 9.2 Update Process

**For PRD.md:**
1. PM identifies need for change
2. PM drafts update with rationale
3. PM notifies TL of change
4. TL assesses impact on technical docs
5. TL updates API_SPEC/ARCHITECTURE if needed
6. TL notifies affected team members

**For API_SPEC.md:**
1. Change needed (bug, feature, or clarification)
2. TL drafts updated spec
3. TL notifies BE and QA of change
4. BE implements change
5. QA updates tests

**For ARCHITECTURE.md:**
1. Design decision made
2. TL documents decision with rationale
3. TL updates affected sections
4. TL notifies BE of any implementation impact

**For STATUS.md:**
1. Any status change occurs (task start, complete, block)
2. TL updates STATUS.md immediately
3. Include timestamp

### 9.3 Version Control for Docs

- All document changes should be committed to git
- Commit message format: `docs(<doc-name>): <brief description>`
- Include date and version in document header
- Maintain changelog section in each document

---

## 10. Conflict Resolution

### 10.1 Types of Conflicts

**Type A: Technical Disagreement**
Example: BE and QA disagree on expected API behavior

**Type B: Scope Disagreement**
Example: PM and TL disagree on what's in MVP

**Type C: Priority Disagreement**
Example: Multiple tasks need same resource

**Type D: Quality vs Speed Tradeoff**
Example: Pressure to ship vs quality concerns

### 10.2 Resolution Process

**Step 1: Document Both Positions**
- Each party writes their position clearly
- Include rationale and evidence

**Step 2: Reference Source of Truth**
Check documents in this order:
1. PRD.md (for product behavior)
2. API_SPEC.md (for API contracts)
3. ARCHITECTURE.md (for technical decisions)
4. TEAM.md (for process)

**Step 3: TL Makes Decision**
- TL reviews positions and documentation
- TL makes binding decision
- TL documents decision with rationale
- TL updates relevant docs if needed

**Step 4: Escalation (if needed)**
If TL cannot resolve or disagreement is with TL:
- Escalate to Human stakeholder
- Present both positions objectively
- Human makes final decision

### 10.3 Decision Authority Matrix

| Conflict Type | First Resolver | Escalation |
|---------------|----------------|------------|
| Technical implementation | TL | Human |
| Product scope/requirements | PM | Human |
| Process/workflow | TL | Human |
| Quality standards | TL (with QA input) | Human |
| Timeline/priority | TL | Human |
| Cross-team resource | TL | Human |

### 10.4 Conflict Prevention

- Use Task Packets to prevent misunderstanding
- Reference docs in all decisions
- Ask clarifying questions early
- Over-communicate rather than assume
- Document decisions as they're made

---

## Appendix A: Quick Reference

### Common Scenarios

**"I'm blocked on a task"**
1. Update task status to BLOCKED in STATUS.md (via TL)
2. Submit Progress Update with blocker details
3. TL will triage and resolve

**"I found a bug"**
1. Document using Bug Report format
2. Submit to TL
3. Wait for triage and task assignment

**"I'm not sure what the requirement means"**
1. Submit Clarification Request
2. TL routes to PM if product-related
3. Wait for clarification before proceeding

**"I finished my task"**
1. Verify DoD checklist is complete
2. Submit Completion Report
3. Wait for TL confirmation

**"I need to change the API"**
1. Propose change to TL with rationale
2. TL updates API_SPEC.md if approved
3. TL notifies affected parties

---

## Appendix B: Abbreviations

| Abbrev | Meaning |
|--------|---------|
| TL | Technical Lead / Orchestrator |
| PM | Product Manager |
| BE | Backend Engineer |
| FE | Frontend Engineer |
| QA | QA Engineer |
| DoD | Definition of Done |
| PRD | Product Requirements Document |
| API_SPEC | API Specification |
| E2E | End-to-End |
| MVP | Minimum Viable Product |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-09 | TL | Initial version |

---

*This document is the operating truth for team collaboration. All team members must follow these protocols. Updates require TL approval and Human notification.*
