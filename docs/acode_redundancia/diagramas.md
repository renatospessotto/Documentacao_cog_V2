# 📊 Diagramas Técnicos - ACODE + Redundância

## 📋 Visão Geral

Esta seção apresenta diagramas técnicos detalhados do sistema ACODE com redundância, incluindo arquitetura de componentes, fluxos de dados, procedimentos de failover e monitoramento.

## 🏗️ Arquitetura Geral do Sistema

```mermaid
graph TB
    subgraph "🏢 ACODE (External Partner)"
        A1[MySQL Primary<br/>db-hsp-farmarcas.acode.com.br<br/>acode_farmarcas]
        A2[MySQL Replica<br/>Internal Backup<br/>acode_farmarcas_replica]
        A1 -.->|Replication| A2
    end
    
    subgraph "☁️ Farmarcas Infrastructure"
        subgraph "📊 Data Platform"
            B1[Airbyte Platform<br/>v0.3.23]
            B2[Source MySQL Primary<br/>source-mysql v1.0.21]
            B3[Source MySQL Backup<br/>source-mysql v1.0.21]
            B4[Destination S3 Primary<br/>destination-s3 v0.3.23]
            B5[Destination S3 Backup<br/>destination-s3 v0.3.23]
        end
        
        subgraph "🗄️ Storage Layer"
            C1[(S3 Primary<br/>farmarcas-production-bronze)]
            C2[(S3 Backup<br/>farmarcas-backup-acode)]
            C3[(S3 Emergency<br/>farmarcas-emergency-backup)]
            C4[(S3 Quarantine<br/>farmarcas-quarantine)]
        end
        
        subgraph "🔄 Orchestration"
            D1[Airflow DAGs<br/>Health Monitoring]
            D2[Auto-Recovery Scripts<br/>Failover Automation]
            D3[Data Quality Validation<br/>Quality Gates]
        end
        
        subgraph "📈 Monitoring"
            E1[Health Checks<br/>Real-time Monitoring]
            E2[Alerting System<br/>Slack + PagerDuty]
            E3[Performance Metrics<br/>DataDog + Logs]
        end
    end
    
    subgraph "📊 Downstream Systems"
        F1[Silver Layer<br/>Data Transformations]
        F2[Executive Dashboards<br/>Business Intelligence]
        F3[Financial Reconciliation<br/>Compliance Reports]
    end
    
    A1 --> B2
    A2 --> B3
    B2 --> B4
    B3 --> B5
    B4 --> C1
    B5 --> C2
    
    B1 --> B2
    B1 --> B3
    B1 --> B4
    B1 --> B5
    
    D1 --> B1
    D2 --> B1
    D3 --> C1
    
    E1 --> D2
    E2 --> E1
    E3 --> E1
    
    C1 --> F1
    C1 --> F2
    C1 --> F3
    C2 -.->|Emergency| F1
    C3 -.->|Disaster Recovery| F1
    C4 -.->|Quality Issues| D3
    
    style A1 fill:#ff6b6b
    style A2 fill:#feca57
    style B1 fill:#4ecdc4
    style C1 fill:#45b7d1
    style C2 fill:#96ceb4
    style C3 fill:#f8b500
    style C4 fill:#e74c3c
    style D1 fill:#9b59b6
    style E1 fill:#2ecc71
```

## 🔄 Fluxo de Dados Detalhado

