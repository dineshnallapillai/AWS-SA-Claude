# AWS SAP-C02 Study Notes: Lake Formation (Permissions, Data Lake)

---

## 1. Core Concepts & Theory

### What is AWS Lake Formation?

**Definition:** A fully managed service that simplifies building, securing, and managing data lakes. It centralizes data governance, access control, and data cataloging — replacing the complex web of IAM policies, S3 bucket policies, and Glue Data Catalog permissions with a single permissions model.

**The problem it solves:** Without Lake Formation, securing a data lake requires coordinating:
- S3 bucket policies
- IAM policies
- Glue Data Catalog resource policies
- Athena workgroup permissions
- Redshift Spectrum IAM roles
- KMS key policies

Lake Formation replaces this with **one central permissions layer** using grant/revoke semantics (like a traditional database RBAC model).

---

### Architecture & How It Fits

```
[Data Sources] → [Lake Formation Blueprints/Workflows] → [S3 Data Lake]
                                                              ↓
                                                    [Glue Data Catalog]
                                                              ↓
                                              [Lake Formation Permissions]
                                                              ↓
                                         [Analytics Services: Athena, Redshift Spectrum,
                                          EMR, Glue ETL, QuickSight]
```

**Core building blocks:**

| Component | Purpose |
|-----------|---------|
| **Data Lake (S3)** | Central storage for raw and processed data |
| **Glue Data Catalog** | Metadata store (databases, tables, columns) — Lake Formation manages permissions ON TOP of this |
| **Lake Formation Permissions** | Grant/revoke access to databases, tables, columns |
| **Blueprints** | Pre-built ingestion workflows (from RDS, CloudTrail logs, etc.) |
| **Workflows** | Orchestrated ETL jobs (Glue crawlers + jobs) created from blueprints |
| **Data Filters** | Row-level and cell-level security |
| **Tags (LF-Tags)** | Attribute-based access control at scale |
| **Governed Tables** | ACID transactions on S3 data (Lake Formation managed) |

---

### Key Components Deep-Dive

#### Data Lake Administrator

- A principal (IAM user/role) with full Lake Formation permissions
- Can grant/revoke permissions to any other principal
- Can register S3 locations
- Can create databases and manage the Data Catalog
- Does NOT automatically have IAM permissions — still needs basic IAM permissions to access the Lake Formation console/API

#### Data Lake Locations (Registered S3 Paths)

- You **register** S3 paths with Lake Formation
- Once registered, Lake Formation manages access (not S3 bucket policies)
- Lake Formation uses a **service-linked role** (`AWSServiceRoleForLakeFormationDataAccess`) to read/write the registered S3 locations
- Alternatively, use a **custom IAM role** for the registered location (the role must have S3 access)

#### Permissions Model

**Two modes:**
1. **Lake Formation permissions (recommended):** Fine-grained, centralized, grant/revoke model
2. **IAM-only mode (legacy):** Falls back to IAM + S3 policies (Lake Formation is essentially bypassed)

**Transition:** When you first enable Lake Formation, a special group called `IAMAllowedPrincipals` has `Super` permissions on all Data Catalog resources → this preserves backward compatibility. To enforce Lake Formation permissions, **remove** the `IAMAllowedPrincipals` grants.

**Permission types:**

| Permission | Applies To | Description |
|------------|-----------|-------------|
| `SELECT` | Table | Read data (query) |
| `INSERT` | Table | Write/add data |
| `DELETE` | Table | Delete data |
| `DESCRIBE` | Database/Table | View metadata |
| `ALTER` | Database/Table | Modify metadata (schema) |
| `DROP` | Database/Table | Delete the resource |
| `CREATE_DATABASE` | Catalog | Create new databases |
| `CREATE_TABLE` | Database | Create tables in a database |
| `DATA_LOCATION_ACCESS` | S3 path | Permission to create tables pointing to that S3 location |
| `SUPER` | Any | All permissions (admin-level) |

**Grantable permissions:** When granting, you can optionally allow the grantee to **re-grant** those permissions to others (similar to `WITH GRANT OPTION` in SQL).

---

#### LF-Tags (Tag-Based Access Control)

**What it is:** Attribute-based access control (ABAC) for data lake resources. Assign key-value tags to databases, tables, or columns. Then grant permissions based on tag expressions.

**How it works:**
1. Define LF-Tag keys and allowed values (e.g., `sensitivity = [public, confidential, restricted]`)
2. Assign tags to Data Catalog resources (e.g., table `customer_pii` gets `sensitivity = restricted`)
3. Grant permissions using tag expressions (e.g., "Data Science role can SELECT on resources where `sensitivity = public OR confidential`")

**Why it's powerful:**
- Scales to thousands of tables without individual grants
- New tables auto-inherit permissions if tagged appropriately
- Single grant covers all current AND future resources matching the tag
- Much easier to audit than per-resource grants

**Example:**
```
LF-Tag: department = [finance, engineering, marketing]
LF-Tag: sensitivity = [public, internal, restricted]

Grant: role/DataAnalyst → SELECT where department=marketing AND sensitivity IN (public, internal)
```

