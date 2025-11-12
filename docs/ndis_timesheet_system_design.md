# Disability Support Timesheet System Design (Australia, 2025-26)

## 1. Introduction
This document outlines a full-stack system design for a disability support organisation in Australia to manage timesheets, aligned with the National Disability Insurance Scheme (NDIS) Pricing Arrangements and Price Limits 2025-26 (effective 1 July 2025, incorporating amendments through 24 November 2025). The solution spans a responsive web application and cross-platform mobile applications (iOS and Android) targeting support workers, administrators/managers, and optional read-only client access.

The goals are to:
- Simplify timesheet capture, approval, and reporting.
- Ensure compliance with NDIS billing rules, worker qualification-based rates, and travel claims.
- Maintain robust security, privacy, and accessibility to meet Australian regulatory expectations.

## 2. High-Level Architecture

```
+-------------------------------+          +-----------------------------+
|           Frontend            |          |         Mobile Apps         |
| (React Web, React Native App) |          |  (React Native / Flutter)   |
+-------------------------------+          +-----------------------------+
            |        ^                               |         ^
            v        |                               v         |
      +-----------------------+              +-----------------------+
      |   API Gateway (REST)  |<------------>|   Sync Service (API)  |
      +----------+------------+              +-----------+-----------+
                 |                                       |
                 v                                       v
         +---------------+                   +-----------------------+
         |  Backend API  |                   | Offline Storage (App) |
         | (NestJS/Node) |                   | (SQLite/Encrypted)    |
         +-------+-------+                   +-----------------------+
                 |
                 v
        +----------------------+      +------------------+
        | Business Services    |<---->|  Rate Service     |
        | (Timesheets, Notes,  |      | (NDIS tables,     |
        |  Approvals, Reports) |      |  rules engine)    |
        +-----+----------------+      +------------------+
              |
              v
    +-----------------------+
    | Persistence Layer     |
    | (PostgreSQL w/ RLS)   |
    +----------+------------+
               |
               v
    +--------------------------+           +------------------------+
    | Object Storage (S3/Azure |           | Integration Layer      |
    |  Blob) for attachments   |           | (PRODA, Payroll, Email |
    +--------------------------+           |  SMS, Push)            |
                                           +------------------------+
```

### Key Components
- **Frontend Web App**: Responsive React.js with TypeScript, served via CDN. Uses API Gateway for authenticated REST/GraphQL calls. Implements WCAG 2.1 AA accessibility.
- **Mobile Apps**: Built with React Native (or Flutter). Support offline entry via encrypted SQLite, background sync, and push notifications via Firebase Cloud Messaging (FCM) / Apple Push Notification service (APNs).
- **Backend API**: NestJS (Node.js + TypeScript) or Django REST Framework. Provides REST endpoints, websockets for notifications. Implements role-based access control (RBAC) and rate calculations.
- **Rate Service**: Rule engine referencing NDIS support item tables configurable via admin UI. Applies rate multipliers for day/time, worker qualification, and location (metro/regional/remote).
- **Persistence**: PostgreSQL with row-level security (RLS) for data segregation. Audit tables for compliance.
- **Integrations**: Optional PRODA claim upload, payroll system exports (Xero, MYOB), email/SMS, calendar sync (Microsoft Graph/Google Calendar).
- **Infrastructure**: Hosted on AWS (preferred) using ECS Fargate or EKS, RDS PostgreSQL, S3, CloudFront, AWS Cognito or custom auth. Azure equivalents possible.

## 3. Detailed Components

### 3.1 Authentication and Authorisation
- **Identity Provider**: AWS Cognito (or Auth0) with email/password, MFA (TOTP/SMS), and OAuth2/OIDC. Provides hosted UI for password reset, onboarding.
- **Access Tokens**: JWT with scopes per role (worker, manager, client). Backend verifies tokens, enforces RBAC via NestJS Guards.
- **Onboarding Workflow**:
  1. Admin invites worker; worker completes profile (qualifications, certifications, ABN if contractor).
  2. Upload mandatory documents (NDIS Worker Screening, First Aid). Stored securely in S3 with KMS encryption.
  3. Manager allocates clients and support item templates.

