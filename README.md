# Cloud Governance: Zero-Trust OIM Audit Log & Secrets Management

## 📖 Background Story
The organization relies on Oracle Identity Manager (OIM 12c) as its central identity governance platform. Hosted on Azure Virtual Machines, this system acts as the primary source of truth for downstream enterprise systems responsible for user provisioning and access management. 

As part of its synchronization workflows, OIM generates highly sensitive flat-file audit logs capturing critical identity governance events, including user provisioning, deprovisioning, access modifications, and potential toxic permission combinations. 

Historically, these audit logs were stored locally within the virtual machine’s file system, while application secrets (database connection strings, TNS configurations) were maintained directly within plain-text configuration files. This legacy architecture created significant compliance risks, limited audit visibility for security teams, and introduced potential exposure vectors for sensitive identity data.

## 🏢 Business Scenario
The Governance and Security team has mandated a modernization of identity audit log storage and infrastructure secret management. The organization requires a centralized, highly secure, and zone-redundant storage repository for OIM audit logs.

This repository must be interoperable across three distinct compute tiers:
1. **Azure Virtual Machines (IaaS):** The legacy OIM environment responsible for generating daily audit logs.
2. **Azure App Service (PaaS):** A secure web portal used by governance teams to query and review logs in real-time.
3. **Azure Container Instances (Serverless):** A Python-based container executing a scheduled cron job (2:00 AM IST) to process the previous day’s logs, generate PDF compliance reports, and dispatch notifications before terminating.

Crucially, the organization mandates a Zero-Trust approach: no hardcoded credentials may exist across the environment. All secrets must be centralized, and access must be identity-driven.

## 🎯 Problem Statement
Architect, deploy, and govern a hybrid compute environment integrated with centralized cloud storage (Azure Files), strictly adhering to the following enterprise governance mandates:

* **Zero Public Exposure (Network Isolation):** Public network access must be completely disabled for the Azure Files share and the secret store. Resources must only be accessible from within the secured Azure Virtual Network (VNet). *Architectural Decision: Azure Service Endpoints will be utilized to enforce subnet-level network isolation over the Azure backbone, balancing strict security with cost optimization.*
* **Identity-Based Access (RBAC & Key Vault):** Legacy authentication (Storage Account Access Keys) is strictly prohibited. Azure Key Vault must broker all secrets. Access to the file share and Key Vault must be granted exclusively via Microsoft Entra ID System-Assigned Managed Identities, enforcing the Principle of Least Privilege.
* **Automated Guardrails (Azure Policy & Locks):** The environment must be self-governing. Custom Azure Policies must automatically block the deployment of unsecured, public-facing storage accounts, and `CanNotDelete` Resource Locks must protect core networking infrastructure from accidental deletion.
* **Centralized Observability (Azure Log Analytics):** All diagnostic logs, access requests, network traffic flows, and policy evaluations across the VNet, VM, App Service, ACI, Key Vault, and Storage Account must stream into a central Log Analytics Workspace for proactive KQL querying and security investigation.

---

## 🏗️ Architecture & Network Design

> **![Architecture Diagram](images/01_architecture.png)**
> *Caption: High-level logical architecture demonstrating network isolation via Service Endpoints and identity brokering.*

### Key Architectural Decisions
1. **Network Isolation (Service Endpoints):** Service Endpoints (SE) lock down the PaaS firewalls (Key Vault, Storage) to specific Virtual Network subnet IDs, ensuring traffic remains entirely on the Azure backbone without the overhead of Private DNS zones.
2. **Passwordless Compute:** Utilized a **System-Assigned Managed Identity** for the Azure Container Instance (ACI). The compute node authenticates to the Key Vault natively via Microsoft Entra ID.
3. **Principle of Least Privilege:** The container identity was granted the `Key Vault Secrets User` RBAC role. It cannot create, delete, or modify secrets—it can only read the specific database credentials required for the extraction script.

## 🛠️ Technology Stack
* **Compute:** Azure Container Instances (ACI), Azure App Service, Azure Virtual Machines
* **Security & Identity:** Azure Key Vault, Microsoft Entra ID (Managed Identities), Azure RBAC
* **Networking:** Azure Virtual Network (VNet), Subnets, Service Endpoints
* **Observability:** Azure Log Analytics Workspace, Kusto Query Language (KQL)
* **Storage:** Azure Files (Secure SMB over Service Endpoints)

---

## 🔒 Implementation Evidence

### 1. The Network Perimeter
Public access to the Key Vault is completely disabled. The firewall explicitly permits traffic only from the administrative IP and the internal container subnet (`Snet-ACI`).

> **![Key Vault Firewall](images/02_network_firewall.png)**
> *Caption: Key Vault firewall configuration restricting access to the private VNet.*

### 2. The Identity Bridge (RBAC)
The System-Assigned Managed Identity of the Container Instance is bound to the Key Vault. This proves the transition from legacy credential management to Zero-Trust identity management.

> **![RBAC Assignment](images/03_rbac_identity.png)**
> *Caption: Role assignment showing the ACI identity bound to the Key Vault.*

### 3. Observability & Telemetry
A core tenet of cloud governance is observability. Centralized logging is enabled for the ACI. The KQL query below validates that the serverless compute node is actively sending heartbeat and operational logs to the Log Analytics Workspace.

> **![KQL Telemetry](images/04_kql_proof.png)**
> *Caption: Log Analytics Workspace results verifying the operational status of the compute layer.*

---

## ⚠️ Challenges & Technical Resolutions
* **VNet Injection UI Constraints:** During ACI deployment into a custom VNet, the Azure Portal UI suppresses the Managed Identity configuration toggle. **Resolution:** Deployed the baseline container first, then executed a secondary configuration pass to enable the System-Assigned Identity post-boot.
* **Telemetry Schema Hydration:** Encountered KQL schema errors ("Invalid Column") immediately after deployment due to standard cloud ingestion latency. **Resolution:** Shifted from specific table querying (`ContainerLog`) to a global `search` command to verify raw data flow before the schema was fully hydrated in the workspace.

## 💰 Cost Governance (FinOps)
This architecture was designed as an ephemeral Proof of Concept. Upon successful validation of the telemetry and RBAC models, the entire resource group (`rg-oimgov-dev-sin-001`) was systematically decommissioned to prevent zombie resources and eliminate idle cloud costs.
