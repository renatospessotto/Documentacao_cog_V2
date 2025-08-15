# 🎯 Boas Práticas - Ingestão do Radar

## 📋 Visão Geral

Este documento apresenta as melhores práticas para operação, manutenção e otimização da ingestão de dados do Radar, baseadas em experiências reais e lições aprendidas.

## 🚀 Práticas Operacionais

### **1. Monitoramento Proativo**

#### **Dashboard de Saúde**
```yaml
# Métricas essenciais para monitorar
key_metrics:
  availability:
    - sync_success_rate: "> 95%"
    - sync_duration: "< 60 minutes"
    - connection_uptime: "> 99%"
  
  performance:
    - mysql_query_time: "< 30 seconds"
    - s3_upload_speed: "> 10 MB/s"
    - memory_usage: "< 80%"
    - cpu_usage: "< 70%"
  
  data_quality:
    - record_count_variance: "< 5%"
    - schema_drift_alerts: "immediate"
    - null_value_percentage: "< 1%"
```

#### **Alertas Configurados**
```python
# Configuração de alertas no Airflow
ALERT_CONFIG = {
    'sync_failure': {
        'channels': ['#alerts', '#data-engineering'],
        'escalation': '15 minutes',
        'severity': 'HIGH'
    },
    'performance_degradation': {
        'threshold': '50% slower than baseline',
        'channels': ['#data-engineering'],
        'severity': 'MEDIUM'
    },
    'data_anomaly': {
        'threshold': 'Record count deviation > 20%',
        'channels': ['#data-quality'],
        'severity': 'HIGH'
    }
}
```

### **2. Janela de Manutenção**

#### **Horários Otimizados**
```yaml
# Schedule otimizado
maintenance_windows:
  daily_sync:
    time: "02:00 UTC"  # 23:00 BRT
    duration: "60 minutes"
    rationale: "Menor carga no MySQL de produção"
  
  weekly_maintenance:
    day: "Sunday"
    time: "01:00 UTC"
    duration: "2 hours"
    activities:
      - schema_validation
      - performance_tuning
      - log_cleanup
  
  monthly_review:
    schedule: "First Saturday"
    activities:
      - capacity_planning
      - cost_optimization
      - security_review
```

#### **Procedimentos de Manutenção**
```bash
#!/bin/bash
# weekly_maintenance.sh

echo "Starting weekly Radar maintenance..."

# 1. Cleanup de logs antigos
echo "Cleaning up old logs..."
find /opt/airflow/logs -name "*.log" -mtime +30 -delete
kubectl logs -n data-platform --since=24h airbyte-worker > /dev/null

# 2. Verificação de performance
echo "Performance check..."
mysql -h $RADAR_HOST -u $RADAR_USER -p$RADAR_PASS $RADAR_DATABASE \
  -e "ANALYZE TABLE store, user_access, product, contest;"

# 3. Validação de schema
echo "Schema validation..."
python /scripts/validate_schema.py

# 4. S3 lifecycle check
echo "S3 lifecycle validation..."
aws s3api get-bucket-lifecycle-configuration \
  --bucket farmarcas-production-bronze \
  --region us-east-2

# 5. Backup de configurações
echo "Backing up configurations..."
kubectl get configmap radar-ingestion-config -n data-platform -o yaml > backup_config_$(date +%Y%m%d).yaml

echo "Weekly maintenance completed!"
```

### **3. Versionamento e Deploy**

#### **GitOps Workflow**
```yaml
# .github/workflows/radar-deploy.yml
name: Deploy Radar Configuration

on:
  push:
    branches: [main]
    paths: ['radar/configs/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Validate YAML
        run: |
          find radar/configs -name "*.yaml" -exec yamllint {} \;
      
      - name: Test MySQL Connection
        run: |
          mysql -h $RADAR_HOST -u $RADAR_USER -p$RADAR_PASS \
                -e "SELECT 'Connection OK';"
  
  deploy:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - name: Update Airbyte Configuration
        run: |
          kubectl apply -f radar/configs/ -n data-platform
          kubectl rollout restart deployment/airbyte-worker -n data-platform
```