### 3.2 Timesheet Management
- **Shift Creation**: Worker selects allocated client, support item code (e.g., `01_011_0107_1_1` Assistance with Daily Life) from filtered list.
- **Rate Determination**: Rate Service references pricing table with columns: support item code, day type, time band, worker level (standard, level 2, high intensity), location modifier. Example rates (2025-26 guide):
  - Weekday Daytime (6am-8pm) standard personal care: $62.55/hr.
  - Weekday Evening (8pm-12am): $68.97/hr.
  - Saturday: $86.10/hr.
  - Sunday: $109.25/hr.
  - Public Holiday: $136.50/hr.
- **Travel Claims**: Capture kilometres (up to 20km one-way, 40km return) at $0.97/km for standard vehicle, $2.76/km for modified vehicle as per guide. Additional time-based travel support items handled by separate entries.
- **Validations**:
  - Worker must be allocated to client for date/time.
  - Prevent overlapping shifts.
  - Enforce caps: e.g., max 24 hours in 36-hour period (fatigue management) or configurable per policy.
  - Ensure notes required when incident flagged.
- **Draft vs Submitted**: Draft stored locally/offline and server-side. Submitted shifts locked pending approval.
- **Audit**: Each update logged with user, timestamp, changes (before/after).

### 3.3 Notes and Collaboration
- **Shared Notes**: Filterable by client, tag (health, behaviour), date. Markdown or rich text with attachments (photos, documents). Access restricted to workers assigned to client; managers have override.
- **Notifications**: Workers subscribe to clients; new notes trigger push/email. Use AWS SNS + Firebase/Apple push.
- **Privacy Controls**: Redact identifying info for other clients, restrict downloads for sensitive attachments, watermark PDF exports.

### 3.4 Reporting & Approvals
- **Dashboards**:
  - Worker: Weekly summary (hours, km, earnings), status (draft, submitted, approved).
  - Manager: Pending approvals, flagged discrepancies (e.g., rate overrides), resource utilisation.
- **Approval Workflow**:
  1. Worker submits timesheet.
  2. Manager reviews, can approve, reject with comments, or request modification.
  3. Approved timesheets flow to billing/payroll integration queue.
- **Exports**: PDF/CSV summarising support item codes, rate, quantity, total, GST status (NDIS supports GST-free). PRODA upload file (CSV/XML) and payroll integration (Xero timesheet API, MYOB import).
- **Analytics**: Power BI or AWS QuickSight dashboards via secure data warehouse (Redshift/Snowflake). Anonymised datasets for compliance reporting.

### 3.5 Offline Capability
- Mobile apps store drafted timesheets and notes in encrypted SQLite (SQLCipher). Use Redux Persist or equivalent. Sync engine queues operations, resolves conflicts (last-write-wins with audit). For attachments, store temporarily encrypted until uploaded.

### 3.6 Security & Compliance
- Data residency in Australian region (AWS ap-southeast-2). Encryption in transit (TLS 1.2+), at rest (KMS, AES-256). Regular penetration tests, vulnerability scanning (OWASP ZAP). Logging via CloudWatch / ELK with immutable audit trail (AWS Audit Manager).
- Privacy compliance with Australian Privacy Principles (APP): consent management, data minimisation, breach response plan.
- NDIS Code of Conduct: training modules, record of compliance in worker profiles, mandatory policy acknowledgements.

### 3.7 Scalability & Reliability
- Microservice-ready but start with modular monolith. Auto-scaling containers, multi-AZ database. Caching frequently used rate tables using Redis/ElastiCache. Feature flag service (LaunchDarkly) for progressive rollout.
- CI/CD via GitHub Actions, Terraform for infrastructure as code, automated backups (daily snapshots, point-in-time recovery).

