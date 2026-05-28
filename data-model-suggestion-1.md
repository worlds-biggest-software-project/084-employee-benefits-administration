# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Employee Benefits Administration · Created: 2026-05-12

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every concept in the benefits domain — employers, employees, dependents, plans, carriers, enrollments, compliance filings, EDI transactions — gets its own dedicated table with explicit foreign key relationships. Reference data (plan types, coverage levels, IRS codes, ACA indicator codes) is stored in lookup tables that enforce data integrity at the database level.

The approach mirrors how incumbent platforms like ADP Workforce Now and Businessolver structure their operational databases: a stable, well-understood schema where every field has a single home, every relationship is explicit, and complex cross-entity queries (e.g., "show all employees whose dependents lost coverage in Q3 and whose COBRA notices have not been sent") can be answered with standard SQL joins.

This is the most natural fit for a domain with heavy compliance requirements (ERISA audit trails, ACA reporting, HIPAA access controls) because every data point is individually addressable, indexable, and auditable. The trade-off is schema rigidity: adding a new benefit type or jurisdiction-specific field requires a migration.

**Best for:** Teams with strong relational database skills building a compliance-first platform where data integrity and queryability matter more than schema flexibility.

**Trade-offs:**
- + Strongest referential integrity; the database enforces business rules via constraints
- + Easiest to reason about for compliance audits and regulatory reporting
- + Well-understood by DBAs; extensive tooling for migrations, backups, monitoring
- + Optimal for complex cross-entity queries (ACA reporting, carrier reconciliation)
- - Schema changes require migrations; adding a jurisdiction-specific field touches DDL
- - High table count (~55-65 tables) increases join complexity for some queries
- - Less flexible for rapidly evolving benefit types (LSAs, stipends, custom perks)
- - Multi-tenant row-level security adds WHERE clauses to every query

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ANSI X12 EDI 834 | Dedicated `edi_834_transaction`, `edi_834_segment` tables model the full 834 envelope (ISA/GS/ST/INS/NM1/HD/REF segments) for generation, parsing, and reconciliation |
| ANSI X12 EDI 820 | `edi_820_transaction` table for premium payment file generation and carrier remittance tracking |
| ACA (IRS 1094-C/1095-C) | `aca_filing`, `aca_1095c_record` tables with IRS-defined indicator codes (1A-1S for Line 14, 2A-2I for Line 16) stored as reference data |
| ERISA | `audit_log` table captures every enrollment action with timestamp, user, and before/after state for fiduciary compliance |
| HIPAA | PHI fields identified and encrypted; `data_access_log` tracks all PHI access; role-based access via `role`, `permission` tables |
| IRS Section 125 | `section_125_plan`, `section_125_election` tables enforce annual limits (FSA $3,300, HSA $4,300/$8,550 for 2025) |
| COBRA | `cobra_event`, `cobra_notice`, `cobra_continuation` tables model the full COBRA lifecycle |
| ISO 3166 | `jurisdiction` table uses ISO 3166-1/3166-2 codes for country/state references |
| HL7 FHIR R5 | Coverage, ExplanationOfBenefit resource mappings documented; FHIR-compatible identifiers stored alongside EDI IDs |
| SCIM 2.0 | Employee identity sync via `scim_sync_log` for HRIS integration |

---

## Core Identity & Multi-Tenancy

