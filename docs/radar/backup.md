# 📊 Documentação da Ingestão do Radar

## 📋 Visão Geral

A **ingestão do Radar** é um processo automatizado que sincroniza dados do sistema Radar (MySQL) para o data lake S3 da Farmarcas. Este sistema é responsável por capturar informações críticas de farmácias, usuários, produtos, campanhas e métricas através de uma pipeline robusta usando Airbyte e Airflow.

> **Sistema**: MySQL Radar → Airbyte → S3 Bronze Layer  
> **Tipo**: Full Refresh (manual schedule via Airflow)  
> **Frequência**: Diária (2h UTC via Airflow orchestrator)  
> **Volume**: 80+ tabelas MySQL com dados críticos do negócio

## 🎯 Objetivo e Importância

### **Por que o Radar é Crítico:**
- 🏪 **Farmácias**: Dados de lojas, status, métricas de performance e documentação
- 👥 **Usuários**: Informações de acesso, permissões e registros de usuários
- 📦 **Produtos**: Catálogo de produtos farmacêuticos, EANs e informações PBM
- 🎯 **Campanhas**: Concursos, objetivos, scores e vouchers
- 📊 **Analytics**: Brand metrics, KPIs e dados para BI
- 🔐 **Compliance**: Documentos, termos legais e auditoria

### **Impacto no Negócio:**
- **Dashboards Executivos**: Métricas de performance das farmácias
- **BI Reports**: Relatórios de campanhas, produtos e usuários  
- **Data Science**: Análise de padrões de comportamento e performance
- **Compliance**: Rastreabilidade de documentos e termos aceitos

## 🏗️ Arquitetura Completa

```mermaid
graph TB
    subgraph "Sistema Radar Production"
        RadarDB[(MySQL Radar<br/>db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com:3306)]
        RadarTables[80+ Tabelas:<br/>• store (farmácias)<br/>• store_metrics (métricas)<br/>• brand_metrics_average<br/>• product (catálogo)<br/>• contest (campanhas)<br/>• user_access (usuários)<br/>• vouchers (benefícios)]
    end
    
    subgraph "Airbyte Platform"
        AirbyteServer[Airbyte Server<br/>v0.3.23]
        SourceRadar[Source MySQL Radar<br/>bi-cognitivo-read user<br/>SSL enabled]
        DestS3[Destination S3<br/>Parquet + SNAPPY]
        Connection[Connection<br/>connection_mysql_s3_radar<br/>Manual Schedule]
    end
    
    subgraph "Airflow Orchestration"
        RadarDAG[DAG: dag_sync_airbyte_connections<br/>Schedule: 0 2 * * *]
        TriggerTask[AirbyteTriggerSyncOperator<br/>Asynchronous Mode]
        SensorTask[AirbyteJobSensor<br/>Timeout: 3600s]
        NotifyTask[Slack Notifications<br/>Success/Failure]
    end
    
    subgraph "AWS S3 Data Lake"
        S3Bronze[s3://farmarcas-production-bronze/<br/>origin=airbyte/database=bronze_radar/]
        DataPartition[table_name/<br/>cog_dt_ingestion=YYYY-MM-DD/<br/>file_table_name_*.parquet]
        Compression[SNAPPY Compression<br/>Metadata Tracking]
    end
    
    subgraph "Downstream Processing"
        Glue[AWS Glue Crawlers<br/>Schema Discovery]
        Athena[Amazon Athena<br/>Query Engine]
        PowerBI[Power BI Reports<br/>Business Analytics]
    end
    
    RadarDB --> RadarTables
    RadarTables --> SourceRadar
    SourceRadar --> Connection
    Connection --> DestS3
    DestS3 --> S3Bronze
    
    RadarDAG --> TriggerTask
    TriggerTask --> AirbyteServer
    AirbyteServer --> Connection
    TriggerTask --> SensorTask
    SensorTask --> NotifyTask
    
    S3Bronze --> DataPartition
    DataPartition --> Compression
    Compression --> Glue
    Glue --> Athena
    Athena --> PowerBI
    
    style RadarDB fill:#e1f5fe
    style S3Bronze fill:#f3e5f5
    style RadarDAG fill:#fff3e0
    style Connection fill:#e8f5e8
```

## 🔧 Componentes Técnicos

### 1. **Source: MySQL Radar**

#### Configuração da Source
```yaml
# sources/source_mysql_radar/configuration.yaml
resource_name: "source_mysql_radar"
definition_type: source
definition_id: 435bb9a5-7887-4809-aa58-28c27df0d7ad
definition_image: airbyte/source-mysql
definition_version: 3.0.0

configuration:
  # Connection Details
  host: db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com
  port: 3306
  database: radar
  username: bi-cognitivo-read
  password: ${RADAR_PASS}
  
  # Security Configuration
  ssl: true
  ssl_mode:
    mode: "preferred"
  
  # Network Configuration
  tunnel_method:
    tunnel_method: "NO_TUNNEL"
  
  # Replication Configuration
  replication_method:
    method: "STANDARD"
    initial_waiting_seconds: 300
```

#### Credenciais e Variáveis
```bash
# Environment Variables
export RADAR_PASS="<secure_password>"

# Verificar conectividade
mysql -h db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com \
      -u bi-cognitivo-read -p${RADAR_PASS} \
      -D radar --ssl-mode=PREFERRED
```

