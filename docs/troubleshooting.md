# ⚠️ Troubleshooting - Cognitivo Data Platform

## 🚨 Problemas Críticos (P1)

### **ACODE Sync Failure**
**Sintomas**: Pipeline ACODE não executa ou falha repetidamente

#### Diagnóstico
```bash
# Verificar status da connection
curl -X GET "https://airbyte.farmarcas.com/api/v1/connections/connection_mysql_s3_acode" \
  -H "Authorization: Bearer $AIRBYTE_TOKEN"

# Testar conectividade MySQL
telnet db-hsp-farmarcas.acode.com.br 3306

# Verificar logs do Airbyte
kubectl logs -n plataforma -l app=airbyte-worker --tail=100
```

#### Soluções
```bash
# 1. Reset da connection
./scripts/acode-reset-connection.sh

# 2. Verificar credenciais
echo $ACODE_PASS | base64 -d

# 3. Trigger manual
./scripts/acode-manual-sync.sh

# 4. Ativar redundância
./scripts/acode-enable-backup.sh
```

### **S3 Access Denied**
**Sintomas**: Falha no upload para S3, erro 403

#### Diagnóstico
```bash
# Verificar credenciais AWS
aws sts get-caller-identity --profile farmarcas-production

# Testar permissões S3
aws s3 ls s3://farmarcas-production-bronze/ --profile farmarcas-production

# Verificar IAM policies
aws iam list-attached-role-policies --role-name airbyte-s3-role
```

#### Soluções
```bash
# 1. Renovar credentials
aws configure --profile farmarcas-production

# 2. Verificar bucket policy
aws s3api get-bucket-policy --bucket farmarcas-production-bronze

# 3. Testar upload manual
aws s3 cp test-file.txt s3://farmarcas-production-bronze/test/
```

## 🟡 Problemas Altos (P2)

### **Radar Sync Lento**
**Sintomas**: Execução > 2 horas, timeout frequente

#### Diagnóstico
```bash
# Verificar recursos Kubernetes
kubectl top pods -n plataforma

# Analisar métricas de rede
kubectl exec -n plataforma <airbyte-pod> -- netstat -i

# Verificar logs de performance
kubectl logs -n plataforma <airbyte-pod> | grep -i "performance\|slow\|timeout"
```

#### Soluções
```bash
# 1. Aumentar recursos
kubectl patch deployment airbyte-worker -p '{"spec":{"template":{"spec":{"containers":[{"name":"airbyte-worker","resources":{"requests":{"cpu":"2","memory":"4Gi"},"limits":{"cpu":"4","memory":"8Gi"}}}]}}}}'

# 2. Otimizar query MySQL
./scripts/radar-optimize-queries.sh

# 3. Paralelizar tabelas
./scripts/radar-parallel-sync.sh
```

### **Google Drive Rate Limit**
**Sintomas**: Erro 429, quota exceeded

#### Diagnóstico
```bash
# Verificar quota atual
python scripts/check-gdrive-quota.py

# Analisar logs do collector
kubectl logs -n collectors -l app=gdrive-collector --tail=50
```

#### Soluções
```bash
# 1. Implementar exponential backoff
./scripts/gdrive-enable-backoff.sh

# 2. Distribuir requests
./scripts/gdrive-batch-requests.sh

# 3. Usar service account diferente
./scripts/gdrive-rotate-credentials.sh
```

## 🟢 Problemas Médios (P3)

### **Data Quality Issues**
**Sintomas**: Dados inconsistentes ou faltantes

#### Diagnóstico
```sql
-- Verificar completude dos dados
SELECT 
    DATE(cog_dt_ingestion) as date,
    COUNT(*) as records,
    COUNT(DISTINCT source_table) as tables
FROM bronze_layer 
WHERE DATE(cog_dt_ingestion) = CURRENT_DATE()
GROUP BY DATE(cog_dt_ingestion);

-- Identificar valores nulos críticos
SELECT 
    source_table,
    SUM(CASE WHEN critical_field IS NULL THEN 1 ELSE 0 END) as null_count,
    COUNT(*) as total_count
FROM bronze_layer
GROUP BY source_table;
```

#### Soluções
```bash
# 1. Executar validação Soda Core
./scripts/run-data-quality-checks.sh

# 2. Reprocessar dados específicos
./scripts/reprocess-date.sh 2025-08-15

# 3. Notificar stakeholders
./scripts/send-quality-report.sh
```

### **Airflow DAG Stuck**
**Sintomas**: DAG em estado "running" por muito tempo

#### Diagnóstico
```bash
# Verificar DAG state
kubectl exec -n plataforma airflow-scheduler -- airflow dags state dag_sync_connection_mysql_s3_acode

# Analisar task instances
kubectl exec -n plataforma airflow-scheduler -- airflow tasks list dag_sync_connection_mysql_s3_acode --tree
```

