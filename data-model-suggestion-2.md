# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Employee Benefits Administration · Created: 2026-05-12

## Philosophy

This model treats every change to the benefits system as an immutable domain event appended to an event store. The event log is the single source of truth; all queryable state (current enrollments, ACA status, carrier files) is derived by replaying or projecting events into materialised read models. This is the Command Query Responsibility Segregation (CQRS) pattern: commands write events, queries read projections.

Benefits administration is a domain where event sourcing provides outsized value. ERISA requires a complete audit trail of every enrollment action. ACA compliance requires answering temporal questions ("was this employee offered coverage in March?"). COBRA requires tracking a chain of qualifying events, notifications, and elections. EDI 834 reconciliation requires comparing what was sent to carriers against what should have been sent. In all of these cases, the event log is not a secondary concern — it IS the business logic.

Real-world precedents include financial ledger systems (every transaction is an immutable entry), healthcare EHR audit systems, and compliance-heavy SaaS platforms in regulated industries. The trade-off is implementation complexity: developers must think in events rather than CRUD, and the read-model projection infrastructure adds operational overhead. But for a platform where "what happened and when" is a regulatory requirement, event sourcing eliminates the impedance mismatch between the data model and the compliance requirements.

**Best for:** Teams building a compliance-first platform where full temporal audit, point-in-time queries, and regulatory replay capability justify the additional architectural complexity.

**Trade-offs:**
- + Perfect, intrinsic audit trail — the event log IS the system of record; no separate audit table needed
- + Temporal queries are native: "what was the state on date X?" is a replay, not a complex historical query
- + Enables AI analytics on change patterns (enrollment velocity, carrier discrepancy trends)
- + New read models can be added without changing the write side (e.g., add a FHIR projection later)
- + Event replay enables testing: replay production events against new business rules to verify correctness
- - Higher implementation complexity; developers must think in events and projections
- - Eventual consistency between write (event store) and read (projections) requires careful design
- - Schema evolution of events requires versioning and upcasting strategies
- - Read model rebuild can be slow for large tenants; requires snapshotting strategy
- - Less familiar to most developers than traditional CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ERISA | The event store itself IS the ERISA audit trail — every enrollment action is an immutable, timestamped event with actor identity |
| HIPAA | PHI fields encrypted in event payloads; event store access controls prevent unauthorised replay; `PhiAccessedEvent` tracks all PHI reads |
| ACA (1094-C/1095-C) | ACA read model is a projection that materialises monthly coverage status from enrollment events; temporal queries are native |
| COBRA | COBRA lifecycle is a natural event chain: `QualifyingEventOccurred` → `CobraNoticeSent` → `CobraElected` → `CobraPremiumPaid` |
| ANSI X12 EDI 834 | `Edi834BatchGenerated` and `Edi834BatchAcknowledged` events; reconciliation compares enrollment event stream against carrier acknowledgment events |
| IRS Section 125 | `Section125ElectionMade` events carry annual limits; projection enforces IRS maximums and tracks YTD contributions |
| HL7 FHIR R5 | FHIR Coverage and ExplanationOfBenefit resources are projections from the event stream — can be rebuilt or versioned independently |
| ISO 30414 | Benefits metrics (cost, participation, utilisation) are computed by projecting aggregate events — no separate reporting schema needed |

---

## Event Store Foundation

```sql
-- =============================================================
-- TENANT (minimal mutable state — only organisational config)
-- =============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    subdomain       VARCHAR(63) UNIQUE,
    ein             VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- EVENT STORE (append-only, immutable, the single source of truth)
-- =============================================================
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    stream_type     VARCHAR(50) NOT NULL,
        -- Employee | Enrollment | BenefitPlan | Carrier
        -- CobraCase | AcaCompliance | Edi834Batch | Section125Account
    stream_id       UUID NOT NULL,                   -- aggregate root ID
    event_type      VARCHAR(100) NOT NULL,
        -- See event catalogue below
    event_version   INTEGER NOT NULL,                -- version within stream (optimistic concurrency)
    event_data      JSONB NOT NULL,                  -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- { "user_id": "...", "ip_address": "...", "correlation_id": "...",
        --   "causation_id": "...", "user_agent": "..." }
    schema_version  INTEGER NOT NULL DEFAULT 1,      -- for event upcasting
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)                 -- optimistic concurrency control
);

-- Append-only: no UPDATE or DELETE permitted (enforce via trigger or RLS policy)
-- Partition by month for performance on large tenants
CREATE INDEX idx_event_store_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store(tenant_id, event_type, created_at);
CREATE INDEX idx_event_store_tenant_time ON event_store(tenant_id, created_at);
CREATE INDEX idx_event_store_correlation ON event_store((metadata->>'correlation_id'));

-- Enforce append-only semantics
CREATE OR REPLACE FUNCTION prevent_event_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Event store is append-only. UPDATE and DELETE are prohibited.';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_event_store_immutable
    BEFORE UPDATE OR DELETE ON event_store
    FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();

-- =============================================================
-- EVENT STORE SNAPSHOT (periodic snapshots for fast rebuild)
-- =============================================================
CREATE TABLE event_store_snapshot (
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,               -- event_version at snapshot time
    snapshot_data   JSONB NOT NULL,                   -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- =============================================================
-- PROJECTION CHECKPOINT (tracks which events each projection has consumed)
-- =============================================================
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) DEFAULT 'running'
        -- running | paused | rebuilding | error
);
```

