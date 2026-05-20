# Designing an Azure Enterprise-Scale Landing Zone for Microsoft Fabric Workloads

*A practical blueprint for bringing Fabric into a governed, enterprise-ready Azure environment.*

---

## Why Fabric needs a landing zone

Microsoft Fabric is delivered as a **SaaS analytics platform** built on top of Azure, unifying Data Factory, Data Engineering, Data Warehouse, Real-Time Intelligence, Data Science, and Power BI on a shared **OneLake** foundation. Because it is SaaS, it's tempting to treat Fabric as "outside" your Azure landing zone — but in reality, every Fabric capacity is a **first-class Azure resource** that consumes identity from Microsoft Entra, bills through your Azure subscription, and connects to the rest of your data estate through Azure networking.

That makes Fabric a natural fit for the [Azure landing zone](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/) (ALZ) approach defined in the Cloud Adoption Framework: a modular, opinionated architecture that applies consistent controls for identity, networking, security, governance, and operations across every subscription.

---

## The reference shape

In an enterprise-scale design, Fabric typically sits inside a **data platform application landing zone** (a spoke), governed by the central **platform landing zone** (hub):

| Layer | Responsibility | Fabric mapping |
|---|---|---|
| Platform – Identity | Microsoft Entra ID tenant, groups, PIM, Conditional Access | Fabric tenant is bound to the same Entra tenant; users, service principals, and **workspace identities** are issued here |
| Platform – Connectivity | Hub VNet, ExpressRoute/VPN, Private DNS, Firewall | Hosts Private Endpoints for OneLake, Warehouse SQL, and on-prem data gateway egress |
| Platform – Security | Microsoft Defender for Cloud, Microsoft Sentinel, Key Vault, Purview, Azure Policy guardrails | Ingests Fabric audit and activity logs into Sentinel; Purview governs OneLake and mirrored sources; AKV stores connection secrets; policies enforce private-link, allowed SKUs, and tag standards |
| Platform – Management | Log Analytics, Defender for Cloud, Policy | Receives Fabric capacity metrics, Purview audit logs, and DLP alerts |
| Application LZ – Data | Subscription(s) holding Fabric capacities and adjacent Azure data services | F-SKU capacities, Azure Storage (shortcuts), Azure SQL/Synapse mirrored sources, AKV |

Use a dedicated **management group** for data & analytics so that Azure Policy can enforce capacity SKUs, allowed regions, private-link requirements, and tag standards consistently across every Fabric subscription.

---

## Architecture diagram

