# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Employee Benefits Administration · Created: 2026-05-12

## Philosophy

This model uses a relational backbone for core entities (employees, plans, enrollments, carriers) while delegating variable and jurisdiction-specific data to JSONB columns. The key insight is that benefits administration has a stable core — every employer has employees, plans, enrollments, and carriers — but enormous variance in the details: plan design fields differ by plan type (medical vs. dental vs. FSA vs. commuter), eligibility rules vary by jurisdiction, EDI segment structures differ by carrier, and international benefits have entirely different field requirements.

Rather than creating dozens of specialised tables (one for medical plan design, one for dental plan design, one for vision plan design) or using entity-attribute-value (EAV) anti-patterns, this model stores the variable portions in JSONB columns with documented schemas and GIN indexes. PostgreSQL's JSONB operators provide efficient querying, and CHECK constraints with `jsonb_typeof` can enforce basic structural validity at the database level.

This approach is common in modern SaaS platforms that serve multiple markets or plan types from a single codebase. Rippling, Gusto, and similar platforms use hybrid schemas to handle the diversity of benefit types, carriers, and jurisdictions without requiring schema migrations for each new plan type or country. The trade-off is weaker type safety on JSONB fields — the application layer must validate structure — but the flexibility to support new benefit types, carriers, and jurisdictions without DDL changes.

**Best for:** Teams building a platform that must rapidly support new benefit types (LSAs, stipends, commuter benefits), multiple jurisdictions (US, EU, Canada), and carrier-specific EDI variations without schema migrations.

**Trade-offs:**
- + New benefit types require no DDL changes — add a new plan_type and document the JSONB schema
- + Jurisdiction-specific fields (state continuation laws, EU GDPR fields) stored without schema bloat
- + Carrier-specific EDI mappings stored as configuration data, not hardcoded schema
- + Faster MVP development; fewer tables to manage
- + Natural fit for API responses — JSONB columns map directly to JSON API payloads
- - Weaker referential integrity on JSONB fields; application must validate
- - JSONB queries can be slower than indexed relational columns for complex predicates
- - Schema documentation must be maintained separately (not enforced by DDL)
- - Harder to enforce NOT NULL and type constraints on nested JSONB fields
- - Database-level reporting tools (BI connectors) may struggle with JSONB columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ANSI X12 EDI 834 | Carrier EDI configuration stored as JSONB in `carrier_edi_config`; segment mappings are data, not schema, enabling per-carrier customisation |
| ACA (1094-C/1095-C) | ACA monthly data uses relational columns for IRS-defined fields; additional state-specific reporting stored in JSONB |
| ERISA | `audit_log` table with relational core + JSONB `old_values`/`new_values` for flexible change tracking |
| HIPAA | PHI fields in relational columns (encrypted); JSONB columns explicitly documented as non-PHI or encrypted |
| IRS Section 125 | Account type limits stored in JSONB on `section_125_config`; supports annual IRS limit updates without migration |
| COBRA | Relational COBRA lifecycle tables; state-specific continuation rules (mini-COBRA) stored as JSONB config |
| HL7 FHIR R5 | FHIR resource representations stored as JSONB for caching and API response; Coverage, EOB resources |
| ISO 3166 | Jurisdiction model uses ISO codes relationally; jurisdiction-specific rules stored in JSONB |
| ISO 30414 | Benefits metrics configuration (which metrics to report, target values) stored as JSONB per tenant |

---

## Core Identity & Multi-Tenancy

