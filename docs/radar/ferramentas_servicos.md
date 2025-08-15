# 🛠️ Ferramentas e Serviços do Radar

## 📋 Stack Tecnológico

A ingestão do Radar utiliza uma stack moderna e robusta para garantir alta disponibilidade, performance e confiabilidade no pipeline de dados.

## 🔧 Airbyte - Plataforma de Integração

### **Versão e Configuração**
- **Versão Airbyte**: 0.3.23
- **Source Connector**: airbyte/source-mysql v3.0.0
- **Destination Connector**: airbyte/destination-s3 v0.3.23
- **Deployment**: Kubernetes com Helm Charts

### **Características Principais**

#### **Source MySQL Connector**
```yaml
# Capacidades do Connector
connector_features:
  - name: "Full Refresh Sync"
    supported: true
  - name: "Incremental Sync"
    supported: true
  - name: "Change Data Capture (CDC)"
    supported: true
  - name: "SSL Support"
    supported: true
  - name: "SSH Tunnel"
    supported: true
  - name: "Normalization"
    supported: false
  - name: "DBT Support"
    supported: false
```

#### **Destination S3 Connector**
```yaml
# Formatos Suportados
supported_formats:
  - CSV
  - JSON Lines
  - Parquet (em uso)
  
# Tipos de Compressão
compression_options:
  - No Compression
  - GZIP
  - SNAPPY (em uso)
  - DEFLATE
  - BZIP2
  - XZ
  - ZSTANDARD
```

### **Configuração de Performance**

#### **Resource Requirements**
```yaml
# Configuração de recursos
resource_requirements:
  cpu_limit: "2000m"       # 2 CPU cores
  cpu_request: "500m"      # 0.5 CPU cores
  memory_limit: "4Gi"      # 4GB RAM
  memory_request: "1Gi"    # 1GB RAM
```

#### **Otimizações Parquet**
```yaml
# Configurações otimizadas
parquet_settings:
  page_size_kb: 1024          # 1MB páginas
  block_size_mb: 128          # 128MB blocos
  dictionary_encoding: true    # Ativo para performance
  max_padding_size_mb: 8      # 8MB padding máximo
  dictionary_page_size_kb: 1024 # 1MB dicionário
```

### **Monitoramento Airbyte**

#### **Health Checks**
```bash
# Verificar saúde do servidor Airbyte
curl -f http://airbyte-server:8001/api/v1/health

# Status da conexão específica
curl -X GET "http://airbyte-server:8001/api/v1/connections/6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad" \
  -H "accept: application/json"
```

#### **Métricas de Performance**
- **Connection Success Rate**: Taxa de sucesso das sincronizações
- **Average Sync Duration**: Tempo médio de sincronização
- **Data Volume Transferred**: Volume de dados transferidos
- **Error Rate**: Taxa de erros por período

---

## ⚡ Airflow - Orquestração

### **Versão e Ambiente**
- **Versão**: Apache Airflow 2.8.0
- **Executor**: KubernetesExecutor
- **Scheduler**: 1 instância com HA
- **Workers**: Auto-scaling baseado em carga

### **DAG Principal: dag_sync_airbyte_connections**

#### **Estrutura do DAG**
```python
from airflow import DAG
from airflow.providers.airbyte.operators.airbyte import AirbyteTriggerSyncOperator
from airflow.providers.airbyte.sensors.airbyte import AirbyteJobSensor
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator

# Configuração do DAG
dag = DAG(
    'dag_sync_airbyte_connections',
    description='Sincronização diária das conexões Airbyte',
    schedule_interval='0 2 * * *',  # 2:00 UTC diariamente
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=1,
    default_args={
        'owner': 'data-engineering',
        'depends_on_past': False,
        'email_on_failure': True,
        'email_on_retry': False,
        'retries': 3,
        'retry_delay': timedelta(minutes=10),
        'on_failure_callback': task_fail_slack_alert
    }
)
```

#### **Tasks Principais**

**1. Trigger Sync Radar**
```python
trigger_radar_sync = AirbyteTriggerSyncOperator(
    task_id='trigger_sync_radar',
    airbyte_conn_id='airbyte_default',
    connection_id='6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad',
    asynchronous=True,
    timeout=3600,  # 1 hora
    wait_seconds=30,
    dag=dag
)
```

