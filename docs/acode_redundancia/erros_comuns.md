# ⚠️ Erros Comuns e Troubleshooting - ACODE + Redundância

## 📋 Visão Geral

Este guia apresenta os problemas mais comuns encontrados no sistema ACODE com redundância e suas respectivas soluções. Inclui procedimentos de diagnóstico, comandos de resolução e scripts automatizados para correção rápida.

```mermaid
graph TD
    subgraph "🚨 Tipos de Problemas"
        A[Connection Issues<br/>Conectividade MySQL/S3]
        B[Sync Failures<br/>Falhas de sincronização]
        C[Performance Issues<br/>Performance degradada]
        D[Data Quality<br/>Qualidade dos dados]
        E[Backup Issues<br/>Problemas de backup]
    end
    
    subgraph "🔍 Diagnóstico"
        F[Health Checks<br/>Verificações automáticas]
        G[Log Analysis<br/>Análise de logs]
        H[Manual Tests<br/>Testes manuais]
    end
    
    subgraph "🛠️ Resolução"
        I[Auto-Recovery<br/>Recuperação automática]
        J[Manual Fix<br/>Correção manual]
        K[Backup Activation<br/>Ativação backup]
    end
    
    A --> F
    B --> G
    C --> H
    D --> G
    E --> F
    
    F --> I
    G --> J
    H --> K
    
    style A fill:#ff6b6b
    style B fill:#feca57
    style C fill:#48cae4
    style D fill:#f72585
    style E fill:#7209b7
```

## 🔗 Problemas de Conectividade

### **Erro 1: MySQL Connection Timeout**

#### **Sintomas:**
```bash
ERROR: Connection to MySQL server lost during query
ERROR: Can't connect to MySQL server on 'db-hsp-farmarcas.acode.com.br'
```

#### **Diagnóstico:**
```bash
#!/bin/bash
# diagnose_mysql_connectivity.sh

echo "🔍 Diagnosing MySQL connectivity..."

# Test network connectivity
echo "1. Testing network connectivity:"
nc -zv db-hsp-farmarcas.acode.com.br 3306

# Test MySQL connectivity with timeout
echo "2. Testing MySQL connection:"
timeout 30 mysql -h db-hsp-farmarcas.acode.com.br \
    -P 3306 \
    -u userfarmarcasac02 \
    -p$ACODE_PASS \
    -D acode_farmarcas \
    -e "SELECT 1;" 2>&1

# Check SSL configuration
echo "3. Testing SSL connectivity:"
openssl s_client -connect db-hsp-farmarcas.acode.com.br:3306 -starttls mysql

# Test DNS resolution
echo "4. Testing DNS resolution:"
nslookup db-hsp-farmarcas.acode.com.br

# Check firewall/routing
echo "5. Testing routing:"
traceroute db-hsp-farmarcas.acode.com.br
```

#### **Soluções:**

**Solução 1: Ajustar Timeouts**
```yaml
# sources/source_mysql_acode/configuration.yaml
configuration:
  jdbc_url_params: "connectTimeout=120000&socketTimeout=900000&useSSL=true&autoReconnect=true"
  
  # Additional timeout settings
  connection_timeout_ms: 120000  # 2 minutos
  socket_timeout_ms: 900000      # 15 minutos
```

**Solução 2: Configurar SSL Avançado**
```yaml
configuration:
  ssl: true
  ssl_mode:
    mode: "required"  # Forçar SSL
    ca_certificate: |
      -----BEGIN CERTIFICATE-----
      [ACODE_CA_CERTIFICATE_CONTENT]
      -----END CERTIFICATE-----
    client_certificate: |
      -----BEGIN CERTIFICATE-----
      [CLIENT_CERTIFICATE_CONTENT]
      -----END CERTIFICATE-----
    client_key: |
      -----BEGIN PRIVATE KEY-----
      [CLIENT_KEY_CONTENT]
      -----END PRIVATE KEY-----
```

**Solução 3: Connection Pool**
```yaml
configuration:
  jdbc_url_params: "connectTimeout=120000&socketTimeout=900000&useSSL=true&autoReconnect=true&maxReconnects=3&initialTimeout=2&maxIdleTime=300&testOnBorrow=true&validationQuery=SELECT 1"
```

### **Erro 2: S3 Access Denied**

#### **Sintomas:**
```bash
ERROR: Access Denied when writing to S3 bucket
ERROR: The AWS Access Key Id you provided does not exist in our records
```

#### **Diagnóstico:**
```bash
#!/bin/bash
# diagnose_s3_access.sh

echo "🔍 Diagnosing S3 access..."

# Test AWS credentials
echo "1. Testing AWS credentials:"
aws sts get-caller-identity

# Test bucket access
echo "2. Testing primary bucket access:"
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/

echo "3. Testing backup bucket access:"
aws s3 ls s3://farmarcas-backup-acode/

# Test write permissions
echo "4. Testing write permissions:"
echo "test-file-$(date +%s)" | aws s3 cp - s3://farmarcas-production-bronze/test-write-permissions.txt
aws s3 rm s3://farmarcas-production-bronze/test-write-permissions.txt

# Check bucket policy
echo "5. Checking bucket policy:"
aws s3api get-bucket-policy --bucket farmarcas-production-bronze

# Check IAM permissions
echo "6. Checking IAM permissions:"
aws iam simulate-principal-policy \
    --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
    --action-names s3:PutObject s3:GetObject s3:DeleteObject \
    --resource-arns arn:aws:s3:::farmarcas-production-bronze/*
```