---

#### Data Filters (Row & Cell-Level Security)

**Row-level security:**
- Define a filter expression (e.g., `country = 'US'`) on a table
- Grant SELECT with the filter → principal can only see rows matching the filter
- Uses PartiQL syntax for filter expressions

**Column-level security:**
- Include or exclude specific columns in a grant
- Two modes: **include list** (only these columns) or **exclude list** (all columns except these)

**Cell-level security (combination):**
- Apply both row filter AND column filter simultaneously
- Example: "Analyst can see columns [name, email, purchase_amount] only for rows where region = 'EU'"

---

#### Governed Tables & ACID Transactions

**What it is:** Tables in Lake Formation that support ACID transactions on S3 data.

**Capabilities:**
- **Atomic writes:** Multiple changes commit or rollback together
- **Consistency:** Readers always see a consistent snapshot
- **Isolation:** Concurrent readers/writers don't interfere
- **Time travel:** Query data as of a specific timestamp
- **Storage optimization:** Automatic compaction of small files

**Note:** AWS has since aligned this with **Apache Iceberg** table format support in Glue/Lake Formation. For the exam, know that Lake Formation supports transactional data lakes.

---

#### Blueprints & Workflows

**Blueprints** = pre-built templates for common ingestion patterns:
- **Database blueprint:** Ingest from JDBC sources (RDS, on-prem databases) — full or incremental load
- **Log blueprint:** Ingest CloudTrail, ELB logs, or other structured logs
- **Custom blueprint:** Build your own

**Workflow** = an instantiation of a blueprint that creates:
- Glue crawlers (to discover schema)
- Glue ETL jobs (to transform/load data)
- Triggers (to orchestrate execution)

---

### Cross-Account Data Sharing

**Lake Formation enables secure cross-account data sharing without copying data.**

**Methods:**

| Method | Description | Use Case |
|--------|-------------|----------|
| **Named resource grants** | Grant permissions directly to an external AWS account/org | Simple cross-account sharing |
| **LF-Tag grants (cross-account)** | Share tags with external accounts; they inherit tag-based permissions | Scalable sharing across many accounts |
| **AWS RAM (Resource Access Manager)** | Share Data Catalog resources via RAM — required for cross-account with external accounts not in your org | External organization sharing |

**How cross-account works:**
1. Producer account grants `SELECT` on a table to consumer account (or shares via RAM)
2. Consumer account creates a **resource link** (pointer) in their local Data Catalog
3. Consumer's users/roles query via the resource link → Lake Formation enforces permissions
4. Data stays in the producer's S3 — no copying

**Important:** Cross-account sharing requires:
- Lake Formation permissions (not IAM-only mode)
- If sharing outside AWS Organization → must use **AWS RAM**
- Consumer account must accept the RAM share (unless auto-accept is enabled for org)

---

### Integration with Analytics Services

| Service | How It Integrates with Lake Formation |
|---------|--------------------------------------|
| **Amazon Athena** | Queries respect Lake Formation permissions automatically (column/row filters applied) |
| **Amazon Redshift Spectrum** | External schema queries respect Lake Formation permissions |
| **AWS Glue ETL** | Jobs use Lake Formation credentials vending for S3 access |
| **Amazon EMR** | Runtime role integration — Lake Formation provides temporary credentials to Spark/Hive |
| **Amazon QuickSight** | Respects Lake Formation permissions when querying via Athena |
| **AWS Glue DataBrew** | Respects Lake Formation column-level permissions |

**Credential vending:** Lake Formation issues temporary, scoped credentials to analytics engines → engines don't need broad S3/Glue permissions. This is how row/column-level security is enforced even for engines like EMR that could otherwise bypass restrictions.

---

### Key Limits & Quotas

| Resource | Default Limit |
|----------|---------------|
| Databases per catalog | 10,000 |
| Tables per database | 200,000 |
| LF-Tags per account | 50 |
| Values per LF-Tag key | 1,000 |
| Data filters per table | 100 |
| Permissions per principal per resource | 100 |
| Registered S3 locations | 10,000 |
| Concurrent blueprint workflows | 25 |
| Cross-account grants per table | 100 |

---

## 2. Design Patterns & Best Practices

### When to Use Lake Formation

**Use Lake Formation when:**
- You have a centralized data lake on S3 queried by multiple teams/services
- Need fine-grained access control (column-level, row-level, cell-level)
- Multiple AWS accounts need to share data without copying
- You want tag-based access control at scale (hundreds of tables, dozens of teams)
- Need to replace complex IAM + S3 + Glue policy combinations
- Need audit trail of who accessed what data
- Building a governed data mesh across multiple accounts

**Don't use Lake Formation when:**
- Simple S3 access with a few IAM policies suffices
- You only have one team and one analytics service
- Your data is not in S3 / Glue Data Catalog (e.g., pure DynamoDB workloads)
- You need real-time streaming access control (Lake Formation is for batch/analytics)

### Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|--------------|-------------|-----|
| Leaving `IAMAllowedPrincipals` with `Super` | Lake Formation permissions are bypassed entirely | Remove this grant to enforce LF permissions |
| Managing S3 policies + LF permissions together | Conflicting/confusing dual permission model | Use Lake Formation exclusively for registered paths |
| One LF-Tag for everything | Doesn't scale; too coarse | Design a taxonomy: sensitivity, department, domain, environment |
| Granting `Super` broadly | Defeats the purpose of fine-grained control | Use specific permissions (SELECT, DESCRIBE, etc.) |
| Not using cross-account sharing | Copying data between accounts | Use LF cross-account grants + resource links |
| Sharing raw data without filters | Exposes PII/sensitive data | Apply data filters (row/column-level) per consumer |

### Data Lake Architecture Pattern (Multi-Layer)

```
[Raw Zone (Bronze)]       → Landing area, raw ingested data
        ↓ (Glue ETL)
[Curated Zone (Silver)]   → Cleaned, deduplicated, standardized
        ↓ (Glue ETL)
[Analytics Zone (Gold)]   → Aggregated, business-ready datasets
```

**Lake Formation permissions at each layer:**
- **Raw:** Only data engineers have access
- **Curated:** Data analysts get `SELECT` with row/column filters
- **Analytics:** Business users get broad `SELECT` access (pre-filtered data)

### Data Mesh Pattern with Lake Formation

```
[Domain Account A] → Publishes tables via LF cross-account sharing
[Domain Account B] → Publishes tables via LF cross-account sharing
[Domain Account C] → Publishes tables via LF cross-account sharing
        ↓ (Resource Links)
[Central Governance Account] → LF-Tags, permissions, audit
        ↓ (Cross-account sharing)
[Consumer Account 1] → Queries via Athena/Redshift
[Consumer Account 2] → Queries via QuickSight
```

- Each domain team owns and publishes their data products
- Central governance account manages LF-Tags and cross-account permissions
- Consumer accounts create resource links and query without data copying

### Well-Architected Alignment

| Pillar | Guidance |
|--------|----------|
| **Reliability** | Lake Formation is Regional; replicate Data Catalog for DR; S3 replication for data durability |
| **Security** | LF-Tags for ABAC; data filters for row/column security; encrypt S3 with KMS; audit with CloudTrail |
| **Cost** | No charge for Lake Formation itself (pay for underlying S3, Glue, Athena); avoid data copies across accounts |
| **Performance** | Partitioned tables; columnar formats (Parquet/ORC); Athena with partition pruning; Glue crawlers for schema |
| **Operational Excellence** | Centralize governance; use LF-Tags to reduce permission management toil; blueprints for consistent ingestion |
| **Sustainability** | Lifecycle policies on S3; delete stale data; governed tables with compaction reduce storage waste |

---

## 3. Security & Compliance

### Permission Enforcement Architecture

```
[User/Role] → [IAM: basic Lake Formation API access]
                     ↓
           [Lake Formation Permission Check]
                     ↓
           [Does principal have SELECT on table/columns/rows?]
                     ↓ (Yes)
           [Lake Formation issues scoped S3 credentials]
                     ↓
           [Analytics engine reads only permitted data from S3]
```

**Critical concept:** Lake Formation sits BETWEEN IAM and the data. Even if an IAM policy allows `s3:GetObject` on the bucket, if Lake Formation permissions deny access to a table, the query fails (assuming `IAMAllowedPrincipals` is removed).

### IAM Requirements

**Minimal IAM needed (with Lake Formation managing data access):**
- `lakeformation:GetDataAccess` — allows the principal to request credentials from Lake Formation
- Basic Glue/Athena permissions to run queries (e.g., `athena:StartQueryExecution`)
- `glue:GetTable`, `glue:GetDatabase` — to read catalog metadata (or Lake Formation `DESCRIBE` grant)
- **NO** `s3:GetObject` needed on the data lake bucket (Lake Formation handles this via credential vending)

### Encryption

| Layer | How |
|-------|-----|
| S3 data at rest | SSE-S3, SSE-KMS, or SSE-C (KMS recommended for key management) |
| Data in transit | HTTPS/TLS for all API calls; Athena/Redshift use encrypted connections |
| Glue Data Catalog | Encrypted with KMS (account-level setting) |
| Cross-account sharing | Data stays encrypted in source account; consumer gets scoped read access |

**KMS + Lake Formation:**
- If S3 data is encrypted with a KMS key, the Lake Formation service role must have `kms:Decrypt` permission on that key
- For cross-account sharing with KMS-encrypted data, the KMS key policy must allow the consumer account's Lake Formation service role to decrypt

### Auditing & Monitoring

| What to Audit | Service |
|---------------|---------|
| Permission grants/revokes | CloudTrail (Lake Formation API calls) |
| Data access (who queried what) | CloudTrail data events for S3 + Athena query logs |
| Permission changes over time | AWS Config (tracks Lake Formation resource configurations) |
| Failed access attempts | CloudTrail (denied API calls) |

### Compliance Patterns