```sql
-- =============================================================
-- TENANT
-- =============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    subdomain       VARCHAR(63) UNIQUE,
    ein             VARCHAR(20),
    ale_status      BOOLEAN DEFAULT FALSE,
    country_code    CHAR(2) DEFAULT 'US',
    settings        JSONB NOT NULL DEFAULT '{}',
        -- Example:
        -- {
        --   "timezone": "America/New_York",
        --   "pay_frequencies": ["biweekly", "semimonthly"],
        --   "fiscal_year_start_month": 1,
        --   "hipaa_entity_type": "covered_entity",
        --   "features_enabled": ["aca_reporting", "cobra", "section_125", "ai_recommendations"],
        --   "iso30414_metrics": {"benefits_cost_ratio": true, "participation_rate": true}
        -- }
    compliance_config JSONB NOT NULL DEFAULT '{}',
        -- Example:
        -- {
        --   "aca": {"measurement_period_months": 12, "admin_period_days": 60},
        --   "cobra": {"admin_fee_pct": 2.0},
        --   "section_125": {"grace_period_days": 0, "carryover_enabled": true},
        --   "state_continuation": {
        --     "US-CA": {"cal_cobra_months": 36, "min_employees": 2},
        --     "US-NY": {"ny_continuation_months": 36, "min_employees": 1}
        --   }
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- EMPLOYEE
-- =============================================================
CREATE TABLE employee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_number VARCHAR(50),
    ssn_encrypted   BYTEA,
    ssn_hash        VARCHAR(64),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    middle_name     VARCHAR(100),
    date_of_birth   DATE,
    gender          VARCHAR(20),
    email           VARCHAR(255),
    hire_date       DATE NOT NULL,
    termination_date DATE,
    employment_status VARCHAR(30) NOT NULL DEFAULT 'active',
    fte_status      VARCHAR(20) DEFAULT 'full_time',
    hours_per_week  NUMERIC(5,2),
    annual_salary   NUMERIC(12,2),
    pay_frequency   VARCHAR(20),
    demographics    JSONB NOT NULL DEFAULT '{}',
        -- Flexible fields that vary by jurisdiction:
        -- {
        --   "preferred_name": "Jane",
        --   "pronouns": "she/her",
        --   "ethnicity": "not_disclosed",         -- US EEO reporting
        --   "national_insurance_number": "...",    -- UK
        --   "tax_file_number": "...",              -- Australia
        --   "sin_encrypted": "...",                -- Canada
        --   "languages": ["en", "es"],
        --   "emergency_contact": {
        --     "name": "John Doe", "phone": "+1-555-0100", "relationship": "spouse"
        --   }
        -- }
    addresses       JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"type": "residential", "line1": "123 Main St", "city": "Austin",
        --    "state": "US-TX", "postal_code": "78701", "country": "US",
        --    "effective_date": "2026-01-01", "is_primary": true},
        --   {"type": "mailing", "line1": "PO Box 456", ...}
        -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_tenant ON employee(tenant_id);
CREATE INDEX idx_employee_ssn_hash ON employee(ssn_hash);
CREATE INDEX idx_employee_status ON employee(tenant_id, employment_status);
CREATE UNIQUE INDEX idx_employee_number ON employee(tenant_id, employee_number);
CREATE INDEX idx_employee_demographics ON employee USING GIN (demographics);

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
    details         JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "is_student": true,
        --   "school_name": "UT Austin",
        --   "is_disabled": false,
        --   "court_order_required": false,          -- QMSCO for divorce situations
        --   "custody_document_url": null
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dependent_employee ON dependent(employee_id);
```

---

## Carrier & Plan Management