```mermaid
flowchart TB
    classDef mg fill:#0a2540,stroke:#0a2540,color:#fff,stroke-width:1px
    classDef platform fill:#e8f0fe,stroke:#1a73e8,color:#0a2540
    classDef data fill:#e6f4ea,stroke:#137333,color:#0a2540
    classDef fabric fill:#fff4e5,stroke:#e8710a,color:#0a2540
    classDef ext fill:#f3e8fd,stroke:#7b1fa2,color:#0a2540
    classDef gov fill:#fce8e6,stroke:#c5221f,color:#0a2540

    Tenant[Microsoft Entra ID Tenant<br/>Users · Groups · Service Principals · PIM · Conditional Access]:::gov

    subgraph TRG[Tenant Root Management Group]
        direction TB
        subgraph PLATMG[Platform MG]
            direction TB
            IDSUB[Identity Subscription<br/>Entra Connect · PIM]:::platform
            MGMTSUB[Management Subscription<br/>Log Analytics · Defender · Purview · Azure Monitor]:::platform
            CONNSUB[Connectivity Subscription<br/>Hub VNet · Azure Firewall · ExpressRoute/VPN<br/>Private DNS Zones]:::platform
            SECSUB[Security Subscription<br/>Microsoft Sentinel · Defender for Cloud<br/>Central Key Vault/HSM · Purview]:::platform
        end

        subgraph DATAMG["Data & Analytics MG — Azure Policy: region · SKU · Private Link · tags"]
            direction TB
            subgraph DEVSUB[Dev Subscription - Application Landing Zone]
                direction TB
                DEVCAP[Fabric Capacity F-SKU - Dev]:::fabric
                DEVWS[Workspaces · Domains<br/>Workspace Identities]:::fabric
                DEVDATA[ADLS Gen2 · Azure SQL · Key Vault · Event Hubs]:::data
            end
            subgraph PRODSUB[Prod Subscription - Application Landing Zone]
                direction TB
                PRODCAP[Fabric Capacity F-SKU - Prod]:::fabric
                PRODWS[Workspaces · Domains<br/>Workspace Identities]:::fabric
                PRODDATA[ADLS Gen2 · Azure SQL · Key Vault · Event Hubs]:::data
                ONELAKE[(OneLake<br/>Lakehouse · Warehouse · Eventhouse)]:::fabric
                PBI[Power BI Reports & Apps]:::fabric
            end
        end
    end

    subgraph EXT[External & On-Premises]
        direction TB
        ONPREM[On-Premises Sources & <br/> Multi Cloud <br/>via Data Gateway / Private Link Scope for Fabric Dataflow v2 / Fabric Notebook]:::ext
        DEVOPS[Azure DevOps / GitHub<br/>Git Integration · Deployment Pipelines · Bicep/Terraform]:::ext
        USERS[Business Users · Analysts · Engineers]:::ext
    end

    Tenant -. authenticates .-> DEVWS
    Tenant -. authenticates .-> PRODWS
    Tenant -. SSO + CA .-> USERS

    USERS -->|HTTPS / Conditional Access| PRODWS
    DEVOPS -->|CI/CD| DEVWS
    DEVOPS -->|Promote| PRODWS

    CONNSUB -->|Private Endpoints<br/>Private Link Scopes| PRODWS
    CONNSUB -->|Private Endpoints<br/>Private Link Scopes| DEVWS
    ONPREM -->|ExpressRoute / VPN| CONNSUB

    PRODWS --> ONELAKE
    DEVWS --> ONELAKE
    ONELAKE --> PBI
    PRODWS -->|Managed PE| PRODDATA
    DEVWS -->|Managed PE| DEVDATA

    MGMTSUB -. Capacity Metrics<br/>Operational Logs .- PRODCAP
    MGMTSUB -. Capacity Metrics<br/>Operational Logs .- DEVCAP

    SECSUB -. Fabric Audit<br/>Sign-in Logs .- PRODWS
    SECSUB -. Fabric Audit<br/>Sign-in Logs .- DEVWS
    SECSUB -. Purview Scan<br/>DLP · Sensitivity Labels .- ONELAKE
    SECSUB -. Defender Posture<br/>Threat Alerts .- PRODDATA
    SECSUB -. Defender Posture<br/>Threat Alerts .- DEVDATA
    SECSUB -. CMK / Encryption .- PRODCAP
    Tenant -. Entra Sign-in & Audit Logs .-> SECSUB
```

---

## Eight design areas, applied to Fabric

### 1. Identity and access

- Bind the Fabric tenant to your corporate Entra ID; reuse Conditional Access and MFA.
- Prefer **workspace identities** and service principals for outbound connections to Azure data sources — not user credentials.
- Use Entra security groups (not individual users) for **workspace roles** (Admin, Member, Contributor, Viewer) and capacity assignment.

### 2. Network topology and connectivity

- Enable **Private Link at the tenant or workspace level** and turn on *Block Public Internet Access* for sensitive tenants.
- Use **Managed VNets and Managed Private Endpoints** so that Spark, pipelines, and OneLake reach Azure PaaS sources (ADLS, SQL, Key Vault) without traversing the public internet.
- Place Private DNS zones for `*.fabric.microsoft.com` and `*.onelake.dfs.fabric.microsoft.com` in the hub.
- Use the following patterns to have a robust Data Ingestion pattern that supports Low Code (Fabric Dataflow v2) and Pro Code (Fabric Notebooks)