### 2. **Connection: MySQL → S3 Radar**

#### Configuração da Connection
```yaml
# connections/connection_mysql_s3_radar/configuration.yaml
resource_name: "connection_mysql_s3_radar"
definition_type: connection
source_configuration_path: sources/source_mysql_radar/configuration.yaml
destination_configuration_path: destinations/destination_s3_radar/configuration.yaml

configuration:
  status: active
  skip_reset: false
  namespace_definition: destination
  namespace_format: "${SOURCE_NAMESPACE}"
  prefix: ""
  
  # Scheduling: Manual (via Airflow)
  schedule_type: manual
  
  # Resource Requirements (customizable)
  resource_requirements:
    cpu_limit: ""      # Default: unbounded
    cpu_request: ""    # Default: unbounded
    memory_limit: ""   # Default: unbounded
    memory_request: "" # Default: unbounded
  
  # Sync Configuration
  sync_catalog:
    streams:
      # Exemplo: store - Dados das farmácias
      - config:
          alias_name: store
          cursor_field: []
          destination_sync_mode: append
          primary_key: [["Id"]]
          selected: true
          suggested: false
          sync_mode: full_refresh
        stream:
          name: store
          namespace: radar
          json_schema:
            properties:
              Id: {airbyte_type: integer, type: number}
              CNPJ: {type: string}
              Company_Name: {type: string}
              Name: {type: string}
              Status: {airbyte_type: integer, type: number}
              Created_At: {airbyte_type: timestamp_without_timezone, format: date-time, type: string}
              Modified_At: {airbyte_type: timestamp_without_timezone, format: date-time, type: string}
              # ... demais campos
              
      # Exemplo: brand_metrics_average - Métricas das marcas
      - config:
          alias_name: brand_metrics_average
          cursor_field: []
          destination_sync_mode: append
          primary_key: [["Id"]]
          selected: true
          sync_mode: full_refresh
        stream:
          name: brand_metrics_average
          namespace: radar
          json_schema:
            properties:
              Id: {airbyte_type: integer, type: number}
              Id_Store: {airbyte_type: integer, type: number}
              Average_Clients_Number: {type: string}
              Average_Ticket: {type: string}
              CMV: {type: string}
              Profit: {type: string}
              Revenues: {type: string}
              # ... demais campos
```

### 3. **Destination: S3 Radar**

#### Configuração do Destination
```yaml
# destinations/destination_s3_radar/configuration.yaml
resource_name: "destination_s3_radar"
definition_type: destination
definition_id: 4816b78f-1489-44c1-9060-4b19d5fa9362
definition_image: airbyte/destination-s3
definition_version: 0.3.23

configuration:
  # S3 Bucket Configuration
  s3_bucket_name: farmarcas-production-bronze
  s3_bucket_path: "origin=airbyte/database=bronze_radar"
  s3_bucket_region: us-east-2
  s3_path_format: ${STREAM_NAME}/cog_dt_ingestion=${YEAR}-${MONTH}-${DAY}/file_${STREAM_NAME}
  
  # AWS Credentials
  access_key_id: ${FARMARCAS_AWS_ACCESS_KEY_ID}
  secret_access_key: ${FARMARCAS_AWS_SECRET_ACCESS_KEY}
  
  # Format Configuration
  format:
    format_type: "Parquet"
    compression_codec: "SNAPPY"
    compression:
      compression_type: "No Compression"
```

#### Estrutura de Arquivos S3
```bash
# Estrutura final no S3
s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/
├── store/
│   └── cog_dt_ingestion=2025-08-07/
│       ├── file_store_part_0.parquet
│       ├── file_store_part_1.parquet
│       └── _SUCCESS
├── brand_metrics_average/
│   └── cog_dt_ingestion=2025-08-07/
│       ├── file_brand_metrics_average_part_0.parquet
│       └── _SUCCESS
├── product/
│   └── cog_dt_ingestion=2025-08-07/
│       ├── file_product_part_0.parquet
│       └── _SUCCESS
# ... demais tabelas
```

### 4. **Airflow DAG de Orquestração**