```mermaid
flowchart TD
    subgraph "📊 Data Source"
        A[ACODE MySQL<br/>External Database]
        A1[Table: farmarcas_si_analitico_diario<br/>~4-8GB daily]
        A2[Table: farmarcas_si_analitico_diario_produtos<br/>~500MB daily]
        A --> A1
        A --> A2
    end
    
    subgraph "🔌 Airbyte Ingestion"
        B1[Source Configuration<br/>MySQL Connector]
        B2[Stream 1: Fiscal Data<br/>Full Refresh + Append]
        B3[Stream 2: Product Data<br/>Full Refresh + Append]
        B4[Data Validation<br/>Schema Enforcement]
        B5[Data Transformation<br/>Type Conversion]
        
        B1 --> B2
        B1 --> B3
        B2 --> B4
        B3 --> B4
        B4 --> B5
    end
    
    subgraph "☁️ S3 Storage"
        C1[Primary Bucket<br/>farmarcas-production-bronze]
        C2[Path Structure<br/>origin=airbyte/database=bronze_acode]
        C3[Partitioning<br/>cog_dt_ingestion=YYYY-MM-DD]
        C4[File Format<br/>Parquet + SNAPPY]
        
        C1 --> C2
        C2 --> C3
        C3 --> C4
    end
    
    subgraph "📊 Data Processing"
        D1[Bronze Layer<br/>Raw Data Storage]
        D2[Silver Layer<br/>Cleaned & Transformed]
        D3[Gold Layer<br/>Business Aggregations]
        
        D1 --> D2
        D2 --> D3
    end
    
    A1 --> B2
    A2 --> B3
    B5 --> C1
    C4 --> D1
    
    style A fill:#ff6b6b
    style B1 fill:#4ecdc4
    style C1 fill:#45b7d1
    style D1 fill:#2ecc71
    style D2 fill:#f39c12
    style D3 fill:#9b59b6
```

## ⚡ Processo de Failover

```mermaid
sequenceDiagram
    participant HM as Health Monitor
    participant PM as Primary MySQL
    participant BM as Backup MySQL
    participant PC as Primary Connection
    participant BC as Backup Connection
    participant AL as Alerting
    participant OP as Operations Team
    
    Note over HM: Every 5 minutes
    HM->>PM: Health Check Query
    
    alt Primary Healthy
        PM-->>HM: Success Response
        HM->>AL: Status OK
    else Primary Failed
        PM--xHM: Connection Failed
        Note over HM: Failure detected
        
        HM->>BM: Check Backup Health
        BM-->>HM: Backup OK
        
        HM->>PC: Deactivate Primary
        PC-->>HM: Primary Inactive
        
        HM->>BC: Activate Backup
        BC-->>HM: Backup Active
        
        HM->>AL: Send Critical Alert
        AL->>OP: "ACODE failover executed"
        
        Note over HM,OP: Monitor backup performance
        
        loop Every 10 minutes
            HM->>PM: Test Primary Recovery
            alt Primary Recovered
                PM-->>HM: Primary OK
                HM->>BC: Deactivate Backup
                HM->>PC: Reactivate Primary
                HM->>AL: Send Recovery Alert
                AL->>OP: "ACODE primary restored"
            else Primary Still Failed
                PM--xHM: Still Failed
                Note over HM: Continue with backup
            end
        end
    end
```

## 🔍 Health Check Flow

```mermaid
flowchart TD
    A[Health Check Trigger<br/>Every 5 minutes] --> B{Test Primary MySQL}
    
    B -->|Success| C[✅ Primary OK]
    B -->|Fail| D[❌ Primary Failed]
    
    C --> E{Test S3 Access}
    D --> F{Test Backup MySQL}
    
    E -->|Success| G[✅ All Systems OK]
    E -->|Fail| H[❌ S3 Issue]
    
    F -->|Success| I{Test Backup S3}
    F -->|Fail| J[🚨 Critical: All Failed]
    
    I -->|Success| K[⚠️ Activate Backup]
    I -->|Fail| J
    
    G --> L[Send Success Alert]
    H --> M[Send S3 Alert]
    J --> N[Send Critical Alert]
    K --> O[Execute Failover]
    
    L --> P[Continue Monitoring]
    M --> Q[Attempt S3 Recovery]
    N --> R[Manual Intervention Required]
    O --> S[Monitor Backup Performance]
    
    S --> T{Primary Recovered?}
    T -->|Yes| U[Switch Back to Primary]
    T -->|No| V[Continue with Backup]
    
    U --> P
    V --> S
    
    style B fill:#e1f5fe
    style F fill:#e1f5fe
    style I fill:#e1f5fe
    style T fill:#e1f5fe
    style G fill:#c8e6c9
    style C fill:#c8e6c9
    style L fill:#c8e6c9
    style U fill:#c8e6c9
    style H fill:#ffcdd2
    style D fill:#ffcdd2
    style J fill:#ffcdd2
    style N fill:#ffcdd2
    style K fill:#fff3e0
    style O fill:#fff3e0
    style S fill:#fff3e0
```

