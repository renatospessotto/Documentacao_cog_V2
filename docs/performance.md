# ⚡ Performance - Cognitivo Data Platform

## 📊 Benchmarks de Performance

### **ACODE - Sistema Crítico**
```yaml
sla_targets:
  latency:
    p50: < 100ms  # 50th percentile
    p95: < 500ms  # 95th percentile
    p99: < 1000ms # 99th percentile
  
  throughput:
    peak_transactions: 1000/min
    sustained_load: 500/min
    
  availability:
    uptime: 99.9%
    max_downtime_monthly: 43.2min
```

### **Radar - Alta Performance**
```yaml
sla_targets:
  data_ingestion:
    records_per_second: 10000
    batch_size: 1000 records
    processing_time: < 30s per batch
    
  sync_frequency:
    critical_tables: 15min
    standard_tables: 1h
    historical_tables: 24h
```

### **Google Drive - Otimizado**
```yaml
sla_targets:
  file_processing:
    small_files_500kb: < 10s
    medium_files_10mb: < 60s
    large_files_100mb: < 300s
    
  api_rate_limits:
    requests_per_100s: 100
    concurrent_downloads: 10
```

## 🔧 Otimizações Implementadas

### **Database Performance (MySQL)**
```sql
-- Índices otimizados para ACODE
CREATE INDEX idx_farmacia_cnpj ON farmacias(cnpj, data_ultima_atualizacao);
CREATE INDEX idx_vendas_data ON vendas(data_venda, farmacia_id);
CREATE INDEX idx_produtos_ativo ON produtos(ativo, categoria_id);

-- Queries otimizadas com EXPLAIN
EXPLAIN SELECT f.nome, COUNT(v.id) as total_vendas
FROM farmacias f
LEFT JOIN vendas v ON f.id = v.farmacia_id
WHERE f.ativo = 1 
  AND v.data_venda >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY f.id, f.nome
ORDER BY total_vendas DESC
LIMIT 100;

-- Particionamento por data
ALTER TABLE vendas PARTITION BY RANGE (YEAR(data_venda)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

### **S3 Performance Optimization**
```bash
# Transfer acceleration habilitado
aws s3api put-bucket-accelerate-configuration \
  --bucket farmarcas-production-bronze \
  --accelerate-configuration Status=Enabled

# Multipart upload para arquivos grandes
aws s3 cp large-file.csv s3://farmarcas-production-bronze/radar/ \
  --cli-write-timeout 0 \
  --cli-read-timeout 0

# Intelligent tiering para otimização de custos
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket farmarcas-production-bronze \
  --id EntireBucket \
  --intelligent-tiering-configuration \
    Id=EntireBucket,Status=Enabled,IncludedObjectFilter={}
```

### **Airbyte Performance Tuning**
```yaml
# airbyte-worker-config.yaml
worker_config:
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"  
      cpu: "2000m"
      
  concurrency:
    max_workers: 10
    max_sync_workers: 5
    
  sync_job_config:
    timeout_hours: 8
    attempts: 3
    retry_delay_seconds: 30
```

## 📈 Monitoring e Alertas

### **Métricas Críticas**
```yaml
performance_metrics:
  acode_response_time:
    threshold: 500ms
    alert_channel: "#data-engineering"
    severity: high
    
  radar_sync_duration:
    threshold: 2h
    alert_channel: "#data-engineering"
    severity: medium
    
  s3_upload_failure_rate:
    threshold: 5%
    alert_channel: "#devops"
    severity: high
    
  airbyte_worker_memory:
    threshold: 80%
    alert_channel: "#infrastructure"
    severity: medium
```

### **Dashboard de Performance**
```sql
-- Query para dashboard de latência ACODE
SELECT 
    DATE(created_at) as data,
    AVG(response_time_ms) as avg_response,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms) as p95_response,
    COUNT(*) as total_requests
FROM performance_logs 
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(created_at)
ORDER BY data DESC;

-- Throughput por hora
SELECT 
    DATE_FORMAT(created_at, '%Y-%m-%d %H:00:00') as hora,
    COUNT(*) as requests_per_hour,
    COUNT(CASE WHEN response_time_ms > 500 THEN 1 END) as slow_requests