- **GDPR / Data Residency:** Keep data in specific Regions; use row-level security to filter by geography
- **PCI DSS:** Column-level security to mask cardholder data columns from non-authorized analysts
- **HIPAA:** Cell-level security (row + column filters) to restrict PHI access by role
- **SOX:** Audit trail via CloudTrail; separation of duties (Data Lake Admin ≠ Data Analyst)
- **Data Classification:** LF-Tags for sensitivity levels → automated enforcement

### Cross-Account Security

- **Within Organization:** Direct grants (no RAM required in newer Lake Formation versions with org-level sharing)
- **Outside Organization:** Must use AWS RAM; consumer explicitly accepts the share
- **Permissions boundary:** Consumer can only grant permissions UP TO what was shared — can't escalate
- **Revocation:** Producer revokes → consumer immediately loses access (no data to delete since no copy)

---

## 4. Cost Optimization

### Lake Formation Pricing

**Lake Formation itself has NO additional charge.** You pay for:

| Component | Pricing |
|-----------|---------|
| S3 storage | Standard S3 pricing (storage classes apply) |
| Glue Data Catalog | Free for first 1M objects; $1/100K objects/month after |
| Glue crawlers | $0.44 per DPU-hour |
| Glue ETL jobs | $0.44 per DPU-hour |
| Athena queries | $5 per TB scanned |
| Governed table transactions | Per-request pricing for ACID operations |
| Cross-account data sharing | No additional charge (data not copied) |

### Cost-Saving Strategies

| Strategy | Savings |
|----------|---------|
| **Cross-account sharing (not copying)** | Eliminate duplicate S3 storage costs |
| **Columnar formats (Parquet/ORC)** | Reduce Athena scan costs by 30-90% |
| **Partitioning** | Athena only scans relevant partitions → lower query costs |
| **Compression (Snappy, GZIP)** | Smaller files = less S3 storage + less Athena scan |
| **S3 Intelligent-Tiering on lake data** | Automatically move infrequently accessed data to cheaper tiers |
| **Lifecycle policies** | Archive/delete old raw data that's already been curated |
| **Athena workgroups with query limits** | Prevent runaway queries from scanning entire lake |
| **Glue job bookmarks** | Avoid re-processing already-processed data (incremental ETL) |

### Cost Traps

- **Unpartitioned tables queried by Athena:** Full table scan every query → expensive at scale
- **Too many Glue crawlers running frequently:** Crawlers charge per DPU-hour; schedule appropriately
- **Governed table transactions on high-volume data:** Per-request costs can add up
- **Storing multiple copies across accounts:** Use cross-account sharing instead of S3 replication for analytics
- **Not compacting small files:** Many small files = slow queries + more Glue overhead

---

## 5. High Availability, Disaster Recovery & Resilience

### Lake Formation HA

- **Regional service:** Operates within a single Region; inherits S3's 11 9s durability
- **Glue Data Catalog:** Managed, highly available within the Region
- **No multi-Region built-in** — you must design cross-Region DR yourself

### DR Strategies

| Strategy | RPO | RTO | Implementation |
|----------|-----|-----|----------------|
| **Backup & Restore** | Hours | Hours | S3 Cross-Region Replication + Glue Data Catalog export (AWS Glue `get-tables` → import in DR Region) |
| **Pilot Light** | Minutes | 30-60 min | S3 CRR + replicated Glue Catalog + Lake Formation permissions re-created via IaC |
| **Warm Standby** | Minutes | Minutes | Active Lake Formation in DR Region with replicated catalog and permissions; analytics services pre-deployed |
| **Active-Active** | Near-zero | Near-zero | Lake Formation in both Regions; S3 bi-directional replication; Glue catalogs synced; Route 53 routes queries to either Region |

### Resilience Considerations

- **S3 versioning:** Enable on data lake buckets for recovery from accidental deletes/overwrites
- **Glue Data Catalog backup:** Use `aws glue get-tables` / `get-databases` to export metadata; re-import in DR Region
- **Lake Formation permissions as code:** Define all LF-Tag assignments and grants in CloudFormation or Terraform → re-deploy in DR Region
- **Cross-Region LF-Tag sync:** Not automatic — must be replicated via IaC or automation

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company has an S3 data lake with 500 tables in the Glue Data Catalog. Different teams need access to different tables and columns. Managing individual IAM policies is becoming unmanageable. What should they use?
**A:** **AWS Lake Formation.** Centralize permissions using grant/revoke at the table/column level. Replace complex IAM + S3 policies with Lake Formation's permission model.

**Q2:** A data governance team needs to ensure that marketing analysts can only see customer records from their own region (e.g., `region = 'EU'`). What Lake Formation feature enables this?
**A:** **Data Filters (row-level security).** Create a filter with expression `region = 'EU'` and grant SELECT with that filter to the marketing analyst role.

**Q3:** A company has 200 tables and 15 teams. Each table has a classification (public, internal, confidential). How should they manage permissions efficiently?
**A:** **LF-Tags.** Create an LF-Tag `classification = [public, internal, confidential]`. Tag each table. Grant permissions based on tag expressions (e.g., "Team A gets SELECT where classification IN (public, internal)").

**Q4:** Account A has a data lake. Account B needs to query the data without copying it. How?
**A:** **Lake Formation cross-account sharing.** Account A grants SELECT to Account B. Account B creates a **resource link** in their Glue Data Catalog and queries via Athena.