```sql
-- =============================================================
-- TENANT / ORGANISATION
-- =============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    subdomain       VARCHAR(63) UNIQUE,
    ein             VARCHAR(20),                    -- Employer Identification Number
    ale_status      BOOLEAN DEFAULT FALSE,          -- Applicable Large Employer (ACA)
    hipaa_entity_type VARCHAR(20),                  -- covered_entity | business_associate
    settings        JSONB DEFAULT '{}',             -- tenant-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_subdomain ON tenant(subdomain);

-- =============================================================
-- EMPLOYEE
-- =============================================================
CREATE TABLE employee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_number VARCHAR(50),
    ssn_encrypted   BYTEA,                          -- AES-256 encrypted SSN (PHI)
    ssn_hash        VARCHAR(64),                    -- SHA-256 hash for lookups
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    middle_name     VARCHAR(100),
    date_of_birth   DATE,
    gender          VARCHAR(20),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    hire_date       DATE NOT NULL,
    termination_date DATE,
    employment_status VARCHAR(30) NOT NULL DEFAULT 'active',
        -- active | terminated | on_leave | retired | cobra
    fte_status      VARCHAR(20) DEFAULT 'full_time',
        -- full_time | part_time | variable_hour
    hours_per_week  NUMERIC(5,2),
    annual_salary   NUMERIC(12,2),
    pay_frequency   VARCHAR(20),                    -- weekly | biweekly | semimonthly | monthly
    job_title       VARCHAR(200),
    department      VARCHAR(200),
    work_location_id UUID REFERENCES work_location(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_tenant ON employee(tenant_id);
CREATE INDEX idx_employee_ssn_hash ON employee(ssn_hash);
CREATE INDEX idx_employee_status ON employee(tenant_id, employment_status);
CREATE UNIQUE INDEX idx_employee_number ON employee(tenant_id, employee_number);

-- =============================================================
-- DEPENDENT
-- =============================================================
CREATE TABLE dependent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE,
    gender          VARCHAR(20),
    ssn_encrypted   BYTEA,
    ssn_hash        VARCHAR(64),
    relationship    VARCHAR(30) NOT NULL,
        -- spouse | child | domestic_partner | disabled_dependent | other
    is_student      BOOLEAN DEFAULT FALSE,
    is_disabled     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dependent_employee ON dependent(employee_id);

-- =============================================================
-- WORK LOCATION (for ACA multi-state tracking)
-- =============================================================
CREATE TABLE work_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_code      VARCHAR(10),                    -- ISO 3166-2
    postal_code     VARCHAR(20),
    country_code    CHAR(2) DEFAULT 'US',           -- ISO 3166-1 alpha-2
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- ADDRESS (employee mailing/residential)
-- =============================================================
CREATE TABLE employee_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    address_type    VARCHAR(20) NOT NULL,            -- mailing | residential
    address_line1   VARCHAR(255) NOT NULL,
    address_line2   VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_code      VARCHAR(10),
    postal_code     VARCHAR(20) NOT NULL,
    country_code    CHAR(2) DEFAULT 'US',
    is_primary      BOOLEAN DEFAULT TRUE,
    effective_date  DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_emp_address_employee ON employee_address(employee_id);
```

---

## Carrier & Plan Management