#### DAG: dag_sync_airbyte_connections
```python
from airflow import DAG
from airflow.providers.airbyte.operators.airbyte import AirbyteTriggerSyncOperator
from airflow.providers.airbyte.sensors.airbyte import AirbyteJobSensor
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=15),
    'email_on_failure': True,
    'email_on_retry': False
}

dag = DAG(
    'dag_sync_airbyte_connections',
    default_args=default_args,
    description='Sync all Airbyte connections including RADAR',
    schedule_interval='0 2 * * *',  # Daily at 2 AM UTC
    catchup=False,
    max_active_runs=1,
    tags=['airbyte', 'radar', 'data-ingestion']
)

# Trigger RADAR sync
trigger_radar_sync = AirbyteTriggerSyncOperator(
    task_id='trigger_radar_sync',
    airbyte_conn_id='airbyte_default',
    connection_id='connection_mysql_s3_radar',
    asynchronous=True,
    timeout=30,
    dag=dag
)

# Wait for RADAR sync completion
wait_radar_sync = AirbyteJobSensor(
    task_id='wait_radar_sync',
    airbyte_conn_id='airbyte_default',
    airbyte_job_id="{{ task_instance.xcom_pull(task_ids='trigger_radar_sync') }}",
    timeout=3600,  # 1 hour timeout
    poke_interval=60,  # Check every minute
    dag=dag
)

# Success notification
radar_success_slack = SlackWebhookOperator(
    task_id='radar_success_notification',
    http_conn_id='slack_webhook',
    message=":white_check_mark: RADAR sync completed successfully!",
    channel='#data-engineering-alerts',
    dag=dag,
    trigger_rule='none_failed_min_one_success'
)

# Failure notification
radar_failure_slack = SlackWebhookOperator(
    task_id='radar_failure_notification',
    http_conn_id='slack_webhook',
    message=":x: RADAR sync failed! Please check logs.",
    channel='#data-engineering-alerts',
    dag=dag,
    trigger_rule='one_failed'
)

trigger_radar_sync >> wait_radar_sync
wait_radar_sync >> [radar_success_slack, radar_failure_slack]
```

## 📊 Dados Sincronizados

### Principais Tabelas (80+ no total)

#### 🏪 **Core Business - Farmácias e Lojas**
```yaml
Tables:
  store:
    description: "Dados principais das farmácias"
    fields: ["Id", "CNPJ", "Company_Name", "Name", "Status", "Brand", "Created_At"]
    volume: "~5000 registros"
    
  store_metrics:
    description: "Métricas de performance das lojas"
    fields: ["Id", "Id_Store", "Revenue", "Profit", "Tickets", "Created_At"]
    volume: "~15000 registros"
    
  brand_metrics_average:
    description: "Médias de métricas por marca"
    fields: ["Id", "Id_Store", "Average_Ticket", "CMV", "Profit", "Revenues"]
    volume: "~2000 registros"
    
  economic_group:
    description: "Grupos econômicos e cartões"
    fields: ["Id", "Name", "Card_Number", "Id_CardHolder"]
    volume: "~500 registros"
```

#### 👥 **Usuários e Acesso**
```yaml
Tables:
  user_access:
    description: "Controle de último acesso dos usuários"
    fields: ["Id_User", "LastAccess_At", "Sent_Email", "Status_Changed"]
    volume: "~8000 registros"
    
  role_permission:
    description: "Permissões por role/função"
    fields: ["Id", "Id_Role", "Id_Permission", "Is_Default"]
    volume: "~200 registros"
    
  user_permission:
    description: "Permissões específicas de usuários"
    fields: ["Id", "Id_User", "Id_Permission", "Id_User_Change"]
    volume: "~1000 registros"
    
  login:
    description: "Dados de autenticação e recovery"
    fields: ["Id_User", "Password", "Recovery_Id", "Refresh_Token"]
    volume: "~8000 registros"
```

#### 📦 **Produtos e PBM**
```yaml
Tables:
  product:
    description: "Catálogo de produtos farmacêuticos"
    fields: ["Id", "EAN", "Name", "Active_Ingredient", "Presentation"]
    volume: "~50000 registros"
    
  product_pbm:
    description: "Programas PBM e descontos"
    fields: ["Id", "Ean", "Discount_Pf", "Discount_Pmc", "Partner"]
    volume: "~5000 registros"
    
  product_export:
    description: "Dados de produtos para exportação"
    fields: ["Id", "EAN", "Nome", "Fabricante", "Genérico"]
    volume: "~50000 registros"
```

#### 🎯 **Campanhas e Gamificação**
```yaml
Tables:
  objectives:
    description: "Objetivos e metas configuradas"
    fields: ["Id", "Type", "Value_Reward", "Schedule", "Description"]
    volume: "~100 registros"
    
  contest:
    description: "Concursos/campanhas ativas"
    fields: ["Id", "Name", "Start", "End", "Image", "Last_Contemplation"]
    volume: "~50 registros"
    
  contest_score:
    description: "Pontuações dos usuários nos concursos"
    fields: ["Id_Contest", "Id_Store", "Id_User", "Score"]
    volume: "~10000 registros"
    
  vouchers:
    description: "Vouchers e benefícios emitidos"
    fields: ["Id", "Id_Store", "Id_Objective", "Amount", "Id_Distributor_User"]
    volume: "~2000 registros"
```

#### 📋 **Documentos e Compliance**
```yaml
Tables:
  store_document:
    description: "Documentos das lojas (validação, expiração)"
    fields: ["Id", "Id_Store", "Filename", "Status", "Valid_To", "No_Expire"]
    volume: "~5000 registros"
    
  document_request:
    description: "Solicitações de documentos"
    fields: ["Id", "Id_User", "Template", "Data", "Sended_At"]
    volume: "~1000 registros"
    
  legal_terms_users:
    description: "Termos legais aceitos pelos usuários"
    fields: ["Id_User", "Id_Term", "Created_At"]
    volume: "~3000 registros"
```

