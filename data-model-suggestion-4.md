# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Employee Benefits Administration · Created: 2026-05-12

## Philosophy

This model combines a traditional relational schema for operational CRUD with a property graph layer for relationship-intensive queries. The key insight is that benefits administration is fundamentally about relationships: employees are covered by plans offered by carriers, dependents are linked to employees, COBRA events chain through notifications to elections to payments, carrier EDI files reconcile against enrollment records, and ACA compliance connects employees to coverage offers across months. Many of the hardest queries in benefits administration are graph traversals.

The graph layer uses PostgreSQL-native `graph_node` and `graph_edge` tables (a property graph implemented in relational tables, sometimes called an "adjacency list on steroids"). Each entity in the system gets a node; each relationship gets an edge with a type label and JSONB properties. This enables queries like "find all employees whose dependents are enrolled in plans from carrier X that had EDI discrepancies in the last 90 days" — a 5-hop traversal that would require complex multi-table joins in a normalized model but is a simple graph path query.

Real-world precedents include LinkedIn's economic graph (people, companies, skills, jobs connected by edges), healthcare knowledge graphs linking patients to conditions to treatments to providers, and financial compliance systems that model ownership chains and conflict-of-interest networks. For benefits administration, the graph layer is particularly valuable for carrier reconciliation (matching enrollment nodes to EDI nodes), dependent eligibility analysis (traversing family trees), and AI-powered recommendation engines (finding similar employees for plan recommendation).

The relational tables handle day-to-day CRUD operations and compliance reporting. The graph layer provides a queryable relationship index that enables complex analytical queries without multi-table joins. The two layers are synchronised: when an enrollment is created in the relational layer, corresponding nodes and edges are created in the graph layer.

**Best for:** Teams building an analytics-heavy platform where relationship queries (carrier reconciliation, dependent networks, coverage gap analysis, AI recommendation similarity) are a core product differentiator.

**Trade-offs:**
- + Complex relationship queries (5+ hops) are natural and efficient
- + AI recommendation engine can leverage graph similarity (employees with similar profiles/dependents)
- + Carrier reconciliation modeled as graph matching between enrollment and EDI nodes
- + New relationship types require no schema changes — just add edges with new labels
- + Dependency/conflict analysis (e.g., "employee A is dependent on employee B's plan") is trivial
- - Dual-write complexity: relational tables and graph layer must stay synchronised
- - Graph queries require different developer skills (recursive CTEs or graph query libraries)
- - Additional storage overhead for graph nodes/edges duplicating relational foreign keys
- - Less mature tooling for PostgreSQL property graphs compared to dedicated graph databases
- - Operational complexity of maintaining two query paradigms

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ANSI X12 EDI 834 | EDI batches and members are graph nodes; `RECONCILES_TO` edges link EDI member nodes to enrollment nodes for automated matching |
| ACA (1094-C/1095-C) | `COVERED_IN_MONTH` edges between employee and plan nodes enable temporal coverage analysis across tax years |
| ERISA | Audit trail stored relationally; graph layer provides analytical views of enrollment change patterns |
| HIPAA | PHI restricted to relational tables with encryption; graph layer stores only non-PHI identifiers and relationship metadata |
| COBRA | COBRA lifecycle modeled as a directed graph: Employee →[QUALIFYING_EVENT]→ CobraCase →[ELECTED]→ CoverageExtension →[PAID]→ Payment |
| IRS Section 125 | Section 125 accounts linked via `HAS_ACCOUNT` edges; contribution/claim edges enable balance traversal |
| HL7 FHIR R5 | FHIR resources (Coverage, ExplanationOfBenefit) map naturally to graph nodes with typed edges to patient/plan/carrier nodes |
| ISO 3166 | Jurisdiction hierarchy modeled as graph: Country →[HAS_STATE]→ State →[HAS_LOCALITY]→ City; location-based eligibility is a graph traversal |

---