#### Soluções
```bash
# 1. Clear DAG run
kubectl exec -n plataforma airflow-scheduler -- airflow dags unpause dag_sync_connection_mysql_s3_acode

# 2. Kill hanging tasks
kubectl exec -n plataforma airflow-scheduler -- airflow tasks clear dag_sync_connection_mysql_s3_acode

# 3. Restart scheduler
kubectl rollout restart deployment/airflow-scheduler -n plataforma
```

## 🔧 Ferramentas de Diagnóstico

### **Health Check Script**
```bash
#!/bin/bash
# health-check-all.sh

echo "🔍 CDP Health Check - $(date)"
echo "================================"

# ACODE
echo "📊 ACODE System:"
if timeout 5 bash -c "</dev/tcp/db-hsp-farmarcas.acode.com.br/3306"; then
    echo "✅ MySQL connectivity: OK"
else
    echo "❌ MySQL connectivity: FAILED"
fi

# Radar  
echo "📡 Radar System:"
if kubectl get pods -n plataforma | grep -q airbyte; then
    echo "✅ Airbyte platform: OK"
else
    echo "❌ Airbyte platform: FAILED"
fi

# Google Drive
echo "📁 Google Drive:"
if python scripts/test-gdrive-connection.py > /dev/null 2>&1; then
    echo "✅ Google Drive API: OK"
else
    echo "❌ Google Drive API: FAILED"  
fi

# S3
echo "☁️ AWS S3:"
if aws s3 ls s3://farmarcas-production-bronze/ > /dev/null 2>&1; then
    echo "✅ S3 access: OK"
else
    echo "❌ S3 access: FAILED"
fi

echo "================================"
```

### **Log Aggregation**
```bash
#!/bin/bash
# collect-logs.sh

NAMESPACE="plataforma"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_DIR="logs_$DATE"

mkdir -p $LOG_DIR

# Airbyte logs
kubectl logs -n $NAMESPACE -l app=airbyte-worker --tail=1000 > $LOG_DIR/airbyte.log

# Airflow logs  
kubectl logs -n $NAMESPACE -l app=airflow-scheduler --tail=1000 > $LOG_DIR/airflow.log

# Collector logs
kubectl logs -n collectors -l app=gdrive-collector --tail=1000 > $LOG_DIR/gdrive.log

# Compress logs
tar -czf logs_$DATE.tar.gz $LOG_DIR/
echo "📦 Logs collected: logs_$DATE.tar.gz"
```

### **Performance Monitor**
```bash
#!/bin/bash
# monitor-performance.sh

while true; do
    echo "$(date): CDP Performance Metrics"
    echo "================================"
    
    # CPU/Memory usage
    kubectl top pods -n plataforma --sort-by=cpu
    
    # Network I/O
    kubectl exec -n plataforma airbyte-worker -- cat /proc/net/dev | grep eth0
    
    # S3 upload rate
    aws cloudwatch get-metric-statistics \
      --namespace AWS/S3 \
      --metric-name NumberOfObjects \
      --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
      --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
      --period 3600 \
      --statistics Sum \
      --dimensions Name=BucketName,Value=farmarcas-production-bronze
    
    sleep 300  # 5 minutos
done
```

## 📞 Escalation Matrix

### **Contatos por Severidade**

#### P1 - Crítico (0-2h SLA)
- **Primary**: Data Engineering Team
- **Secondary**: DevOps Team  
- **Escalation**: CTO
- **Channels**: PagerDuty + Phone + Slack

#### P2 - Alto (2-8h SLA)
- **Primary**: Data Engineering Team
- **Secondary**: Product Team
- **Channels**: Slack + Email

#### P3 - Médio (24h SLA)
- **Primary**: Data Engineering Team
- **Channels**: Slack + JIRA Ticket

### **Runbooks por Sistema**

#### ACODE Emergency Runbook
```bash
# Passo 1: Verificar redundância
./scripts/acode-check-backup-status.sh

# Passo 2: Ativar failover se necessário
./scripts/acode-activate-failover.sh

# Passo 3: Notificar stakeholders
./scripts/notify-acode-incident.sh

# Passo 4: Documentar incident
./scripts/create-incident-report.sh ACODE_SYNC_FAILURE
```

#### Radar Recovery Runbook
```bash
# Passo 1: Identificar tabelas faltantes
./scripts/radar-identify-missing-tables.sh

# Passo 2: Reprocessar tabelas específicas
./scripts/radar-reprocess-tables.sh <table_list>

# Passo 3: Validar integridade
./scripts/radar-validate-data.sh

# Passo 4: Atualizar data catalog
./scripts/radar-refresh-catalog.sh
```

## 📊 Métricas de Troubleshooting

### **MTTR (Mean Time To Resolution)**
- **P1**: Target < 2h | Atual: 1.5h
- **P2**: Target < 8h | Atual: 4h  
- **P3**: Target < 24h | Atual: 12h

### **Incident Categories (Last 30 days)**
- **Connectivity Issues**: 35%
- **Resource Constraints**: 25%
- **Data Quality**: 20%
- **Configuration Errors**: 15%
- **External Dependencies**: 5%

---

**Próximo**: [Monitoramento e Alertas](./monitoramento.md)