```sql
-- =============================================================
-- INSURANCE CARRIER
-- =============================================================
CREATE TABLE carrier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    carrier_code    VARCHAR(50),                    -- internal code
    naic_code       VARCHAR(10),                    -- National Association of Insurance Commissioners
    edi_sender_id   VARCHAR(30),                    -- ISA segment sender ID for EDI 834
    edi_receiver_id VARCHAR(30),
    contact_name    VARCHAR(200),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(30),
    fhir_endpoint   VARCHAR(500),                   -- FHIR API base URL if available
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- BENEFIT PLAN TYPE (reference data)
-- =============================================================
CREATE TABLE benefit_plan_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,
        -- medical | dental | vision | life | std | ltd | fsa_health
        -- fsa_dependent | hsa | hra | retirement_401k | voluntary
        -- ad_d | critical_illness | accident | legal | pet | commuter
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(30) NOT NULL,
        -- health | financial | voluntary | retirement | lifestyle
    is_pretax       BOOLEAN DEFAULT FALSE,          -- Section 125 eligible
    requires_hipaa  BOOLEAN DEFAULT FALSE            -- PHI handling required
);

-- =============================================================
-- BENEFIT PLAN
-- =============================================================
CREATE TABLE benefit_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID REFERENCES carrier(id),
    plan_type_id    UUID NOT NULL REFERENCES benefit_plan_type(id),
    plan_name       VARCHAR(255) NOT NULL,
    plan_code       VARCHAR(50),                    -- carrier plan code / group number
    group_number    VARCHAR(50),
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    provides_mec    BOOLEAN DEFAULT FALSE,          -- ACA Minimum Essential Coverage
    provides_mv     BOOLEAN DEFAULT FALSE,          -- ACA Minimum Value
    coverage_levels TEXT[],                          -- {employee_only, employee_spouse, employee_children, family}
    waiting_period_days INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_tenant ON benefit_plan(tenant_id);
CREATE INDEX idx_plan_type ON benefit_plan(plan_type_id);
CREATE INDEX idx_plan_carrier ON benefit_plan(carrier_id);

-- =============================================================
-- PLAN RATE (cost tiers per coverage level)
-- =============================================================
CREATE TABLE plan_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    coverage_level  VARCHAR(30) NOT NULL,
        -- employee_only | employee_spouse | employee_children | family
    total_premium   NUMERIC(10,2) NOT NULL,         -- total monthly premium
    employer_contribution NUMERIC(10,2) NOT NULL,   -- employer pays
    employee_cost   NUMERIC(10,2) NOT NULL,         -- employee pays (pre-tax if Section 125)
    effective_date  DATE NOT NULL,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_plan ON plan_rate(plan_id);

-- =============================================================
-- PLAN DESIGN DETAILS (deductibles, copays, OOP maximums)
-- =============================================================
CREATE TABLE plan_design (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    network_tier    VARCHAR(30) DEFAULT 'in_network',
        -- in_network | out_of_network | tier1 | tier2
    individual_deductible NUMERIC(10,2),
    family_deductible NUMERIC(10,2),
    individual_oop_max NUMERIC(10,2),
    family_oop_max  NUMERIC(10,2),
    copay_primary   NUMERIC(10,2),                  -- primary care visit copay
    copay_specialist NUMERIC(10,2),
    copay_er        NUMERIC(10,2),
    coinsurance_pct NUMERIC(5,2),                   -- e.g., 20.00 = 20%
    rx_copay_generic NUMERIC(10,2),
    rx_copay_brand  NUMERIC(10,2),
    rx_copay_specialty NUMERIC(10,2),
    effective_date  DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_design_plan ON plan_design(plan_id);

-- =============================================================
-- ELIGIBILITY RULE
-- =============================================================
CREATE TABLE eligibility_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    rule_type       VARCHAR(50) NOT NULL,
        -- fte_status | employment_class | hire_date_min | age_min
        -- location | department | union_membership
    operator        VARCHAR(20) NOT NULL,           -- eq | neq | gt | lt | gte | lte | in
    value           VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_elig_rule_plan ON eligibility_rule(plan_id);
```

---

## Enrollment & Life Events

