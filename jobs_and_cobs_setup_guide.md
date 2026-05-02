# Apache Fineract: Jobs & Close of Business (COB) Deep Dive

This technical guide provides a comprehensive breakdown of Fineract's asynchronous processing mechanisms: Scheduled Batch Jobs and the Close of Business (COB) architecture.

---

## 1. Batch Jobs & Scheduler Overview

Fineract relies on batch jobs for heavy processing, recurring tasks, and bulk data updates (e.g., applying penalties, calculating accruals, updating arrears). These tasks run in the background without blocking the main API threads.

### The Global Scheduler (`/v1/scheduler`)
Think of the Scheduler as the **main circuit breaker**. It controls the Quartz Scheduler engine itself. It doesn't care about individual tasks; it only dictates whether the automated system is turned on or off.

**Scheduler APIs:**
- `GET /v1/scheduler`: Returns whether the global engine is `Active` or `Standby`.
- `POST /v1/scheduler?command=start`: Turns the main power on. Jobs will now run automatically on their cron schedules.
- `POST /v1/scheduler?command=stop`: Turns the main power off. **No jobs will run automatically**, regardless of their individual schedules.

### Individual Jobs (`/v1/jobs`)
Think of Jobs as the **individual light switches**. These endpoints manage the specific tasks (tasklets/jobs) inside the engine. You use these to configure what a job does, when it does it, and to look at past performance.

**Job APIs:**
- `GET /v1/jobs`: Lists all configured jobs, their specific cron expressions, and active status.
- `GET /v1/jobs/{jobId}`: Retrieves details for a specific job.
- `PUT /v1/jobs/{jobId}`: Updates a job (e.g., changes the Cron expression or toggles its active status).
- `POST /v1/jobs/{jobId}?command=executeJob`: Manually triggers the job immediately. This acts as a manual override, forcing the job to run **right now**, even if the global `/scheduler` is in Standby.
- `GET /v1/jobs/{jobId}/runhistory`: Retrieves the execution history, success/failure status, and start/end times.

---

## 2. Close of Business (COB) Architecture

While regular batch jobs handle isolated tasks, **Close of Business (COB)** is a specialized, highly orchestrated batch process designed for robust end-of-day operations, ensuring account integrity across days.

Fineract categorizes COB execution into three main styles:

### A. Daily COB (Scheduled)
Runs automatically at a predetermined time (usually midnight or the end of the business day). It processes all relevant accounts (loans, savings) for the current business date, ensuring all end-of-day rules (accruals, arrears, dormancy) are applied before the new day begins.

### B. Catch-Up COB
If the Fineract system is offline during the Daily COB window, or if a tenant's business date falls behind, Catch-Up COB is utilized.
Catch-Up COB automatically identifies the oldest unprocessed day and runs the complete COB cycle day-by-day, sequentially advancing the system date until it catches up to the current date.

**Catch-Up APIs:**
- `GET /v1/loans/oldest-cob-closed`: Retrieves the oldest loan account business date that has completed COB.
- `POST /v1/loans/catch-up`: Triggers the Catch-Up COB sequence.
- `GET /v1/loans/is-catch-up-running`: Checks if the Catch-Up process is currently executing.

### C. Inline COB
Inline COB guarantees data consistency on a per-account basis. If a teller attempts to post a transaction to an account that is lagging behind the global business date (e.g., missed Daily COB), Fineract will synchronously trigger "Inline COB" for that specific account. 
It temporarily locks the account, runs the missed end-of-day processes up to the current date, and then successfully processes the teller's transaction.

---

## 3. The Business Step Framework

COB is not a monolithic script; it is composed of modular **Business Steps**. Administrators can configure exactly which steps run during COB and in what order. 

### Job Names and Categories
Fineract defines specific overarching job pipelines for COB, such as:
- `LOAN_COB`
- `SAVINGS_COB`
- `WORKING_CAPITAL_LOAN_COB`