```sql
-- =============================================================
-- CARRIER
-- =============================================================
CREATE TABLE carrier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    carrier_code    VARCHAR(50),
    naic_code       VARCHAR(10),
    is_active       BOOLEAN DEFAULT TRUE,
    contact         JSONB NOT NULL DEFAULT '{}',
        -- { "name": "John Smith", "email": "john@carrier.com",
        --   "phone": "+1-555-0200", "account_manager": "Sarah Lee" }
    edi_config      JSONB NOT NULL DEFAULT '{}',
        -- Carrier-specific EDI 834 configuration:
        -- {
        --   "sender_id": "EMPLOYER01",
        --   "receiver_id": "CARRIER01",
        --   "version": "005010X220A1",
        --   "transport": "sftp",
        --   "sftp_host": "edi.carrier.com",
        --   "sftp_path": "/inbound/834/",
        --   "file_naming": "{tenant}_{date}_{seq}.edi",
        --   "segment_terminators": {"element": "*", "segment": "~"},
        --   "custom_ref_qualifiers": {"member_id": "0F", "group_id": "1L"},
        --   "acknowledgment_expected": true,
        --   "reconciliation_frequency": "monthly"
        -- }
    fhir_config     JSONB DEFAULT NULL,
        -- { "base_url": "https://api.carrier.com/fhir/R4",
        --   "auth_type": "oauth2", "token_url": "...",
        --   "supported_resources": ["Coverage", "ExplanationOfBenefit"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- BENEFIT PLAN (unified across all plan types)
-- =============================================================
CREATE TABLE benefit_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID REFERENCES carrier(id),
    plan_type       VARCHAR(30) NOT NULL,
        -- medical | dental | vision | life | std | ltd | fsa_health
        -- fsa_dependent | hsa | hra | retirement_401k | voluntary
        -- ad_d | critical_illness | accident | legal | pet
        -- commuter_transit | commuter_parking | lsa | stipend
    plan_name       VARCHAR(255) NOT NULL,
    group_number    VARCHAR(50),
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    is_pretax       BOOLEAN DEFAULT FALSE,
    waiting_period_days INTEGER DEFAULT 0,
    coverage_levels VARCHAR(30)[] DEFAULT '{employee_only}',
    aca_flags       JSONB NOT NULL DEFAULT '{}',
        -- { "provides_mec": true, "provides_mv": true,
        --   "affordability_safe_harbor": "rate_of_pay" }
    plan_design     JSONB NOT NULL DEFAULT '{}',
        -- Structure varies by plan_type:
        --
        -- MEDICAL:
        -- { "network_type": "ppo",
        --   "in_network": {
        --     "individual_deductible": 1500, "family_deductible": 3000,
        --     "individual_oop_max": 6000, "family_oop_max": 12000,
        --     "copay_primary": 25, "copay_specialist": 50, "copay_er": 250,
        --     "coinsurance_pct": 20,
        --     "rx": {"generic": 10, "brand": 35, "specialty": 100}
        --   },
        --   "out_of_network": { ... }
        -- }
        --
        -- DENTAL:
        -- { "annual_max": 2000, "preventive_coverage_pct": 100,
        --   "basic_coverage_pct": 80, "major_coverage_pct": 50,
        --   "ortho_coverage_pct": 50, "ortho_lifetime_max": 1500 }
        --
        -- VISION:
        -- { "exam_copay": 10, "frames_allowance": 150,
        --   "contacts_allowance": 150, "exam_frequency_months": 12 }
        --
        -- LIFE / AD&D:
        -- { "coverage_multiple": 2, "max_benefit": 500000,
        --   "age_reduction_schedule": [{"age": 65, "reduction_pct": 35}],
        --   "voluntary_increments": 10000 }
        --
        -- HSA:
        -- { "employer_seed_individual": 500, "employer_seed_family": 1000 }
        --
        -- LSA (Lifestyle Spending Account):
        -- { "annual_allowance": 1200,
        --   "eligible_categories": ["fitness", "education", "wellness", "childcare"],
        --   "rollover": false }
    rates           JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "employee_only": {"total": 500, "employer": 400, "employee": 100},
        --   "employee_spouse": {"total": 900, "employer": 650, "employee": 250},
        --   "employee_children": {"total": 850, "employer": 600, "employee": 250},
        --   "family": {"total": 1200, "employer": 850, "employee": 350},
        --   "effective_date": "2026-01-01"
        -- }
    eligibility_rules JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"field": "fte_status", "operator": "in", "value": ["full_time", "part_time_benefits_eligible"]},
        --   {"field": "employment_class", "operator": "eq", "value": "salaried"},
        --   {"field": "hire_date", "operator": "gte", "value": "2020-01-01"}
        -- ]
    carrier_metadata JSONB NOT NULL DEFAULT '{}',
        -- Carrier-specific codes and mappings:
        -- { "carrier_plan_id": "PPO-GOLD-2026",
        --   "edi_hd_segment": {"insurance_line_code": "HLT", "coverage_level_code": "EMP"},
        --   "sbc_document_url": "https://..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_tenant ON benefit_plan(tenant_id);
CREATE INDEX idx_plan_type ON benefit_plan(plan_type);
CREATE INDEX idx_plan_carrier ON benefit_plan(carrier_id);
CREATE INDEX idx_plan_design ON benefit_plan USING GIN (plan_design);
CREATE INDEX idx_plan_aca ON benefit_plan USING GIN (aca_flags);

-- =============================================================
-- PLAN RATE HISTORY (for temporal rate tracking)
-- =============================================================
CREATE TABLE plan_rate_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    rates           JSONB NOT NULL,                  -- same structure as benefit_plan.rates
    effective_date  DATE NOT NULL,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_history_plan ON plan_rate_history(plan_id, effective_date);
```