#### 🎪 **Outros Módulos Importantes**
```yaml
Tables:
  distributor:
    description: "Distribuidores e fornecedores"
    fields: ["Id", "Name", "CNPJ", "Email", "Type"]
    volume: "~200 registros"
    
  menu:
    description: "Estrutura de menus do sistema"
    fields: ["Id", "Name", "Url", "Id_Area", "External"]
    volume: "~100 registros"
    
  lives:
    description: "Lives e eventos transmitidos"
    fields: ["Id", "Title", "Start_Live", "End_Live", "Event_Id"]
    volume: "~50 registros"
```

### Schema Detalhado - Tabela `store`
```sql
-- Estrutura principal da tabela store
CREATE TABLE store (
    Id INTEGER PRIMARY KEY,                    -- ID único da loja
    CNPJ VARCHAR(20),                         -- CNPJ da farmácia
    Company_Name VARCHAR(255),                -- Razão social
    Name VARCHAR(255),                        -- Nome fantasia
    Status INTEGER,                           -- Status ativo/inativo
    Brand INTEGER,                            -- ID da marca/rede
    Business_Model INTEGER,                   -- Modelo de negócio
    Created_At TIMESTAMP,                     -- Data de criação
    Modified_At TIMESTAMP,                    -- Última modificação
    Disabled_At TIMESTAMP,                    -- Data de desativação
    Branch VARCHAR(255),                      -- Filial
    Email VARCHAR(255),                       -- Email principal
    Phone VARCHAR(50),                        -- Telefone
    Cellphone VARCHAR(50),                    -- Celular
    IE VARCHAR(50),                          -- Inscrição estadual
    TaxRegime INTEGER,                       -- Regime tributário
    Type INTEGER,                            -- Tipo de estabelecimento
    Opened_At TIMESTAMP,                     -- Data de abertura
    Farmarcas_Name VARCHAR(255),             -- Nome no sistema Farmarcas
    Group_Name VARCHAR(255),                 -- Nome do grupo
    Grouped_At TIMESTAMP,                    -- Data de agrupamento
    -- ... demais campos específicos
);
```

## ⚙️ Pré-requisitos Técnicos

### 1. **Permissões e Acessos**

#### AWS S3 Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-production-bronze",
        "arn:aws:s3:::farmarcas-production-bronze/*"
      ]
    }
  ]
}
```

#### MySQL Database Permissions
```sql
-- Usuário: bi-cognitivo-read
GRANT SELECT ON radar.* TO 'bi-cognitivo-read'@'%';
GRANT SHOW VIEW ON radar.* TO 'bi-cognitivo-read'@'%';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'bi-cognitivo-read'@'%';
FLUSH PRIVILEGES;

-- Verificar permissões
SHOW GRANTS FOR 'bi-cognitivo-read'@'%';
```

### 2. **Credenciais e Variáveis de Ambiente**

#### Environment Variables
```bash
# Airbyte Environment
export RADAR_PASS="<mysql_password>"
export FARMARCAS_AWS_ACCESS_KEY_ID="<aws_access_key>"
export FARMARCAS_AWS_SECRET_ACCESS_KEY="<aws_secret_key>"

# Airflow Connections
airflow connections add 'airbyte_default' \
    --conn-type 'Airbyte' \
    --conn-host 'airbyte-server' \
    --conn-port '8001'

airflow connections add 'slack_webhook' \
    --conn-type 'HTTP' \
    --conn-host 'hooks.slack.com' \
    --conn-password '<slack_webhook_token>'
```

#### Secrets Management
```yaml
# Using AWS Secrets Manager
apiVersion: v1
kind: Secret
metadata:
  name: radar-credentials
  namespace: airbyte
type: Opaque
data:
  mysql-password: <base64_encoded_password>
  aws-access-key: <base64_encoded_access_key>
  aws-secret-key: <base64_encoded_secret_key>
```

### 3. **Estrutura de Pastas**

#### Airbyte Configuration Structure
```bash
airbyte/
├── sources/
│   └── source_mysql_radar/
│       └── configuration.yaml
├── destinations/
│   └── destination_s3_radar/
│       └── configuration.yaml
├── connections/
│   └── connection_mysql_s3_radar/
│       ├── configuration.yaml
│       └── state_*.yaml
└── octavia/
    └── project.yaml
```

#### Airflow DAGs Structure  
```bash
airflow/
├── dags/
│   ├── dag_sync_airbyte_connections.py
│   └── dag_radar_specific.py
├── plugins/
│   └── airbyte_operators/
├── variables/
│   ├── airbyte_config.json
│   └── radar_config.json
└── requirements.txt
```

## 🛠️ Ferramentas e Serviços

### **Stack Tecnológico**
```yaml
Infrastructure:
  AWS:
    - RDS MySQL (Source Database)
    - S3 (Data Lake Bronze)
    - IAM (Permissions Management)
    - VPC (Network Security)
  
  Kubernetes:
    - Airbyte Platform (Data Integration)
    - Airflow (Orchestration)
    - Monitoring (Prometheus + Grafana)
  
  Tools:
    - Octavia CLI (Airbyte Config Management)
    - DBeaver (Database Management)
    - AWS CLI (S3 Operations)
    - kubectl (Kubernetes Management)
```

### **Versions e Compatibilidade**
```yaml
Components:
  airbyte/source-mysql: "3.0.0"
  airbyte/destination-s3: "0.3.23" 
  apache-airflow: "2.5.0"
  mysql-server: "8.0"
  python: "3.9+"
  
