# Mastering Apache Fineract Client Module

The Client module in Apache Fineract is the foundational entity around which all core banking operations revolve. A client must exist before any portfolio products (savings, loans, shares) can be attached, and their lifecycle dictates what operations are permitted across the core banking system.

This guide provides an extensive overview of the Client Module, exploring the state machine, related sub-entities (KYC, Identifiers, Addresses), transaction orchestration, and API usage to help you model your core banking workflows effectively.

---

## 1. Client Lifecycle Management

Clients in Fineract follow a strict state machine. Understanding these states is crucial, as business rules prevent certain actions depending on the client's current status (e.g., loans cannot be disbursed to an inactive client).

### The Client State Machine

**Database Context**: Managed in the `m_client` table primarily via the `status_enum` column. 
- `id` (Primary Key)
- `account_no` (Auto-generated unique string)
- `status_enum` (100 = Pending, 300 = Active, 600 = Closed, 700 = Rejected, 800 = Withdrawn)
- `activation_date` (Date transitioned to 300)
- `closedon_date` (Date transitioned to 600)

> [!NOTE]
> **Bankrupt, Deceased, or Blacklisted?**
> Fineract does not have a dedicated core status for "Bankrupt" or "Deceased". Instead, the client is transitioned to **Closed** (`status_enum = 600`). The exact reason (e.g., Deceased, Written Off, Bankrupt) is stored in the `closure_reason_cv_id` column, which links to the `m_code_value` table (representing the "ClientClosureReason" dropdown).

1. **Pending**: The initial state when a client profile is created. The client is recorded but cannot yet hold active accounts or transact.
2. **Active**: The approved state. The client is now a fully-fledged member of the financial institution and can open savings accounts, apply for loans, and transact.
3. **Closed**: The client has left the program or their relationship with the institution is terminated. A client can only be closed if all their linked accounts (loans, savings) are closed.
4. **Rejected**: The client's application for membership was denied.
5. **Withdrawn**: The client withdrew their application before activation.

### Managing Client States via API

State transitions are fundamentally tied to how the client is initially created, and subsequently managed via the `command` query parameter on the `POST /v1/clients/{clientId}` endpoint.

> [!IMPORTANT]
> A client command API call requires a JSON payload with specific tracking dates (e.g., `activationDate`, `closureDate`) and reasons (where applicable).

- **Create Client**: `POST /v1/clients` 
  - If the JSON payload contains `"active": true` and an `"activationDate"`, the profile is created and **instantly activated**. You do not need to fire a separate activate command.
  - If the payload contains `"active": false`, the profile is created in a **Pending** state and must be explicitly activated later.
- **Activate**: `POST /v1/clients/{clientId}?command=activate` (Requires `activationDate`. Used only if the client was created in a Pending state.)
- **Close**: `POST /v1/clients/{clientId}?command=close` (Requires `closureDate`, `closureReasonId`)
- **Reject**: `POST /v1/clients/{clientId}?command=reject` (Requires `rejectionDate`, `rejectionReasonId`)
- **Withdraw**: `POST /v1/clients/{clientId}?command=withdraw` (Requires `withdrawalDate`, `withdrawalReasonId`)
- **Reactivate**: `POST /v1/clients/{clientId}?command=reactivate` (Requires `reactivationDate`)
- **Undo Rejection**: `POST /v1/clients/{clientId}?command=undoRejection` (Requires `reopenedDate`)
- **Undo Withdrawal**: `POST /v1/clients/{clientId}?command=undoWithdrawal` (Requires `reopenedDate`)

---

## 2. Client Identity & KYC Features

To satisfy Know Your Customer (KYC) requirements, Fineract allows linking various distinct data structures to a core Client record. 

### Client Identifiers
**Database Context**: Managed in the `m_client_identifier` table.
- `client_id` (Foreign Key to `m_client.id`)
- `document_type_id` (Foreign Key to `m_code_value.id` representing Passport, National ID, etc.)
- `document_key` (The actual string/number of the ID)
- `status` (Active / Inactive)