```sql
-- =============================================================
-- ENROLLMENT PERIOD (open enrollment windows)
-- =============================================================
CREATE TABLE enrollment_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    period_type     VARCHAR(30) NOT NULL,            -- open_enrollment | special_enrollment | new_hire
    start_date      TIMESTAMPTZ NOT NULL,
    end_date        TIMESTAMPTZ NOT NULL,
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    status          VARCHAR(20) DEFAULT 'scheduled',
        -- scheduled | active | closed | cancelled
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enroll_period_tenant ON enrollment_period(tenant_id);

-- =============================================================
-- LIFE EVENT (qualifying life events triggering special enrollment)
-- =============================================================
CREATE TABLE life_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    event_type      VARCHAR(50) NOT NULL,
        -- marriage | divorce | birth | adoption | death_of_dependent
        -- loss_of_coverage | gain_of_coverage | address_change
        -- employment_status_change | medicare_eligibility
    event_date      DATE NOT NULL,
    reported_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    special_enrollment_deadline DATE,               -- typically event_date + 30/60 days
    documentation_received BOOLEAN DEFAULT FALSE,
    status          VARCHAR(20) DEFAULT 'pending',
        -- pending | approved | denied | expired
    approved_by     UUID,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_life_event_employee ON life_event(employee_id);

-- =============================================================
-- ENROLLMENT (the core fact: employee elected plan X)
-- =============================================================
CREATE TABLE enrollment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    enrollment_period_id UUID REFERENCES enrollment_period(id),
    life_event_id   UUID REFERENCES life_event(id),
    coverage_level  VARCHAR(30) NOT NULL,
        -- employee_only | employee_spouse | employee_children | family
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending | active | waived | terminated | cobra
    effective_date  DATE NOT NULL,
    termination_date DATE,
    employee_cost_per_period NUMERIC(10,2),
    employer_contribution_per_period NUMERIC(10,2),
    total_premium_per_period NUMERIC(10,2),
    elected_at      TIMESTAMPTZ,
    elected_by      UUID,                           -- user who made the election
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enrollment_employee ON enrollment(employee_id);
CREATE INDEX idx_enrollment_plan ON enrollment(plan_id);
CREATE INDEX idx_enrollment_status ON enrollment(tenant_id, status);
CREATE INDEX idx_enrollment_effective ON enrollment(effective_date, termination_date);

-- =============================================================
-- ENROLLMENT DEPENDENT (which dependents are on this enrollment)
-- =============================================================
CREATE TABLE enrollment_dependent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id   UUID NOT NULL REFERENCES enrollment(id),
    dependent_id    UUID NOT NULL REFERENCES dependent(id),
    effective_date  DATE NOT NULL,
    termination_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enroll_dep_enrollment ON enrollment_dependent(enrollment_id);
CREATE INDEX idx_enroll_dep_dependent ON enrollment_dependent(dependent_id);

-- =============================================================
-- ENROLLMENT WAIVER (employee explicitly declined coverage)
-- =============================================================
CREATE TABLE enrollment_waiver (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    plan_type_id    UUID NOT NULL REFERENCES benefit_plan_type(id),
    enrollment_period_id UUID REFERENCES enrollment_period(id),
    waiver_reason   VARCHAR(100),
        -- other_coverage | spouse_coverage | medicaid | religious | cost | other
    other_coverage_source VARCHAR(200),
    waived_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_waiver_employee ON enrollment_waiver(employee_id);
```

---

## Section 125 & Spending Accounts

```sql
-- =============================================================
-- SECTION 125 PLAN
-- =============================================================
CREATE TABLE section_125_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    fsa_health_limit NUMERIC(10,2) DEFAULT 3300.00,       -- IRS 2025 limit
    fsa_dependent_limit NUMERIC(10,2) DEFAULT 5000.00,
    hsa_individual_limit NUMERIC(10,2) DEFAULT 4300.00,   -- IRS 2025 limit
    hsa_family_limit NUMERIC(10,2) DEFAULT 8550.00,
    carryover_limit NUMERIC(10,2) DEFAULT 640.00,
    grace_period_days INTEGER DEFAULT 0,                   -- 0 or up to 75
    run_out_period_days INTEGER DEFAULT 90,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- SECTION 125 ELECTION
-- =============================================================
CREATE TABLE section_125_election (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    section_125_plan_id UUID NOT NULL REFERENCES section_125_plan(id),
    account_type    VARCHAR(30) NOT NULL,
        -- fsa_health | fsa_dependent | hsa | hra
    annual_election NUMERIC(10,2) NOT NULL,
    per_period_deduction NUMERIC(10,2) NOT NULL,
    employer_contribution NUMERIC(10,2) DEFAULT 0,
    effective_date  DATE NOT NULL,
    end_date        DATE,
    status          VARCHAR(20) DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_s125_election_employee ON section_125_election(employee_id);

-- =============================================================
-- SPENDING ACCOUNT TRANSACTION
-- =============================================================
CREATE TABLE spending_account_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    election_id     UUID NOT NULL REFERENCES section_125_election(id),
    transaction_type VARCHAR(20) NOT NULL,
        -- contribution | claim | reimbursement | carryover | forfeiture
    amount          NUMERIC(10,2) NOT NULL,
    transaction_date DATE NOT NULL,
    description     VARCHAR(500),
    receipt_url     VARCHAR(500),
    status          VARCHAR(20) DEFAULT 'pending',
        -- pending | approved | denied | paid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sa_txn_election ON spending_account_transaction(election_id);
```