Dependencies:
  airbyte-python-connector: ">=0.2.0"
  apache-airflow-providers-airbyte: ">=3.0.0"
  boto3: ">=1.20.0"
  mysql-connector-python: ">=8.0.0"
```
```
    initial_waiting_seconds: 300
    
  ssl_mode:
    mode: preferred
    
  jdbc_url_params: "allowPublicKeyRetrieval=true&useSSL=false"
```

#### Principais Tabelas Sincronizadas
| Tabela | Descrição | Sync Mode | Primary Key |
|--------|-----------|-----------|-------------|
| `transacoes` | Transações de vendas | Incremental | `[id]` |
| `store_metrics` | Métricas de lojas | Full Refresh | `[Id]` |
| `product_export` | Catálogo de produtos | Full Refresh | `[]` |
| `contest_ganhadores` | Vencedores de promoções | Full Refresh | `[]` |
| `distributor_user` | Usuários distribuidores | Incremental | `[Id]` |
| `lives` | Transmissões ao vivo | Full Refresh | `[Id]` |
| `store_import` | Dados de importação lojas | Incremental | `[Id]` |

### 2. **Connection: MySQL → S3 Radar**

#### Configuração da Connection
```yaml
definition_type: connection
source_id: mysql_radar_source_id
destination_id: s3_radar_destination_id

configuration:
  status: active
  
  sync_catalog:
    streams:
      - stream:
          name: transacoes
          json_schema: {...}
        config:
          sync_mode: incremental
          cursor_field: ["data_transacao"]
          destination_sync_mode: append_dedup
          primary_key: [["id"]]
          
      - stream:
          name: store_metrics
          json_schema: {...}
        config:
          sync_mode: full_refresh
          destination_sync_mode: overwrite
  
  schedule_type: cron
  cron_expression: "0 1 * * *"  # Daily at 1 AM UTC
  
  resource_requirements:
    cpu_request: "0.5"
    cpu_limit: "1.0"
    memory_request: "1Gi"
    memory_limit: "2Gi"
```

### 3. **Destination: S3 Bronze Layer**

#### Configuração S3
```yaml
configuration:
  s3_bucket_name: farmarcas-production-bronze
  s3_bucket_path: airbyte/radar
  s3_bucket_region: us-east-2
  
  access_key_id: ${FARMARCAS_AWS_ACCESS_KEY_ID}
  secret_access_key: ${FARMARCAS_AWS_SECRET_ACCESS_KEY}
  
  format:
    format_type: Parquet
    compression_codec: GZIP
    
  s3_path_format: ${YEAR}/${MONTH}/${DAY}/${stream_name}
  s3_filename_pattern: ${STREAM_NAME}_${YEAR}_${MONTH}_${DAY}_${EPOCH}_part_${PART_NUMBER}
```

### 4. **Airflow DAG de Orquestração**

#### DAG: sync_connection_mysql_s3_radar
```python
from airflow import DAG
from airflow.providers.airbyte.operators.airbyte import AirbyteTriggerSyncOperator
from airflow.providers.airbyte.sensors.airbyte import AirbyteJobSensor
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=15)
}

dag = DAG(
    'sync_connection_mysql_s3_radar',
    default_args=default_args,
    description='Sincronização diária Radar → S3',
    schedule_interval='0 1 * * *',  # 1 AM UTC daily
    catchup=False,
    max_active_runs=1
)

# Trigger RADAR sync
trigger_radar_sync = AirbyteTriggerSyncOperator(
    task_id='trigger_radar_sync',
    airbyte_conn_id='airbyte_default',
    connection_id='connection_mysql_s3_radar',
    asynchronous=True,
    dag=dag
)

# Wait for RADAR sync completion
wait_radar_sync = AirbyteJobSensor(
    task_id='wait_radar_sync',
    airbyte_conn_id='airbyte_default',
    airbyte_job_id="{{ task_instance.xcom_pull(task_ids='trigger_radar_sync') }}",
    timeout=3600,  # 1 hour timeout
    poke_interval=60,  # Check every minute
    dag=dag
)

trigger_radar_sync >> wait_radar_sync
```

## ⚙️ Configuração e Setup

### Pré-requisitos

#### 1. **Credenciais de Banco**
```bash
# MySQL Radar Database
HOST: radar-host.com
PORT: 3306
DATABASE: radar_db
USERNAME: airbyte_user
PASSWORD: ${RADAR_PASS}  # Stored in Airbyte secrets
```

#### 2. **Permissões MySQL**
```sql
-- Permissões necessárias para CDC
GRANT SELECT ON radar_db.* TO 'airbyte_user'@'%';
GRANT REPLICATION SLAVE ON *.* TO 'airbyte_user'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'airbyte_user'@'%';

-- Verificar binlog habilitado
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
```

#### 3. **Credenciais AWS**
```bash
# S3 Access
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET=farmarcas-production-bronze
S3_REGION=us-east-2
```

### Comandos de Configuração

#### Setup via Octavia CLI
```bash
# Configurar source
octavia apply sources/source_mysql_radar

# Configurar destination
octavia apply destinations/destination_s3_radar

# Configurar connection
octavia apply connections/connection_mysql_s3_radar

