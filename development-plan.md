# Employee Benefits Administration -- Development Plan

> Project: Employee Benefits Administration (Candidate #84)
> Created: 2026-05-25
> Status: Comprehensive Phased Plan

---

## Technology Decisions & Rationale

### Data Model: Hybrid Relational + JSONB (Model 3) with Event Sourcing for Audit (Model 2)

**Decision:** Adopt the Hybrid Relational + JSONB schema (data-model-suggestion-3) as the primary operational store, augmented with an append-only event log from the Event-Sourced model (data-model-suggestion-2) for audit and compliance.

**Rationale:**
- The hybrid model delivers 18 core tables (~55% fewer than the fully normalized model) while retaining relational integrity where compliance demands it (ACA monthly tracking, enrollment records, employee identity).
- JSONB columns handle the enormous variance across plan types (medical, dental, vision, FSA, HSA, LSA, life, voluntary), carrier EDI configurations, and jurisdiction-specific rules without requiring schema migrations for each new plan type or country.
- ERISA mandates a complete audit trail; HIPAA mandates PHI access logging. An append-only event log satisfies both without the full operational complexity of CQRS projections -- we write events for audit/compliance replay while the hybrid tables serve as the primary read/write path.
- The graph-relational model (suggestion 4) adds dual-write complexity and unfamiliar query patterns for most teams; its benefits (similarity edges, reconciliation-as-graph-matching) are better delivered later as an analytics layer once the operational core is stable.
- The pure normalized model (suggestion 1) at 40 tables creates unnecessary join complexity for common queries and requires migrations every time a new plan type or jurisdiction field is introduced.

### Backend: Node.js (TypeScript) with NestJS

**Rationale:**
- TypeScript provides type safety across the codebase including JSONB schema validation (via Zod or io-ts), which mitigates the weakest aspect of the hybrid model.
- NestJS provides a modular, testable architecture with built-in support for guards (RBAC), interceptors (audit logging), and middleware (tenant isolation) -- all critical for benefits administration.
- The Node.js ecosystem has mature libraries for EDI parsing (x12-parser, node-x12), PDF generation (puppeteer, pdfkit for 1095-C forms), and FHIR (fhir.js).
- Strong hiring pool and contributor base for an open-source project.

### Database: PostgreSQL 16+

**Rationale:**
- First-class JSONB support with GIN indexes, row-level security for multi-tenant isolation, and native UUID generation.
- The hybrid model relies on PostgreSQL-specific features (JSONB containment operators, GIN indexes, array types, range types for temporal queries).
- Supports the append-only event table with trigger-based immutability enforcement.
- Self-hostable with mature operational tooling (pg_dump, pgBackRest, pg_stat_statements), aligning with the HIPAA self-hosting requirement.

### Frontend: Next.js (App Router) with React

**Rationale:**
- Server-side rendering provides fast initial load for the enrollment portal -- critical during high-traffic open enrollment windows.
- React component ecosystem enables rich plan comparison UIs, cost calculators, and enrollment wizards.
- Next.js API routes can serve as a BFF (Backend For Frontend) layer, reducing direct API exposure.
- shadcn/ui for accessible, customizable components that work in self-hosted environments without CDN dependencies.

### AI/ML: LLM-augmented rule engine, not LLM-dependent

**Rationale:**
- Plan recommendations use a transparent, rule-based scoring engine (not opaque ML) to satisfy DOL 2025 ERISA fiduciary guidance -- every recommendation must have a human-readable rationale.
- LLMs augment the platform for SPD summarization, plain-language explanations, and natural-language query interfaces, but are not in the critical path for enrollment decisions.
- The AI layer is abstracted behind a provider interface so deployments can use OpenAI, Anthropic, local models, or disable AI entirely for self-hosted HIPAA environments that prohibit data egress.

### Authentication & Authorization

- OpenID Connect / OAuth 2.0 for SSO (via next-auth or Auth.js)
- SCIM 2.0 for automated employee provisioning from HRIS systems
- Row-level security in PostgreSQL enforcing tenant isolation at the database layer
- Role-based access control: system_admin, tenant_admin, hr_admin, benefits_manager, payroll_manager, broker, employee

### Deployment

- Docker Compose for self-hosted on-premises (primary target -- HIPAA compliance)
- Kubernetes Helm chart for cloud-native self-hosted
- Optional managed SaaS deployment for non-HIPAA customers
- All PHI stays within the deployment perimeter; no external service calls required for core operations

---

## Project Structure

```
employee-benefits-admin/
  apps/
    web/                          # Next.js frontend (enrollment portal + admin console)
      src/
        app/                      # App Router pages
          (employee)/             # Employee-facing enrollment portal
          (admin)/                # HR admin console
          (broker)/               # Broker portal
          api/                    # API routes (BFF)
        components/               # React components
          enrollment/             # Plan selection, comparison, wizard
          compliance/             # ACA dashboard, COBRA tracking
          edi/                    # EDI batch management
          shared/                 # Common UI components
        lib/                      # Frontend utilities
  packages/
    core/                         # Shared domain logic (TypeScript)
      src/
        domain/                   # Domain entities, value objects
          employee/
          enrollment/
          plan/
          carrier/
          cobra/
          aca/
          section125/
        services/                 # Application services
          enrollment-service/
          compliance-service/
          edi-service/
          recommendation-service/
        events/                   # Event definitions and event store
        rules/                    # Eligibility rules engine, ACA rules, Section 125 limits
    api/                          # NestJS API server
      src/
        modules/
          auth/                   # OAuth 2.0 / OIDC / SCIM
          tenant/                 # Multi-tenant management
          employee/               # Employee CRUD + lifecycle
          plan/                   # Plan configuration + rates
          enrollment/             # Enrollment workflows
          carrier/                # Carrier management
          edi/                    # EDI 834/820 generation, parsing, reconciliation
          cobra/                  # COBRA lifecycle management
          aca/                    # ACA compliance + 1095-C generation
          section125/             # Section 125 account management
          payroll/                # Payroll deduction generation
          audit/                  # Audit log + PHI access log
          ai/                     # Recommendation engine + LLM integration
        middleware/               # Tenant isolation, request logging
        guards/                   # RBAC guards
        interceptors/             # Audit logging interceptor
    edi-parser/                   # EDI 834/820/835 parser and generator library
      src/
        parser/                   # X12 segment parsing
        generator/                # X12 file generation
        reconciliation/           # Enrollment-to-EDI matching
    db/                           # Database migrations and seeds
      migrations/
      seeds/
  docker/                         # Docker Compose for self-hosted deployment
  helm/                           # Kubernetes Helm chart
  docs/                           # Architecture decision records, API docs
  e2e/                            # End-to-end Playwright tests
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Plan Management ------+
    |                           |
    v                           v
Phase 3: Enrollment         Phase 4: Compliance (ACA + COBRA)
    |                           |
    +-------+-------------------+
            |
            v
    Phase 5: EDI Integration
            |
            v
    Phase 6: Payroll Integration
            |
            v
    Phase 7: Employee Portal
            |
            v
    Phase 8: AI Decision Support
            |
            v
    Phase 9: Section 125 & Spending Accounts
            |
            v
    Phase 10: Broker Portal & Multi-Carrier Operations
            |
            v
    Phase 11: International Benefits
            |
            v
    Phase 12: Production Hardening & SOC 2
```

**Critical path:** Phases 1 -> 2 -> 3 -> 5 -> 7 (core enrollment flow)
**Compliance path:** Phases 1 -> 2 -> 4 (ACA/COBRA must be available before first open enrollment)
**Parallel opportunities:** Phases 3 and 4 can proceed in parallel after Phase 2.

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the database, multi-tenant architecture, authentication, audit logging, and CI/CD pipeline. No business features -- purely the platform on which everything else is built.

**Duration estimate:** 4-5 weeks

### Task 1.1: Database Schema -- Core Identity & Multi-Tenancy

**What:** Create PostgreSQL schema with the `tenant`, `employee`, `dependent` tables and the append-only `event_store` table. Implement row-level security policies for tenant isolation. Set up migration tooling.

**Design:**

```sql
-- Row-level security for tenant isolation
ALTER TABLE employee ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_employee ON employee
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Repeat for every tenant-scoped table
```

```typescript
// packages/db/migrations/001_core_identity.ts
import { Kysely, sql } from 'kysely';

export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createTable('tenant')
    .addColumn('id', 'uuid', (col) => col.primaryKey().defaultTo(sql`gen_random_uuid()`))
    .addColumn('name', 'varchar(255)', (col) => col.notNull())
    .addColumn('subdomain', 'varchar(63)', (col) => col.unique())
    .addColumn('ein', 'varchar(20)')
    .addColumn('ale_status', 'boolean', (col) => col.defaultTo(false))
    .addColumn('country_code', 'char(2)', (col) => col.defaultTo('US'))
    .addColumn('settings', 'jsonb', (col) => col.notNull().defaultTo(sql`'{}'::jsonb`))
    .addColumn('compliance_config', 'jsonb', (col) => col.notNull().defaultTo(sql`'{}'::jsonb`))
    .addColumn('created_at', 'timestamptz', (col) => col.notNull().defaultTo(sql`now()`))
    .addColumn('updated_at', 'timestamptz', (col) => col.notNull().defaultTo(sql`now()`))
    .execute();

  // employee, dependent, event_store tables follow same pattern
}
```

```typescript
// packages/core/src/events/event-store.ts
export interface DomainEvent {
  eventId: string;
  tenantId: string;
  streamType: string;
  streamId: string;
  eventType: string;
  eventVersion: number;
  eventData: Record<string, unknown>;
  metadata: {
    userId?: string;
    ipAddress?: string;
    correlationId: string;
    causationId?: string;
  };
  schemaVersion: number;
  createdAt: Date;
}

export class EventStore {
  async append(event: Omit<DomainEvent, 'eventId' | 'createdAt'>): Promise<DomainEvent> {
    // Insert into event_store table; the UNIQUE(stream_id, event_version)
    // constraint provides optimistic concurrency control
  }

  async getStream(streamId: string, afterVersion?: number): Promise<DomainEvent[]> {
    // Retrieve all events for a stream, optionally after a given version
  }
}
```

**Testing:**
- Unit test: Verify `tenant` table creation with all columns and constraints
- Unit test: Verify `employee` table foreign key to `tenant` is enforced
- Unit test: Verify `event_store` append-only trigger rejects UPDATE and DELETE
- Unit test: Verify `event_store` optimistic concurrency -- two concurrent writes with same `stream_id` + `event_version` should fail one
- Integration test: Row-level security -- set `app.current_tenant_id` to tenant A, verify queries only return tenant A data
- Integration test: Insert employee for tenant B, verify tenant A RLS policy hides it
- Migration test: Run migration up and down, verify idempotent re-run

### Task 1.2: Authentication & Authorization

**What:** Implement OpenID Connect SSO, API key authentication for service-to-service calls, and role-based access control with the `app_user` table (using PostgreSQL arrays for roles as per hybrid model).

**Design:**

```typescript
// packages/api/src/modules/auth/auth.module.ts
@Module({
  imports: [
    PassportModule,
    JwtModule.register({ secret: process.env.JWT_SECRET, signOptions: { expiresIn: '8h' } }),
  ],
  providers: [OidcStrategy, ApiKeyStrategy, JwtStrategy, AuthService, RbacGuard],
  exports: [AuthService],
})
export class AuthModule {}

// packages/api/src/guards/rbac.guard.ts
@Injectable()
export class RbacGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    const user = context.switchToHttp().getRequest().user;
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}

// Usage:
@Roles('hr_admin', 'benefits_manager')
@UseGuards(RbacGuard)
@Get('employees')
async listEmployees() { /* ... */ }
```

**Testing:**
- Unit test: OIDC strategy correctly extracts user claims and maps to `app_user` record
- Unit test: RbacGuard allows `hr_admin` to access `employees` endpoint
- Unit test: RbacGuard denies `employee` role from accessing admin endpoints
- Unit test: API key strategy validates key and returns associated service account
- Integration test: Full OIDC flow with mock identity provider (using oidc-provider library)
- Integration test: JWT token issued after OIDC login contains correct roles and tenant_id
- Security test: Expired JWT is rejected; malformed JWT returns 401
- Security test: API key for tenant A cannot access tenant B resources

### Task 1.3: Audit Logging Infrastructure

**What:** Implement the `audit_log` table and a NestJS interceptor that automatically captures every mutation with before/after state, user identity, IP address, and PHI access flags.

**Design:**

```typescript
// packages/api/src/interceptors/audit.interceptor.ts
@Injectable()
export class AuditInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const method = request.method;

    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(method)) {
      const beforeState = await this.captureBeforeState(request);
      return next.handle().pipe(
        tap(async (response) => {
          await this.auditService.log({
            tenantId: request.user.tenantId,
            userId: request.user.id,
            action: this.deriveAction(method, request.route),
            entityType: this.deriveEntityType(request.route),
            entityId: this.deriveEntityId(request, response),
            changes: {
              old: beforeState,
              new: response,
              fields_changed: this.diffFields(beforeState, response),
            },
            context: {
              ip_address: request.ip,
              user_agent: request.headers['user-agent'],
              phi_accessed: this.containsPhi(request.route),
              phi_fields: this.identifyPhiFields(request.route, response),
            },
          });
        }),
      );
    }
    return next.handle();
  }
}
```

Additionally, write each audit record as a domain event to the event store:

```typescript
// packages/core/src/events/audit-event-publisher.ts
export class AuditEventPublisher {
  async publishAuditEvent(auditRecord: AuditRecord): Promise<void> {
    await this.eventStore.append({
      tenantId: auditRecord.tenantId,
      streamType: 'AuditTrail',
      streamId: auditRecord.entityId,
      eventType: `audit.${auditRecord.action}`,
      eventVersion: await this.nextVersion(auditRecord.entityId),
      eventData: auditRecord,
      metadata: {
        userId: auditRecord.userId,
        ipAddress: auditRecord.context.ip_address,
        correlationId: auditRecord.correlationId,
      },
      schemaVersion: 1,
    });
  }
}
```

**Testing:**
- Unit test: AuditInterceptor captures before/after state for PUT requests
- Unit test: AuditInterceptor ignores GET requests (no audit for reads, except PHI)
- Unit test: PHI field detection correctly flags SSN, date_of_birth access
- Integration test: Create an employee via API, verify audit_log row exists with correct old (null) and new values
- Integration test: Update employee, verify audit_log captures both old and new values and `fields_changed` array
- Integration test: Verify event_store receives corresponding audit event
- Compliance test: Query audit_log for all actions by a specific user in the last 30 days (ERISA audit trail)
- Performance test: Audit logging adds less than 50ms latency per request

### Task 1.4: CI/CD Pipeline & Docker Setup

**What:** Configure GitHub Actions for lint, type check, unit tests, integration tests (with PostgreSQL in Docker), and build. Create Docker Compose for local development and self-hosted deployment.

**Design:**

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: benefits_admin
      POSTGRES_USER: benefits
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U benefits"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: docker/api.Dockerfile
    environment:
      DATABASE_URL: postgresql://benefits:${DB_PASSWORD}@postgres:5432/benefits_admin
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "3001:3001"

  web:
    build:
      context: .
      dockerfile: docker/web.Dockerfile
    environment:
      NEXT_PUBLIC_API_URL: http://api:3001
    depends_on:
      - api
    ports:
      - "3000:3000"

volumes:
  pgdata:
```

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: benefits_test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 5s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test:unit
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/benefits_test
      - run: npm run build
```

**Testing:**
- Smoke test: `docker compose up` starts all services and API responds to health check within 60 seconds
- Smoke test: `docker compose down && docker compose up` restores state from persisted volume
- CI test: Pipeline runs green on a clean checkout with no test failures
- CI test: TypeScript strict mode (`noUncheckedIndexedAccess`, `strictNullChecks`) passes
- Integration test: Database migrations run automatically on API container startup

### Definition of Done -- Phase 1
- [ ] PostgreSQL database created with `tenant`, `employee`, `dependent`, `app_user`, `event_store`, `audit_log` tables
- [ ] Row-level security policies active on all tenant-scoped tables
- [ ] Append-only trigger on `event_store` prevents mutation
- [ ] OIDC authentication working with at least one IdP (e.g., Auth0 or Keycloak dev instance)
- [ ] RBAC guard enforcing role-based access on all API endpoints
- [ ] Audit interceptor logging all mutations with before/after state
- [ ] Docker Compose starts the full stack locally
- [ ] CI pipeline green with >80% unit test coverage on core and api packages
- [ ] API health check endpoint returning 200 with database connectivity status

---

## Phase 2: Plan & Carrier Management

**Goal:** Enable HR administrators to configure benefit plans, carrier connections, eligibility rules, and rate structures. This is the configuration foundation that enrollment, EDI, and compliance all depend on.

**Duration estimate:** 3-4 weeks

### Task 2.1: Carrier Management CRUD

**What:** Implement the `carrier` table and full CRUD API with carrier-specific EDI configuration stored as JSONB.

**Design:**

```typescript
// packages/core/src/domain/carrier/carrier.schema.ts
import { z } from 'zod';

export const EdiConfigSchema = z.object({
  sender_id: z.string().max(30),
  receiver_id: z.string().max(30),
  version: z.string().default('005010X220A1'),
  transport: z.enum(['sftp', 'as2', 'api']),
  sftp_host: z.string().optional(),
  sftp_path: z.string().optional(),
  file_naming: z.string().default('{tenant}_{date}_{seq}.edi'),
  segment_terminators: z.object({
    element: z.string().default('*'),
    segment: z.string().default('~'),
  }).default({}),
  custom_ref_qualifiers: z.record(z.string()).default({}),
  acknowledgment_expected: z.boolean().default(true),
  reconciliation_frequency: z.enum(['daily', 'weekly', 'monthly']).default('monthly'),
});

export const CarrierSchema = z.object({
  name: z.string().min(1).max(255),
  carrier_code: z.string().max(50).optional(),
  naic_code: z.string().max(10).optional(),
  is_active: z.boolean().default(true),
  contact: z.object({
    name: z.string().optional(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
  }).default({}),
  edi_config: EdiConfigSchema.default({}),
  fhir_config: z.object({
    base_url: z.string().url(),
    auth_type: z.enum(['oauth2', 'api_key']),
    supported_resources: z.array(z.string()),
  }).nullable().default(null),
});
```

```typescript
// packages/api/src/modules/carrier/carrier.controller.ts
@Controller('carriers')
@UseGuards(RbacGuard)
export class CarrierController {
  @Post()
  @Roles('hr_admin', 'benefits_manager')
  async create(@Body() dto: CreateCarrierDto, @CurrentTenant() tenantId: string) {
    const validated = CarrierSchema.parse(dto);
    return this.carrierService.create(tenantId, validated);
  }

  @Get()
  @Roles('hr_admin', 'benefits_manager', 'broker')
  async list(@CurrentTenant() tenantId: string, @Query() filters: CarrierFilterDto) {
    return this.carrierService.list(tenantId, filters);
  }

  @Put(':id')
  @Roles('hr_admin', 'benefits_manager')
  async update(@Param('id') id: string, @Body() dto: UpdateCarrierDto) {
    return this.carrierService.update(id, dto);
  }
}
```

**Testing:**
- Unit test: `CarrierSchema` validates correct EDI config; rejects missing `sender_id`
- Unit test: `EdiConfigSchema` defaults to `005010X220A1` version when omitted
- Integration test: POST `/carriers` creates carrier; GET `/carriers` returns it
- Integration test: PUT `/carriers/:id` updates EDI config; verify JSONB correctly merged
- Integration test: Carrier with `is_active: false` excluded from default list query
- Authorization test: `employee` role cannot create carriers (403)
- Audit test: Carrier creation produces audit_log entry

### Task 2.2: Benefit Plan Configuration

**What:** Implement the `benefit_plan` table with JSONB `plan_design`, `rates`, and `eligibility_rules` columns. Build a plan configuration API with Zod validation that varies by `plan_type`.

**Design:**

```typescript
// packages/core/src/domain/plan/plan-design.schemas.ts
const MedicalDesignSchema = z.object({
  network_type: z.enum(['hmo', 'ppo', 'epo', 'pos', 'hdhp']),
  in_network: z.object({
    individual_deductible: z.number().nonnegative(),
    family_deductible: z.number().nonnegative(),
    individual_oop_max: z.number().nonnegative(),
    family_oop_max: z.number().nonnegative(),
    copay_primary: z.number().nonnegative(),
    copay_specialist: z.number().nonnegative(),
    copay_er: z.number().nonnegative(),
    coinsurance_pct: z.number().min(0).max(100),
    rx: z.object({
      generic: z.number().nonnegative(),
      brand: z.number().nonnegative(),
      specialty: z.number().nonnegative(),
    }),
  }),
  out_of_network: z.object({ /* similar structure */ }).optional(),
});

const DentalDesignSchema = z.object({
  annual_max: z.number().nonnegative(),
  preventive_coverage_pct: z.number().min(0).max(100),
  basic_coverage_pct: z.number().min(0).max(100),
  major_coverage_pct: z.number().min(0).max(100),
  ortho_coverage_pct: z.number().min(0).max(100).optional(),
  ortho_lifetime_max: z.number().nonnegative().optional(),
});

// Discriminated union by plan_type
export function validatePlanDesign(planType: string, design: unknown) {
  switch (planType) {
    case 'medical': return MedicalDesignSchema.parse(design);
    case 'dental': return DentalDesignSchema.parse(design);
    case 'vision': return VisionDesignSchema.parse(design);
    // ... additional plan types
    default: return z.record(z.unknown()).parse(design); // flexible for new types
  }
}
```

```typescript
// packages/core/src/domain/plan/eligibility-rule.schema.ts
export const EligibilityRuleSchema = z.object({
  field: z.enum(['fte_status', 'employment_class', 'hire_date', 'age', 'location', 'department', 'union_membership']),
  operator: z.enum(['eq', 'neq', 'gt', 'lt', 'gte', 'lte', 'in', 'not_in']),
  value: z.union([z.string(), z.number(), z.array(z.string())]),
});

export const EligibilityRulesSchema = z.array(EligibilityRuleSchema);
```

```typescript
// packages/core/src/rules/eligibility-engine.ts
export class EligibilityEngine {
  evaluate(employee: Employee, rules: EligibilityRule[]): boolean {
    return rules.every((rule) => this.evaluateRule(employee, rule));
  }

  private evaluateRule(employee: Employee, rule: EligibilityRule): boolean {
    const fieldValue = this.getFieldValue(employee, rule.field);
    switch (rule.operator) {
      case 'eq': return fieldValue === rule.value;
      case 'in': return (rule.value as string[]).includes(fieldValue as string);
      case 'gte': return (fieldValue as number) >= (rule.value as number);
      // ... other operators
    }
  }
}
```

**Testing:**
- Unit test: `MedicalDesignSchema` validates a correct Gold PPO plan design
- Unit test: `MedicalDesignSchema` rejects negative deductible
- Unit test: `DentalDesignSchema` validates with optional ortho fields omitted
- Unit test: `validatePlanDesign('medical', {...})` dispatches to correct schema
- Unit test: `EligibilityEngine.evaluate` returns true when employee matches all rules
- Unit test: `EligibilityEngine.evaluate` returns false when employee fails any rule
- Unit test: Eligibility rule with `in` operator matches employee in allowed list
- Integration test: POST `/plans` creates a medical plan with full design, rates, and eligibility rules
- Integration test: GET `/plans?plan_type=medical` returns only medical plans
- Integration test: GET `/plans/:id/eligible-employees` returns employees matching eligibility rules
- Integration test: Plan rate history recorded in `plan_rate_history` when rates updated
- Validation test: Plan with `plan_year_end` before `plan_year_start` is rejected (400)

### Task 2.3: Plan Rate Management

**What:** Implement rate configuration per coverage level with temporal tracking via `plan_rate_history`.

**Design:**

```typescript
// packages/core/src/domain/plan/rate.schema.ts
export const RateSchema = z.object({
  employee_only: RateTierSchema,
  employee_spouse: RateTierSchema.optional(),
  employee_children: RateTierSchema.optional(),
  family: RateTierSchema.optional(),
  effective_date: z.string().date(),
});

const RateTierSchema = z.object({
  total: z.number().positive(),
  employer: z.number().nonnegative(),
  employee: z.number().nonnegative(),
}).refine(
  (data) => Math.abs(data.total - data.employer - data.employee) < 0.01,
  { message: 'Total premium must equal employer + employee contributions' }
);
```

```typescript
// packages/api/src/modules/plan/plan.service.ts
async updateRates(planId: string, newRates: RateInput): Promise<void> {
  await this.db.transaction(async (trx) => {
    const currentPlan = await trx.selectFrom('benefit_plan').where('id', '=', planId).executeTakeFirst();

    // Archive current rates to history
    await trx.insertInto('plan_rate_history').values({
      plan_id: planId,
      rates: currentPlan.rates,
      effective_date: currentPlan.rates.effective_date,
      end_date: newRates.effective_date,
    }).execute();

    // Update current rates
    await trx.updateTable('benefit_plan').set({ rates: newRates }).where('id', '=', planId).execute();

    // Publish event
    await this.eventStore.append({
      streamType: 'BenefitPlan',
      streamId: planId,
      eventType: 'PlanRateSet',
      eventData: { previous_rates: currentPlan.rates, new_rates: newRates },
    });
  });
}
```

**Testing:**
- Unit test: `RateTierSchema` rejects when total != employer + employee
- Unit test: `RateTierSchema` accepts when total == employer + employee within $0.01 tolerance
- Integration test: Update rates on a plan; verify `plan_rate_history` row created with old rates
- Integration test: Query "what was the rate on date X" returns correct historical rate
- Integration test: Rate change produces `PlanRateSet` event in event store

### Definition of Done -- Phase 2
- [ ] Carrier CRUD API with EDI and FHIR configuration
- [ ] Benefit plan CRUD with type-specific JSONB validation for medical, dental, vision, life, FSA, HSA
- [ ] Eligibility rules stored as JSONB, evaluated by EligibilityEngine
- [ ] Plan rates with temporal history tracking
- [ ] Admin UI pages for carrier and plan management
- [ ] >85% test coverage on plan validation and eligibility rule evaluation

---

## Phase 3: Enrollment Workflows

**Goal:** Implement the core enrollment lifecycle: open enrollment periods, new-hire enrollment, life events, enrollment elections, and dependent coverage. This is the central business capability.

**Duration estimate:** 5-6 weeks

### Task 3.1: Enrollment Period Management

**What:** Implement `enrollment_period` table and API for HR admins to create, configure, and manage open enrollment windows with configurable reminder schedules.

**Design:**

```typescript
// packages/api/src/modules/enrollment/enrollment-period.service.ts
export class EnrollmentPeriodService {
  async create(tenantId: string, input: CreateEnrollmentPeriodInput): Promise<EnrollmentPeriod> {
    const period = await this.db.insertInto('enrollment_period').values({
      tenant_id: tenantId,
      name: input.name,
      period_type: input.period_type, // open_enrollment | special_enrollment | new_hire
      start_date: input.start_date,
      end_date: input.end_date,
      plan_year_start: input.plan_year_start,
      plan_year_end: input.plan_year_end,
      status: 'scheduled',
      config: {
        reminder_schedule: input.reminder_schedule ?? ['7d_before_end', '3d_before_end', '1d_before_end'],
        auto_enroll_defaults: input.auto_enroll_defaults ?? false,
        require_waiver_reason: input.require_waiver_reason ?? true,
      },
    }).returningAll().executeTakeFirst();

    await this.eventStore.append({
      streamType: 'EnrollmentPeriod',
      streamId: period.id,
      eventType: 'EnrollmentPeriodCreated',
      eventData: period,
    });

    return period;
  }

  async activate(periodId: string): Promise<void> {
    // Validates: no overlapping active periods for same tenant
    // Sets status to 'active'
    // Schedules reminder notifications
  }

  async close(periodId: string): Promise<void> {
    // Sets status to 'closed'
    // Triggers auto-enrollment for employees who did not make elections (if configured)
    // Generates enrollment summary report
  }
}
```

**Testing:**
- Unit test: Cannot create enrollment period with `end_date` before `start_date`
- Unit test: Cannot activate period when another period with overlapping dates is already active
- Integration test: Create period, activate it, verify status transitions correctly
- Integration test: Close period, verify employees who did not enroll are flagged
- Integration test: Enrollment period CRUD produces corresponding events in event store

### Task 3.2: Core Enrollment Election

**What:** Implement the `enrollment` table and the enrollment election flow: employee selects a plan, chooses coverage level, adds dependents, and confirms election. Enforce eligibility rules and Section 125 pre-tax constraints.

**Design:**

```typescript
// packages/core/src/services/enrollment-service/enrollment-service.ts
export class EnrollmentService {
  async initiateEnrollment(input: InitiateEnrollmentInput): Promise<Enrollment> {
    // 1. Verify enrollment period is active
    const period = await this.enrollmentPeriodRepo.findActive(input.tenantId);
    if (!period) throw new BusinessError('NO_ACTIVE_ENROLLMENT_PERIOD');

    // 2. Verify employee eligibility for this plan
    const plan = await this.planRepo.findById(input.planId);
    const employee = await this.employeeRepo.findById(input.employeeId);
    const eligible = this.eligibilityEngine.evaluate(employee, plan.eligibility_rules);
    if (!eligible) throw new BusinessError('EMPLOYEE_NOT_ELIGIBLE', { planId: plan.id });

    // 3. Validate coverage level
    if (!plan.coverage_levels.includes(input.coverageLevel)) {
      throw new BusinessError('INVALID_COVERAGE_LEVEL');
    }

    // 4. Validate dependents (if coverage level includes dependents)
    if (['employee_spouse', 'employee_children', 'family'].includes(input.coverageLevel)) {
      await this.validateDependents(input.employeeId, input.dependentIds, input.coverageLevel);
    }

    // 5. Calculate costs from plan rates
    const rateTier = plan.rates[input.coverageLevel];
    if (!rateTier) throw new BusinessError('NO_RATE_FOR_COVERAGE_LEVEL');

    // 6. Create enrollment record
    const enrollment = await this.db.insertInto('enrollment').values({
      tenant_id: input.tenantId,
      employee_id: input.employeeId,
      plan_id: input.planId,
      enrollment_period_id: period.id,
      life_event_id: input.lifeEventId ?? null,
      coverage_level: input.coverageLevel,
      status: 'pending',
      effective_date: period.plan_year_start,
      costs: {
        employee_per_period: rateTier.employee,
        employer_per_period: rateTier.employer,
        total_premium_per_period: rateTier.total,
        annual_employee_cost: rateTier.employee * 12,
        annual_employer_cost: rateTier.employer * 12,
      },
      covered_dependents: input.dependentIds ?? [],
      election_metadata: {
        elected_at: new Date().toISOString(),
        elected_by: input.userId,
        election_source: input.source ?? 'employee_portal',
      },
    }).returningAll().executeTakeFirst();

    // 7. Publish event
    await this.eventStore.append({
      streamType: 'Enrollment',
      streamId: enrollment.id,
      eventType: 'EnrollmentInitiated',
      eventData: { ...enrollment },
    });

    return enrollment;
  }

  async confirmEnrollment(enrollmentId: string): Promise<Enrollment> {
    // Transitions status from 'pending' to 'active'
    // Validates no conflicting active enrollments for same plan type
    // Publishes EnrollmentConfirmed event
  }

  async waiveEnrollment(input: WaiveEnrollmentInput): Promise<void> {
    // Records explicit waiver with reason
    // Publishes EnrollmentWaived event
    // Required for ACA compliance tracking
  }

  async terminateEnrollment(enrollmentId: string, terminationDate: Date, reason: string): Promise<void> {
    // Sets termination_date and status to 'terminated'
    // Checks COBRA eligibility
    // Publishes EnrollmentTerminated event
  }
}
```

**Testing:**
- Unit test: `initiateEnrollment` rejects when no active enrollment period
- Unit test: `initiateEnrollment` rejects when employee fails eligibility rules
- Unit test: `initiateEnrollment` rejects invalid coverage level
- Unit test: Cost calculation matches plan rate tier for selected coverage level
- Unit test: Cannot have two active enrollments for the same plan type (e.g., two medical plans)
- Unit test: Dependent validation rejects employee_only coverage with dependents
- Unit test: Dependent validation rejects family coverage with no dependents
- Integration test: Full enrollment flow -- initiate, confirm, verify active status
- Integration test: Waive enrollment, verify waiver record created with reason
- Integration test: Terminate enrollment, verify termination_date set and event published
- Integration test: Terminate enrollment for COBRA-eligible event triggers COBRA event (Phase 4 integration point)
- Event test: Each enrollment state transition produces the correct event type in event store

### Task 3.3: Life Event Workflows

**What:** Implement life event reporting and special enrollment windows. Employees report qualifying life events (marriage, birth, divorce, loss of coverage) which trigger a special enrollment period allowing mid-year plan changes.

**Design:**

```typescript
// packages/core/src/services/enrollment-service/life-event-service.ts
export class LifeEventService {
  private static readonly ENROLLMENT_DEADLINES: Record<string, number> = {
    marriage: 30,
    birth: 30,
    adoption: 30,
    divorce: 30,
    death_of_dependent: 30,
    loss_of_coverage: 60,
    gain_of_coverage: 30,
    address_change: 30,
    employment_status_change: 30,
    medicare_eligibility: 60,
  };

  async reportLifeEvent(input: ReportLifeEventInput): Promise<LifeEvent> {
    const deadlineDays = LifeEventService.ENROLLMENT_DEADLINES[input.eventType];
    const deadline = addDays(new Date(input.eventDate), deadlineDays);

    const lifeEvent = await this.db.insertInto('life_event').values({
      tenant_id: input.tenantId,
      employee_id: input.employeeId,
      event_type: input.eventType,
      event_date: input.eventDate,
      special_enrollment_deadline: deadline,
      status: 'pending',
      details: {
        documentation_received: false,
        notes: input.notes,
      },
    }).returningAll().executeTakeFirst();

    // Publish event
    await this.eventStore.append({
      streamType: 'Employee',
      streamId: input.employeeId,
      eventType: 'LifeEventReported',
      eventData: lifeEvent,
    });

    return lifeEvent;
  }

  async approveLifeEvent(lifeEventId: string, approvedBy: string): Promise<void> {
    // Validate documentation received
    // Set status to 'approved'
    // Opens special enrollment window for the employee
    // Publish LifeEventApproved event
  }
}
```

**Testing:**
- Unit test: Marriage event gets 30-day special enrollment deadline
- Unit test: Loss of coverage event gets 60-day deadline
- Unit test: Life event with past deadline (reported_date > event_date + deadline) is flagged as late
- Integration test: Report life event, approve it, verify employee can now make enrollment changes
- Integration test: Denied life event prevents enrollment changes
- Integration test: Expired (unapproved) life event auto-transitions to 'expired' status

### Task 3.4: Enrollment Admin Dashboard

**What:** Build the HR admin interface for managing enrollment periods, monitoring enrollment progress, and reviewing life events.

**Design:**

```typescript
// apps/web/src/app/(admin)/enrollment/page.tsx
// Server component that fetches enrollment period status and progress metrics
export default async function EnrollmentDashboard() {
  const period = await getActiveEnrollmentPeriod();
  const stats = await getEnrollmentStats(period.id);

  return (
    <div>
      <EnrollmentPeriodHeader period={period} />
      <EnrollmentProgressBar
        total={stats.totalEligible}
        enrolled={stats.enrolled}
        waived={stats.waived}
        pending={stats.pending}
      />
      <NotEnrolledList employees={stats.notEnrolled} />
      <PendingLifeEvents events={stats.pendingLifeEvents} />
    </div>
  );
}
```

**Testing:**
- E2E test (Playwright): HR admin navigates to enrollment dashboard, sees progress bar
- E2E test: HR admin creates a new enrollment period with date range and plan selection
- E2E test: HR admin activates enrollment period, sees status change to 'active'
- E2E test: HR admin approves a life event from the pending list
- E2E test: HR admin closes enrollment period, sees summary report

### Definition of Done -- Phase 3
- [ ] Enrollment period CRUD with status lifecycle (scheduled -> active -> closed)
- [ ] Core enrollment election: initiate, confirm, waive, terminate
- [ ] Eligibility rule enforcement at enrollment time
- [ ] Dependent coverage management within enrollments
- [ ] Life event reporting, approval, and special enrollment windows
- [ ] Enrollment admin dashboard with progress tracking
- [ ] All enrollment state transitions produce domain events
- [ ] >85% test coverage on enrollment service and life event service

---

## Phase 4: Compliance Engine (ACA + COBRA)

**Goal:** Implement ACA compliance tracking and 1095-C generation, plus COBRA qualifying event detection, notice generation, and continuation coverage management. Compliance is a primary purchase driver.

**Duration estimate:** 5-6 weeks

### Task 4.1: ACA Eligibility Tracking

**What:** Implement `aca_employee_month` tracking, ALE determination, and look-back measurement period logic for variable-hour employees.

**Design:**

```typescript
// packages/core/src/services/compliance-service/aca-service.ts
export class AcaService {
  async recordMonthlyStatus(input: AcaMonthlyInput): Promise<void> {
    const offerIndicator = this.determineOfferIndicator(input);
    const safeHarbor = this.determineSafeHarborCode(input);

    await this.db.insertInto('aca_employee_month').values({
      tenant_id: input.tenantId,
      employee_id: input.employeeId,
      tax_year: input.taxYear,
      month: input.month,
      hours_of_service: input.hoursOfService,
      is_fte: input.hoursOfService >= 130, // IRS FTE threshold: 130 hours/month
      offer_indicator: offerIndicator,
      employee_share_lowest_cost: input.employeeShareLowestCost,
      safe_harbor_code: safeHarbor,
      additional_data: input.stateSpecificData ?? {},
    }).onConflict((oc) =>
      oc.columns(['employee_id', 'tax_year', 'month']).doUpdateSet({
        hours_of_service: input.hoursOfService,
        offer_indicator: offerIndicator,
        safe_harbor_code: safeHarbor,
        updated_at: sql`now()`,
      })
    ).execute();
  }

  // IRS Line 14 indicator codes
  private determineOfferIndicator(input: AcaMonthlyInput): string {
    if (!input.coverageOffered) return '1H'; // No offer of coverage
    if (input.coverageLevel === 'employee_only') return '1B'; // MEC to employee only
    if (input.coverageLevel === 'family') return '1E'; // MEC to employee, spouse, and dependents
    // ... additional codes 1A through 1S
    return '1A'; // Qualifying offer
  }

  async calculateAleDetermination(tenantId: string, taxYear: number): Promise<AleResult> {
    // Count FTEs + FTE-equivalents for each month of the prior year
    // ALE threshold: average of 50+ FTEs across all 12 months
    const monthlyFteCounts = await this.db
      .selectFrom('aca_employee_month')
      .where('tenant_id', '=', tenantId)
      .where('tax_year', '=', taxYear - 1)
      .groupBy('month')
      .select([
        'month',
        sql<number>`COUNT(*) FILTER (WHERE is_fte = true)`.as('fte_count'),
        sql<number>`SUM(hours_of_service) FILTER (WHERE is_fte = false) / 120`.as('fte_equivalent'),
      ])
      .execute();

    const averageFtes = monthlyFteCounts.reduce(
      (sum, m) => sum + m.fte_count + m.fte_equivalent, 0
    ) / 12;

    return { isAle: averageFtes >= 50, averageFtes, monthlyBreakdown: monthlyFteCounts };
  }
}
```

**Testing:**
- Unit test: Employee with 130+ hours in a month is classified as FTE
- Unit test: Employee with 100 hours is non-FTE
- Unit test: Offer indicator `1E` assigned for family coverage offer
- Unit test: Offer indicator `1H` assigned when no coverage offered
- Unit test: ALE determination returns true when average FTEs >= 50
- Unit test: ALE determination correctly calculates FTE-equivalents from part-time hours
- Integration test: Record 12 months of ACA data for 60 employees; verify ALE determination = true
- Integration test: Record data for 40 employees; verify ALE = false
- Compliance test: Variable-hour employee look-back period correctly determines FTE status using measurement period

### Task 4.2: 1095-C Generation

**What:** Generate IRS Form 1095-C from `aca_employee_month` data. Produce both PDF forms for employee distribution and XML for IRS AIR electronic filing.

**Design:**

```typescript
// packages/core/src/services/compliance-service/aca-1095c-generator.ts
export class Aca1095cGenerator {
  async generateForTaxYear(tenantId: string, taxYear: number): Promise<Aca1095cBatch> {
    const employees = await this.getAcaEmployees(tenantId, taxYear);
    const filing = await this.createFiling(tenantId, taxYear, employees.length);

    const forms: Aca1095cForm[] = [];
    for (const employee of employees) {
      const monthlyData = await this.getMonthlyData(employee.id, taxYear);
      const form = this.buildForm(employee, monthlyData);
      forms.push(form);
    }

    // Generate PDF for employee furnishing
    const pdfs = await this.pdfGenerator.generateBatch(forms);

    // Generate XML for IRS AIR filing
    const airXml = this.airXmlGenerator.generate(filing, forms);

    return { filing, forms, pdfs, airXml };
  }

  private buildForm(employee: Employee, monthlyData: AcaEmployeeMonth[]): Aca1095cForm {
    return {
      employeeName: `${employee.first_name} ${employee.last_name}`,
      employeeSsn: employee.ssn_encrypted, // decrypt at render time only
      employerEin: this.tenant.ein,
      line14: Object.fromEntries(monthlyData.map(m => [`month_${m.month}`, m.offer_indicator])),
      line15: Object.fromEntries(monthlyData.map(m => [`month_${m.month}`, m.employee_share_lowest_cost])),
      line16: Object.fromEntries(monthlyData.map(m => [`month_${m.month}`, m.safe_harbor_code])),
    };
  }
}
```

**Testing:**
- Unit test: `buildForm` maps monthly data to correct Line 14/15/16 fields
- Unit test: Form correctly handles mid-year hires (blank months before hire date)
- Unit test: Form correctly handles mid-year terminations
- Integration test: Generate 1095-C batch for a tenant with 100 employees; verify all forms produced
- Compliance test: Verify generated form matches IRS Form 1095-C field layout exactly
- PDF test: Generated PDF is readable and contains correct employee data
- XML test: Generated AIR XML validates against IRS schema

### Task 4.3: COBRA Lifecycle Management

**What:** Implement `cobra_case` table and the full COBRA lifecycle: qualifying event detection, notice generation, election tracking, premium billing, and termination.

**Design:**

```typescript
// packages/core/src/services/compliance-service/cobra-service.ts
export class CobraService {
  async handleQualifyingEvent(input: CobraQualifyingEventInput): Promise<CobraCase> {
    const continuationMonths = this.getContinuationPeriod(input.eventType);
    const notificationDeadline = addDays(input.eventDate, 30);

    const cobraCase = await this.db.insertInto('cobra_case').values({
      tenant_id: input.tenantId,
      employee_id: input.employeeId,
      event_type: input.eventType,
      event_date: input.eventDate,
      max_continuation_months: continuationMonths,
      status: 'pending',
      lifecycle: {
        notification: {
          deadline: notificationDeadline.toISOString(),
          sent_date: null,
          sent_to: null,
          method: null,
        },
        election: null,
        payments: [],
        termination: null,
      },
      state_continuation: await this.getStateContinuation(input.tenantId, input.employeeId),
    }).returningAll().executeTakeFirst();

    await this.eventStore.append({
      streamType: 'CobraCase',
      streamId: cobraCase.id,
      eventType: 'CobraQualifyingEventOccurred',
      eventData: cobraCase,
    });

    return cobraCase;
  }

  async sendNotice(caseId: string): Promise<void> {
    const cobraCase = await this.findById(caseId);
    const electionDeadline = addDays(new Date(), 60);

    // Generate COBRA election notice (DOL model notice)
    const notice = await this.noticeGenerator.generateElectionNotice(cobraCase);

    // Send via email
    await this.emailService.send(cobraCase.employee.email, notice);

    // Update lifecycle JSONB
    await this.updateLifecycle(caseId, {
      notification: {
        ...cobraCase.lifecycle.notification,
        sent_date: new Date().toISOString(),
        sent_to: cobraCase.employee.email,
        method: 'email',
      },
      election: {
        deadline: electionDeadline.toISOString(),
      },
    });

    // Transition status
    await this.updateStatus(caseId, 'notified');
  }

  async recordElection(caseId: string, plansElected: string[]): Promise<void> {
    // Calculate monthly premium (102% of full premium -- 2% admin fee per COBRA)
    // Transition enrollments to COBRA status
    // Set continuation end date
    // Generate payment schedule
  }

  async recordPayment(caseId: string, paymentPeriod: string, amount: number): Promise<void> {
    // Apply payment to next due period
    // Check grace period compliance (30-day grace period)
    // If payment lapses past grace period, terminate COBRA coverage
  }

  private getContinuationPeriod(eventType: string): number {
    switch (eventType) {
      case 'termination':
      case 'reduction_in_hours':
        return 18;
      case 'death':
      case 'divorce':
      case 'medicare_entitlement':
      case 'dependent_aging_out':
        return 36;
      default:
        return 18;
    }
  }

  private async getStateContinuation(tenantId: string, employeeId: string): Promise<object | null> {
    // Check tenant's compliance_config for state-specific mini-COBRA laws
    // e.g., Cal-COBRA extends to 36 months for employers with 2-19 employees
    // e.g., NY Continuation extends to 36 months for all sizes
  }
}
```

**Testing:**
- Unit test: Termination event gets 18-month continuation
- Unit test: Divorce event gets 36-month continuation
- Unit test: Notification deadline is 30 days from event date
- Unit test: Election deadline is 60 days from notice date
- Unit test: COBRA premium is 102% of full premium
- Unit test: Payment within 30-day grace period is accepted
- Unit test: Payment after grace period triggers coverage termination
- Integration test: Full COBRA lifecycle: qualifying event -> notice -> election -> 3 months payments -> termination
- Integration test: State mini-COBRA (Cal-COBRA) extends continuation period correctly
- Integration test: COBRA case lifecycle JSONB accurately tracks all dates and statuses
- Event test: Each COBRA state transition produces the correct event type

### Definition of Done -- Phase 4
- [ ] ACA monthly tracking with FTE determination and offer indicator codes
- [ ] ALE determination calculation based on 12-month look-back
- [ ] 1095-C form generation (PDF and IRS AIR XML)
- [ ] ACA filing status tracking (draft -> generated -> filed -> accepted/rejected)
- [ ] COBRA qualifying event detection and case creation
- [ ] COBRA notice generation using DOL model notice template
- [ ] COBRA election, premium billing, payment tracking, and grace period enforcement
- [ ] State mini-COBRA law support (at least California and New York)
- [ ] All compliance actions produce audit trail events
- [ ] >90% test coverage on ACA and COBRA services (compliance code requires highest coverage)

---

## Phase 5: EDI 834 Carrier Integration

**Goal:** Parse, generate, and reconcile ANSI X12 EDI 834 benefit enrollment files. This is the mechanism by which enrollment data reaches insurance carriers and is the most expensive/error-prone part of benefits administration.

**Duration estimate:** 5-6 weeks

### Task 5.1: EDI 834 Parser

**What:** Build a parser for inbound EDI 834 files that extracts ISA/GS/ST envelopes and INS/NM1/HD/REF segments into structured data.

**Design:**

```typescript
// packages/edi-parser/src/parser/x12-834-parser.ts
export interface Edi834ParseResult {
  interchange: {
    controlNumber: string;
    senderId: string;
    receiverId: string;
    date: string;
  };
  functionalGroup: {
    controlNumber: string;
  };
  transactions: Edi834Transaction[];
  errors: Edi834ParseError[];
}

export interface Edi834Transaction {
  members: Edi834Member[];
}

export interface Edi834Member {
  insSegment: {
    subscriberIndicator: string;  // Y = subscriber, N = dependent
    maintenanceType: string;      // 021 = addition, 024 = cancellation, 001 = change
    maintenanceReason: string;
    benefitStatus: string;        // A = active, C = cobra, T = terminated
  };
  names: Array<{
    entityCode: string;    // IL = insured, QD = dependent
    lastName: string;
    firstName: string;
    middleName?: string;
    suffix?: string;
    idQualifier: string;
    idValue: string;       // SSN or member ID
  }>;
  healthCoverage: Array<{
    maintenanceType: string;
    insuranceLineCode: string;  // HLT = health, DEN = dental, VIS = vision
    coverageLevelCode: string;  // EMP = employee, FAM = family, etc.
    effectiveDate: string;
    terminationDate?: string;
  }>;
  references: Array<{
    qualifier: string;     // 0F = subscriber ID, 1L = group number, etc.
    value: string;
  }>;
  demographics: {
    dateOfBirth?: string;
    gender?: string;
  };
}

export class X12834Parser {
  parse(rawEdi: string): Edi834ParseResult {
    const segments = this.tokenize(rawEdi);
    // Walk through ISA -> GS -> ST -> loop structure
    // Extract INS + subordinate segments for each member
  }

  private tokenize(raw: string): string[][] {
    const segmentTerminator = raw.charAt(105) || '~'; // ISA16 defines segment terminator
    const elementSeparator = raw.charAt(3) || '*';     // ISA field 4 position
    return raw.split(segmentTerminator)
      .filter(s => s.trim().length > 0)
      .map(s => s.trim().split(elementSeparator));
  }
}
```

**Testing:**
- Unit test: Parse a minimal valid EDI 834 file with one member addition
- Unit test: Parse a file with subscriber + two dependents
- Unit test: Parse a cancellation (maintenance type 024) correctly
- Unit test: Handle non-standard segment terminators (configurable per carrier)
- Unit test: Return parse errors for malformed segments without crashing
- Unit test: Extract all reference segments (subscriber ID, group number, plan code)
- Integration test: Parse a real-world carrier 834 acknowledgment file
- Compliance test: Verify parsing aligns with X12 5010X220A1 implementation guide

### Task 5.2: EDI 834 Generator

**What:** Generate outbound EDI 834 files from enrollment data, using carrier-specific configuration from the `carrier.edi_config` JSONB.

**Design:**

```typescript
// packages/edi-parser/src/generator/x12-834-generator.ts
export class X12834Generator {
  generate(batch: Edi834BatchInput): string {
    const segments: string[] = [];
    const config = batch.carrierConfig;

    // ISA - Interchange Control Header
    segments.push(this.buildIsa(config, batch.controlNumber));

    // GS - Functional Group Header
    segments.push(this.buildGs(config, batch.groupControlNumber));

    // ST - Transaction Set Header
    segments.push(`ST${SEP}834${SEP}${batch.transactionControlNumber}`);

    // BGN - Beginning Segment
    segments.push(`BGN${SEP}00${SEP}${batch.referenceId}${SEP}${formatDate(new Date())}`);

    // Loop through members
    for (const member of batch.members) {
      segments.push(...this.buildMemberSegments(member, config));
    }

    // SE/GE/IEA trailers
    segments.push(`SE${SEP}${segments.length}${SEP}${batch.transactionControlNumber}`);
    segments.push(`GE${SEP}1${SEP}${batch.groupControlNumber}`);
    segments.push(`IEA${SEP}1${SEP}${batch.controlNumber}`);

    return segments.join(config.segment_terminators.segment) +
           config.segment_terminators.segment;
  }

  private buildMemberSegments(member: Edi834MemberInput, config: EdiConfig): string[] {
    const segments: string[] = [];
    // INS segment
    segments.push(`INS${SEP}${member.isSubscriber ? 'Y' : 'N'}${SEP}18${SEP}${member.maintenanceType}${SEP}${SEP}A`);
    // REF segments (member ID, group number)
    segments.push(`REF${SEP}0F${SEP}${member.subscriberId}`);
    if (config.custom_ref_qualifiers.group_id) {
      segments.push(`REF${SEP}${config.custom_ref_qualifiers.group_id}${SEP}${member.groupNumber}`);
    }
    // NM1 segment (name)
    segments.push(`NM1${SEP}IL${SEP}1${SEP}${member.lastName}${SEP}${member.firstName}${SEP}${SEP}${SEP}${SEP}34${SEP}${member.ssn}`);
    // DMG segment (demographics)
    segments.push(`DMG${SEP}D8${SEP}${formatDate(member.dateOfBirth)}${SEP}${member.gender}`);
    // HD segment (health coverage)
    segments.push(`HD${SEP}${member.maintenanceType}${SEP}${SEP}${member.insuranceLineCode}${SEP}${member.planCode}${SEP}${member.coverageLevelCode}`);
    // DTP segment (effective date)
    segments.push(`DTP${SEP}348${SEP}D8${SEP}${formatDate(member.effectiveDate)}`);
    return segments;
  }
}
```

**Testing:**
- Unit test: Generate a valid EDI 834 file for a single member addition
- Unit test: Generate a file with mixed additions and terminations
- Unit test: Carrier-specific segment terminators are applied correctly
- Unit test: Custom reference qualifiers from carrier config are included
- Unit test: ISA segment is padded to exactly 106 characters per X12 specification
- Integration test: Generate 834 from actual enrollment records; parse the output with the parser and verify round-trip fidelity
- Integration test: Generate files for multiple carriers with different EDI configs in the same batch run

### Task 5.3: EDI 834 Reconciliation Engine

**What:** Compare outbound EDI 834 member records against enrollment data to detect ghost enrollments (carrier has member not in our system), missed terminations (we terminated but carrier still shows active), and premium discrepancies.

**Design:**

```typescript
// packages/edi-parser/src/reconciliation/edi-reconciliation.ts
export interface ReconciliationResult {
  matched: ReconciliationMatch[];
  discrepancies: ReconciliationDiscrepancy[];
  summary: {
    totalMembers: number;
    matched: number;
    ghostEnrollments: number;
    missedTerminations: number;
    premiumMismatches: number;
    dataDiscrepancies: number;
  };
}

export class Edi834ReconciliationEngine {
  async reconcile(
    ediBatch: EdiBatch,
    enrollments: Enrollment[],
    carrierMembers: Edi834Member[]
  ): Promise<ReconciliationResult> {
    const enrollmentIndex = new Map(
      enrollments.map(e => [this.buildMatchKey(e), e])
    );

    const result: ReconciliationResult = {
      matched: [],
      discrepancies: [],
      summary: { totalMembers: carrierMembers.length, matched: 0, ghostEnrollments: 0,
                  missedTerminations: 0, premiumMismatches: 0, dataDiscrepancies: 0 },
    };

    for (const member of carrierMembers) {
      const key = this.buildMemberMatchKey(member);
      const enrollment = enrollmentIndex.get(key);

      if (!enrollment) {
        // Ghost enrollment: carrier has member we don't recognize
        result.discrepancies.push({
          type: 'ghost_enrollment',
          ediMember: member,
          enrollment: null,
          description: `Carrier shows active member ${member.names[0]?.firstName} ${member.names[0]?.lastName} not found in enrollment records`,
        });
        result.summary.ghostEnrollments++;
        continue;
      }

      // Check for missed terminations
      if (enrollment.status === 'terminated' && !member.healthCoverage[0]?.terminationDate) {
        result.discrepancies.push({
          type: 'missed_termination',
          ediMember: member,
          enrollment,
          description: `Employee terminated on ${enrollment.termination_date} but carrier still shows active coverage`,
        });
        result.summary.missedTerminations++;
        continue;
      }

      // Matched
      result.matched.push({ ediMember: member, enrollment });
      result.summary.matched++;
      enrollmentIndex.delete(key);
    }

    // Remaining enrollments not found in carrier data
    for (const [, enrollment] of enrollmentIndex) {
      if (enrollment.status === 'active') {
        result.discrepancies.push({
          type: 'missing_from_carrier',
          ediMember: null,
          enrollment,
          description: `Active enrollment for ${enrollment.employee_id} not found in carrier file`,
        });
      }
    }

    return result;
  }

  private buildMatchKey(enrollment: Enrollment): string {
    return `${enrollment.employee_id}:${enrollment.plan_id}`;
  }
}
```

**Testing:**
- Unit test: Perfect match -- all carrier members match enrollments
- Unit test: Ghost enrollment detected when carrier has unknown member
- Unit test: Missed termination detected when we terminated but carrier shows active
- Unit test: Missing from carrier detected when we have active enrollment carrier does not
- Unit test: Match key handles case sensitivity and whitespace in names
- Integration test: Reconcile a batch of 50 members with 3 planted discrepancies; verify all 3 detected
- Integration test: Reconciliation results persisted in `edi_batch.reconciliation` JSONB
- Integration test: Reconciliation produces domain events for each discrepancy found
- Performance test: Reconcile 10,000 members in under 5 seconds

### Task 5.4: EDI Batch Management API

**What:** API for generating, sending (via SFTP), tracking, and reconciling EDI batches.

**Design:**

```typescript
// packages/api/src/modules/edi/edi.controller.ts
@Controller('edi')
@Roles('hr_admin', 'benefits_manager')
export class EdiController {
  @Post('834/generate')
  async generate834(@Body() dto: Generate834Dto): Promise<EdiBatch> {
    // Gather all enrollment changes since last batch for specified carrier
    // Generate EDI 834 file
    // Store in edi_batch table with raw_content
  }

  @Post('834/:batchId/send')
  async send834(@Param('batchId') batchId: string): Promise<void> {
    // Send via SFTP using carrier's edi_config
    // Update batch status to 'sent'
  }

  @Post('834/:batchId/reconcile')
  async reconcile834(@Param('batchId') batchId: string, @Body() carrierResponse: string): Promise<ReconciliationResult> {
    // Parse carrier response
    // Run reconciliation engine
    // Store results in batch.reconciliation JSONB
  }

  @Get('834/batches')
  async listBatches(@Query() filters: EdiBatchFilterDto): Promise<EdiBatch[]> {
    // List batches with status, discrepancy counts, etc.
  }
}
```

**Testing:**
- Integration test: Generate 834 batch for a carrier; verify batch record created
- Integration test: Send batch via mock SFTP; verify status transitions to 'sent'
- Integration test: Upload carrier acknowledgment; verify reconciliation results stored
- E2E test: Admin views EDI batch list, sees batch with 2 discrepancies, clicks to view details
- E2E test: Admin generates a correction file for ghost enrollments

### Definition of Done -- Phase 5
- [ ] EDI 834 parser handles all common segment types (ISA/GS/ST/INS/NM1/DMG/HD/DTP/REF)
- [ ] EDI 834 generator produces valid X12 5010 files with carrier-specific config
- [ ] Round-trip test: generate -> parse -> compare returns identical data
- [ ] Reconciliation engine detects ghost enrollments, missed terminations, and missing-from-carrier
- [ ] EDI batch management API with generate, send, reconcile endpoints
- [ ] SFTP transport for outbound files (mock in tests, real in staging)
- [ ] Admin UI for EDI batch management and discrepancy review
- [ ] >90% test coverage on parser and generator (critical infrastructure)

---

## Phase 6: Payroll Integration

**Goal:** Generate payroll deduction files that reflect benefit elections, Section 125 pre-tax deductions, and employer contributions. Support export formats for major payroll systems.

**Duration estimate:** 2-3 weeks

### Task 6.1: Payroll Deduction Calculation

**What:** Calculate per-period deductions from active enrollments and Section 125 elections, accounting for pay frequency (weekly, biweekly, semimonthly, monthly).

**Design:**

```typescript
// packages/core/src/services/payroll-service/deduction-calculator.ts
export class DeductionCalculator {
  calculateForPayPeriod(
    employee: Employee,
    enrollments: Enrollment[],
    section125Elections: Section125Account[],
    payPeriodStart: Date,
    payPeriodEnd: Date,
  ): PayrollDeduction[] {
    const periodsPerYear = this.getPeriodsPerYear(employee.pay_frequency);
    const deductions: PayrollDeduction[] = [];

    for (const enrollment of enrollments) {
      if (enrollment.status !== 'active') continue;
      if (enrollment.effective_date > payPeriodEnd) continue;
      if (enrollment.termination_date && enrollment.termination_date < payPeriodStart) continue;

      const monthlyEmployeeCost = enrollment.costs.employee_per_period;
      const perPeriodAmount = (monthlyEmployeeCost * 12) / periodsPerYear;

      deductions.push({
        employee_id: employee.id,
        enrollment_id: enrollment.id,
        type: `${enrollment.plan_type}_premium`,
        is_pretax: enrollment.is_pretax,
        employee_amount: round(perPeriodAmount, 2),
        employer_amount: round((enrollment.costs.employer_per_period * 12) / periodsPerYear, 2),
      });
    }

    // Section 125 elections (FSA, HSA, etc.)
    for (const election of section125Elections) {
      if (election.status !== 'active') continue;
      deductions.push({
        employee_id: employee.id,
        election_id: election.id,
        type: election.account_type,
        is_pretax: true,
        employee_amount: election.per_period_deduction,
        employer_amount: election.employer_contribution / periodsPerYear,
      });
    }

    return deductions;
  }

  private getPeriodsPerYear(frequency: string): number {
    switch (frequency) {
      case 'weekly': return 52;
      case 'biweekly': return 26;
      case 'semimonthly': return 24;
      case 'monthly': return 12;
      default: throw new Error(`Unknown pay frequency: ${frequency}`);
    }
  }
}
```

**Testing:**
- Unit test: Monthly premium of $300 with biweekly pay = $138.46/period ($300 * 12 / 26)
- Unit test: Enrollment effective after pay period is excluded
- Unit test: Terminated enrollment before pay period is excluded
- Unit test: FSA election deduction calculated correctly per period
- Unit test: Employer contribution calculated correctly
- Integration test: Generate deductions for 50 employees; verify totals match expected

### Task 6.2: Payroll Export File Generation

**What:** Generate payroll deduction batch files in formats compatible with ADP, Workday, Gusto, and generic CSV.

**Design:**

```typescript
// packages/api/src/modules/payroll/payroll-export.service.ts
export class PayrollExportService {
  async generateBatch(tenantId: string, payPeriod: PayPeriodInput): Promise<PayrollBatch> {
    const employees = await this.employeeRepo.findActive(tenantId);
    const allDeductions: PayrollDeductionItem[] = [];

    for (const employee of employees) {
      const enrollments = await this.enrollmentRepo.findActiveForEmployee(employee.id);
      const elections = await this.section125Repo.findActiveForEmployee(employee.id);
      const deductions = this.calculator.calculateForPayPeriod(
        employee, enrollments, elections,
        payPeriod.start, payPeriod.end,
      );
      allDeductions.push({ employee, deductions });
    }

    // Persist batch
    const batch = await this.db.insertInto('payroll_deduction_batch').values({
      tenant_id: tenantId,
      pay_period_start: payPeriod.start,
      pay_period_end: payPeriod.end,
      status: 'draft',
      deductions: allDeductions,
      export_config: {},
    }).returningAll().executeTakeFirst();

    return batch;
  }

  async exportBatch(batchId: string, format: 'csv' | 'adp' | 'workday'): Promise<Buffer> {
    const batch = await this.findById(batchId);
    switch (format) {
      case 'csv': return this.csvExporter.export(batch);
      case 'adp': return this.adpExporter.export(batch);
      case 'workday': return this.workdayExporter.export(batch);
    }
  }
}
```

**Testing:**
- Integration test: Generate batch for 20 employees; verify all deductions included
- Integration test: Export as CSV; verify headers and data rows are correct
- Integration test: Export as ADP format; verify field mapping matches ADP import spec
- Integration test: Batch status transitions: draft -> exported -> confirmed
- Validation test: Pre-tax deductions are flagged correctly in export file

### Definition of Done -- Phase 6
- [ ] Deduction calculator handles all pay frequencies with correct per-period amounts
- [ ] Payroll batch generation aggregates all active enrollments and Section 125 elections
- [ ] Export formats: generic CSV, ADP import format
- [ ] Batch status tracking (draft -> exported -> confirmed)
- [ ] Admin UI for batch generation, review, and export download
- [ ] >85% test coverage on deduction calculator

---

## Phase 7: Employee Self-Service Portal

**Goal:** Build the employee-facing enrollment portal with plan comparison, cost calculators, and a guided enrollment wizard. This is the primary user-facing product surface.

**Duration estimate:** 5-6 weeks

### Task 7.1: Plan Comparison & Cost Calculator

**What:** Employee-facing UI that displays available plans side-by-side with real dollar cost projections based on the employee's salary, family status, and coverage level.

**Design:**

```typescript
// apps/web/src/components/enrollment/plan-comparison.tsx
interface PlanComparisonProps {
  plans: BenefitPlan[];
  employee: { salary: number; payFrequency: string; dependentCount: number };
}

export function PlanComparison({ plans, employee }: PlanComparisonProps) {
  const [selectedCoverage, setSelectedCoverage] = useState<CoverageLevel>('employee_only');

  const plansWithCosts = plans.map(plan => ({
    ...plan,
    annualEmployeeCost: calculateAnnualCost(plan, selectedCoverage),
    biweeklyCost: calculatePerPeriodCost(plan, selectedCoverage, employee.payFrequency),
    estimatedOopCost: estimateAnnualOop(plan, selectedCoverage), // based on plan design
    totalEstimatedCost: calculateAnnualCost(plan, selectedCoverage) +
                        estimateAnnualOop(plan, selectedCoverage),
  }));

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
      <CoverageLevelSelector value={selectedCoverage} onChange={setSelectedCoverage} />
      {plansWithCosts.map(plan => (
        <PlanCard key={plan.id} plan={plan}>
          <CostBreakdown
            premium={plan.biweeklyCost}
            deductible={plan.plan_design.in_network?.individual_deductible}
            oopMax={plan.plan_design.in_network?.individual_oop_max}
            estimatedTotal={plan.totalEstimatedCost}
          />
          <PlanDesignDetails design={plan.plan_design} />
          <EnrollButton planId={plan.id} coverageLevel={selectedCoverage} />
        </PlanCard>
      ))}
    </div>
  );
}
```

**Testing:**
- Unit test: `calculateAnnualCost` returns correct amount for family coverage tier
- Unit test: `calculatePerPeriodCost` divides annual cost by correct number of periods
- Unit test: `estimateAnnualOop` returns deductible + typical coinsurance estimate
- E2E test (Playwright): Employee views plan comparison, switches coverage level, costs update
- E2E test: Plan cards display deductible, copay, and OOP max from plan_design JSONB
- Accessibility test: Plan comparison passes WCAG 2.1 AA (keyboard navigation, screen reader labels)

### Task 7.2: Enrollment Wizard

**What:** Multi-step guided enrollment flow: select coverage level -> choose plan -> add dependents -> review -> confirm.

**Design:**

```typescript
// apps/web/src/components/enrollment/enrollment-wizard.tsx
type WizardStep = 'coverage' | 'plan' | 'dependents' | 'review' | 'confirm';

export function EnrollmentWizard({ planType, enrollmentPeriod }: EnrollmentWizardProps) {
  const [step, setStep] = useState<WizardStep>('coverage');
  const [selections, setSelections] = useState<EnrollmentSelections>({});

  return (
    <WizardContainer currentStep={step}>
      <StepIndicator steps={['Coverage', 'Plan', 'Dependents', 'Review', 'Confirm']} current={step} />

      {step === 'coverage' && (
        <CoverageLevelStep
          onSelect={(level) => { setSelections(s => ({ ...s, coverageLevel: level })); setStep('plan'); }}
        />
      )}

      {step === 'plan' && (
        <PlanSelectionStep
          planType={planType}
          coverageLevel={selections.coverageLevel!}
          onSelect={(planId) => { setSelections(s => ({ ...s, planId })); setStep('dependents'); }}
          onBack={() => setStep('coverage')}
        />
      )}

      {step === 'dependents' && (
        <DependentSelectionStep
          coverageLevel={selections.coverageLevel!}
          onSelect={(depIds) => { setSelections(s => ({ ...s, dependentIds: depIds })); setStep('review'); }}
          onBack={() => setStep('plan')}
        />
      )}

      {step === 'review' && (
        <ReviewStep
          selections={selections}
          onConfirm={async () => {
            await enrollmentApi.initiateAndConfirm(selections);
            setStep('confirm');
          }}
          onBack={() => setStep('dependents')}
        />
      )}

      {step === 'confirm' && <ConfirmationStep selections={selections} />}
    </WizardContainer>
  );
}
```

**Testing:**
- E2E test: Employee completes full enrollment wizard: coverage -> plan -> dependents -> review -> confirm
- E2E test: Employee can navigate back to previous steps and change selections
- E2E test: Employee with employee_only coverage skips dependent selection step
- E2E test: Review step shows accurate cost summary before confirmation
- E2E test: After confirmation, enrollment appears in employee's active benefits
- E2E test: Employee waives coverage, provides reason, sees confirmation
- Mobile test: Enrollment wizard is usable on a 375px-wide viewport
- Error test: Network error during confirmation shows retry option, does not lose selections

### Task 7.3: Employee Benefits Dashboard

**What:** Post-enrollment dashboard showing active benefits, coverage details, dependent information, and upcoming deadlines.

**Design:**

```typescript
// apps/web/src/app/(employee)/dashboard/page.tsx
export default async function EmployeeDashboard() {
  const employee = await getCurrentEmployee();
  const enrollments = await getActiveEnrollments(employee.id);
  const upcomingDeadlines = await getUpcomingDeadlines(employee.id);

  return (
    <div>
      <WelcomeHeader employee={employee} />
      <ActiveBenefitsGrid enrollments={enrollments} />
      <CoverageCards enrollments={enrollments} />
      <DependentsSummary employeeId={employee.id} />
      <DeadlineReminders deadlines={upcomingDeadlines} />
      <QuickActions>
        <ReportLifeEventButton />
        <ViewIdCardsButton />
        <ContactBrokerButton />
      </QuickActions>
    </div>
  );
}
```

**Testing:**
- E2E test: Employee logs in, sees dashboard with active medical, dental, and vision enrollments
- E2E test: Employee clicks on a benefit card to see plan design details (deductible, copay, OOP max)
- E2E test: Employee sees list of covered dependents
- E2E test: Upcoming open enrollment deadline shown in reminders section

### Definition of Done -- Phase 7
- [ ] Plan comparison UI with side-by-side cost projections per coverage level
- [ ] Guided enrollment wizard with multi-step flow (coverage -> plan -> dependents -> review -> confirm)
- [ ] Employee benefits dashboard showing active coverage, dependents, and deadlines
- [ ] Mobile-responsive layouts passing viewport tests at 375px
- [ ] WCAG 2.1 AA accessibility compliance on all employee-facing pages
- [ ] >80% E2E test coverage on enrollment flows

---

## Phase 8: AI Decision Support

**Goal:** Implement the AI-powered plan recommendation engine and plain-language benefits explanation generator. These features differentiate the platform from incumbents.

**Duration estimate:** 4-5 weeks

### Task 8.1: Plan Recommendation Engine

**What:** A transparent, rule-based scoring engine that ranks available plans for each employee based on their salary, family size, historical utilization (if available), and life stage. Recommendations include human-readable rationale per DOL 2025 ERISA guidance.

**Design:**

```typescript
// packages/core/src/services/recommendation-service/plan-recommender.ts
export class PlanRecommender {
  async recommend(input: RecommendationInput): Promise<PlanRecommendation[]> {
    const { employee, availablePlans, historicalClaims } = input;
    const scored: ScoredPlan[] = [];

    for (const plan of availablePlans) {
      const factors: ScoringFactor[] = [];

      // Factor 1: Annual total cost (premium + estimated OOP)
      const annualPremium = this.calculateAnnualPremium(plan, employee);
      const estimatedOop = this.estimateOop(plan, employee, historicalClaims);
      const totalCost = annualPremium + estimatedOop;
      factors.push({ name: 'total_cost', score: this.normalizeCost(totalCost, availablePlans), weight: 0.40 });

      // Factor 2: Coverage adequacy for family size
      const coverageScore = this.scoreCoverageAdequacy(plan, employee.dependentCount);
      factors.push({ name: 'coverage_adequacy', score: coverageScore, weight: 0.25 });

      // Factor 3: HSA eligibility (HDHP plans enable tax-advantaged HSA)
      const hsaScore = plan.plan_design.network_type === 'hdhp' ? 0.8 : 0.3;
      factors.push({ name: 'hsa_eligibility', score: hsaScore, weight: 0.15 });

      // Factor 4: Provider network breadth
      const networkScore = plan.plan_design.network_type === 'ppo' ? 0.9 : 0.5;
      factors.push({ name: 'network_breadth', score: networkScore, weight: 0.10 });

      // Factor 5: Prescription coverage generosity
      const rxScore = this.scoreRxCoverage(plan);
      factors.push({ name: 'rx_coverage', score: rxScore, weight: 0.10 });

      const totalScore = factors.reduce((sum, f) => sum + f.score * f.weight, 0);
      const rationale = this.generateRationale(plan, factors, employee);

      scored.push({ plan, score: totalScore, factors, rationale, totalCost, estimatedOop });
    }

    scored.sort((a, b) => b.score - a.score);

    // Persist recommendations
    await this.db.insertInto('plan_recommendation').values({
      tenant_id: input.tenantId,
      employee_id: employee.id,
      enrollment_period_id: input.enrollmentPeriodId,
      recommendations: scored.map((s, i) => ({
        plan_id: s.plan.id,
        rank: i + 1,
        score: round(s.score, 3),
        rationale: s.rationale,
        projected_annual_cost: s.totalCost,
        projected_oop_cost: s.estimatedOop,
      })),
      model_version: 'v1.0-rule-based',
    }).execute();

    return scored;
  }

  private generateRationale(plan: BenefitPlan, factors: ScoringFactor[], employee: Employee): string {
    const parts: string[] = [];
    const topFactor = factors.sort((a, b) => (b.score * b.weight) - (a.score * a.weight))[0];

    if (topFactor.name === 'total_cost') {
      parts.push(`This plan has the lowest estimated total annual cost for your family size and salary range.`);
    }
    if (plan.plan_design.network_type === 'hdhp') {
      parts.push(`As an HDHP, this plan qualifies you for a Health Savings Account (HSA) with tax-free contributions and growth.`);
    }
    if (employee.dependentCount >= 3 && plan.plan_design.in_network?.family_oop_max) {
      parts.push(`The family out-of-pocket maximum of $${plan.plan_design.in_network.family_oop_max} provides cost protection for your family of ${employee.dependentCount + 1}.`);
    }
    parts.push(`Note: This recommendation is for informational purposes only and does not constitute financial or medical advice. Please consult with a benefits advisor for personalized guidance.`);

    return parts.join(' ');
  }
}
```

**Testing:**
- Unit test: HDHP plan scores higher on HSA factor
- Unit test: PPO scores higher on network breadth factor
- Unit test: Cheapest total cost plan has highest cost factor score
- Unit test: Rationale includes ERISA disclaimer text
- Unit test: Rationale mentions HDHP/HSA advantage for HDHP plans
- Unit test: Recommendations are ranked by total weighted score
- Integration test: Generate recommendations for an employee with 3 dependents; verify family-oriented plan ranks higher
- Integration test: Recommendations persisted with model_version and rationale
- ERISA compliance test: Every recommendation includes the disclaimer per DOL 2025 guidance
- Transparency test: Each factor score and weight is stored and retrievable for audit

### Task 8.2: Plain-Language Benefits Explanation Generator

**What:** Use an LLM (behind a provider abstraction) to generate plain-language summaries of plan details, SPDs, and benefits communications in multiple languages.

**Design:**

```typescript
// packages/core/src/services/ai-service/explanation-generator.ts
export class ExplanationGenerator {
  constructor(private llmProvider: LlmProvider) {}

  async generatePlanSummary(plan: BenefitPlan, language: string = 'en'): Promise<string> {
    const prompt = `
      Summarize the following health insurance plan in plain language that a non-expert employee
      can understand. Use short sentences and avoid jargon. Include key numbers
      (deductible, copays, out-of-pocket max). Output in ${language}.

      Plan: ${plan.plan_name}
      Type: ${plan.plan_type} (${plan.plan_design.network_type})
      Deductible: $${plan.plan_design.in_network?.individual_deductible} individual / $${plan.plan_design.in_network?.family_deductible} family
      Out-of-pocket max: $${plan.plan_design.in_network?.individual_oop_max} individual / $${plan.plan_design.in_network?.family_oop_max} family
      Primary care copay: $${plan.plan_design.in_network?.copay_primary}
      Specialist copay: $${plan.plan_design.in_network?.copay_specialist}
      ER copay: $${plan.plan_design.in_network?.copay_er}
      Coinsurance: ${plan.plan_design.in_network?.coinsurance_pct}%
      Rx: Generic $${plan.plan_design.in_network?.rx?.generic}, Brand $${plan.plan_design.in_network?.rx?.brand}
    `;

    return this.llmProvider.generate(prompt);
  }
}

// Provider abstraction for self-hosted environments
export interface LlmProvider {
  generate(prompt: string): Promise<string>;
}

export class AnthropicProvider implements LlmProvider { /* Claude API */ }
export class OpenAiProvider implements LlmProvider { /* OpenAI API */ }
export class LocalProvider implements LlmProvider { /* Ollama or similar local model */ }
export class DisabledProvider implements LlmProvider {
  async generate(): Promise<string> {
    return 'AI summaries are not available in this deployment configuration.';
  }
}
```

**Testing:**
- Unit test: `DisabledProvider` returns fallback message
- Integration test: Generate plan summary with mock LLM provider; verify output contains key plan numbers
- Integration test: Spanish language request produces Spanish output (with real or mock provider)
- E2E test: Employee views plan details page with AI-generated plain-language summary
- Configuration test: Self-hosted deployment with AI disabled falls back to structured plan details

### Definition of Done -- Phase 8
- [ ] Plan recommendation engine with weighted scoring across 5+ factors
- [ ] Recommendations include human-readable rationale with ERISA disclaimer
- [ ] Recommendations persisted with model version for audit
- [ ] LLM provider abstraction with Anthropic, OpenAI, local, and disabled implementations
- [ ] Plain-language plan summaries available in at least English and Spanish
- [ ] Recommendation UI integrated into enrollment wizard (Phase 7)
- [ ] >85% test coverage on recommendation scoring logic

---

## Phase 9: Section 125 & Spending Accounts

**Goal:** Implement FSA, HSA, HRA, and dependent care FSA account management with IRS annual limit enforcement, contribution tracking, and claim processing.

**Duration estimate:** 3-4 weeks

### Task 9.1: Section 125 Account Management

**What:** Implement the `section_125_account` table and election logic with IRS limit enforcement.

**Design:**

```typescript
// packages/core/src/services/section125-service/section125-service.ts
export class Section125Service {
  private static readonly LIMITS_2025 = {
    fsa_health: 3300,
    fsa_dependent: 5000,
    hsa_individual: 4300,
    hsa_family: 8550,
    fsa_carryover: 640,
  };

  async createElection(input: Section125ElectionInput): Promise<Section125Account> {
    // Validate annual election does not exceed IRS limit
    const limit = Section125Service.LIMITS_2025[input.accountType];
    if (input.annualElection > limit) {
      throw new BusinessError('EXCEEDS_IRS_LIMIT', { limit, elected: input.annualElection });
    }

    // HSA: verify employee is enrolled in HDHP
    if (input.accountType === 'hsa') {
      const hasHdhp = await this.enrollmentRepo.hasActiveHdhpEnrollment(input.employeeId);
      if (!hasHdhp) throw new BusinessError('HSA_REQUIRES_HDHP');
    }

    const periodsPerYear = this.getPeriodsPerYear(input.payFrequency);
    const perPeriodDeduction = round(input.annualElection / periodsPerYear, 2);

    return this.db.insertInto('section_125_account').values({
      tenant_id: input.tenantId,
      employee_id: input.employeeId,
      account_type: input.accountType,
      plan_year_start: input.planYearStart,
      plan_year_end: input.planYearEnd,
      annual_election: input.annualElection,
      per_period_deduction: perPeriodDeduction,
      employer_contribution: input.employerContribution ?? 0,
      status: 'active',
      balance: {
        ytd_contributions: 0,
        ytd_claims_approved: 0,
        ytd_claims_pending: 0,
        available_balance: input.annualElection + (input.employerContribution ?? 0),
        carryover_from_prior: input.carryoverFromPrior ?? 0,
      },
      transactions: [],
    }).returningAll().executeTakeFirst();
  }

  async submitClaim(accountId: string, claim: ClaimInput): Promise<void> {
    // Validate claim amount <= available balance
    // Add claim to transactions JSONB array
    // Update balance.ytd_claims_pending
  }

  async approveClaim(accountId: string, claimId: string): Promise<void> {
    // Move claim from pending to approved
    // Update available_balance
  }
}
```

**Testing:**
- Unit test: FSA health election at $3,301 is rejected (exceeds $3,300 limit)
- Unit test: FSA health election at $3,300 is accepted
- Unit test: HSA election rejected when employee has no HDHP enrollment
- Unit test: HSA election accepted when employee has active HDHP
- Unit test: Per-period deduction calculated correctly for biweekly frequency
- Integration test: Create FSA election, submit claim, approve claim, verify balance updated
- Integration test: Carryover from prior year included in available balance
- Integration test: End-of-year forfeiture when balance remains (use-it-or-lose-it)

### Definition of Done -- Phase 9
- [ ] Section 125 account creation with IRS limit enforcement
- [ ] FSA, HSA, HRA, and DCFSA account types supported
- [ ] HSA requires active HDHP enrollment
- [ ] Claim submission, approval, and denial workflow
- [ ] Balance tracking with YTD contributions, claims, and available balance
- [ ] Carryover and grace period logic (configurable per tenant)
- [ ] Year-end forfeiture processing
- [ ] >90% test coverage on limit enforcement and balance calculations

---

## Phase 10: Broker Portal & Multi-Carrier Operations

**Goal:** Build a broker-facing portal for managing client employer groups and add EDI 820 premium payment file generation.

**Duration estimate:** 3-4 weeks

### Task 10.1: Broker Portal

**What:** Broker role with access to assigned client tenants, plan benchmarking, enrollment statistics, and carrier management on behalf of clients.

**Design:**
- Broker `app_user` with role `broker` and association to one or more tenants
- Dashboard showing all client tenants with enrollment status and renewal dates
- Plan benchmarking: compare plan designs and rates across client groups
- RBAC ensures brokers can only see assigned tenants

**Testing:**
- Integration test: Broker can view assigned tenant's enrollment data
- Integration test: Broker cannot access unassigned tenant (403)
- E2E test: Broker dashboard shows client list with enrollment progress

### Task 10.2: EDI 820 Premium Payment Generation

**What:** Generate EDI 820 premium payment/remittance files for carrier billing reconciliation.

**Design:**
- Extend `edi_batch` table support for `edi_type = '820'`
- Generate 820 files from active enrollments grouped by carrier, plan, and coverage level
- Include employer and employee contribution breakdowns

**Testing:**
- Unit test: 820 file totals match sum of all active enrollment premiums for carrier
- Integration test: Generate 820 for a carrier with 3 plans; verify member-level detail lines
- Integration test: Reconcile 820 total against carrier invoice amount

### Definition of Done -- Phase 10
- [ ] Broker portal with client tenant management
- [ ] Broker RBAC restricting access to assigned tenants only
- [ ] EDI 820 premium payment file generation
- [ ] Carrier premium reconciliation (generated vs. invoiced)
- [ ] Plan benchmarking UI for brokers

---

## Phase 11: International Benefits

**Goal:** Extend the platform to support non-US benefits structures, starting with EU and Canada. The hybrid JSONB model was chosen specifically to enable this without schema migrations.

**Duration estimate:** 4-5 weeks

### Task 11.1: Multi-Jurisdiction Framework

**What:** Implement a jurisdiction framework that drives locale-specific rules, field requirements, and compliance logic based on ISO 3166 country/subdivision codes.

**Design:**
- Jurisdiction configuration in `tenant.compliance_config` JSONB
- Employee `demographics` JSONB stores jurisdiction-specific identifiers (UK NIN, Canadian SIN, etc.)
- Plan types extended for non-US benefits (private medical insurance, pension auto-enrollment, etc.)

**Testing:**
- Unit test: Canadian employee demographics include SIN field
- Unit test: UK auto-enrollment pension rules trigger at correct thresholds
- Integration test: Create a UK tenant with pension auto-enrollment plans

### Task 11.2: GDPR Compliance Layer

**What:** Implement GDPR data subject rights (access, rectification, erasure) and consent management for EU employee data.

**Design:**
- Data subject access request API: export all personal data for an employee
- Right to erasure: anonymize employee records after retention period
- Consent tracking for optional data processing (AI recommendations, analytics)
- Data Processing Agreement support documentation

**Testing:**
- Integration test: Data subject access request returns complete employee record
- Integration test: Erasure request anonymizes employee data while retaining aggregate statistics
- Compliance test: Anonymized records cannot be linked back to individuals

### Definition of Done -- Phase 11
- [ ] Jurisdiction framework supporting US, UK, Canada, and EU
- [ ] Locale-specific employee demographics via JSONB
- [ ] GDPR data subject rights (access, rectification, erasure)
- [ ] Consent management for AI features
- [ ] Non-US benefit plan types (pension, private medical insurance)

---

## Phase 12: Production Hardening & SOC 2 Readiness

**Goal:** Prepare the platform for production use with security hardening, performance optimization, monitoring, and documentation required for SOC 2 Type II certification.

**Duration estimate:** 4-5 weeks

### Task 12.1: Security Hardening

**What:** Penetration testing remediation, dependency vulnerability scanning, PHI encryption verification, and security headers.

**Design:**
- Automated dependency scanning (Snyk or GitHub Dependabot)
- OWASP Top 10 verification
- PHI field encryption verification (AES-256) and key rotation support
- CSP, HSTS, and other security headers on all responses
- Rate limiting on authentication endpoints

**Testing:**
- Security test: SQL injection attempts on all API endpoints return 400 (parameterized queries)
- Security test: XSS payloads in employee name fields are sanitized
- Security test: PHI fields (SSN, DOB) are never returned in API responses without explicit PHI scope
- Security test: Rate limiting blocks >10 failed login attempts in 5 minutes
- Penetration test: External pen test with remediation of all critical/high findings

### Task 12.2: Performance Optimization & Monitoring

**What:** Database query optimization, connection pooling, caching, and observability infrastructure.

**Design:**
- Connection pooling via PgBouncer for self-hosted deployments
- Redis caching for plan design and rate data (read-heavy, change-infrequent)
- OpenTelemetry tracing on all API endpoints and database queries
- Prometheus metrics: enrollment rates, EDI batch processing times, API latency percentiles
- Alert thresholds for enrollment period deadline approaching with low completion

**Testing:**
- Performance test: 500 concurrent enrollment elections during open enrollment complete in <2s each
- Performance test: EDI 834 generation for 10,000 members completes in <30 seconds
- Performance test: Plan comparison page loads in <1 second with 20 plans
- Load test: System handles 200 concurrent users without degradation

### Task 12.3: SOC 2 Documentation & Controls

**What:** Document all security controls, access policies, and audit procedures required for SOC 2 Type II certification.

**Design:**
- Access control policy documentation (who can access what, how access is granted/revoked)
- Change management procedures (code review, CI/CD, deployment approvals)
- Incident response playbook
- Data retention and disposal policy
- Vendor management policy (for carrier and cloud provider relationships)
- Audit log retention: 7 years for ERISA compliance

**Testing:**
- Documentation review: All SOC 2 trust service criteria (Security, Availability, Confidentiality) have documented controls
- Audit test: Pull 1 year of audit_log entries; verify no gaps in logging
- Access review: Generate report of all user access grants and verify with HR admin

### Definition of Done -- Phase 12
- [ ] OWASP Top 10 vulnerabilities addressed
- [ ] PHI encryption verified with key rotation tested
- [ ] Performance benchmarks met (500 concurrent enrollments, 10K member EDI generation)
- [ ] OpenTelemetry tracing active on all API endpoints
- [ ] Prometheus metrics and Grafana dashboards deployed
- [ ] SOC 2 control documentation complete
- [ ] Incident response playbook written and tabletop-tested
- [ ] Data retention policy implemented (7-year ERISA, configurable per tenant)
- [ ] External penetration test completed with all critical/high findings resolved
- [ ] Self-hosted deployment guide with HIPAA compliance checklist

---

## Summary

| Phase | Name | Duration | Dependencies | Key Deliverable |
|-------|------|----------|-------------|-----------------|
| 1 | Foundation & Core Infrastructure | 4-5 weeks | None | Database, auth, audit, CI/CD |
| 2 | Plan & Carrier Management | 3-4 weeks | Phase 1 | Plan CRUD, eligibility rules, rates |
| 3 | Enrollment Workflows | 5-6 weeks | Phase 2 | Core enrollment election, life events |
| 4 | Compliance Engine (ACA + COBRA) | 5-6 weeks | Phase 2 | ACA 1095-C, COBRA lifecycle |
| 5 | EDI 834 Carrier Integration | 5-6 weeks | Phases 3, 4 | EDI parse/generate/reconcile |
| 6 | Payroll Integration | 2-3 weeks | Phase 3 | Deduction calculation, export files |
| 7 | Employee Self-Service Portal | 5-6 weeks | Phase 3 | Enrollment wizard, plan comparison |
| 8 | AI Decision Support | 4-5 weeks | Phase 7 | Plan recommendations, LLM summaries |
| 9 | Section 125 & Spending Accounts | 3-4 weeks | Phase 3 | FSA/HSA/HRA account management |
| 10 | Broker Portal & Multi-Carrier Ops | 3-4 weeks | Phase 5 | Broker portal, EDI 820 |
| 11 | International Benefits | 4-5 weeks | Phase 3 | EU/Canada jurisdiction support, GDPR |
| 12 | Production Hardening & SOC 2 | 4-5 weeks | All phases | Security, performance, SOC 2 readiness |

**Total estimated duration:** 48-59 weeks (12-15 months)

**MVP scope (production-ready for first customer):** Phases 1-7 = 30-36 weeks (7-9 months)
