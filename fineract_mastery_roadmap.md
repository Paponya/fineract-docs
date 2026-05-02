# Apache Fineract Mastery Roadmap

## From T24 Developer to Fineract White-Label Expert

---

## T24 → Fineract Concept Map

| T24 Concept                   | Fineract Equivalent                 | Where in Code                           |
| ----------------------------- | ----------------------------------- | --------------------------------------- |
| APPLICATION / VERSION         | REST API Endpoints (JAX-RS)         | `portfolio/*/api/`                      |
| ENQUIRY                       | Read Platform Services              | `*ReadPlatformService.java`             |
| LOCAL.TABLE / FIELD           | JPA Entities + Liquibase migrations | `domain/` + `db/changelog/`             |
| ROUTINE (jBC)                 | Command Handlers + Service layer    | `handler/` + `service/`                 |
| COB (Close of Business)       | Spring Batch COB Jobs               | `fineract-cob/`                         |
| OVERRIDE / LOCAL.REF          | Custom Modules (`/custom/`)         | `custom/{company}/{category}/{module}`  |
| T24.PRODUCT                   | Loan/Savings Product configuration  | `loanproduct/`, `savings/` product APIs |
| AA (Arrangement Architecture) | Loan/Savings Account lifecycle      | `loanaccount/`, `savings/`              |
| STMT.ENTRY / CATEG.ENTRY      | Journal Entries (double-entry)      | `fineract-accounting/`                  |
| EB.API                        | Fineract REST API + Swagger         | `/fineract-provider/api/v1/`            |
| Multi-company                 | Multi-tenancy (tenant per DB)       | `FineractPlatformTenant.java`           |
| OFS Messages                  | Command Source / Command Wrapper    | `fineract-command/`                     |
| COMPANY / BRANCH              | Office hierarchy                    | `organisation/office/`                  |

---

## Phase 1: Environment Setup & First Run (Days 1-3)

### Prerequisites

- Java 21 (Azul Zulu recommended)
- PostgreSQL 18+ (run via Docker)
- IntelliJ IDEA (recommended IDE)
- Docker Desktop

### Quick Start (Your cloned repo at `D:\AI\fineract`)

```powershell
# Start PostgreSQL in Docker
docker run --name postgres -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=postgres -d postgres:18.3

# Create databases
.\gradlew createPGDB -PdbName=fineract_tenants
.\gradlew createPGDB -PdbName=fineract_default

# Set environment variables (PowerShell)
$env:FINERACT_DEFAULT_TENANTDB_PORT="5432"
$env:FINERACT_HIKARI_DRIVER_SOURCE_CLASS_NAME="org.postgresql.Driver"
$env:FINERACT_HIKARI_JDBC_URL="jdbc:postgresql://localhost:5432/fineract_tenants"
$env:POSTGRES_PASSWORD="postgres"
$env:FINERACT_HIKARI_PASSWORD="postgres"
$env:FINERACT_DEFAULT_TENANTDB_PWD="postgres"

# Run Fineract
.\gradlew devRun
```

### Verify

```powershell
# Health check
curl --insecure https://localhost:8443/fineract-provider/actuator/health

# Authenticated test (default: mifos/password)
curl --insecure https://localhost:8443/fineract-provider/api/v1/clients `
  -H "Content-Type: application/json" `
  -H "Fineract-Platform-TenantId: default" `
  -H "Authorization: Basic bWlmb3M6cGFzc3dvcmQ="
```

### Swagger UI

Open: `https://localhost:8443/fineract-provider/swagger-ui/index.html`

### Frontend (Mifos X Web App)

```powershell
git clone https://github.com/openMF/web-app.git
cd web-app
npm install
npm start
# Opens at http://localhost:4200 — login: mifos/password
```

---

## Phase 2: Codebase Architecture (Days 4-10)

### Module Map

