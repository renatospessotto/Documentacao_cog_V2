# 🔧 Pré-requisitos - Sistema ACODE + Redundância

## 📋 Visão Geral dos Pré-requisitos

Esta seção detalha todos os pré-requisitos necessários para implementar e operar o sistema de ingestão ACODE com redundância completa. Siga esta documentação para garantir que todos os componentes estejam configurados corretamente.

```mermaid
graph TD
    subgraph "🔐 Credenciais e Acesso"
        A1[fa:fa-key MySQL ACODE Primary<br/>userfarmarcasac02]
        A2[fa:fa-key MySQL ACODE Backup<br/>userfarmarcasac02_backup]
        A3[fa:fa-cloud AWS Credentials<br/>S3 + IAM]
    end
    
    subgraph "🏗️ Infraestrutura"
        B1[fa:fa-server Airbyte Platform<br/>v0.3.23+]
        B2[fa:fa-rocket Airflow<br/>v2.5.0+]
        B3[fa:fa-network Network Connectivity<br/>ACODE + AWS]
    end
    
    subgraph "🛠️ Ferramentas"
        C1[fa:fa-terminal Octavia CLI<br/>Deployment]
        C2[fa:fa-code Python 3.8+<br/>Scripts]
        C3[fa:fa-docker Docker<br/>Containers]
    end
    
    subgraph "📊 Monitoramento"
        D1[fa:fa-chart-bar DataDog<br/>Metrics]
        D2[fa:fa-bell Slack<br/>Notifications]
        D3[fa:fa-phone PagerDuty<br/>Alerts]
    end
    
    A1 --> B1
    A2 --> B1
    A3 --> B1
    B1 --> C1
    B2 --> C2
    C1 --> D1
    D1 --> D2
    D2 --> D3
    
    style A1 fill:#ff6b6b
    style A3 fill:#4ecdc4
    style B1 fill:#45b7d1
    style D2 fill:#96ceb4
```

## 🔐 Credenciais e Acesso

### **1. MySQL ACODE Primary**

#### Credenciais Básicas
```bash
# Variáveis de ambiente obrigatórias
export ACODE_PASS="senha_acode_production_segura"
export ACODE_HOST="db-hsp-farmarcas.acode.com.br"
export ACODE_PORT="3306"
export ACODE_DATABASE="acode_farmarcas"
export ACODE_USER="userfarmarcasac02"
```

#### Teste de Conectividade Primary
```bash
#!/bin/bash
# test_acode_primary_connectivity.sh

echo "🔍 Testing ACODE Primary Connectivity"
echo "=================================="

# Test 1: Basic network connectivity
echo "1. Testing network connectivity..."
ping -c 3 $ACODE_HOST && echo "✅ Network OK" || echo "❌ Network FAILED"

# Test 2: Port accessibility
echo "2. Testing port accessibility..."
nc -zv $ACODE_HOST $ACODE_PORT && echo "✅ Port OK" || echo "❌ Port FAILED"

# Test 3: MySQL authentication
echo "3. Testing MySQL authentication..."
mysql -h $ACODE_HOST \
      -P $ACODE_PORT \
      -u $ACODE_USER \
      -p$ACODE_PASS \
      -D $ACODE_DATABASE \
      --ssl-mode=PREFERRED \
      --connect-timeout=10 \
      -e "SELECT 'Authentication OK' as status, NOW() as timestamp" \
      && echo "✅ Authentication OK" || echo "❌ Authentication FAILED"

# Test 4: SSL connectivity
echo "4. Testing SSL connectivity..."
mysql -h $ACODE_HOST \
      -P $ACODE_PORT \
      -u $ACODE_USER \
      -p$ACODE_PASS \
      --ssl-mode=REQUIRED \
      --connect-timeout=10 \
      -e "SHOW STATUS LIKE 'Ssl_cipher'" \
      && echo "✅ SSL OK" || echo "❌ SSL FAILED"

# Test 5: Table access permissions
echo "5. Testing table access permissions..."
mysql -h $ACODE_HOST \
      -P $ACODE_PORT \
      -u $ACODE_USER \
      -p$ACODE_PASS \
      -D $ACODE_DATABASE \
      --ssl-mode=PREFERRED \
      -e "
SELECT 
    'farmarcas_si_analitico_diario' as table_name,
    COUNT(*) as record_count 
FROM farmarcas_si_analitico_diario 
LIMIT 1;

SELECT 
    'farmarcas_si_analitico_diario_produtos' as table_name,
    COUNT(*) as record_count 
FROM farmarcas_si_analitico_diario_produtos 
LIMIT 1;
" && echo "✅ Table Access OK" || echo "❌ Table Access FAILED"

echo "=================================="
echo "🔍 Primary connectivity test completed"
```

