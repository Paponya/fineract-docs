# Fineract Savings Charges & GL Account Mapping — Complete Guide

## Table of Contents
1. [Concepts & Architecture](#1-concepts--architecture)
2. [Journal Entry Type Convention](#2-journal-entry-type-convention)
3. [GL Account Setup](#3-gl-account-setup)
4. [Setting Up Charges](#4-setting-up-charges)
5. [Linking Charges Before Savings Product Creation](#5-linking-charges-before-savings-product-creation)
6. [Linking Charges After Savings Product Creation](#6-linking-charges-after-savings-product-creation)
7. [Payment-Type-Specific Charges](#7-payment-type-specific-charges)
8. [Excise Tax via Dual Charges](#8-excise-tax-via-dual-charges)
9. [GL Account Mapping for Charges](#9-gl-account-mapping-for-charges)
10. [The `acc_product_mapping` Table — Deep Dive](#10-the-acc_product_mapping-table--deep-dive)
11. [How Journal Entries Are Generated](#11-how-journal-entries-are-generated)
12. [Verification Queries](#12-verification-queries)
13. [Known Bugs & Fixes](#13-known-bugs--fixes)

---

## 1. Concepts & Architecture

### Entity Hierarchy
```
m_charge                        ← Master charge definition (template)
  └── m_savings_product_charge  ← Links charge to a savings product (template)
        └── m_savings_account_charge ← Links charge to a specific savings account (live)
```

### Key Tables
| Table | Purpose |
|---|---|
| `m_charge` | Defines the charge: name, amount, type, GL account, payment type |
| `m_savings_product_charge` | Associates a charge with a savings product |
| `m_savings_account_charge` | Per-account charge instance; this is what gets applied on transactions |
| `acc_gl_account` | Chart of accounts |
| `acc_product_mapping` | Maps savings product → GL accounts by account type |
| `acc_gl_journal_entry` | Actual double-entry journal entries per transaction |

### Charge Lifecycle
```
Create Charge (m_charge)
    → Link to Savings Product (m_savings_product_charge)
        → Account is created/charged: entries copied to m_savings_account_charge
            → Transaction occurs: payWithdrawalFee() fires
                → Journal entries created in acc_gl_journal_entry
```

### Important: Charges Must Exist in `m_savings_account_charge`
If `m_savings_account_charge` has no record for a charge, the charge will **never fire** on transactions — even if it's in `m_savings_product_charge`. When a savings account is opened, charges from the product template are **copied** to `m_savings_account_charge`. For accounts created before a charge was added to the product, you must manually add the charge via API.

---

## 2. Journal Entry Type Convention

> [!CAUTION]
> This is the most common source of confusion. **Fineract uses `CREDIT=1`, `DEBIT=2`** — the opposite of what you might assume.

### Enum Definition (`JournalEntryType.java`)
```java
CREDIT(1, "journalEntryType.credit")   // type_enum = 1
DEBIT(2,  "journalEntryType.debit")    // type_enum = 2
```

### Correct SQL CASE Statement
```sql
-- ✅ CORRECT
CASE WHEN je.type_enum = 2 THEN 'DEBIT' ELSE 'CREDIT' END AS entry_type

-- ❌ WRONG (common mistake)
CASE WHEN je.type_enum = 1 THEN 'DEBIT' ELSE 'CREDIT' END AS entry_type
```

### Savings Accounting Convention (Bank's Perspective)
| Account | Classification | Fee Charge Effect | Entry |
|---|---|---|---|
| Voluntary Savings | LIABILITY (2) | Balance decreases → liability decreases | **DEBIT** |
| Fees and Charges | INCOME (4) | Revenue earned | **CREDIT** |
| Excise Duty Payable | LIABILITY (2) | Tax collected for govt | **CREDIT** |

So for a KES 60 fee + KES 12 excise tax on a withdrawal:
```
DR  Voluntary Savings (10101)     KES 72   ← customer balance reduced
    CR  Fees and Charges (30101)  KES 60   ← fee income
    CR  Excise Duty Payable (10401) KES 12 ← tax liability
```

---

## 3. GL Account Setup

### GL Account Types (`classification_enum`)
| Value | Type | Used For |
|---|---|---|
| 1 | ASSET | Cash, Investments |
| 2 | LIABILITY | Savings deposits, Payables |
| 3 | EQUITY | Capital |
| 4 | INCOME | Fee income, Interest income |
| 5 | EXPENSE | Operating costs |

### GL Account Usage (`account_usage`)
| Value | Meaning |
|---|---|
| 1 | **Detail** — can receive journal entries |
| 2 | **Header** — grouping only, CANNOT receive journal entries |

> [!IMPORTANT]
> You can only assign a `glAccountId` to a charge if it points to a **Detail account** (`account_usage = 1`). Header accounts will cause errors or silent failures.

### Check Existing GL Accounts
```sql
-- All detail income accounts (for fee charges)
SELECT id, name, gl_code, parent_id
FROM acc_gl_account
WHERE account_usage = 1 AND classification_enum = 4
ORDER BY gl_code;

-- All detail liability accounts (for excise/tax charges)
SELECT id, name, gl_code, parent_id
FROM acc_gl_account
WHERE account_usage = 1 AND classification_enum = 2
ORDER BY gl_code;
```

### Create a New Detail GL Account
```http
POST /fineract-provider/api/v1/glaccounts
```
```json
{
    "name": "Excise Duty Payable",
    "glCode": "10401",
    "type": 2,
    "usage": 1,
    "parentId": 4,
    "manualEntriesAllowed": true,
    "description": "Excise tax on M-Pesa fees - KRA payable"
}
```

> [!WARNING]
> If you get `duplicate key value violates unique constraint "acc_gl_account_pkey"`, the PostgreSQL sequence is out of sync. Fix it with:
> ```sql
> SELECT setval(
>     pg_get_serial_sequence('acc_gl_account', 'id'),
>     (SELECT MAX(id) FROM acc_gl_account)
> );
> ```

---

## 4. Setting Up Charges

### Charge Definition Fields
| Field | Description | Values |
|---|---|---|
| `chargeAppliesTo` | What product type | 1=Loan, 2=Savings, 4=Client |
| `chargeTimeType` | When charge fires | 1=Disbursement, 5=Withdrawal, 6=AnnualFee... |
| `chargeCalculationType` | How amount is calculated | 1=Flat, 2=% of Amount |
| `chargePaymentMode` | How collected | 0=Regular, 1=Account Transfer |
| `enablePaymentType` | Restrict to payment type | true/false |
| `paymentTypeId` | Specific payment type | ID from `m_payment_type` |
| `glAccountId` | Income/liability GL | ID from `acc_gl_account` (detail only) |

### Create a Withdrawal Fee Charge
```http
POST /fineract-provider/api/v1/charges
```
```json
{
    "name": "M-Pesa Outgoing Charge",
    "currencyCode": "KES",
    "amount": 60,
    "chargeTimeType": 5,
    "chargeAppliesTo": 2,
    "chargeCalculationType": 1,
    "chargePaymentMode": 0,
    "enablePaymentType": true,
    "paymentTypeId": 5,
    "glAccountId": 36,
    "active": true,
    "locale": "en"
}
```

> [!NOTE]
> `chargeTimeType: 5` = Withdrawal Fee. This makes the charge fire automatically when `payWithdrawalFee()` is called during a withdrawal transaction matching the payment type.

### Verify Charge Was Created
```sql
SELECT 
    c.id, c.name, c.amount,
    c.charge_time_enum, c.charge_applies_to_enum,
    c.is_payment_type, c.payment_type_id,
    pt.value AS payment_type_name,
    g.id AS gl_id, g.name AS gl_name, g.gl_code
FROM m_charge c
LEFT JOIN m_payment_type pt ON pt.id = c.payment_type_id
LEFT JOIN acc_gl_account g ON g.id = c.income_or_liability_account_id
WHERE c.is_deleted = false
ORDER BY c.id;
```

---

## 5. Linking Charges Before Savings Product Creation

This is the recommended approach — charges are baked into the product template and automatically assigned to all new accounts.

### Step 1: Create the Charge (see Section 4)

### Step 2: Include Charge When Creating Savings Product
```http
POST /fineract-provider/api/v1/savingsproducts
```
```json
{
    "name": "Voluntary Savings",
    "shortName": "VS01",
    "currencyCode": "KES",
    "digitsAfterDecimal": 2,
    "nominalAnnualInterestRate": 0,
    "interestCompoundingPeriodType": 1,
    "interestPostingPeriodType": 4,
    "interestCalculationType": 1,
    "interestCalculationDaysInYearType": 365,
    "accountingRule": 2,
    "charges": [
        { "id": 1 },
        { "id": 2 }
    ],
    "feeToIncomeAccountMappings": [
        { "chargeId": 1, "incomeAccountId": 36 },
        { "chargeId": 2, "incomeAccountId": 56 }
    ],
    "locale": "en"
}
```

### Step 3: Verify Product-Charge Link
```sql
SELECT spc.savings_product_id, spc.charge_id, c.name
FROM m_savings_product_charge spc
JOIN m_charge c ON c.id = spc.charge_id
WHERE spc.savings_product_id = 4;
```

---

## 6. Linking Charges After Savings Product Creation

### A) Add Charge to the Product (future accounts)
```http
PUT /fineract-provider/api/v1/savingsproducts/{productId}
```
```json
{
    "charges": [{ "id": 1 }, { "id": 2 }],
    "locale": "en"
}
```

### B) Add Charge to an Existing Account (existing accounts)

> [!IMPORTANT]
> Existing accounts do NOT automatically receive charges added to the product. You must add them manually per account.

```http
POST /fineract-provider/api/v1/savingsaccounts/{accountId}/charges
```
```json
{
    "chargeId": 1,
    "amount": 60,
    "locale": "en"
}
```

**Do NOT include** `chargeTimeType` or `chargeCalculationType` — these are read from `m_charge` and the API will reject them.

### C) Verify Charge Is on the Account
```sql
SELECT 
    sac.id, sac.savings_account_id, sac.charge_id,
    c.name, sac.amount, sac.is_active,
    sac.charge_time_enum
FROM m_savings_account_charge sac
JOIN m_charge c ON c.id = sac.charge_id
WHERE sac.savings_account_id = 3;
```

**If this query returns 0 rows**, the charge will NEVER fire on that account, regardless of product configuration.

---

## 7. Payment-Type-Specific Charges

Charges can be restricted to fire only when the transaction uses a specific payment method (e.g., M-Pesa, Bank Transfer).

### How It Works
In `SavingsAccount.payWithdrawalFee()`:
```java
// Simplified logic
for (SavingsAccountCharge charge : charges) {
    if (charge.isWithdrawalFee()) {
        if (!charge.isEnabledForPaymentType() 
            || charge.getPaymentType().equals(transactionPaymentType)) {
            // Apply this charge
        }
    }
}
```

The transaction's payment type is matched against `m_charge.payment_type_id`. If `is_payment_type = true` and payment types don't match, the charge is skipped silently.

### Verify Payment Type Mapping
```sql
SELECT 
    c.id, c.name, c.is_payment_type,
    pt.id AS pt_id, pt.value AS payment_type_name
FROM m_charge c
LEFT JOIN m_payment_type pt ON pt.id = c.payment_type_id
WHERE c.charge_applies_to_enum = 2;  -- savings charges only
```

---

## 8. Excise Tax via Dual Charges

Since Fineract's `taxGroup` on `m_charge` is **not wired into savings withdrawal processing**, the cleanest way to apply excise tax is a second charge of the same payment type.

### Setup

**Charge 1 — Base Fee:**
```json
{
    "name": "M-Pesa Outgoing Charge",
    "amount": 60,
    "chargeTimeType": 5,
    "chargeAppliesTo": 2,
    "chargeCalculationType": 1,
    "enablePaymentType": true,
    "paymentTypeId": 5,
    "glAccountId": 36,
    "active": true,
    "locale": "en"
}
```

**Charge 2 — Excise Tax:**
```json
{
    "name": "Excise Tax on M-Pesa Fee",
    "amount": 12,
    "chargeTimeType": 5,
    "chargeAppliesTo": 2,
    "chargeCalculationType": 1,
    "enablePaymentType": true,
    "paymentTypeId": 5,
    "glAccountId": 56,
    "active": true,
    "locale": "en"
}
```

Both charges share `paymentTypeId: 5`. Both fire together on every M-Pesa withdrawal.

### Add Both to Account
```http
POST /savingsaccounts/3/charges
{ "chargeId": 1, "amount": 60, "locale": "en" }

POST /savingsaccounts/3/charges
{ "chargeId": 2, "amount": 12, "locale": "en" }
```

### Resulting Journal Entries
```
DR  Voluntary Savings (10101)      KES 72   ← total charge deducted from balance
    CR  Fees and Charges (30101)   KES 60   ← base fee income
    CR  Excise Duty Payable (10401) KES 12  ← tax liability for KRA
```

---

## 9. GL Account Mapping for Charges

### How the Credit GL Is Resolved (Priority Order)

In `AccountingProcessorHelper.getLinkedGLAccountForSavingsCharges()`:

```
1. charge.getAccount()              ← m_charge.income_or_liability_account_id   (HIGHEST PRIORITY)
2. charge-specific product mapping  ← acc_product_mapping WHERE charge_id IS SET
3. product-level INCOME_FROM_FEES   ← acc_product_mapping (default fallback)
```

**Best practice:** Always set `glAccountId` on the charge definition so each charge posts to its own specific GL.

### Set GL Account on Charge at Creation
```json
POST /fineract-provider/api/v1/charges
{
    "glAccountId": 36,
    ...
}
```

### Update GL Account on Existing Charge
```http
PUT /fineract-provider/api/v1/charges/{chargeId}
```
```json
{
    "glAccountId": 36,
    "locale": "en"
}
```

> [!WARNING]
> **Known Bug (fixed in code):** When `income_or_liability_account_id` is currently NULL, the `PUT /charges/{id}` API does not update the field due to a bug in `Charge.update()` where `isChangeInLongParameterNamed(field, null)` returns false. **Workaround:** Update directly in the database:
> ```sql
> UPDATE m_charge SET income_or_liability_account_id = 36 WHERE id = 1;
> UPDATE m_charge SET income_or_liability_account_id = 56 WHERE id = 2;
> ```
> The code fix (in `Charge.java`) changes the check to `parameterExists()` + `Objects.equals()`.

### Verify GL Mapping on Charges
```sql
SELECT 
    c.id AS charge_id,
    c.name AS charge_name,
    c.amount,
    g.id AS gl_account_id,
    g.name AS gl_account_name,
    g.gl_code,
    g.classification_enum   -- 4=Income, 2=Liability
FROM m_charge c
LEFT JOIN acc_gl_account g ON g.id = c.income_or_liability_account_id
WHERE c.is_deleted = false
ORDER BY c.id;
```

---

## 10. The `acc_product_mapping` Table — Deep Dive

`acc_product_mapping` is the **central routing table** that tells Fineract's accounting engine which GL account to use for each financial activity type of each product. It is the bridge between a savings/loan product and the Chart of Accounts.

---

### 10.1 Table Schema

| Column | Type | Purpose |
|---|---|---|
| `id` | bigint PK | Auto-generated primary key |
| `gl_account_id` | bigint | FK → `acc_gl_account.id` — the target GL account |
| `product_id` | bigint | FK → the product (savings, loan, etc.) |
| `product_type` | smallint | Identifies the product category (see below) |
| `payment_type` | integer | FK → `m_payment_type.id` — **NULL for default mappings**, non-NULL for payment-channel overrides |
| `charge_id` | integer | FK → `m_charge.id` — **NULL for default mappings**, non-NULL for per-charge GL overrides |
| `financial_account_type` | smallint | Enum — identifies which accounting role this row configures (see Section 10.2) |
| `charge_off_reason_id` | integer | For loan charge-off categorisation (usually NULL for savings) |
| `capitalized_income_classification_id` | integer | For capitalised income (usually NULL for savings) |
| `buydown_fee_classification_id` | integer | For loan buydown fees (usually NULL for savings) |
| `write_off_reason_id` | bigint | For loan write-off categorisation (usually NULL for savings) |

---

### 10.2 `product_type` Values

| Value | Product Category |
|---|---|
| 1 | Loan |
| 2 | **Saving** ← savings products use this |
| 4 | Share |

---

### 10.3 `financial_account_type` Values

#### For Savings Products (`product_type = 2`)
Defined in `CashAccountsForSavings` enum:

| Value | Constant | GL Role | Typical Account |
|---|---|---|---|
| 1 | `SAVINGS_REFERENCE` | Cash/liquid asset representing the money physically held | Cash In Hand (ASSET) |
| 2 | `SAVINGS_CONTROL` | Liability tracking what the bank owes customers | Voluntary Savings (LIABILITY) |
| 3 | `INTEREST_ON_SAVINGS` | Expense for interest paid to depositors | Interest Paid To Depositors (EXPENSE) |
| 4 | `INCOME_FROM_FEES` | Default income account for fee charges (fallback) | Fees and Charges (INCOME) |
| 5 | `INCOME_FROM_PENALTIES` | Income account for penalty charges | Penalties (INCOME) |
| 6 | `TRANSFERS_SUSPENSE` | Holding account during account-to-account transfers | Transfers Suspense (LIABILITY) |
| 7 | `OVERDRAFT_PORTFOLIO_CONTROL` | Tracks overdraft balances | Current Account Overdrafts (ASSET) |
| 8 | `LOSSES_WRITTEN_OFF` | Expense for irrecoverable overdraft losses | Losses Written Off (EXPENSE) |
| 9 | `ESCHEAT_LIABILITY` | Unclaimed/dormant account balances due to the state | Escheat Liability (LIABILITY) |
| 10 | `TRANSFERS_SUSPENSE` (extended) | Payment-channel-specific transfers suspense | Liability Transfer (Temp) |
| 11 | `OVERDRAFT_PORTFOLIO_CONTROL` (extended) | Overdraft portfolio for specific payment channels | Current Account Overdrafts |
| 12 | `INCOME_FROM_INTEREST` | Interest income on overdrafts | Interest Received from Borrowers (INCOME) |
| 13 | `LOSSES_WRITTEN_OFF` (extended) | Write-off expense for specific scenarios | Losses Written Off (EXPENSE) |

#### For Loan Products (`product_type = 1`)
Defined in `CashAccountsForLoan` enum (partial):

| Value | Constant |
|---|---|
| 1 | `FUND_SOURCE` |
| 2 | `LOAN_PORTFOLIO` |
| 3 | `INTEREST_ON_LOANS` |
| 4 | `INCOME_FROM_FEES` |
| 5 | `INCOME_FROM_PENALTIES` |
| 6 | `LOSSES_WRITTEN_OFF` |
| 7 | `TRANSFERS_SUSPENSE` |
| 8 | `INTEREST_RECEIVABLE` |
| 9 | `FEES_RECEIVABLE` |
| 10 | `PENALTIES_RECEIVABLE` |

---

### 10.4 Actual Data — Savings Product 4 Mapping

From the verified query on your system (product_id=4, product_type=2):

| id | gl_account_id | GL Name | financial_account_type | Account Role | payment_type | charge_id |
|---|---|---|---|---|---|---|
| 8 | 32 | Cash In Hand (20301) | 1 | SAVINGS_REFERENCE (default) | NULL | NULL |
| 9 | 35 | Current Account Overdrafts | 11 | OVERDRAFT extended | NULL | NULL |
| 10 | 36 | Fees and Charges (30101) | 4 | INCOME_FROM_FEES (default) | NULL | NULL |
| 11 | 37 | Penalties (30102) | 5 | INCOME_FROM_PENALTIES | NULL | NULL |
| 12 | 38 | Interest Received from Borrowers | 12 | INCOME_FROM_INTEREST | NULL | NULL |
| 13 | 42 | Interest Paid To Depositors (40102) | 3 | INTEREST_ON_SAVINGS | NULL | NULL |
| 14 | 41 | Losses Written Off (40101) | 13 | LOSSES_WRITTEN_OFF | NULL | NULL |
| 15 | 27 | Voluntary Savings (10101) | 2 | SAVINGS_CONTROL | NULL | NULL |
| 16 | 55 | Liability Transfer (Temp) | 10 | TRANSFERS_SUSPENSE | NULL | NULL |
| 17 | 55 | Liability Transfer (Temp) | 1 | SAVINGS_REFERENCE override | **5** (M-Pesa) | NULL |

**Key observation — Row 17:** This is a **payment-type-specific SAVINGS_REFERENCE override**. When a transaction uses payment type 5 (M-Pesa), `getLinkedGLAccountForSavingsProduct()` returns GL 55 (Liability Transfer Temp) instead of the default GL 32 (Cash In Hand). This allows M-Pesa transactions to post to a different cash/settlement account than, say, bank transfers.

---

### 10.5 How the Three Columns (`payment_type`, `charge_id`, `financial_account_type`) Interact

```
Lookup order inside AccountingProcessorHelper:

1. Does a row exist WHERE product_id=X AND payment_type=<txn_payment_type> AND financial_account_type=Y?
   → If yes, use that GL account (payment-channel-specific mapping)

2. Does a row exist WHERE product_id=X AND charge_id=<charge_id> AND financial_account_type=Y?
   → If yes, use that GL account (per-charge product-level mapping)

3. Does a row exist WHERE product_id=X AND payment_type IS NULL AND charge_id IS NULL AND financial_account_type=Y?
   → This is the DEFAULT mapping — always falls back here
```

For charge GL resolution specifically, `m_charge.income_or_liability_account_id` takes **even higher priority** than all rows in `acc_product_mapping` (see Section 9).

---

### 10.6 Check Product GL Mapping — Full Query

```sql
SELECT 
    m.id,
    m.gl_account_id,
    g.name                      AS gl_account_name,
    g.gl_code,
    g.classification_enum,      -- 1=Asset,2=Liability,3=Equity,4=Income,5=Expense
    g.account_usage,            -- 1=Detail,2=Header
    m.financial_account_type,
    CASE m.financial_account_type
        WHEN 1  THEN 'SAVINGS_REFERENCE'
        WHEN 2  THEN 'SAVINGS_CONTROL'
        WHEN 3  THEN 'INTEREST_ON_SAVINGS'
        WHEN 4  THEN 'INCOME_FROM_FEES'
        WHEN 5  THEN 'INCOME_FROM_PENALTIES'
        WHEN 6  THEN 'TRANSFERS_SUSPENSE'
        WHEN 7  THEN 'OVERDRAFT_PORTFOLIO_CONTROL'
        WHEN 8  THEN 'LOSSES_WRITTEN_OFF'
        WHEN 9  THEN 'ESCHEAT_LIABILITY'
        WHEN 10 THEN 'TRANSFERS_SUSPENSE (extended)'
        WHEN 11 THEN 'OVERDRAFT_PORTFOLIO_CONTROL (extended)'
        WHEN 12 THEN 'INCOME_FROM_INTEREST'
        WHEN 13 THEN 'LOSSES_WRITTEN_OFF (extended)'
        ELSE         'UNKNOWN_' || m.financial_account_type::text
    END                         AS account_type_name,
    m.payment_type              AS payment_type_id,
    pt.value                    AS payment_type_name,
    m.charge_id,
    c.name                      AS charge_name
FROM acc_product_mapping m
JOIN  acc_gl_account g  ON g.id  = m.gl_account_id
LEFT JOIN m_payment_type pt ON pt.id = m.payment_type
LEFT JOIN m_charge c        ON c.id  = m.charge_id
WHERE m.product_id   = 4    -- replace with your savings product ID
  AND m.product_type = 2    -- 2 = SAVING
ORDER BY m.financial_account_type, m.payment_type NULLS FIRST;
```

---

### 10.7 Correct vs Incorrect Mapping — What to Check

| financial_account_type | Expected GL Classification | If Wrong |
|---|---|---|
| 2 (SAVINGS_CONTROL) | LIABILITY (2) | Withdrawal debits wrong account |
| 1 (SAVINGS_REFERENCE) | ASSET (1) | Cash side posts incorrectly |
| 4 (INCOME_FROM_FEES) | INCOME (4) | Fee default credits wrong account |
| 3 (INTEREST_ON_SAVINGS) | EXPENSE (5) | Interest expense posts wrongly |

> [!WARNING]
> If `SAVINGS_CONTROL` (type=2) is mapped to an INCOME or ASSET account (wrong classification), fee charges will produce entries that look reversed — customer balance will increase instead of decrease. Always confirm `classification_enum = 2` (LIABILITY) for the SAVINGS_CONTROL GL account.

---

### 10.8 Manually Insert a Missing Mapping

If a required mapping row is absent (e.g., you created the product before accounting was configured):

```sql
-- Insert default INCOME_FROM_FEES mapping for product 4
INSERT INTO acc_product_mapping 
    (gl_account_id, product_id, product_type, financial_account_type)
VALUES 
    (36, 4, 2, 4);  -- GL 36=Fees and Charges, product 4, SAVING, INCOME_FROM_FEES

-- Insert payment-type-specific SAVINGS_REFERENCE for M-Pesa (payment_type=5)
INSERT INTO acc_product_mapping 
    (gl_account_id, product_id, product_type, payment_type, financial_account_type)
VALUES 
    (55, 4, 2, 5, 1);  -- GL 55=Liability Transfer, payment_type 5=M-Pesa, SAVINGS_REFERENCE
```

> [!CAUTION]
> Prefer using the API (`PUT /savingsproducts/{id}` with the full accounting payload) over direct SQL inserts. Direct inserts bypass validation and may leave the Fineract cache stale until the server restarts.

---

### 10.9 Prerequisite: Currency Must Be Registered

> [!IMPORTANT]
> `ChargeReadPlatformServiceImpl` does an **INNER JOIN** with `m_organisation_currency`. If `KES` is not registered, `GET /charges` returns `[]` and `GET /charges/{id}` returns 404 — even though records exist.

```sql
SELECT * FROM m_organisation_currency;
```

If KES is missing:
```http
PUT /fineract-provider/api/v1/currencies
{ "currencies": ["KES"] }
```

---

## 11. How Journal Entries Are Generated

### Flow for a Withdrawal with Fees
```
1. POST /savingsaccounts/{id}/transactions (withdrawal)
2. SavingsAccountWritePlatformServiceJpaRepositoryImpl.handleWithdrawal()
3. SavingsAccount.withdraw() → SavingsAccount.payWithdrawalFee()
4. payWithdrawalFee() iterates m_savings_account_charge:
   - Matches payment type
   - Calls payCharge() → creates SavingsAccountTransaction (type=FEE_DEDUCTION)
5. Accounting processor triggered:
   - CashBasedAccountingProcessorForSavings.createJournalEntriesForSavings()
   - Detects isFeeDeduction() → calls createCashBasedJournalEntriesAndReversalsForSavingsCharges()
   - AccountingProcessorHelper resolves GL accounts:
       - SAVINGS_CONTROL → from acc_product_mapping (e.g., Voluntary Savings)
       - charge GL → from m_charge.income_or_liability_account_id (e.g., Fees and Charges)
   - Creates: DR SAVINGS_CONTROL, CR charge-specific GL
```

### Journal Entry Creation Code
```java
// CashBasedAccountingProcessorForSavings.java (line 218-221)
this.helper.createCashBasedJournalEntriesAndReversalsForSavingsCharges(
    office, currencyCode,
    CashAccountsForSavings.SAVINGS_CONTROL,   // → DEBITED (Voluntary Savings)
    CashAccountsForSavings.INCOME_FROM_FEES,   // → lookup type for charge GL
    savingsProductId, paymentTypeId, savingsId,
    transactionId, transactionDate, amount, isReversal, feePayments
);
```

### For Reversals
When `isReversal=true` (e.g., transaction is reversed/undone):
```
DR  Fees and Charges (30101)   KES 60   ← reverses the credit
    CR  Voluntary Savings (10101)  KES 60   ← restores customer balance
```

---

## 12. Verification Queries

### Full Pre-Transaction Checklist
```sql
-- 1. Charges exist and are active
SELECT id, name, is_active, is_deleted, income_or_liability_account_id
FROM m_charge WHERE id IN (1, 2);

-- 2. Charges linked to savings product
SELECT spc.savings_product_id, spc.charge_id, c.name
FROM m_savings_product_charge spc
JOIN m_charge c ON c.id = spc.charge_id
WHERE spc.savings_product_id = 4;

-- 3. Charges linked to savings account (CRITICAL)
SELECT sac.id, sac.charge_id, c.name, sac.amount, sac.is_active
FROM m_savings_account_charge sac
JOIN m_charge c ON c.id = sac.charge_id
WHERE sac.savings_account_id = 3;

-- 4. Charge GL account mapping
SELECT c.id, c.name, g.gl_code, g.name AS gl_name, 
       g.classification_enum, g.account_usage
FROM m_charge c
LEFT JOIN acc_gl_account g ON g.id = c.income_or_liability_account_id
WHERE c.id IN (1, 2);

-- 5. Savings product GL mapping
SELECT m.financial_account_type, g.gl_code, g.name
FROM acc_product_mapping m
JOIN acc_gl_account g ON g.id = m.gl_account_id
WHERE m.product_id = 4 AND m.product_type = 2
ORDER BY m.financial_account_type;

-- 6. Currency registered
SELECT code FROM m_organisation_currency WHERE code = 'KES';
```

### Post-Transaction Journal Entry Audit
```sql
-- Full journal entries for a transaction (use corrected DEBIT/CREDIT labels)
SELECT 
    je.transaction_id,
    acc.gl_code,
    acc.name AS gl_account_name,
    CASE WHEN je.type_enum = 2 THEN 'DEBIT' ELSE 'CREDIT' END AS entry_type,
    je.amount,
    acc.classification_enum  -- 1=Asset,2=Liability,3=Equity,4=Income,5=Expense
FROM acc_gl_journal_entry je
JOIN acc_gl_account acc ON acc.id = je.account_id
WHERE je.transaction_id = 'S183'   -- replace with actual transaction ID
ORDER BY je.type_enum DESC, je.amount DESC;

-- All charge-related journal entries for a savings account
SELECT 
    je.entry_date,
    je.transaction_id,
    acc.gl_code,
    acc.name AS gl_account_name,
    CASE WHEN je.type_enum = 2 THEN 'DEBIT' ELSE 'CREDIT' END AS entry_type,
    je.amount
FROM acc_gl_journal_entry je
JOIN acc_gl_account acc ON acc.id = je.account_id
WHERE je.savings_transaction_id IN (
    SELECT id FROM m_savings_account_transaction
    WHERE savings_account_id = 3
      AND transaction_type_enum = 9  -- 9 = FEE_DEDUCTION
)
ORDER BY je.entry_date DESC, je.transaction_id;
```

---

## 13. Known Bugs & Fixes

### Bug 1: `PUT /charges/{id}` Does Not Update `glAccountId` When Currently NULL

**Symptom:** API returns 200 but `income_or_liability_account_id` remains NULL.

**Root Cause (`Charge.java` line ~600):**
```java
// BEFORE (buggy):
if (command.isChangeInLongParameterNamed(ChargesApiConstants.glAccountIdParamName, getIncomeAccountId())) {
    actualChanges.put(ChargesApiConstants.glAccountIdParamName, newValue);
}
// When getIncomeAccountId() = null, isChangeInLongParameterNamed returns false → no update
```

**Fix Applied:**
```java
// AFTER (fixed):
if (command.parameterExists(ChargesApiConstants.glAccountIdParamName)) {
    final Long newValue = command.longValueOfParameterNamed(ChargesApiConstants.glAccountIdParamName);
    final Long currentValue = getIncomeAccountId();
    if (!Objects.equals(currentValue, newValue)) {
        actualChanges.put(ChargesApiConstants.glAccountIdParamName, newValue);
    }
}
```

**Immediate Workaround (no redeploy needed):**
```sql
UPDATE m_charge SET income_or_liability_account_id = 36 WHERE id = 1;
UPDATE m_charge SET income_or_liability_account_id = 56 WHERE id = 2;
```

---

### Bug 2: PostgreSQL Sequence Out of Sync

**Symptom:** `POST /glaccounts` returns 403 with `duplicate key value violates unique constraint "acc_gl_account_pkey"`.

**Root Cause:** DB was populated via Flyway/bulk import without advancing the sequence.

**Fix:**
```sql
SELECT setval(
    pg_get_serial_sequence('acc_gl_account', 'id'),
    (SELECT MAX(id) FROM acc_gl_account)
);
```

---

### Bug 3: `GET /charges` Returns Empty `[]` Despite Records Existing

**Symptom:** `GET /charges` → `[]`, `GET /charges/1` → 404.

**Root Cause:** `ChargeReadPlatformServiceImpl` does an INNER JOIN with `m_organisation_currency`. If the currency (e.g., `KES`) is not registered, the join returns nothing.

**Fix:**
```http
PUT /fineract-provider/api/v1/currencies
{ "currencies": ["KES"] }
```

---

### Bug 4: `POST /savingsaccounts/{id}/charges` Rejects Extra Fields

**Symptom:** API returns 400 with "unsupported parameter" error.

**Cause:** The endpoint's validator (`SavingsAccountChargeDataValidator`) explicitly rejects `chargeTimeType` and `chargeCalculationType` — these are read from the charge definition, not the request.

**Correct Minimal Payload:**
```json
{
    "chargeId": 1,
    "amount": 60,
    "locale": "en"
}
```

---

## Quick Reference: Charge Type Enums

### `chargeTimeType` for Savings (`chargeAppliesTo: 2`)
| Value | Description |
|---|---|
| 2 | Specified Due Date |
| 5 | **Withdrawal Fee** ← use this for M-Pesa/withdrawal charges |
| 6 | Annual Fee |
| 7 | Monthly Fee |
| 8 | Overdraft Fee |

### `chargeCalculationType`
| Value | Description |
|---|---|
| 1 | **Flat** — fixed amount (KES 60) |
| 2 | Percent of Amount |

### `chargeAppliesTo`
| Value | Description |
|---|---|
| 1 | Loan |
| 2 | **Savings** |
| 4 | Client |

---

## 14. Integrating External Payment Channels

When integrating external payment gateways (Mobile Money, SWIFT, ATMs, Remittances) with Fineract, the standard and recommended practice is to use the core savings transaction endpoints.

### API Endpoints
- **Receiving Money (Inflow):** Use `/v1/savingsaccounts/{savingsAccountId}/transactions?command=deposit`
- **Sending Money (Outflow):** Use `/v1/savingsaccounts/{savingsAccountId}/transactions?command=withdrawal`
- **Account-to-Account Transfers:** Use `/v1/accounttransfers` (for internal transfers between two Fineract accounts)

### Passing External Metadata
Fineract natively supports tracking external transaction details via the **Payment Types** configuration. When making a `deposit` or `withdrawal` API call, you can include the following metadata in the JSON payload:

| Field | Purpose |
|---|---|
| `paymentTypeId` | **CRITICAL:** ID of the payment type (e.g., M-Pesa, ATM). Maps to `m_payment_type.id`. |
| `receiptNumber` | The external transaction ID (e.g., M-Pesa receipt). Crucial for reconciliation. |
| `routingCode` | Useful for SWIFT or RTGS transfers. |
| `accountNumber` | The external bank account or mobile wallet number. |
| `bankNumber` | Useful for external banking integrations. |
| `checkNumber` | Used if dealing with physical cheques. |

### Example Payload (Receiving Mobile Money)
```json
{
  "dateFormat": "dd MMMM yyyy",
  "locale": "en",
  "transactionDate": "30 April 2026",
  "transactionAmount": 500.00,
  "paymentTypeId": 2, 
  "receiptNumber": "RTY789UIO", 
  "accountNumber": "+254712345678" 
}
```

### Why This is the Standard Approach
1. **Automated Accounting:** By using standard deposit/withdrawal endpoints, Fineract's accounting engine automatically generates the correct double-entry journal entries based on the Product and Charge GL mappings.
2. **Reconciliation:** Saving the external reference as `receiptNumber` enables the finance team to easily reconcile Fineract's ledger against the external gateway's statements.
3. **Triggering Payment-Specific Charges:** As discussed in Section 7, using `paymentTypeId` ensures that only the relevant charges (e.g., M-Pesa withdrawal fees) are applied to the transaction.