---

## COBRA Administration

```sql
-- =============================================================
-- COBRA QUALIFYING EVENT
-- =============================================================
CREATE TABLE cobra_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    event_type      VARCHAR(50) NOT NULL,
        -- termination | reduction_in_hours | death | divorce
        -- medicare_entitlement | dependent_aging_out
    event_date      DATE NOT NULL,
    max_continuation_months INTEGER NOT NULL,        -- 18 or 36 depending on event
    notification_deadline DATE NOT NULL,             -- employer must notify within 30 days
    notification_sent_date DATE,
    election_deadline DATE,                          -- 60 days from notice
    status          VARCHAR(20) DEFAULT 'pending',
        -- pending | notified | elected | declined | expired | terminated
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cobra_event_employee ON cobra_event(employee_id);
CREATE INDEX idx_cobra_event_status ON cobra_event(tenant_id, status);

-- =============================================================
-- COBRA CONTINUATION (elected COBRA coverage)
-- =============================================================
CREATE TABLE cobra_continuation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cobra_event_id  UUID NOT NULL REFERENCES cobra_event(id),
    enrollment_id   UUID NOT NULL REFERENCES enrollment(id),
    monthly_premium NUMERIC(10,2) NOT NULL,          -- full premium + 2% admin fee
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- COBRA PREMIUM PAYMENT
-- =============================================================
CREATE TABLE cobra_payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    continuation_id UUID NOT NULL REFERENCES cobra_continuation(id),
    payment_period_start DATE NOT NULL,
    payment_period_end DATE NOT NULL,
    amount_due      NUMERIC(10,2) NOT NULL,
    amount_paid     NUMERIC(10,2),
    payment_date    DATE,
    grace_period_end DATE NOT NULL,                  -- 30-day grace period
    status          VARCHAR(20) DEFAULT 'due',
        -- due | paid | late | grace_period | lapsed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cobra_payment_status ON cobra_payment(status);
```

---

## ACA Compliance & Reporting