## 🗂️ Data Schema & Structure

```mermaid
erDiagram
    farmarcas_si_analitico_diario {
        bigint idpk PK "Primary Key"
        decimal ACODE_Val_Total "Total Value"
        varchar CNPJ "Pharmacy CNPJ"
        varchar CNPJ_Fornecedor "Supplier CNPJ"
        date Data "Transaction Date"
        datetime Data_Processamento "Processing Timestamp"
        varchar EAN "Product Barcode"
        varchar Fornecedor "Supplier Name"
        int NF_Numero "Invoice Number"
        decimal Qtd_Trib "Taxable Quantity"
        decimal Val_Prod "Product Value"
        decimal Val_Trib "Tax Value"
        int CFOP "Tax Operation Code"
        int NCM "Mercosul Nomenclature"
        varchar CST "Tax Situation Code"
        decimal Aliquota_ICMS "ICMS Rate"
        decimal Valor_ICMS "ICMS Value"
        decimal IPI "IPI Tax"
        decimal ST "Tax Substitution"
        decimal STRet "Retained Tax Substitution"
    }
    
    farmarcas_si_analitico_diario_produtos {
        bigint idpk PK "Primary Key"
        varchar EAN "Product Barcode"
        varchar Produto "Product Name"
        varchar Fabricante "Manufacturer"
        varchar P_Ativo "Active Ingredient"
        varchar Classe "Therapeutic Class"
        varchar Sub_Classe "Sub Class"
        varchar Familia "Product Family"
        varchar Grupo "Product Group"
        varchar Tipo "Product Type"
        varchar Desc_Marca "Brand Description"
        varchar Holding "Corporate Group"
    }
    
    farmarcas_si_analitico_diario ||--o{ farmarcas_si_analitico_diario_produtos : "EAN"
```

## 📊 Monitoring Dashboard Layout

```mermaid
graph TB
    subgraph "📊 ACODE Monitoring Dashboard"
        subgraph "🟢 System Health"
            A1[Primary MySQL Status<br/>🟢 Online | 🔴 Offline]
            A2[Backup MySQL Status<br/>🟢 Online | 🔴 Offline]
            A3[S3 Access Status<br/>🟢 Available | 🔴 Failed]
            A4[Airbyte Connection<br/>🟢 Active | 🟡 Standby | 🔴 Failed]
        end
        
        subgraph "📈 Performance Metrics"
            B1[Query Response Time<br/>< 30s Target]
            B2[Sync Duration<br/>< 3h Target]
            B3[Data Freshness<br/>< 25h Target]
            B4[Replication Lag<br/>< 5min Target]
        end
        
        subgraph "📊 Data Quality"
            C1[Records Processed<br/>Daily Count]
            C2[Quality Score<br/>> 90% Target]
            C3[Quarantined Records<br/>< 1% Target]
            C4[Validation Errors<br/>Count & Trends]
        end
        
        subgraph "🚨 Alerts & Actions"
            D1[Active Alerts<br/>Critical/Warning/Info]
            D2[Recent Failovers<br/>Last 24h Events]
            D3[Manual Actions<br/>Required Interventions]
            D4[System Recovery<br/>Auto/Manual Status]
        end
        
        subgraph "📅 Historical Trends"
            E1[Sync Success Rate<br/>7/30 day trends]
            E2[Data Volume Trends<br/>GB per day]
            E3[Performance Trends<br/>Response time history]
            E4[Failure Patterns<br/>Incident analysis]
        end
    end
    
    style A1 fill:#c8e6c9
    style A2 fill:#c8e6c9
    style A3 fill:#c8e6c9
    style A4 fill:#c8e6c9
    style B1 fill:#e1f5fe
    style B2 fill:#e1f5fe
    style B3 fill:#e1f5fe
    style B4 fill:#e1f5fe
    style C1 fill:#f3e5f5
    style C2 fill:#f3e5f5
    style C3 fill:#f3e5f5
    style C4 fill:#f3e5f5
    style D1 fill:#ffebee
    style D2 fill:#ffebee
    style D3 fill:#ffebee
    style D4 fill:#ffebee
    style E1 fill:#fff3e0
    style E2 fill:#fff3e0
    style E3 fill:#fff3e0
    style E4 fill:#fff3e0
```