---

## Enrollment

```sql
-- =============================================================
-- ENROLLMENT PERIOD
-- =============================================================
CREATE TABLE enrollment_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    period_type     VARCHAR(30) NOT NULL,
    start_date      TIMESTAMPTZ NOT NULL,
    end_date        TIMESTAMPTZ NOT NULL,
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    status          VARCHAR(20) DEFAULT 'scheduled',
    config          JSONB NOT NULL DEFAULT '{}',
        -- { "reminder_schedule": ["7d_before_end", "3d_before_end", "1d_before_end"],
        --   "auto_enroll_defaults": true,
        --   "require_waiver_reason": true }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- LIFE EVENT
-- =============================================================
CREATE TABLE life_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    event_type      VARCHAR(50) NOT NULL,
    event_date      DATE NOT NULL,
    reported_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    special_enrollment_deadline DATE,
    status          VARCHAR(20) DEFAULT 'pending',
    details         JSONB NOT NULL DEFAULT '{}',
        -- { "documentation_received": true,
        --   "document_urls": ["s3://..."],
        --   "approved_by": "uuid...",
        --   "notes": "Marriage certificate verified" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_life_event_employee ON life_event(employee_id);

-- =============================================================
-- ENROLLMENT
-- =============================================================
CREATE TABLE enrollment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    enrollment_period_id UUID REFERENCES enrollment_period(id),
    life_event_id   UUID REFERENCES life_event(id),
    coverage_level  VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    effective_date  DATE NOT NULL,
    termination_date DATE,
    costs           JSONB NOT NULL DEFAULT '{}',
        -- { "employee_per_period": 250.00,
        --   "employer_per_period": 750.00,
        --   "total_premium_per_period": 1000.00,
        --   "annual_employee_cost": 6000.00,
        --   "annual_employer_cost": 18000.00 }
    covered_dependents UUID[] DEFAULT '{}',           -- array of dependent IDs
    election_metadata JSONB NOT NULL DEFAULT '{}',
        -- { "elected_at": "2026-01-10T14:30:00Z",
        --   "elected_by": "uuid...",
        --   "election_source": "employee_portal",
        --   "ai_recommendation_rank": 1,
        --   "ai_recommendation_id": "uuid...",
        --   "previous_plan_id": null,
        --   "waiver_reason": null }
    carrier_confirmation JSONB DEFAULT NULL,
        -- { "confirmed_at": "2026-01-15",
        --   "carrier_member_id": "MBR-12345",
        --   "carrier_group_id": "GRP-67890",
        --   "edi_transaction_id": "uuid..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enrollment_employee ON enrollment(employee_id);
CREATE INDEX idx_enrollment_plan ON enrollment(plan_id);
CREATE INDEX idx_enrollment_status ON enrollment(tenant_id, status);
CREATE INDEX idx_enrollment_effective ON enrollment(effective_date, termination_date);
CREATE INDEX idx_enrollment_dependents ON enrollment USING GIN (covered_dependents);
```

---

## COBRA & Compliance

