# Employee Benefits Administration

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, self-hostable platform for benefits enrollment, carrier integrations, and cost modeling — built for organisations that incumbents underserve or cannot legally serve.

Employee Benefits Administration is an open-source platform for managing employer-sponsored benefits: open enrollment, carrier EDI 834 integration, ACA and COBRA compliance, and pre-tax election handling under IRS Section 125. It targets HR and total rewards teams at SMBs without dedicated benefits staff, as well as healthcare, government, and defence organisations that require HIPAA-compliant self-hosted deployment.

---

## Why Employee Benefits Administration?

- Incumbent platforms (ADP Benefits, Benefitfocus, Businessolver, bswift, Workday Benefits) are priced from ~$3 to $20/employee/month and configured around enterprise deployments — leaving organisations under 200 employees managing open enrollment via broker spreadsheets.
- No current vendor offers a HIPAA-compliant self-hosted deployment, blocking healthcare employers, government agencies, and defence contractors that cannot send PHI to unvetted SaaS vendors without BAAs and security reviews.
- Strategic uncertainty surrounds several incumbents: Benefitfocus (Voya, 2022), bswift (CVS Health, 2014), and Alight's services-heavy model have slowed innovation velocity.
- EDI 834 reconciliation — detecting ghost enrollments, missed terminations, and premium mismatches — remains largely manual, and actuarial cost modeling typically requires $50K–$200K/year in external consultants.
- The benefits administration software market reached $1.82B in 2025 and is projected to grow to $2.98B by 2030 (Mordor Intelligence, 10.40% CAGR), but the top 10 vendors hold only 53% of share — leaving meaningful room for an AI-native open-source entrant.

---

## Key Features

### Enrollment & Plan Management

- Configurable open enrollment windows, eligibility rules, and plan offering logic for medical, dental, vision, life, disability, FSA, HSA, and HRA
- Life event change workflows (marriage, birth, divorce, qualifying status changes) with automated carrier notification
- New-hire enrollment workflows with decision support questions
- Mobile-accessible employee self-service portal with plan comparison and cost calculators
- Section 125 cafeteria plan enforcement with IRS annual election limits

### Carrier Integration & Reconciliation

- EDI 834 (ANSI X12) carrier file generation and parsing for major US health, dental, vision, life, and voluntary benefit carriers
- Automated EDI 834 reconciliation: detect and flag ghost enrollments, missed terminations, and premium discrepancies
- EDI 820 multi-carrier premium payment file generation
- Carrier acknowledgment tracking and error-handling controls

### Compliance & Reporting

- ACA compliance engine: Applicable Large Employer (ALE) determination, minimum essential coverage tracking, and 1094-C/1095-C generation and filing
- COBRA administration: qualifying event triggering, notice generation, premium billing, and continuation coverage tracking
- ERISA-compliant audit trail logging every enrollment action with timestamps and user identity
- HIPAA-compliant data handling with encrypted PHI storage, role-based access controls, and audit logs

### Decision Support & Cost Modeling

- AI-powered personalised plan recommendation engine at open enrollment using transparent rule-based logic and actuarial cost projections
- Plan design cost simulation using utilisation data and peer benchmarks
- Plain-language SPD and benefits communication generation in multiple languages
- Side-by-side plan comparison with real dollar projections based on salary and family status

### Deployment & Integration

- Self-hosted deployment option for HIPAA-compliant on-premises operation
- Payroll deduction sync with pre-tax elections reflected automatically in payroll runs
- Integrations with major HRIS and payroll systems via standard exports and APIs
- Benefits broker portal for broker-managed small group clients

---

## AI-Native Advantage

AI-native capabilities target the parts of benefits administration that consume the most labour at incumbent platforms. Personalised enrollment guidance analyses each employee's usage history, life stage, and dependents to recommend optimal plan combinations — replacing generic guides with decision support. Automated EDI 834 reconciliation parses, validates, and reconciles carrier files, detecting discrepancy patterns and generating correction requests with the potential to reduce reconciliation labour by 60–80% (per project research). Continuous plan design cost modeling democratises actuarial-style simulation that today requires expensive external consultants. All AI-assisted plan recommendations are designed with disclaimers and human oversight provisions to align with DOL 2025 guidance on ERISA fiduciary obligations.

---

## Tech Stack & Deployment

- **Self-hosted** deployment is a first-class target so that healthcare, government, and defence organisations can run the platform within their own HIPAA-compliant infrastructure with no third-party data egress.
- **Open standards**: EDI 834 (ANSI X12) and EDI 820 for carrier interoperability; ACA 1094-C/1095-C as published by the IRS; FHIR (HL7) for emerging health plan data exchange.
- **Compliance posture**: HIPAA, ERISA, ACA, COBRA, and IRS Section 125 enforcement built into the data model and workflows; SOC 2 Type II as a target for hosted deployments.
- **Integration approach**: standard payroll connectors, HRIS APIs, and a broker portal for distribution through benefits consultants.

---

## Market Context

The global benefits administration software market was valued at ~$1.82B in 2025 and is projected to reach $2.98B by 2030 at a 10.40% CAGR (Mordor Intelligence); a parallel estimate places it at $1.25B in 2024 growing to $3.83B by 2033 (Business Research Insights). Incumbent pricing ranges from ~$3–$8/employee/month (ADP) up to $10–$20/user/month (Workday Benefits), with services-heavy bundles such as Alight reaching $6–$12/employee/month. Primary buyers are VPs of Total Rewards and Benefits Directors at 500–20,000-employee companies, CFOs modelling cost trends and COBRA exposure, payroll managers, and benefits brokers advising clients.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