### Configuring Business Steps for LOAN_COB
For a `LOAN_COB`, you can mix, match, and reorder available business steps. Examples of Loan Business Steps include:
- `UpdateLoanArrearsAgingBusinessStep`: Updates the arrears age and amount.
- `ApplyChargeToOverdueLoansBusinessStep`: Applies late fees.
- `AddPeriodicAccrualEntriesBusinessStep`: Posts interest accrual entries.
- `CheckDueInstallmentsBusinessStep`: Validates installments that are due.
- `SetLoanDelinquencyTagsBusinessStep`: Tags loans as delinquent based on aging.

**Business Step APIs:**
- `GET /v1/jobs/names`: Returns available COB job categories (e.g., "LOAN_COB").
- `GET /v1/jobs/{jobName}/available-steps`: Lists all predefined steps in the system available for that job category.
- `GET /v1/jobs/{jobName}/steps`: Retrieves the currently active sequence of steps for the given job.
- `PUT /v1/jobs/{jobName}/steps`: Updates the execution sequence and active steps.

---

## 4. Setup, Verify, Execute, Observe Workflow

This workflow demonstrates how to orchestrate Jobs and COB via Postman or your custom Core Banking integration.

### Phase 1: Verify and Start the Scheduler
Ensure the global batch engine is running.
```http
// 1. Check Status
GET {{baseUrl}}/v1/scheduler
Authorization: Basic {{auth}}

// 2. Start Scheduler (if inactive)
POST {{baseUrl}}/v1/scheduler?command=start
Authorization: Basic {{auth}}
```

### Phase 2: Configure COB Sequencing
Define the exact end-of-day behavior for loans.
```http
// 1. Check currently active LOAN_COB steps
GET {{baseUrl}}/v1/jobs/LOAN_COB/steps
Authorization: Basic {{auth}}

// 2. Re-configure the LOAN_COB pipeline
PUT {{baseUrl}}/v1/jobs/LOAN_COB/steps
Authorization: Basic {{auth}}
Content-Type: application/json

{
  "businessSteps": [
    {
      "stepName": "UpdateLoanArrearsAgingBusinessStep",
      "order": 1
    },
    {
      "stepName": "ApplyChargeToOverdueLoansBusinessStep",
      "order": 2
    },
    {
      "stepName": "AddPeriodicAccrualEntriesBusinessStep",
      "order": 3
    }
  ]
}
```

### Phase 3: Triggering Execution
Force a task to run immediately for testing or recovery.
```http
// 1. Manually execute a specific standalone Job (find ID via GET /v1/jobs)
POST {{baseUrl}}/v1/jobs/{jobId}?command=executeJob
Authorization: Basic {{auth}}

// 2. Force Catch-Up COB across all accounts
POST {{baseUrl}}/v1/loans/catch-up
Authorization: Basic {{auth}}
```

### Phase 4: Observability and Monitoring
Track the outcome of background processes.
```http
// 1. View execution history for a specific Job
GET {{baseUrl}}/v1/jobs/{jobId}/runhistory?limit=10&offset=0
Authorization: Basic {{auth}}

// 2. Check if the Catch-Up process is still running
GET {{baseUrl}}/v1/loans/is-catch-up-running
Authorization: Basic {{auth}}
```

---

## 5. Important Distinctions & Common Pitfalls

### What is tracked in the `job` table?
Fineract draws a strict line between **Scheduled/Batch Jobs** and **Real-time Asynchronous Tasks**. 
Not every asynchronous process has an entry in the `job` table. 

- **IN the `job` table:** Any process managed by the Quartz Scheduler or Spring Batch. These are heavy, recurring, or bulk-processing tasks (e.g., COB, Standing Instructions, Accruals, Report mailing). They can be managed via the `/v1/jobs` API.
- **NOT IN the `job` table:** Fineract uses standard Java thread pools, Spring `@Async`, and external message brokers (ActiveMQ/Kafka) for immediate, non-blocking tasks. Examples include:
  - **Business Event Dispatching:** Firing webhooks when a client is created.
  - **Remote Partitioning Workers:** Worker nodes processing chunks of a batch job via Kafka/JMS.
  - **Inline COB Executors:** Synchronous, on-demand execution of COB for a single lagging account.

### Standing Instructions vs. COB
A common misconception is that "Execute Standing Instruction" is part of the COB pipeline. **It is not.**

