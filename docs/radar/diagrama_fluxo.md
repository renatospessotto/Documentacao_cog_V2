# 📊 Diagrama de Fluxo - Ingestão do Radar

## 📋 Visão Geral

Este documento apresenta diagramas visuais detalhados do processo de ingestão do Radar, incluindo fluxos de dados, arquitetura de sistema, processos de erro e procedimentos de recovery.

## 🏗️ Arquitetura Geral do Sistema

```mermaid
graph TB
    subgraph "Production Environment"
        subgraph "MySQL Radar Database"
            RadarDB[(MySQL RDS<br/>db-mysql-radar-production<br/>:3306)]
            RadarTables[80+ Tables:<br/>• store<br/>• store_metrics<br/>• user_access<br/>• product<br/>• contest<br/>• voucher<br/>• brand_metrics_average]
        end
        
        subgraph "Airbyte Data Platform"
            AirbyteServer[Airbyte Server<br/>v0.3.23<br/>:8001]
            SourceMySQL[Source MySQL<br/>bi-cognitivo-read<br/>SSL enabled]
            DestS3[Destination S3<br/>Parquet + SNAPPY]
            Connection[Connection<br/>6c7fda57-ebdb-...<br/>Manual Schedule]
            Workers[Airbyte Workers<br/>Auto-scaling<br/>2-8 replicas]
        end
        
        subgraph "Airflow Orchestration"
            Scheduler[Airflow Scheduler<br/>dag_sync_airbyte_connections]
            DAGTasks[Tasks:<br/>• trigger_sync_radar<br/>• wait_sync_radar<br/>• validate_data<br/>• notify_success]
            WebServer[Airflow Web UI<br/>:8080]
        end
        
        subgraph "AWS Data Lake"
            S3Bronze[S3 Bronze Layer<br/>farmarcas-production-bronze<br/>origin=airbyte/database=bronze_radar]
            S3Structure[Directory Structure:<br/>table_name/<br/>cog_dt_ingestion=YYYY-MM-DD/<br/>file_table_*.parquet]
            GlueCrawler[AWS Glue Crawlers<br/>Schema Discovery<br/>Daily Updates]
            DataCatalog[AWS Glue Data Catalog<br/>bronze_radar database<br/>80+ tables]
        end
        
        subgraph "Analytics Layer"
            Athena[Amazon Athena<br/>Query Engine<br/>SQL Interface]
            PowerBI[Power BI<br/>Business Dashboards<br/>Executive Reports]
            DataScience[Data Science<br/>ML Pipelines<br/>Advanced Analytics]
        end
    end
    
    subgraph "Monitoring & Alerting"
        CloudWatch[CloudWatch<br/>Metrics & Logs]
        Slack[Slack Notifications<br/>#data-engineering<br/>#alerts]
        Grafana[Grafana Dashboard<br/>Performance Metrics]
    end
    
    %% Data Flow
    RadarDB --> RadarTables
    RadarTables --> SourceMySQL
    SourceMySQL --> Connection
    Connection --> DestS3
    DestS3 --> S3Bronze
    S3Bronze --> S3Structure
    S3Structure --> GlueCrawler
    GlueCrawler --> DataCatalog
    
    %% Orchestration Flow
    Scheduler --> DAGTasks
    DAGTasks --> AirbyteServer
    AirbyteServer --> Connection
    Workers --> Connection
    
    %% Analytics Flow
    DataCatalog --> Athena
    Athena --> PowerBI
    Athena --> DataScience
    
    %% Monitoring Flow
    AirbyteServer --> CloudWatch
    Scheduler --> CloudWatch
    CloudWatch --> Grafana
    DAGTasks --> Slack
    
    %% Styling
    classDef database fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef processing fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef storage fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef orchestration fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef analytics fill:#fce4ec,stroke:#880e4f,stroke-width:2px
    classDef monitoring fill:#f1f8e9,stroke:#33691e,stroke-width:2px
    
    class RadarDB,RadarTables database
    class AirbyteServer,SourceMySQL,DestS3,Connection,Workers processing
    class S3Bronze,S3Structure,GlueCrawler,DataCatalog storage
    class Scheduler,DAGTasks,WebServer orchestration
    class Athena,PowerBI,DataScience analytics
    class CloudWatch,Slack,Grafana monitoring
```

## 🔄 Fluxo Detalhado de Sincronização