#### Permissões MySQL Necessárias
```sql
-- Verificar permissões atuais
SHOW GRANTS FOR 'userfarmarcasac02'@'%';

-- Permissões mínimas obrigatórias
GRANT SELECT ON acode_farmarcas.farmarcas_si_analitico_diario TO 'userfarmarcasac02'@'%';
GRANT SELECT ON acode_farmarcas.farmarcas_si_analitico_diario_produtos TO 'userfarmarcasac02'@'%';

-- Permissões para monitoramento (opcional)
GRANT PROCESS ON *.* TO 'userfarmarcasac02'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'userfarmarcasac02'@'%';

-- Aplicar mudanças
FLUSH PRIVILEGES;

-- Verificar conectividade SSL
SELECT 
    @@ssl_ca,
    @@ssl_cert,
    @@ssl_key,
    @@have_ssl;

-- Verificar configurações de timeout
SHOW VARIABLES LIKE '%timeout%';
```

### **2. MySQL ACODE Backup**

#### Setup do Servidor Backup
```bash
# Variáveis de ambiente para backup
export ACODE_BACKUP_PASS="senha_backup_acode_segura"
export ACODE_BACKUP_HOST="acode-backup.farmarcas.internal"
export ACODE_BACKUP_PORT="3306"
export ACODE_BACKUP_DATABASE="acode_farmarcas_replica"
export ACODE_BACKUP_USER="userfarmarcasac02_backup"
```

#### Configuração de Replicação Master-Slave
```sql
-- No servidor PRIMARY (ACODE)
-- Configurações de master
SET GLOBAL log_bin = ON;
SET GLOBAL server_id = 1;
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_do_db = 'acode_farmarcas';
SET GLOBAL expire_logs_days = 7;

-- Criar usuário de replicação
CREATE USER 'replication_acode'@'%' IDENTIFIED BY 'replication_password_muito_segura';
GRANT REPLICATION SLAVE ON *.* TO 'replication_acode'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'replication_acode'@'%';
FLUSH PRIVILEGES;

-- Backup inicial para sincronização
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;  -- Anotar File e Position
-- Fazer dump aqui (em outra sessão)
UNLOCK TABLES;

-- No servidor BACKUP
-- Configurações de slave
SET GLOBAL server_id = 2;
SET GLOBAL read_only = ON;
SET GLOBAL relay_log_purge = ON;
SET GLOBAL slave_net_timeout = 60;

-- Importar dump inicial
SOURCE /path/to/acode_initial_dump.sql;

-- Configurar replicação
STOP SLAVE;
CHANGE MASTER TO 
  MASTER_HOST='db-hsp-farmarcas.acode.com.br',
  MASTER_USER='replication_acode',
  MASTER_PASSWORD='replication_password_muito_segura',
  MASTER_LOG_FILE='mysql-bin.000001',  -- Do SHOW MASTER STATUS
  MASTER_LOG_POS=107;                  -- Do SHOW MASTER STATUS

START SLAVE;

-- Verificar replicação
SHOW SLAVE STATUS\G

-- Monitoramento contínuo
SELECT 
    MASTER_HOST,
    MASTER_LOG_FILE,
    MASTER_LOG_POS,
    SLAVE_IO_RUNNING,
    SLAVE_SQL_RUNNING,
    SECONDS_BEHIND_MASTER,
    LAST_ERROR
FROM performance_schema.replication_connection_status
JOIN performance_schema.replication_applier_status_by_worker;
```