#### **Controle de Versão**
```bash
# Estrutura de versionamento
radar/
├── configs/
│   ├── v1.0/
│   │   ├── source_mysql_radar.yaml
│   │   ├── destination_s3_radar.yaml
│   │   └── connection_mysql_s3_radar.yaml
│   ├── v1.1/  # Nova versão
│   └── current -> v1.1/  # Symlink para versão atual
├── scripts/
│   ├── deploy.sh
│   ├── rollback.sh
│   └── validate.sh
└── docs/
    └── CHANGELOG.md
```

---

## 🔒 Segurança e Compliance

### **1. Gestão de Credenciais**

#### **Rotação Automática**
```yaml
# Política de rotação de credenciais
credential_rotation:
  mysql_password:
    frequency: "90 days"
    automation: "AWS Secrets Manager"
    notification: "7 days before expiry"
  
  aws_access_keys:
    frequency: "60 days"
    automation: "IAM role rotation"
    monitoring: "CloudTrail logs"
  
  slack_tokens:
    frequency: "180 days"
    manual_process: true
    reminder: "14 days before expiry"
```

#### **Least Privilege Access**
```sql
-- Permissões mínimas para bi-cognitivo-read
GRANT SELECT ON radar.* TO 'bi-cognitivo-read'@'%';
GRANT SHOW VIEW ON radar.* TO 'bi-cognitivo-read'@'%';
GRANT PROCESS ON *.* TO 'bi-cognitivo-read'@'%';

-- Revogar permissões desnecessárias
REVOKE INSERT, UPDATE, DELETE ON radar.* FROM 'bi-cognitivo-read'@'%';
REVOKE CREATE, DROP, ALTER ON *.* FROM 'bi-cognitivo-read'@'%';
```

#### **Auditoria de Acesso**
```bash
#!/bin/bash
# audit_access.sh - Auditoria de acessos

echo "MySQL Access Audit - $(date)"
echo "================================"

# 1. Verificar conexões ativas
mysql -h $RADAR_HOST -u admin_user -p \
  -e "
  SELECT 
      USER,
      HOST,
      DB,
      COMMAND,
      TIME,
      STATE,
      INFO
  FROM information_schema.PROCESSLIST 
  WHERE USER = 'bi-cognitivo-read';"

# 2. S3 Access Logs
aws logs filter-log-events \
  --log-group-name '/aws/s3/farmarcas-production-bronze' \
  --start-time $(date -d '24 hours ago' +%s)000 \
  --filter-pattern '[timestamp, request_id, remote_ip, requester, request_id, operation, bucket, key="*bronze_radar*"]'

# 3. Airbyte Access Logs
kubectl logs -n data-platform -l app=airbyte-server --since=24h | grep "connection_mysql_s3_radar"
```

### **2. Compliance e Governance**

#### **Data Lineage**
```python
# data_lineage_tracker.py
class RadarDataLineage:
    def track_ingestion(self, execution_date):
        """Registra linhagem dos dados ingeridos"""
        lineage_record = {
            'source': {
                'system': 'MySQL Radar',
                'host': 'db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com',
                'database': 'radar',
                'tables': self.get_synced_tables(),
                'extraction_time': execution_date
            },
            'pipeline': {
                'tool': 'Airbyte',
                'connection_id': '6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad',
                'transformation': 'MySQL -> Parquet',
                'validation': self.get_validation_results()
            },
            'destination': {
                'system': 'AWS S3',
                'bucket': 'farmarcas-production-bronze',
                'path': 'origin=airbyte/database=bronze_radar',
                'format': 'Parquet/SNAPPY',
                'partition': f'cog_dt_ingestion={execution_date}'
            },
            'metadata': {
                'pipeline_version': '1.0',
                'data_classification': 'Internal',
                'retention_policy': '7 years',
                'compliance_tags': ['LGPD', 'SOX']
            }
        }
        
        # Salvar no Data Catalog
        self.save_to_datacatalog(lineage_record)
```