## Relational Layer (Operational CRUD)

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
    settings        JSONB DEFAULT '{}',
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
    date_of_birth   DATE,
    email           VARCHAR(255),
    hire_date       DATE NOT NULL,
    termination_date DATE,
    employment_status VARCHAR(30) NOT NULL DEFAULT 'active',
    fte_status      VARCHAR(20) DEFAULT 'full_time',
    hours_per_week  NUMERIC(5,2),
    annual_salary   NUMERIC(12,2),
    department      VARCHAR(200),
    job_title       VARCHAR(200),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_tenant ON employee(tenant_id);
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
    relationship    VARCHAR(30) NOT NULL,
    ssn_encrypted   BYTEA,
    ssn_hash        VARCHAR(64),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dependent_employee ON dependent(employee_id);

-- =============================================================
-- CARRIER
-- =============================================================
CREATE TABLE carrier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    carrier_code    VARCHAR(50),
    naic_code       VARCHAR(10),
    edi_config      JSONB DEFAULT '{}',
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- BENEFIT PLAN
-- =============================================================
CREATE TABLE benefit_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID REFERENCES carrier(id),
    plan_type       VARCHAR(30) NOT NULL,
    plan_name       VARCHAR(255) NOT NULL,
    group_number    VARCHAR(50),
    plan_year_start DATE NOT NULL,
    plan_year_end   DATE NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    provides_mec    BOOLEAN DEFAULT FALSE,
    provides_mv     BOOLEAN DEFAULT FALSE,
    plan_design     JSONB DEFAULT '{}',
    rates           JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_tenant ON benefit_plan(tenant_id);
CREATE INDEX idx_plan_carrier ON benefit_plan(carrier_id);

-- =============================================================
-- ENROLLMENT
-- =============================================================
CREATE TABLE enrollment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    plan_id         UUID NOT NULL REFERENCES benefit_plan(id),
    coverage_level  VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    effective_date  DATE NOT NULL,
    termination_date DATE,
    employee_cost   NUMERIC(10,2),
    employer_contribution NUMERIC(10,2),
    total_premium   NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enrollment_employee ON enrollment(employee_id);
CREATE INDEX idx_enrollment_plan ON enrollment(plan_id);
CREATE INDEX idx_enrollment_status ON enrollment(tenant_id, status);

-- =============================================================
-- EDI BATCH & MEMBER (for reconciliation)
-- =============================================================
CREATE TABLE edi_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    carrier_id      UUID NOT NULL REFERENCES carrier(id),
    edi_type        VARCHAR(10) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending',
    member_count    INTEGER,
    processing      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE edi_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id        UUID NOT NULL REFERENCES edi_batch(id),
    member_id       VARCHAR(50),
    subscriber_id   VARCHAR(50),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    plan_code       VARCHAR(50),
    effective_date  DATE,
    termination_date DATE,
    maintenance_type VARCHAR(5),
    reconciliation_status VARCHAR(20) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi_member_batch ON edi_member(batch_id);