#### Script de Monitoramento de Replicação
```python
#!/usr/bin/env python3
# monitor_replication.py

import mysql.connector
import time
import logging
import os
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ReplicationMonitor:
    def __init__(self):
        self.backup_config = {
            'host': os.getenv('ACODE_BACKUP_HOST'),
            'port': int(os.getenv('ACODE_BACKUP_PORT', 3306)),
            'database': os.getenv('ACODE_BACKUP_DATABASE'),
            'user': os.getenv('ACODE_BACKUP_USER'),
            'password': os.getenv('ACODE_BACKUP_PASS')
        }
        
    def check_replication_status(self):
        """Check MySQL replication status"""
        try:
            conn = mysql.connector.connect(**self.backup_config)
            cursor = conn.cursor(dictionary=True)
            
            cursor.execute("SHOW SLAVE STATUS")
            slave_status = cursor.fetchone()
            
            if slave_status:
                status = {
                    'master_host': slave_status.get('Master_Host'),
                    'master_log_file': slave_status.get('Master_Log_File'),
                    'slave_io_running': slave_status.get('Slave_IO_Running'),
                    'slave_sql_running': slave_status.get('Slave_SQL_Running'),
                    'seconds_behind_master': slave_status.get('Seconds_Behind_Master'),
                    'last_error': slave_status.get('Last_Error'),
                    'replication_healthy': (
                        slave_status.get('Slave_IO_Running') == 'Yes' and
                        slave_status.get('Slave_SQL_Running') == 'Yes' and
                        (slave_status.get('Seconds_Behind_Master') or 0) < 300  # 5 min
                    ),
                    'check_timestamp': datetime.now().isoformat()
                }
                
                logger.info(f"Replication status: {status}")
                return status
            else:
                logger.error("No slave status found - replication not configured")
                return {'replication_healthy': False, 'error': 'No slave status'}
                
        except Exception as e:
            logger.error(f"Replication check failed: {e}")
            return {'replication_healthy': False, 'error': str(e)}
        finally:
            if 'conn' in locals():
                conn.close()
    
    def restart_replication_if_needed(self):
        """Restart replication if it's stopped"""
        status = self.check_replication_status()
        
        if not status.get('replication_healthy', False):
            logger.warning("Replication unhealthy - attempting restart")
            
            try:
                conn = mysql.connector.connect(**self.backup_config)
                cursor = conn.cursor()
                
                # Stop and start slave
                cursor.execute("STOP SLAVE")
                time.sleep(2)
                cursor.execute("START SLAVE")
                
                conn.close()
                
                # Wait and check again
                time.sleep(10)
                new_status = self.check_replication_status()
                
                if new_status.get('replication_healthy', False):
                    logger.info("✅ Replication restart successful")
                    return True
                else:
                    logger.error("❌ Replication restart failed")
                    return False
                    
            except Exception as e:
                logger.error(f"Replication restart error: {e}")
                return False
        else:
            logger.info("✅ Replication is healthy")
            return True

if __name__ == "__main__":
    monitor = ReplicationMonitor()
    
    # Continuous monitoring mode
    while True:
        try:
            monitor.restart_replication_if_needed()
            time.sleep(300)  # Check every 5 minutes
        except KeyboardInterrupt:
            logger.info("Monitoring stopped by user")
            break
        except Exception as e:
            logger.error(f"Monitoring error: {e}")
            time.sleep(60)  # Wait 1 minute on error
```

### **3. AWS Credentials e S3**