```mermaid
flowchart LR
    subgraph Fabric["Microsoft Fabric"]
        FabricNB["Fabric Notebooks"]
        FabricSvc["Fabric Service<br/>(Dataflows / Pipelines)"]
    end

    subgraph PL_Scope["Azure Private Link Scope"]
        PE["Azure Private Link Scope"]
    end

    subgraph Azure_VNet["Azure Virtual Network"]
        subgraph PLS_Subnet["Private Link Service Subnet"]
            PLS["Azure Private Endpoint"]
            ILB["Azure Internal Load Balancer"]
        end
        subgraph NAT_Subnet["NAT / Proxy Subnet"]
            Azure_VPN_GCP["Azure VPN Gateway (GCP)"]
            Azure_VPN_AWS["Azure VPN Gateway (AWS)"]
            Azure_VPN_OnPrem["Azure VPN Gateway (Kyndryl)"]
            NAT_VM1["NAT VM 1"]
            NAT_VM2["NAT VM 2"]
        end
        subgraph Subnet["VM Subnet"]
            GW_VM["Fabric On-Premises<br/>Data Gateway VM"]
        end
    end

    subgraph GCP["Google Cloud Platform"]
        BQ["Google BigQuery"]
        GCP_VPN["GCP Cloud VPN Gateway"]
    end

    subgraph AWS["Amazon Web Services"]
        SF["Snowflake"]
        AWS_VPN["AWS VPN Gateway"]
    end

    subgraph OnPrem["Kyndrl Cloud"]
        DB2["IBM DB2"]
        OnPrem_VPN["Kyndryl VPN Gateway"]
    end

    BQ -- "BigQuery API<br/>(Private Google Access)" --> GCP_VPN
    SF -- "Snowflake API<br/>(PrivateLink / Direct)" --> AWS_VPN
    DB2 -- "DB2 Connect (TCP/IP)" --> OnPrem_VPN

    GCP_VPN <-- "IPsec Tunnel" --> Azure_VPN_GCP
    AWS_VPN <-- "IPsec Tunnel" --> Azure_VPN_AWS
    OnPrem_VPN <-- "IPsec Tunnel" --> Azure_VPN_OnPrem

    Azure_VPN_GCP -- "Routed Traffic" --> GW_VM
    Azure_VPN_AWS -- "Routed Traffic" --> GW_VM
    Azure_VPN_OnPrem -- "Routed Traffic" --> GW_VM

    NAT_VM1 -- "VPN Route (GCP)" --> Azure_VPN_GCP
    NAT_VM2 -- "VPN Route (GCP)" --> Azure_VPN_GCP
    NAT_VM1 -- "VPN Route (AWS)" --> Azure_VPN_AWS
    NAT_VM2 -- "VPN Route (AWS)" --> Azure_VPN_AWS
    NAT_VM1 -- "VPN Route (On-Prem)" --> Azure_VPN_OnPrem
    NAT_VM2 -- "VPN Route (On-Prem)" --> Azure_VPN_OnPrem

    ILB --> NAT_VM1
    ILB --> NAT_VM2

    PLS -- "Fronts" --> ILB
    PE -- "Private Link Connection" --> PLS
    FabricNB -- "Managed VNet<br/>Private Endpoint Connection" --> PE

    GW_VM -- "Outbound HTTPS<br/>(Gateway Registration)" --> FabricSvc
    FabricSvc -- "Data Requests" --> GW_VM

    classDef gcp fill:#4285F4,stroke:#2a56c6,color:#fff
    classDef aws fill:#FF9900,stroke:#cc7a00,color:#fff
    classDef onprem fill:#2E7D32,stroke:#1b5e20,color:#fff
    classDef azure fill:#0078D4,stroke:#005a9e,color:#fff
    classDef fabric fill:#F2C811,stroke:#111,color:#333
    classDef pls fill:#7B3F99,stroke:#5c2d82,color:#fff

    class BQ,GCP_VPN gcp
    class SF,AWS_VPN aws
    class DB2,OnPrem_VPN onprem
    class Azure_VPN_GCP,Azure_VPN_AWS,Azure_VPN_OnPrem,NAT_VM1,NAT_VM2,GW_VM,ILB azure
    class FabricNB,FabricSvc fabric
    class PE,PLS pls
```

### 3. Resource organization

- One **management group** for analytics; child subscriptions for *Platform-Shared*, *Dev*, *Test*, and *Prod* Fabric capacities.
- Capacities act as **isolation and chargeback boundaries** — split them by environment and, where needed, by business domain.

