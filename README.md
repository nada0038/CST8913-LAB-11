# Tailwind Traders – Migration Plan
## Akash Nadackanal Vinod

---

# Task 1 – Set Up Tooling for Discovery

To begin the migration process, Tailwind needs the right discovery setup in Azure Migrate. The goal here is to collect accurate data about the on-premises environment before any assessment or planning happens.

## Discovery Approach  
A mix of **agentless** and **agent-based** discovery works best for this environment:

- **Agentless** discovery covers WEB01, WEB02, and APP01 since they’re running on VMware. This gives Azure Migrate enough information about hardware, performance, and basic dependency data.
- **Agent-based** discovery is needed for SQL01. It’s a physical server, and SQL visibility (instances, databases, configuration) requires an agent.

## Number of Appliances  
One Azure Migrate appliance is enough. All servers are in the same datacenter, the workload count is small, and the environment doesn’t need multiple collection points.

## Credentials Needed  
To collect the right data, Tailwind prepares:

- **Software inventory:** Domain or local admin credentials  
- **Dependency mapping:** Admin rights for agent installation  
- **SQL discovery:** SQL sysadmin or Windows authentication with equivalent permissions  

These match the credentials listed in the slide workflow.

## Best Practices While Running Discovery  
Tailwind follows the recommended practices:

1. Keep discovery running for about a month to match the assessment performance window.  
2. Filter out known system noise so dependency mapping stays meaningful.  
3. Check regularly that the appliance is uploading data successfully.

---

# Task 2 – Perform Assessment Planning

After discovery, Tailwind configures the Azure Migrate assessments using the parameters from the slides.

## Assessment Type  
Since this is a live production workload with downtime restrictions, Tailwind runs a **production** assessment.

## Target Region & Performance Window  
- **Region:** Canada Central  
- **Performance history:** 1 month  

This matches the slide guidance and ensures sizing is based on actual usage.

## Sizing Approach  
Tailwind follows the suggested approach from the slides and uses **performance-based sizing**. This avoids relying on oversized on-prem VM specs and leads to more accurate and cost-efficient recommendations.

## Comfort Factor, Pricing, and Licensing  
Tailwind sets:

- **Comfort factor:** 15–20%  
- **Pricing:** Reserved instances for long-running workloads  
- **Licensing:** Azure Hybrid Benefit where applicable  

These are the same parameters identified in the assessment configuration guidance.

---

# Task 3 – Dependency Analysis

Dependency analysis helps confirm how all parts of the application interact. Tailwind uses both the agentless and agent-based data and cleans the results based on the steps in the slides.

## Application Components  
The application consists of:

- WEB01, WEB02 – Web tier  
- APP01 – Application/API tier  
- SQL01 – SQL Server  
- LB01 – Internal load balancer  
- Backup target  
- Firewall rules supporting the environment

## Key Dependencies (Cleaned Over One Month)
1. WEB -> APP (TCP 443/80)  
2. APP -> SQL (TCP 1433)  
3. WEB -> LB01 (health probes)  
4. APP -> external payment endpoint  
5. SQL -> backup storage  
6. WEB -> Active Directory (389/636/88)  
7. APP -> file share (SMB 445)  
8. SQL -> monitoring agents  

## Noise Removed  
Following the slide instructions, Tailwind removes:

- Well-known system processes  
- Non-RFC 1918 public endpoints  
- NTP traffic  
- AV update traffic  
- Vulnerability scanner IPs  

## Business Requirements Collected  
In discussions with the application owner, Tailwind documents:

- **Criticality:** High  
- **Architecture:** Three-tier application including load balancer  
- **Users:** Internal and customer-facing  
- **Data classification:** Confidential  
- **SLA:** Needs to match or exceed current levels  
- **Downtime:** Maximum 1 hour for cutover  
- **Backup needs:** Nightly jobs  
- **Monitoring:** Must integrate into Azure Monitor  
- **Patching:** Monthly  
- **Licensing:** Windows + SQL licensing considerations  
- **Firewall/IP impacts:** Must understand the effect of IP changes  

