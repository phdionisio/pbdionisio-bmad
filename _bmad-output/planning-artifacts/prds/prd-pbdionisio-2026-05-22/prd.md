---
title: "Ledger & Processing Cash Advances and Liquidation"
status: draft
created: 2026-05-22
updated: 2026-05-22
---

# PRD: Ledger & Processing Cash Advances and Liquidation

## 1. Vision

P.B. Dionisio & Co. staff perform cash payments on behalf of clients for government-related third-party services (notarization, drug tests, ID processing, transport permits, etc.). Today this is tracked manually by a single person using a spreadsheet. The result is human error, lost money trails, and no operational visibility.

This initiative delivers an internal web platform that replaces the spreadsheet with a structured, auditable, role-controlled system for tracking all cash movements. The platform is built in two layers:

- **Ledger** (`pbdionisio-ledger`) — a generic, reusable cash-tracking library that manages funds, transactions, balances, and audit trails. It is framework-agnostic and designed to power multiple future tools.
- **Processing Cash Advances and Liquidation** — the first tool built on the Ledger; it digitalizes the BRP (Budget Request Processing) workflow covering staff cash advances, third-party expense payments, and Finance liquidation.

---

## 2. Problem

The current process is fully manual:

- Staff record cash movements in an Excel/Access spreadsheet maintained by one person.
- No enforced controls — balances can be miscalculated, entries can be skipped or duplicated.
- Money trail is fragile: if the spreadsheet owner is unavailable, operations are blocked.
- No visibility for managers or Finance without direct access to the file.
- Audit after the fact is unreliable.

The core failure is a lack of structure: there is no system of record, no enforced workflow, and no audit trail.

---

## 3. Goals and Success Metrics

| Goal | Success Metric |
|---|---|
| Full money trail | Every peso advanced and spent is traceable to a transaction, actor, and timestamp |
| No lost allocations | Every BRP cash advance is fully accounted for at liquidation; zero unreconciled advances |
| Operational visibility | Managers can see fund balances and transaction status in real time without contacting staff |
| Replaced spreadsheet | Finance processes liquidation through the tool, not Excel/Access |
| Audit readiness | Any transaction can be reconstructed from the audit log for a 10-year lookback |

**Counter-metrics (watch for):** Tool adoption friction causing staff to revert to spreadsheet; over-complicated approval flows slowing operations.

---

## 4. User Roles and Permissions