```sql
-- =============================================================
-- COBRA CASE (unified lifecycle)
-- =============================================================
CREATE TABLE cobra_case (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    event_type      VARCHAR(50) NOT NULL,
    event_date      DATE NOT NULL,
    max_continuation_months INTEGER NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending',
    lifecycle       JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "notification": {
        --     "deadline": "2026-07-30", "sent_date": "2026-07-05",
        --     "sent_to": "jane.doe@email.com", "method": "email"
        --   },
        --   "election": {
        --     "deadline": "2026-09-03", "elected_date": "2026-08-01",
        --     "plans_elected": ["uuid1", "uuid2"],
        --     "monthly_premium": 1020.00,
        --     "start_date": "2026-07-01", "end_date": "2027-12-31"
        --   },
        --   "payments": [
        --     {"period": "2026-07", "amount_due": 1020.00, "amount_paid": 1020.00,
        --      "paid_date": "2026-07-25", "grace_period_end": "2026-08-30", "status": "paid"},
        --     {"period": "2026-08", "amount_due": 1020.00, "status": "due",
        --      "grace_period_end": "2026-09-30"}
        --   ],
        --   "termination": null
        -- }
    state_continuation JSONB DEFAULT NULL,
        -- For states with mini-COBRA laws:
        -- { "law": "cal_cobra", "additional_months": 18,
        --   "extended_end_date": "2029-06-30" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cobra_employee ON cobra_case(employee_id);
CREATE INDEX idx_cobra_status ON cobra_case(tenant_id, status);

-- =============================================================
-- ACA EMPLOYEE MONTH (relational — IRS-defined fields)
-- =============================================================
CREATE TABLE aca_employee_month (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    tax_year        INTEGER NOT NULL,
    month           INTEGER NOT NULL CHECK (month BETWEEN 1 AND 12),
    hours_of_service NUMERIC(7,2),
    is_fte          BOOLEAN,
    offer_indicator VARCHAR(5),
    employee_share_lowest_cost NUMERIC(10,2),
    safe_harbor_code VARCHAR(5),
    additional_data JSONB DEFAULT '{}',
        -- State-specific ACA-adjacent reporting:
        -- { "ca_state_mandate_indicator": "1A",
        --   "nj_health_mandate_covered": true }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(employee_id, tax_year, month)
);

CREATE INDEX idx_aca_month_tenant_year ON aca_employee_month(tenant_id, tax_year);

-- =============================================================
-- ACA FILING
-- =============================================================
CREATE TABLE aca_filing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    tax_year        INTEGER NOT NULL,
    filing_type     VARCHAR(20) NOT NULL,
    status          VARCHAR(20) DEFAULT 'draft',
    filing_details  JSONB NOT NULL DEFAULT '{}',
        -- { "total_1095c_count": 234, "total_fte_count": 180,
        --   "filed_at": "2026-03-15T10:00:00Z",
        --   "irs_receipt_id": "AIR-2026-001",
        --   "furnishing_deadline": "2026-03-02",
        --   "filing_deadline": "2026-03-31" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- SECTION 125 ACCOUNT
-- =============================================================
CREATE TABLE section_125_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    account_type    VARCHAR(30) NOT NULL,
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    annual_election NUMERIC(10,2) NOT NULL,
    per_period_deduction NUMERIC(10,2) NOT NULL,
    employer_contribution NUMERIC(10,2) DEFAULT 0,
    status          VARCHAR(20) DEFAULT 'active',
    balance         JSONB NOT NULL DEFAULT '{}',
        -- { "ytd_contributions": 1095.62,
        --   "ytd_claims_approved": 450.00,
        --   "ytd_claims_pending": 45.00,
        --   "available_balance": 645.62,
        --   "carryover_from_prior": 150.00,
        --   "last_updated": "2026-05-01" }
    transactions    JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"type": "contribution", "amount": 109.62, "date": "2026-01-15",
        --    "pay_period_end": "2026-01-15"},
        --   {"type": "claim", "amount": 45.00, "date": "2026-01-20",
        --    "description": "Pharmacy copay", "status": "approved",
        --    "receipt_url": "s3://..."}
        -- ]
        -- NOTE: For high-volume accounts, transactions may overflow JSONB.
        -- Consider archiving older transactions to a separate table if
        -- a single account exceeds ~500 transactions/year.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_s125_employee ON section_125_account(employee_id);
CREATE INDEX idx_s125_type ON section_125_account(tenant_id, account_type);
```

---

## EDI & Carrier Integration

```sql
-- =============================================================
-- EDI BATCH (unified for 834, 820, and future EDI types)
-- =============================================================
CREATE TABLE edi_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID NOT NULL REFERENCES carrier(id),
    edi_type        VARCHAR(10) NOT NULL,            -- 834 | 820 | 835 | 270 | 271
    direction       VARCHAR(10) NOT NULL,            -- outbound | inbound
    status          VARCHAR(20) DEFAULT 'pending',
    envelope        JSONB NOT NULL DEFAULT '{}',
        -- { "isa_control_number": "000000123",
        --   "gs_control_number": "123",
        --   "transaction_count": 45,
        --   "file_path": "s3://edi/2026/01/834_000123.edi",
        --   "file_hash": "sha256:abc123..." }
    processing      JSONB NOT NULL DEFAULT '{}',
        -- { "sent_at": "2026-01-02T08:00:00Z",
        --   "acknowledged_at": "2026-01-02T10:30:00Z",
        --   "accepted_count": 44, "rejected_count": 1,
        --   "rejections": [{"member_id": "M-123", "reason": "invalid_ssn"}] }
    reconciliation  JSONB DEFAULT NULL,
        -- { "completed_at": "2026-01-05T14:00:00Z",
        --   "matched": 44, "discrepancies": 1,
        --   "details": [{"type": "ghost_enrollment", "member_id": "M-456",
        --                "enrollment_id": "uuid...", "action_taken": "correction_sent"}] }
    raw_content     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi_batch_tenant ON edi_batch(tenant_id, carrier_id);
CREATE INDEX idx_edi_batch_status ON edi_batch(status);
CREATE INDEX idx_edi_batch_type ON edi_batch(edi_type);
```