### 4. Governance

- Adopt **OneLake Catalog + Microsoft Purview** (built into Fabric) for discovery, sensitivity labels, lineage, and DLP.
- Use Fabric **domains and subdomains** to mirror your business architecture; assign domain owners from the COE.
- Enforce metadata scanning via the **Admin scanner APIs** for tenant-wide cataloging.

### 5. Security

- Apply sensitivity labels at ingestion; let them flow through OneLake to Power BI exports.
- Layer **item-level controls** — RLS/CLS on Warehouse, SQL analytics endpoint, and KQL databases — on top of workspace roles.
- Store secrets in **Azure Key Vault** and reference them through Fabric connections, never inline.

### 6. Management and monitoring

- Stream **Fabric capacity metrics**, Purview audit logs, and Entra sign-in logs into a central **Log Analytics workspace** in the management subscription.
- Use the **Capacity Metrics app** plus Azure Monitor workbooks for throttling, smoothing, and chargeback insight.

### 7. Business continuity and DR

- Choose capacity **home regions** aligned with data residency; enable **OneLake disaster recovery** and cross-region replication for critical lakehouses.
- Externalize CI/CD artifacts in **Git (Azure DevOps or GitHub)** using Fabric's built-in Git integration so workspaces are reproducible.

### 8. Platform automation and DevOps

- Provision capacities, workspace identities, role assignments, and Private Endpoints with **Bicep / Terraform** as part of subscription vending.
- Promote Fabric items across DTAP through **deployment pipelines** or Git-based CI/CD; keep dev workspaces isolated per developer.

---

## A typical landing pattern

1. **Platform team** vends a *Data Analytics* subscription with policy-enforced tagging, region, and Private Link requirements.
2. **Data platform team** deploys an **F-SKU Fabric capacity**, attaches it to a domain, and creates Dev/Test/Prod **workspaces** bound to Entra groups.
3. **Workspace identity** is provisioned and granted least-privilege RBAC on the Azure data sources it must read (ADLS Gen2, Azure SQL, Event Hubs).
4. **Managed Private Endpoints** are approved for each source; on-prem systems reach Fabric through the **on-premises data gateway** or mirroring.
5. **Purview** scans the tenant; sensitivity labels and DLP policies activate automatically across new items.
6. **Monitoring** lights up: capacity metrics, audit logs, and Defender alerts flow to the central Log Analytics workspace.

---

## Anti-patterns to avoid

- ❌ One giant shared capacity for every team — you lose isolation and chargeback signal.
- ❌ Per-user workspace permissions — unmanageable at scale; always use Entra groups.
- ❌ Public endpoints on production tenants — enable Private Link and block public access.
- ❌ Treating Fabric as "outside" governance — it belongs inside your ALZ policy and Purview scope.
- ❌ Manual workspace creation — automate via subscription vending and Git-backed deployment pipelines.

---

## Closing thought

Microsoft Fabric removes most of the infrastructure plumbing, but it does **not** remove the need for an enterprise-scale operating model. By landing Fabric inside an Azure landing zone — with Entra-driven identity, Private Link networking, Purview governance, capacity-based isolation, and IaC-driven vending — you give your analytics teams a paved road that is secure, compliant, and ready to scale from a single lakehouse to a tenant-wide data mesh.

### References
- [What is an Azure landing zone?](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [What is Microsoft Fabric?](https://learn.microsoft.com/fabric/fundamentals/microsoft-fabric-overview)
- [Integrate Microsoft Fabric with external systems and platform connectivity](https://learn.microsoft.com/fabric/fundamentals/external-integration)
- [Fabric governance overview and guidance](https://learn.microsoft.com/fabric/governance/governance-compliance-overview)
- [Workspace identity in Microsoft Fabric](https://learn.microsoft.com/fabric/security/workspace-identity)
- [Managed Private Endpoints in Fabric](https://learn.microsoft.com/fabric/security/security-managed-private-endpoints-overview)
- [Enterprise BI with Microsoft Fabric (reference architecture)](https://learn.microsoft.com/azure/architecture/example-scenario/analytics/enterprise-bi-microsoft-fabric)