# Verificar status
octavia list connections
```

#### Verificação da Configuração
```bash
# Testar conectividade MySQL
mysql -h radar-host.com -u airbyte_user -p -e "SELECT 1"

# Testar conectividade S3
aws s3 ls s3://farmarcas-production-bronze/airbyte/radar/

# Verificar status Airbyte
curl -X GET "http://airbyte-server:8001/api/v1/health"
```

## 📊 Estrutura de Dados

### Schema das Principais Tabelas

#### Tabela: transacoes
```json
{
  "properties": {
    "id": {"type": "number", "airbyte_type": "integer"},
    "data_transacao": {"type": "string", "format": "date-time"},
    "valor_total": {"type": "number"},
    "id_loja": {"type": "number", "airbyte_type": "integer"},
    "id_produto": {"type": "number", "airbyte_type": "integer"},
    "quantidade": {"type": "number"},
    "desconto": {"type": "number"},
    "status": {"type": "string"}
  }
}
```

#### Tabela: store_metrics
```json
{
  "properties": {
    "Id": {"type": "number", "airbyte_type": "integer"},
    "Nome_Loja": {"type": "string"},
    "CNPJ": {"type": "string"},
    "Cidade": {"type": "string"},
    "UF": {"type": "string"},
    "Rede": {"type": "string"},
    "Score": {"type": "number"},
    "Data_Atualizacao": {"type": "string", "format": "date-time"}
  }
}
```

#### Tabela: product_export
```json
{
  "properties": {
    "Id": {"type": "number", "airbyte_type": "integer"},
    "Nome": {"type": "string"},
    "EAN": {"type": "string"},
    "Fabricante": {"type": "string"},
    "Principio_Ativo": {"type": "string"},
    "Apresentacao": {"type": "string"},
    "Tipo": {"type": "string"},
    "Registro_MS": {"type": "string"},
    "Generico": {"type": "string"}
  }
}
```

### Estrutura S3 Output

```
s3://farmarcas-production-bronze/airbyte/radar/
├── 2025/
│   └── 08/
│       └── 07/
│           ├── transacoes/
│           │   ├── transacoes_2025_08_07_1691234567_part_0.parquet
│           │   └── transacoes_2025_08_07_1691234567_part_1.parquet
│           ├── store_metrics/
│           │   └── store_metrics_2025_08_07_1691234567_part_0.parquet
│           ├── product_export/
│           │   └── product_export_2025_08_07_1691234567_part_0.parquet
│           └── distributor_user/
│               └── distributor_user_2025_08_07_1691234567_part_0.parquet
```

## 📋 Operação e Monitoramento

### Execução Diária

#### 1. **Schedule Automático**
- **Horário**: 1h UTC (22h BRT do dia anterior)
- **Frequência**: Diária
- **Duração Média**: 45-60 minutos
- **Volume**: ~2-5GB de dados processados

#### 2. **Fluxo de Execução**
1. **01:00 UTC**: Airflow trigger DAG
2. **01:01 UTC**: Airbyte inicia sincronização
3. **01:05 UTC**: CDC captura mudanças desde última sync
4. **01:15 UTC**: Full refresh de tabelas configuradas
5. **01:45 UTC**: Upload de arquivos Parquet para S3
6. **02:00 UTC**: Finalização e logs de sucesso

### Monitoramento

#### Métricas-Chave
```bash
# Status da sincronização
octavia list connections | grep radar

# Logs detalhados
octavia logs connections/connection_mysql_s3_radar

# Verificar volume de dados
aws s3 ls --recursive --human-readable \
  s3://farmarcas-production-bronze/airbyte/radar/$(date +%Y/%m/%d)/
```

#### Dashboards e Alertas
- **Airbyte UI**: Status em tempo real das sincronizações
- **Airflow UI**: Logs e execução das DAGs
- **CloudWatch**: Métricas de volume e performance
- **DataDog**: Alertas de falha ou atraso

### Health Checks

#### Verificações Diárias
```bash
# 1. Conectividade MySQL
mysql -h radar-host.com -u airbyte_user -p -e "SELECT COUNT(*) FROM transacoes WHERE DATE(data_transacao) = CURDATE()"

## ✅ Validação e Testes

### **Testes de Conectividade**
```bash
# 1. Test MySQL connection
mysql -h db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com \
      -u bi-cognitivo-read -p${RADAR_PASS} \
      -D radar -e "SELECT COUNT(*) FROM store;" --ssl-mode=PREFERRED

# 2. Test S3 access  
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/

# 3. Test Airbyte API
curl -X GET "http://airbyte-server:8001/api/v1/health"

# 4. Verify recent data
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/ \
    --recursive --human-readable | grep $(date +%Y-%m-%d)
```