These align exactly with the dependency analysis data points from the slides.

---

# Task 4 – Validate Assessment Results with Application Owner

Tailwind reviews the Azure Migrate results with the application owner before finalizing the plan.

## Recommended VM Sizes  
Based on the assessment:

- **WEB01/WEB02:** D2s v5  
- **APP01:** D4s v5  
- **SQL01:** Memory-optimized size such as E-series (depending on performance history)

## Components That May Need Changes in Azure  
- LB01 replaced with Azure Load Balancer or Application Gateway  
- SQL instance reviewed for modernization options  
- Backup mechanism moving to Azure Backup  
- Storage recommendations adjusted based on assessment results

## Dependency Validation  
The application owner confirms:

- All necessary flows between WEB -> APP -> SQL are accurate  
- External API dependencies are recorded  
- Authentication, monitoring, and backup flows are properly captured  

## SLA and Downtime Check  
Azure’s SLA meets the application expectations, and the migration plan fits within the 1-hour cutover requirement.

## SQL Migration Options (Reviewed, Not Selected)
Per the slides, Tailwind reviews the available options:

1. **Rehost** (SQL on Azure VM)  
2. **Azure SQL Managed Instance**  
3. **Azure SQL Database**  

Each option is considered, but the slides do not require choosing one.

---

# Task 5 – Migration Plan (Runbook)

The following plan outlines the migration steps Tailwind will follow, integrating results from assessments, dependencies, and discussions with the owner.

## Pre-Migration Tasks  
- Validate all assessment results  
- Confirm the agreed-upon downtime window  
- Take snapshots/backups of all servers  
- Prepare Azure networking (VNets, NSGs, subnets)  
- Deploy Azure Load Balancer  
- Prepare SQL target depending on chosen option later  
- Reduce DNS TTL for fast cutover  
- Confirm credentials and access  

---

## Migration Steps

### **Web Tier (WEB01 & WEB02)**
1. Replicate servers using Azure Migrate  
2. Run test migration  
3. Cut over during maintenance window  
4. Add VMs to Azure Load Balancer  
5. Validate basic site access  

### **Application Tier (APP01)**
1. Replicate and test migrate  
2. Cut over after the web tier is stable  
3. Update application configuration  
4. Validate SQL connectivity  

### **Database Tier (SQL01)**
Depending on the SQL approach:

#### **If rehosted to an Azure VM:**
- Replicate, test migrate, then cut over  
- Validate database integrity and jobs  

#### **If migrated to MI or SQL DB:**
- Use DMA for assessments and migration  
- Migrate schema and data  
- Update connection strings  

---

## DNS Updates  
- Update DNS records for WEB and APP servers  
- Ensure low TTL allows quick propagation  

## Connection String Updates  
- Update APP -> SQL  
- Update WEB -> APP if IP/endpoint changes  

## Load Balancer Adjustments  
- Validate LB probes  
- Confirm routing and backend pool settings  

---

## Post-Migration Checks  
- Full end-to-end functionality  
- SQL performance validation  
- Backup job validation  
- Monitoring and alerting setup  
- Firewall and NSG rule checks  

---

## Back-Out Plan  
- Shut down Azure VMs  
- Reactivate on-prem servers  
- Revert DNS changes  
- Restore snapshots if needed  
- Validate application before reopening to users  

---

# Task 6 – Migration Waves

Tailwind groups the servers into waves based on dependency order and criticality.

## Final waves

| Wave | Servers | Reason |
|------|---------|--------|
| **Wave 1** | WEB01, WEB02 | Stateless, low-risk, confirms networking early |
| **Wave 2** | APP01 | Depends on web tier and easier to validate after Wave 1 |
| **Wave 3** | SQL01 | High-risk, data-centric, requires final validation |