| Role | Capabilities |
|---|---|
| **Super Admin** | All Admin capabilities + user management (create, deactivate, assign roles) |
| **Admin** | All Manager capabilities + system configuration |
| **Manager** | Add transactions, update transactions, approve transactions, void transactions (with mandatory reason), generate and export reports |
| **Standard User** | Add transactions, update transactions (own), view all transactions (including other users' records) |

**Void rule:** Any transaction may be voided by a Manager or above. Void is irreversible and requires a mandatory written reason. Voided transactions are retained in full — they are never deleted.

**Visibility rule:** Standard Users have read access to all records, not just their own. There is no record-level privacy between staff.

---

## 5. Architecture Overview

The two layers map to the monorepo structure:

```
libs/
  pbdionisio-ledger/        ← Generic cash-tracking library (new)

apps/
  pbdionisio-api/           ← New routers/services/repos for both layers
  pbdionisio-internal-web/  ← New pages under src/pages/Tools/
```

The Ledger library exposes a clean service/repository interface. The Processing tool is an application-layer feature that calls the Ledger's service — it does not touch Ledger internals directly.

---

## 6. Functional Requirements

### Layer 1 — Ledger Library (`pbdionisio-ledger`)

The Ledger is general-purpose. It has no knowledge of BRP workflows, application types, or staff roles — those are Processing Tool concerns.

---

**FR-L1: Fund Management**

The Ledger must support multiple named funds. Each fund has:
- A name and optional description
- A current balance
- A created-by actor and timestamps

Funds can be created and updated. Fund balance is authoritative — it is always the sum of settled transactions, not a free-form field.

---

**FR-L2: Transaction Lifecycle**

Transactions move through a strict state machine:

```
Pending → Approved → Settled → (end)
                  ↘ Voided
Pending → Voided
Approved → Voided
```

Each state transition is recorded with the acting user and timestamp. Backward transitions are not permitted. Only Managers and above may approve or settle; voiding requires a Manager or above plus a mandatory written reason.

Each transaction records: fund, amount, direction (debit/credit), description, created-by, and all state transition history.

---

**FR-L3: Hard Balance Enforcement**

The Ledger must enforce that no debit transaction is approved or settled if it would cause the fund balance to go negative. This is a hard constraint — it cannot be bypassed by any role, including Super Admin.

---

**FR-L4: Audit Log**

Every state change to a fund or transaction must be recorded in an immutable audit log entry capturing: entity type, entity ID, previous state, new state, acting user, and UTC timestamp. Audit log entries are never deleted or updated.

---

**FR-L5: Query Interface**

The Ledger exposes list and detail queries for funds and transactions. Filters: fund, state, date range, acting user. Pagination follows the platform convention (limit/offset + X-Total-Count header).

---

### Layer 2 — Processing Cash Advances and Liquidation Tool

This tool implements the BRP workflow on top of the Ledger. It introduces application-domain concepts (BRP, cash advance, expense line, liquidation report) that the Ledger does not know about.

---

**FR-P1: BRP Request Management**

Staff can create a Budget Request Processing (BRP) entry that:
- Specifies the requesting staff member and the application(s) being processed
- Records the total cash amount being advanced
- Links to the corresponding Ledger fund and debit transaction

A BRP moves from Draft → Submitted → Approved → Liquidated. The Ledger transaction mirrors this with its own state.

---

**FR-P2: Cash Advance Tracking**

When a BRP is approved, the cash advance is tracked as:
- Amount advanced (from approved BRP)
- Amount spent (sum of recorded expenses)
- Balance remaining (advance minus spent)

The remaining balance is always visible to the staff member holding the advance and to Managers.

---

**FR-P3: Expense Recording**

Staff record individual third-party payments against an open BRP. Each expense line captures:
- Payee / service type
- Amount
- Payment method (Landbank online or cash)
- Date of payment
- Notes

The following expense categories are recognized (from the BRP process):

| Category | Payment Method |
|---|---|
| Tax (PTT) | Landbank online |
| ID processing | Landbank online |
| PTT fee | Landbank online |
| Notary | Cash |
| Drug Test / Neuro | Cash |
| GSS / Piso-Piso | Cash |
| Courier | Case-by-case |
| Representation | Cash |
| Others | Cash |

---

**FR-P4: Duplicate Prevention**

The system must detect and flag potential duplicate expense entries: same payee, same amount, same date, same BRP. Duplicates are surfaced as a warning before save; the user may override with a written explanation or discard the entry.

---

**FR-P5: Liquidation and Validation**

When all expenses are recorded, the staff member submits the BRP for liquidation. Finance (Manager role) validates:
- Total expenses match total advance
- No unresolved duplicate flags
- All required fields are complete

On validation, the BRP is marked Liquidated and the Ledger transaction is settled. If the advance and spend do not reconcile, the discrepancy is recorded as an open item requiring resolution before the BRP can close.

---

**FR-P6: Report Generation**

The tool generates the following reports on demand:

| Report | Scope | Notes |
|---|---|---|
| LTOPF Summary (New) | Roces | Per application type |
| LTOPF Summary (Renewal) | Roces | Per application type |
| FA Registration Summary (New) | Roces | Per application type |
| FA Registration Summary (Renewal) | Roces | Per application type |
| Airgun Registration Summary | Roces | Per application type |
| PTT Summary | Roces | Per application type |
| Unified Report | Crame | Representation, Others, Piso-Piso |
| Revolving Fund Replenishment | All | Fund replenishment request summary |

Reports include standard signatories: Prepared by (assigned staff), Noted/Endorsed by (Processing Operation Head — Cherry S. Bencito), Validated by (Finance/Acctg), Received by (Accounting/Finance Department).

---

**FR-P7: Data Export**

Two export modes:

1. **Formatted report export** — Generate any report (FR-P6) as a downloadable PDF or Excel file.
2. **Raw transaction export** — Export transaction data for a date range as CSV or Excel. Includes all fields: BRP ID, staff, application type, expense lines, amounts, payment method, states, timestamps.

Export is available to Manager and above.

---

**FR-P8: Audit Log (Processing Layer)**

All Processing-layer events (BRP created, submitted, approved, liquidated, expense added, duplicate override) are recorded in the platform audit log in addition to the Ledger-level audit entries.

---

## 7. Non-Functional Requirements

**Data Retention:** All transaction records, audit log entries, and reports must be retained for a minimum of 10 years. Retention applies to voided and rejected records — nothing is purged.

**Access:** Browser-based web application. Must be responsive and usable on both desktop and mobile browsers. Deployed internally on a dedicated machine; no public internet exposure required.

**Availability:** Business-hours critical. Downtime during office hours directly impacts operations. Recovery must be fast; the system is not expected to be 24/7 but must be reliably up when staff are working.

**Privacy:** All data is internal. No external regulatory compliance requirements (no GDPR, PCI, or equivalent). Standard internal access controls (role-based) are sufficient.

**Performance:** List views with pagination must load within 3 seconds under normal office-network conditions. Report generation may take longer but must provide user feedback (progress indicator) if exceeding 5 seconds.

**Security:** Authentication via existing Google SSO (cookie-based, server-authoritative). Role enforcement is server-side; the frontend never makes access decisions.

---

## 8. Out of Scope

The following are explicitly not part of this release:

- **Payroll and salaries** — this system tracks operational cash advances only
- **Landbank API integration** — Landbank payments are recorded manually; no direct bank connection
- **Email/SMS notifications** — no automated alerts for approvals or state changes
- **Native mobile application** — mobile access is via web browser only
- **External accounting system sync** — no integration with third-party finance software in this release

---

## 9. Open Questions

| # | Question | Owner | Revisit Condition |
|---|---|---|---|
| OQ-1 | Multi-branch support: should Roces, Crame, and Davao be tracked as distinct entities within the system? | Stakeholders | Before architecture phase; affects Fund and Report data model |
| OQ-2 | Receipt/document uploads: should staff be able to attach scanned receipts or photos to expense lines? | Stakeholders | Before architecture phase; affects storage design |

---

## 10. Glossary

| Term | Definition |
|---|---|
| **BRP** | Budget Request Processing — the internal form authorizing a cash advance for third-party payments |
| **Ledger** | The generic cash-tracking library (`pbdionisio-ledger`) that manages funds, transactions, and audit trails |
| **Fund** | A named pool of money tracked in the Ledger |
| **Cash Advance** | Money issued to a staff member from an approved BRP to pay third-party services |
| **Liquidation** | Finance's validation that all advanced cash is accounted for in recorded expenses |
| **Void** | Irreversible cancellation of a transaction, with mandatory written reason; record is retained |
| **LTOPF** | License to Own and Possess Firearm |
| **FA Registration** | Firearm Registration |
| **PTT** | Permit to Transport |