**Q5:** What is the `IAMAllowedPrincipals` group in Lake Formation?
**A:** A backward-compatibility mechanism. It grants `Super` permission on all Catalog resources to any IAM principal, effectively bypassing Lake Formation permissions. **Remove it** to enforce Lake Formation's fine-grained controls.

**Q6:** A company needs to ingest data from an RDS MySQL database into their S3 data lake on a daily schedule. Which Lake Formation feature automates this?
**A:** **Lake Formation Blueprints** (Database blueprint). It creates a Glue workflow that extracts from JDBC sources (RDS) on a schedule — full or incremental loads.

**Q7:** What is a resource link in Lake Formation?
**A:** A pointer in the consumer account's Glue Data Catalog that references a shared table/database in another account. Queries against the resource link are served from the source account's data.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company enabled Lake Formation but their existing Athena queries still work without any Lake Formation grants. Why?

**A:** The `IAMAllowedPrincipals` group still has `Super` permission on all resources. This is the default after enabling Lake Formation for backward compatibility. Lake Formation permissions are effectively bypassed until this grant is removed.

*Why tricky:* Candidates may think Lake Formation permissions are active immediately upon enablement. They aren't — you must remove the legacy grants.

**Keywords:** "just enabled Lake Formation" + "still works without grants" → `IAMAllowedPrincipals` not removed

---

**Q2:** A company uses Lake Formation for access control. A data engineer has `SELECT` and the grant option on table `sales`. They grant `SELECT` on `sales` to an analyst. Later, the data lake admin revokes the engineer's `SELECT` permission. Can the analyst still query the table?

**A:** **No.** When a grantor's permission is revoked, all downstream grants made by that grantor are **cascadingly revoked**. The analyst loses access because their grant was derived from the engineer's permission.

*Why tricky:* This is classic cascading revoke behavior — same as in relational databases. Candidates unfamiliar with RBAC cascade may think the analyst retains access.

---

**Q3:** A healthcare company needs analysts to see patient visit data but NOT the `ssn` and `diagnosis` columns, and only for patients in their assigned state. Which Lake Formation feature(s) do they need?

**A:** **Cell-level security** — a combination of:
1. **Column-level security:** Exclude `ssn` and `diagnosis` columns from the grant
2. **Row-level security:** Data filter with expression `state = 'assigned_state'`

Create a data filter with both the row filter and column inclusion/exclusion, then grant SELECT with that filter.

*Why tricky:* Candidates might pick only row-level OR column-level. Cell-level security requires BOTH applied together.

---

**Q4:** Account A shares a table with Account B via Lake Formation cross-account grants. The S3 data is encrypted with a KMS CMK. Account B users get "Access Denied" when querying. What's missing?

**A:** The **KMS key policy** in Account A must allow Account B's Lake Formation service role (or the specific principals) to use `kms:Decrypt`. Lake Formation grants table-level access, but KMS encryption requires separate key policy permissions.

*Why tricky:* Lake Formation cross-account grants handle S3 and Catalog access but NOT KMS. Encryption keys need separate cross-account configuration.

**Keywords:** "cross-account" + "KMS encrypted" + "Access Denied" → KMS key policy missing cross-account decrypt

---

**Q5:** A company has 1,000 tables across 50 databases. They're adding a new data science team that should have SELECT access to all tables tagged as `domain = customer_analytics`. New tables are added weekly. How should they implement this with MINIMUM ongoing management?

**A:** Use **LF-Tags**. Tag relevant tables with `domain = customer_analytics`. Grant the data science role SELECT where `domain = customer_analytics`. **New tables tagged with this LF-Tag automatically inherit the permission** — no manual grants needed.

*Why wrong alternatives:*
- Individual table grants: 1,000+ grants, must manually add for each new table — high ongoing effort
- IAM policies with resource ARNs: Doesn't scale, must update for each new table
- S3 bucket policies: Too coarse (bucket-level, not table/column-level)

**Keywords:** "1000 tables" + "new tables added" + "minimum ongoing management" → LF-Tags

---

**Q6:** A company wants to share their data lake with a partner company in a different AWS Organization. They've set up Lake Formation cross-account grants, but the partner can't see the shared resources. What's missing?

**A:** Cross-account sharing **outside** your AWS Organization requires **AWS RAM (Resource Access Manager)**. The sharing account must create a RAM resource share containing the Lake Formation resources (databases/tables). The external account must **accept** the RAM invitation.

*Why tricky:* Within the same Organization, direct cross-account grants work. Outside the org, RAM is required — this is a common exam distinction.

**Keywords:** "different Organization" / "external account" + "cross-account sharing" → AWS RAM required

---

**Q7:** A company uses Lake Formation with LF-Tags for access control. They register an S3 location and create a table, but forget to assign an LF-Tag to the new table. A role that has tag-based grants (`sensitivity = public`) tries to query it. What happens?

**A:** **Access Denied.** If a table has NO LF-Tags assigned, it doesn't match any tag-based permission expression. The role's grant only applies to resources with `sensitivity = public` tag explicitly assigned. Untagged resources are implicitly denied.

