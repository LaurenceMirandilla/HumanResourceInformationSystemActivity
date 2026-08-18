# Part 3 — Bounded Contexts (HRIS)

**System:** Human Resource Information System (HRIS)
**Repository:** https://github.com/LaurenceMirandilla/HumanResourceInformationSystemActivity

This document decomposes the HRIS monolith into three independent bounded contexts. Each
context owns its own data, its own database, and its own deployable microservice. No context
reads or writes another context's database directly — all cross-context communication happens
through the API Gateway (synchronous REST) or the message broker (asynchronous events).

---

## Overview

| # | Bounded Context | Microservice | Database | Primary Responsibility |
|---|-----------------|--------------|----------|------------------------|
| 1 | Employee Records (Core HR) | `employee-service` | `employee_db` (PostgreSQL) | System of record for people, jobs, and org structure |
| 2 | Payroll & Compensation | `payroll-service` | `payroll_db` (PostgreSQL) | Salary computation, deductions, payslip generation |
| 3 | Leave & Time-off Management | `leave-service` | `leave_db` (PostgreSQL) | Leave balances, applications, approval workflow |

---

## 1. Employee Records (Core HR)

**What it does.** The Employee Records context is the authoritative system of record for every
person employed by the organization; it owns onboarding, profile maintenance, job and department
assignment, and offboarding, and it is the only context permitted to declare that an employee
exists or has been terminated. It exposes employee identity and employment status to the rest of
the HRIS so that no other context has to guess who is currently active.

**Key entities.** `Employee` (employee ID, name, contact details, date hired, employment status),
`JobPosition` (title, job grade, salary band), `Department`, `EmploymentContract` (contract type,
start/end date), and `ReportingLine` (which manager an employee reports to).

**Integration points.** It publishes the domain events `EmployeeHired`, `EmployeeProfileUpdated`,
`EmployeeTransferred`, and `EmployeeTerminated` to the message broker; Payroll subscribes to these
to open or close a payroll record, and Leave Management subscribes to them to provision or freeze a
leave balance. It also exposes a synchronous `GET /employees/{id}` REST endpoint that Leave
Management calls to validate an approver's reporting line, and that Payroll calls to confirm an
employee's job grade before a pay run. It never calls the other two services — it is a pure
upstream (supplier) context, which keeps the dependency graph acyclic.

---

## 2. Payroll & Compensation

**What it does.** The Payroll context is responsible for turning employment facts into money: it
runs the pay cycle, computes gross pay from salary and allowances, applies statutory and voluntary
deductions, and produces the payslip and bank disbursement file for each pay period. It owns all
compensation rules and is the only context allowed to declare a pay run final. It deliberately does
*not* own employee identity — it keeps a local read-model copy of the employee data it needs so a
pay run can complete even if Employee Records is temporarily unavailable.

**Key entities.** `PayrollRun` (pay period, status), `SalaryStructure` (basic pay, allowances),
`Deduction` (tax, SSS/PhilHealth/Pag-IBIG, loans), `Payslip` (gross pay, net pay, breakdown), and
`PaymentTransaction`.

**Integration points.** It subscribes to `EmployeeHired` and `EmployeeTerminated` from Employee
Records to create or deactivate a salary structure, and it subscribes to `LeaveApproved` and
`LeaveCancelled` from Leave Management so that unpaid leave days are converted into salary
deductions on the next pay run. It calls Employee Records synchronously over REST
(`GET /employees/{id}`) to confirm job grade and bank details immediately before finalizing a run.
After a run completes it publishes `PayrollProcessed` and `PayslipGenerated`, which the client app
consumes (via the gateway) to notify employees that a payslip is ready.

---

## 3. Leave & Time-off Management

**What it does.** The Leave context owns the full time-off lifecycle: it maintains each employee's
leave entitlement and running balance, accepts leave applications, routes them through the approval
workflow to the correct approver, and records the final approved or rejected outcome. It enforces
leave policy — accrual rates, carry-over caps, and whether a given leave type is paid or unpaid —
and is the single source of truth for whether an employee was legitimately absent on a given date.

**Key entities.** `LeaveType` (vacation, sick, emergency, unpaid — each with a paid/unpaid flag),
`LeaveBalance` (entitled, used, remaining days per employee per year), `LeaveApplication` (date
range, reason, status), `ApprovalWorkflow` (approver, decision, timestamp), and `Holiday` calendar.

**Integration points.** It subscribes to `EmployeeHired` from Employee Records to provision an
opening leave balance and to `EmployeeTerminated` to freeze any remaining balance. When an
application is approved it publishes `LeaveApproved` (carrying employee ID, leave type, paid/unpaid
flag, and the affected dates) to the message broker, which Payroll subscribes to for deduction
computation; a reversal publishes `LeaveCancelled`. It calls Employee Records synchronously over
REST to resolve an employee's manager before routing an approval request. It never calls Payroll
directly — the paid/unpaid consequence is communicated purely as an event, so leave approval stays
fast and is not blocked by a running pay cycle.

---

## Context Map Summary

```
Employee Records  ──(events: EmployeeHired / Transferred / Terminated)──►  Payroll
Employee Records  ──(events: EmployeeHired / Terminated)──────────────►  Leave Management
Leave Management  ──(events: LeaveApproved / LeaveCancelled)──────────►  Payroll
Payroll           ──(event: PayrollProcessed / PayslipGenerated)──────►  Client (via Gateway)

Leave Management  ──(REST: GET /employees/{id})──►  Employee Records
Payroll           ──(REST: GET /employees/{id})──►  Employee Records
```

**Relationship patterns used**

- **Employee Records → Payroll / Leave:** *Customer–Supplier*. Employee Records is upstream; the
  two downstream contexts conform to the employee event schema it publishes.
- **Leave → Payroll:** *Published Language*. The `LeaveApproved` event is a stable, versioned
  contract; Payroll does not know Leave's internal workflow model.
- **Anti-Corruption Layer:** Payroll and Leave each translate incoming employee events into their
  own local models rather than storing Employee Records' schema verbatim, so an upstream schema
  change does not ripple through the whole system.

**Why these boundaries?** Each context changes for a different reason and at a different rate —
payroll rules change with tax legislation, leave rules change with company policy, and employee
records change with org structure. Splitting along those lines means a change to one rarely forces
a redeploy of the others, which is the practical test of a good bounded context.
