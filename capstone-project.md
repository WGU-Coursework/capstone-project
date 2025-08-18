# Part 1 — Proposal

## A. Business problem or opportunity

**Context.** A mid-size healthcare network runs daily clinical and operational reports on a small on-prem SQL server. Jobs overrun, extracts arrive late, and analysts wait hours for ad-hoc queries. Stale data slows staffing decisions and delays quality dashboards.

**Drivers.**

1. **Speed and scale.** Current hardware can’t keep up with growing data from EHR logs, devices, and claims files.
2. **Cost pressure.** Capex refresh is due; leadership prefers pay-as-you-go.
3. **Compliance.** The org wants stronger controls and a clear HIPAA posture without building everything from scratch. (Microsoft documents Azure’s HIPAA program and BAA for in-scope services. ([Microsoft Learn][1]))

## B. Proposed cloud solution and evaluation framework

**Solution (summary).** Build a **lakehouse on Azure**:

* **Azure Data Lake Storage Gen2 (ADLS Gen2)** as the central data lake (hierarchical namespace for file-like directories). ([Microsoft Learn][2])
* **Azure Synapse Analytics (serverless SQL pool)** for ad-hoc SQL over lake files; no clusters to manage. ([Microsoft Learn][3])
* **Private Endpoints (Azure Private Link)** to keep traffic off the public internet. ([Microsoft Learn][4])
* **Key Vault–backed customer-managed keys (CMK)** for encryption at rest. ([Microsoft Learn][5])
* **Immutable and soft-delete policies** on storage for ransomware/accidental delete guardrails. ([Microsoft Learn][6])
* **Azure Monitor + Log Analytics** for health, SLAs, and cost visibility. ([Microsoft Learn][7])
* **Microsoft Purview** for cataloging and data governance. ([Microsoft Learn][8])

**Evaluation framework (how we’ll judge success).**

* **Data freshness:** 95% of daily feeds land by 6 a.m.; <15-minute lag for streaming sources. Measured via Synapse job logs and Azure Monitor alerts. ([Microsoft Learn][7])
* **Analyst experience:** Median ad-hoc query finishes in <60 seconds on serverless SQL pool. ([Microsoft Learn][3])
* **Cost:** Monthly spend within ±10% of estimate from the Azure Pricing Calculator; variances explained in Cost Management. ([Microsoft Azure][9], [Microsoft Learn][10])
* **Security posture:** No “public access enabled” findings in Defender for Cloud; CMK in place; private endpoints enforced. ([Microsoft Learn][11])
* **Compliance support:** Services in scope for HIPAA with BAA executed. ([Microsoft Learn][1])

### B1. Security fit for the problem

* **Network isolation.** Storage and Synapse use **Private Endpoints** so data paths stay on Microsoft’s backbone, not the public internet. ([Microsoft Learn][4])
* **Encryption at rest with our keys.** ADLS Gen2 uses **CMK** from **Azure Key Vault** (or Managed HSM) with soft-delete and purge protection on the vault. ([Microsoft Learn][5], [Azure Documentation][12])
* **Identity over keys.** Access uses Entra ID, **RBAC**, and **managed identities** for services. No embedded secrets. ([Microsoft Learn][13])
* **Tamper-proof backups.** **Immutable WORM** policies and **soft delete** reduce damage from ransomware or mistakes. ([Microsoft Learn][6])
* **Defender for Cloud** recommendations track misconfigurations (e.g., public blob access). ([Microsoft Learn][14])
* **Regulatory support.** Azure offers HIPAA documentation and a **BAA** for in-scope services. ([Microsoft Learn][1])

## C. Stakeholders and impacts

* **Clinicians and operations leaders.** Need timely, accurate dashboards to staff units and manage throughput. Impact: faster refresh, fewer manual extracts.
* **Data analysts/BI team.** Need self-service SQL on raw files and curated tables. Impact: serverless SQL speeds ad-hoc work without cluster ops. ([Microsoft Learn][15])
* **IT/Sec/Compliance.** Need enforceable guardrails and audit trails. Impact: RBAC, Private Link, CMK, Immutable storage, and Defender posture views. ([Microsoft Learn][13])
* **Finance.** Needs spend predictability. Impact: monthly reporting in Cost Management with budgets and alerts. ([Microsoft Learn][10])