```mermaid
sequenceDiagram
    participant AF as Airflow Scheduler
    participant AB as Airbyte Server
    participant W as Airbyte Worker
    participant MySQL as MySQL Radar
    participant S3 as S3 Bronze
    participant GC as Glue Crawler
    participant SL as Slack
    
    Note over AF: Daily at 2:00 UTC
    AF->>AB: Trigger Sync (connection_mysql_s3_radar)
    AB->>W: Assign Worker
    
    Note over W: Connection Setup
    W->>MySQL: Test Connection (bi-cognitivo-read)
    MySQL-->>W: Connection OK
    W->>S3: Test S3 Access
    S3-->>W: Access OK
    
    Note over W: Data Extraction Phase
    loop For each table (80+ tables)
        W->>MySQL: SELECT * FROM table_name
        MySQL-->>W: Return data batch
        W->>W: Transform to Parquet
        W->>S3: Upload batch to S3
        Note over S3: Path: table_name/cog_dt_ingestion=YYYY-MM-DD/
    end
    
    W-->>AB: Sync Progress Updates
    AB-->>AF: Job Status Updates
    
    Note over W: Completion Phase
    W->>S3: Finalize all uploads
    W-->>AB: Sync Completed Successfully
    AB-->>AF: Job Finished
    
    Note over AF: Post-Processing
    AF->>GC: Trigger Crawler (optional)
    AF->>SL: Send Success Notification
    
    Note over SL: Team Notification
    SL-->>AF: Notification Sent
    
    rect rgb(200, 255, 200)
        Note over AF,SL: Success Flow Complete
    end
```

## ⚠️ Fluxo de Tratamento de Erros

```mermaid
flowchart TD
    A[Sync Started] --> B{MySQL Connection OK?}
    
    B -->|No| C[Retry Connection<br/>3 attempts]
    C --> D{Retry Successful?}
    D -->|No| E[Send Alert to #alerts<br/>Mark as Failed]
    D -->|Yes| F[Continue to S3 Check]
    
    B -->|Yes| F{S3 Access OK?}
    F -->|No| G[Check AWS Credentials<br/>Test S3 permissions]
    G --> H{S3 Access Fixed?}
    H -->|No| E
    H -->|Yes| I[Start Data Extraction]
    
    F -->|Yes| I[Start Data Extraction]
    
    I --> J{Table Extraction OK?}
    J -->|No| K[Log Table Error<br/>Continue with next table]
    J -->|Yes| L[Transform to Parquet]
    
    K --> M{Critical Table?}
    M -->|Yes| N[Abort Sync<br/>Send Critical Alert]
    M -->|No| O[Log Warning<br/>Continue]
    
    L --> P{Upload to S3 OK?}
    P -->|No| Q[Retry Upload<br/>3 attempts]
    Q --> R{Upload Successful?}
    R -->|No| S[Log Upload Error<br/>Continue if not critical]
    R -->|Yes| T[Mark Table Complete]
    
    P -->|Yes| T[Mark Table Complete]
    
    T --> U{More Tables?}
    U -->|Yes| J
    U -->|No| V[Validate Sync Results]
    
    V --> W{Validation Passed?}
    W -->|No| X[Send Warning<br/>Partial Success]
    W -->|Yes| Y[Send Success Notification]
    
    O --> U
    S --> U
    
    E --> Z[Update Monitoring<br/>Escalate if needed]
    N --> Z
    X --> AA[Log for Investigation]
    Y --> BB[Complete Successfully]
    
    %% Styling
    classDef errorNode fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef successNode fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef warningNode fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef processNode fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    
    class E,N,Z errorNode
    class BB,Y,T successNode
    class X,AA,S warningNode
    class A,I,L,V processNode
```

## 🔧 Fluxo de Configuração e Deploy

```mermaid
gitgraph
    commit id: "Initial Config"
    
    branch feature/radar-optimization
    checkout feature/radar-optimization
    commit id: "Update timeouts"
    commit id: "Optimize Parquet settings"
    commit id: "Add new table mapping"
    
    checkout main
    merge feature/radar-optimization
    
    commit id: "Deploy v1.1" type: HIGHLIGHT
    
    branch hotfix/mysql-connection
    checkout hotfix/mysql-connection
    commit id: "Fix SSL configuration"
    
    checkout main
    merge hotfix/mysql-connection
    
    commit id: "Deploy v1.1.1" type: HIGHLIGHT
    
    branch feature/performance-tuning
    checkout feature/performance-tuning
    commit id: "Increase worker resources"
    commit id: "Add compression optimization"
    commit id: "Implement retry logic"
    
    checkout main
    merge feature/performance-tuning
    
    commit id: "Deploy v1.2" type: HIGHLIGHT
```

## 🏥 Disaster Recovery Flow