```sql
-- =============================================================
-- ACA MEASUREMENT PERIOD (for variable-hour employees)
-- =============================================================
CREATE TABLE aca_measurement_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    period_type     VARCHAR(20) NOT NULL,            -- standard | initial
    measurement_start DATE NOT NULL,
    measurement_end DATE NOT NULL,
    stability_start DATE NOT NULL,
    stability_end   DATE NOT NULL,
    admin_start     DATE,
    admin_end       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- ACA MONTHLY TRACKING (per-employee per-month)
-- =============================================================
CREATE TABLE aca_employee_month (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    tax_year        INTEGER NOT NULL,
    month           INTEGER NOT NULL CHECK (month BETWEEN 1 AND 12),
    hours_of_service NUMERIC(7,2),
    is_fte          BOOLEAN,
    offer_indicator VARCHAR(5),                      -- IRS Line 14 codes: 1A-1S
    employee_share_lowest_cost NUMERIC(10,2),        -- Line 15: employee's monthly share
    safe_harbor_code VARCHAR(5),                     -- IRS Line 16 codes: 2A-2I
    coverage_start_date DATE,
    coverage_end_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(employee_id, tax_year, month)
);

CREATE INDEX idx_aca_month_employee ON aca_employee_month(employee_id, tax_year);

-- =============================================================
-- ACA FILING (1094-C transmittal + 1095-C records)
-- =============================================================
CREATE TABLE aca_filing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    tax_year        INTEGER NOT NULL,
    filing_type     VARCHAR(20) NOT NULL,            -- original | corrected
    total_1095c_count INTEGER,
    total_fte_count INTEGER,
    filed_at        TIMESTAMPTZ,
    irs_receipt_id  VARCHAR(100),
    status          VARCHAR(20) DEFAULT 'draft',
        -- draft | generated | filed | accepted | rejected
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- ACA 1095-C RECORD (one per employee per tax year)
-- =============================================================
CREATE TABLE aca_1095c_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filing_id       UUID NOT NULL REFERENCES aca_filing(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    -- Line 14 (offer of coverage) for each month: 1A through 1S
    line_14_jan     VARCHAR(5), line_14_feb VARCHAR(5), line_14_mar VARCHAR(5),
    line_14_apr     VARCHAR(5), line_14_may VARCHAR(5), line_14_jun VARCHAR(5),
    line_14_jul     VARCHAR(5), line_14_aug VARCHAR(5), line_14_sep VARCHAR(5),
    line_14_oct     VARCHAR(5), line_14_nov VARCHAR(5), line_14_dec VARCHAR(5),
    -- Line 15 (employee cost) for each month
    line_15_jan     NUMERIC(10,2), line_15_feb NUMERIC(10,2), line_15_mar NUMERIC(10,2),
    line_15_apr     NUMERIC(10,2), line_15_may NUMERIC(10,2), line_15_jun NUMERIC(10,2),
    line_15_jul     NUMERIC(10,2), line_15_aug NUMERIC(10,2), line_15_sep NUMERIC(10,2),
    line_15_oct     NUMERIC(10,2), line_15_nov NUMERIC(10,2), line_15_dec NUMERIC(10,2),
    -- Line 16 (safe harbor) for each month: 2A through 2I
    line_16_jan     VARCHAR(5), line_16_feb VARCHAR(5), line_16_mar VARCHAR(5),
    line_16_apr     VARCHAR(5), line_16_may VARCHAR(5), line_16_jun VARCHAR(5),
    line_16_jul     VARCHAR(5), line_16_aug VARCHAR(5), line_16_sep VARCHAR(5),
    line_16_oct     VARCHAR(5), line_16_nov VARCHAR(5), line_16_dec VARCHAR(5),
    furnished_to_employee_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_1095c_filing ON aca_1095c_record(filing_id);
CREATE INDEX idx_1095c_employee ON aca_1095c_record(employee_id);
```

---

## EDI & Carrier Integration

