# Standards & API Reference

> Project: Employee Benefits Administration · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 30414:2025 — Human Resource Management: Requirements and Recommendations for Human Capital Reporting and Disclosure**
- URL: https://www.iso.org/standard/30414
- The second edition (published August 2025) of the world's first global standard for human capital reporting. Covers 58+ metrics across 11 core areas including compliance and ethics, costs, diversity, leadership, productivity, recruitment and turnover, and workforce availability. Directly relevant to benefits reporting: defines standardised metrics for benefits costs, participation rates, and employer contributions. The 2025 revision makes certain disclosures mandatory rather than purely recommendatory, and aligns with SEC and EU CSRD human capital reporting requirements.

**ISO 30405:2016 — Human Resource Management: Guidelines on Recruitment**
- URL: https://iso-docs.com/blogs/iso-concepts/human-resource-management-system-iso-30405
- Part of the ISO TC 260 HR standards suite. Establishes guidelines for the full employee lifecycle including onboarding, which triggers benefits enrolment workflows. Benefits administration platforms must interoperate with recruitment and onboarding processes covered by this standard.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The primary international standard for information security management. Highly relevant given that benefits administration systems process sensitive personal health data (PHI), financial data, and dependent information. SOC 2 Type II compliance (common in US benefits platforms) maps to many ISO 27001 controls.

**ISO/IEC 29101:2018 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/72553.html
- Provides an architectural framework for privacy implementation in information systems. Directly applicable to benefits systems that must handle PHI under HIPAA, personal data under GDPR, and California resident data under CCPA.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The de-facto standard for authorising third-party access to benefits data via API. Used by every major HRIS platform (Workday, ADP, Rippling, Gusto, TriNet) as the authentication mechanism for their REST APIs. Benefits platforms integrate with dozens of downstream carriers and third-party tools, all requiring standardised delegated authorisation.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Used alongside OAuth 2.0 for bearer token authentication in benefits administration APIs. Enables stateless session management for API calls between employers, benefits platforms, and insurance carriers.

**RFC 7643 & 7644 — SCIM 2.0 (System for Cross-domain Identity Management)**
- URL: https://datatracker.ietf.org/doc/html/rfc7643 and https://datatracker.ietf.org/doc/html/rfc7644
- The SCIM 2.0 protocol standardises automated provisioning and de-provisioning of employee identity data across systems. In benefits administration, SCIM is used to synchronise employee records between HRIS platforms, identity providers (Okta, Azure AD), and benefits portals so that joiners, movers, and leavers trigger eligibility changes automatically. Eliminates manual re-keying of employee records across carrier systems.

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Used for single sign-on (SSO) between HR platforms, benefits portals, and carrier sites so employees authenticate once and access all benefits tools without re-login.

---

### EDI & Data Exchange Standards

**ANSI X12 EDI 834 — Benefit Enrollment and Maintenance (Version 005010X220A1)**
- URL: https://x12.org/examples/005010x220
- The primary US standard for electronic benefits enrollment data exchange between employers/plan administrators and insurance carriers. Mandated under HIPAA for health plan enrollment. The 834 transaction set carries: new enrolments, dependent additions, coverage changes, reinstatements, and terminations. Key segments include ISA/IEA (interchange envelope), GS/GE (functional group), ST/SE (transaction set), INS (subscriber/dependent indicator), NM1 (individual names), HD (health coverage details), DSB (disability coverage), and REF (member ID cross-references). Version 5010 has been the required HIPAA standard since January 2012. Wisconsin DHS published updated 834 5010X220A1 implementation instructions in February 2026.

**ANSI X12 EDI 835 — Health Care Claim Payment/Advice**
- URL: https://x12.org/
- Used by health insurers to communicate claim payments, adjustments, and denials back to providers and employers. Benefits administration platforms consume 835 files to reconcile carrier payments against plan claims and produce explanation-of-benefits (EOB) reports for employees.

**ANSI X12 EDI 837 — Health Care Claim (Professional 837P, Institutional 837I, Dental 837D)**
- URL: https://x12.org/
- Used to submit healthcare claims from providers to payers. Benefits administration platforms that include self-insured plan management need to process 837 transaction sets for claims adjudication. HIPAA mandated 5010 compliance for all electronic claims since 2012. Relevant code sets include ICD-10, CPT, HCPCS, CDT, and NDC.

**ANSI X12 EDI 270/271 — Health Care Eligibility Benefit Inquiry and Response**
- URL: https://x12.org/
- Allows employers, HR systems, and providers to query carrier systems in real time to verify an employee's current benefit eligibility and coverage details. The 271 response confirms or denies coverage, returning plan type, deductible amounts, copay information, and out-of-pocket maximums.