---

## Event Catalogue

The following event types form the domain language. Each event's `event_data` JSONB payload is documented with example structure.

```sql
-- ============================================================
-- EMPLOYEE LIFECYCLE EVENTS
-- ============================================================
-- Stream: Employee

-- EmployeeHired
-- { "employee_number": "EMP-001", "first_name": "Jane", "last_name": "Doe",
--   "ssn_encrypted": "...", "hire_date": "2026-01-15", "fte_status": "full_time",
--   "hours_per_week": 40, "annual_salary": 85000, "department": "Engineering",
--   "work_location_id": "uuid..." }

-- EmployeeTerminated
-- { "termination_date": "2026-06-30", "reason": "voluntary_resignation",
--   "cobra_eligible": true }

-- EmployeeAddressChanged
-- { "address_type": "residential", "address_line1": "123 Main St",
--   "city": "Austin", "state_code": "US-TX", "postal_code": "78701" }

-- DependentAdded
-- { "dependent_id": "uuid...", "first_name": "John", "last_name": "Doe",
--   "relationship": "spouse", "date_of_birth": "1990-05-20" }

-- DependentRemoved
-- { "dependent_id": "uuid...", "reason": "divorce" }

-- ============================================================
-- ENROLLMENT EVENTS
-- ============================================================
-- Stream: Enrollment

-- EnrollmentInitiated
-- { "employee_id": "uuid...", "plan_id": "uuid...", "coverage_level": "family",
--   "enrollment_period_id": "uuid...", "effective_date": "2026-01-01" }

-- EnrollmentConfirmed
-- { "employee_cost_per_period": 250.00, "employer_contribution_per_period": 750.00,
--   "total_premium_per_period": 1000.00, "dependents_covered": ["uuid1", "uuid2"] }

-- EnrollmentWaived
-- { "plan_type": "medical", "waiver_reason": "spouse_coverage",
--   "other_coverage_source": "Spouse employer - Acme Corp" }

-- EnrollmentTerminated
-- { "termination_date": "2026-06-30", "reason": "employee_termination",
--   "cobra_eligible": true }

-- CoverageLevelChanged
-- { "old_coverage_level": "employee_only", "new_coverage_level": "family",
--   "effective_date": "2026-04-01", "life_event_id": "uuid..." }

-- DependentAddedToEnrollment
-- { "dependent_id": "uuid...", "effective_date": "2026-04-01" }

-- DependentRemovedFromEnrollment
-- { "dependent_id": "uuid...", "effective_date": "2026-07-01", "reason": "aging_out" }

-- ============================================================
-- LIFE EVENT EVENTS
-- ============================================================
-- Stream: Employee

-- LifeEventReported
-- { "event_type": "marriage", "event_date": "2026-03-15",
--   "special_enrollment_deadline": "2026-04-14", "documentation_required": true }

-- LifeEventApproved
-- { "life_event_id": "uuid...", "approved_by": "uuid..." }

-- LifeEventDenied
-- { "life_event_id": "uuid...", "denied_by": "uuid...", "reason": "insufficient_documentation" }

-- ============================================================
-- BENEFIT PLAN EVENTS
-- ============================================================
-- Stream: BenefitPlan

-- PlanCreated
-- { "plan_name": "Gold PPO", "plan_type": "medical", "carrier_id": "uuid...",
--   "group_number": "GRP-12345", "provides_mec": true, "provides_mv": true,
--   "plan_year_start": "2026-01-01", "plan_year_end": "2026-12-31" }

-- PlanRateSet
-- { "coverage_level": "family", "total_premium": 1200.00,
--   "employer_contribution": 900.00, "employee_cost": 300.00,
--   "effective_date": "2026-01-01" }

-- PlanDesignUpdated
-- { "network_tier": "in_network", "individual_deductible": 1500.00,
--   "family_deductible": 3000.00, "individual_oop_max": 6000.00 }

-- PlanDeactivated
-- { "effective_date": "2026-12-31", "reason": "carrier_change" }

-- ============================================================
-- COBRA EVENTS
-- ============================================================
-- Stream: CobraCase

-- CobraQualifyingEventOccurred
-- { "employee_id": "uuid...", "event_type": "termination",
--   "event_date": "2026-06-30", "max_continuation_months": 18,
--   "notification_deadline": "2026-07-30" }

-- CobraNoticeSent
-- { "sent_date": "2026-07-05", "sent_to": "jane.doe@email.com",
--   "election_deadline": "2026-09-03" }

-- CobraElected
-- { "plans_elected": ["uuid1", "uuid2"], "monthly_premium": 1020.00,
--   "start_date": "2026-07-01", "end_date": "2027-12-31" }

-- CobraDeclined
-- { "declined_date": "2026-08-01" }

-- CobraPremiumPaid
-- { "payment_period": "2026-07", "amount": 1020.00, "payment_date": "2026-07-25" }

-- CobraPremiumLapsed
-- { "payment_period": "2026-09", "grace_period_end": "2026-10-01" }

-- CobraCoverageTerminated
-- { "termination_date": "2026-10-01", "reason": "non_payment" }

-- ============================================================
-- SECTION 125 EVENTS
-- ============================================================
-- Stream: Section125Account

-- Section125ElectionMade
-- { "employee_id": "uuid...", "account_type": "fsa_health",
--   "annual_election": 2850.00, "per_period_deduction": 109.62,
--   "plan_year_start": "2026-01-01" }

-- Section125ContributionReceived
-- { "amount": 109.62, "pay_period_end": "2026-01-15", "ytd_total": 109.62 }

-- Section125ClaimSubmitted
-- { "amount": 45.00, "service_date": "2026-01-20", "description": "Pharmacy copay" }

-- Section125ClaimApproved
-- { "claim_id": "uuid...", "amount": 45.00 }

-- Section125FundsForfeited
-- { "forfeited_amount": 150.00, "plan_year_end": "2026-12-31" }

-- ============================================================
-- EDI 834 EVENTS
-- ============================================================
-- Stream: Edi834Batch

-- Edi834BatchGenerated
-- { "carrier_id": "uuid...", "member_count": 45,
--   "isa_control_number": "000000123", "file_hash": "sha256:..." }

-- Edi834BatchSent
-- { "sent_at": "2026-01-02T08:00:00Z", "transport": "sftp" }

-- Edi834BatchAcknowledged
-- { "acknowledged_at": "2026-01-02T10:30:00Z", "accepted_count": 44,
--   "rejected_count": 1, "rejection_details": [{"member_id": "M-123", "reason": "invalid_ssn"}] }

-- Edi834ReconciliationCompleted
-- { "matched": 44, "discrepancies": 1,
--   "discrepancy_details": [{"type": "ghost_enrollment", "member_id": "M-456"}] }

-- ============================================================
-- ACA COMPLIANCE EVENTS
-- ============================================================
-- Stream: AcaCompliance

-- AcaMonthlyStatusRecorded
-- { "employee_id": "uuid...", "tax_year": 2026, "month": 1,
--   "hours_of_service": 173, "is_fte": true, "offer_indicator": "1E",
--   "employee_share_lowest_cost": 250.00, "safe_harbor_code": "2C" }

-- Aca1095cGenerated
-- { "employee_id": "uuid...", "tax_year": 2025 }

-- Aca1094cFiled
-- { "tax_year": 2025, "total_1095c_count": 234, "irs_receipt_id": "AIR-2026-001" }
```

