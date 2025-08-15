# ✅ Pré-requisitos para Ingestão do Radar

## 📋 Visão Geral

Este documento lista todos os pré-requisitos necessários para configurar e executar com sucesso a ingestão de dados do Radar, incluindo permissões, variáveis de ambiente, conectividade e recursos de infraestrutura.

## 🔐 Credenciais e Autenticação

### **1. AWS Credentials**

#### **Variáveis de Ambiente Obrigatórias**
```bash
# AWS Access Keys para S3
export FARMARCAS_AWS_ACCESS_KEY_ID="AKIA*****************"
export FARMARCAS_AWS_SECRET_ACCESS_KEY="****************************************"

# Região AWS
export AWS_DEFAULT_REGION="us-east-2"
export AIRBYTE_OCTAVIA_REGION="us-east-2"
```

#### **Permissões IAM Necessárias**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3AccessRadarBronze",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:GetObjectAcl",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-production-bronze",
        "arn:aws:s3:::farmarcas-production-bronze/origin=airbyte/database=bronze_radar/*"
      ]
    },
    {
      "Sid": "GlueDataCatalog",
      "Effect": "Allow",
      "Action": [
        "glue:CreateTable",
        "glue:UpdateTable",
        "glue:GetTable",
        "glue:GetTables",
        "glue:CreateDatabase",
        "glue:GetDatabase"
      ],
      "Resource": [
        "arn:aws:glue:us-east-2:*:catalog",
        "arn:aws:glue:us-east-2:*:database/bronze_radar",
        "arn:aws:glue:us-east-2:*:table/bronze_radar/*"
      ]
    }
  ]
}
```

### **2. MySQL Radar Credentials**

#### **Variáveis de Ambiente**
```bash
# MySQL Radar Database
export RADAR_HOST="db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com"
export RADAR_PORT="3306"
export RADAR_DATABASE="radar"
export RADAR_USER="bi-cognitivo-read"
export RADAR_PASS="**********************"
```

#### **Verificação de Conectividade**
```bash
# Teste de conexão MySQL
mysql -h ${RADAR_HOST} \
      -P ${RADAR_PORT} \
      -u ${RADAR_USER} \
      -p${RADAR_PASS} \
      ${RADAR_DATABASE} \
      --ssl-mode=PREFERRED \
      -e "SELECT VERSION(), CONNECTION_ID(), DATABASE();"
```

#### **Permissões MySQL Necessárias**
```sql
-- Verificar permissões do usuário
SHOW GRANTS FOR 'bi-cognitivo-read'@'%';

-- Permissões mínimas necessárias:
-- GRANT SELECT ON radar.* TO 'bi-cognitivo-read'@'%';
-- GRANT SHOW VIEW ON radar.* TO 'bi-cognitivo-read'@'%';
-- GRANT PROCESS ON *.* TO 'bi-cognitivo-read'@'%';
```

### **3. Airbyte Configuration**

#### **Variáveis de Ambiente do Airbyte**
```bash
# Airbyte Server
export AIRBYTE_SERVER_HOST="airbyte-server"
export AIRBYTE_SERVER_PORT="8001"
export AIRBYTE_API_VERSION="v1"

# Connection ID específica do Radar
export RADAR_CONNECTION_ID="6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad"
```

#### **Verificação do Airbyte**
```bash
# Health check do servidor Airbyte
curl -f http://${AIRBYTE_SERVER_HOST}:${AIRBYTE_SERVER_PORT}/api/v1/health

# Verificar conexão específica
curl -X GET "http://${AIRBYTE_SERVER_HOST}:${AIRBYTE_SERVER_PORT}/api/v1/connections/${RADAR_CONNECTION_ID}" \
  -H "accept: application/json"
```

---

## 🌐 Conectividade de Rede

### **1. Conectividade MySQL**

#### **Ports e Protocolos**
```yaml
# Requisitos de rede para MySQL
mysql_connectivity:
  protocol: TCP
  port: 3306
  ssl: required (TLS 1.2+)
  endpoint: db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com
  
# Security Groups
security_groups:
  source: sg-airbyte-workers
  destination: sg-mysql-radar-production
  port_range: 3306
  protocol: TCP
```

#### **Teste de Conectividade**
```bash
# Teste de conectividade TCP
nc -zv db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com 3306