## 4. Database Schema (Relational)

### Core Tables
- **users** (`id`, `email`, `password_hash`, `role`, `status`, `cognito_sub`, `phone`, `mfa_enabled`, timestamps).
- **user_profiles** (`user_id`, `first_name`, `last_name`, `date_of_birth`, `qualifications`, `worker_level`, `ndis_screening_number`, `cert_expiry`, `emergency_contact`, `address`, `aboriginal_torres_status`, timestamps).
- **clients** (`id`, `ndis_number`, `first_name`, `last_name`, `plan_manager`, `service_agreement_id`, `address`, `privacy_flags`, timestamps).
- **client_allocations** (`id`, `client_id`, `worker_id`, `start_date`, `end_date`, `access_level`).
- **support_items** (`id`, `code`, `description`, `category`, `is_travel`, `unit`, `default_rate`, `effective_from`, `effective_to`).
- **ndis_rates** (`id`, `support_item_id`, `day_type`, `time_band`, `worker_level`, `location_modifier`, `rate`, `km_rate`, `effective_from`, `effective_to`).
- **timesheets** (`id`, `worker_id`, `week_start`, `status`, `submitted_at`, `approved_at`, `approved_by`, `total_amount`, `total_hours`, `total_km`).
- **timesheet_entries** (`id`, `timesheet_id`, `client_id`, `support_item_id`, `date`, `start_time`, `end_time`, `duration_minutes`, `day_type`, `time_band`, `worker_level`, `rate`, `quantity`, `km`, `km_rate`, `km_amount`, `notes`, `incident_flag`, `location`, `created_offline`, `synced_at`).
- **approvals** (`id`, `timesheet_id`, `manager_id`, `status`, `comments`, `actioned_at`).
- **notes** (`id`, `client_id`, `author_id`, `title`, `content`, `note_type`, `visibility`, `created_at`, `updated_at`).
- **note_access_logs** (`id`, `note_id`, `viewer_id`, `viewed_at`, `action`).
- **attachments** (`id`, `entity_type`, `entity_id`, `file_url`, `file_type`, `storage_key`, `checksum`, `uploaded_by`, `created_at`).
- **travel_logs** (`id`, `timesheet_entry_id`, `start_location`, `end_location`, `km_claimed`, `vehicle_type`, `gps_trace`, `created_at`).
- **audit_logs** (`id`, `entity`, `entity_id`, `action`, `performed_by`, `before`, `after`, `timestamp`).
- **integrations** (`id`, `type`, `config`, `status`, `last_synced`).
- **feature_flags**, **system_settings** (for configurable thresholds, rate versioning).

### Indexing & Constraints
- Unique constraint on `support_items (code, effective_from)`.
- Partial indexes for active allocations (`end_date IS NULL OR end_date >= current_date`).
- Foreign keys enforcing referential integrity, cascade updates.
- Row-level security policies: workers can access their own records; managers by region/team.

## 5. Wireframe Descriptions

1. **Worker Dashboard (Web/Mobile)**:
   - Header with total hours, km, estimated earnings this week.
   - Tabs: "Timesheets", "Notes", "Profile".
   - Timesheets tab shows cards for each week (status chips: Draft, Submitted, Approved). CTA button “Create Timesheet”.

2. **Weekly Timesheet View**:
   - Calendar grid Monday-Sunday. Each day cell shows existing entries with client initials and support item icon.
   - Floating action button “Add Shift”.
   - Add Shift form: Client dropdown, date picker, start/end time sliders, support item list filtered by client plan. Rate preview box showing base rate, multiplier, total. Travel km input with toggle for GPS auto-capture. Notes text area with voice input button (mobile).

3. **Manager Approval Screen**:
   - Filter panel (worker, client, status, date range).
   - Table listing timesheets with totals and variance flags. Selecting row opens side panel with detailed entries, notes, travel logs. Buttons for Approve, Reject (requires reason), Request Info.