FROM performance_logs 
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
GROUP BY DATE_FORMAT(created_at, '%Y-%m-%d %H:00:00')
ORDER BY hora DESC;
```

## 🎯 Capacity Planning

### **Projeções de Crescimento**
```yaml
growth_projections:
  acode_data:
    current: 50GB/month
    projected_6m: 75GB/month
    projected_12m: 100GB/month
    
  radar_records:
    current: 10M records/month
    projected_6m: 15M records/month
    projected_12m: 25M records/month
    
  storage_costs:
    current: $500/month
    projected_6m: $750/month
    projected_12m: $1200/month
```

### **Scaling Strategy**
```yaml
horizontal_scaling:
  airbyte_workers:
    current: 5 workers
    scale_trigger: CPU > 80% for 10min
    max_workers: 15
    
  mysql_read_replicas:
    current: 2 replicas
    scale_trigger: connection_pool > 80%
    max_replicas: 5
    
vertical_scaling:
  database_instance:
    current: db.t3.large
    next_tier: db.t3.xlarge
    trigger: memory > 85% for 30min
```

## ⚡ Performance Tuning Recipes

### **Slow Query Optimization**
```sql
-- Before: Slow query (>5s)
SELECT * FROM vendas v
JOIN produtos p ON v.produto_id = p.id
WHERE v.data_venda BETWEEN '2024-01-01' AND '2024-01-31'
  AND p.categoria = 'Medicamentos';

-- After: Optimized query (<100ms)
SELECT v.id, v.data_venda, v.quantidade, v.valor,
       p.nome, p.categoria
FROM vendas v
FORCE INDEX (idx_vendas_data)
JOIN produtos p FORCE INDEX (idx_produtos_categoria)
  ON v.produto_id = p.id
WHERE v.data_venda >= '2024-01-01'
  AND v.data_venda < '2024-02-01'
  AND p.categoria = 'Medicamentos'
LIMIT 10000;
```

### **Airbyte Sync Optimization**
```yaml
# Connection configuration
connection_config:
  frequency: 
    type: "cron"
    cron: "0 */6 * * *"  # Every 6 hours instead of hourly
    
  advanced_settings:
    batch_size: 1000     # Increased from 500
    max_workers: 3       # Parallel processing
    timeout: "8h"        # Extended timeout
    
  normalization:
    enabled: false       # Disabled for performance
    
  transformation:
    dbt_enabled: true    # Custom transformations
```

### **S3 Performance Best Practices**
```bash
# Optimal prefix structure para performance
s3://farmarcas-production-bronze/
├── origin=airbyte/
│   ├── database=bronze_acode/
│   │   ├── year=2024/month=01/day=15/hour=14/
│   │   └── year=2024/month=01/day=15/hour=15/
│   └── database=bronze_radar/
│       ├── year=2024/month=01/day=15/
│       └── year=2024/month=01/day=16/

# Parallelização de uploads
parallel --jobs 4 aws s3 cp {} s3://bucket/prefix/ ::: file*.csv

# Compressão para reduzir custos e melhorar transfer
gzip -9 large-dataset.csv
aws s3 cp large-dataset.csv.gz s3://bucket/ --content-encoding gzip
```

## 📊 Performance Reports

### **Weekly Performance Summary**
```bash
#!/bin/bash
# weekly-performance-report.sh

echo "=== ACODE Performance ==="
mysql -e "
SELECT 
    'Avg Response Time' as metric,
    CONCAT(ROUND(AVG(response_time_ms), 2), 'ms') as value
FROM performance_logs 
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY);
"

echo "=== Airbyte Sync Performance ==="
curl -s -H "Authorization: Bearer $AIRBYTE_TOKEN" \
  "https://airbyte.farmarcas.com/api/v1/jobs" \
  | jq '.jobs[] | select(.status == "succeeded") | .job.updatedAt'

echo "=== S3 Storage Growth ==="
aws s3api list-objects-v2 --bucket farmarcas-production-bronze \
  --query 'Contents[].Size' --output text | \
  awk '{sum+=$1} END {print "Total: " sum/1024/1024/1024 " GB"}'
```

---

**Relacionado**: [Monitoramento](./monitoramento.md) | [Arquitetura](./arquitetura.md)