---

## Materialised Read Models (Projections)

These tables are rebuilt from the event store. They can be dropped and reconstructed at any time.

```sql
-- =============================================================
-- PROJECTION: Current Employee State
-- =============================================================
CREATE TABLE rm_employee (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_number VARCHAR(50),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    ssn_encrypted   BYTEA,
    ssn_hash        VARCHAR(64),
    date_of_birth   DATE,
    hire_date       DATE,
    termination_date DATE,
    employment_status VARCHAR(30),
    fte_status      VARCHAR(20),
    hours_per_week  NUMERIC(5,2),
    annual_salary   NUMERIC(12,2),
    department      VARCHAR(200),
    current_address JSONB,
    last_event_id   UUID NOT NULL,                   -- last event that updated this row
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_employee_tenant ON rm_employee(tenant_id);
CREATE INDEX idx_rm_employee_status ON rm_employee(tenant_id, employment_status);

-- =============================================================
-- PROJECTION: Current Enrollments
-- =============================================================
CREATE TABLE rm_enrollment (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    plan_id         UUID NOT NULL,
    plan_name       VARCHAR(255),
    plan_type       VARCHAR(30),
    coverage_level  VARCHAR(30),
    status          VARCHAR(30),
    effective_date  DATE,
    termination_date DATE,
    employee_cost   NUMERIC(10,2),
    employer_contribution NUMERIC(10,2),
    total_premium   NUMERIC(10,2),
    covered_dependents JSONB DEFAULT '[]',
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_enrollment_employee ON rm_enrollment(employee_id);
CREATE INDEX idx_rm_enrollment_status ON rm_enrollment(tenant_id, status);

-- =============================================================
-- PROJECTION: COBRA Cases
-- =============================================================
CREATE TABLE rm_cobra_case (
    stream_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    event_type      VARCHAR(50),
    event_date      DATE,
    status          VARCHAR(30),
        -- pending_notification | notified | elected | declined
        -- active | lapsed | terminated | expired
    monthly_premium NUMERIC(10,2),
    next_payment_due DATE,
    end_date        DATE,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_cobra_status ON rm_cobra_case(tenant_id, status);

-- =============================================================
-- PROJECTION: ACA Monthly Coverage (for 1095-C generation)
-- =============================================================
CREATE TABLE rm_aca_monthly (
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    tax_year        INTEGER NOT NULL,
    month           INTEGER NOT NULL,
    hours_of_service NUMERIC(7,2),
    is_fte          BOOLEAN,
    offer_indicator VARCHAR(5),
    employee_share  NUMERIC(10,2),
    safe_harbor_code VARCHAR(5),
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (employee_id, tax_year, month)
);

CREATE INDEX idx_rm_aca_tenant_year ON rm_aca_monthly(tenant_id, tax_year);

-- =============================================================
-- PROJECTION: Section 125 Account Balances
-- =============================================================
CREATE TABLE rm_section125_balance (
    stream_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    account_type    VARCHAR(30),
    annual_election NUMERIC(10,2),
    ytd_contributions NUMERIC(10,2),
    ytd_claims_approved NUMERIC(10,2),
    available_balance NUMERIC(10,2),
    plan_year_start DATE,
    plan_year_end   DATE,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_s125_employee ON rm_section125_balance(employee_id);

-- =============================================================
-- PROJECTION: EDI 834 Reconciliation Status
-- =============================================================
CREATE TABLE rm_edi834_batch (
    stream_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    carrier_id      UUID NOT NULL,
    member_count    INTEGER,
    status          VARCHAR(30),
    sent_at         TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    matched_count   INTEGER,
    discrepancy_count INTEGER,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_edi834_status ON rm_edi834_batch(tenant_id, status);

-- =============================================================
-- PROJECTION: Carrier (minimal mutable reference)
-- =============================================================
CREATE TABLE rm_carrier (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255),
    carrier_code    VARCHAR(50),
    naic_code       VARCHAR(10),
    edi_sender_id   VARCHAR(30),
    edi_receiver_id VARCHAR(30),
    fhir_endpoint   VARCHAR(500),
    is_active       BOOLEAN,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- PROJECTION: Benefit Plan (current state)
-- =============================================================
CREATE TABLE rm_benefit_plan (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    carrier_id      UUID,
    plan_name       VARCHAR(255),
    plan_type       VARCHAR(30),
    group_number    VARCHAR(50),
    provides_mec    BOOLEAN,
    provides_mv     BOOLEAN,
    is_active       BOOLEAN,
    plan_year_start DATE,
    plan_year_end   DATE,
    rates           JSONB DEFAULT '{}',
        -- { "employee_only": {"total": 500, "employer": 400, "employee": 100}, ... }
    design          JSONB DEFAULT '{}',
        -- { "in_network": {"deductible_individual": 1500, "oop_max_individual": 6000, ...} }
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_plan_tenant ON rm_benefit_plan(tenant_id);
```

