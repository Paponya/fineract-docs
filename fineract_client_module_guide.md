# Mastering Apache Fineract Client Module

The Client module in Apache Fineract is the foundational entity around which all core banking operations revolve. A client must exist before any portfolio products (savings, loans, shares) can be attached, and their lifecycle dictates what operations are permitted across the core banking system.

This guide provides an extensive overview of the Client Module, exploring the state machine, related sub-entities (KYC, Identifiers, Addresses), transaction orchestration, and API usage to help you model your core banking workflows effectively.

---

## 1. Client Lifecycle Management

Clients in Fineract follow a strict state machine. Understanding these states is crucial, as business rules prevent certain actions depending on the client's current status (e.g., loans cannot be disbursed to an inactive client).

### The Client State Machine

1. **Pending**: The initial state when a client profile is created. The client is recorded but cannot yet hold active accounts or transact.
2. **Active**: The approved state. The client is now a fully-fledged member of the financial institution and can open savings accounts, apply for loans, and transact.
3. **Closed**: The client has left the program or their relationship with the institution is terminated. A client can only be closed if all their linked accounts (loans, savings) are closed.
4. **Rejected**: The client's application for membership was denied.
5. **Withdrawn**: The client withdrew their application before activation.

### Managing Client States via API

Fineract exposes state transition operations via the `command` query parameter on the `POST /v1/clients/{clientId}` endpoint.

> [!IMPORTANT]
> A client command API call requires a JSON payload with specific tracking dates (e.g., `activationDate`, `closureDate`) and reasons (where applicable).

- **Activate**: `POST /v1/clients/{clientId}?command=activate` (Requires `activationDate`)
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
Fineract has a dedicated API for capturing family structure, which is often used in microfinance for tracking guarantors, next of kin, or household economics.
- **Manage Family**: `POST /v1/clients/{clientId}/familymembers`

### Client Addresses
Addresses support tracking Multiple locations (Residential, Business, Permanent) over time.
- **Manage Addresses**: `POST /v1/client/{clientId}/addresses`

### Documents and Notes

> [!WARNING]
> The generic `/v1/{entityType}/{entityId}/images` endpoint has been **deprecated** in Fineract. Profile pictures and other image assets should now be uploaded and retrieved as standard documents using the `/documents` API.

- **Documents (including Images)**: Upload scanned PDFs, ID copies, and profile pictures (JPEGs/PNGs) linked to the client via `POST /v1/clients/{clientId}/documents`.
- **Notes**: Append chronological comments or teller interactions (`POST /v1/clients/{clientId}/notes`).

---

## 3. Client Portfolio & Accounts

The client portfolio represents the financial footprint of a client. It aggregates all active, pending, and closed accounts across the different product types (Loans, Savings, Shares).

> [!TIP]
> Use the **Accounts API** to render an omnichannel customer dashboard view in a single API call.

- **Retrieve Portfolio**: `GET /v1/clients/{clientId}/accounts`
  Returns a summary JSON separating `loanAccounts`, `savingsAccounts`, and `shareAccounts` for the given client.

### Default Savings Account
When applying for loans or orchestrating automated clearing, you often need a primary funding/settlement account. Fineract allows you to peg a specific savings account as the client's default.
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

## Summary

The Client Module serves as the hub of your Fineract core banking instance. When building your white-label systems:
1. Implement the **Lifecycle** rules to govern when customers can perform operations.
2. Rely on the **Portfolio Accounts API** for dashboard renders.
3. Utilize the **External ID** paths to seamlessly connect your Identity Provider (IdP) or CRM with Fineract's transactional engine.