**2. Monitor Job Status**
```python
wait_sync_radar = AirbyteJobSensor(
    task_id='wait_sync_radar',
    airbyte_conn_id='airbyte_default',
    airbyte_job_id="{{ ti.xcom_pull(task_ids='trigger_sync_radar')['job_id'] }}",
    timeout=3600,
    poke_interval=60,
    dag=dag
)
```

**3. Notification Success**
```python
notify_success = SlackWebhookOperator(
    task_id='notify_success',
    http_conn_id='slack_webhook',
    message=generate_success_message,
    channel='#data-engineering',
    dag=dag
)
```

#### **Fluxo de Dependências**
```python
trigger_radar_sync >> wait_sync_radar >> notify_success
```

### **Configurações de Sistema**

#### **Variables Airflow**
```json
{
  "airbyte_config": {
    "server_host": "airbyte-server",
    "server_port": 8001,
    "api_version": "v1"
  },
  "radar_connection": {
    "connection_id": "6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad",
    "timeout": 3600,
    "retry_attempts": 3
  },
  "notification_config": {
    "slack_channel": "#data-engineering",
    "email_alerts": ["data-team@farmarcas.com"]
  }
}
```

#### **Connections Airflow**
```bash
# Conexão Airbyte
airflow connections add 'airbyte_default' \
  --conn-type 'airbyte' \
  --conn-host 'airbyte-server' \
  --conn-port 8001

# Conexão Slack
airflow connections add 'slack_webhook' \
  --conn-type 'http' \
  --conn-host 'hooks.slack.com' \
  --conn-password 'webhook_token'
```

---

## 🗄️ MySQL Radar - Source Database

### **Configuração do Servidor**

#### **Especificações Técnicas**
```yaml
# Detalhes do RDS MySQL
instance_details:
  engine: MySQL 8.0.35
  instance_class: db.r5.xlarge
  cpu_cores: 4
  memory: 32GB
  storage: 500GB GP2
  multi_az: true
  backup_retention: 7 days
  
# Configuração de Rede
network_config:
  vpc: vpc-farmarcas-production
  subnet_group: db-subnet-private
  security_group: sg-mysql-radar-production
  public_access: false
```

#### **Connection Details**
```yaml
# Configuração de conexão
database_config:
  endpoint: db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com
  port: 3306
  database: radar
  username: bi-cognitivo-read
  ssl_mode: preferred
  charset: utf8mb4
  collation: utf8mb4_unicode_ci
```

### **Permissões e Segurança**

#### **User bi-cognitivo-read**
```sql
-- Permissões do usuário de leitura
GRANT SELECT ON radar.* TO 'bi-cognitivo-read'@'%';
GRANT SHOW VIEW ON radar.* TO 'bi-cognitivo-read'@'%';
GRANT PROCESS ON *.* TO 'bi-cognitivo-read'@'%';
FLUSH PRIVILEGES;
```

#### **SSL Configuration**
```yaml
ssl_settings:
  ssl_ca: rds-ca-2019-root.pem
  ssl_mode: PREFERRED
  ssl_verify_server_cert: false
  require_ssl: true
```

### **Performance e Monitoramento**

#### **Query Performance**
```sql
-- Queries de monitoramento
SELECT 
    table_schema,
    table_name,
    table_rows,
    data_length,
    index_length,
    (data_length + index_length) as total_size
FROM information_schema.tables 
WHERE table_schema = 'radar'
ORDER BY total_size DESC;
```

#### **Connection Monitoring**
```sql
-- Monitorar conexões ativas
SELECT 
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE
FROM information_schema.PROCESSLIST 
WHERE USER = 'bi-cognitivo-read';
```

---

## ☁️ AWS S3 - Data Lake Storage

### **Configuração do Bucket**

#### **Bucket Details**
```yaml
# Configuração S3
bucket_config:
  name: farmarcas-production-bronze
  region: us-east-2
  storage_class: STANDARD
  versioning: Enabled
  encryption: AES-256
  
# Lifecycle Policies
lifecycle_rules:
  - name: "Transition to IA"
    days: 30
    storage_class: STANDARD_IA
  - name: "Transition to Glacier"
    days: 90
    storage_class: GLACIER
  - name: "Delete old versions"
    noncurrent_days: 365
    action: DELETE
```