#### **Data Quality Gates**
```python
# quality_gates.py
class RadarQualityGates:
    def __init__(self):
        self.quality_rules = {
            'store': {
                'mandatory_fields': ['Id', 'Name', 'Status'],
                'null_tolerance': 0.01,  # 1% máximo de nulls
                'duplicate_tolerance': 0,
                'volume_variance': 0.05  # 5% de variação aceitável
            },
            'user_access': {
                'mandatory_fields': ['Id', 'User_Id', 'Store_Id'],
                'referential_integrity': {
                    'Store_Id': 'store.Id'
                }
            }
        }
    
    def validate_ingestion(self, table_name, df):
        """Valida qualidade dos dados ingeridos"""
        rules = self.quality_rules.get(table_name, {})
        violations = []
        
        # Check mandatory fields
        for field in rules.get('mandatory_fields', []):
            if field not in df.columns:
                violations.append(f"Missing mandatory field: {field}")
        
        # Check null tolerance
        null_tolerance = rules.get('null_tolerance', 0)
        for col in df.select_dtypes(include=['object']).columns:
            null_rate = df[col].isnull().sum() / len(df)
            if null_rate > null_tolerance:
                violations.append(f"High null rate in {col}: {null_rate:.2%}")
        
        return violations
```

---

## 📈 Performance e Otimização

### **1. Otimização de Queries MySQL**

#### **Indexes Recomendados**
```sql
-- Indexes para otimizar extração
-- (executar no MySQL de produção com cuidado)

-- Para tabela store
CREATE INDEX idx_store_created_at ON store(Created_At);
CREATE INDEX idx_store_status ON store(Status);

-- Para tabela user_access
CREATE INDEX idx_user_access_created_at ON user_access(Created_At);
CREATE INDEX idx_user_access_store_id ON user_access(Store_Id);

-- Para tabela store_metrics
CREATE INDEX idx_store_metrics_date ON store_metrics(Metric_Date);

-- Verificar performance das queries
EXPLAIN SELECT * FROM store WHERE Created_At >= '2024-01-01';
EXPLAIN SELECT * FROM user_access WHERE Store_Id IN (1,2,3,4,5);
```

#### **Query Optimization**
```python
# Otimização de queries para Airbyte
class QueryOptimizer:
    @staticmethod
    def get_optimized_query(table_name, batch_size=10000):
        """Gera queries otimizadas para cada tabela"""
        
        if table_name == 'store_metrics':
            # Para tabelas grandes, usar ordenação por PK
            return f"""
            SELECT * FROM {table_name} 
            WHERE Id > ? 
            ORDER BY Id 
            LIMIT {batch_size}
            """
        
        elif table_name in ['store', 'user_access']:
            # Para tabelas com timestamp, usar particionamento temporal
            return f"""
            SELECT * FROM {table_name} 
            WHERE Created_At >= ? AND Created_At < ?
            ORDER BY Id
            """
        
        else:
            # Query padrão para tabelas menores
            return f"SELECT * FROM {table_name} ORDER BY Id"
```

### **2. Otimização S3 e Parquet**

#### **Configuração Avançada Parquet**
```yaml
# destination_s3_radar/configuration.yaml - Configuração otimizada
format:
  format_type: "Parquet"
  
  # Otimizações para diferentes tipos de tabela
  compression_codec: "SNAPPY"  # Balance speed/compression
  
  # Para tabelas grandes (>1M records)
  page_size_kb: 2048      # 2MB pages para melhor compressão
  block_size_mb: 256      # 256MB row groups para analytics
  
  # Para tabelas médias (100K-1M records)  
  page_size_kb: 1024      # 1MB pages
  block_size_mb: 128      # 128MB row groups
  
  # Para tabelas pequenas (<100K records)
  page_size_kb: 512       # 512KB pages
  block_size_mb: 64       # 64MB row groups
  
  # Otimizações gerais
  dictionary_encoding: true
  max_padding_size_mb: 8
  dictionary_page_size_kb: 1024
```

