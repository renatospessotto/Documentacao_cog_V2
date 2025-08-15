# ⚙️ Configurações de Exemplo - Radar

## 📋 Visão Geral

Este documento apresenta exemplos completos e comentados de todas as configurações necessárias para a ingestão do Radar, incluindo arquivos YAML, scripts de deployment e configurações de ambiente.

## 🔧 Airbyte Configurations

### **1. Source MySQL Configuration**

#### **source_mysql_radar/configuration.yaml**
```yaml
# Configuration for airbyte/source-mysql
# Documentation: https://docs.airbyte.com/integrations/sources/mysql
resource_name: "source_mysql_radar"
definition_type: source
definition_id: 435bb9a5-7887-4809-aa58-28c27df0d7ad
definition_image: airbyte/source-mysql
definition_version: 3.0.0

configuration:
  # Database Connection Settings
  host: db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com
  port: 3306
  database: radar
  username: bi-cognitivo-read
  password: ${RADAR_PASS}  # Environment variable for security
  
  # SSL Configuration
  ssl: true
  ssl_mode:
    mode: "preferred"  # Options: disabled, preferred, required, verify_ca, verify_identity
  
  # Replication Method
  replication_method:
    method: "STANDARD"  # Standard replication for full refresh
    server_time_zone: "America/Sao_Paulo"  # Optional: configure timezone
    initial_waiting_seconds: 300  # Wait time before sync starts
  
  # Connection Security
  tunnel_method:
    tunnel_method: "NO_TUNNEL"  # No SSH tunnel required in VPC
  
  # Optional JDBC Parameters
  # jdbc_url_params: "useSSL=true&requireSSL=true&serverTimezone=America/Sao_Paulo"
```

#### **Exemplo de Teste de Conexão**
```bash
#!/bin/bash
# test_mysql_connection.sh

# Variáveis de ambiente
export RADAR_HOST="db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com"
export RADAR_USER="bi-cognitivo-read"
export RADAR_PASS="your_password_here"
export RADAR_DB="radar"

# Teste de conectividade
echo "Testando conexão MySQL Radar..."
mysql -h $RADAR_HOST -P 3306 -u $RADAR_USER -p$RADAR_PASS $RADAR_DB \
    --ssl-mode=PREFERRED \
    --connect-timeout=30 \
    -e "
    SELECT 
        'Connection successful' as status,
        VERSION() as mysql_version,
        DATABASE() as current_database,
        USER() as current_user,
        NOW() as server_time;
    
    -- Verificar tabelas disponíveis
    SELECT 
        COUNT(*) as total_tables
    FROM information_schema.tables 
    WHERE table_schema = '$RADAR_DB';
    
    -- Verificar algumas tabelas principais
    SELECT 
        table_name,
        table_rows,
        ROUND(data_length/1024/1024, 2) as size_mb
    FROM information_schema.tables 
    WHERE table_schema = '$RADAR_DB'
      AND table_name IN ('store', 'user_access', 'product', 'contest')
    ORDER BY data_length DESC;
    "
```

### **2. Destination S3 Configuration**

#### **destination_s3_radar/configuration.yaml**
```yaml
# Configuration for airbyte/destination-s3
# Documentation: https://docs.airbyte.com/integrations/destinations/s3
resource_name: "destination_s3_radar"
definition_type: destination
definition_id: 4816b78f-1489-44c1-9060-4b19d5fa9362
definition_image: airbyte/destination-s3
definition_version: 0.3.23

configuration:
  # AWS S3 Settings
  s3_bucket_name: farmarcas-production-bronze
  s3_bucket_region: us-east-2
  s3_bucket_path: "origin=airbyte/database=bronze_radar"
  
  # AWS Credentials (using environment variables)
  access_key_id: ${FARMARCAS_AWS_ACCESS_KEY_ID}
  secret_access_key: ${FARMARCAS_AWS_SECRET_ACCESS_KEY}
  
  # File Organization
  s3_path_format: "${STREAM_NAME}/cog_dt_ingestion=${YEAR}-${MONTH}-${DAY}/file_${STREAM_NAME}"
  # Results in: store/cog_dt_ingestion=2024-01-15/file_store_0001.parquet
  
  # Data Format Configuration
  format:
    format_type: "Parquet"
    
    # Compression Settings
    compression_codec: "SNAPPY"  # Balance between compression ratio and speed
    
    # Performance Tuning
    page_size_kb: 1024       # 1MB pages for optimal compression
    block_size_mb: 128       # 128MB row groups for good I/O performance
    dictionary_encoding: true # Enable dictionary encoding for repeated values
    max_padding_size_mb: 8   # Maximum padding for row group alignment
    dictionary_page_size_kb: 1024  # Dictionary page size
  
  # Optional: Custom file naming pattern
  # file_name_pattern: "radar_{timestamp}_{part_number}"
  
  # Optional: S3 endpoint override (for testing)
  # s3_endpoint: "http://localhost:9000"  # For MinIO testing
```