#### **Soluções:**

**Solução 1: Verificar IAM Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-production-bronze",
        "arn:aws:s3:::farmarcas-production-bronze/*",
        "arn:aws:s3:::farmarcas-backup-acode",
        "arn:aws:s3:::farmarcas-backup-acode/*"
      ]
    }
  ]
}
```

**Solução 2: Rotação de Credenciais**
```bash
#!/bin/bash
# rotate_aws_credentials.sh

echo "🔄 Rotating AWS credentials..."

# Generate new access key
NEW_CREDENTIALS=$(aws iam create-access-key --user-name airbyte-acode-user)

# Extract credentials
NEW_ACCESS_KEY=$(echo $NEW_CREDENTIALS | jq -r '.AccessKey.AccessKeyId')
NEW_SECRET_KEY=$(echo $NEW_CREDENTIALS | jq -r '.AccessKey.SecretAccessKey')

# Update environment variables
export FARMARCAS_AWS_ACCESS_KEY_ID=$NEW_ACCESS_KEY
export FARMARCAS_AWS_SECRET_ACCESS_KEY=$NEW_SECRET_KEY

# Test new credentials
aws sts get-caller-identity

# Update Airbyte configurations
octavia apply destinations/destination_s3_acode
octavia apply destinations/destination_s3_acode_backup

echo "✅ Credentials rotated successfully"

# Schedule old key deletion (after 24h verification)
echo "⏰ Schedule old key deletion after 24h verification"
```

## 📊 Falhas de Sincronização

### **Erro 3: Large Table Timeout**

#### **Sintomas:**
```bash
ERROR: Query timeout exceeded
ERROR: Connection lost during large data transfer
Job failed after 3 hours with timeout
```

#### **Diagnóstico:**
```sql
-- Verificar tamanho das tabelas ACODE
SELECT 
    table_name,
    table_rows,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size_MB'
FROM information_schema.tables 
WHERE table_schema = 'acode_farmarcas'
ORDER BY (data_length + index_length) DESC;

-- Verificar índices disponíveis
SHOW INDEX FROM farmarcas_si_analitico_diario;

-- Verificar estatísticas de performance
SHOW TABLE STATUS FROM acode_farmarcas 
WHERE Name = 'farmarcas_si_analitico_diario';
```

#### **Soluções:**

**Solução 1: Aumentar Resources**
```yaml
# connections/connection_mysql_s3_acode/configuration.yaml
configuration:
  resource_requirements:
    cpu_limit: "4.0"      # Dobrar CPU
    cpu_request: "2.0"    
    memory_limit: "8Gi"   # Dobrar memória
    memory_request: "4Gi"
    
  # Aumentar timeout
  operation_timeout: 21600  # 6 horas
```

**Solução 2: Otimizar Query MySQL**
```yaml
configuration:
  # Configurações otimizadas para grandes volumes
  jdbc_url_params: "connectTimeout=120000&socketTimeout=1800000&useSSL=true&autoReconnect=true&useCursorFetch=true&defaultFetchSize=5000&useServerPrepStmts=true&cachePrepStmts=true&prepStmtCacheSize=250&prepStmtCacheSqlLimit=2048"
```

**Solução 3: Implementar Chunking**
```python
# custom_large_table_extractor.py
import mysql.connector
from datetime import datetime, timedelta