```mermaid
flowchart TB
    subgraph "Disaster Detection"
        A[System Health Monitor] --> B{Primary System Down?}
        B -->|Yes| C[Assess Impact Level]
        B -->|No| D[Continue Normal Operations]
    end
    
    subgraph "Impact Assessment"
        C --> E{Critical Service Affected?}
        E -->|MySQL Down| F[MySQL Failover Process]
        E -->|S3 Inaccessible| G[S3 Failover Process]
        E -->|Airbyte Down| H[Airbyte Recovery Process]
        E -->|Network Issues| I[Network Troubleshooting]
    end
    
    subgraph "MySQL Failover"
        F --> F1[Switch to Read Replica]
        F1 --> F2[Update Airbyte Source Config]
        F2 --> F3[Test Connection]
        F3 --> F4{Connection OK?}
        F4 -->|Yes| F5[Resume Operations]
        F4 -->|No| F6[Escalate to DBA Team]
    end
    
    subgraph "S3 Failover"
        G --> G1[Switch to DR Bucket]
        G1 --> G2[Update Destination Config]
        G2 --> G3[Test Write Access]
        G3 --> G4{Access OK?}
        G4 -->|Yes| G5[Resume Operations]
        G4 -->|No| G6[Escalate to Infrastructure]
    end
    
    subgraph "Airbyte Recovery"
        H --> H1[Check Worker Status]
        H1 --> H2[Restart Failed Components]
        H2 --> H3[Verify Configuration]
        H3 --> H4{Service Healthy?}
        H4 -->|Yes| H5[Resume Operations]
        H4 -->|No| H6[Full Service Restart]
    end
    
    subgraph "Recovery Validation"
        F5 --> J[Run Validation Tests]
        G5 --> J
        H5 --> J
        J --> K{All Tests Pass?}
        K -->|Yes| L[Send Recovery Notification]
        K -->|No| M[Continue Troubleshooting]
    end
    
    subgraph "Post-Recovery"
        L --> N[Update Runbooks]
        N --> O[Schedule Post-Mortem]
        O --> P[Implement Improvements]
    end
    
    %% Error Escalation
    F6 --> Q[Page On-Call Engineer]
    G6 --> Q
    M --> Q
    Q --> R[Emergency Response]
    
    %% Styling
    classDef critical fill:#ffebee,stroke:#c62828,stroke-width:3px
    classDef warning fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef process fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    
    class F6,G6,Q,R critical
    class C,E,M warning
    class F5,G5,H5,L success
    class A,J,N,O,P process
```

## 📊 Monitoramento e Alertas

```mermaid
graph TB
    subgraph "Data Sources"
        A1[Airbyte Metrics]
        A2[Airflow Logs]
        A3[MySQL Performance]
        A4[S3 Access Logs]
        A5[Kubernetes Metrics]
    end
    
    subgraph "Monitoring Stack"
        B1[CloudWatch]
        B2[Prometheus]
        B3[Grafana]
        B4[Custom Scripts]
    end
    
    subgraph "Alert Conditions"
        C1[Sync Failure]
        C2[High Duration >90min]
        C3[Data Volume Anomaly >20%]
        C4[Connection Timeout]
        C5[Resource Exhaustion]
    end
    
    subgraph "Notification Channels"
        D1[Slack #data-engineering]
        D2[Slack #alerts]
        D3[Email Alerts]
        D4[PagerDuty]
        D5[SMS Emergency]
    end
    
    subgraph "Alert Routing"
        E1{Alert Severity}
        E1 -->|INFO| D1
        E1 -->|WARNING| D1
        E1 -->|ERROR| D2
        E1 -->|CRITICAL| D4
        E1 -->|EMERGENCY| D5
    end
    
    %% Data Flow
    A1 --> B1
    A2 --> B1
    A3 --> B2
    A4 --> B1
    A5 --> B2
    
    B1 --> B3
    B2 --> B3
    B3 --> B4
    
    B4 --> C1
    B4 --> C2
    B4 --> C3
    B4 --> C4
    B4 --> C5
    
    C1 --> E1
    C2 --> E1
    C3 --> E1
    C4 --> E1
    C5 --> E1
    
    %% Response Actions
    D2 --> F1[Auto-retry Logic]
    D4 --> F2[On-call Engineer]
    D5 --> F3[Emergency Response]
    
    classDef dataSource fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef monitoring fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef alert fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef notification fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef action fill:#ffebee,stroke:#c62828,stroke-width:2px
    
    class A1,A2,A3,A4,A5 dataSource
    class B1,B2,B3,B4 monitoring
    class C1,C2,C3,C4,C5 alert
    class D1,D2,D3,D4,D5 notification
    class F1,F2,F3 action
```

## 🔄 Data Lifecycle Management