---

## Payroll & Deductions

```sql
-- =============================================================
-- PAYROLL DEDUCTION BATCH (per pay period)
-- =============================================================
CREATE TABLE payroll_deduction_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    pay_period_start DATE NOT NULL,
    pay_period_end  DATE NOT NULL,
    status          VARCHAR(20) DEFAULT 'draft',
        -- draft | exported | confirmed
    deductions      JSONB NOT NULL DEFAULT '[]',
        -- [
        --   { "employee_id": "uuid...", "employee_number": "EMP-001",
        --     "items": [
        --       {"enrollment_id": "uuid...", "type": "medical_premium",
        --        "is_pretax": true, "employee_amount": 250.00,
        --        "employer_amount": 750.00},
        --       {"election_id": "uuid...", "type": "fsa_health",
        --        "is_pretax": true, "employee_amount": 109.62,
        --        "employer_amount": 0}
        --     ],
        --     "total_employee_pretax": 359.62,
        --     "total_employee_posttax": 0,
        --     "total_employer": 750.00
        --   }
        -- ]
    export_config   JSONB DEFAULT '{}',
        -- { "format": "csv", "payroll_system": "adp",
        --   "exported_at": "2026-01-14T06:00:00Z",
        --   "file_path": "s3://payroll/2026-01-15_deductions.csv" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payroll_batch_tenant ON payroll_deduction_batch(tenant_id);
CREATE INDEX idx_payroll_batch_period ON payroll_deduction_batch(pay_period_start);
```

---

## Audit, Access Control & AI

```sql
-- =============================================================
-- APP USER & ROLES (relational for security)
-- =============================================================
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(200),
    employee_id     UUID REFERENCES employee(id),
    roles           VARCHAR(50)[] NOT NULL DEFAULT '{employee}',
        -- {system_admin, tenant_admin, hr_admin, benefits_manager,
        --  payroll_manager, broker, employee}
    permissions     VARCHAR(100)[] DEFAULT '{}',
        -- Additional fine-grained permissions beyond role defaults
    is_active       BOOLEAN DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_user_email ON app_user(tenant_id, email);
CREATE INDEX idx_user_roles ON app_user USING GIN (roles);

-- =============================================================
-- AUDIT LOG
-- =============================================================
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
        -- { "old": {"status": "pending"}, "new": {"status": "active"},
        --   "fields_changed": ["status"] }
    context         JSONB NOT NULL DEFAULT '{}',
        -- { "ip_address": "10.0.0.1", "user_agent": "Mozilla/5.0...",
        --   "phi_accessed": false, "phi_fields": [] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);

-- =============================================================
-- AI PLAN RECOMMENDATION
-- =============================================================
CREATE TABLE plan_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    enrollment_period_id UUID REFERENCES enrollment_period(id),
    recommendations JSONB NOT NULL,
        -- [
        --   { "plan_id": "uuid...", "rank": 1, "score": 0.92,
        --     "rationale": "Based on your family size and historical utilisation...",
        --     "projected_annual_cost": 4200.00,
        --     "projected_oop_cost": 1800.00,
        --     "comparison": {
        --       "vs_current_plan_savings": 600.00,
        --       "risk_level": "moderate"
        --     }
        --   },
        --   { "plan_id": "uuid...", "rank": 2, ... }
        -- ]
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recommendation_employee ON plan_recommendation(employee_id, enrollment_period_id);
```

---

## Example Queries