```sql
-- =============================================================
-- EDI 834 TRANSACTION (one per batch file sent/received)
-- =============================================================
CREATE TABLE edi_834_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID NOT NULL REFERENCES carrier(id),
    direction       VARCHAR(10) NOT NULL,            -- outbound | inbound
    isa_control_number VARCHAR(20),
    gs_control_number VARCHAR(20),
    transaction_count INTEGER,
    file_path       VARCHAR(500),
    raw_content     TEXT,                            -- raw EDI for audit
    status          VARCHAR(20) DEFAULT 'pending',
        -- pending | sent | acknowledged | error | reconciled
    sent_at         TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    error_details   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi834_tenant_carrier ON edi_834_transaction(tenant_id, carrier_id);
CREATE INDEX idx_edi834_status ON edi_834_transaction(status);

-- =============================================================
-- EDI 834 MEMBER RECORD (individual enrollment within a batch)
-- =============================================================
CREATE TABLE edi_834_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id  UUID NOT NULL REFERENCES edi_834_transaction(id),
    enrollment_id   UUID REFERENCES enrollment(id),
    ins_indicator   VARCHAR(5),                      -- INS01: subscriber/dependent indicator
    maintenance_type VARCHAR(5),                     -- INS03: 021=addition, 024=cancellation, 001=change
    member_id       VARCHAR(50),                     -- REF segment member ID
    subscriber_id   VARCHAR(50),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    ssn             VARCHAR(11),
    date_of_birth   DATE,
    gender          VARCHAR(5),
    plan_code       VARCHAR(50),                     -- HD segment
    coverage_level  VARCHAR(30),
    effective_date  DATE,
    termination_date DATE,
    reconciliation_status VARCHAR(20) DEFAULT 'pending',
        -- pending | matched | discrepancy | resolved
    discrepancy_details TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi834_member_txn ON edi_834_member(transaction_id);
CREATE INDEX idx_edi834_member_enrollment ON edi_834_member(enrollment_id);
CREATE INDEX idx_edi834_member_recon ON edi_834_member(reconciliation_status);

-- =============================================================
-- EDI 820 TRANSACTION (premium payment)
-- =============================================================
CREATE TABLE edi_820_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID NOT NULL REFERENCES carrier(id),
    payment_period  DATE NOT NULL,
    total_premium   NUMERIC(12,2) NOT NULL,
    employee_count  INTEGER,
    status          VARCHAR(20) DEFAULT 'draft',
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Payroll Integration

```sql
-- =============================================================
-- PAYROLL DEDUCTION (per employee per pay period)
-- =============================================================
CREATE TABLE payroll_deduction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    enrollment_id   UUID REFERENCES enrollment(id),
    election_id     UUID REFERENCES section_125_election(id),
    deduction_type  VARCHAR(30) NOT NULL,
        -- benefit_premium | fsa_health | fsa_dependent | hsa | hra | retirement_401k
    deduction_code  VARCHAR(20),
    is_pretax       BOOLEAN NOT NULL,
    amount          NUMERIC(10,2) NOT NULL,
    employer_match  NUMERIC(10,2) DEFAULT 0,
    pay_period_start DATE NOT NULL,
    pay_period_end  DATE NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending',
        -- pending | exported | confirmed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payroll_deduction_employee ON payroll_deduction(employee_id);
CREATE INDEX idx_payroll_deduction_period ON payroll_deduction(pay_period_start, pay_period_end);
```

---

## Audit & Access Control

```sql
-- =============================================================
-- ROLE-BASED ACCESS CONTROL
-- =============================================================
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(200),
    employee_id     UUID REFERENCES employee(id),    -- null for broker/admin users
    is_active       BOOLEAN DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_user_email ON app_user(tenant_id, email);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenant(id),      -- null = system-wide role
    name            VARCHAR(50) NOT NULL,
        -- system_admin | tenant_admin | hr_admin | benefits_manager
        -- payroll_manager | broker | employee
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
        -- enrollment.read | enrollment.write | phi.read | phi.write
        -- aca.manage | cobra.manage | edi.manage | admin.manage
    description     TEXT
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);

-- =============================================================
-- AUDIT LOG (ERISA-compliant)
-- =============================================================
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
        -- enrollment.created | enrollment.updated | enrollment.terminated
        -- plan.created | cobra.notified | aca.filed | phi.accessed
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at DESC);

-- =============================================================
-- DATA ACCESS LOG (HIPAA PHI access tracking)
-- =============================================================
CREATE TABLE data_access_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,            -- employee | dependent | enrollment
    resource_id     UUID NOT NULL,
    phi_fields_accessed TEXT[],                       -- {ssn, date_of_birth, medical_plan}
    access_type     VARCHAR(10) NOT NULL,            -- read | write
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_phi_access_tenant ON data_access_log(tenant_id, created_at DESC);
```

---

## AI Decision Support

```sql
-- =============================================================
-- PLAN RECOMMENDATION (AI-generated at enrollment)
-- =============================================================
CREATE TABLE plan_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    enrollment_period_id UUID REFERENCES enrollment_period(id),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    rank            INTEGER NOT NULL,
    score           NUMERIC(5,3),                    -- 0.000 to 1.000
    rationale       TEXT,                            -- human-readable explanation
    projected_annual_cost NUMERIC(10,2),
    projected_oop_cost NUMERIC(10,2),
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recommendation_employee ON plan_recommendation(employee_id, enrollment_period_id);