```
fineract/
├── fineract-core/          # Foundation: commands, security, multi-tenancy, infrastructure
├── fineract-security/      # OAuth2, Basic Auth, permissions
├── fineract-command/        # Command pattern framework (like T24 OFS)
├── fineract-command-jdbc/   # JDBC-based command persistence
├── fineract-command-async/  # Async command processing
├── fineract-accounting/     # Chart of Accounts, GL, Journal Entries
├── fineract-loan/           # Loan domain: entities, schedules, delinquency
├── fineract-progressive-loan/ # Advanced progressive loan schedule engine
├── fineract-savings/        # Savings domain: accounts, interest, charges
├── fineract-cob/            # Close of Business batch processing
├── fineract-charge/         # Fee/charge definitions
├── fineract-tax/            # Tax group/component management
├── fineract-branch/         # Branch/office management
├── fineract-investor/       # Loan investor/participation features
├── fineract-document/       # Document management
├── fineract-report/         # Reporting engine
├── fineract-mix/            # MIX Market reporting
├── fineract-validation/     # Input validation framework
├── fineract-provider/       # Main Spring Boot app — assembles everything
├── fineract-war/            # WAR packaging (for Tomcat deployment)
├── fineract-client/         # Auto-generated API client SDK
├── fineract-avro-schemas/   # Avro schemas for external events
├── custom/                  # YOUR CUSTOMIZATIONS GO HERE
│   └── acme/                # Example custom module structure
├── integration-tests/       # Integration test suite
├── fineract-e2e-tests-*/    # Cucumber E2E tests
└── fineract-doc/            # AsciiDoc documentation source
```

### Layered Architecture Per Domain Module

Each business domain (loans, savings, clients, etc.) follows this **consistent pattern**:

```
portfolio/{domain}/
├── api/           # REST controllers (JAX-RS @Path resources)
├── data/          # DTOs, read-only data objects
├── domain/        # JPA entities, repositories, domain logic
├── handler/       # Command handlers (Create/Update/Delete)
├── service/       # Business logic services
├── serialization/ # JSON deserialization & validation
├── exception/     # Domain-specific exceptions
├── mapper/        # Entity ↔ DTO mappers
└── starter/       # Spring auto-configuration
```

> **T24 Parallel**: This is like having a separate APPLICATION file (api), ENQUIRY (service read), ROUTINE (handler + service write), and LOCAL.TABLE (domain) per module.

### The Command Pattern (Critical to Understand)

Fineract routes **all write operations** through a command pattern — similar to T24's OFS messages:

```
HTTP Request → API Resource → CommandWrapper (built)
  → CommandSourceService.processCommand()
    → CommandHandler.processCommand(JsonCommand)
      → Domain Service (business logic)
        → JPA Entity (state change)
          → Event published
```

**Key files to study:**