*Why tricky:* Candidates might think untagged = accessible by everyone. The opposite is true — no tag match = no permission (deny by default).

---

### Common Exam Traps & Pitfalls

1. **`IAMAllowedPrincipals` must be removed** — Lake Formation doesn't enforce until you remove this legacy grant. The exam loves testing this.

2. **Lake Formation ≠ replacement for IAM** — Users still need basic IAM permissions to call Lake Formation/Glue/Athena APIs. Lake Formation controls DATA access, not API access.

3. **`DATA_LOCATION_ACCESS` is separate** — Even with `CREATE_TABLE` permission, you can't create a table pointing to an S3 location unless you have `DATA_LOCATION_ACCESS` for that path.

4. **Cross-account outside org = RAM** — Direct grants work within an Organization. External accounts need AWS RAM.

5. **KMS cross-account is separate** — Lake Formation grants don't extend to KMS key policies. Configure separately.

6. **LF-Tags are NOT S3 object tags** — They're applied to Glue Data Catalog resources (databases, tables, columns), not to S3 objects directly.

7. **Row-level security is per-grant** — Different principals can have different row filters on the same table. Filters are applied transparently during query.

8. **Cascading revoke** — Revoking a permission cascades to all downstream grants made by that principal.

9. **Resource links ≠ copies** — A resource link is just a pointer. Data stays in the source account. No duplicate storage.

10. **Lake Formation is free** — No charge for Lake Formation itself. Costs come from underlying services (S3, Glue, Athena).

---

## 7. Cheat Sheet

### Must-Know Facts

- **Lake Formation** = centralized permissions for S3 data lakes (grant/revoke, like a database)
- **Sits on top of Glue Data Catalog** — same databases/tables, but with LF permissions layer
- **Remove `IAMAllowedPrincipals`** to activate Lake Formation permissions (exam favorite)
- **LF-Tags** = ABAC for data lake at scale; auto-applies to new resources matching tags
- **Data Filters** = row-level, column-level, or cell-level (both) security
- **Cross-account** = grants within org; RAM required outside org; resource links in consumer
- **No data copying** in cross-account sharing — data stays in source
- **Blueprints** = pre-built ingestion workflows (Database, Log)
- **Governed Tables** = ACID transactions on S3 data
- **Credential vending** = Lake Formation issues scoped temporary credentials to analytics engines
- **Free service** — pay only for S3, Glue, Athena, etc.
- **KMS cross-account** = requires separate key policy configuration
- **Cascading revoke** = revoking grantor's permission revokes all their downstream grants

### Decision Flowchart

```
"Fine-grained data lake permissions" OR "column/row-level security on S3"
  → Lake Formation

"Manage access to 100s of tables at scale" + "minimum ongoing effort"
  → LF-Tags (tag-based access control)

"Share data across accounts without copying"
  → Lake Formation cross-account sharing

"Cross-account outside the organization"
  → Lake Formation + AWS RAM

"Ingest from RDS to S3 data lake"
  → Lake Formation Blueprint (Database)

"Replace complex IAM + S3 + Glue policies"
  → Lake Formation (centralized model)

"ACID transactions on S3 data lake"
  → Lake Formation Governed Tables (or Iceberg tables)

"Audit who accessed what data in the lake"
  → CloudTrail + Lake Formation API logs

"Different teams see different rows of same table"
  → Data Filters (row-level security)
```

### Key Differentiators

| Signal in Question | Answer |
|---|---|
| "Centralized data governance" + "data lake" | Lake Formation |
| "Column/row-level access control on S3" | Lake Formation Data Filters |
| "Tag-based access" + "scales to 1000s of tables" | LF-Tags |
| "Share data lake across accounts" | Lake Formation cross-account + resource links |
| "External organization" + "data sharing" | Lake Formation + AWS RAM |
| "Enable Lake Formation" + "existing queries still work" | `IAMAllowedPrincipals` not removed |
| "Ingest RDS → S3 data lake" (scheduled) | Lake Formation Blueprint |
| "KMS" + "cross-account" + "access denied" | KMS key policy not updated |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company has 300 tables. They want to transition from IAM-only access to Lake Formation permissions. They cannot have any service disruption. What is the safest migration approach?

**A:** Phased approach:
1. Enable Lake Formation (creates `IAMAllowedPrincipals` Super grants by default — no disruption)
2. Identify all principals and their current access patterns (audit with CloudTrail)
3. Create equivalent Lake Formation grants for each principal
4. **Test** by removing `IAMAllowedPrincipals` from ONE database first (not all at once)
5. Validate that all queries still work for that database
6. Gradually remove `IAMAllowedPrincipals` from remaining databases
7. Remove broad S3/Glue IAM policies once LF permissions are enforced everywhere

*Why tricky:* "No disruption" means you can't just remove `IAMAllowedPrincipals` globally at once. The phased approach is critical.

---