#### **Exemplo de Teste S3**
```bash
#!/bin/bash
# test_s3_access.sh

# Variáveis de ambiente
export AWS_ACCESS_KEY_ID="${FARMARCAS_AWS_ACCESS_KEY_ID}"
export AWS_SECRET_ACCESS_KEY="${FARMARCAS_AWS_SECRET_ACCESS_KEY}"
export AWS_DEFAULT_REGION="us-east-2"

# Variáveis do bucket
BUCKET="farmarcas-production-bronze"
PREFIX="origin=airbyte/database=bronze_radar"

echo "Testando acesso S3..."

# 1. Listar bucket
echo "1. Verificando acesso ao bucket..."
aws s3 ls s3://$BUCKET/ --region $AWS_DEFAULT_REGION

# 2. Testar escrita
echo "2. Testando permissão de escrita..."
echo "Test file content - $(date)" > /tmp/test_radar.txt
aws s3 cp /tmp/test_radar.txt s3://$BUCKET/$PREFIX/test/test_radar.txt --region $AWS_DEFAULT_REGION

# 3. Verificar se foi criado
echo "3. Verificando arquivo criado..."
aws s3 ls s3://$BUCKET/$PREFIX/test/ --region $AWS_DEFAULT_REGION

# 4. Testar leitura
echo "4. Testando permissão de leitura..."
aws s3 cp s3://$BUCKET/$PREFIX/test/test_radar.txt /tmp/test_radar_downloaded.txt --region $AWS_DEFAULT_REGION
cat /tmp/test_radar_downloaded.txt

# 5. Limpar arquivo de teste
echo "5. Limpando arquivo de teste..."
aws s3 rm s3://$BUCKET/$PREFIX/test/test_radar.txt --region $AWS_DEFAULT_REGION

# 6. Listar estrutura atual (se existir)
echo "6. Estrutura atual do diretório radar:"
aws s3 ls s3://$BUCKET/$PREFIX/ --recursive --human-readable --summarize

echo "Teste S3 concluído!"
```

### **3. Connection Configuration**

#### **connection_mysql_s3_radar/configuration.yaml (resumido)**
```yaml
# Configuration for connection connection_mysql_s3_radar
definition_type: connection
resource_name: "connection_mysql_s3_radar"
source_configuration_path: sources/source_mysql_radar/configuration.yaml
destination_configuration_path: destinations/destination_s3_radar/configuration.yaml

configuration:
  # Connection Status
  status: active  # active | inactive | deprecated
  
  # Sync Configuration
  schedule_type: manual  # Controlled by Airflow
  skip_reset: false      # Allow reset after updates
  
  # Namespace Configuration
  namespace_definition: destination  # Use destination namespace
  namespace_format: "${SOURCE_NAMESPACE}"
  prefix: ""  # No prefix for table names
  
  # Resource Allocation
  resource_requirements:
    cpu_limit: "2000m"     # 2 CPU cores max
    cpu_request: "500m"    # 0.5 CPU cores min
    memory_limit: "4Gi"    # 4GB RAM max
    memory_request: "1Gi"  # 1GB RAM min
  
  # Table Sync Configuration (exemplo de algumas tabelas)
  sync_catalog:
    streams:
      # Tabela de lojas (exemplo)
      - config:
          alias_name: store
          cursor_field: []
          destination_sync_mode: append
          primary_key: [["Id"]]
          selected: true
          sync_mode: full_refresh
        stream:
          name: store
          namespace: radar
          json_schema:
            type: object
            properties:
              Id:
                type: number
                airbyte_type: integer
              Name:
                type: string
              Status:
                type: string
              Created_At:
                type: string
                format: date-time
                airbyte_type: timestamp_without_timezone
      
      # Tabela de métricas (exemplo)
      - config:
          alias_name: store_metrics
          cursor_field: []
          destination_sync_mode: append
          primary_key: [["Id"]]
          selected: true
          sync_mode: full_refresh
        stream:
          name: store_metrics
          namespace: radar
          # Schema seria definido automaticamente pelo Airbyte
```