- [CommandWrapper.java](file:///d:/AI/fineract/fineract-core/src/main/java/org/apache/fineract/commands/domain/CommandWrapper.java) — Wraps every command with entity/action/resource metadata
- [NewCommandSourceHandler.java](file:///d:/AI/fineract/fineract-core/src/main/java/org/apache/fineract/commands/handler/NewCommandSourceHandler.java) — Interface all handlers implement
- [CommandWrapperConstants.java](file:///d:/AI/fineract/fineract-core/src/main/java/org/apache/fineract/commands/domain/CommandWrapperConstants.java) — All action/entity constants

### Multi-Tenancy Model

```
┌─────────────────┐     ┌──────────────────────┐
│ HTTP Request     │     │ fineract_tenants DB  │ (shared)
│ Header:          │────▶│ ┌──────────────────┐ │
│ Fineract-Platform│     │ │ tenant_id: "abc" │ │
│ -TenantId: abc   │     │ │ db_name, db_host │ │
└─────────────────┘     │ │ db_port, db_user │ │
                         │ └──────────────────┘ │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ fineract_abc DB       │ (isolated)
                         │ All client, loan,     │
                         │ savings, accounting   │
                         │ data for tenant "abc" │
                         └──────────────────────┘
```

Each tenant gets its **own database** — complete data isolation. The `Fineract-Platform-TenantId` HTTP header selects which tenant DB to use per request.

---

## Phase 3: Business Domain Deep Dive (Days 11-30)

### 3.1 Organisation Setup (Like T24 COMPANY/DEPT)

**Study order → Code path:**

| Step                       | API                     | Key Code                    |
| -------------------------- | ----------------------- | --------------------------- |
| 1. Create Office hierarchy | `POST /offices`         | `organisation/office/`      |
| 2. Define currencies       | `PUT /currencies`       | `organisation/monetary/`    |
| 3. Create staff            | `POST /staff`           | `organisation/staff/`       |
| 4. Configure holidays      | `POST /holidays`        | `organisation/holiday/`     |
| 5. Set working days        | `PUT /workingdays`      | `organisation/workingdays/` |
| 6. Chart of Accounts       | `POST /glaccounts`      | `fineract-accounting/`      |
| 7. Payment types           | `POST /paymenttypes`    | `infrastructure/codes/`     |
| 8. Create users & roles    | `POST /users`, `/roles` | `useradministration/`       |

### 3.2 Loan Lifecycle (Like T24 LD/AA)

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Product  │──▶│ Apply    │──▶│ Approve  │──▶│ Disburse │
│ Setup    │   │ (submit) │   │          │   │          │
└──────────┘   └──────────┘   └──────────┘   └────┬─────┘
                                                   │
    ┌──────────┐   ┌──────────┐   ┌──────────┐   │
    │ Close /  │◀──│ Repay    │◀──│ Active   │◀──┘
    │ Write-off│   │          │   │ (COB     │
    └──────────┘   └──────────┘   │ accruals)│
                                   └──────────┘
```

**Key code files to study (in order):**

1. **Product Definition**: `fineract-provider/.../loanproduct/` — Like T24 AA.PRODUCT.DESIGNER
2. **Application**: [LoanApplicationWritePlatformServiceJpaRepositoryImpl.java](file:///d:/AI/fineract/fineract-provider/src/main/java/org/apache/fineract/portfolio/loanaccount/service/LoanApplicationWritePlatformServiceJpaRepositoryImpl.java) (~50K lines — core application logic)
3. **Schedule Generation**: `fineract-loan/.../loanschedule/` — Repayment schedule calculator
4. **Disbursement**: [LoanDisbursementService.java](file:///d:/AI/fineract/fineract-provider/src/main/java/org/apache/fineract/portfolio/loanaccount/service/LoanDisbursementService.java)
5. **Repayment Processing**: [LoanWritePlatformServiceJpaRepositoryImpl.java](file:///d:/AI/fineract/fineract-provider/src/main/java/org/apache/fineract/portfolio/loanaccount/service/LoanWritePlatformServiceJpaRepositoryImpl.java) (~222K — the biggest file, handles all loan transactions)
6. **Accruals**: [LoanAccrualsProcessingServiceImpl.java](file:///d:/AI/fineract/fineract-provider/src/main/java/org/apache/fineract/portfolio/loanaccount/service/LoanAccrualsProcessingServiceImpl.java)
7. **API Layer**: [LoansApiResource.java](file:///d:/AI/fineract/fineract-provider/src/main/java/org/apache/fineract/portfolio/loanaccount/api/LoansApiResource.java) — All loan REST endpoints

### 3.3 Savings Lifecycle (Like T24 ACCOUNT/SAVINGS)

```
Product Setup → Open Account → Activate → Deposit/Withdraw → Interest Calc → Close
```

**Key files:**

- `fineract-provider/.../savings/api/SavingsAccountsApiResource.java`
- `fineract-savings/.../savings/domain/` — Entities & business rules
- Fixed Deposits, Recurring Deposits also under `savings/api/`

### 3.4 Accounting (Like T24 STMT.ENTRY)

Fineract uses **double-entry accounting**. Loan/Savings products are linked to GL accounts via Product-Account Mapping.

**Key path**: `fineract-accounting/` + `fineract-provider/.../accounting/`

| Accounting Type    | Description                           |
| ------------------ | ------------------------------------- |
| None               | No GL entries                         |
| Cash-based         | Entries on cash movement              |
| Accrual (periodic) | Income recognized over time           |
| Accrual (upfront)  | All income recognized at disbursement |

### 3.5 Close of Business (Like T24 COB)

The `fineract-cob/` module handles nightly batch processing via **Spring Batch**:

- Loan interest accrual
- Delinquency classification
- Overdue penalty application
- Account aging

COB can run on multiple worker nodes via ActiveMQ or Kafka for scalability.

---

## Phase 4: Customization & White-Labeling (Days 31-60)

### 4.1 The Custom Module Framework

Fineract provides a `/custom/` directory with auto-discovery. Structure:

```
custom/
└── {your-company}/           # e.g., "hillarybank"
    └── {category}/           # e.g., "loan", "savings", "kyc"
        └── {module}/         # e.g., "custom-interest-calc"
            ├── build.gradle
            ├── dependencies.gradle
            └── src/
                └── main/
                    ├── java/         # Your custom code
                    └── resources/
                        └── db/changelog/  # Custom Liquibase migrations
```

This is auto-discovered by `settings.gradle` lines 89-101 — no core code changes needed.

### 4.2 Extension Strategies (Do NOT Modify Core)

| Strategy                  | Use Case                       | How                                        |
| ------------------------- | ------------------------------ | ------------------------------------------ |
| **Override a Service**    | Replace default business logic | `@ConditionalOnMissingBean` + your `@Bean` |
| **Custom COB Step**       | Add nightly processing         | Implement `COBBusinessStep` interface      |
| **Custom API Endpoint**   | New functionality              | Add JAX-RS `@Path` resource in your module |
| **Custom DB Tables**      | Store extra data               | Liquibase changesets in your module        |
| **Custom Event Listener** | React to business events       | Listen to Spring/Kafka events              |
| **Custom Loan Schedule**  | Different interest calc        | Implement schedule generator interface     |

### 4.3 Building Your White-Label Product

**Step 1: Fork Strategy**

```
apache/fineract (upstream) ──pull──▶ your-org/fineract-core (never modify)
                                            │
                                    custom/{your-company}/ (all customizations)
                                            │
                                    your-org/web-app (branded Angular frontend)
```

**Step 2: Custom Frontend (Angular)**

- Clone `https://github.com/openMF/web-app` (Angular + Material)
- Rebrand: logos, colors, themes in `src/assets/` and `angular.json`
- Add/remove modules per client requirements
- Build separate Angular apps per white-label client if needed

**Step 3: Multi-Tenant Deployment**

```
┌───────────────────────────────────────┐
│ Your White-Label Platform             │
│                                       │
│  ┌─────────┐  ┌─────────┐  ┌───────┐│
│  │ Bank A  │  │ Bank B  │  │Bank C ││
│  │ tenant  │  │ tenant  │  │tenant ││
│  │ (own DB)│  │ (own DB)│  │(own DB)││
│  └─────────┘  └─────────┘  └───────┘│
│                                       │
│  Single Fineract Instance + Custom    │
│  Modules + Branded Frontend           │
└───────────────────────────────────────┘
```

**Step 4: Dockerized Deployment**

```dockerfile
# custom/docker/Dockerfile includes your modules
FROM apache/fineract:latest
COPY custom-modules/*.jar /app/libs/
```

Or build with: `.\gradlew :custom:docker:jibDockerBuild`

---

## Phase 5: Production Readiness (Days 61-90)

### Checklist

- [ ] **Security**: OAuth2 setup, HTTPS, role-based access, API rate limiting
- [ ] **Database**: Connection pooling (HikariCP), read replicas, backup strategy
- [ ] **Monitoring**: Actuator endpoints, Prometheus metrics, Grafana dashboards
- [ ] **Logging**: Structured logging, Loki/ELK stack integration
- [ ] **COB Scaling**: Kafka/ActiveMQ for distributed batch processing
- [ ] **CI/CD**: GitHub Actions / Jenkins pipeline for build + test + deploy
- [ ] **Testing**: Unit tests (`.\gradlew test`), Integration tests, E2E Cucumber tests
- [ ] **Compliance**: Audit trail (command_source table logs everything), data encryption

### Key Configuration

All configuration is via environment variables prefixed `FINERACT_*`:

- `FINERACT_HIKARI_*` — Database connection pool
- `FINERACT_DEFAULT_TENANTDB_*` — Default tenant database
- `FINERACT_REMOTE_JOB_MESSAGE_HANDLER_*` — COB message broker

---

## Recommended Study Path (File-by-File)

### Week 1: Foundation

1. [ServerApplication.java](file:///d:/AI/fineract/fineract-provider/src/main/java/org/apache/fineract/ServerApplication.java) — Entry point
2. [FineractPlatformTenant.java](file:///d:/AI/fineract/fineract-core/src/main/java/org/apache/fineract/infrastructure/core/domain/FineractPlatformTenant.java) — Multi-tenancy
3. [CommandWrapper.java](file:///d:/AI/fineract/fineract-core/src/main/java/org/apache/fineract/commands/domain/CommandWrapper.java) — Command pattern
4. [CommandWrapperConstants.java](file:///d:/AI/fineract/fineract-core/src/main/java/org/apache/fineract/commands/domain/CommandWrapperConstants.java) — All operations catalog

### Week 2: Client & Organisation

5. `fineract-provider/.../portfolio/client/` — Client management (start with `api/`, then `service/`, then `domain/`)
6. `fineract-provider/.../organisation/office/` — Office hierarchy
7. `fineract-provider/.../useradministration/` — Users, roles, permissions

### Week 3: Loans

8. `fineract-provider/.../portfolio/loanproduct/` — Product configuration
9. `fineract-provider/.../portfolio/loanaccount/api/LoansApiResource.java` — All loan endpoints
10. `fineract-loan/.../portfolio/loanaccount/domain/` — Loan entity model
11. `fineract-provider/.../portfolio/loanaccount/service/LoanApplicationWritePlatformServiceJpaRepositoryImpl.java` — Application flow

### Week 4: Savings & Accounting

12. `fineract-provider/.../portfolio/savings/api/SavingsAccountsApiResource.java`
13. `fineract-savings/.../portfolio/savings/domain/` — Savings entities
14. `fineract-accounting/.../accounting/` — GL, journal entries, closures

### Week 5: COB & Events

15. `fineract-cob/` — Batch processing framework
16. `fineract-avro-schemas/` — External event schemas
17. `fineract-provider/.../infrastructure/event/` — Event publishing

### Week 6: Customization

18. `custom/acme/` — Study the example custom module
19. Create your first custom module under `custom/{your-company}/`
20. `settings.gradle` lines 89-101 — Auto-discovery mechanism

---

## Key Resources

| Resource         | URL                                                                        |
| ---------------- | -------------------------------------------------------------------------- |
| Official Docs    | https://fineract.apache.org/docs/current                                   |
| Swagger (live)   | https://sandbox.mifos.community/fineract-provider/swagger-ui/index.html    |
| Mifos X Web App  | https://github.com/openMF/web-app                                          |
| Wiki             | https://cwiki.apache.org/confluence/display/FINERACT                       |
| Mailing List     | https://fineract.apache.org/#contribute                                    |
| Slack            | https://app.slack.com/client/T0F5GHE8Y/C028634A61L                         |
| JIRA             | https://issues.apache.org/jira/secure/Dashboard.jspa?selectPageId=12335824 |
| Fineract Academy | https://fineract-academy.com                                               |

---

## Advantages

1. **Product configuration thinking** — T24's AA architecture maps well to Fineract's product-based approach
2. **COB/batch understanding** — You already understand end-of-day processing concepts
3. **Multi-entity/multi-currency** — T24's multi-company model maps to Fineract's multi-tenancy
4. **Accounting integration** — T24's tight GL integration is mirrored in Fineract's accounting module
5. **Regulatory compliance** — Your experience with banking regulations transfers directly

**Key differences to adapt to:**

- Fineract is **API-first** (no built-in UI) vs T24's integrated Browser/Classic
- Fineract uses **Spring Boot + JPA** vs T24's jBC + Universe DB
- Fineract's customization is via **separate modules** vs T24's LOCAL.REF/OVERRIDE
- Fineract is **open source** — you can read, modify, and redistribute everything