4. **Client Notes Timeline**:
   - Timeline layout with date separators. Cards show author, time, tags, summary. Search bar and tag filters on top. Access log icon indicates viewers.

5. **Settings – Rate Management**:
   - Admin UI showing current NDIS rate tables by effective period. Buttons to import new CSV from NDIS dataset, review changes, schedule go-live date. Diff view highlighting updated rates.

## 6. Tech Stack Recommendations

| Layer | Technology | Rationale |
| --- | --- | --- |
| Web Frontend | React.js + TypeScript, Next.js for SSR | Mature ecosystem, component reuse, accessibility tooling, SSR for performance |
| Mobile Apps | React Native + TypeScript (Expo or bare) | Code-sharing with web logic, strong community, native performance, supports offline features |
| Backend | NestJS (Node.js) + TypeScript | Structured modular architecture, TypeScript end-to-end consistency, supports GraphQL/REST |
| Database | PostgreSQL 15 + PostGIS | Reliable relational DB, support for complex queries, geospatial for travel validation |
| Caching | Redis (AWS ElastiCache) | Rate table caching, session store |
| Storage | AWS S3 with KMS | Secure document storage |
| Auth | AWS Cognito | Managed MFA, compliance, seamless mobile integration |
| Infrastructure | AWS ECS Fargate, API Gateway, CloudFront, Route 53, WAF | Scalability, security (shield, WAF), serverless containers |
| CI/CD | GitHub Actions + Terraform + AWS CodePipeline | Automated testing/deployment, infra as code |
| Monitoring | CloudWatch, AWS X-Ray, Sentry, Datadog | Observability, tracing, error tracking |
| Analytics | AWS QuickSight / Power BI via secure data export | Business intelligence and compliance reports |
| Testing | Jest, React Testing Library, Detox/E2E, OWASP ZAP | Automated coverage, security scans |

## 7. Development Roadmap

### Phase 0 – Discovery & Compliance (Weeks 0-2)
- Stakeholder workshops, user journey mapping.
- Review NDIS pricing guide 2025-26, confirm support categories.
- Security/privacy assessment, threat modelling.
- UX research, design system foundations (Figma components).

### Phase 1 – MVP Foundations (Weeks 3-6)
- Set up repo, CI/CD, infrastructure baseline (dev/staging).
- Implement auth (Cognito integration), user roles, onboarding flows.
- Build core database schema and rate import service.
- Develop worker dashboard with weekly timesheet entry (web + mobile) supporting single client per entry, manual rate application.
- Implement offline draft storage in mobile, sync service stub.
- Basic manager approval (approve/reject) and PDF export.
- Unit/integration tests, initial accessibility audit.

### Phase 2 – Compliance & Collaboration (Weeks 7-10)
- Extend rate engine for day/time bands, worker qualifications, location modifiers.
- Add travel claim logic with km caps and vehicle types.
- Implement shared notes with access control and notifications.
- Introduce audit logging, incident reporting, document attachments.
- Build analytics dashboard MVP and CSV exports for PRODA/payroll.
- Harden security (RLS policies, WAF rules), penetration test.

### Phase 3 – Optimisation & Integrations (Weeks 11-14)
- Offline sync conflict resolution, background sync, GPS integration.
- Calendar integrations (Google/Outlook), push notifications.
- PRODA API automation (where available), payroll integration connectors.
- Performance tuning (caching, indexing), load testing.
- Comprehensive accessibility testing, user acceptance testing (UAT) with pilot group.

### Phase 4 – Launch & Operations (Weeks 15-16)
- Production readiness review, data migration, training materials.
- Go-live with support plan (hypercare), monitor metrics.
- Post-launch backlog: client portal, advanced analytics, AI-assisted note summaries.

## 8. Potential Challenges & Mitigations