---

## Example Queries

### Point-in-time query: "What enrollments did employee X have on March 15, 2026?"

```sql
-- Replay enrollment events up to the target date
SELECT
    e.stream_id AS enrollment_id,
    e.event_type,
    e.event_data,
    e.created_at
FROM event_store e
WHERE e.stream_type = 'Enrollment'
  AND e.event_data->>'employee_id' = :employee_id
  AND e.created_at <= '2026-03-15T23:59:59Z'
ORDER BY e.stream_id, e.event_version;

-- The application replays these events to reconstruct the state at that moment
```

### Compliance query: "Show the COBRA notification timeline for case X"

```sql
SELECT
    event_type,
    event_data,
    metadata->>'user_id' AS actor,
    created_at
FROM event_store
WHERE stream_type = 'CobraCase'
  AND stream_id = :cobra_case_id
ORDER BY event_version;

-- Returns the full chain: QualifyingEventOccurred → NoticeSent → Elected → PremiumPaid → ...
```

### Reconciliation: "Find all EDI 834 batches with unresolved discrepancies"

```sql
SELECT *
FROM rm_edi834_batch
WHERE tenant_id = :tenant_id
  AND discrepancy_count > 0
  AND status != 'reconciled';
```

### AI analytics: "Enrollment change velocity during open enrollment"