```mermaid
timeline
    title Data Lifecycle - Radar Ingestion
    
    section Real-time
        MySQL Production : Active transactions
                         : User interactions
                         : System updates
    
    section Daily Ingestion
        02:00 UTC : Airflow triggers sync
                  : 80+ tables extracted
                  : Data transformed to Parquet
        03:00 UTC : Upload to S3 Bronze
                  : Glue crawler updates schema
                  : Data available for analytics
    
    section Processing
        04:00 UTC : Bronze to Silver transformation
                  : Data quality validation
                  : Business logic application
        06:00 UTC : Silver to Gold aggregation
                  : KPI calculations
                  : Dashboard updates
    
    section Analytics
        08:00 UTC : Reports generation
                  : Executive dashboards refresh
                  : ML pipeline triggers
        
    section Retention
        Day 30    : Move to S3 IA storage class
        Day 90    : Move to Glacier storage
        Day 2555  : Delete per retention policy (7 years)
```

## 🎯 Performance Optimization Flow

```mermaid
flowchart LR
    A[Performance Issue Detected] --> B{Issue Type?}
    
    B -->|Slow Sync| C[MySQL Optimization]
    B -->|High Memory| D[Resource Scaling]
    B -->|Network Latency| E[Network Optimization]
    B -->|S3 Upload Slow| F[Parquet Optimization]
    
    C --> C1[Add Indexes]
    C1 --> C2[Optimize Queries]
    C2 --> C3[Connection Pooling]
    
    D --> D1[Increase Memory Limits]
    D1 --> D2[Horizontal Scaling]
    D2 --> D3[JVM Tuning]
    
    E --> E1[Regional Optimization]
    E1 --> E2[Network Policies]
    E2 --> E3[Connection Multiplexing]
    
    F --> F1[Compression Tuning]
    F1 --> F2[Block Size Optimization]
    F2 --> F3[Parallel Uploads]
    
    C3 --> G[Test Performance]
    D3 --> G
    E3 --> G
    F3 --> G
    
    G --> H{Improvement Achieved?}
    H -->|Yes| I[Document Solution]
    H -->|No| J[Try Alternative Approach]
    
    I --> K[Update Runbooks]
    J --> B
    K --> L[Monitor Continued Performance]
```

---

## 🔍 Troubleshooting Decision Tree

```mermaid
flowchart TD
    Start[Radar Ingestion Issue Reported] --> CheckType{What type of issue?}
    
    CheckType -->|No Data| NoData[Check last successful sync]
    CheckType -->|Slow Performance| SlowPerf[Check resource usage]
    CheckType -->|Partial Data| PartialData[Check error logs]
    CheckType -->|Wrong Data| WrongData[Check source tables]
    
    NoData --> NoDataCheck{Last sync when?}
    NoDataCheck -->|>24h ago| CriticalIssue[CRITICAL: Escalate immediately]
    NoDataCheck -->|Today| CheckSchedule[Verify Airflow schedule]
    
    SlowPerf --> PerfCheck{Duration > 90min?}
    PerfCheck -->|Yes| CheckResources[Check CPU/Memory usage]
    PerfCheck -->|No| NormalVariation[Normal variation - monitor]
    
    PartialData --> LogCheck[Check Airbyte worker logs]
    LogCheck --> TableCheck{Which tables missing?}
    TableCheck -->|Critical tables| CriticalMissing[High priority fix]
    TableCheck -->|Non-critical| LogWarning[Log warning - investigate]
    
    WrongData --> SourceCheck[Validate source MySQL data]
    SourceCheck --> SchemaCheck{Schema changed?}
    SchemaCheck -->|Yes| ResetConnection[Reset Airbyte connection]
    SchemaCheck -->|No| DataIssue[Investigate data quality]
    
    CheckSchedule --> ScheduleOK{Schedule active?}
    ScheduleOK -->|No| EnableSchedule[Enable DAG schedule]
    ScheduleOK -->|Yes| ManualTrigger[Trigger manual sync]
    
    CheckResources --> ResourceOK{Resources sufficient?}
    ResourceOK -->|No| ScaleResources[Scale up resources]
    ResourceOK -->|Yes| NetworkCheck[Check network latency]
    
    %% Resolution paths
    CriticalIssue --> OnCall[Page on-call engineer]
    CriticalMissing --> OnCall
    EnableSchedule --> Monitor[Monitor next execution]
    ManualTrigger --> Monitor
    ScaleResources --> Monitor
    ResetConnection --> Monitor
    
    %% Styling
    classDef critical fill:#ffebee,stroke:#c62828,stroke-width:3px
    classDef warning fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef info fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef success fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    
    class CriticalIssue,OnCall,CriticalMissing critical
    class LogWarning,NormalVariation warning
    class Start,CheckType,NoData,SlowPerf info
    class Monitor,EnableSchedule success
```

---

**📍 Documentos Relacionados:**
- [README Principal](README.md)
- [Fluxo de Ingestão](fluxo_ingestao.md) 
- [Boas Práticas](boas_praticas.md)
- [Erros Comuns](erros_comuns.md)