---

## 🚀 Airflow Configurations

### **1. DAG Principal**

#### **dag_sync_airbyte_connections.py**
```python
"""
DAG para sincronização das conexões Airbyte
Executa diariamente às 2:00 UTC a ingestão do Radar
"""

from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.airbyte.operators.airbyte import AirbyteTriggerSyncOperator
from airflow.providers.airbyte.sensors.airbyte import AirbyteJobSensor
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
from airflow.models import Variable
import json

# Configurações do DAG
DEFAULT_ARGS = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=10),
    'max_active_runs': 1
}

# Carregar configurações das variáveis Airflow
airbyte_config = Variable.get("airbyte_config", deserialize_json=True)
radar_config = Variable.get("radar_connection", deserialize_json=True)
notification_config = Variable.get("notification_config", deserialize_json=True)

def task_fail_slack_alert(context):
    """Callback para notificações de falha"""
    slack_msg = f"""
    🚨 *Radar Sync Failed*
    
    • Task: {context.get('task_instance').task_id}
    • DAG: {context.get('task_instance').dag_id}
    • Execution Date: {context.get('execution_date')}
    • Try Number: {context.get('task_instance').try_number}
    • Max Tries: {context.get('task_instance').max_tries}
    • Error: {str(context.get('exception'))[:500]}
    • Log URL: {context.get('task_instance').log_url}
    """
    
    send_slack_notification(slack_msg, channel='#alerts')

def generate_success_message(context):
    """Gera mensagem de sucesso dinâmica"""
    ti = context['task_instance']
    execution_date = context['execution_date'].strftime('%Y-%m-%d')
    
    return f"""
    ✅ *Radar Sync Completed Successfully*
    
    📊 **Execution Details:**
    • Date: {execution_date}
    • Duration: {ti.duration} seconds
    • Start Time: {ti.start_date}
    • End Time: {ti.end_date}
    
    📁 **Data Location:**
    • S3 Path: s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/
    • Partition: cog_dt_ingestion={execution_date}
    
    🔗 **Quick Links:**
    • [Airbyte Connection](http://airbyte-server:8001/connections/{radar_config['connection_id']})
    • [Airflow DAG](http://airflow-webserver:8080/dags/dag_sync_airbyte_connections)
    """

# Definição do DAG
dag = DAG(
    'dag_sync_airbyte_connections',
    default_args=DEFAULT_ARGS,
    description='Sincronização diária das conexões Airbyte (Radar)',
    schedule_interval='0 2 * * *',  # 2:00 UTC diariamente
    catchup=False,
    max_active_runs=1,
    tags=['airbyte', 'radar', 'ingestion', 'daily'],
    on_failure_callback=task_fail_slack_alert
)

# Task 1: Trigger Radar Sync
trigger_radar_sync = AirbyteTriggerSyncOperator(
    task_id='trigger_sync_radar',
    airbyte_conn_id='airbyte_default',
    connection_id=radar_config['connection_id'],
    asynchronous=True,
    timeout=radar_config['timeout'],
    wait_seconds=30,
    dag=dag
)

# Task 2: Wait for Sync Completion
wait_sync_radar = AirbyteJobSensor(
    task_id='wait_sync_radar',
    airbyte_conn_id='airbyte_default',
    airbyte_job_id="{{ ti.xcom_pull(task_ids='trigger_sync_radar')['job_id'] }}",
    timeout=radar_config['timeout'],
    poke_interval=60,  # Check every minute
    dag=dag
)

# Task 3: Validate Data (optional)
def validate_radar_data(**context):
    """Validação básica dos dados ingeridos"""
    import boto3
    from datetime import datetime
    
    s3 = boto3.client('s3', region_name='us-east-2')
    bucket = 'farmarcas-production-bronze'
    prefix = f"origin=airbyte/database=bronze_radar/"
    
    # Data de hoje
    today = context['execution_date'].strftime('%Y-%m-%d')
    
    # Verificar se existem objetos para hoje
    response = s3.list_objects_v2(
        Bucket=bucket,
        Prefix=f"{prefix}store/cog_dt_ingestion={today}/"
    )
    
    if 'Contents' not in response:
        raise ValueError(f"Nenhum arquivo encontrado para {today}")
    
    # Contar arquivos por tabela (exemplo básico)
    tables_to_check = ['store', 'user_access', 'product', 'contest']
    file_counts = {}
    
    for table in tables_to_check:
        table_response = s3.list_objects_v2(
            Bucket=bucket,
            Prefix=f"{prefix}{table}/cog_dt_ingestion={today}/"
        )
        file_counts[table] = len(table_response.get('Contents', []))
    
    context['task_instance'].xcom_push(key='file_counts', value=file_counts)
    return f"Validation completed: {file_counts}"

validate_data = PythonOperator(
    task_id='validate_radar_data',
    python_callable=validate_radar_data,
    dag=dag
)

# Task 4: Success Notification
notify_success = SlackWebhookOperator(
    task_id='notify_success',
    http_conn_id='slack_webhook',
    message=generate_success_message,
    channel=notification_config['slack_channel'],
    dag=dag
)

# Definir dependências
trigger_radar_sync >> wait_sync_radar >> validate_data >> notify_success
```