### **Data Quality Validation**
```python
# data_quality_check.py
import boto3
import pandas as pd
from datetime import datetime

def validate_radar_data(date_str):
    s3 = boto3.client('s3')
    bucket = 'farmarcas-production-bronze'
    
    # Check critical tables exist
    critical_tables = ['store', 'product', 'user_access', 'brand_metrics_average']
    
    for table in critical_tables:
        prefix = f"origin=airbyte/database=bronze_radar/{table}/cog_dt_ingestion={date_str}/"
        
        response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
        
        if 'Contents' not in response:
            print(f"❌ Missing data for table: {table}")
            return False
        else:
            file_count = len(response['Contents'])
            print(f"✅ {table}: {file_count} files found")
    
    return True

# Execute validation
if __name__ == "__main__":
    today = datetime.now().strftime('%Y-%m-%d')
    is_valid = validate_radar_data(today)
    
    if is_valid:
        print("✅ All critical tables validated successfully")
    else:
        print("❌ Data validation failed")
        exit(1)
```

### **Performance Benchmarks**
```yaml
# Expected performance metrics
Performance_Targets:
  sync_duration: "< 2 hours"
  data_volume: "~500MB-2GB total"
  table_count: "80+ tables"
  
  Critical_Tables:
    store: "< 30 minutes, ~5000 records"
    product: "< 45 minutes, ~50000 records"
    brand_metrics_average: "< 15 minutes, ~2000 records"
    user_access: "< 20 minutes, ~8000 records"
    
  S3_Files:
    format: "Parquet with SNAPPY compression"
    size_range: "1MB - 100MB per file"
    partitioning: "By ingestion date (YYYY-MM-DD)"
```

---

## 📚 Documentos Relacionados

- [ACODE Redundância](../acode_redundancia/README.md) - Sistema de backup e redundância
- [Google Drive Collector](../google_drive/README.md) - Ingestão de dados do Google Drive  
- [Airbyte Platform](../../documentacao_gerada/application-system/AIRBYTE.md) - Documentação completa do Airbyte
- [Airflow Orchestration](../../documentacao_gerada/application-system/AIRFLOW.md) - Setup e configuração do Airflow

**Última Atualização**: 07/08/2025 - Baseado nas configurações reais de produção
# 1. Refresh schema
octavia apply connections/connection_mysql_s3_radar --refresh-schema

# 2. Verificar breaking changes
octavia diff connections/connection_mysql_s3_radar

# 3. Reconfigurar streams se necessário
octavia edit connections/connection_mysql_s3_radar
```

### 4. **S3 Permission Denied**
```bash
# Problema: Falha no upload para S3
# Causa: Credenciais AWS inválidas ou políticas IAM

# Solução:
# 1. Verificar credenciais AWS
aws sts get-caller-identity

# 2. Testar permissões S3
aws s3 cp test.txt s3://farmarcas-production-bronze/airbyte/radar/test.txt

# 3. Verificar políticas IAM
aws iam get-user-policy --user-name airbyte-user --policy-name S3Access
```

### 5. **Large Table Sync Issues**
```bash
# Problema: Timeout em tabelas grandes (> 1M registros)
# Causa: Configuração de recursos insuficiente

# Solução:
# 1. Aumentar recursos da connection
octavia edit connections/connection_mysql_s3_radar
# Aumentar memory_limit para 4Gi e cpu_limit para 2.0

# 2. Implementar particionamento por data
# Adicionar cursor_field baseado em data para incremental sync

# 3. Considerar full_refresh → incremental
# Para tabelas muito grandes que não mudam frequentemente
```

## 🔄 Troubleshooting Avançado

### Debug Logs
```bash
# Logs detalhados Airbyte
kubectl logs -n airbyte -l app=airbyte-worker --tail=1000

# Logs da DAG Airflow
airflow logs show sync_connection_mysql_s3_radar trigger_radar_sync 2025-08-07

# Logs S3 uploads
aws logs tail /aws/s3/farmarcas-production-bronze --since 1h --follow
```

### Performance Tuning
```yaml
# Otimizações de performance
configuration:
  resource_requirements:
    cpu_request: "1.0"
    cpu_limit: "2.0" 
    memory_request: "2Gi"
    memory_limit: "4Gi"
    
  normalization:
    option: basic  # Desabilitar para melhor performance
    
  sync_mode_overrides:
    # Para tabelas que não mudam muito
    store_metrics: 
      sync_mode: full_refresh
      schedule_override: "0 2 * * 0"  # Semanal aos domingos