#### **Path Structure**
```
s3://farmarcas-production-bronze/
└── origin=airbyte/
    └── database=bronze_radar/
        ├── store/
        │   └── cog_dt_ingestion=2024-01-15/
        │       ├── file_store_0.parquet
        │       ├── file_store_1.parquet
        │       └── _metadata.json
        ├── store_metrics/
        │   └── cog_dt_ingestion=2024-01-15/
        ├── user_access/
        │   └── cog_dt_ingestion=2024-01-15/
        └── [80+ tables]/
```

### **Permissões IAM**

#### **Policy para Airbyte**
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
      ],
      "Condition": {
        "StringLike": {
          "s3:prefix": "origin=airbyte/database=bronze_radar/*"
        }
      }
    }
  ]
}
```

### **Monitoramento S3**

#### **Métricas CloudWatch**
```yaml
# Métricas principais
s3_metrics:
  - BucketSizeBytes
  - NumberOfObjects
  - BucketRequests
  - 4xxErrors
  - 5xxErrors
  - FirstByteLatency
  - TotalRequestLatency
```

#### **Alertas Configurados**
```yaml
# CloudWatch Alarms
s3_alarms:
  - name: "S3-HighErrorRate"
    metric: "4xxErrors"
    threshold: 10
    period: 300
    
  - name: "S3-HighLatency"
    metric: "FirstByteLatency"
    threshold: 1000
    period: 300
```

---

## 🔍 AWS Glue - Schema Discovery

### **Crawlers Configurados**

#### **Crawler: crawler-bronze-radar**
```yaml
# Configuração do Crawler
crawler_config:
  name: crawler-bronze-radar
  role: AWSGlueServiceRole-radar
  database: bronze_radar
  schedule: "cron(0 6 * * ? *)"  # 6:00 UTC diariamente
  
# Targets
s3_targets:
  - path: "s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/"
    exclusions:
      - "**/_metadata/**"
      - "**/temp/**"
```

#### **Schema Evolution**
```yaml
# Configurações de schema
schema_settings:
  schema_change_policy:
    update_behavior: "UPDATE_IN_DATABASE"
    delete_behavior: "LOG"
  
  recrawl_policy:
    recrawl_behavior: "CRAWL_EVERYTHING"
  
  classification: "parquet"
```

### **Data Catalog Integration**

#### **Database Structure**
```sql
-- Estrutura no AWS Glue Data Catalog
DATABASE: bronze_radar
├── TABLE: store
├── TABLE: store_metrics
├── TABLE: user_access
├── TABLE: product
├── TABLE: contest
└── [75+ outras tabelas]
```

---

## 📊 Amazon Athena - Query Engine

### **Configuração de Workgroup**

#### **Workgroup: radar-analytics**
```yaml
# Configuração Athena
workgroup_config:
  name: radar-analytics
  description: "Queries para análise dos dados Radar"
  
  # Performance
  result_configuration:
    output_location: "s3://farmarcas-athena-results/radar/"
    encryption_option: "SSE_S3"
  
  # Cost Control
  bytes_scanned_cutoff_per_query: 1073741824  # 1GB
  enforce_workgroup_configuration: true
  publish_cloudwatch_metrics: true
```

#### **Queries Otimizadas**
```sql
-- Exemplo de query otimizada com partições
SELECT 
    store_id,
    COUNT(*) as total_transactions,
    SUM(amount) as total_amount
FROM bronze_radar.store_metrics
WHERE cog_dt_ingestion >= '2024-01-01'
  AND cog_dt_ingestion <= '2024-01-31'
GROUP BY store_id
ORDER BY total_amount DESC;
```

---

## 🔐 Segurança e Compliance

### **Encryption**
- **At Rest**: AES-256 (S3), TDE (MySQL)
- **In Transit**: TLS 1.2+ para todas as conexões
- **Keys**: AWS KMS para gerenciamento

### **Access Control**
- **RBAC**: Role-based access control
- **IAM**: Princípio do menor privilégio
- **Audit**: CloudTrail para todas as operações

### **Data Governance**
- **Lineage**: Tracking via AWS Glue
- **Quality**: Validações automatizadas
- **Retention**: Políticas de lifecycle S3

---

**📍 Próximos Passos:**
- [Pré-requisitos](pre_requisitos.md)
- [Configurações de Exemplo](configuracoes_exemplo.md)
- [Erros Comuns](erros_comuns.md)