**HL7 FHIR R5 — Fast Healthcare Interoperability Resources**
- URL: https://hl7.org/fhir/
- The modern REST-based healthcare data exchange standard published by HL7 International. FHIR defines standardised resources directly relevant to benefits administration: Claim, Coverage, CoverageEligibilityRequest/Response, ExplanationOfBenefit, Account, Invoice, and ChargeItem. Insurance carriers are increasingly building FHIR APIs alongside (or replacing) EDI 834/835/837 pipelines, and CMS has mandated FHIR APIs for payer-to-payer data exchange under the Interoperability and Patient Access Rule. Benefits platforms integrating with carriers need FHIR R4/R5 support.

---

### US Regulatory Frameworks

**HIPAA — Health Insurance Portability and Accountability Act (1996)**
- URL: https://www.cms.gov/priorities/key-initiatives/burden-reduction/administrative-simplification/hipaa/adopted-standards-operating-rules
- Mandates the use of standard electronic transactions (EDI 834, 835, 837, 270/271) for health plan administration and establishes the Privacy Rule and Security Rule for Protected Health Information (PHI). Any benefits administration platform handling health insurance data is a Business Associate under HIPAA and must sign BAAs with covered entities, implement administrative/physical/technical safeguards, and maintain audit trails.

**ERISA — Employee Retirement Income Security Act (1974)**
- URL: https://www.dol.gov/general/topic/health-plans/erisa
- The primary federal law governing private employer benefit plans. Key ERISA compliance requirements for benefits software: Summary Plan Description (SPD) generation and distribution, Form 5500 annual reporting to DOL/IRS, fiduciary responsibility tracking, non-discrimination testing (for Section 125, 401(k), and other plans), COBRA administration, and participant appeals/grievance process management. Benefits software must produce ERISA-compliant plan documents and maintain auditable records.

**ACA — Affordable Care Act (2010)**
- URL: https://www.irs.gov/e-file-providers/affordable-care-act-information-returns-air
- Requires Applicable Large Employers (ALEs, 50+ FTE) to offer minimum essential coverage and to file Forms 1094-C and 1095-C with the IRS. As of tax year 2023, electronic filing is mandatory for 10+ information returns. Benefits platforms must track monthly employee eligibility status, identify affordability thresholds, generate 1095-C forms automatically, and e-file via the IRS AIR (Affordable Care Act Information Returns) system.

**IRC Section 125 — Cafeteria Plans**
- URL: https://www.irs.gov/pub/irs-drop/n-05-42.pdf
- Governs pre-tax benefit election plans. Benefits software must support Section 125 plan document requirements, annual non-discrimination testing (Eligibility Test, Contributions and Benefits Test, Key Employee Concentration Test), and track FSA contribution limits ($3,300 individual for 2025), dependent care FSA limits ($5,000), and HSA limits ($4,300 individual / $8,550 family for 2025), as well as carryover rules and grace period elections.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.info/
- Applies to benefits platforms processing personal data of EU employees. Relevant obligations: lawful basis for processing, data subject rights (access, rectification, erasure), data minimisation, cross-border transfer restrictions, 72-hour breach notification, and Data Processing Agreements (DPAs) with sub-processors such as insurance carriers and analytics vendors.

**CCPA / CPRA — California Consumer Privacy Act / California Privacy Rights Act**
- URL: https://oag.ca.gov/privacy/ccpa
- California's comprehensive privacy law applies to benefits administration for California-based employees. Requires disclosure of data categories collected, opt-out rights for data sales, and non-discrimination obligations. The CPRA (effective 2023) expanded employee and contractor data rights that had previously been exempt.

---

### Security Standards

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The comprehensive US government security control catalogue. Enterprise HR and benefits platforms selling to federal contractors or public sector organisations must demonstrate alignment with relevant NIST 800-53 control families, particularly AC (Access Control), AU (Audit and Accountability), IA (Identification and Authentication), and SC (System and Communications Protection).

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/topic/audit-attest/audit-and-attest-standards/auditing-standards/soc-2
- The de-facto industry certification for SaaS platforms handling sensitive data. All major benefits administration vendors (ADP, Workday, Rippling, Finch, Merge) maintain SOC 2 Type II reports covering Security, Availability, and Confidentiality trust service criteria. Required by enterprise buyers during procurement.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The standard for describing REST APIs. All modern benefits administration APIs (Gusto, Rippling, TriNet, Merge, Finch, Unified.to) publish OpenAPI 3.x schemas enabling code generation, automated testing, and third-party integration catalogues.

---

## Similar Products — Developer Documentation & APIs