## 🛠️ Deployment Architecture

```mermaid
graph TB
    subgraph "🚀 Deployment Environment"
        subgraph "📦 Kubernetes Cluster"
            K1[Namespace: airbyte<br/>Airbyte Platform]
            K2[Namespace: monitoring<br/>Health Checks]
            K3[Namespace: orchestration<br/>Airflow DAGs]
        end
        
        subgraph "🐳 Docker Images"
            D1[airbyte/source-mysql:1.0.21<br/>MySQL Connector]
            D2[airbyte/destination-s3:0.3.23<br/>S3 Connector]
            D3[custom/acode-health-monitor<br/>Health Check Services]
            D4[custom/acode-auto-recovery<br/>Failover Automation]
        end
        
        subgraph "📝 Configuration Management"
            C1[Octavia CLI<br/>Configuration as Code]
            C2[Helm Charts<br/>Kubernetes Deployments]
            C3[Environment Variables<br/>Secrets Management]
            C4[YAML Configurations<br/>Source/Destination/Connection]
        end
        
        subgraph "🔧 Operational Tools"
            O1[kubectl<br/>Cluster Management]
            O2[AWS CLI<br/>S3 Operations]
            O3[MySQL Client<br/>Database Access]
            O4[Monitoring Scripts<br/>Health & Performance]
        end
    end
    
    K1 --> D1
    K1 --> D2
    K2 --> D3
    K3 --> D4
    
    C1 --> K1
    C2 --> K2
    C3 --> K3
    C4 --> C1
    
    O1 --> K1
    O2 --> D2
    O3 --> D1
    O4 --> K2
    
    style K1 fill:#e3f2fd
    style K2 fill:#e8f5e8
    style K3 fill:#fff3e0
    style D1 fill:#f3e5f5
    style D2 fill:#f3e5f5
    style D3 fill:#e1f5fe
    style D4 fill:#e1f5fe
```

## 🔄 Data Lifecycle Management

```mermaid
timeline
    title ACODE Data Lifecycle
    
    section Data Ingestion
        00:00 : Health Check
              : Primary MySQL Test
              : Connection Validation
        
        01:00 : Sync Start
              : Airbyte Job Launch
              : Resource Allocation
              
        01:30 : Data Extract
              : MySQL Query Execution
              : 4-8GB Transfer
              
        02:30 : Data Transform
              : Schema Validation
              : Type Conversion
              
        03:00 : Data Load
              : S3 Upload (Parquet)
              : SNAPPY Compression
              
        03:30 : Sync Complete
              : Quality Validation
              : Success Notification
    
    section Monitoring
        Every 5min : Health Checks
                   : MySQL Connectivity
                   : S3 Access Test
                   
        Every 15min : Performance Check
                    : Query Response Time
                    : Replication Lag
                    
        Every 1hr : Data Quality
                  : Validation Rules
                  : Business Logic
                  
        Daily : Trend Analysis
              : Volume Patterns
              : Performance Metrics
    
    section Backup & Recovery
        Continuous : MySQL Replication
                   : Primary to Backup
                   : < 5min Lag
                   
        On Failure : Auto Failover
                   : Backup Activation
                   : Alert Notification
                   
        Recovery : Primary Restoration
                 : Automatic Switchback
                 : System Validation
                 
        Weekly : Backup Testing
               : DR Procedures
               : Recovery Validation
```

## 🏗️ Network Architecture