## D. Alternatives compared

1. **Keep it on-prem (bigger SQL server / Hadoop-style cluster).**

   * **Pros:** Full local control; predictable fixed cost.
   * **Cons:** Slow to scale; high capex; limited elasticity; ongoing patching and DR complexity.
2. **Build on AWS (S3 + Athena + Glue + PrivateLink).**

   * **Pros:** Similar serverless query over lake data (Athena) and centralized catalog (Glue). Mature security, default at-rest encryption on S3. ([AWS Documentation][16])
   * **Cons:** Team skills today skew Azure; governance already standardized on Entra and Purview; cross-cloud increases integration work.
     **Why Azure now.** It best fits existing identity, budget controls, and Microsoft stack in this org while meeting HIPAA program needs. ([Microsoft Learn][1])

## E. Two notable risks (likelihood × impact)

1. **Misconfigured public access on storage.** Someone enables anonymous reads on a container. **Likelihood:** medium without policy guardrails. **Impact:** high (PHI exposure). **Affected:** ADLS Gen2. **Mitigation:** set account-level “Allow Blob anonymous access” = Disabled; enforce with Policy; use Private Endpoints only. ([Microsoft Learn][17])
2. **Credential compromise / over-privileged access.** A leaked key or broad role grants data exfiltration. **Likelihood:** medium. **Impact:** high. **Affected:** Synapse workspace, storage, Key Vault. **Mitigation:** use **managed identities**, least-privilege **RBAC**, monitor with Defender and Log Analytics; avoid shared keys. ([Microsoft Learn][18])

---

# Part 2 — Implementation Plan

## A. Problem summary

Reporting is slow and fragile on aging on-prem gear. We need faster queries, cheaper scale, and stronger, easier-to-audit controls for PHI.

## B. How two core services interact

**ADLS Gen2 + Synapse serverless SQL.** Data lands as CSV/Parquet in ADLS Gen2 (organized by directories/partitions). Analysts query those files directly through **Synapse serverless** using external tables/OPENROWSET. No clusters run idle; you pay per TB scanned. This pairing keeps storage cheap and compute elastic. ([Microsoft Learn][2])

## C. Security and disaster recovery

* **Access:** Entra ID + **RBAC** + **managed identities** for pipelines and apps. No secrets in code. ([Microsoft Learn][13])
* **Network:** **Private Endpoints** for Storage and Synapse; deny public network access. ([Microsoft Learn][4])
* **Encryption:** Storage encryption with **CMK** in **Key Vault/Managed HSM**. ([Microsoft Learn][5])
* **Data immutability & recovery:** **Immutable WORM** where required, plus **soft delete** and **versioning** to undo accidental deletes. ([Microsoft Learn][6])
* **Backups/DR:** Use **Azure Backup** for supported data sources and **Azure Site Recovery** for critical VMs if any remain. Test failover twice a year. ([Microsoft Learn][19])
* **Monitoring:** **Azure Monitor/Log Analytics** for alerts on failed jobs, cost anomalies, and security recommendations. ([Microsoft Learn][7])

## D. Costs and how we’ll estimate

**Main cost drivers:**

* **Storage:** ADLS Gen2 capacity + transactions + optional immutability retention.
* **Compute:** Serverless SQL (per-TB processed).
* **Data movement / orchestration:** Small if using serverless + light functions; larger if heavy pipelines.
* **Governance/monitoring:** Purview scans; Log Analytics ingestion.
  **Estimation method:** Use the **Azure Pricing Calculator** with monthly data volumes (TB read by serverless, GB stored, expected transactions). Track actuals in **Cost Management** with budgets and alerts; review weekly during rollout. ([Microsoft Azure][9], [Microsoft Learn][10])
  **Note.** Prices vary by region and tier; we’ll keep a 15% contingency and adjust after the first month’s Cost Management report. ([Microsoft Azure][20])

## E. Methodology and milestones (Agile)

Two-week sprints with clear gates.