| Challenge | Mitigation |
| --- | --- |
| Frequent NDIS rate updates | Build configurable rate tables with effective dates. Provide admin import (CSV/JSON) and automated validation against official data. Version control rates and allow retroactive adjustments with audit trail. |
| Offline data conflicts | Implement conflict detection (timestamp + hash). Provide user-friendly resolution UI showing differences. Maintain audit log for overrides. |
| Data privacy & consent | Enforce principle of least privilege, consent management in client profiles, audit logging. Regular staff training and privacy impact assessments. |
| Public holiday & regional rate variations | Integrate Australian public holiday API, allow custom calendars per state/territory. Store remoteness area classification per client/worker to apply relevant loadings. |
| GPS data accuracy | Allow manual override and verification. Store GPS trace hashed/anonymised. Explain to workers how data is used, obtain consent. |
| Accessibility compliance | Adopt accessible design system, test with screen readers (VoiceOver, NVDA). Include accessibility acceptance criteria in Definition of Done. |
| Integration dependencies (PRODA) | Design modular integration layer with retry/backoff, message queue (SQS). Provide manual export fallback. |
| Security incidents | Implement SOC monitoring, incident response plan, regular drills. Use AWS GuardDuty, Security Hub alerts. |

## 9. Cost Estimates (AUD, mid-size org ~120 workers)

### Development (6-month build including discovery and iterations)
- Product/Project Management: $60k
- UX/UI Design: $45k
- Frontend (Web) Development: $120k
- Mobile Development: $140k
- Backend/DevOps Engineering: $160k
- QA & Accessibility Testing: $50k
- Security/Compliance Audits: $30k
- Contingency (15%): ~$90k
- **Total Development Estimate**: ~$695k AUD

### Ongoing Annual Operating Costs
- AWS Infrastructure (prod + staging): $4k/month (~$48k/year) including EC2/ECS, RDS, S3, CloudFront, WAF.
- Authentication (Cognito/SES/SNS): $6k/year.
- Monitoring/Logging (Datadog/Sentry): $12k/year.
- Maintenance & Support Team (1 FTE dev, 0.5 FTE support, 0.5 FTE DevOps): ~$220k/year.
- Security & Compliance (audits, penetration tests): $15k/year.
- Licensing & Tools (Figma, GitHub, CI/CD): $10k/year.
- **Total Annual Opex**: ~$311k AUD.

### Cost Optimisation Opportunities
- Use reserved instances/savings plans for RDS/ECS.
- Evaluate Flutter for shared UI code if team skillset aligns.
- Automate rate imports to reduce manual processing.

## 10. Testing & Compliance Strategy
- **Automated Testing**: Unit (Jest), integration (Supertest), E2E (Cypress/Detox). Accessibility tests via axe-core.
- **Performance Testing**: k6/Gatling for load, ensure sub-2s response for 95th percentile.
- **Security Testing**: Static code analysis (SonarQube), dependency scanning (Dependabot), dynamic testing (OWASP ZAP), penetration tests twice yearly.
- **Compliance Audits**: Annual review against APP, NDIS Practice Standards; maintain documentation of policies and incident responses.
- **Disaster Recovery**: RPO < 1 hour via point-in-time recovery, RTO < 4 hours with warm standby environment.

## 11. Future Enhancements
- AI-assisted note summarisation with consent (Azure OpenAI with PHI safeguards).
- Rostering module with shift bidding and availability.
- Client portal with goal tracking and plan budgets.
- Integration with assistive technologies (switch control support).
- Data warehouse for predictive analytics (staffing optimisation).

## 12. Conclusion
This architecture delivers a secure, scalable, and compliant timesheet solution tailored to Australian disability support providers. By combining configurable NDIS rate management, robust offline-capable mobile apps, and strict adherence to privacy and accessibility requirements, the system empowers workers and administrators to manage service delivery efficiently while meeting regulatory obligations.