### **2. Variáveis Airflow**

#### **airbyte_config.json**
```json
{
  "server_host": "airbyte-server",
  "server_port": 8001,
  "api_version": "v1",
  "default_timeout": 3600,
  "retry_attempts": 3,
  "health_check_endpoint": "/api/v1/health"
}
```

#### **radar_connection.json**
```json
{
  "connection_id": "6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad",
  "source_id": "source_mysql_radar",
  "destination_id": "destination_s3_radar",
  "timeout": 3600,
  "retry_attempts": 3,
  "expected_tables": [
    "store",
    "store_metrics",
    "user_access",
    "product",
    "contest",
    "voucher"
  ]
}
```

#### **notification_config.json**
```json
{
  "slack_channel": "#data-engineering",
  "alert_channel": "#alerts",
  "email_alerts": [
    "data-team@farmarcas.com",
    "engineering@farmarcas.com"
  ],
  "success_notifications": true,
  "failure_notifications": true,
  "warning_notifications": true
}
```

### **3. Conexões Airflow**

#### **Comandos de Setup**
```bash
#!/bin/bash
# setup_airflow_connections.sh

# Airbyte Connection
airflow connections add 'airbyte_default' \
  --conn-type 'airbyte' \
  --conn-host 'airbyte-server' \
  --conn-port 8001 \
  --conn-schema 'http' \
  --conn-extra '{"api_version": "v1"}'

# Slack Webhook Connection
airflow connections add 'slack_webhook' \
  --conn-type 'http' \
  --conn-host 'hooks.slack.com' \
  --conn-password 'your_slack_webhook_token_here' \
  --conn-extra '{"webhook_token": "your_webhook_token"}'

# AWS Connection (opcional, para validações)
airflow connections add 'aws_default' \
  --conn-type 'aws' \
  --conn-extra '{
    "aws_access_key_id": "'${FARMARCAS_AWS_ACCESS_KEY_ID}'",
    "aws_secret_access_key": "'${FARMARCAS_AWS_SECRET_ACCESS_KEY}'",
    "region_name": "us-east-2"
  }'

echo "Conexões Airflow configuradas com sucesso!"
```

---

## 🔐 Environment Variables

### **1. Docker Compose Example**