- **COB (`LOAN_COB`, `SAVINGS_COB`)** is built to handle complex, sequential end-of-day math for individual accounts. It locks individual accounts to guarantee strict execution order (interest, then arrears, then penalties).
- **Execute Standing Instruction** is a **standalone Scheduled Batch Job** (a standard Spring Batch `Tasklet`). Standing Instructions are account transfers (e.g., transferring $50 from Savings to Loan). Because they touch multiple accounts simultaneously, locking them inside a single account's COB pipeline would cause database deadlocks.

**Best Practice:** Configure your COB to run first (to settle all interest and penalties for the day), and schedule the "Execute Standing Instruction" job to run *afterward* so it executes transfers based on the newly updated balances.

---

## 6. Advanced Operations & Error Handling

### 1. Monitoring & Error Reporting
Beyond the basic API endpoints, how do you monitor jobs in production?
- **The `runhistory` API**: `GET /v1/jobs/{jobId}/runhistory` returns `SUCCESS` or `FAILED`, start/end times, and the error message if it failed. Fineract stores this in the `scheduled_job_run_history` database table.
- **Spring Batch Meta-Tables**: Because Fineract uses Spring Batch under the hood, advanced administrators monitor the `batch_job_execution` and `batch_step_execution` database tables directly. These tables contain granular execution details, including precisely which step failed and the full exception stack trace.
- **Log Aggregation (ELK/Datadog)**: Failed batch jobs log errors at the `ERROR` level. In a production environment, you should stream Fineract's logs to an aggregator (like Datadog or ELK) and set up alerts to ping a Slack channel or PagerDuty if a job throws an exception.

### 2. What Happens if a Job Fails?
- **COB Jobs (`LOAN_COB`)**: If a COB step fails for a *specific account* (e.g., due to a bizarre data corruption on that one account), Fineract does **not** crash the entire COB. It uses "fault tolerance." The specific account is marked as failed, the error is logged, and COB continues processing the remaining thousands of accounts. The failed account will attempt to process again during the next Catch-Up or Inline COB.
- **Catch-Up COB**: If Catch-Up COB completely fails, it halts. Once you resolve the underlying issue, you can simply trigger Catch-Up again, and it will safely resume from the oldest unprocessed date.
- **Standard Jobs**: If a standard job fails halfway, it is marked as `FAILED`. Fineract jobs are generally designed to be *idempotent* (safe to retry), so it will attempt to process the remaining data on its next scheduled run.

### 3. Pre-COB vs. Post-COB Jobs
Fineract does not hardcode jobs as "Pre-COB" or "Post-COB". Everything is simply a "Job". However, standard banking operations dictate a logical grouping:
- **Pre-COB**: Jobs that prepare data before EOD calculations. Example: `Update Non Performing Assets` (ensuring classifications are correct before interest runs).
- **COB**: `LOAN_COB`, `SAVINGS_COB`, `WORKING_CAPITAL_LOAN_COB`.
- **Post-COB**: Jobs that depend on the final EOD balances. Examples: `Execute Standing Instructions` (transfers based on new balances) and `Send Messages to SMS Gateway` (sending "Payment Due" texts only *after* penalties are applied).

### 4. How is the Sequence of Jobs Maintained?
This is a critical architectural point:
- **Within COB**: The sequence of calculations (Arrears -> Charges -> Accruals) is strictly maintained by the **Business Steps Framework**.
- **Between Different Jobs**: Fineract's internal Quartz Scheduler is **Time-Based (Cron), not Dependency-Based**. It has no concept of "Run Job B only after Job A finishes". If you rely on the internal scheduler, you must space them out by time (e.g., COB at 12:00 AM, Standing Instructions at 02:00 AM).
- **The Enterprise Solution (Airflow / Control-M)**: Relying on time gaps is dangerous (what if COB takes 3 hours on a heavy day?). Real-world Fineract deployments **disable the internal scheduler** completely. Instead, they use an external orchestrator like Apache Airflow. Airflow uses the API to trigger COB, continuously polls the `runhistory` endpoint until it returns `SUCCESS`, and only *then* triggers the Post-COB jobs like Standing Instructions.