#### **Particionamento Inteligente**
```python
# partition_strategy.py
class PartitionStrategy:
    @staticmethod
    def get_partition_config(table_name, estimated_size_gb):
        """Estratégia de particionamento baseada no tamanho"""
        
        if estimated_size_gb > 10:  # Tabelas grandes
            return {
                'partition_cols': ['cog_dt_ingestion', 'cog_hour'],
                'file_size_mb': 128,
                'max_files_per_partition': 100
            }
        elif estimated_size_gb > 1:  # Tabelas médias
            return {
                'partition_cols': ['cog_dt_ingestion'],
                'file_size_mb': 64,
                'max_files_per_partition': 50
            }
        else:  # Tabelas pequenas
            return {
                'partition_cols': ['cog_dt_ingestion'],
                'file_size_mb': 32,
                'max_files_per_partition': 10
            }
```

### **3. Resource Management**

#### **Auto-scaling Configuration**
```yaml
# kubernetes/radar-airbyte-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: airbyte-worker-radar
spec:
  replicas: 2  # Base replicas
  template:
    spec:
      containers:
      - name: airbyte-worker
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "4000m"
            memory: "8Gi"
        env:
        - name: SYNC_JOB_MAX_ATTEMPTS
          value: "3"
        - name: SYNC_JOB_MAX_TIMEOUT_DAYS
          value: "1"

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: airbyte-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: airbyte-worker-radar
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 🔄 Disaster Recovery

### **1. Backup Strategy**

#### **Configurações Automatizadas**
```bash
#!/bin/bash
# backup_radar_configs.sh

BACKUP_DIR="/backups/radar/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

echo "Backing up Radar configurations..."

# 1. Backup Airbyte configurations
kubectl get configmap -n data-platform -o yaml > $BACKUP_DIR/configmaps.yaml
kubectl get secret -n data-platform -o yaml > $BACKUP_DIR/secrets.yaml

# 2. Backup connection configurations
cp -r /airbyte/configs/connections/connection_mysql_s3_radar $BACKUP_DIR/
cp -r /airbyte/configs/sources/source_mysql_radar $BACKUP_DIR/
cp -r /airbyte/configs/destinations/destination_s3_radar $BACKUP_DIR/

# 3. Backup Airflow DAGs
cp /opt/airflow/dags/dag_sync_airbyte_connections.py $BACKUP_DIR/

# 4. Export Airflow variables and connections
airflow variables export $BACKUP_DIR/airflow_variables.json
airflow connections export $BACKUP_DIR/airflow_connections.json

# 5. Backup para S3
aws s3 sync $BACKUP_DIR s3://farmarcas-backups/radar/configurations/$(date +%Y%m%d)/ \
  --region us-east-2

echo "Backup completed: $BACKUP_DIR"
```

#### **Recovery Procedures**
```bash
#!/bin/bash
# restore_radar_configs.sh

BACKUP_DATE=${1:-$(date +%Y%m%d)}
BACKUP_DIR="/backups/radar/$BACKUP_DATE"

echo "Restoring Radar configurations from $BACKUP_DATE..."

# 1. Download from S3 if needed
if [[ ! -d "$BACKUP_DIR" ]]; then
    aws s3 sync s3://farmarcas-backups/radar/configurations/$BACKUP_DATE/ $BACKUP_DIR \
      --region us-east-2
fi

# 2. Restore Kubernetes resources
kubectl apply -f $BACKUP_DIR/configmaps.yaml -n data-platform
kubectl apply -f $BACKUP_DIR/secrets.yaml -n data-platform

# 3. Restore Airbyte configurations
cp -r $BACKUP_DIR/connection_mysql_s3_radar /airbyte/configs/connections/
cp -r $BACKUP_DIR/source_mysql_radar /airbyte/configs/sources/
cp -r $BACKUP_DIR/destination_s3_radar /airbyte/configs/destinations/