-- =============================================================
-- COBRA CASE
-- =============================================================
CREATE TABLE cobra_case (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    event_type      VARCHAR(50) NOT NULL,
    event_date      DATE NOT NULL,
    max_continuation_months INTEGER NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending',
    monthly_premium NUMERIC(10,2),
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cobra_employee ON cobra_case(employee_id);

-- =============================================================
-- ACA MONTHLY (relational for IRS reporting)
-- =============================================================
CREATE TABLE aca_employee_month (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    tax_year        INTEGER NOT NULL,
    month           INTEGER NOT NULL,
    hours_of_service NUMERIC(7,2),
    is_fte          BOOLEAN,
    offer_indicator VARCHAR(5),
    employee_share  NUMERIC(10,2),
    safe_harbor_code VARCHAR(5),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(employee_id, tax_year, month)
);

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
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Graph Layer (Analytical Relationship Index)

```sql
-- =============================================================
-- GRAPH NODE (every entity gets a node)
-- =============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
        -- Employee | Dependent | BenefitPlan | Carrier | Enrollment
        -- EdiBatch | EdiMember | CobraCase | Department | Location
        -- EnrollmentPeriod | LifeEvent | AcaMonth
    entity_id       UUID NOT NULL,                   -- FK to the relational table
    label           VARCHAR(255),                    -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',
        -- Denormalised searchable properties:
        -- Employee: { "name": "Jane Doe", "status": "active", "fte": true,
        --             "department": "Engineering", "hire_date": "2026-01-15" }
        -- BenefitPlan: { "name": "Gold PPO", "type": "medical", "mec": true }
        -- Enrollment: { "status": "active", "coverage": "family",
        --               "effective": "2026-01-01" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_node_tenant_type ON graph_node(tenant_id, node_type);
CREATE UNIQUE INDEX idx_node_entity ON graph_node(node_type, entity_id);
CREATE INDEX idx_node_properties ON graph_node USING GIN (properties);

-- =============================================================
-- GRAPH EDGE (typed, directional relationships)
-- =============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(50) NOT NULL,
        -- Relationship types (source → target):
        --
        -- EMPLOYMENT & FAMILY:
        --   Employee →[WORKS_IN]→ Department
        --   Employee →[LOCATED_AT]→ Location
        --   Employee →[HAS_DEPENDENT]→ Dependent
        --   Dependent →[DEPENDENT_OF]→ Employee (reverse edge)
        --
        -- PLAN OFFERING:
        --   Tenant →[OFFERS_PLAN]→ BenefitPlan
        --   BenefitPlan →[ADMINISTERED_BY]→ Carrier
        --   BenefitPlan →[PLAN_YEAR]→ EnrollmentPeriod
        --
        -- ENROLLMENT:
        --   Employee →[ENROLLED_IN]→ BenefitPlan (via Enrollment node)
        --   Employee →[HAS_ENROLLMENT]→ Enrollment
        --   Enrollment →[COVERS_PLAN]→ BenefitPlan
        --   Enrollment →[COVERS_DEPENDENT]→ Dependent
        --
        -- EDI RECONCILIATION:
        --   EdiBatch →[SENT_TO]→ Carrier
        --   EdiBatch →[CONTAINS]→ EdiMember
        --   EdiMember →[RECONCILES_TO]→ Enrollment
        --   EdiMember →[DISCREPANCY_WITH]→ Enrollment
        --
        -- COBRA:
        --   Employee →[HAS_COBRA_CASE]→ CobraCase
        --   CobraCase →[TRIGGERED_BY]→ LifeEvent
        --   CobraCase →[CONTINUES_COVERAGE]→ Enrollment
        --
        -- ACA:
        --   Employee →[COVERED_IN_MONTH]→ AcaMonth
        --   AcaMonth →[COVERAGE_FROM]→ BenefitPlan
        --
        -- AI:
        --   Employee →[SIMILAR_TO]→ Employee (similarity edges for recommendation)
        --   Employee →[RECOMMENDED]→ BenefitPlan
    properties      JSONB NOT NULL DEFAULT '{}',
        -- Edge-specific data:
        -- ENROLLED_IN: { "effective_date": "2026-01-01", "coverage_level": "family",
        --                "status": "active" }
        -- RECONCILES_TO: { "matched_at": "2026-01-05", "match_confidence": 0.98 }
        -- DISCREPANCY_WITH: { "type": "ghost_enrollment", "detected_at": "2026-01-05" }
        -- SIMILAR_TO: { "similarity_score": 0.87, "factors": ["family_size", "age_band", "salary_band"] }
        -- COVERED_IN_MONTH: { "tax_year": 2026, "month": 1, "offer_code": "1E" }
    effective_date  DATE,                            -- when the relationship started
    end_date        DATE,                            -- when the relationship ended (null = current)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_edge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_edge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_edge_temporal ON graph_edge(effective_date, end_date) WHERE end_date IS NULL;
CREATE INDEX idx_edge_properties ON graph_edge USING GIN (properties);
```

---

## Graph Synchronisation

```sql
-- =============================================================
-- Trigger: When an enrollment is created, create graph nodes and edges
-- =============================================================
CREATE OR REPLACE FUNCTION sync_enrollment_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    v_enrollment_node_id UUID;
    v_employee_node_id UUID;
    v_plan_node_id UUID;
BEGIN
    -- Ensure employee node exists
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (NEW.tenant_id, 'Employee', NEW.employee_id,
            (SELECT first_name || ' ' || last_name FROM employee WHERE id = NEW.employee_id),
            jsonb_build_object('status',
                (SELECT employment_status FROM employee WHERE id = NEW.employee_id)))
    ON CONFLICT (node_type, entity_id) DO UPDATE SET updated_at = now()
    RETURNING id INTO v_employee_node_id;

    -- Ensure plan node exists
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (NEW.tenant_id, 'BenefitPlan', NEW.plan_id,
            (SELECT plan_name FROM benefit_plan WHERE id = NEW.plan_id),
            jsonb_build_object('type',
                (SELECT plan_type FROM benefit_plan WHERE id = NEW.plan_id)))
    ON CONFLICT (node_type, entity_id) DO UPDATE SET updated_at = now()
    RETURNING id INTO v_plan_node_id;

    -- Create enrollment node
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (NEW.tenant_id, 'Enrollment', NEW.id,
            'Enrollment',
            jsonb_build_object(
                'status', NEW.status,
                'coverage_level', NEW.coverage_level,
                'effective_date', NEW.effective_date
            ))
    ON CONFLICT (node_type, entity_id) DO UPDATE
        SET properties = EXCLUDED.properties, updated_at = now()
    RETURNING id INTO v_enrollment_node_id;

    -- Create edges
    INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type,
                            effective_date, properties)
    VALUES
        -- Employee →[HAS_ENROLLMENT]→ Enrollment
        (NEW.tenant_id, v_employee_node_id, v_enrollment_node_id, 'HAS_ENROLLMENT',
         NEW.effective_date,
         jsonb_build_object('status', NEW.status)),
        -- Enrollment →[COVERS_PLAN]→ BenefitPlan
        (NEW.tenant_id, v_enrollment_node_id, v_plan_node_id, 'COVERS_PLAN',
         NEW.effective_date,
         jsonb_build_object('coverage_level', NEW.coverage_level))
    ON CONFLICT DO NOTHING;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enrollment_graph_sync
    AFTER INSERT OR UPDATE ON enrollment
    FOR EACH ROW EXECUTE FUNCTION sync_enrollment_to_graph();
```

---

## Example Graph Queries

### Find all employees whose dependents have coverage from a specific carrier

```sql
-- 3-hop traversal: Employee → Dependent → Enrollment → Plan → Carrier
WITH RECURSIVE coverage_path AS (
    -- Start from the carrier
    SELECT gn.id AS node_id, gn.entity_id, gn.node_type, 0 AS depth,
           ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.node_type = 'Carrier' AND gn.entity_id = :carrier_id

    UNION ALL

    -- Traverse backwards through edges
    SELECT gn2.id, gn2.entity_id, gn2.node_type, cp.depth + 1,
           cp.path || gn2.id
    FROM coverage_path cp
    JOIN graph_edge ge ON ge.target_node_id = cp.node_id
    JOIN graph_node gn2 ON gn2.id = ge.source_node_id
    WHERE cp.depth < 5
      AND NOT (gn2.id = ANY(cp.path))  -- prevent cycles
)
SELECT DISTINCT cp.entity_id AS employee_id, e.first_name, e.last_name
FROM coverage_path cp
JOIN employee e ON e.id = cp.entity_id
WHERE cp.node_type = 'Employee';
```

### EDI Reconciliation: Find unmatched EDI members

```sql
-- Find EdiMember nodes that have DISCREPANCY_WITH edges but no RECONCILES_TO edges
SELECT gn.entity_id AS edi_member_id,
       gn.label,
       gn.properties->>'first_name' AS member_name,
       ge.properties->>'type' AS discrepancy_type,
       ge.properties->>'detected_at' AS detected_at
FROM graph_node gn
JOIN graph_edge ge ON ge.source_node_id = gn.id AND ge.edge_type = 'DISCREPANCY_WITH'
WHERE gn.node_type = 'EdiMember'
  AND gn.tenant_id = :tenant_id
  AND NOT EXISTS (
      SELECT 1 FROM graph_edge ge2
      WHERE ge2.source_node_id = gn.id
        AND ge2.edge_type = 'RECONCILES_TO'
  );
```

### AI Recommendation: Find employees similar to a given employee

```sql
-- Similarity graph for plan recommendation
SELECT
    target_node.entity_id AS similar_employee_id,
    e.first_name, e.last_name,
    ge.properties->>'similarity_score' AS similarity,
    ge.properties->'factors' AS similarity_factors
FROM graph_edge ge
JOIN graph_node source_node ON source_node.id = ge.source_node_id
JOIN graph_node target_node ON target_node.id = ge.target_node_id
JOIN employee e ON e.id = target_node.entity_id
WHERE source_node.node_type = 'Employee'
  AND source_node.entity_id = :employee_id
  AND ge.edge_type = 'SIMILAR_TO'
  AND (ge.properties->>'similarity_score')::numeric > 0.7
ORDER BY (ge.properties->>'similarity_score')::numeric DESC
LIMIT 20;

-- Then: find what plans those similar employees chose
SELECT bp.plan_name, bp.plan_type, COUNT(*) AS chosen_count
FROM graph_edge ge_sim
JOIN graph_node sim_node ON sim_node.id = ge_sim.target_node_id
JOIN graph_edge ge_enroll ON ge_enroll.source_node_id = sim_node.id
    AND ge_enroll.edge_type = 'HAS_ENROLLMENT'
    AND ge_enroll.end_date IS NULL
JOIN graph_node enroll_node ON enroll_node.id = ge_enroll.target_node_id
JOIN graph_edge ge_plan ON ge_plan.source_node_id = enroll_node.id
    AND ge_plan.edge_type = 'COVERS_PLAN'
JOIN graph_node plan_node ON plan_node.id = ge_plan.target_node_id
JOIN benefit_plan bp ON bp.id = plan_node.entity_id
WHERE ge_sim.source_node_id = (
    SELECT id FROM graph_node WHERE node_type = 'Employee' AND entity_id = :employee_id
)
AND ge_sim.edge_type = 'SIMILAR_TO'
GROUP BY bp.plan_name, bp.plan_type
ORDER BY chosen_count DESC;
```

### Temporal query: Employee coverage changes over time

```sql
-- All coverage relationships for an employee, including ended ones
SELECT
    ge.edge_type,
    ge.effective_date,
    ge.end_date,
    target_node.node_type AS target_type,
    target_node.label AS target_label,
    ge.properties
FROM graph_node source_node
JOIN graph_edge ge ON ge.source_node_id = source_node.id
JOIN graph_node target_node ON target_node.id = ge.target_node_id
WHERE source_node.node_type = 'Employee'
  AND source_node.entity_id = :employee_id
  AND ge.edge_type IN ('HAS_ENROLLMENT', 'HAS_COBRA_CASE', 'COVERED_IN_MONTH')
ORDER BY ge.effective_date DESC;
```

### COBRA chain: Full lifecycle as a graph path

```sql
-- Traverse the COBRA lifecycle graph
SELECT
    gn.node_type,
    gn.label,
    ge.edge_type,
    ge.properties,
    ge.effective_date
FROM graph_node cobra_node
JOIN graph_edge ge ON ge.source_node_id = cobra_node.id
    OR ge.target_node_id = cobra_node.id
JOIN graph_node gn ON (gn.id = ge.source_node_id OR gn.id = ge.target_node_id)
    AND gn.id != cobra_node.id
WHERE cobra_node.node_type = 'CobraCase'
  AND cobra_node.entity_id = :cobra_case_id
ORDER BY ge.effective_date;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational: Core Identity | 3 | tenant, employee, dependent |
| Relational: Plans & Carriers | 2 | carrier, benefit_plan |
| Relational: Enrollment | 1 | enrollment |
| Relational: EDI | 2 | edi_batch, edi_member |
| Relational: COBRA | 1 | cobra_case |
| Relational: ACA | 1 | aca_employee_month |
| Relational: Audit | 1 | audit_log |
| Graph Layer | 2 | graph_node, graph_edge |
| **Total** | **13** | Plus triggers for graph synchronisation |

---

## Key Design Decisions

1. **PostgreSQL-native property graph** — Rather than introducing a separate graph database (Neo4j, Amazon Neptune), the graph layer uses two PostgreSQL tables (`graph_node`, `graph_edge`). This keeps the entire system in one database, simplifying transactions, backups, and deployments. For teams that outgrow PostgreSQL graph performance, the same schema maps cleanly to Neo4j or Amazon Neptune.

2. **Dual-write via triggers** — Database triggers synchronise the relational and graph layers automatically. When an enrollment is inserted or updated, the trigger creates/updates the corresponding graph nodes and edges. This eliminates application-level dual-write complexity and ensures consistency within a single transaction.

3. **Temporal edges** — Every edge has `effective_date` and `end_date` columns. When an enrollment is terminated, the edge's `end_date` is set rather than deleting the edge. This enables temporal graph queries: "what was the coverage graph on date X?" by filtering edges where `effective_date <= X AND (end_date IS NULL OR end_date > X)`.

4. **Similarity edges for AI** — `SIMILAR_TO` edges between employee nodes store pre-computed similarity scores with the factors that contributed to the similarity (family size, age band, salary band, department). These edges are periodically recomputed by a background job and enable the AI recommendation engine to find "employees like you" without expensive real-time computation.

5. **Reconciliation as graph matching** — EDI reconciliation is modeled as graph matching: `EdiMember` nodes are connected to `Enrollment` nodes via `RECONCILES_TO` (matched) or `DISCREPANCY_WITH` (unmatched) edges. This makes the reconciliation state visible and queryable without complex multi-table joins.

6. **Lean relational layer** — The relational tables are intentionally simpler than in Model 1 (normalized). Section 125 accounts, payroll deductions, enrollment periods, and other secondary entities are not given dedicated tables in this model. They can be added as needed, or their data can be stored as graph node properties. The graph layer handles the relationship complexity that would otherwise require junction tables.

7. **GIN indexes on node and edge properties** — JSONB properties on both nodes and edges are GIN-indexed, enabling efficient property-based filtering within graph traversals (e.g., "find all ENROLLED_IN edges where status = 'active'").

8. **Entity_id links graph to relational** — Every `graph_node` carries an `entity_id` that references the primary key in the corresponding relational table. This enables joining graph query results with relational data for display, export, and compliance reporting. The `UNIQUE(node_type, entity_id)` constraint prevents duplicate nodes.

9. **Edge types as a vocabulary** — The edge type labels (`HAS_ENROLLMENT`, `COVERS_PLAN`, `RECONCILES_TO`, `SIMILAR_TO`) form a controlled vocabulary that acts as the domain's relationship ontology. New relationship types can be added without schema changes — just insert edges with new type labels.

10. **Graph layer is rebuildable** — Like read models in event sourcing, the graph layer can be dropped and reconstructed from the relational tables. This provides a safety net: if the graph becomes inconsistent, a rebuild job can regenerate all nodes and edges from the source-of-truth relational data.