#### **.env file**
```bash
# MySQL Radar Database
RADAR_HOST=db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com
RADAR_PORT=3306
RADAR_DATABASE=radar
RADAR_USER=bi-cognitivo-read
RADAR_PASS=your_secure_password_here

# AWS Credentials
FARMARCAS_AWS_ACCESS_KEY_ID=AKIA******************
FARMARCAS_AWS_SECRET_ACCESS_KEY=****************************************
AWS_DEFAULT_REGION=us-east-2
AIRBYTE_OCTAVIA_REGION=us-east-2

# Airbyte Configuration
AIRBYTE_SERVER_HOST=airbyte-server
AIRBYTE_SERVER_PORT=8001
AIRBYTE_API_VERSION=v1

# Connection IDs
RADAR_CONNECTION_ID=6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad

# S3 Configuration
S3_BUCKET_NAME=farmarcas-production-bronze
S3_BUCKET_REGION=us-east-2
S3_RADAR_PREFIX=origin=airbyte/database=bronze_radar

# Slack Notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_CHANNEL=#data-engineering
SLACK_ALERT_CHANNEL=#alerts

# Performance Tuning
AIRBYTE_CPU_LIMIT=2000m
AIRBYTE_MEMORY_LIMIT=4Gi
AIRBYTE_WORKER_TIMEOUT=3600

# Logging
LOG_LEVEL=INFO
AIRFLOW_LOG_LEVEL=INFO
```

#### **docker-compose.yml (excerpt)**
```yaml
version: '3.8'
services:
  airbyte-server:
    image: airbyte/server:0.3.23
    ports:
      - "8001:8001"
    environment:
      - DATABASE_URL=postgresql://airbyte:airbyte@db:5432/airbyte
      - CONFIG_DATABASE_URL=postgresql://airbyte:airbyte@db:5432/airbyte
      - TEMPORAL_HOST=temporal:7233
    env_file:
      - .env
    depends_on:
      - db
      - temporal
    
  airbyte-worker:
    image: airbyte/worker:0.3.23
    environment:
      - WORKSPACE_ROOT=/tmp/workspace
      - WORKSPACE_DOCKER_MOUNT=airbyte_workspace
      - LOCAL_ROOT=/tmp/airbyte_local
      - LOCAL_DOCKER_MOUNT=/tmp/airbyte_local
      - CONFIG_ROOT=/data
      - TRACKING_STRATEGY=segment
    env_file:
      - .env
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - airbyte_workspace:/tmp/workspace
      - airbyte_local:/tmp/airbyte_local
    depends_on:
      - airbyte-server
```

### **2. Kubernetes ConfigMap**

#### **radar-config.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: radar-ingestion-config
  namespace: data-platform
data:
  # Airbyte Configuration
  AIRBYTE_SERVER_HOST: "airbyte-server"
  AIRBYTE_SERVER_PORT: "8001"
  AIRBYTE_API_VERSION: "v1"
  
  # Database Configuration
  RADAR_HOST: "db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com"
  RADAR_PORT: "3306"
  RADAR_DATABASE: "radar"
  RADAR_USER: "bi-cognitivo-read"
  
  # AWS Configuration
  AWS_DEFAULT_REGION: "us-east-2"
  S3_BUCKET_NAME: "farmarcas-production-bronze"
  S3_RADAR_PREFIX: "origin=airbyte/database=bronze_radar"
  
  # Connection Configuration
  RADAR_CONNECTION_ID: "6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad"
  
  # Performance Tuning
  AIRBYTE_CPU_LIMIT: "2000m"
  AIRBYTE_MEMORY_LIMIT: "4Gi"
  SYNC_TIMEOUT: "3600"

---
apiVersion: v1
kind: Secret
metadata:
  name: radar-ingestion-secrets
  namespace: data-platform
type: Opaque
stringData:
  # Sensitive credentials
  RADAR_PASS: "your_secure_password_here"
  FARMARCAS_AWS_ACCESS_KEY_ID: "AKIA******************"
  FARMARCAS_AWS_SECRET_ACCESS_KEY: "****************************************"
  SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

---

## 🧪 Testing Configurations

### **1. Integration Test Script**