**Q2:** A financial services company has `accounts` table with columns: `account_id`, `name`, `balance`, `ssn`, `credit_score`. Compliance requires: (a) Customer service sees all columns but only their assigned region, (b) Risk team sees `account_id`, `balance`, `credit_score` for all regions, (c) Marketing sees only `account_id`, `name` for all regions. How to implement?

**A:** Create three **data filters:**
1. **Customer Service:** Row filter: `region = 'assigned_region'`, Columns: ALL included
2. **Risk Team:** Row filter: none (all rows), Columns: include only `account_id`, `balance`, `credit_score`
3. **Marketing:** Row filter: none (all rows), Columns: include only `account_id`, `name`

Grant SELECT with the appropriate data filter to each role.

---

**Q3:** An organization has a central data lake account and 10 consumer accounts. They use LF-Tags for access control. A new `fraud_detection` table is created in the data lake but the fraud team in a consumer account can't see it, despite having a grant based on `domain = fraud`. What's likely wrong?

**A:** The new `fraud_detection` table was **not tagged** with `domain = fraud`. LF-Tag-based grants only apply to resources that have the matching tag explicitly assigned. The table must be tagged after creation (or via automation).

*Fix:* Assign `domain = fraud` LF-Tag to the table. Implement automation (Lambda trigger on Glue Data Catalog events) to auto-tag new tables based on naming convention or database.

---

**Q4:** A company uses Lake Formation for governance. Their EMR cluster with Spark jobs bypasses Lake Formation permissions and reads S3 data directly. How should they fix this?

**A:** Enable **Lake Formation integration with EMR** using **runtime roles**. This forces EMR to request credentials from Lake Formation (credential vending). Without this, EMR uses its EC2 instance profile which has direct S3 access, bypassing Lake Formation.

Steps:
1. Configure EMR with Lake Formation session-level credentials (EMR Runtime Role)
2. Remove direct S3 permissions from the EMR instance profile
3. Grant Lake Formation permissions to the EMR execution role

*Why tricky:* By default, EMR CAN bypass Lake Formation if it has S3 access via IAM. You must explicitly configure the integration.

---

**Q5:** A company uses Lake Formation cross-account sharing. Account A shares table `orders` with Account B. An analyst in Account B can query the table, but when they try to grant access to another analyst in Account B, they get "Access Denied." Why?

**A:** Account A granted SELECT **without the grantable (WITH GRANT OPTION) permission**. For the first analyst to re-grant, Account A must grant with `GrantablPermissions = [SELECT]`. Without this, the consumer account can use the permission but cannot delegate it.

---

**Q6:** A company's data lake has Parquet files partitioned by `year/month/day`. They enable Lake Formation and create a row-level filter: `year = '2024'`. When analysts query with `WHERE year = '2023'`, Athena returns no results. Is partition pruning still effective?

**A:** **Yes.** Lake Formation row-level filters work WITH Athena partition pruning. The filter `year = '2024'` combined with the query's `WHERE year = '2023'` means no partitions match → Athena scans zero data. Partition pruning is still applied before/during the Lake Formation filter enforcement.

*Why tricky:* Candidates might worry that row-level security prevents partition pruning optimization. It doesn't — they complement each other.

---

**Q7:** A company uses Lake Formation to manage their data lake. They have a CloudFormation template that creates Glue databases and tables. After deployment, they notice Lake Formation permissions don't apply and the Glue Data Catalog has `IAMAllowedPrincipals` on the new databases. How should they fix this in their IaC?

**A:** In CloudFormation, use `AWS::LakeFormation::Permissions` and `AWS::LakeFormation::DataLakeSettings` resources:
1. Set `DataLakeSettings` to remove `IAMAllowedPrincipals` as a data lake admin creating database creator
2. Use `AWS::LakeFormation::PrincipalPermissions` to grant specific permissions
3. Set `CreateDatabaseDefaultPermissions` and `CreateTableDefaultPermissions` to empty (removes default `IAMAllowedPrincipals` grants on new resources)

*Key:* The `DataLakeSettings` must explicitly override the defaults that grant `IAMAllowedPrincipals`.

---

**Q8:** A retail company has data in three accounts (NA, EU, APAC). They want a centralized analytics account to query all three without data movement. NA and EU are in the same Organization; APAC is in a separate Organization (acquired company). What's the architecture?

**A:**
- **NA & EU accounts:** Use direct Lake Formation cross-account grants to the analytics account (same Organization — no RAM needed)
- **APAC account:** Use **AWS RAM** to share Lake Formation resources to the analytics account (external Organization)
- **Analytics account:** Create **resource links** for all shared tables. Athena queries reference resource links → data stays in source accounts.

*Why tricky:* Mixed approach — same org vs. different org requires different sharing mechanisms.

---

**Q9:** A data team registered an S3 location `/data-lake/raw/` with Lake Formation. A data engineer has `CREATE_TABLE` permission on the `raw_db` database. They try to create a table pointing to `/data-lake/raw/customers/` but get "Access Denied." They have no IAM issues. What's missing?

**A:** The engineer is missing **`DATA_LOCATION_ACCESS`** permission for the S3 path `/data-lake/raw/customers/` (or parent path). `CREATE_TABLE` allows creating the metadata entry, but `DATA_LOCATION_ACCESS` is required to associate a table with a specific registered S3 location.