### ADP Workforce Now
- **Description:** Market-leading HCM platform for mid-market and enterprise organisations. Offers benefits administration, payroll, time and attendance, and talent management.
- **API Documentation:** https://developers.adp.com/
- **SDKs/Libraries:** Available through ADP Marketplace and partner integrations. SDKs for JavaScript and Java documented through developer portal.
- **Developer Guide:** https://rollout.com/integration-guides/adp-workforce-now/api-essentials
- **Standards:** REST/JSON; requires ADP API Central (purchased separately) for credentials. OAuth 2.0 authentication.
- **Authentication:** OAuth 2.0 (client credentials flow)
- **Benefits Endpoints:** Workers v2 API for employee data; dedicated benefits and pay data input APIs. Covers employee benefits enrolment, coverage elections, carrier connections, and deduction management.

### Workday
- **Description:** Enterprise HCM platform dominant in large-enterprise HR, finance, and planning. Benefits administration is deeply integrated with payroll and compensation.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html (SOAP WWS v46.1); REST docs via Workday Community
- **SDKs/Libraries:** SOAP-based; partner integrations via Workday Studio
- **Developer Guide:** https://samaintegrations.com/mastering-workday-rest-api-integration-the-complete-2025-guide-for-businesses/
- **Standards:** Both SOAP/WSDL (legacy) and REST/JSON (newer endpoints). OAuth 2.0 required.
- **Authentication:** OAuth 2.0 (Bearer token)
- **Benefits Endpoints:** Benefits enrolment, compensation, pay calculation, and eligibility management. Carrier connections via EDI 834 integrations built in Workday Studio.

### Gusto Embedded
- **Description:** SMB-focused payroll and benefits platform with an embedded payroll API for software builders. Integrates with SimplyInsured for health benefits.
- **API Documentation:** https://docs.gusto.com/
- **SDKs/Libraries:** Python, Ruby, JavaScript, Go (via embedded platform)
- **Developer Guide:** https://docs.gusto.com/embedded-payroll/docs/getting-started
- **Standards:** REST/JSON; OpenAPI 3.x schema available
- **Authentication:** OAuth 2.0
- **Benefits Endpoints:** `PUT /v1/companies/{company_id}/payrolls/{payroll_id}/calculate` for benefits deduction calculations. Employee benefits plan management and deduction configuration endpoints available in Embedded tier.

### Rippling
- **Description:** Workforce management platform combining HR, IT, and finance. Benefits administration covers health, 401(k), and commuter benefits with direct carrier connections.
- **API Documentation:** https://developer.rippling.com/documentation/rest-api/reference/rippling-platform-api
- **SDKs/Libraries:** REST-native; no official SDK — standard HTTP clients
- **Developer Guide:** https://developer.rippling.com/docs/rippling-api/a310f900b0f84-api-reference
- **Standards:** REST/JSON; production APIs at `https://rest.ripplingapis.com/`; sandbox at `https://rest.ripplingsandboxapis.com/`
- **Authentication:** OAuth 2.0
- **Benefits Endpoints:** Read and write benefits information including plan creation and individual enrolment. Supports health insurance, 401(k), commuter benefits, and custom benefit plans.

### TriNet (formerly Zenefits)
- **Description:** PEO and HR platform targeting SMBs. Post-acquisition, Zenefits operates as TriNet HR Plus. Offers benefits administration, payroll, and compliance tools.
- **API Documentation:** https://developers.zenefits.com/reference/getting-started
- **SDKs/Libraries:** REST-native
- **Developer Guide:** https://developers.zenefits.com/
- **Standards:** REST/JSON; OAuth 2.0 based
- **Authentication:** OAuth 2.0
- **Benefits Endpoints:** Five APIs covering employees, companies, payroll, money, and time off. Benefits data accessible through the employee data API; enrolment and deduction data available to authenticated customers.

### Finch
- **Description:** Unified employment API aggregating 250+ HRIS and payroll integrations into a single data model. Provides a standardised interface for benefits data across ADP, Workday, BambooHR, Gusto, and others.
- **API Documentation:** https://developer.tryfinch.com/
- **SDKs/Libraries:** JavaScript, Python, Java, Kotlin, Go
- **Developer Guide:** https://developer.tryfinch.com/how-finch-works/unified-employment-api-glossary
- **Standards:** REST/JSON; unified data model
- **Authentication:** OAuth 2.0; SOC 2 Type II, HIPAA, GDPR, CCPA compliant
- **Benefits Endpoints:** Deductions product group provides standardised access to employee benefit deductions and employer contributions. The `Benefit` object represents enrolled benefits; `Deduction` represents employee withholdings; `Contribution` represents employer-paid amounts. Data syncs on periodic cadence depending on automated vs. assisted integrations.