#### **test_radar_integration.py**
```python
#!/usr/bin/env python3
"""
Script de teste de integração para a ingestão do Radar
Valida todas as configurações e conectividades
"""

import os
import sys
import json
import boto3
import mysql.connector
import requests
from datetime import datetime

class RadarIntegrationTest:
    def __init__(self):
        self.errors = []
        self.warnings = []
        
    def test_environment_variables(self):
        """Testa se todas as variáveis de ambiente estão definidas"""
        required_vars = [
            'RADAR_HOST', 'RADAR_USER', 'RADAR_PASS', 'RADAR_DATABASE',
            'FARMARCAS_AWS_ACCESS_KEY_ID', 'FARMARCAS_AWS_SECRET_ACCESS_KEY',
            'AIRBYTE_SERVER_HOST', 'RADAR_CONNECTION_ID'
        ]
        
        print("Testing environment variables...")
        for var in required_vars:
            if not os.getenv(var):
                self.errors.append(f"Environment variable {var} not set")
            else:
                print(f"✅ {var}: defined")
    
    def test_mysql_connection(self):
        """Testa conectividade com MySQL"""
        print("\nTesting MySQL connection...")
        try:
            conn = mysql.connector.connect(
                host=os.getenv('RADAR_HOST'),
                port=int(os.getenv('RADAR_PORT', 3306)),
                user=os.getenv('RADAR_USER'),
                password=os.getenv('RADAR_PASS'),
                database=os.getenv('RADAR_DATABASE'),
                ssl_disabled=False
            )
            
            cursor = conn.cursor()
            cursor.execute("SELECT VERSION(), DATABASE(), USER()")
            result = cursor.fetchone()
            print(f"✅ MySQL connected: {result}")
            
            # Teste de permissões
            cursor.execute("SHOW GRANTS FOR CURRENT_USER()")
            grants = cursor.fetchall()
            print(f"✅ User grants: {len(grants)} permissions")
            
            conn.close()
        except Exception as e:
            self.errors.append(f"MySQL connection failed: {e}")
    
    def test_s3_access(self):
        """Testa acesso ao S3"""
        print("\nTesting S3 access...")
        try:
            s3 = boto3.client(
                's3',
                aws_access_key_id=os.getenv('FARMARCAS_AWS_ACCESS_KEY_ID'),
                aws_secret_access_key=os.getenv('FARMARCAS_AWS_SECRET_ACCESS_KEY'),
                region_name='us-east-2'
            )
            
            bucket = 'farmarcas-production-bronze'
            
            # Teste de leitura
            response = s3.list_objects_v2(
                Bucket=bucket,
                Prefix='origin=airbyte/database=bronze_radar/',
                MaxKeys=1
            )
            print("✅ S3 read access: OK")
            
            # Teste de escrita
            test_key = 'origin=airbyte/database=bronze_radar/test/integration_test.txt'
            s3.put_object(
                Bucket=bucket,
                Key=test_key,
                Body=f'Integration test - {datetime.now()}'
            )
            print("✅ S3 write access: OK")
            
            # Limpeza
            s3.delete_object(Bucket=bucket, Key=test_key)
            print("✅ S3 delete access: OK")
            
        except Exception as e:
            self.errors.append(f"S3 access failed: {e}")
    
    def test_airbyte_connectivity(self):
        """Testa conectividade com Airbyte"""
        print("\nTesting Airbyte connectivity...")
        try:
            host = os.getenv('AIRBYTE_SERVER_HOST', 'airbyte-server')
            port = os.getenv('AIRBYTE_SERVER_PORT', '8001')
            
            # Health check
            response = requests.get(f"http://{host}:{port}/api/v1/health", timeout=30)
            response.raise_for_status()
            print("✅ Airbyte server health: OK")
            
            # Verificar conexão específica
            connection_id = os.getenv('RADAR_CONNECTION_ID')
            response = requests.get(
                f"http://{host}:{port}/api/v1/connections/{connection_id}",
                timeout=30
            )
            response.raise_for_status()
            connection_data = response.json()
            print(f"✅ Radar connection status: {connection_data.get('status', 'unknown')}")
            
        except Exception as e:
            self.errors.append(f"Airbyte connectivity failed: {e}")
    
    def test_table_schemas(self):
        """Testa acesso às principais tabelas"""
        print("\nTesting table schemas...")
        try:
            conn = mysql.connector.connect(
                host=os.getenv('RADAR_HOST'),
                port=int(os.getenv('RADAR_PORT', 3306)),
                user=os.getenv('RADAR_USER'),
                password=os.getenv('RADAR_PASS'),
                database=os.getenv('RADAR_DATABASE'),
                ssl_disabled=False
            )
            
            cursor = conn.cursor()
            
            # Tabelas principais para verificar
            main_tables = ['store', 'user_access', 'product', 'contest']
            
            for table in main_tables:
                cursor.execute(f"SELECT COUNT(*) FROM {table} LIMIT 1")
                count = cursor.fetchone()[0]
                print(f"✅ Table {table}: {count} records")
            
            conn.close()
            
        except Exception as e:
            self.warnings.append(f"Table schema test failed: {e}")
    
    def run_all_tests(self):
        """Executa todos os testes"""
        print("=" * 60)
        print("RADAR INTEGRATION TESTS")
        print("=" * 60)
        
        self.test_environment_variables()
        self.test_mysql_connection()
        self.test_s3_access()
        self.test_airbyte_connectivity()
        self.test_table_schemas()
        
        # Relatório final
        print("\n" + "=" * 60)
        print("TEST RESULTS")
        print("=" * 60)
        
        if not self.errors and not self.warnings:
            print("🎉 All tests passed successfully!")
            return 0
        else:
            if self.errors:
                print(f"❌ ERRORS ({len(self.errors)}):")
                for error in self.errors:
                    print(f"   - {error}")
            
            if self.warnings:
                print(f"⚠️  WARNINGS ({len(self.warnings)}):")
                for warning in self.warnings:
                    print(f"   - {warning}")
            
            return 1 if self.errors else 0

if __name__ == "__main__":
    tester = RadarIntegrationTest()
    exit_code = tester.run_all_tests()
    sys.exit(exit_code)
```