```sql
SELECT
    date_trunc('hour', created_at) AS hour,
    COUNT(*) AS enrollment_events,
    COUNT(*) FILTER (WHERE event_type = 'EnrollmentConfirmed') AS confirmations,
    COUNT(*) FILTER (WHERE event_type = 'EnrollmentWaived') AS waivers
FROM event_store
WHERE tenant_id = :tenant_id
  AND stream_type = 'Enrollment'
  AND created_at BETWEEN :oe_start AND :oe_end
GROUP BY 1
ORDER BY 1;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 4 | tenant, event_store, event_store_snapshot, projection_checkpoint |
| Read Model: People | 1 | rm_employee |
| Read Model: Plans & Carriers | 2 | rm_benefit_plan, rm_carrier |
| Read Model: Enrollments | 1 | rm_enrollment |
| Read Model: COBRA | 1 | rm_cobra_case |
| Read Model: ACA | 1 | rm_aca_monthly |
| Read Model: Section 125 | 1 | rm_section125_balance |
| Read Model: EDI | 1 | rm_edi834_batch |
| **Total** | **12** | Plus ~35 event types in the event catalogue |

---

## Key Design Decisions

1. **Single event_store table with JSONB payloads** — Rather than a table per event type, all events go into one table. The `event_type` column enables filtering; the `event_data` JSONB column carries the payload. This makes adding new event types zero-migration: just start writing events with a new `event_type` string.

2. **Optimistic concurrency via stream_id + event_version** — The `UNIQUE(stream_id, event_version)` constraint prevents concurrent conflicting writes to the same aggregate. If two processes try to append version 5 to the same enrollment stream, one will fail and retry.

3. **Append-only enforcement via trigger** — A database trigger prevents UPDATE and DELETE on `event_store`. This is a critical integrity control: the event store must be immutable for the audit trail to be trustworthy.

4. **Snapshots for performance** — For aggregates with hundreds of events (e.g., a long-running COBRA case with monthly premium payments), periodic snapshots avoid replaying the full event history on every read. The snapshot stores serialised aggregate state at a known event version.

5. **Projection checkpoints** — Each read model tracks the last event it has processed. If a projection crashes, it resumes from the checkpoint rather than rebuilding from scratch. If a new projection is added, it starts from the beginning and catches up.

6. **Read models are disposable** — Every `rm_*` table can be dropped and rebuilt by replaying the event store. This means new reporting requirements never require data backfills — just create a new projection and replay.

7. **PHI encryption in event payloads** — SSNs and other PHI are encrypted before being stored in `event_data` JSONB. The encryption key is managed outside the database. This satisfies HIPAA technical safeguards while keeping the event store self-contained.

8. **Correlation and causation IDs in metadata** — Every event carries a `correlation_id` (the business operation that triggered it) and `causation_id` (the specific event that caused this one). This enables full causal tracing: "this COBRA notice was sent because of this termination event, which was part of this HR workflow."

9. **Event schema versioning** — The `schema_version` field enables forward-compatible event evolution. When an event's payload structure changes, the version increments. Event consumers upcast old versions to the current schema during replay, avoiding the need to migrate historical events.

10. **Monthly partitioning recommended** — For production deployments, the `event_store` table should be range-partitioned by `created_at` month. This enables efficient time-range queries and archival of old partitions. The DDL above shows a single table for clarity; partitioning is an operational concern.