*Why tricky:* Candidates often forget that `CREATE_TABLE` and `DATA_LOCATION_ACCESS` are separate permissions. You need BOTH to create a table pointing to an S3 path.

---

**Q10:** A company uses governed tables in Lake Formation for their slowly changing dimension (SCD) tables. They run concurrent Glue jobs that update and read the same governed table. Without governed tables, they experienced inconsistent reads (reading partially updated data). After enabling governed tables, queries are slower. Why, and is the trade-off acceptable?

**A:** Governed tables use **ACID transactions** which add overhead:
- **Locking/versioning:** Concurrent access requires coordination (optimistic concurrency control)
- **Time travel snapshots:** Maintaining multiple versions adds storage I/O
- **Compaction needed:** Small transactions create small files that slow reads until compacted

The trade-off is acceptable because **data consistency** (no dirty reads) is more important than raw speed for SCD tables. Optimize by: scheduling compaction, batching small writes, and ensuring proper transaction isolation levels.

---

### Compare: Lake Formation vs. IAM vs. S3 Policies vs. Glue Policies

| Dimension | Lake Formation | IAM Policies | S3 Bucket Policies | Glue Resource Policies |
|-----------|---------------|-------------|-------------------|----------------------|
| **Granularity** | Column, row, cell, table, database | Resource ARN level | Bucket/prefix level | Database/table |
| **Scalability** | LF-Tags (ABAC) — scales to 1000s | Per-resource policies — doesn't scale | Per-bucket — limited | Per-catalog — limited |
| **Cross-account** | Built-in (grants + resource links) | Assume role | Bucket policy principals | Resource policy |
| **Row-level security** | Yes (data filters) | No | No | No |
| **Column-level security** | Yes | No | No | No |
| **Audit** | CloudTrail + centralized view | CloudTrail | S3 access logs / CloudTrail | CloudTrail |
| **Management** | Centralized console / API | Distributed across policies | Per-bucket | Per-catalog |
| **Exam preference** | "Data lake governance" | "API access control" | "Public access" / "bucket-level" | Legacy (before Lake Formation) |

### When Does the Exam Expect Lake Formation vs. Alternatives?

**Pick Lake Formation when:**
- "Data lake" + "fine-grained access control"
- "Column-level" or "row-level" security on S3 data
- "Multiple teams" + "centralized governance"
- "Cross-account data sharing without copying"
- "Tag-based access control" for analytics data
- "Replace complex IAM + S3 + Glue policies"
- "Data mesh" or "data governance" at scale

**Pick IAM policies when:**
- Controlling AWS API access (who can call what service)
- Simple resource-level permissions (e.g., "this role can read this S3 prefix")
- No need for column/row-level security
- Single-account, single-team scenarios

**Pick S3 Bucket Policies when:**
- Need to control access at the bucket/prefix level
- Public access configuration
- Cross-account S3 access for non-analytics use cases
- Simple "who can read/write this bucket" scenarios

**Pick Glue Resource Policies when:**
- Cross-account Glue Data Catalog sharing WITHOUT Lake Formation
- Legacy setups before Lake Formation adoption

### All "Gotcha" Differences

| Gotcha | Detail |
|--------|--------|
| LF-Tags ≠ S3 tags ≠ resource tags | LF-Tags are unique to Lake Formation; applied in Data Catalog, NOT on S3 objects or as AWS resource tags |
| `IAMAllowedPrincipals` default | Must be explicitly removed per database/table to activate LF permissions |
| `DATA_LOCATION_ACCESS` ≠ `SELECT` | Creating tables requires location access; querying requires SELECT — they're independent |
| Cross-account ≠ data copy | Resource links point to source; no data movement |
| RAM required outside org | Direct grants only work within the same AWS Organization |
| Governed tables ≠ regular tables | Different performance characteristics; require compaction maintenance |
| Lake Formation is free | No charge for LF itself — costs are S3, Glue, Athena |
| LF doesn't replace IAM for APIs | Users still need IAM to call Athena/Glue APIs; LF controls data access only |
| EMR bypass risk | EMR can bypass LF if instance profile has direct S3 access — must use runtime roles |
| Cascading revoke | Revoking a grant cascades to all downstream grants from that grantor |

### Decision Tree for Data Lake Security

```
START: Need to control access to S3 data lake?
├── Simple use case (one team, one bucket, no column/row restrictions)?
│   └── IAM policies + S3 bucket policies (sufficient)
├── Multiple teams, need column/row-level security?
│   └── AWS Lake Formation
│       ├── Few tables, stable? → Named resource grants
│       └── Many tables, dynamic? → LF-Tags
├── Cross-account sharing needed?
│   ├── Same Organization? → LF cross-account grants (direct)
│   └── Different Organization? → LF + AWS RAM
├── Need to audit all data access?
│   └── Lake Formation + CloudTrail data events
└── Need ACID transactions on S3?
    └── Lake Formation Governed Tables (or Apache Iceberg via Glue)
```

---

*End of study notes for Lake Formation (Permissions, Data Lake)*