```mermaid
graph TB
    subgraph "🌐 External Network"
        E1[ACODE Systems<br/>db-hsp-farmarcas.acode.com.br<br/>Port 3306/SSL]
    end
    
    subgraph "🔒 DMZ/Firewall"
        F1[Load Balancer<br/>SSL Termination]
        F2[Security Groups<br/>Port 3306 Allow]
        F3[VPN Gateway<br/>Secure Tunnel]
    end
    
    subgraph "☁️ AWS VPC"
        subgraph "🌍 Public Subnet"
            P1[NAT Gateway<br/>Internet Access]
            P2[Internet Gateway<br/>Outbound Only]
        end
        
        subgraph "🔒 Private Subnet"
            V1[Airbyte Platform<br/>EKS Cluster]
            V2[Health Monitors<br/>Monitoring Services]
            V3[Auto Recovery<br/>Failover Scripts]
        end
        
        subgraph "🗄️ Data Subnet"
            D1[S3 VPC Endpoint<br/>Direct S3 Access]
            D2[Internal MySQL<br/>Backup Replica]
        end
    end
    
    subgraph "📊 AWS Services"
        A1[S3 Buckets<br/>Data Storage]
        A2[CloudWatch<br/>Metrics & Logs]
        A3[IAM<br/>Access Control]
        A4[Secrets Manager<br/>Credential Storage]
    end
    
    E1 --> F1
    F1 --> F2
    F2 --> F3
    F3 --> V1
    
    V1 --> P1
    P1 --> P2
    P2 --> E1
    
    V1 --> D1
    V1 --> D2
    V2 --> V1
    V3 --> V1
    
    D1 --> A1
    V1 --> A2
    V1 --> A3
    V1 --> A4
    
    style E1 fill:#ffcdd2
    style F1 fill:#fff3e0
    style F2 fill:#fff3e0
    style F3 fill:#fff3e0
    style V1 fill:#e3f2fd
    style V2 fill:#e8f5e8
    style V3 fill:#f3e5f5
    style D1 fill:#e1f5fe
    style D2 fill:#e1f5fe
    style A1 fill:#c8e6c9
    style A2 fill:#c8e6c9
    style A3 fill:#c8e6c9
    style A4 fill:#c8e6c9
```

## 📊 Performance Metrics Dashboard

```mermaid
graph TB
    subgraph "📊 Real-time Metrics"
        subgraph "🔄 Connection Status"
            M1[Primary Connection<br/>🟢 Active | 🔴 Failed]
            M2[Backup Connection<br/>🟡 Standby | 🟢 Active]
            M3[Failover Status<br/>🟢 Ready | 🔄 In Progress]
        end
        
        subgraph "⚡ Performance KPIs"
            P1[Query Response<br/>Current: 12.3s | Target: <30s]
            P2[Sync Duration<br/>Current: 2.1h | Target: <3h]
            P3[Data Throughput<br/>Current: 45MB/min]
            P4[Success Rate<br/>Last 24h: 98.5%]
        end
        
        subgraph "📈 Data Metrics"
            D1[Records/Day<br/>~2.8M records]
            D2[Data Volume<br/>~6.2GB daily]
            D3[Quality Score<br/>94.2% | Target: >90%]
            D4[Replication Lag<br/>3.2min | Target: <5min]
        end
        
        subgraph "🚨 Alert Summary"
            A1[Critical: 0<br/>🟢 All Clear]
            A2[Warnings: 2<br/>🟡 Minor Issues]
            A3[Last Incident<br/>48h ago - Resolved]
            A4[MTTR<br/>Average: 15min]
        end
    end
    
    style M1 fill:#c8e6c9
    style M2 fill:#fff3e0
    style M3 fill:#c8e6c9
    style P1 fill:#c8e6c9
    style P2 fill:#c8e6c9
    style P3 fill:#e1f5fe
    style P4 fill:#c8e6c9
    style D1 fill:#f3e5f5
    style D2 fill:#f3e5f5
    style D3 fill:#c8e6c9
    style D4 fill:#c8e6c9
    style A1 fill:#c8e6c9
    style A2 fill:#fff3e0
    style A3 fill:#e1f5fe
    style A4 fill:#c8e6c9
```

## 🔧 Technical Component Interactions