```

## 📈 Métricas e KPIs

### Operational Metrics
- **Sync Success Rate**: > 95% de sincronizações bem-sucedidas
- **Average Sync Duration**: < 60 minutos
- **Data Volume**: 2-5GB por dia
- **Records Processed**: ~1M+ registros por sync
- **Error Rate**: < 5% de falhas

### Business Metrics
- **Data Freshness**: Dados com menos de 2 horas de atraso
- **Coverage**: 100% das tabelas críticas sincronizadas
- **Quality Score**: Validação de integridade > 98%
- **Availability**: 99.5% de uptime da pipeline

## 👥 Responsáveis

### Equipe de Data Engineering
- **Tech Lead**: [Nome] - Arquitetura e estratégia
- **DevOps**: [Nome] - Infraestrutura e deploy  
- **Analytics Engineer**: [Nome] - Modelagem e qualidade
- **On-call**: Plantão 24/7 via PagerDuty

### Contatos de Suporte
- **Slack**: #data-engineering-radar
- **Email**: data-radar@farmarcas.com
- **Escalation**: CTO ou Head of Data
- **Documentação**: Confluence - Radar Data Pipeline

---

## 📚 Documentação Modular

Esta documentação está organizada em módulos especializados para facilitar a navegação e manutenção:

### **🔄 Processos Técnicos**
- **[Fluxo de Ingestão](fluxo_ingestao.md)** - Pipeline completo do Radar (fonte → destino)
- **[Ferramentas e Serviços](ferramentas_servicos.md)** - Airbyte, Airflow, S3, MySQL e integração
- **[Diagrama de Fluxo](diagrama_fluxo.md)** - Visualizações Mermaid do sistema completo

### **⚙️ Configuração e Setup**
- **[Pré-requisitos](pre_requisitos.md)** - Credenciais, permissões e conectividade
- **[Configurações de Exemplo](configuracoes_exemplo.md)** - YAMLs comentados e scripts de teste

### **🛠️ Operação e Manutenção**
- **[Erros Comuns](erros_comuns.md)** - Diagnóstico e soluções de problemas frequentes
- **[Boas Práticas](boas_praticas.md)** - Operação, monitoramento e otimização

### **📊 Informações do Sistema**

#### **Especificações Técnicas**
```yaml
sistema_radar:
  fonte:
    tipo: "MySQL 8.0.35"
    host: "db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com"
    porta: 3306
    database: "radar"
    usuario: "bi-cognitivo-read"
    ssl: true
    
  plataforma:
    airbyte_version: "0.3.23"
    source_connector: "airbyte/source-mysql:3.0.0"
    destination_connector: "airbyte/destination-s3:0.3.23"
    
  destino:
    tipo: "AWS S3"
    bucket: "farmarcas-production-bronze"
    caminho: "origin=airbyte/database=bronze_radar"
    formato: "Parquet + SNAPPY"
    
  orquestração:
    ferramenta: "Apache Airflow"
    dag: "dag_sync_airbyte_connections"
    schedule: "0 2 * * *"  # 2:00 UTC diariamente
    
  monitoramento:
    alertas: "Slack (#data-engineering, #alerts)"
    metricas: "CloudWatch + Grafana"
    logs: "Kubernetes + Airbyte native"
```

#### **Impacto no Negócio**
- **📊 Dashboards Executivos**: Performance de farmácias e KPIs
- **🎯 Campanhas**: Análise de concursos, vouchers e engajamento
- **👥 Gestão de Usuários**: Controle de acesso e documentação
- **📦 Catálogo de Produtos**: EANs, PBM e categorização
- **📈 Analytics**: Brand metrics e indicadores de performance
- **🔐 Compliance**: Rastreabilidade e auditoria (LGPD, SOX)

#### **SLA e Disponibilidade**
- **Disponibilidade**: 99.5% mensal
- **RTO (Recovery Time Objective)**: 4 horas
- **RPO (Recovery Point Objective)**: 24 horas
- **Janela de Execução**: 2:00-3:00 UTC (23:00-00:00 BRT)
- **Duração Típica**: 45-60 minutos para 80+ tabelas

---

## 🚨 Suporte e Contatos

### **Escalação de Problemas**

#### **Nível 1 - Operação**
- **Slack**: `#data-engineering`
- **Escopo**: Monitoramento, validação, troubleshooting básico
- **SLA**: Resposta em 30 minutos (horário comercial)

#### **Nível 2 - Engenharia**
- **Slack**: `#alerts` + `@data-engineering-oncall`
- **Escopo**: Problemas técnicos, configuração, performance
- **SLA**: Resposta em 15 minutos (24/7)

#### **Nível 3 - Infraestrutura**
- **PagerDuty**: Escalação automática
- **Escopo**: Falhas críticas, disaster recovery
- **SLA**: Resposta em 5 minutos (24/7)

### **Runbooks Rápidos**

#### **Problemas Comuns - Soluções Imediatas**
```bash
# MySQL não conecta
kubectl logs -n data-platform -l app=airbyte-worker | grep "connection_mysql_s3_radar"
mysql -h db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com -u bi-cognitivo-read -p

# Sync travada há >90 minutos
curl -X POST "http://airbyte-server:8001/api/v1/connections/6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad/reset"

# S3 access denied
aws sts get-caller-identity
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/

# Airflow DAG não executa
airflow dags state dag_sync_airbyte_connections
airflow dags trigger dag_sync_airbyte_connections
```

### **Documentação Relacionada**
- **Airbyte Official**: https://docs.airbyte.com/
- **Airflow Documentation**: https://airflow.apache.org/docs/
- **AWS S3 Best Practices**: https://docs.aws.amazon.com/s3/
- **MySQL Connector**: https://docs.airbyte.com/integrations/sources/mysql

---

**🔗 Links Rápidos:**
- [Dashboard Grafana](http://grafana.farmarcas.internal/d/radar-ingestion)
- [Airflow Web UI](http://airflow.farmarcas.internal/dags/dag_sync_airbyte_connections)
- [Airbyte Console](http://airbyte.farmarcas.internal/connections/6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad)
- [S3 Console](https://s3.console.aws.amazon.com/s3/buckets/farmarcas-production-bronze)

---

**📅 Última Atualização**: Janeiro 2024  
**📝 Versão da Documentação**: v2.0  
**👥 Responsáveis**: Data Engineering Team  
**🔄 Próxima Revisão**: Março 2024