### Merge
- **Description:** Unified HRIS and payroll API providing a single integration layer across 50+ HR platforms. Exposes standardised `Benefit` and `EmployerBenefit` objects.
- **API Documentation:** https://docs.merge.dev/hris/
- **SDKs/Libraries:** REST-based; multiple language SDKs
- **Developer Guide:** https://www.merge.dev/blog/guide-to-hris-api-integrations
- **Standards:** REST/JSON; one authentication model, consistent pagination and rate limiting across all providers
- **Authentication:** OAuth 2.0 via Merge Link; API key for server-to-server
- **Benefits Endpoints:** `Benefit` object represents employee-enrolled benefits (coverage, plan type, deductions). `EmployerBenefit` object represents benefit plans offered by the company. Fully-searchable integration logs and automatic issue detection.

### Unified.to
- **Description:** Real-time unified API for B2B SaaS covering HRIS, benefits, payroll, time off, and more from major platforms (BambooHR, Workday, ADP, Gusto).
- **API Documentation:** https://docs.unified.to/hris/overview
- **SDKs/Libraries:** Available via developer documentation
- **Developer Guide:** https://unified.to/integrations/rippling
- **Standards:** REST/JSON; unified data model across providers
- **Authentication:** OAuth 2.0
- **Benefits Endpoints:** Employee benefit information, eligibility, and enrolment status accessible through unified HRIS endpoints. Supports creating and reading benefit enrolment records across connected HR systems.

### BambooHR
- **Description:** HRIS platform popular with SMBs. Offers a marketplace of benefits administration integrations rather than native benefits tools; provides open REST API for employee data access.
- **API Documentation:** https://documentation.bamboohr.com/reference/get-employee
- **SDKs/Libraries:** REST-native; HTTP Basic Auth
- **Developer Guide:** https://documentation.bamboohr.com/docs/past-changes-to-the-api
- **Standards:** REST with JSON or XML responses
- **Authentication:** HTTP Basic Auth with API key
- **Benefits Endpoints:** `GET /api/v1/benefit/employee_benefit` returns employee enrolment records filterable by employee ID, benefit plan ID, or effective date. Response includes coverage level, enrolment status, deduction dates, and employee/company contribution amounts.

### Humana (Explanation of Benefits API)
- **Description:** Carrier-side FHIR-based API for accessing Explanation of Benefits (EOB) data. Example of a health insurance payer exposing claims and benefits data via HL7 FHIR R4.
- **API Documentation:** https://developers.humana.com/explanation-of-benefits-api/doc
- **SDKs/Libraries:** FHIR client libraries (HAPI FHIR, firely-net-sdk)
- **Developer Guide:** Available via Humana Developer Portal
- **Standards:** HL7 FHIR R4; REST/JSON
- **Authentication:** OAuth 2.0
- **Benefits Endpoints:** ExplanationOfBenefit FHIR resource providing claims payment details, adjustment reasons, and coverage information. Represents the new generation of carrier APIs replacing EDI 835 for EOB data delivery.

---

## Notes

### Evolving Landscape: EDI to FHIR Transition
The benefits administration space is undergoing a significant transition from batch-oriented ANSI X12 EDI transactions (834, 835, 837) to real-time REST/FHIR APIs. CMS mandates are accelerating payer adoption of FHIR R4/R5, and carriers like Humana, Cigna, and UnitedHealth are publishing FHIR APIs alongside legacy EDI systems. New benefits platforms should support both EDI (for backward compatibility with existing carrier connections) and FHIR (for forward-looking integrations).

### Unified API Layer Opportunity
The proliferation of HR/payroll platforms (Workday, ADP, Gusto, Rippling, BambooHR, TriNet, UKG, Paychex, etc.) with incompatible APIs has created a robust market for unified API aggregators (Finch, Merge, Unified.to, Knit). An AI-native benefits administration platform would benefit from integrating with these aggregators rather than building 1:1 connections to each HRIS platform.

### Compliance Automation Gap
Current market solutions largely automate form generation (1095-C, Form 5500) but rely on manual data validation and correction workflows. The non-discrimination testing requirements of Section 125 cafeteria plans, ACA affordability calculations, and ERISA compliance audits remain operationally intensive. This is an underserved area for AI-augmented compliance automation.

### International Standards Gap
ISO 30414:2025 provides a global framework for human capital reporting including benefits metrics, but there is no equivalent to the US ANSI X12 / HIPAA EDI ecosystem for non-US benefits data exchange. International benefits administration (multi-country employers) relies largely on proprietary carrier formats and point-to-point integrations. This represents a significant integration challenge and market opportunity.