# Teste com timeout
timeout 10 bash -c 'cat < /dev/null > /dev/tcp/db-mysql-radar-production.cxsfxyp2ge90.us-east-2.rds.amazonaws.com/3306'
echo $? # 0 = sucesso, 124 = timeout
```

### **2. Conectividade S3**

#### **Endpoints e Regiões**
```yaml
# Configuração S3
s3_connectivity:
  region: us-east-2
  endpoint: s3.us-east-2.amazonaws.com
  bucket: farmarcas-production-bronze
  path_prefix: origin=airbyte/database=bronze_radar
  
# DNS Resolution
dns_requirements:
  - s3.us-east-2.amazonaws.com
  - farmarcas-production-bronze.s3.us-east-2.amazonaws.com
```

#### **Teste de Acesso S3**
```bash
# Verificar acesso ao bucket
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/ \
  --region us-east-2

# Teste de escrita
echo "test" | aws s3 cp - s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/test.txt \
  --region us-east-2

# Limpeza do teste
aws s3 rm s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/test.txt \
  --region us-east-2
```

### **3. Conectividade Airbyte-Airflow**

#### **Communication Requirements**
```yaml
# Requisitos de comunicação
airflow_airbyte:
  airbyte_api_endpoint: "http://airbyte-server:8001/api/v1"
  authentication: none (internal network)
  timeout: 3600 seconds
  retry_attempts: 3
  
# Health Check Endpoint
health_check:
  url: "http://airbyte-server:8001/api/v1/health"
  expected_response: 200
  timeout: 30 seconds
```

---

## 💾 Recursos de Infraestrutura

### **1. Airbyte Resources**

#### **Minimum Requirements**
```yaml
# Recursos mínimos para Airbyte
airbyte_resources:
  cpu:
    request: "500m"
    limit: "2000m"
  memory:
    request: "1Gi"
    limit: "4Gi"
  
  # Disk Space
  storage:
    temp_space: "10Gi"
    logs: "5Gi"
  
  # Network
  bandwidth: "100Mbps"
```

#### **Recommended for Production**
```yaml
# Recursos recomendados
production_resources:
  cpu:
    request: "1000m"
    limit: "4000m"
  memory:
    request: "2Gi"
    limit: "8Gi"
  
  # Storage
  storage:
    temp_space: "50Gi"
    logs: "20Gi"
  
  # Para workloads pesados
  heavy_workload:
    cpu_limit: "8000m"
    memory_limit: "16Gi"
```

### **2. MySQL Performance**

#### **Database Configuration**
```sql
-- Configurações recomendadas para o MySQL
-- (estas configurações devem estar aplicadas no RDS)

-- Connection settings
SHOW VARIABLES LIKE 'max_connections';          -- Deve ser >= 200
SHOW VARIABLES LIKE 'max_user_connections';     -- Deve ser >= 50

-- Buffer settings
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';  -- Deve ser ~70% da RAM
SHOW VARIABLES LIKE 'query_cache_size';         -- Para consultas repetitivas

-- Timeout settings
SHOW VARIABLES LIKE 'wait_timeout';             -- Deve ser >= 3600
SHOW VARIABLES LIKE 'interactive_timeout';      -- Deve ser >= 3600
```

#### **Performance Monitoring**
```sql
-- Queries para monitorar performance
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM performance_schema.global_status 
WHERE VARIABLE_NAME IN (
    'Connections',
    'Max_used_connections',
    'Aborted_connects',
    'Bytes_sent',
    'Bytes_received'
);
```

### **3. S3 Performance**

#### **Request Rate Guidelines**
```yaml
# Limites de performance S3
s3_performance:
  request_rate:
    GET: "5500 requests/second/prefix"
    PUT: "3500 requests/second/prefix"
    DELETE: "3500 requests/second/prefix"
  
  # Throughput
  throughput:
    single_part: "100 MB/s"
    multipart: "10 GB/s"
  
  # Best practices
  prefix_strategy: "random prefix distribution"
  object_size: "optimal 1MB - 5GB"
```

---

## 🔍 Validações e Health Checks

### **1. Script de Validação Geral**

```bash
#!/bin/bash
# health_check_radar.sh - Validação completa dos pré-requisitos

set -e

echo "🔍 Validando pré-requisitos para ingestão do Radar..."

# 1. Verificar variáveis de ambiente
echo "1. Verificando variáveis de ambiente..."
required_vars=(
    "FARMARCAS_AWS_ACCESS_KEY_ID"
    "FARMARCAS_AWS_SECRET_ACCESS_KEY"
    "RADAR_HOST"
    "RADAR_USER"
    "RADAR_PASS"
    "RADAR_DATABASE"
)

for var in "${required_vars[@]}"; do
    if [[ -z "${!var}" ]]; then
        echo "❌ Variável $var não está definida"
        exit 1
    else
        echo "✅ $var: definida"
    fi