1. **Landing zone & guardrails** (RBAC, networking, policies, Key Vault).
2. **Ingestion MVP** (one high-value feed into ADLS Gen2; serverless view).
3. **Governance** (Purview catalog, data domains, sensitivity labels). ([Microsoft Learn][8])
4. **Scale-out ingestion** (top 10 sources).
5. **UAT & training** (analyst playbooks, cost dashboards).
6. **Go-live** (cutover for priority dashboards) with rollback plan.

## F. Testing plan

**Test types and examples:**

* **Functional:** Query parity vs. on-prem reports; row counts and aggregates match.
* **Performance:** Serverless query on 100 GB Parquet returns <60s median. ([Microsoft Learn][3])
* **Security:** Verify **public access disabled**; only managed identities and RBAC roles can read/write; CMK in use; Private Endpoints enforced. ([Microsoft Learn][17])
* **Resilience:** Delete sample data; restore via soft delete/versioning; validate WORM retention where enabled. ([Microsoft Learn][21])
* **DR:** **Azure Backup** restore test and **Site Recovery** test failover runbooks. ([Microsoft Learn][19])
  **Acceptance criteria:** All priority dashboards refresh by 6 a.m.; P1 security findings = 0; budget burn within target; UAT sign-off from BI, Security, and Operations.

## G. Post-implementation plan

* **Operate:** Weekly cost and performance reviews in **Cost Management** and **Monitor**; tag resources for showback. ([Microsoft Learn][10])
* **Governance:** Keep Purview catalog current; quarterly access reviews in RBAC. ([Microsoft Learn][22])
* **Security posture:** Remediate new Defender recommendations within SLA; run phishing/credential hygiene campaigns. ([Microsoft Learn][11])
* **DR & restore:** Twice-yearly failover and restore drills; audit immutability policies. ([Microsoft Learn][23])
* **Iteration:** Add new sources, optimize file formats (Parquet), and refine partitioning as volumes grow.

---

## References

**Microsoft**. (2025). *Use private endpoints for Azure Storage*. [https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints) ([Microsoft Learn][4])

**Microsoft**. (2024). *Azure Data Lake Storage: Introduction* (hierarchical namespace). [https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction) ([Microsoft Learn][2])

**Microsoft**. (2024). *Serverless SQL pool — Azure Synapse Analytics (overview)*. [https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/on-demand-workspace-overview](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/on-demand-workspace-overview) ([Microsoft Learn][3])

**Microsoft**. (2023). *Azure Storage encryption for data at rest* (CMK via Key Vault/HSM). [https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption) ([Microsoft Learn][5])

**Microsoft**. (2024). *Immutable storage for Azure Blob Storage (WORM)*. [https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview) ([Microsoft Learn][6])

**Microsoft**. (2025). *Azure Monitor overview* and *Log Analytics overview*. [https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/overview](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/overview) ; [https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview) ([Microsoft Learn][7])