#### IAM Policy para ACODE
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ACODEBronzeLayerAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-production-bronze/origin=airbyte/database=bronze_acode/*",
        "arn:aws:s3:::farmarcas-production-bronze"
      ]
    },
    {
      "Sid": "ACODEBackupBucketAccess", 
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-backup-acode/*",
        "arn:aws:s3:::farmarcas-backup-acode"
      ]
    },
    {
      "Sid": "ACODEEmergencyBucketAccess",
      "Effect": "Allow", 
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-emergency-backup/*",
        "arn:aws:s3:::farmarcas-emergency-backup"
      ]
    },
    {
      "Sid": "ACODEQuarantineBucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject", 
        "s3:ListBucket",
        "s3:PutObjectTagging"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-quarantine/*",
        "arn:aws:s3:::farmarcas-quarantine"
      ]
    }
  ]
}
```

#### Configuração de Credenciais AWS
```bash
# Variáveis de ambiente AWS
export FARMARCAS_AWS_ACCESS_KEY_ID="AKIA..."
export FARMARCAS_AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-2"

# Teste de credenciais AWS
aws sts get-caller-identity

# Teste de acesso aos buckets
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/
aws s3 ls s3://farmarcas-backup-acode/
aws s3 ls s3://farmarcas-emergency-backup/
aws s3 ls s3://farmarcas-quarantine/

# Teste de upload/download
echo "test-$(date)" > /tmp/acode-test.txt
aws s3 cp /tmp/acode-test.txt s3://farmarcas-production-bronze/test/acode-test.txt
aws s3 cp s3://farmarcas-production-bronze/test/acode-test.txt /tmp/acode-download-test.txt
diff /tmp/acode-test.txt /tmp/acode-download-test.txt && echo "✅ S3 Access OK"

# Cleanup
aws s3 rm s3://farmarcas-production-bronze/test/acode-test.txt
rm /tmp/acode-test.txt /tmp/acode-download-test.txt
```

#### Criação dos Buckets S3
```bash
#!/bin/bash
# create_acode_s3_buckets.sh

echo "🪣 Creating ACODE S3 Buckets"
echo "=========================="

# Primary bucket (já deve existir)
aws s3 ls s3://farmarcas-production-bronze/ >/dev/null 2>&1 || {
    echo "Creating primary bucket..."
    aws s3 mb s3://farmarcas-production-bronze --region us-east-2
}

# Backup bucket
aws s3 ls s3://farmarcas-backup-acode/ >/dev/null 2>&1 || {
    echo "Creating backup bucket..."
    aws s3 mb s3://farmarcas-backup-acode --region us-east-2
    
    # Configure lifecycle policy
    cat > /tmp/backup-lifecycle.json << EOF
{
    "Rules": [{
        "ID": "acode-backup-lifecycle",
        "Status": "Enabled",
        "Filter": {"Prefix": "origin=backup/"},
        "Transitions": [
            {
                "Days": 30,
                "StorageClass": "STANDARD_IA"
            },
            {
                "Days": 90,
                "StorageClass": "GLACIER"
            }
        ],
        "Expiration": {
            "Days": 365
        }
    }]
}
EOF
    
    aws s3api put-bucket-lifecycle-configuration \
        --bucket farmarcas-backup-acode \
        --lifecycle-configuration file:///tmp/backup-lifecycle.json
}

# Emergency bucket
aws s3 ls s3://farmarcas-emergency-backup/ >/dev/null 2>&1 || {
    echo "Creating emergency bucket..."
    aws s3 mb s3://farmarcas-emergency-backup --region us-east-1  # Different region
    
    # Configure lifecycle policy (longer retention)
    cat > /tmp/emergency-lifecycle.json << EOF
{
    "Rules": [{
        "ID": "acode-emergency-lifecycle",
        "Status": "Enabled",
        "Filter": {"Prefix": "acode/"},
        "Transitions": [
            {
                "Days": 7,
                "StorageClass": "STANDARD_IA"
            },
            {
                "Days": 30,
                "StorageClass": "GLACIER"
            }
        ],
        "Expiration": {
            "Days": 2555
        }
    }]
}
EOF
    
    aws s3api put-bucket-lifecycle-configuration \
        --bucket farmarcas-emergency-backup \
        --lifecycle-configuration file:///tmp/emergency-lifecycle.json
}

# Quarantine bucket
aws s3 ls s3://farmarcas-quarantine/ >/dev/null 2>&1 || {
    echo "Creating quarantine bucket..."
    aws s3 mb s3://farmarcas-quarantine --region us-east-2
    
    # Configure lifecycle policy (analysis period)
    cat > /tmp/quarantine-lifecycle.json << EOF
{
    "Rules": [{
        "ID": "acode-quarantine-lifecycle",
        "Status": "Enabled",
        "Filter": {"Prefix": "acode/"},
        "Transitions": [
            {
                "Days": 1,
                "StorageClass": "STANDARD_IA"
            }
        ],
        "Expiration": {
            "Days": 90
        }
    }]
}
EOF
    
    aws s3api put-bucket-lifecycle-configuration \
        --bucket farmarcas-quarantine \
        --lifecycle-configuration file:///tmp/quarantine-lifecycle.json
}

echo "✅ All ACODE S3 buckets created and configured"
echo "=========================="

# Cleanup temp files
rm -f /tmp/*-lifecycle.json
```

## 🏗️ Infraestrutura

### **1. Airbyte Platform**

#### Versão Mínima e Configuração
```yaml
# Requisitos Airbyte
airbyte_version: ">=0.3.23"
platform_edition: "OSS" # ou "Cloud"

# Recursos mínimos recomendados
resources:
  server:
    cpu: "2 cores"
    memory: "4Gi"
    storage: "50Gi SSD"
  
  worker:
    cpu: "4 cores"  # Para processamento ACODE
    memory: "8Gi"   # Para tabelas grandes
    storage: "100Gi SSD"
  
  database:
    engine: "PostgreSQL 13+"
    cpu: "2 cores"
    memory: "4Gi"
    storage: "100Gi SSD"

# Network requirements
network:
  external_access:
    - "db-hsp-farmarcas.acode.com.br:3306"  # ACODE Primary
    - "acode-backup.farmarcas.internal:3306" # ACODE Backup
    - "*.amazonaws.com:443"                   # AWS S3
  
  internal_access:
    - "airbyte-server:8001"
    - "airbyte-webapp:80"
    - "airbyte-db:5432"
```

#### Verificação de Instalação Airbyte
```bash
#!/bin/bash
# verify_airbyte_installation.sh

echo "🔍 Verifying Airbyte Installation"
echo "==============================="

# Check Airbyte server
echo "1. Checking Airbyte server..."
curl -s http://airbyte-server:8001/api/v1/health | jq -r '.available' | grep -q "true" \
    && echo "✅ Airbyte server OK" || echo "❌ Airbyte server FAILED"

# Check Airbyte webapp
echo "2. Checking Airbyte webapp..."
curl -s -o /dev/null -w "%{http_code}" http://airbyte-webapp:80/ | grep -q "200" \
    && echo "✅ Airbyte webapp OK" || echo "❌ Airbyte webapp FAILED"

# Check Airbyte database
echo "3. Checking Airbyte database..."
kubectl exec -n airbyte airbyte-db-0 -- psql -U airbyte -d airbyte -c "SELECT 1" >/dev/null 2>&1 \
    && echo "✅ Airbyte database OK" || echo "❌ Airbyte database FAILED"

# Check available connectors
echo "4. Checking required connectors..."
curl -s http://airbyte-server:8001/api/v1/connector_definitions | jq -r '.connectorDefinitions[].name' | grep -q "MySQL" \
    && echo "✅ MySQL source connector available" || echo "❌ MySQL source connector MISSING"

curl -s http://airbyte-server:8001/api/v1/connector_definitions | jq -r '.connectorDefinitions[].name' | grep -q "S3" \
    && echo "✅ S3 destination connector available" || echo "❌ S3 destination connector MISSING"

# Check workspace
echo "5. Checking workspace..."
WORKSPACE_ID=$(curl -s http://airbyte-server:8001/api/v1/workspaces | jq -r '.workspaces[0].workspaceId')
[ ! -z "$WORKSPACE_ID" ] && [ "$WORKSPACE_ID" != "null" ] \
    && echo "✅ Workspace OK: $WORKSPACE_ID" || echo "❌ No workspace found"

echo "==============================="
echo "🔍 Airbyte verification completed"
```

### **2. Airflow Platform**

#### Configuração Mínima do Airflow
```yaml
# Airflow requirements
airflow_version: ">=2.5.0"

# Providers necessários
providers:
  - apache-airflow-providers-airbyte>=3.0.0
  - apache-airflow-providers-mysql>=3.0.0
  - apache-airflow-providers-amazon>=6.0.0
  - apache-airflow-providers-slack>=7.0.0

# Configurações de pool
pools:
  acode_pool:
    slots: 2  # Máximo 2 DAGs ACODE simultâneas
    description: "Pool for ACODE related tasks"

# Configurações de variáveis
variables:
  ACODE_PASS: "{{ conn.acode_mysql.password }}"
  SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/..."
  AIRBYTE_CONN_ID: "airbyte_default"

# Configurações de conexões
connections:
  airbyte_default:
    conn_type: "http"
    host: "airbyte-server"
    port: 8001
    
  acode_mysql:
    conn_type: "mysql"
    host: "db-hsp-farmarcas.acode.com.br"
    port: 3306
    schema: "acode_farmarcas"
    login: "userfarmarcasac02"
    password: "..."
    
  aws_default:
    conn_type: "aws"
    extra: {
      "aws_access_key_id": "...",
      "aws_secret_access_key": "...",
      "region_name": "us-east-2"
    }
```

#### Teste de Configuração Airflow
```python
#!/usr/bin/env python3
# test_airflow_setup.py

from airflow.models import Connection, Variable
from airflow.providers.airbyte.hooks.airbyte import AirbyteHook
from airflow.providers.mysql.hooks.mysql import MySqlHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def test_airflow_connections():
    """Test all required Airflow connections"""
    
    # Test Airbyte connection
    try:
        airbyte_hook = AirbyteHook(airbyte_conn_id='airbyte_default')
        health = airbyte_hook.test_connection()
        logger.info(f"✅ Airbyte connection: {health}")
    except Exception as e:
        logger.error(f"❌ Airbyte connection failed: {e}")
    
    # Test MySQL ACODE connection
    try:
        mysql_hook = MySqlHook(mysql_conn_id='acode_mysql')
        result = mysql_hook.get_first("SELECT 1 as test")
        logger.info(f"✅ MySQL ACODE connection: {result}")
    except Exception as e:
        logger.error(f"❌ MySQL ACODE connection failed: {e}")
    
    # Test AWS S3 connection
    try:
        s3_hook = S3Hook(aws_conn_id='aws_default')
        buckets = s3_hook.list_buckets()
        acode_buckets = [b for b in buckets if 'acode' in b['Name'].lower()]
        logger.info(f"✅ AWS S3 connection: {len(acode_buckets)} ACODE buckets found")
    except Exception as e:
        logger.error(f"❌ AWS S3 connection failed: {e}")

def test_airflow_variables():
    """Test required Airflow variables"""
    
    required_vars = [
        'SLACK_WEBHOOK_URL',
        'AIRBYTE_CONN_ID'
    ]
    
    for var_name in required_vars:
        try:
            value = Variable.get(var_name)
            logger.info(f"✅ Variable {var_name}: {'*' * 10}")  # Masked
        except Exception as e:
            logger.error(f"❌ Variable {var_name} not found: {e}")

if __name__ == "__main__":
    test_airflow_connections()
    test_airflow_variables()
```

### **3. Network Connectivity**

#### Verificação de Conectividade de Rede
```bash
#!/bin/bash
# test_network_connectivity.sh

echo "🌐 Testing Network Connectivity"
echo "=============================="

# Test ACODE Primary
echo "1. Testing ACODE Primary connectivity..."
nc -zv db-hsp-farmarcas.acode.com.br 3306 && echo "✅ ACODE Primary OK" || echo "❌ ACODE Primary FAILED"

# Test ACODE Backup
echo "2. Testing ACODE Backup connectivity..."
nc -zv acode-backup.farmarcas.internal 3306 && echo "✅ ACODE Backup OK" || echo "❌ ACODE Backup FAILED"

# Test AWS endpoints
echo "3. Testing AWS S3 connectivity..."
curl -s -I https://s3.us-east-2.amazonaws.com/ | head -1 | grep -q "200" \
    && echo "✅ AWS S3 OK" || echo "❌ AWS S3 FAILED"

# Test internal services
echo "4. Testing internal services..."
nc -zv airbyte-server 8001 && echo "✅ Airbyte Server OK" || echo "❌ Airbyte Server FAILED"
nc -zv airflow-webserver 8080 && echo "✅ Airflow Webserver OK" || echo "❌ Airflow Webserver FAILED"

# Test DNS resolution
echo "5. Testing DNS resolution..."
nslookup db-hsp-farmarcas.acode.com.br >/dev/null 2>&1 \
    && echo "✅ ACODE DNS OK" || echo "❌ ACODE DNS FAILED"

nslookup s3.us-east-2.amazonaws.com >/dev/null 2>&1 \
    && echo "✅ AWS DNS OK" || echo "❌ AWS DNS FAILED"

echo "=============================="
echo "🌐 Network connectivity test completed"
```

## 🛠️ Ferramentas

### **1. Octavia CLI**

#### Instalação e Configuração
```bash
# Instalar Octavia CLI
pip install octavia-cli

# Configurar Octavia
mkdir -p ~/.octavia
cat > ~/.octavia/config.yaml << EOF
airbyte_server_url: http://airbyte-server:8001
workspace_id: ${AIRBYTE_WORKSPACE_ID}
airbyte_version: 0.3.23
EOF

# Inicializar projeto Octavia
cd /path/to/acode/configurations
octavia init

# Verificar configuração
octavia list workspaces
octavia list sources
octavia list destinations
octavia list connections
```

#### Estrutura de Arquivos Octavia
```
acode_configurations/
├── .octavia/
│   └── config.yaml
├── sources/
│   ├── source_mysql_acode/
│   │   └── configuration.yaml
│   └── source_mysql_acode_backup/
│       └── configuration.yaml
├── destinations/
│   ├── destination_s3_acode/
│   │   └── configuration.yaml
│   └── destination_s3_acode_backup/
│       └── configuration.yaml
└── connections/
    ├── connection_mysql_s3_acode/
    │   └── configuration.yaml
    └── connection_mysql_s3_acode_backup/
        └── configuration.yaml
```

### **2. Python Environment**

#### Dependências Python
```txt
# requirements.txt para scripts ACODE
# Core dependencies
python>=3.8
pandas>=1.3.0
numpy>=1.21.0
mysql-connector-python>=8.0.28
boto3>=1.24.0
requests>=2.28.0

# Airbyte integration
octavia-cli>=0.40.0

# Data processing
pyarrow>=8.0.0  # Para Parquet
sqlalchemy>=1.4.0

# Monitoring and alerting
slack-sdk>=3.18.0
datadog>=0.44.0

# Utils
python-dotenv>=0.19.0
click>=8.0.0
pyyaml>=6.0
```

#### Setup do Ambiente Python
```bash
#!/bin/bash
# setup_python_environment.sh

echo "🐍 Setting up Python Environment for ACODE"
echo "=========================================="

# Create virtual environment
python3 -m venv acode_env
source acode_env/bin/activate

# Upgrade pip
pip install --upgrade pip

# Install requirements
pip install -r requirements.txt

# Install additional tools
pip install jupyter notebook  # Para análise de dados
pip install black isort flake8  # Para qualidade de código

# Verify installation
echo "Verifying installation..."
python -c "import pandas; print(f'Pandas: {pandas.__version__}')"
python -c "import boto3; print(f'Boto3: {boto3.__version__}')"
python -c "import mysql.connector; print(f'MySQL Connector: {mysql.connector.__version__}')"

# Create environment activation script
cat > activate_acode_env.sh << 'EOF'
#!/bin/bash
source acode_env/bin/activate
export PYTHONPATH="${PYTHONPATH}:$(pwd)/scripts"
echo "✅ ACODE Python environment activated"
EOF

chmod +x activate_acode_env.sh

echo "✅ Python environment setup completed"
echo "Run: source activate_acode_env.sh"
```

### **3. Docker Environment**

#### Docker Compose para Desenvolvimento
```yaml
# docker-compose.acode-dev.yml
version: '3.8'

services:
  mysql-acode-dev:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: acode_farmarcas_dev
      MYSQL_USER: userfarmarcasac02
      MYSQL_PASSWORD: ${ACODE_PASS}
    ports:
      - "3306:3306"
    volumes:
      - mysql_acode_data:/var/lib/mysql
      - ./dev_data:/docker-entrypoint-initdb.d
    command: --log-bin=mysql-bin --server-id=1
    
  mysql-acode-backup-dev:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: acode_farmarcas_replica
      MYSQL_USER: userfarmarcasac02_backup
      MYSQL_PASSWORD: ${ACODE_BACKUP_PASS}
    ports:
      - "3307:3306"
    volumes:
      - mysql_backup_data:/var/lib/mysql
    command: --server-id=2 --read-only=1
    depends_on:
      - mysql-acode-dev
      
  localstack:
    image: localstack/localstack:latest
    environment:
      SERVICES: s3
      DEBUG: 1
      DATA_DIR: /tmp/localstack/data
    ports:
      - "4566:4566"
    volumes:
      - localstack_data:/tmp/localstack
      
  acode-scripts:
    build: 
      context: .
      dockerfile: Dockerfile.acode-scripts
    environment:
      ACODE_HOST: mysql-acode-dev
      ACODE_BACKUP_HOST: mysql-acode-backup-dev
      AWS_ENDPOINT_URL: http://localstack:4566
    volumes:
      - ./scripts:/app/scripts
      - ./data:/app/data
    depends_on:
      - mysql-acode-dev
      - mysql-acode-backup-dev
      - localstack

volumes:
  mysql_acode_data:
  mysql_backup_data:
  localstack_data:
```

#### Dockerfile para Scripts
```dockerfile
# Dockerfile.acode-scripts
FROM python:3.9-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    default-mysql-client \
    curl \
    netcat \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy scripts
COPY scripts/ ./scripts/

# Make scripts executable
RUN chmod +x scripts/*.sh scripts/*.py

# Create entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["bash"]
```

## 📊 Monitoramento

### **1. DataDog Integration**

#### Configuração DataDog
```yaml
# datadog.yaml para ACODE
api_key: ${DATADOG_API_KEY}
app_key: ${DATADOG_APP_KEY}

# Tags padrão
tags:
  - env:production
  - service:acode-ingestion
  - team:data-engineering

# Custom metrics
custom_metrics:
  - name: acode.sync.duration
    type: histogram
    description: "Duration of ACODE sync operations"
    
  - name: acode.sync.records_processed
    type: count
    description: "Number of records processed in ACODE sync"
    
  - name: acode.health.primary_status
    type: gauge
    description: "ACODE primary system health (1=healthy, 0=unhealthy)"
    
  - name: acode.health.backup_status
    type: gauge
    description: "ACODE backup system health (1=healthy, 0=unhealthy)"
    
  - name: acode.replication.lag_seconds
    type: gauge
    description: "MySQL replication lag in seconds"

# Dashboards
dashboards:
  - name: "ACODE System Overview"
    widgets:
      - type: timeseries
        title: "Sync Duration"
        metrics: ["acode.sync.duration"]
        
      - type: query_value
        title: "Primary System Status"
        metrics: ["acode.health.primary_status"]
        
      - type: query_value
        title: "Backup System Status" 
        metrics: ["acode.health.backup_status"]
        
      - type: timeseries
        title: "Replication Lag"
        metrics: ["acode.replication.lag_seconds"]
```

### **2. Slack Notifications**

#### Webhook Configuration
```bash
# Configurar webhook Slack
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"

# Testar webhook
curl -X POST $SLACK_WEBHOOK_URL \
     -H "Content-Type: application/json" \
     -d '{
       "text": "🧪 ACODE System Test Notification",
       "channel": "#data-engineering-alerts",
       "username": "ACODE Monitor",
       "icon_emoji": ":gear:"
     }'
```

### **3. PagerDuty Integration**

#### Service Configuration
```yaml
# PagerDuty service for ACODE
service_name: "ACODE Data Ingestion"
service_key: ${PAGERDUTY_SERVICE_KEY}

escalation_policy:
  name: "ACODE Critical Escalation"
  escalation_rules:
    - escalation_delay_in_minutes: 15
      targets:
        - type: user
          id: "data-engineering-oncall"
    - escalation_delay_in_minutes: 30
      targets:
        - type: user
          id: "head-of-data"
    - escalation_delay_in_minutes: 60
      targets:
        - type: user
          id: "cto"

alert_grouping:
  type: intelligent
  grouping_window: 300  # 5 minutes

urgency_rules:
  - conditions:
      - field: "summary"
        operator: "contains"
        value: "ACODE"
    actions:
      - type: urgency
        value: high
```

---

## ✅ Checklist de Pré-requisitos

### **Credenciais e Acesso** ✓
- [ ] MySQL ACODE Primary credentials configured
- [ ] MySQL ACODE Backup credentials configured  
- [ ] MySQL replication setup completed
- [ ] AWS credentials configured
- [ ] S3 buckets created and accessible
- [ ] IAM policies applied

### **Infraestrutura** ✓
- [ ] Airbyte platform installed (v0.3.23+)
- [ ] Airflow platform installed (v2.5.0+)
- [ ] Required Airbyte connectors available
- [ ] Network connectivity verified
- [ ] Resource requirements met

### **Ferramentas** ✓
- [ ] Octavia CLI installed and configured
- [ ] Python environment setup completed
- [ ] Required Python packages installed
- [ ] Docker environment configured (if applicable)

### **Monitoramento** ✓
- [ ] DataDog integration configured
- [ ] Slack webhook configured
- [ ] PagerDuty service setup
- [ ] Health check scripts deployed

### **Testes** ✓
- [ ] Primary MySQL connectivity tested
- [ ] Backup MySQL replication tested
- [ ] S3 access tested
- [ ] Airbyte API accessible
- [ ] Airflow connections tested
- [ ] All scripts executed successfully

---

## 🔗 Próximos Passos

1. **[Configurações Exemplo](./configuracoes_exemplo.md)** - Deploy completo das configurações
2. **[Fluxo ACODE](./fluxo_acode.md)** - Entender o fluxo de dados
3. **[Redundância](./redundancia.md)** - Implementar mecanismos de backup
4. **[Troubleshooting](./erros_comuns.md)** - Resolver problemas comuns

---

**Data de Atualização**: 07/08/2025 - Pré-requisitos validados para produção