done

# 2. Testar conectividade MySQL
echo "2. Testando conectividade MySQL..."
mysql -h $RADAR_HOST -P 3306 -u $RADAR_USER -p$RADAR_PASS $RADAR_DATABASE \
    --ssl-mode=PREFERRED \
    -e "SELECT 'MySQL connection OK' as status;" || {
    echo "❌ Falha na conectividade MySQL"
    exit 1
}
echo "✅ MySQL: conectividade OK"

# 3. Testar acesso S3
echo "3. Testando acesso S3..."
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_radar/ \
    --region us-east-2 > /dev/null || {
    echo "❌ Falha no acesso S3"
    exit 1
}
echo "✅ S3: acesso OK"

# 4. Verificar Airbyte
echo "4. Verificando Airbyte..."
curl -f http://airbyte-server:8001/api/v1/health > /dev/null || {
    echo "❌ Airbyte server não está acessível"
    exit 1
}
echo "✅ Airbyte: server OK"

# 5. Verificar conexão específica
echo "5. Verificando conexão Radar..."
curl -f "http://airbyte-server:8001/api/v1/connections/6c7fda57-ebdb-4c6b-9bc3-6b5d5cb9e1ad" \
    -H "accept: application/json" > /dev/null || {
    echo "❌ Conexão Radar não encontrada"
    exit 1
}
echo "✅ Conexão Radar: configurada"

echo "🎉 Todos os pré-requisitos validados com sucesso!"
```

### **2. Monitoramento Contínuo**

#### **Airflow Health Check DAG**
```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def validate_prerequisites():
    """Validação dos pré-requisitos via Python"""
    import boto3
    import mysql.connector
    import requests
    
    # Validar S3
    s3 = boto3.client('s3', region_name='us-east-2')
    s3.head_bucket(Bucket='farmarcas-production-bronze')
    
    # Validar MySQL
    mysql_conn = mysql.connector.connect(
        host=os.getenv('RADAR_HOST'),
        port=3306,
        user=os.getenv('RADAR_USER'),
        password=os.getenv('RADAR_PASS'),
        database=os.getenv('RADAR_DATABASE'),
        ssl_disabled=False
    )
    mysql_conn.close()
    
    # Validar Airbyte
    response = requests.get('http://airbyte-server:8001/api/v1/health')
    response.raise_for_status()
    
    return "All prerequisites validated successfully"

# DAG para monitoramento
health_check_dag = DAG(
    'radar_health_check',
    description='Health check dos pré-requisitos do Radar',
    schedule_interval='0 1 * * *',  # 1:00 UTC diariamente
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args={
        'owner': 'data-engineering',
        'retries': 2,
        'retry_delay': timedelta(minutes=5)
    }
)

validate_task = PythonOperator(
    task_id='validate_prerequisites',
    python_callable=validate_prerequisites,
    dag=health_check_dag
)
```

---

## 📚 Checklist de Instalação

### **Para Novos Ambientes**

#### **1. Infraestrutura Base**
- [ ] AWS Account configurado
- [ ] VPC e Subnets configuradas
- [ ] Security Groups criados
- [ ] IAM Roles e Policies aplicadas
- [ ] S3 Bucket criado com políticas adequadas

#### **2. Databases**
- [ ] MySQL RDS provisionado
- [ ] Usuario bi-cognitivo-read criado
- [ ] Permissões aplicadas
- [ ] SSL configurado
- [ ] Connection pooling otimizado

#### **3. Airbyte Setup**
- [ ] Airbyte server deployado
- [ ] Source MySQL configurada
- [ ] Destination S3 configurada
- [ ] Connection criada e testada
- [ ] Health checks funcionando

#### **4. Airflow Integration**
- [ ] DAG importado
- [ ] Connections configuradas
- [ ] Variables definidas
- [ ] Notifications configuradas
- [ ] Schedule ativado

#### **5. Monitoramento**
- [ ] CloudWatch alarms criados
- [ ] Slack notifications configuradas
- [ ] Health check DAG funcionando
- [ ] Performance metrics coletadas
- [ ] Error alerting ativo

#### **6. Validação Final**
- [ ] Execução manual bem-sucedida
- [ ] Dados no S3 validados
- [ ] Schema no Glue atualizado
- [ ] Queries Athena funcionando
- [ ] Dashboards atualizados

---

**📍 Próximos Passos:**
- [Configurações de Exemplo](configuracoes_exemplo.md)
- [Fluxo de Ingestão](fluxo_ingestao.md)
- [Erros Comuns](erros_comuns.md)