### Find all medical plans with deductible under $2000

```sql
SELECT id, plan_name,
       (plan_design->'in_network'->>'individual_deductible')::numeric AS deductible
FROM benefit_plan
WHERE tenant_id = :tenant_id
  AND plan_type = 'medical'
  AND is_active = true
  AND (plan_design->'in_network'->>'individual_deductible')::numeric < 2000
ORDER BY deductible;
```

### Get COBRA case with full payment history

```sql
SELECT id, employee_id, event_type, status,
       lifecycle->'election'->>'monthly_premium' AS monthly_premium,
       jsonb_array_length(lifecycle->'payments') AS payment_count,
       lifecycle->'payments' AS payments
FROM cobra_case
WHERE id = :cobra_case_id;
```

### Find employees with state-specific continuation rights

```sql
SELECT e.id, e.first_name, e.last_name, c.state_continuation
FROM cobra_case c
JOIN employee e ON e.id = c.employee_id
WHERE c.tenant_id = :tenant_id
  AND c.state_continuation IS NOT NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 3 | tenant, employee, dependent |
| Carrier & Plans | 3 | carrier, benefit_plan, plan_rate_history |
| Enrollment | 3 | enrollment_period, life_event, enrollment |
| COBRA | 1 | cobra_case (lifecycle in JSONB) |
| ACA Compliance | 2 | aca_employee_month, aca_filing |
| Section 125 | 1 | section_125_account (transactions in JSONB) |
| EDI | 1 | edi_batch (unified for all EDI types) |
| Payroll | 1 | payroll_deduction_batch |
| Access Control | 1 | app_user (roles as array) |
| Audit | 1 | audit_log |
| AI | 1 | plan_recommendation |
| **Total** | **18** | ~55% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for plan design variance** — Medical, dental, vision, life, FSA, HSA, and LSA plans all share the same `benefit_plan` table. The `plan_design` JSONB column carries type-specific fields. This avoids 8+ separate plan design tables while keeping the core enrollment logic unified.

2. **JSONB for carrier EDI configuration** — Rather than a rigid carrier schema, each carrier's EDI settings (sender/receiver IDs, SFTP paths, segment terminators, custom reference qualifiers) are stored as JSONB. This enables per-carrier customisation without schema changes when onboarding a new carrier with unusual requirements.

3. **COBRA lifecycle as a single JSONB document** — The full COBRA lifecycle (notification, election, payments, termination) is a single document in one row. This simplifies the common query pattern ("show me the full COBRA case") from a 4-table join to a single row fetch. State-specific mini-COBRA rules are a separate JSONB column.

4. **Unified EDI batch table** — All EDI transaction types (834, 820, 835, 270/271) share one table with a `edi_type` discriminator. Processing details and reconciliation results are JSONB. This avoids duplicating identical envelope/status/audit patterns across 4+ tables.

5. **Roles as PostgreSQL arrays** — Rather than a role/user_role/permission junction table chain, roles are stored as a `VARCHAR[]` array on `app_user`. This is simpler for the common case (most users have 1-2 roles) and supports GIN index queries like `WHERE roles @> ARRAY['hr_admin']`.

6. **Addresses as JSONB array on employee** — Most employees have 1-2 addresses. Storing them as a JSONB array avoids a separate table and join while supporting historical addresses via the `effective_date` field in each address object.

7. **Relational where compliance demands it** — ACA monthly tracking uses relational columns for IRS-defined fields (offer indicator, safe harbor code) because these are standardised, fixed-structure, and heavily queried for reporting. The `additional_data` JSONB column handles state-specific extensions.

8. **GIN indexes on queryable JSONB** — Every JSONB column that participates in WHERE clauses gets a GIN index. This ensures JSONB containment queries (`@>`) perform within acceptable bounds even at scale.

9. **Section 125 transactions in JSONB with overflow strategy** — For most employees, FSA/HSA transactions number 20-50 per year, well within JSONB performance. The schema documents the overflow strategy: archive older transactions to a separate table if volume exceeds ~500/year per account.

10. **Demographics JSONB for international flexibility** — Rather than adding columns for every country's national ID format (SSN, NIN, SIN, TFN), the `demographics` JSONB column holds jurisdiction-specific fields. This enables international expansion without migrations.