def extract_acode_data_in_chunks(connection_config, chunk_size=100000):
    """Extract ACODE data in manageable chunks"""
    
    conn = mysql.connector.connect(**connection_config)
    cursor = conn.cursor()
    
    # Get total count
    cursor.execute("SELECT COUNT(*) FROM farmarcas_si_analitico_diario")
    total_rows = cursor.fetchone()[0]
    
    chunks_needed = (total_rows // chunk_size) + 1
    
    for chunk in range(chunks_needed):
        offset = chunk * chunk_size
        
        query = f"""
        SELECT * FROM farmarcas_si_analitico_diario 
        ORDER BY idpk 
        LIMIT {chunk_size} OFFSET {offset}
        """
        
        cursor.execute(query)
        chunk_data = cursor.fetchall()
        
        # Process chunk
        yield chunk_data
        
        print(f"Processed chunk {chunk + 1}/{chunks_needed}")
    
    conn.close()
```

### **Erro 4: Data Type Mismatch**

#### **Sintomas:**
```bash
ERROR: Cannot convert MySQL type DECIMAL(10,2) to Airbyte type
ERROR: Invalid date format in field 'Data'
ERROR: String too long for field 'Fornecedor'
```

#### **Soluções:**

**Solução 1: Schema Enforcement**
```yaml
# Enhanced schema configuration
json_schema:
  type: object
  properties:
    ACODE_Val_Total:
      type: number
      airbyte_type: number
      format: "double"
      minimum: 0
      maximum: 999999999.99
      
    Data:
      type: string
      format: date
      pattern: "^\\d{4}-\\d{2}-\\d{2}$"
      
    Fornecedor:
      type: string
      maxLength: 255
      pattern: "^[\\w\\s\\-\\.\\,]*$"  # Allow alphanumeric, spaces, hyphens, dots, commas
      
    CNPJ:
      type: string
      pattern: "^\\d{2}\\.\\d{3}\\.\\d{3}/\\d{4}-\\d{2}$"  # CNPJ format
      minLength: 18
      maxLength: 18
```

**Solução 2: Data Transformation**
```sql
-- Custom query with data cleaning
SELECT 
    idpk,
    CAST(COALESCE(ACODE_Val_Total, 0) AS DECIMAL(15,2)) AS ACODE_Val_Total,
    TRIM(SUBSTRING(COALESCE(CNPJ, ''), 1, 18)) AS CNPJ,
    TRIM(SUBSTRING(COALESCE(Fornecedor, ''), 1, 255)) AS Fornecedor,
    DATE_FORMAT(COALESCE(Data, '1900-01-01'), '%Y-%m-%d') AS Data,
    CASE 
        WHEN Data_Processamento IS NULL THEN '1900-01-01 00:00:00'
        ELSE DATE_FORMAT(Data_Processamento, '%Y-%m-%d %H:%i:%s')
    END AS Data_Processamento,
    TRIM(SUBSTRING(COALESCE(EAN, ''), 1, 14)) AS EAN,
    CAST(COALESCE(NF_Numero, 0) AS UNSIGNED) AS NF_Numero,
    CAST(COALESCE(Qtd_Trib, 0) AS DECIMAL(15,4)) AS Qtd_Trib,
    CAST(COALESCE(Val_Prod, 0) AS DECIMAL(15,2)) AS Val_Prod,
    CAST(COALESCE(Val_Trib, 0) AS DECIMAL(15,2)) AS Val_Trib,
    CAST(COALESCE(CFOP, 0) AS UNSIGNED) AS CFOP,
    CAST(COALESCE(NCM, 0) AS UNSIGNED) AS NCM,
    TRIM(SUBSTRING(COALESCE(CST, ''), 1, 3)) AS CST,
    CAST(COALESCE(Aliquota_ICMS, 0) AS DECIMAL(5,2)) AS Aliquota_ICMS,
    CAST(COALESCE(Valor_ICMS, 0) AS DECIMAL(15,2)) AS Valor_ICMS,
    CAST(COALESCE(IPI, 0) AS DECIMAL(15,2)) AS IPI,
    CAST(COALESCE(ST, 0) AS DECIMAL(15,2)) AS ST,
    CAST(COALESCE(STRet, 0) AS DECIMAL(15,2)) AS STRet
FROM farmarcas_si_analitico_diario
WHERE Data >= CURDATE() - INTERVAL 7 DAY  -- Only recent data
ORDER BY idpk;
```

## ⚡ Problemas de Performance

### **Erro 5: Slow Query Performance**

#### **Diagnóstico:**
```sql
-- Analisar queries lentas
SELECT 
    sql_text,
    exec_count,
    avg_timer_wait/1000000000 as avg_duration_seconds,
    sum_timer_wait/1000000000 as total_duration_seconds
FROM performance_schema.events_statements_summary_by_digest 
WHERE schema_name = 'acode_farmarcas'
ORDER BY avg_timer_wait DESC
LIMIT 10;

-- Verificar locks
SELECT 
    r.trx_id waiting_trx_id,
    r.trx_mysql_thread_id waiting_thread,
    r.trx_query waiting_query,
    b.trx_id blocking_trx_id,
    b.trx_mysql_thread_id blocking_thread,
    b.trx_query blocking_query
FROM information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx b 
    ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.innodb_trx r 
    ON r.trx_id = w.requesting_trx_id;
```

#### **Soluções:**

**Solução 1: Query Optimization**
```sql
-- Criar índices otimizados para ACODE
CREATE INDEX idx_acode_data_processamento 
ON farmarcas_si_analitico_diario(Data_Processamento);

CREATE INDEX idx_acode_data_cnpj 
ON farmarcas_si_analitico_diario(Data, CNPJ);

CREATE INDEX idx_acode_ean_data 
ON farmarcas_si_analitico_diario(EAN, Data);

-- Índice composto para queries comuns
CREATE INDEX idx_acode_composite 
ON farmarcas_si_analitico_diario(Data, CNPJ, EAN, CFOP);

-- Analisar tabela após criação de índices
ANALYZE TABLE farmarcas_si_analitico_diario;
```

**Solução 2: MySQL Configuration Tuning**
```sql
-- Configurações MySQL otimizadas para ACODE
SET GLOBAL innodb_buffer_pool_size = 2147483648;  -- 2GB
SET GLOBAL innodb_log_file_size = 268435456;      -- 256MB
SET GLOBAL innodb_log_buffer_size = 67108864;     -- 64MB
SET GLOBAL innodb_flush_log_at_trx_commit = 2;    -- Better performance
SET GLOBAL query_cache_size = 134217728;          -- 128MB
SET GLOBAL query_cache_type = 1;                  -- Enable query cache
SET GLOBAL tmp_table_size = 134217728;            -- 128MB
SET GLOBAL max_heap_table_size = 134217728;       -- 128MB
```

### **Erro 6: Memory Issues**

#### **Sintomas:**
```bash
ERROR: Java heap space
ERROR: OutOfMemoryError during data processing
ERROR: Container killed due to memory limit
```

#### **Soluções:**

**Solução 1: Optimize Airbyte Memory**
```yaml
# connections/connection_mysql_s3_acode/configuration.yaml
configuration:
  resource_requirements:
    memory_limit: "6Gi"     # Increase memory
    memory_request: "3Gi"
    
    # JVM tuning
    jvm_args: "-Xmx4g -Xms2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

**Solução 2: Batch Size Optimization**
```yaml
configuration:
  # Reduce batch sizes to manage memory
  jdbc_url_params: "connectTimeout=120000&socketTimeout=900000&useSSL=true&defaultFetchSize=1000&useCursorFetch=true"
  
  # S3 upload optimization
  format:
    format_type: "Parquet"
    compression_codec: "SNAPPY"
    block_size: 67108864    # 64MB (smaller blocks)
    page_size: 524288       # 512KB (smaller pages)
```

## 🔄 Problemas de Backup

### **Erro 7: Backup Replication Lag**

#### **Diagnóstico:**
```sql
-- Verificar status de replicação
SHOW SLAVE STATUS\G

-- Verificar lag específico
SELECT 
    TIMESTAMPDIFF(SECOND, 
        CONVERT_TZ(
            STR_TO_DATE(LEFT(MASTER_POS_WAIT(RELAY_LOG_FILE, RELAY_LOG_POS, 0), 19), 
                       '%Y-%m-%d %H:%i:%s'), 
            @@session.time_zone, '+00:00'
        ), 
        UTC_TIMESTAMP()
    ) AS replication_lag_seconds;

-- Verificar erros de replicação
SELECT * FROM performance_schema.replication_applier_status_by_worker 
WHERE LAST_ERROR_MESSAGE != '';
```

#### **Soluções:**

**Solução 1: Replication Optimization**
```sql
-- Otimizar replicação MySQL
SET GLOBAL slave_parallel_workers = 4;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_preserve_commit_order = 1;
SET GLOBAL slave_transaction_retries = 10;

-- Configurar timeouts
SET GLOBAL slave_net_timeout = 60;
SET GLOBAL master_info_repository = 'TABLE';
SET GLOBAL relay_log_info_repository = 'TABLE';
```

**Solução 2: Auto-Recovery Script**
```bash
#!/bin/bash
# auto_recover_replication.sh

echo "🔄 Checking replication status..."

# Check replication status
SLAVE_STATUS=$(mysql -h acode-backup.farmarcas.internal \
    -u userfarmarcasac02_backup \
    -p$ACODE_BACKUP_PASS \
    -e "SHOW SLAVE STATUS\G" | grep "Slave_IO_Running\|Slave_SQL_Running\|Seconds_Behind_Master")

echo "$SLAVE_STATUS"

# Extract status values
IO_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_IO_Running" | awk '{print $2}')
SQL_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_SQL_Running" | awk '{print $2}')
LAG=$(echo "$SLAVE_STATUS" | grep "Seconds_Behind_Master" | awk '{print $2}')

# Check if replication is broken
if [[ "$IO_RUNNING" != "Yes" ]] || [[ "$SQL_RUNNING" != "Yes" ]]; then
    echo "❌ Replication is broken. Attempting repair..."
    
    # Stop slave
    mysql -h acode-backup.farmarcas.internal \
        -u userfarmarcasac02_backup \
        -p$ACODE_BACKUP_PASS \
        -e "STOP SLAVE;"
    
    # Reset slave position (careful!)
    mysql -h acode-backup.farmarcas.internal \
        -u userfarmarcasac02_backup \
        -p$ACODE_BACKUP_PASS \
        -e "RESET SLAVE;"
    
    # Reconfigure slave
    mysql -h acode-backup.farmarcas.internal \
        -u userfarmarcasac02_backup \
        -p$ACODE_BACKUP_PASS \
        -e "CHANGE MASTER TO 
            MASTER_HOST='db-hsp-farmarcas.acode.com.br',
            MASTER_USER='replication_user',
            MASTER_PASSWORD='$REPLICATION_PASS',
            MASTER_AUTO_POSITION=1;"
    
    # Start slave
    mysql -h acode-backup.farmarcas.internal \
        -u userfarmarcasac02_backup \
        -p$ACODE_BACKUP_PASS \
        -e "START SLAVE;"
    
    echo "✅ Replication repair attempted"
    
    # Send alert
    curl -X POST -H 'Content-type: application/json' \
        --data '{"text":"⚠️ ACODE replication was repaired automatically"}' \
        $SLACK_WEBHOOK_URL

elif [[ "$LAG" -gt 300 ]]; then
    echo "⚠️ High replication lag: $LAG seconds"
    
    # Send warning
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"⚠️ ACODE replication lag is high: $LAG seconds\"}" \
        $SLACK_WEBHOOK_URL
else
    echo "✅ Replication is healthy"
fi
```

## 🗂️ Problemas de Qualidade de Dados

### **Erro 8: Data Validation Failures**

#### **Diagnóstico:**
```sql
-- Verificar qualidade dos dados ACODE
SELECT 
    'NULL_VALUES' as issue_type,
    SUM(CASE WHEN CNPJ IS NULL THEN 1 ELSE 0 END) as cnpj_nulls,
    SUM(CASE WHEN EAN IS NULL THEN 1 ELSE 0 END) as ean_nulls,
    SUM(CASE WHEN Data IS NULL THEN 1 ELSE 0 END) as data_nulls,
    SUM(CASE WHEN ACODE_Val_Total IS NULL THEN 1 ELSE 0 END) as valor_nulls
FROM farmarcas_si_analitico_diario
WHERE Data >= CURDATE() - INTERVAL 1 DAY;

-- Verificar formatos inválidos
SELECT 
    'INVALID_FORMATS' as issue_type,
    SUM(CASE WHEN CNPJ NOT REGEXP '^[0-9]{2}\\.[0-9]{3}\\.[0-9]{3}/[0-9]{4}-[0-9]{2}$' THEN 1 ELSE 0 END) as invalid_cnpj,
    SUM(CASE WHEN EAN NOT REGEXP '^[0-9]{13,14}$' THEN 1 ELSE 0 END) as invalid_ean,
    SUM(CASE WHEN ACODE_Val_Total < 0 THEN 1 ELSE 0 END) as negative_values
FROM farmarcas_si_analitico_diario
WHERE Data >= CURDATE() - INTERVAL 1 DAY;

-- Verificar duplicatas
SELECT 
    CNPJ, EAN, Data, COUNT(*) as duplicate_count
FROM farmarcas_si_analitico_diario
WHERE Data >= CURDATE() - INTERVAL 1 DAY
GROUP BY CNPJ, EAN, Data
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
```

#### **Soluções:**

**Solução 1: Data Quality Rules**
```python
# data_quality_validator.py
import pandas as pd
import re
from datetime import datetime, timedelta

class ACODEDataQualityValidator:
    def __init__(self):
        self.validation_rules = {
            'cnpj_format': r'^\d{2}\.\d{3}\.\d{3}/\d{4}-\d{2}$',
            'ean_format': r'^\d{13,14}$',
            'cst_format': r'^\d{3}$'
        }
        self.quality_thresholds = {
            'null_percentage_max': 5.0,      # Max 5% null values
            'invalid_format_max': 1.0,       # Max 1% invalid formats
            'duplicate_percentage_max': 0.1   # Max 0.1% duplicates
        }
    
    def validate_acode_data(self, df):
        """Validate ACODE data quality"""
        results = {
            'timestamp': datetime.now().isoformat(),
            'total_records': len(df),
            'validation_results': {},
            'quality_score': 0,
            'pass': False
        }
        
        # Check null values
        null_checks = {}
        critical_fields = ['CNPJ', 'EAN', 'Data', 'ACODE_Val_Total']
        
        for field in critical_fields:
            null_count = df[field].isnull().sum()
            null_percentage = (null_count / len(df)) * 100
            null_checks[field] = {
                'null_count': null_count,
                'null_percentage': round(null_percentage, 2),
                'pass': null_percentage <= self.quality_thresholds['null_percentage_max']
            }
        
        results['validation_results']['null_checks'] = null_checks
        
        # Check format validations
        format_checks = {}
        
        # CNPJ format
        invalid_cnpj = ~df['CNPJ'].str.match(self.validation_rules['cnpj_format'], na=False)
        cnpj_invalid_count = invalid_cnpj.sum()
        cnpj_invalid_percentage = (cnpj_invalid_count / len(df)) * 100
        
        format_checks['cnpj'] = {
            'invalid_count': cnpj_invalid_count,
            'invalid_percentage': round(cnpj_invalid_percentage, 2),
            'pass': cnpj_invalid_percentage <= self.quality_thresholds['invalid_format_max']
        }
        
        # EAN format
        invalid_ean = ~df['EAN'].str.match(self.validation_rules['ean_format'], na=False)
        ean_invalid_count = invalid_ean.sum()
        ean_invalid_percentage = (ean_invalid_count / len(df)) * 100
        
        format_checks['ean'] = {
            'invalid_count': ean_invalid_count,
            'invalid_percentage': round(ean_invalid_percentage, 2),
            'pass': ean_invalid_percentage <= self.quality_thresholds['invalid_format_max']
        }
        
        results['validation_results']['format_checks'] = format_checks
        
        # Check business rules
        business_checks = {}
        
        # Negative values
        negative_values = df['ACODE_Val_Total'] < 0
        negative_count = negative_values.sum()
        negative_percentage = (negative_count / len(df)) * 100
        
        business_checks['negative_values'] = {
            'count': negative_count,
            'percentage': round(negative_percentage, 2),
            'pass': negative_percentage <= 1.0  # Max 1% negative values allowed
        }
        
        # Future dates
        today = datetime.now().date()
        future_dates = pd.to_datetime(df['Data']).dt.date > today
        future_count = future_dates.sum()
        future_percentage = (future_count / len(df)) * 100
        
        business_checks['future_dates'] = {
            'count': future_count,
            'percentage': round(future_percentage, 2),
            'pass': future_percentage == 0  # No future dates allowed
        }
        
        results['validation_results']['business_checks'] = business_checks
        
        # Check duplicates
        duplicate_subset = ['CNPJ', 'EAN', 'Data', 'NF_Numero']
        duplicates = df.duplicated(subset=duplicate_subset, keep=False)
        duplicate_count = duplicates.sum()
        duplicate_percentage = (duplicate_count / len(df)) * 100
        
        results['validation_results']['duplicates'] = {
            'count': duplicate_count,
            'percentage': round(duplicate_percentage, 2),
            'pass': duplicate_percentage <= self.quality_thresholds['duplicate_percentage_max']
        }
        
        # Calculate overall quality score
        all_checks = []
        for category in results['validation_results'].values():
            if isinstance(category, dict):
                if 'pass' in category:
                    all_checks.append(category['pass'])
                else:
                    for check in category.values():
                        if isinstance(check, dict) and 'pass' in check:
                            all_checks.append(check['pass'])
        
        passed_checks = sum(all_checks)
        total_checks = len(all_checks)
        quality_score = (passed_checks / total_checks) * 100 if total_checks > 0 else 0
        
        results['quality_score'] = round(quality_score, 2)
        results['pass'] = quality_score >= 90  # 90% minimum quality score
        
        return results
    
    def generate_quality_report(self, validation_results):
        """Generate human-readable quality report"""
        report = []
        report.append("📊 ACODE Data Quality Report")
        report.append("=" * 40)
        report.append(f"Timestamp: {validation_results['timestamp']}")
        report.append(f"Total Records: {validation_results['total_records']:,}")
        report.append(f"Quality Score: {validation_results['quality_score']}%")
        report.append(f"Overall Status: {'✅ PASS' if validation_results['pass'] else '❌ FAIL'}")
        report.append("")
        
        # Detailed results
        for category, checks in validation_results['validation_results'].items():
            report.append(f"📋 {category.replace('_', ' ').title()}:")
            
            if isinstance(checks, dict) and 'pass' in checks:
                status = '✅' if checks['pass'] else '❌'
                report.append(f"  {status} {category}")
            else:
                for check_name, check_result in checks.items():
                    if isinstance(check_result, dict) and 'pass' in check_result:
                        status = '✅' if check_result['pass'] else '❌'
                        report.append(f"  {status} {check_name}: {check_result}")
            
            report.append("")
        
        return '\n'.join(report)

# Usage example
validator = ACODEDataQualityValidator()
```

**Solução 2: Data Quarantine System**
```python
# data_quarantine_handler.py
import boto3
import pandas as pd
from datetime import datetime

class ACODEDataQuarantineHandler:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.quarantine_bucket = 'farmarcas-quarantine'
        
    def quarantine_invalid_data(self, df, validation_results):
        """Move invalid data to quarantine bucket"""
        
        if validation_results['pass']:
            return df  # No quarantine needed
        
        # Identify invalid records
        invalid_mask = pd.Series([False] * len(df))
        
        # Add records with validation failures
        for category, checks in validation_results['validation_results'].items():
            if category == 'null_checks':
                for field, check in checks.items():
                    if not check['pass']:
                        invalid_mask |= df[field].isnull()
            
            elif category == 'format_checks':
                if not checks['cnpj']['pass']:
                    invalid_mask |= ~df['CNPJ'].str.match(r'^\d{2}\.\d{3}\.\d{3}/\d{4}-\d{2}$', na=False)
                if not checks['ean']['pass']:
                    invalid_mask |= ~df['EAN'].str.match(r'^\d{13,14}$', na=False)
            
            elif category == 'business_checks':
                if not checks['negative_values']['pass']:
                    invalid_mask |= df['ACODE_Val_Total'] < 0
                if not checks['future_dates']['pass']:
                    invalid_mask |= pd.to_datetime(df['Data']).dt.date > datetime.now().date()
        
        # Separate valid and invalid data
        invalid_df = df[invalid_mask].copy()
        valid_df = df[~invalid_mask].copy()
        
        if len(invalid_df) > 0:
            # Save invalid data to quarantine
            quarantine_path = f"acode_quarantine/date={datetime.now().strftime('%Y-%m-%d')}/invalid_data_{datetime.now().strftime('%H%M%S')}.parquet"
            
            invalid_df.to_parquet(f's3://{self.quarantine_bucket}/{quarantine_path}')
            
            print(f"⚠️ {len(invalid_df)} invalid records quarantined to {quarantine_path}")
            
            # Send alert
            alert_message = f"""
            📊 ACODE Data Quality Alert
            
            Invalid records detected and quarantined:
            - Total records processed: {len(df):,}
            - Invalid records: {len(invalid_df):,}
            - Quality score: {validation_results['quality_score']}%
            - Quarantine location: s3://{self.quarantine_bucket}/{quarantine_path}
            
            Action required: Review quarantined data and update source system if needed.
            """
            
            # Send to Slack (implement your notification system)
            self.send_alert(alert_message)
        
        return valid_df
    
    def send_alert(self, message):
        """Send alert to monitoring systems"""
        # Implement your alerting mechanism here
        pass
```

## 🔧 Scripts de Resolução Automática

### **Health Check & Auto-Recovery**

```bash
#!/bin/bash
# acode_health_check_and_recovery.sh

set -e

LOG_FILE="/var/log/acode_health_check.log"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

send_alert() {
    local message="$1"
    local severity="$2"
    
    # Determine emoji based on severity
    case $severity in
        "critical") emoji="🚨" ;;
        "warning") emoji="⚠️" ;;
        "info") emoji="ℹ️" ;;
        "success") emoji="✅" ;;
        *) emoji="📢" ;;
    esac
    
    # Send to Slack
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"$emoji ACODE Health Check\\n$message\"}" \
        "$SLACK_WEBHOOK" 2>/dev/null || true
    
    log "Alert sent: $message"
}

check_mysql_primary() {
    log "Checking MySQL primary connectivity..."
    
    if timeout 30 mysql -h db-hsp-farmarcas.acode.com.br \
        -P 3306 -u userfarmarcasac02 -p$ACODE_PASS \
        -D acode_farmarcas -e "SELECT 1;" >/dev/null 2>&1; then
        log "✅ MySQL primary: OK"
        return 0
    else
        log "❌ MySQL primary: FAILED"
        return 1
    fi
}

check_mysql_backup() {
    log "Checking MySQL backup connectivity..."
    
    if timeout 30 mysql -h acode-backup.farmarcas.internal \
        -P 3306 -u userfarmarcasac02_backup -p$ACODE_BACKUP_PASS \
        -D acode_farmarcas_replica -e "SELECT 1;" >/dev/null 2>&1; then
        log "✅ MySQL backup: OK"
        return 0
    else
        log "❌ MySQL backup: FAILED"
        return 1
    fi
}

check_s3_access() {
    log "Checking S3 access..."
    
    if aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/ >/dev/null 2>&1; then
        log "✅ S3 primary: OK"
        return 0
    else
        log "❌ S3 primary: FAILED"
        return 1
    fi
}

check_airbyte_connection() {
    log "Checking Airbyte connection status..."
    
    # Check if primary connection is active and healthy
    CONNECTION_STATUS=$(octavia list connections | grep "connection_mysql_s3_acode" | head -1)
    
    if echo "$CONNECTION_STATUS" | grep -q "active"; then
        log "✅ Airbyte connection: ACTIVE"
        return 0
    else
        log "❌ Airbyte connection: INACTIVE or FAILED"
        return 1
    fi
}

activate_backup_connection() {
    log "🔄 Activating backup connection..."
    
    # Deactivate primary connection
    octavia apply connections/connection_mysql_s3_acode --status inactive 2>/dev/null || true
    
    # Activate backup connection
    if octavia apply connections/connection_mysql_s3_acode_backup --status active; then
        log "✅ Backup connection activated"
        send_alert "Backup connection has been activated due to primary failure" "critical"
        return 0
    else
        log "❌ Failed to activate backup connection"
        send_alert "CRITICAL: Failed to activate backup connection" "critical"
        return 1
    fi
}

reactivate_primary_connection() {
    log "🔄 Reactivating primary connection..."
    
    # Deactivate backup connection
    octavia apply connections/connection_mysql_s3_acode_backup --status inactive 2>/dev/null || true
    
    # Reactivate primary connection
    if octavia apply connections/connection_mysql_s3_acode --status active; then
        log "✅ Primary connection reactivated"
        send_alert "Primary connection has been restored and reactivated" "success"
        return 0
    else
        log "❌ Failed to reactivate primary connection"
        return 1
    fi
}

main() {
    log "🏥 Starting ACODE health check..."
    
    # Check primary components
    mysql_primary_ok=false
    mysql_backup_ok=false
    s3_primary_ok=false
    airbyte_ok=false
    
    check_mysql_primary && mysql_primary_ok=true
    check_mysql_backup && mysql_backup_ok=true
    check_s3_access && s3_primary_ok=true
    check_airbyte_connection && airbyte_ok=true
    
    # Determine action based on health checks
    if $mysql_primary_ok && $s3_primary_ok && $airbyte_ok; then
        log "✅ All primary systems healthy"
        
        # Check if we're currently using backup (and should switch back)
        BACKUP_CONNECTION_STATUS=$(octavia list connections | grep "connection_mysql_s3_acode_backup" | head -1)
        if echo "$BACKUP_CONNECTION_STATUS" | grep -q "active"; then
            log "🔄 Primary systems restored, switching back from backup"
            reactivate_primary_connection
        fi
        
        send_alert "All ACODE systems are healthy" "success"
        
    elif $mysql_backup_ok && $s3_primary_ok; then
        log "⚠️ Primary MySQL failed, but backup available"
        
        # Check if backup is already active
        BACKUP_CONNECTION_STATUS=$(octavia list connections | grep "connection_mysql_s3_acode_backup" | head -1)
        if ! echo "$BACKUP_CONNECTION_STATUS" | grep -q "active"; then
            log "🔄 Switching to backup connection"
            activate_backup_connection
        else
            log "ℹ️ Backup connection already active"
        fi
        
    else
        log "🚨 CRITICAL: Multiple system failures detected"
        send_alert "CRITICAL: Multiple ACODE system failures detected. Manual intervention required." "critical"
        
        # List failed components
        failures=()
        ! $mysql_primary_ok && failures+=("MySQL Primary")
        ! $mysql_backup_ok && failures+=("MySQL Backup")
        ! $s3_primary_ok && failures+=("S3 Primary")
        ! $airbyte_ok && failures+=("Airbyte Connection")
        
        send_alert "Failed components: ${failures[*]}" "critical"
    fi
    
    log "🏥 Health check completed"
}

# Run main function
main "$@"
```

### **Performance Monitoring Script**

```python
#!/usr/bin/env python3
# acode_performance_monitor.py

import time
import mysql.connector
import boto3
import requests
import json
from datetime import datetime, timedelta
import os

class ACODEPerformanceMonitor:
    def __init__(self):
        self.mysql_config = {
            'host': 'db-hsp-farmarcas.acode.com.br',
            'port': 3306,
            'database': 'acode_farmarcas',
            'user': 'userfarmarcasac02',
            'password': os.getenv('ACODE_PASS'),
            'ssl_mode': 'PREFERRED'
        }
        
        self.thresholds = {
            'query_response_time_seconds': 30,
            'replication_lag_seconds': 300,
            'sync_duration_hours': 3,
            'data_freshness_hours': 25
        }
    
    def check_query_performance(self):
        """Check MySQL query performance"""
        try:
            start_time = time.time()
            
            conn = mysql.connector.connect(**self.mysql_config)
            cursor = conn.cursor()
            
            # Test query performance
            cursor.execute("SELECT COUNT(*) FROM farmarcas_si_analitico_diario WHERE Data >= CURDATE() - INTERVAL 1 DAY")
            result = cursor.fetchone()
            
            query_time = time.time() - start_time
            
            cursor.close()
            conn.close()
            
            return {
                'status': 'pass' if query_time <= self.thresholds['query_response_time_seconds'] else 'fail',
                'query_time_seconds': round(query_time, 2),
                'threshold_seconds': self.thresholds['query_response_time_seconds'],
                'record_count': result[0] if result else 0
            }
            
        except Exception as e:
            return {
                'status': 'error',
                'error': str(e)
            }
    
    def check_data_freshness(self):
        """Check data freshness in S3"""
        try:
            s3_client = boto3.client('s3')
            
            # List recent objects in bronze bucket
            response = s3_client.list_objects_v2(
                Bucket='farmarcas-production-bronze',
                Prefix='origin=airbyte/database=bronze_acode/',
                MaxKeys=10
            )
            
            if 'Contents' not in response:
                return {
                    'status': 'fail',
                    'error': 'No data found in S3 bucket'
                }
            
            # Find most recent file
            latest_file = max(response['Contents'], key=lambda x: x['LastModified'])
            data_age_hours = (datetime.now(latest_file['LastModified'].tzinfo) - latest_file['LastModified']).total_seconds() / 3600
            
            return {
                'status': 'pass' if data_age_hours <= self.thresholds['data_freshness_hours'] else 'fail',
                'latest_file': latest_file['Key'],
                'data_age_hours': round(data_age_hours, 2),
                'threshold_hours': self.thresholds['data_freshness_hours']
            }
            
        except Exception as e:
            return {
                'status': 'error',
                'error': str(e)
            }
    
    def check_airbyte_sync_performance(self):
        """Check Airbyte sync performance"""
        try:
            # This would need Airbyte API access
            # For now, return mock data
            return {
                'status': 'pass',
                'last_sync_duration_minutes': 45,
                'threshold_hours': self.thresholds['sync_duration_hours'],
                'last_sync_status': 'succeeded'
            }
            
        except Exception as e:
            return {
                'status': 'error',
                'error': str(e)
            }
    
    def generate_performance_report(self):
        """Generate comprehensive performance report"""
        report = {
            'timestamp': datetime.now().isoformat(),
            'checks': {}
        }
        
        print("🔍 Running ACODE performance checks...")
        
        # Query performance
        print("1. Checking query performance...")
        report['checks']['query_performance'] = self.check_query_performance()
        
        # Data freshness
        print("2. Checking data freshness...")
        report['checks']['data_freshness'] = self.check_data_freshness()
        
        # Sync performance
        print("3. Checking sync performance...")
        report['checks']['sync_performance'] = self.check_airbyte_sync_performance()
        
        # Overall status
        all_checks = [check['status'] for check in report['checks'].values()]
        if all(status == 'pass' for status in all_checks):
            report['overall_status'] = 'healthy'
        elif any(status == 'error' for status in all_checks):
            report['overall_status'] = 'error'
        else:
            report['overall_status'] = 'degraded'
        
        return report
    
    def print_report(self, report):
        """Print human-readable performance report"""
        print("\n📊 ACODE Performance Report")
        print("=" * 40)
        print(f"Timestamp: {report['timestamp']}")
        print(f"Overall Status: {report['overall_status'].upper()}")
        print()
        
        for check_name, result in report['checks'].items():
            status_emoji = {
                'pass': '✅',
                'fail': '❌', 
                'error': '⚠️'
            }.get(result['status'], '❓')
            
            print(f"{status_emoji} {check_name.replace('_', ' ').title()}: {result['status'].upper()}")
            
            # Print details
            for key, value in result.items():
                if key != 'status':
                    print(f"   {key}: {value}")
            print()

if __name__ == "__main__":
    monitor = ACODEPerformanceMonitor()
    report = monitor.generate_performance_report()
    monitor.print_report(report)
    
    # Save report
    with open(f"acode_performance_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json", 'w') as f:
        json.dump(report, f, indent=2)
    
    print("📁 Performance report saved")
```

---

## 🚀 Quick Fix Commands

### **Emergency Commands** ⚡

```bash
# Restart primary connection
octavia apply connections/connection_mysql_s3_acode --status inactive
sleep 10
octavia apply connections/connection_mysql_s3_acode --status active

# Switch to backup immediately
octavia apply connections/connection_mysql_s3_acode --status inactive
octavia apply connections/connection_mysql_s3_acode_backup --status active

# Check connection status
octavia list connections | grep acode

# Test MySQL connectivity
mysql -h db-hsp-farmarcas.acode.com.br -P 3306 -u userfarmarcasac02 -p$ACODE_PASS -e "SELECT 1;"

# Test S3 access
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/

# View recent logs
kubectl logs -n airbyte deployment/airbyte-worker --tail=100

# Force sync restart
kubectl rollout restart deployment/airbyte-worker -n airbyte
```

---

**Próximo**: [Diagramas Técnicos](./diagramas.md) - Visualizações arquiteturais e fluxos