-- =============================================================
-- COST MODEL SCENARIO
-- =============================================================
CREATE TABLE cost_model_scenario (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    created_by      UUID REFERENCES app_user(id),
    plan_year       DATE,
    total_employer_cost NUMERIC(14,2),
    total_employee_cost NUMERIC(14,2),
    scenario_parameters JSONB NOT NULL,
    results         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 5 | tenant, employee, dependent, work_location, employee_address |
| Carrier & Plan Management | 5 | carrier, benefit_plan_type, benefit_plan, plan_rate, plan_design |
| Eligibility | 1 | eligibility_rule |
| Enrollment & Life Events | 5 | enrollment_period, life_event, enrollment, enrollment_dependent, enrollment_waiver |
| Section 125 & Spending Accounts | 3 | section_125_plan, section_125_election, spending_account_transaction |
| COBRA | 3 | cobra_event, cobra_continuation, cobra_payment |
| ACA Compliance | 4 | aca_measurement_period, aca_employee_month, aca_filing, aca_1095c_record |
| EDI & Carrier Integration | 4 | edi_834_transaction, edi_834_member, edi_820_transaction |
| Payroll | 1 | payroll_deduction |
| Access Control | 5 | app_user, role, user_role, permission, role_permission |
| Audit & Compliance | 2 | audit_log, data_access_log |
| AI & Decision Support | 2 | plan_recommendation, cost_model_scenario |
| **Total** | **40** | |

---

## Key Design Decisions

1. **Tenant-per-row multi-tenancy** — Every data table carries a `tenant_id` foreign key. This avoids the operational complexity of schema-per-tenant while enabling row-level security policies in PostgreSQL. All queries must include `tenant_id` in WHERE clauses; application middleware enforces this.

2. **Encrypted SSN with lookup hash** — SSNs are stored AES-256 encrypted (`ssn_encrypted`) with a separate SHA-256 hash (`ssn_hash`) for equality lookups. This satisfies HIPAA technical safeguards without preventing efficient search.

3. **Dedicated 1095-C monthly columns** — Rather than a normalised month-by-month table, the `aca_1095c_record` table uses 36 monthly columns (12 months x 3 lines) to match the IRS form layout directly. This makes form generation trivial and avoids 12-row joins per employee.

4. **Explicit EDI 834 member records** — Each member within an EDI 834 batch gets its own row in `edi_834_member`, linked back to the `enrollment` it represents. This enables automated reconciliation: compare `edi_834_member` records against `enrollment` records to detect ghost enrollments and missed terminations.

5. **Separate audit_log and data_access_log** — The `audit_log` tracks business actions (enrollment changes, COBRA notices) for ERISA compliance. The `data_access_log` tracks PHI field access for HIPAA compliance. Separating them enables different retention policies and access controls.

6. **Plan rates as temporal records** — `plan_rate` rows have `effective_date` and `end_date`, enabling historical rate tracking without mutating existing records. This supports "what was the cost on date X?" queries for ACA affordability calculations.

7. **Eligibility rules as data** — Rather than hardcoding eligibility logic, `eligibility_rule` rows define conditions declaratively (e.g., `fte_status eq full_time`). The application evaluates these rules at enrollment time, enabling HR administrators to configure eligibility without code changes.

8. **AI recommendations with transparency** — `plan_recommendation` stores the model version and human-readable rationale alongside the score, satisfying DOL 2025 guidance on ERISA fiduciary transparency for AI-assisted benefit decisions.