**Microsoft**. (2025). *Data governance with Microsoft Purview*. [https://learn.microsoft.com/en-us/purview/data-governance-overview](https://learn.microsoft.com/en-us/purview/data-governance-overview) ([Microsoft Learn][8])

**Microsoft**. (2023/2025). *HIPAA offering overview* and *Microsoft and HIPAA/HITECH*. [https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-hipaa-us](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-hipaa-us) ; [https://learn.microsoft.com/en-us/compliance/regulatory/offering-hipaa-hitech](https://learn.microsoft.com/en-us/compliance/regulatory/offering-hipaa-hitech) ([Microsoft Learn][1])

**Microsoft**. (2025). *Configure anonymous read access / disallow blob public access*. [https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure) ([Microsoft Learn][17])

**Microsoft**. (2024/2025). *Azure Backup* and *Azure Site Recovery overview*. [https://learn.microsoft.com/en-us/azure/backup/](https://learn.microsoft.com/en-us/azure/backup/) ; [https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview) ([Microsoft Learn][19])

**Microsoft**. (2024–2025). *Azure RBAC overview*; *Managed identities overview*. [https://learn.microsoft.com/en-us/azure/role-based-access-control/overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) ; [https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) ([Microsoft Learn][13])

**Microsoft**. (2024). *Soft delete for blobs*. [https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview) ([Microsoft Learn][21])

**Microsoft**. (2025). *Microsoft Defender for Cloud — data security recommendations*. [https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference-data](https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference-data) ([Microsoft Learn][11])

**Microsoft**. (2025). *Azure Pricing Calculator* and *Cost Management overview*. [https://azure.microsoft.com/en-us/pricing/calculator/](https://azure.microsoft.com/en-us/pricing/calculator/) ; [https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management) ([Microsoft Azure][9], [Microsoft Learn][10])

**Amazon Web Services**. (2024–2025). *Amazon Athena: What it is / Getting started*; *AWS Glue Data Catalog overview*; *S3 server-side encryption defaults*. [https://docs.aws.amazon.com/athena/latest/ug/what-is.html](https://docs.aws.amazon.com/athena/latest/ug/what-is.html) ; [https://docs.aws.amazon.com/athena/](https://docs.aws.amazon.com/athena/) ; [https://docs.aws.amazon.com/prescriptive-guidance/latest/serverless-etl-aws-glue/aws-glue-data-catalog.html](https://docs.aws.amazon.com/prescriptive-guidance/latest/serverless-etl-aws-glue/aws-glue-data-catalog.html) ; [https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html) ([AWS Documentation][16])

**Amazon Web Services**. (n.d.). *AWS PrivateLink overview*. [https://aws.amazon.com/privatelink/](https://aws.amazon.com/privatelink/) ([Amazon Web Services, Inc.][24])

[1]: https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-hipaa-us?utm_source=chatgpt.com "HIPAA - Azure Compliance"
[2]: https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction?utm_source=chatgpt.com "Azure Data Lake Storage Introduction"
[3]: https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/on-demand-workspace-overview?utm_source=chatgpt.com "Serverless SQL pool - Azure Synapse Analytics"
[4]: https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints?utm_source=chatgpt.com "Use private endpoints - Azure Storage"
[5]: https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption?utm_source=chatgpt.com "Azure Storage encryption for data at rest"
[6]: https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview?utm_source=chatgpt.com "Overview of immutable storage for blob data - Azure"
[7]: https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/overview?utm_source=chatgpt.com "Azure Monitor overview"
[8]: https://learn.microsoft.com/en-us/purview/data-governance-overview?utm_source=chatgpt.com "Data governance with Microsoft Purview"
[9]: https://azure.microsoft.com/en-us/pricing/calculator/?utm_source=chatgpt.com "Pricing Calculator"
[10]: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management?utm_source=chatgpt.com "Overview of Cost Management"
[11]: https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference-data?utm_source=chatgpt.com "Reference table for all data security recommendations ..."
[12]: https://docs.azure.cn/en-us/storage/common/customer-managed-keys-configure-existing-account?utm_source=chatgpt.com "Configure customer-managed keys in the same tenant for ..."
[13]: https://learn.microsoft.com/en-us/azure/role-based-access-control/overview?utm_source=chatgpt.com "What is Azure role-based access control (Azure RBAC)?"
[14]: https://learn.microsoft.com/en-us/azure/defender-for-cloud/review-security-recommendations?utm_source=chatgpt.com "Review security recommendations - Microsoft Defender for ..."
[15]: https://learn.microsoft.com/en-us/azure/synapse-analytics/quickstart-serverless-sql-pool?utm_source=chatgpt.com "Quickstart: Use serverless SQL pool - Azure"
[16]: https://docs.aws.amazon.com/athena/latest/ug/what-is.html?utm_source=chatgpt.com "What is Amazon Athena? - Amazon Athena"
[17]: https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure?utm_source=chatgpt.com "Configure anonymous read access for containers and blobs"
[18]: https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview?utm_source=chatgpt.com "Managed identities for Azure resources"
[19]: https://learn.microsoft.com/en-us/azure/backup/?utm_source=chatgpt.com "Azure Backup service documentation"
[20]: https://azure.microsoft.com/en-us/pricing?utm_source=chatgpt.com "Azure Pricing Overview"
[21]: https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview?utm_source=chatgpt.com "Soft delete for blobs - Azure Storage"
[22]: https://learn.microsoft.com/en-us/purview/purview?utm_source=chatgpt.com "Learn about Microsoft Purview"
[23]: https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview?utm_source=chatgpt.com "About Azure Site Recovery"
[24]: https://aws.amazon.com/privatelink/?utm_source=chatgpt.com "AWS PrivateLink - VPC Networking"