# 4. Restart Airbyte services
kubectl rollout restart deployment/airbyte-server -n data-platform
kubectl rollout restart deployment/airbyte-worker -n data-platform

# 5. Restore Airflow configurations
airflow variables import $BACKUP_DIR/airflow_variables.json
airflow connections import $BACKUP_DIR/airflow_connections.json

echo "Restore completed successfully!"
```

### **2. Business Continuity**

#### **Failover Procedures**
```python
# failover_manager.py
class RadarFailoverManager:
    def __init__(self):
        self.primary_endpoint = "db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com"
        self.failover_endpoint = "db-mysql-radar-replica.cxsfxyp2ge90.us-east-2.rds.amazonaws.com"
        self.s3_primary = "farmarcas-production-bronze"
        self.s3_failover = "farmarcas-dr-bronze"
    
    def check_primary_health(self):
        """Verifica saúde do sistema primário"""
        try:
            # Test MySQL connectivity
            conn = mysql.connector.connect(
                host=self.primary_endpoint,
                user='bi-cognitivo-read',
                timeout=30
            )
            conn.close()
            
            # Test S3 accessibility
            s3 = boto3.client('s3')
            s3.head_bucket(Bucket=self.s3_primary)
            
            return True
        except Exception as e:
            logger.error(f"Primary system health check failed: {e}")
            return False
    
    def initiate_failover(self):
        """Inicia procedimento de failover"""
        logger.warning("Initiating failover procedure...")
        
        # 1. Update Airbyte source configuration
        self.update_source_config(host=self.failover_endpoint)
        
        # 2. Update S3 destination
        self.update_destination_config(bucket=self.s3_failover)
        
        # 3. Restart Airbyte services
        self.restart_airbyte_services()
        
        # 4. Send notifications
        self.send_failover_notification()
    
    def validate_failover(self):
        """Valida se o failover foi bem-sucedido"""
        try:
            # Test connection with failover endpoint
            self.test_sync_connection()
            return True
        except Exception as e:
            logger.error(f"Failover validation failed: {e}")
            return False
```

---

## 📊 Métricas e KPIs

### **1. SLAs e Métricas**

#### **Service Level Objectives**
```yaml
radar_slos:
  availability:
    target: "99.5%"
    measurement: "Monthly uptime"
    
  performance:
    sync_duration:
      target: "< 60 minutes"
      p95_target: "< 90 minutes"
    
    data_freshness:
      target: "< 4 hours"
      measurement: "Time from source update to S3 availability"
  
  reliability:
    sync_success_rate:
      target: "> 95%"
      measurement: "Successful syncs / Total syncs"
    
    data_quality:
      target: "> 99%"
      measurement: "Records passing quality checks"
```

#### **Monitoring Dashboard**
```python
# monitoring_dashboard.py
class RadarMonitoringDashboard:
    def get_key_metrics(self, period='24h'):
        """Coleta métricas principais do período"""
        return {
            'sync_metrics': {
                'total_syncs': self.count_syncs(period),
                'successful_syncs': self.count_successful_syncs(period),
                'failed_syncs': self.count_failed_syncs(period),
                'success_rate': self.calculate_success_rate(period)
            },
            'performance_metrics': {
                'avg_sync_duration': self.get_avg_duration(period),
                'p95_sync_duration': self.get_p95_duration(period),
                'data_volume_gb': self.get_data_volume(period)
            },
            'system_health': {
                'mysql_connection_time': self.test_mysql_latency(),
                's3_upload_speed': self.test_s3_speed(),
                'airbyte_cpu_usage': self.get_airbyte_cpu_usage(),
                'airbyte_memory_usage': self.get_airbyte_memory_usage()
            }
        }
```

---

**📍 Documentos Relacionados:**
- [Diagrama de Fluxo](diagrama_fluxo.md)
- [Erros Comuns](erros_comuns.md)
- [README Principal](README.md)