### **2. Performance Test**

#### **test_radar_performance.py**
```python
#!/usr/bin/env python3
"""
Teste de performance para conexão Radar
"""

import time
import mysql.connector
import boto3
from concurrent.futures import ThreadPoolExecutor

def test_mysql_performance():
    """Testa performance da conexão MySQL"""
    print("Testing MySQL performance...")
    
    start_time = time.time()
    
    conn = mysql.connector.connect(
        host=os.getenv('RADAR_HOST'),
        port=int(os.getenv('RADAR_PORT', 3306)),
        user=os.getenv('RADAR_USER'),
        password=os.getenv('RADAR_PASS'),
        database=os.getenv('RADAR_DATABASE'),
        ssl_disabled=False
    )
    
    cursor = conn.cursor()
    
    # Teste de queries simples
    queries = [
        "SELECT COUNT(*) FROM store",
        "SELECT COUNT(*) FROM user_access",
        "SELECT COUNT(*) FROM product",
        "SELECT COUNT(*) FROM contest"
    ]
    
    for query in queries:
        query_start = time.time()
        cursor.execute(query)
        result = cursor.fetchone()
        query_time = time.time() - query_start
        print(f"Query: {query} | Result: {result[0]} | Time: {query_time:.2f}s")
    
    conn.close()
    
    total_time = time.time() - start_time
    print(f"Total MySQL test time: {total_time:.2f}s")

def test_s3_performance():
    """Testa performance do S3"""
    print("\nTesting S3 performance...")
    
    s3 = boto3.client('s3', region_name='us-east-2')
    bucket = 'farmarcas-production-bronze'
    prefix = 'origin=airbyte/database=bronze_radar/'
    
    start_time = time.time()
    
    # Lista objetos
    response = s3.list_objects_v2(
        Bucket=bucket,
        Prefix=prefix,
        MaxKeys=100
    )
    
    list_time = time.time() - start_time
    object_count = len(response.get('Contents', []))
    
    print(f"S3 list operation: {object_count} objects in {list_time:.2f}s")

if __name__ == "__main__":
    test_mysql_performance()
    test_s3_performance()
```

---

**📍 Próximos Passos:**
- [Erros Comuns](erros_comuns.md)
- [Boas Práticas](boas_praticas.md)
- [Diagrama de Fluxo](diagrama_fluxo.md)
