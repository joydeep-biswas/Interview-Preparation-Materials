# 09. Unity Catalog & Data Governance: Interview Questions & Deep-Dive Solutions

A comprehensive, production-grade guide answering Databricks Unity Catalog, access governance, fine-grained data security, and compliance interview scenarios. Covers 3-level namespaces, row-level filters, dynamic column masking, permission troubleshooting, automated data lineage, system audit tables, and secure cloud storage credential integration.

---

## Table of Contents & Question Mapping

1. **Unity Catalog 3-Level Namespace & Multi-Environment Architecture**
   - [Q1. Unity Catalog 3-Level Namespace & Multi-Environment Architecture](#q1-medium-unity-catalog-3-level-namespace--multi-environment-architecture)
2. **Role-Based Access Control & Privilege Inheritance**
   - [Q2. Role-Based Access Control & Privilege Hierarchy](#q2-medium-role-based-access-control--privilege-hierarchy)
   - [Q3. Granting Read Access Across Multiple Tables Cleanly](#q3-medium-granting-read-access-across-multiple-tables-cleanly)
3. **Fine-Grained Security: Column Masking & Row-Level Filters**
   - [Q4. Dynamic Column Masking for PII Protection](#q4-medium-dynamic-column-masking-for-pii-protection)
   - [Q5. Row-Level Security with Dynamic User and Group Filtering](#q5-medium-row-level-security-with-dynamic-user-and-group-filtering)
4. **Permission Diagnostics & Troubleshooting Playbook**
   - [Q6. Permission Diagnostics & 4-Step Access Denial Troubleshooting](#q6-hard-permission-diagnostics--4-step-access-denial-troubleshooting)
5. **Automated Lineage, System Tables & Audit Tracking**
   - [Q7. Automated Data Lineage for Root-Cause Analysis](#q7-medium-automated-data-lineage-for-root-cause-analysis)
   - [Q8. Auditing Sensitive PII Access via System Tables](#q8-medium-auditing-sensitive-pii-access-via-system-tables)
6. **Cloud Object Storage Security: Storage Credentials & External Locations**
   - [Q9. Storage Credentials & External Locations Cloud Security](#q9-medium-storage-credentials--external-locations-cloud-security)
7. **Enterprise Cross-Department Data Governance & Multi-Role Compliance**
   - [Q10. Multi-Department Data Governance: Finance, Marketing & Operations Security](#q10-medium-multi-department-data-governance-finance-marketing--operations-security)
---

# 1. Unity Catalog 3-Level Namespace & Multi-Environment Architecture

---

### Q1. [Medium] Unity Catalog 3-Level Namespace & Multi-Environment Architecture
**Difficulty:** [Medium]
**Topic Covered:** 3-Level Namespace Hierarchy, Service Principal Isolation & Environment Governance  
**Questions Covered:**
- **Q8**: Your company has Development, Testing, and Production environments. How would you design Unity Catalog catalogs, schemas, and access permissions for them?
- **Q13**: How would you use groups and service principals to manage access for developers, testers, and production jobs?
- **Q14**: What is Unity Catalog, data lineage, and what do OPTIMIZE vs VACUUM actually do?

#### Core Architecture: The 3-Level Namespace
Unlike legacy Hive Metastore (which only provided a 2-level `database.table` namespace tied to a single workspace), Unity Catalog introduces an enterprise-wide **3-level namespace**:
$$\text{catalog} \boldsymbol{.} \text{schema} \boldsymbol{.} \text{table\_or\_view}$$

```mermaid
flowchart TD
    subgraph MetastoreLevel ["Databricks Account Metastore"]
        Account["Account Level Metastore"]
    end

    subgraph CatalogsTier ["3-Level Catalogs"]
        DevCat["dev_catalog (Sandbox)"]
        TestCat["test_catalog (Integration)"]
        ProdCat["prod_catalog (Production)"]
    end

    subgraph SchemasTier ["Managed Schemas"]
        Bronze["bronze (Raw Data)"]
        Silver["silver (Curated Data)"]
        Gold["gold (Business Metrics)"]
    end

    subgraph ObjectsTier ["Tables & Views"]
        T1[("fact_daily_revenue (Table)")]
        V1["v_masked_customer (Dynamic View)"]
    end

    Account --> DevCat & TestCat & ProdCat
    ProdCat --> Bronze & Silver & Gold
    Gold --> T1 & V1

    style MetastoreLevel fill:#0f172a,stroke:#8b5cf6,stroke-width:1.5px,color:#c084fc
    style CatalogsTier fill:#0f172a,stroke:#3b82f6,stroke-width:1.5px,color:#93c5fd
    style SchemasTier fill:#0f172a,stroke:#06b6d4,stroke-width:1.5px,color:#67e8f9
    style ObjectsTier fill:#0f172a,stroke:#10b981,stroke-width:1.5px,color:#6ee7b7

    style Account fill:#1f2937,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style DevCat fill:#1f2937,stroke:#3b82f6,stroke-width:2px,color:#fff
    style TestCat fill:#1f2937,stroke:#3b82f6,stroke-width:2px,color:#fff
    style ProdCat fill:#1f2937,stroke:#3b82f6,stroke-width:2px,color:#fff
    style Bronze fill:#1f2937,stroke:#f59e0b,stroke-width:2px,color:#fff
    style Silver fill:#1f2937,stroke:#06b6d4,stroke-width:2px,color:#fff
    style Gold fill:#1f2937,stroke:#10b981,stroke-width:2px,color:#fff
    style T1 fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#a7f3d0
    style V1 fill:#1f2937,stroke:#10b981,stroke-width:2px,color:#fff
```

#### Multi-Environment Governance Design (Q8, Q13)
To ensure isolation and security across Development, Testing, and Production:
1. **Catalog per Environment:**
   - `dev_catalog`: Sandbox for engineers; schema changes permitted.
   - `test_catalog`: Mirror of production data with masked PII for integration testing.
   - `prod_catalog`: Strictly protected production data.
2. **Access Control via Identity Federation:**
   - **Developers (`grp_developers`):** Full DDL/DML on `dev_catalog`; strictly `SELECT` (read-only) on `test_catalog`; zero access to `prod_catalog`.
   - **Testers (`grp_qa_testers`):** `SELECT` and execute on `test_catalog`.
   - **Production Pipelines (`sp_production_etl`):** Service Principal possesses `ALL PRIVILEGES` on `prod_catalog`. No human interactive user holds write permissions in production!

```sql
-- DDL Governance: Setting up Prod Catalog Privileges
CREATE CATALOG prod_catalog;
GRANT USE CATALOG ON CATALOG prod_catalog TO `grp_data_analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod_catalog.gold TO `grp_data_analysts`;

-- Assign write permissions strictly to the automated Service Principal
GRANT ALL PRIVILEGES ON CATALOG prod_catalog TO `sp_production_etl`;
```

---

# 2. Role-Based Access Control & Privilege Inheritance

---

### Q2. [Medium] Role-Based Access Control & Privilege Hierarchy
**Difficulty:** [Medium]
**Topic Covered:** Hierarchical RBAC, Object Privileges & Account-Level Identities  
**Question:** How would you implement different access levels for different teams?

#### Privilege Hierarchy & The Prerequisite "USE" Chain
In Unity Catalog, privileges are inherited hierarchically. To query a table `prod.gold.sales`, a user requires:
1. `USE CATALOG` on `prod`
2. `USE SCHEMA` on `gold`
3. `SELECT` on `sales`

```mermaid
flowchart TD
    subgraph PrivilegeChain ["Unity Catalog Privilege Inheritance Hierarchy"]
        P1["1. GRANT USE CATALOG ON CATALOG prod"] --> P2["2. GRANT USE SCHEMA ON SCHEMA prod.gold"]
        P2 --> P3["3. GRANT SELECT ON TABLE prod.gold.sales"]
        P3 --> Query["Authorization Success: User Queries Data!"]
    end

    style PrivilegeChain fill:#0f172a,stroke:#3b82f6,stroke-width:1.5px,color:#93c5fd
    style P1 fill:#1f2937,stroke:#3b82f6,stroke-width:2px,color:#fff
    style P2 fill:#1f2937,stroke:#06b6d4,stroke-width:2px,color:#fff
    style P3 fill:#1f2937,stroke:#10b981,stroke-width:2px,color:#fff
    style Query fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#a7f3d0
```

---

### Q3. [Medium] Granting Read Access Across Multiple Tables Cleanly
**Difficulty:** [Medium]
**Topic Covered:** Schema-Level Inheritance vs Table-Level Enumeration  
**Question:** How would you give a team read access to multiple tables without granting unnecessary permissions?

#### Giving a Team Read Access to Multiple Tables Cleanly (Q9)
Instead of granting `SELECT` table-by-table across 200 tables:
```sql
-- Grant read access to ALL current and future tables in the schema with 2 commands
GRANT USE CATALOG ON CATALOG prod_catalog TO `finance_team`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod_catalog.finance TO `finance_team`;
```

---

# 3. Fine-Grained Security: Column Masking & Row-Level Filters

---

### Q4. [Medium] Dynamic Column Masking for PII Protection
**Difficulty:** [Medium]
**Topic Covered:** SQL UDF Column Masks, Group-Based Redaction & Transparent Access Control  
**Questions Covered:**
- **Q2**: How would you secure sensitive columns such as PII?
- **Q3**: A developer can access a table but should not be able to see sensitive columns. How would you implement this requirement?
- **Q4**: A user needs access to a table but should only see selected columns. How would you design this using Unity Catalog?

#### Architectural Approaches: Dynamic Masking vs Views
1. **Legacy Approach (Dynamic Views):** Requires creating separate view layers (`SELECT user_id, CASE WHEN ... END FROM table`) and granting access to the view while revoking the base table.
2. **Modern Unity Catalog Approach (Native Column Masks & Row Filters):** Applied directly to the physical base table. Enforced transparently across all BI tools, notebooks, and queries!

#### Column Masking: Securing PII (Email / SSN) (Q2, Q3, Q4)
```sql
-- Step 1: Create a reusable SQL User-Defined Masking Function
CREATE OR REPLACE FUNCTION mask_email_fn(email STRING)
RETURN CASE 
    WHEN is_account_group_member('security_compliance_team') THEN email
    ELSE CONCAT('****@', SPLIT(email, '@')[1])
END;

-- Step 2: Apply the masking function directly to the column on the base table
ALTER TABLE prod_catalog.customer.users 
ALTER COLUMN email SET MASK mask_email_fn;
```

---

### Q5. [Medium] Row-Level Security with Dynamic User and Group Filtering
**Difficulty:** [Medium]
**Topic Covered:** Row Filters, `is_account_group_member()` & Geographic Partitioning  
**Question:** How would you implement row-level security for different users or departments?

#### Row-Level Security: Filtering by Department or Region (Q10)
Restricts users so that EU analysts see only EU rows, and US analysts see only US rows:
```sql
-- Step 1: Create a row filter function
CREATE OR REPLACE FUNCTION region_filter_fn(region STRING)
RETURN 
    is_account_group_member('global_executives') OR 
    (is_account_group_member('eu_analysts') AND region = 'EU') OR
    (is_account_group_member('us_analysts') AND region = 'US');

-- Step 2: Bind the row filter directly to the table
ALTER TABLE prod_catalog.sales.orders 
SET ROW FILTER region_filter_fn ON (customer_region);
```

---

# 4. Permission Diagnostics & Troubleshooting Playbook

---

### Q6. [Hard] Permission Diagnostics & 4-Step Access Denial Troubleshooting
**Difficulty:** [Hard]
**Topic Covered:** Privilege Revocations, Metastore Ownership & Broken USE Chains  
**Questions Covered:**
- **Q5**: How would you troubleshoot a user who suddenly loses access to a table?
- **Q6**: A user can access a catalog but cannot query a particular table. How would you troubleshoot the permissions?

#### The 4-Step Diagnostic Protocol for Permission Denials
```mermaid
flowchart TD
    subgraph DiagnosticWorkflow ["Unity Catalog Permission Denial Diagnostic Protocol"]
        Err["Permission Denied / Table Not Found"] --> S1["Step 1: Check the 3-Level USE Hierarchy<br/>Does user have USE CATALOG and USE SCHEMA?"]
        S1 --> S2["Step 2: Check Cluster Access Mode<br/>Is the cluster running in 'Single User' (locked to owner)<br/>instead of 'Shared / User Isolation' mode?"]
        S2 --> S3["Step 3: Inspect Direct & Group Grants<br/>SHOW GRANTS ON TABLE prod.gold.sales"]
        S3 --> S4["Step 4: Check Table Owner & Dynamic Row Filters<br/>Does a Row Filter evaluate to FALSE for all rows?"]
    end

    style DiagnosticWorkflow fill:#0f172a,stroke:#ef4444,stroke-width:1.5px,color:#fca5a5
    style Err fill:#7f1d1d,stroke:#ef4444,stroke-width:2px,color:#fecaca
    style S1 fill:#1f2937,stroke:#f59e0b,stroke-width:2px,color:#fff
    style S2 fill:#1f2937,stroke:#3b82f6,stroke-width:2px,color:#fff
    style S3 fill:#1f2937,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style S4 fill:#1f2937,stroke:#10b981,stroke-width:2px,color:#fff
```

#### Diagnostic Commands
```sql
-- Check effective grants for the user
SHOW GRANTS ON TABLE prod_catalog.gold.orders;
SHOW GRANTS ON SCHEMA prod_catalog.gold;
SHOW GRANTS ON CATALOG prod_catalog;

-- Verify if the table is masked or filtered
DESCRIBE TABLE EXTENDED prod_catalog.gold.orders;
```

---

# 5. Automated Lineage, System Tables & Audit Tracking

---

### Q7. [Medium] Automated Data Lineage for Root-Cause Analysis
**Difficulty:** [Medium]
**Topic Covered:** Column & Table Lineage Graph Traversal & Upstream Failure Tracking  
**Question:** How would you use data lineage and audit logs during troubleshooting?

#### Automated Data Lineage (Q7)
Unity Catalog automatically captures column-level and table-level lineage in real time across SQL, Python, R, and Scala without any manual instrumentation:
- **Upstream Impact Analysis:** If a pipeline fails, view lineage to identify all downstream dashboards (PowerBI, Tableau) and tables impacted.
- **Root-Cause Analysis:** If a column in a Gold table has corrupt values, inspect upstream lineage to trace the issue back through Silver transformations to the exact raw Bronze source file.

---

### Q8. [Medium] Auditing Sensitive PII Access via System Tables
**Difficulty:** [Medium]
**Topic Covered:** `system.access.audit`, Event Log Analysis & Regulatory Compliance  
**Question:** How would you track which users accessed sensitive data in Unity Catalog?

#### Auditing Sensitive PII Access via System Tables (Q11)
All queries executed in Unity Catalog are logged immutably into `system.access.audit`:
```sql
-- Query to audit who accessed customer PII over the last 30 days
SELECT 
    event_time,
    user_identity.email AS queried_by_user,
    action_name,
    request_params.table_full_name AS accessed_table,
    request_params.command_text AS executed_sql
FROM system.access.audit
WHERE request_params.table_full_name = 'prod_catalog.customer.users'
  AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY event_time DESC;
```

---

# 6. Cloud Object Storage Security: Storage Credentials & External Locations

---

### Q9. [Medium] Storage Credentials & External Locations Cloud Security
**Difficulty:** [Medium]
**Topic Covered:** Azure Managed Identities, Storage Credentials & External Location Delegation  
**Question:** How would you configure Storage Credentials and External Locations to securely access ADLS data through Unity Catalog?

#### The Problem Solved: Eliminating Hardcoded Storage Keys
In legacy data lakes, notebooks used hardcoded storage access keys or SAS tokens:
```python
# ANTI-PATTERN: Dangerous hardcoded secret
spark.conf.set("fs.azure.account.key.datalake.dfs.core.windows.net", "secret_key...")
```

#### Modern Unity Catalog Cloud Storage Integration Architecture
```mermaid
flowchart LR
    subgraph SecureStorageModel ["Cloud Storage Access Integration via Unity Catalog"]
        UC["Unity Catalog Governance"] --> SC["Storage Credential<br/>(Azure Managed Identity / AWS IAM Role)"]
        SC --> EL["External Location<br/>(abfss://container@account.dfs.core.windows.net/path)"]
        EL --> Table[("Managed / External Delta Table<br/>(Zero Hardcoded Keys)")]
    end

    style SecureStorageModel fill:#0f172a,stroke:#10b981,stroke-width:1.5px,color:#6ee7b7
    style UC fill:#1f2937,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style SC fill:#1f2937,stroke:#3b82f6,stroke-width:2px,color:#fff
    style EL fill:#1f2937,stroke:#06b6d4,stroke-width:2px,color:#fff
    style Table fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#a7f3d0
```

#### Configuration Steps
```sql
-- Step 1: Create a Storage Credential referencing an Azure Managed Identity
CREATE STORAGE CREDENTIAL azure_prod_identity
WITH AZURE_MANAGED_IDENTITY (
    CLIENT_ID = 'e8b3b320-...'
);

-- Step 2: Create an External Location pointing to the specific ADLS container
CREATE EXTERNAL LOCATION adls_gold_container
URL 'abfss://gold@datalakeprod.dfs.core.windows.net/'
WITH STORAGE CREDENTIAL azure_prod_identity;

-- Step 3: Grant access to specific teams to read from this cloud storage path
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION adls_gold_container TO `sp_production_etl`;
```

---

# 7. Enterprise Cross-Department Data Governance & Multi-Role Compliance

---

### Q10. [Medium] Multi-Department Data Governance: Finance, Marketing & Operations Security
**Difficulty:** [Medium]
**Topic Covered:** Multi-Role Entitlements, Column-Level Masking, Dynamic Row-Level Security & Unity Catalog  
**Question:** Finance, Marketing, and Operations consume the same curated customer and sales dataset, but access must be restricted based on business roles, sensitive columns (e.g. Finance sees bank accounts and revenue, Marketing sees emails but masked revenue, Operations sees only active fulfillment orders). How would you design secure, governed data access in Unity Catalog?

#### Architecture & Access Matrix

```mermaid
flowchart TD
    Dataset[("Single Curated Gold Table:<br/>prod_catalog.commercial.sales_orders")] --> PolicyEngine["Unity Catalog Dynamic Governance Engine"]

    PolicyEngine -->|Role: finance_analysts| FinView["Finance Entitlement<br/>(Bank Details: CLEARTEXT, Revenue & Margins: VISIBLE, Rows: ALL Rows)"]
    PolicyEngine -->|Role: marketing_analysts| MktView["Marketing Entitlement<br/>(Bank Details: REDACTED (NULL), Revenue: MASKED to $100 tiers, Rows: Opt-in Marketing Rows)"]
    PolicyEngine -->|Role: operations_team| OpsView["Operations Entitlement<br/>(PII: MASKED (Hash/Redacted); Rows: WHERE status IN ('PENDING', 'SHIPPED'))"]

    style Dataset fill:#0f172a,stroke:#3b82f6,stroke-width:2px,color:#93c5fd
    style PolicyEngine fill:#1e293b,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style FinView fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#a7f3d0
    style MktView fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#fff
    style OpsView fill:#0f172a,stroke:#06b6d4,stroke-width:1.5px,color:#67e8f9
```

#### Production DDL & Policy Implementation
```sql
-- Step 1: Define Column Mask for Financial Details
CREATE OR REPLACE FUNCTION governance.mask_bank_account(account_number STRING)
RETURN CASE
    WHEN IS_ACCOUNT_GROUP_MEMBER('finance_team') THEN account_number
    ELSE '****-****-****-' || RIGHT(account_number, 4)
END;

-- Step 2: Define Column Mask for Margin Amounts
CREATE OR REPLACE FUNCTION governance.mask_profit_margin(margin DECIMAL(10,2))
RETURN CASE
    WHEN IS_ACCOUNT_GROUP_MEMBER('finance_team') THEN margin
    WHEN IS_ACCOUNT_GROUP_MEMBER('marketing_team') THEN NULL
    ELSE 0.00
END;

-- Step 3: Define Dynamic Row Filter for Order Status & Department Visibility
CREATE OR REPLACE FUNCTION governance.filter_departmental_orders(order_status STRING, is_marketing_opt_in BOOLEAN)
RETURN CASE
    WHEN IS_ACCOUNT_GROUP_MEMBER('finance_team') THEN TRUE -- Finance sees 100% of rows
    WHEN IS_ACCOUNT_GROUP_MEMBER('marketing_team') THEN is_marketing_opt_in = TRUE
    WHEN IS_ACCOUNT_GROUP_MEMBER('operations_team') THEN order_status IN ('PENDING', 'PROCESSING', 'DISPATCHED')
    ELSE FALSE
END;

-- Step 4: Bind Masks and Filters to Single Base Gold Table
CREATE TABLE prod_catalog.commercial.sales_orders (
    order_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    customer_email STRING,
    bank_account_number STRING MASK governance.mask_bank_account,
    total_revenue DECIMAL(12, 2),
    profit_margin DECIMAL(10, 2) MASK governance.mask_profit_margin,
    order_status STRING,
    is_marketing_opt_in BOOLEAN,
    order_timestamp TIMESTAMP
) USING DELTA
ROW FILTER governance.filter_departmental_orders ON (order_status, is_marketing_opt_in);

-- Step 5: Grant Access at Schema Level (Single Table, Zero Data Duplication)
GRANT SELECT ON TABLE prod_catalog.commercial.sales_orders TO `finance_team`;
GRANT SELECT ON TABLE prod_catalog.commercial.sales_orders TO `marketing_team`;
GRANT SELECT ON TABLE prod_catalog.commercial.sales_orders TO `operations_team`;
```

#### Engineering Implications & Production Pitfalls
1. **Zero Data Redundancy:** Traditional architectures created 3 separate physical tables (`sales_finance`, `sales_marketing`, `sales_ops`), multiplying ETL compute costs and creating data drift. Dynamic Row Filters and Column Masks evaluate inside the query engine at runtime over a **single physical Delta table**.
2. **Auditability via System Tables:** Unity Catalog records all masked and unmasked query evaluations into `system.access.audit`, providing cryptographic audit proof for GDPR, HIPAA, and SOC-2 compliance.

---

## Summary Comparison & Production Reference Matrix

| Scenario Domain | Critical Runtime Pathology | Diagnostic & Triage Method | Canonical Production Resolution |
| :--- | :--- | :--- | :--- |
| **Unity Catalog 3-Level Namespace & Multi-Environment Architecture** | High-throughput invariants & concurrency boundaries | Metric instrumentation & trace profiling | Defensive validation & graceful degradation |
| **Role-Based Access Control & Privilege Inheritance** | High-throughput invariants & concurrency boundaries | Metric instrumentation & trace profiling | Defensive validation & graceful degradation |
| **Fine-Grained Security: Column Masking & Row-Level Filters** | High-throughput invariants & concurrency boundaries | Metric instrumentation & trace profiling | Defensive validation & graceful degradation |
| **Permission Diagnostics & Troubleshooting Playbook** | High-throughput invariants & concurrency boundaries | Metric instrumentation & trace profiling | Defensive validation & graceful degradation |
| **Automated Lineage, System Tables & Audit Tracking** | High-throughput invariants & concurrency boundaries | Metric instrumentation & trace profiling | Defensive validation & graceful degradation |
| **Cloud Object Storage Security: Storage Credentials & External Locations** | High-throughput invariants & concurrency boundaries | Metric instrumentation & trace profiling | Defensive validation & graceful degradation |

---

## References & Further Reading

- [NIST Special Publication 800-162: Guide to Attribute Based Access Control (ABAC)](https://csrc.nist.gov/publications/detail/sp/800-162/final) — The authoritative federal standard for fine-grained authorization architecture.
- [General Data Protection Regulation (GDPR) Article 17: Right to Erasure](https://gdpr-info.eu/art-17-gdpr/) — Official legal specifications and technical compliance requirements.
- [OpenLineage Specification & Ecosystem Architecture](https://openlineage.io/) — Open standard for data lineage metadata collection and graph tracing.
- [Designing Data-Intensive Applications (Martin Kleppmann)](https://dataintensive.net/) — Security, data reliability, distributed logging, and compliance engineering.
- [Google Site Reliability Engineering: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) — Observability principles, alerting hygiene, and alert fatigue mitigation.