```mermaid
graph LR
    subgraph "🔌 Data Sources"
        DS1[ACODE Primary<br/>MySQL External]
        DS2[ACODE Backup<br/>MySQL Internal]
    end
    
    subgraph "🚀 Airbyte Platform"
        AB1[Source Connectors<br/>MySQL Drivers]
        AB2[Destination Connectors<br/>S3 Writers]
        AB3[Sync Orchestrator<br/>Job Management]
        AB4[Resource Manager<br/>CPU/Memory]
    end
    
    subgraph "☁️ Storage Systems"
        ST1[S3 Primary<br/>Bronze Layer]
        ST2[S3 Backup<br/>Redundancy]
        ST3[S3 Quarantine<br/>Quality Issues]
    end
    
    subgraph "🔍 Monitoring Layer"
        MN1[Health Checks<br/>System Status]
        MN2[Performance Metrics<br/>KPI Tracking]
        MN3[Data Quality<br/>Validation Rules]
        MN4[Alerting System<br/>Notifications]
    end
    
    subgraph "🔄 Automation"
        AT1[Auto-Recovery<br/>Failover Logic]
        AT2[Load Balancer<br/>Connection Routing]
        AT3[Backup Scheduler<br/>Sync Timing]
    end
    
    DS1 --> AB1
    DS2 --> AB1
    AB1 --> AB3
    AB2 --> AB3
    AB3 --> AB4
    AB2 --> ST1
    AB2 --> ST2
    AB2 --> ST3
    
    AB3 --> MN1
    ST1 --> MN2
    ST2 --> MN3
    MN1 --> MN4
    MN2 --> MN4
    MN3 --> MN4
    
    MN1 --> AT1
    AT1 --> AT2
    AT2 --> AB1
    AT3 --> AB3
    
    style DS1 fill:#ff6b6b
    style DS2 fill:#feca57
    style AB1 fill:#4ecdc4
    style AB2 fill:#4ecdc4
    style AB3 fill:#4ecdc4
    style AB4 fill:#4ecdc4
    style ST1 fill:#45b7d1
    style ST2 fill:#96ceb4
    style ST3 fill:#e74c3c
    style MN1 fill:#2ecc71
    style MN2 fill:#2ecc71
    style MN3 fill:#2ecc71
    style MN4 fill:#2ecc71
    style AT1 fill:#9b59b6
    style AT2 fill:#9b59b6
    style AT3 fill:#9b59b6
```

---

## 🎯 Diagrama de Casos de Uso

```mermaid
graph TD
    subgraph "👥 Actors"
        U1[Data Engineer<br/>System Operator]
        U2[Business Analyst<br/>Data Consumer]
        U3[System Admin<br/>Infrastructure]
        U4[ACODE System<br/>External Partner]
    end
    
    subgraph "🎯 Use Cases"
        UC1[Configure Data Ingestion<br/>Setup Sources & Destinations]
        UC2[Monitor System Health<br/>Real-time Status Checks]
        UC3[Execute Failover<br/>Switch to Backup Systems]
        UC4[Validate Data Quality<br/>Business Rule Checks]
        UC5[Troubleshoot Issues<br/>Diagnose & Resolve]
        UC6[Generate Reports<br/>Performance Analytics]
        UC7[Maintain Configurations<br/>Update Settings]
        UC8[Provide Data Feed<br/>External Data Source]
    end
    
    U1 --> UC1
    U1 --> UC2
    U1 --> UC3
    U1 --> UC5
    U1 --> UC7
    
    U2 --> UC4
    U2 --> UC6
    
    U3 --> UC2
    U3 --> UC3
    U3 --> UC5
    U3 --> UC7
    
    U4 --> UC8
    
    UC8 --> UC1
    UC2 --> UC3
    UC4 --> UC5
    UC6 --> UC4
    
    style U1 fill:#e3f2fd
    style U2 fill:#e8f5e8
    style U3 fill:#fff3e0
    style U4 fill:#ffebee
    style UC1 fill:#f3e5f5
    style UC2 fill:#e1f5fe
    style UC3 fill:#ffcdd2
    style UC4 fill:#c8e6c9
    style UC5 fill:#fff3e0
    style UC6 fill:#e8f5e8
    style UC7 fill:#f3e5f5
    style UC8 fill:#ffebee
```

---

**Próximo**: [Boas Práticas](./boas_praticas.md) - Recomendações de operação e manutenção