Identifiers represent official documents (e.g., National ID, Passport, Driver's License) used to uniquely identify a client within a system or legally.

- **List Identifiers**: `GET /v1/clients/{clientId}/identifiers`
- **Add Identifier**: `POST /v1/clients/{clientId}/identifiers`
  ```json
  {
    "documentKey": "ID-847583-XYZ",
    "documentTypeId": 1,
    "description": "National Identity Card",
    "status": "Active"
  }
  ```

### Client Family Members
**Database Context**: Managed in the `m_family_members` table.
- `client_id` (Foreign Key to `m_client.id`)
- `first_name` & `last_name`
- `relationship_cv_id` (Foreign Key to `m_code_value.id` representing Spouse, Dependent, Guarantor)

Fineract has a dedicated API for capturing family structure, which is often used in microfinance for tracking guarantors, next of kin, or household economics.
- **Manage Family**: `POST /v1/clients/{clientId}/familymembers`

### Client Addresses
**Database Context**: Managed via a normalized two-table structure:
1. `m_address`: Stores the physical location details (`street`, `city`, `postal_code`, `country_id`).
2. `m_client_address`: The mapping table linking the address to the client (`client_id`, `address_id`, `address_type_id`, `is_active`).

This structure allows tracking multiple locations (Residential, Business, Permanent) over time.
- **Manage Addresses**: `POST /v1/client/{clientId}/addresses`

### Documents and Notes

> [!WARNING]
> The generic `/v1/{entityType}/{entityId}/images` endpoint has been **deprecated** in Fineract. Profile pictures and other image assets should now be uploaded and retrieved as standard documents using the `/documents` API.

#### Best Practice: Identifiers vs. Documents Workflow
A common point of confusion is knowing when to use Identifiers vs. Documents. 
- **Identifiers** store the structured *metadata* (e.g., the actual Passport Number, Issue Date, Document Type).
- **Documents** store the actual *binary file* (e.g., the scanned PDF or JPEG of the Passport).

To strictly link a physical scan to its metadata in the database, **do not** upload the ID scan directly to the client. Instead, upload it to the identifier entity:

1. **Create the Identifier:**
   First, register the passport metadata.
   `POST /v1/clients/{clientId}/identifiers`
   ```json
   {
     "documentTypeId": 12, 
     "documentKey": "PASSPORT-A1234567",
     "description": "Customer's primary travel passport",
     "status": "Active"
   }
   ```
   *(Assume the response returns `"resourceId": 99`. This is your `identifierId`.)*

2. **Upload the Scan:**
   Use the generic documents API, setting the `entityType` to `client_identifiers` and the `entityId` to the `identifierId` we just created (`99`).
   `POST /v1/client_identifiers/99/documents`
   
   This must be a **Multipart Form Data** request:
   - `name`: "Scanned Passport Copy"
   - `description`: "High-resolution JPEG scan of the passport data page"
   - `file`: *(Binary file payload, e.g., `passport_scan.jpg`)*

   Example cURL Request:
   ```bash
   curl -X POST "https://your-fineract-server.com/fineract-provider/api/v1/client_identifiers/99/documents" \
     -H "Fineract-Platform-TenantId: default" \
     -H "Authorization: Basic ..." \
     -F "name=Scanned Passport Copy" \
     -F "description=High-resolution JPEG scan of the passport data page" \
     -F "file=@/path/to/passport_scan.jpg"
   ```

#### Where Are Binary Files Stored?

By default, Fineract does **not** store the binary contents (e.g., the actual PDF or JPEG file) inside the relational database (like MySQL or PostgreSQL). Instead, the Fineract database's `m_document` table only acts as a metadata index, while the physical files are offloaded to a dedicated storage layer.

Depending on your server configuration, Fineract routes the binary files to one of two places:

1. **Local File System (Default)**
   Out of the box, Fineract uses a local disk storage strategy. It creates a folder hierarchy on the host server running the Spring Boot application, defaulting to a directory like `~/.fineract/` (or the path defined by the `fineract.content.filesystem.rootFolder` property in `application.properties`).
   *Example path:* `~/.fineract/documents/client_identifiers/99/passport_scan.jpg`

   > [!TIP]
   > **The Filesystem Toggle**: The entire local disk storage engine is controlled by the `fineract.content.filesystem.enabled` Spring Boot property (which defaults to `true`).

2. **Amazon S3 (For Production / Containers)**
   If Fineract is deployed in a clustered environment (like Kubernetes) where local disk storage is ephemeral, saving files locally causes data loss. Fineract natively supports AWS S3. If S3 is configured via `application.properties` (using `fineract.content.s3.enabled=true`), the Multipart form upload is streamed directly into an S3 bucket.

   > [!WARNING]
   > **Production Safeguards**: When using S3 in production, it is highly recommended to explicitly set `fineract.content.filesystem.enabled=false`. This acts as a strict safeguard, guaranteeing Fineract will *never* accidentally write sensitive KYC documents to the local server disk, forcing 100% of traffic to the encrypted AWS bucket.

**How Fineract tracks it:**
Whether using File System or S3, Fineract inserts a row into the `m_document` table containing the original filename, MIME type, the physical `location` (absolute disk path or S3 Object Key), and a **`storage_type_enum`** (`1` for File System, `2` for S3). When you download a document via `GET /v1/.../documents/{documentId}/attachment`, Fineract uses this metadata to fetch the stream from the correct backend.

- **Other Documents**: For general client files (e.g., signed contracts, profile pictures), upload them directly to the client via `POST /v1/clients/{clientId}/documents`.
### Client Notes (Lightweight CRM)
**Database Context**: Managed in the `m_note` table.
- `client_id` (Foreign Key to `m_client.id`)
- `note` (Text blob)
- `created_by_id` (User who wrote the note)
- `created_date` (Timestamp)

The **Notes API** (`POST /v1/clients/{clientId}/notes`) functions as a built-in CRM, allowing staff to attach qualitative, chronological text records to a client's profile. Fineract automatically tracks **who** created the note and **when**, creating a valuable audit trail.

**Common Use Cases:**
1. **Call Center / Support Logs**: `"Customer called regarding failed deposit. Advised to check back in 2 hours."`
2. **Relationship Manager Field Updates**: `"Visited the client's business. Inventory is fully stocked. Recommend approving loan increase."`
3. **Compliance Audit Trails**: `"Client flagged in PEP tool, but DOB differs. Cleared for onboarding after secondary review."`
4. **Collections / Debt Recovery**: `"Called client at 10:00 AM regarding overdue payment. Promised to pay by Friday."`

> [!TIP]
> **Notes vs. Data Tables**: Use Data Tables for structured, strict data (like a dropdown of Risk Ratings). Use **Notes** for unstructured, free-flowing text that tells the "story" of the client's relationship with your institution!

---

## 3. Client Portfolio & Accounts

The client portfolio represents the financial footprint of a client. It aggregates all active, pending, and closed accounts across the different product types (Loans, Savings, Shares).

> [!TIP]
> Use the **Accounts API** to render an omnichannel customer dashboard view in a single API call.

- **Retrieve Portfolio**: `GET /v1/clients/{clientId}/accounts`
  Returns a summary JSON separating `loanAccounts`, `savingsAccounts`, and `shareAccounts` for the given client.

### Default Savings Account
**Database Context**: Managed directly in the `m_client` table via the `default_savings_account` column (which stores the ID of the specific `m_savings_account`).

When applying for loans or configuring Standing Instructions, you often need a primary funding or settlement account. Fineract allows you to peg a specific savings account as the client's default.

This is critical for workflows like:
- **Auto-Repayment / Auto-Sweep**: The system can automatically draw down funds from this default savings account to settle loan installments on their due date.
- **Pay In / Pay Out Account**: When a loan is disbursed, the system can automatically deposit the principal into this default savings account.

- **Update Default Savings**: `POST /v1/clients/{clientId}?command=updateSavingsAccount`
  ```json
  {
    "savingsAccountId": 45
  }
  ```

---

## 4. Branch Transfers & Hierarchy

Fineract provides a robust mechanism to move clients across different Branches/Offices, re-assigning their financial tracking and general ledger implications.

The transfer process can be executed synchronously or through a proposal-acceptance workflow:
1. **Propose Transfer**: `POST /v1/clients/{clientId}?command=proposeTransfer` (Sent from source branch)
2. **Accept Transfer**: `POST /v1/clients/{clientId}?command=acceptTransfer` (Approved by destination branch)
3. **Reject Transfer**: `POST /v1/clients/{clientId}?command=rejectTransfer`
4. **Withdraw Transfer**: `POST /v1/clients/{clientId}?command=withdrawTransfer`
5. **Direct Transfer (Admin)**: `POST /v1/clients/{clientId}?command=proposeAndAcceptTransfer`

Additionally, you can assign an institutional Staff Member (Loan Officer / Relationship Manager) directly to a client:
- **Assign Staff**: `POST /v1/clients/{clientId}?command=assignStaff`

---

## 5. Client Transactions & Charges

While transactions mostly occur at the Account level (e.g., a Savings Deposit), some financial movements happen directly at the Client level, primarily related to generic client fees (like a membership join fee or an annual maintenance charge that isn't tied to a specific product).

### Client Charges
**Database Context**: Managed in the `m_client_charge` table.
- `client_id` (Foreign Key to `m_client.id`)
- `charge_id` (Foreign Key to `m_charge.id` defining the fee rules)
- `amount_outstanding` (Remaining balance to be paid)
- `is_paid` (Boolean flag)
- **Apply a Charge**: `POST /v1/clients/{clientId}/charges`
- **Pay a Charge**: `POST /v1/clients/{clientId}/charges/{chargeId}?command=paycharge`
- **Waive a Charge**: `POST /v1/clients/{clientId}/charges/{chargeId}?command=waive`

### Client Transactions
Track payments made against client-level charges.
- **Retrieve Transactions**: `GET /v1/clients/{clientId}/transactions`
- **Undo Transaction**: `POST /v1/clients/{clientId}/transactions/{transactionId}?command=undo`

---

## 6. External IDs 

For seamless integration with third-party CRM systems (like Salesforce, HubSpot, or a custom backend), Fineract supports a dual-identifier strategy. 
Every Client API endpoint supports routing by the internal Database `clientId` or an `externalId` (UUID or string) mapped to your external architecture.

- **Internal ID routing**: `GET /v1/clients/15`
- **External ID routing**: `GET /v1/clients/external-id/cust-ab88-xyz9`

> [!CAUTION]
> Always enforce uniqueness for `externalId` when integrating external CRMs to prevent API lookup collisions during automated syncs.

---

## 7. Deep Dive: The Concept of Data Tables

Out-of-the-box, Fineract's `m_client` table only has columns for standard data (Name, Date of Birth, Gender, Mobile Number). However, every financial institution has unique data requirements (e.g., Tax ID, Religion, Risk Rating, PEP Status, Alternative Phone Numbers).

Instead of altering the core Fineract source code and modifying `m_client` (which breaks upgrades), Fineract provides **Data Tables**.

### How Data Tables Work
A Data Table is a custom MySQL table that Fineract dynamically registers via the `/v1/datatables` API. 
1. **Creation**: You define the columns (String, Number, Date, Boolean) via API. Fineract creates the actual physical MySQL table behind the scenes (e.g., `client_pep_details`).
2. **Registration**: Fineract links this new table to a core entity (like `m_client`) via the `x_registered_table` internal mapping.
3. **Cardinality**: 
   - **One-to-One**: e.g., A client has exactly one PEP status. The `client_id` is the primary key.
   - **One-to-Many**: e.g., A client can have multiple previous employers. The `client_id` is a foreign key, and Fineract auto-generated a separate primary key for the datatable.

---

## 8. Practical Walkthrough: Production Client Setup

Let's orchestrate a full client onboarding flow combining all the modules discussed above.

### Step 8.1: Setup (Run Once per Environment)

Before onboarding customers, you must configure your Data Tables and KYC Code Values.

**1. Create a "PEP Status" Data Table**
We will create a One-to-One data table linked to `m_client` to track if the client is a Politically Exposed Person.

```http
POST /v1/datatables/client_pep_details
```
```json
{
  "apptableName": "m_client",
  "multiRow": false,
  "columns": [
    {
      "name": "is_pep",
      "type": "Boolean",
      "mandatory": true
    },
    {
      "name": "pep_risk_rating",
      "type": "String",
      "length": 50,
      "mandatory": false
    }
  ]
}
```

**2. Setup Identifier Codes**
Identity document types (Passport, National ID) must exist in the `m_code_value` table linked to the `Customer Identifier` code.

```http
POST /v1/codes/1/codevalues
```
```json
{
  "name": "National ID Card",
  "description": "Government issued National ID",
  "isActive": true
}
```
*(Assume this returns `codeValueId`: 15)*

---

### Step 8.2: Execute (Per Customer Onboarding)

Now, let's create our production client via an orchestration layer.

**1. Create the Base Client Profile**

```http
POST /v1/clients
```
```json
{
  "officeId": 1,
  "firstname": "John",
  "lastname": "Doe",
  "mobileNo": "+254700000000",
  "active": true,
  "activationDate": "07 May 2026",
  "dateFormat": "dd MMMM yyyy",
  "locale": "en",
  "legalFormId": 1
}
```
*(Response returns `clientId`: 42)*

**2. Attach Identity Documents (KYC)**
Link the National ID we set up earlier to the client. This inserts into the `m_client_identifier` table.

```http
POST /v1/clients/42/identifiers
```
```json
{
  "documentTypeId": 15,
  "documentKey": "ID-987654321",
  "description": "Verified via E-Citizen API",
  "status": "Active"
}
```

> [!TIP]
> After attaching the identifier, you would immediately use the **`/v1/clients/42/documents`** API to upload the scanned JPEG/PDF of the actual National ID card.

**3. Attach PEP Status via Data Tables**
We insert data into our custom `client_pep_details` table. 

```http
POST /v1/datatables/client_pep_details/42
```
```json
{
  "is_pep": true,
  "pep_risk_rating": "HIGH",
  "dateFormat": "dd MMMM yyyy",
  "locale": "en"
}
```

**4. Add Family Members**
Track next of kin or dependents. This inserts into `m_family_members`.

```http
POST /v1/clients/42/familymembers
```
```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "relationshipId": 25, 
  "genderId": 13,
  "mobileNumber": "+254711111111"
}
```

**5. Apply Activation Fee (Client Charge)**
If your institution charges a one-off membership fee, apply it here. This inserts into `m_client_charge`.

```http
POST /v1/clients/42/charges
```
```json
{
  "chargeId": 5, 
  "amount": 1000.00,
  "locale": "en"
}
```
*Note: A client charge is an obligation. The client must either pay it (`?command=paycharge`) or it will be auto-recovered if mapped to a savings account.*

---

### Step 8.3: Verify & Observe

To fetch the comprehensive 360-degree view of the customer, your frontend or orchestration layer will need to make parallel queries:

1. **Get Base Details**: `GET /v1/clients/42`
2. **Get Identifiers**: `GET /v1/clients/42/identifiers`
3. **Get Family**: `GET /v1/clients/42/familymembers`
4. **Get PEP Data**: `GET /v1/datatables/client_pep_details/42`
5. **Get Outstanding Charges**: `GET /v1/clients/42/charges?pendingPayment=true`

By utilizing Data Tables alongside Fineract's native sub-modules (Identifiers, Family, Charges), you achieve a fully compliant, production-ready KYC engine without writing custom backend Java code.

---

---

## 9. Background Jobs & Scheduled Tasks

While Fineract relies heavily on the Close of Business (COB) batch jobs for Loans and Savings (e.g., calculating interest, adding dormancy charges), the **Client Module** is primarily real-time and transactional. State transitions (activation, closure) occur instantly via API rather than through async overnight batch processing.

However, there is a background job relevant to group methodologies:
- **Update Client Meetings**: This scheduled job synchronizes center and group meeting schedules down to the individual client level, which is critical for Group/JLG (Joint Liability Group) lending models.

---

## 10. Client Relations, Groups & Centers (JLG)

Fineract provides flexible ways to link clients to family members, other clients, and larger community structures for Joint Liability Group (JLG) lending.

### 10.1 Family Members vs. Client-to-Client Links
- **Family Members**: Fineract treats family members (spouses, dependents) purely as metadata attached to a primary client. They are stored in `m_family_members` as raw text (First Name, Last Name) and do not have their own unique Client IDs or accounts.
- **Client-to-Client**: At the core CRM level, you cannot directly link "Client A" to "Client B" as business partners natively. However, they are linked transactionally at the account level (e.g., Client B acts as a **Guarantor** for Client A's loan, stored in `m_guarantor`).

### 10.2 The Group & Center Hierarchy
For community-based savings and JLG lending, Fineract uses a strict hierarchy managed by the `m_group` and `m_group_client` tables:
1. **Center**: A large geographical gathering (e.g., a village meeting). (`m_group` where `level_id = 1`)
2. **Group**: A small pod of 5-10 individuals who guarantee each other's loans. (`m_group` where `level_id = 2`)
3. **Client**: The individual borrower. (`m_client` mapped via `m_group_client`)

> [!NOTE]
> **Joint Liability Group (JLG) Rules (Client Perspective)**
> When clients are bundled into a Group, they share financial liability. From the client's perspective, if *one* member of their Group defaults on a loan repayment, the JLG rules can automatically freeze the savings accounts or block future loan disbursements for *all other members* in that Group until the arrears are cleared.

### 10.3 Practical Walkthrough: Creating a Center and Group

**Step 1: Setup (Create a Center)**
Create the parent community gathering.
`POST /v1/centers`
```json
{
  "name": "Nairobi Central Meeting",
  "officeId": 1,
  "active": true,
  "activationDate": "01 May 2026",
  "dateFormat": "dd MMMM yyyy",
  "locale": "en"
}
```
*(Assume this returns `groupId`: 100)*

**Step 2: Execute (Create a Group under the Center)**
Create the small lending pod and attach it to the Center.
`POST /v1/groups`
```json
{
  "name": "Pod Alpha",
  "officeId": 1,
  "centerId": 100,
  "active": true,
  "activationDate": "01 May 2026",
  "dateFormat": "dd MMMM yyyy",
  "locale": "en"
}
```
*(Assume this returns `groupId`: 101)*

**Step 3: Associate Clients to the Group**
Link individual `m_client` profiles to "Pod Alpha".
`POST /v1/groups/101?command=associateClients`
```json
{
  "clientMembers": [42, 88, 105] 
}
```

**Step 4: Verify & Observe**
Check the group to see its active members.
`GET /v1/groups/101?associations=clientMembers`
You will see Clients 42, 88, and 105 successfully mapped. They are now financially linked under JLG rules!

---

## 11. Deep Dive: Guarantors & On-Hold Funds

While a standard Client profile does not natively link to other Client profiles for CRM purposes, Fineract deeply links clients together when risk and liability are involved. The most prominent example is the **Guarantor** framework.

### 11.1 Types of Guarantors
Fineract supports four distinct types of guarantors, defined by the `guarantor_type_enum` in the `m_guarantor` table:

1. **Existing Customer (Type 1)**: The guarantor is another active `m_client` in your system.
2. **Staff Member (Type 2)**: The guarantor is an employee/loan officer (`m_staff`).
3. **External (Type 3)**: The guarantor is a third party not registered in your system. Fineract simply stores their raw text metadata (First Name, Last Name, Address, DOB, Phone) in the `m_guarantor` table.
4. **Group (Type 4)**: An entire `m_group` acts as the guarantor.

### 11.2 Guarantor Funding & Fund Locking
The most powerful feature of the Guarantor module is **Guarantor Funding** (`m_guarantor_funding_details` table). 

If Client B acts as a Guarantor for Client A, and Client B has a Fineract Savings Account, you can configure the guarantee to explicitly **lock funds**. Fineract will place an "On Hold" block on Client B's savings account for the guaranteed amount. Client B will not be able to withdraw those funds until Client A successfully repays the loan!

### 11.3 Practical Walkthrough: Setting up a Funded Guarantor

In this scenario, Client A (Borrower) applies for a loan, and Client B (Guarantor) pledges 5,000 KES from their active savings account to back it.

**Step 1: Setup (Identify the Entities)**
- Borrower Loan ID: `150` (Pending Approval)
- Guarantor Client ID: `45`
- Guarantor Savings Account ID: `99` (Must have at least 5,000 KES available balance)

**Step 2: Execute (Create the Guarantor)**
We attach the Guarantor to the *Borrower's Loan*, passing the funding details.
`POST /v1/loans/150/guarantors`
```json
{
  "guarantorTypeId": 1,
  "entityId": 45,
  "clientRelationshipTypeId": 1,
  "savingsId": 99,
  "amount": 5000.00
}
```

**Step 3: Verify (Check the Loan)**
Verify that the guarantor is successfully attached to the loan.
`GET /v1/loans/150/guarantors`
*(You should see Client 45 listed as an active guarantor for 5,000 KES).*

**Step 4: Observe (Check the Guarantor's Savings Account)**
This is the most critical observation. Fetch the Guarantor's savings account:
`GET /v1/savingsaccounts/99`
In the JSON response, under the `summary` object, you will see:
```json
"summary": {
  "accountBalance": 15000.00,
  "onHoldFunds": 5000.00,
  "availableBalance": 10000.00
}
```
The Fineract engine has automatically frozen 5,000 KES! As Client A repays the loan, the `onHoldFunds` will proportionally decrease (depending on your product configuration), eventually returning to zero.

---

## Summary

The Client Module serves as the hub of your Fineract core banking instance. When building your white-label systems:
1. Implement the **Lifecycle** rules to govern when customers can perform operations.
2. Rely on the **Portfolio Accounts API** for dashboard renders.
3. Utilize the **External ID** paths to seamlessly connect your Identity Provider (IdP) or CRM with Fineract's transactional engine.
4. Leverage **Data Tables** to extend client attributes dynamically for KYC/AML requirements without altering core source code.
