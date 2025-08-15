# 🔄 Backup & Recovery - Cognitivo Data Platform

## 📋 Estratégias de Backup

### **ACODE - Sistema Crítico**
```bash
# Backup automático MySQL
mysqldump -h db-hsp-farmarcas.acode.com.br \
  -u backup_user -p acode_farmarcas \
  --single-transaction \
  --routines \
  --triggers \
  > backup_acode_$(date +%Y%m%d_%H%M%S).sql

# Backup S3 - Cross-region replication
aws s3 sync s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/ \
  s3://farmarcas-backup-us-west-2/bronze_acode/ \
  --storage-class GLACIER
```

### **Radar - Alta Disponibilidade**
```bash
# Backup incremental das tabelas críticas
./scripts/radar-backup-critical-tables.sh

# Backup metadados Airbyte
curl -X GET "https://airbyte.farmarcas.com/api/v1/connections/connection_mysql_s3_radar" \
  -H "Authorization: Bearer $AIRBYTE_TOKEN" > radar_connection_backup.json
```

### **Google Drive - Redundância**
```bash
# Backup arquivos coletados
./scripts/gdrive-backup-collected-files.sh

# Backup configurações
cp files_configs.yaml backups/files_configs_$(date +%Y%m%d).yaml
```

## 🚨 Procedimentos de Recovery

### **Cenário 1: ACODE Database Failure**
```bash
# 1. Ativar failover automático
./scripts/acode-activate-failover.sh

# 2. Validar backup connection
./scripts/acode-test-backup-connection.sh

# 3. Notificar stakeholders
./scripts/notify-acode-failover.sh "Database failure - failover activated"
```

### **Cenário 2: S3 Bucket Corruption**
```bash
# 1. Identificar escopo do problema
aws s3 ls s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/ \
  --recursive | grep $(date +%Y-%m-%d)

# 2. Restaurar do backup cross-region
aws s3 sync s3://farmarcas-backup-us-west-2/bronze_acode/ \
  s3://farmarcas-production-bronze/origin=airbyte/database=bronze_acode/ \
  --exclude "*" --include "*$(date +%Y-%m-%d)*"

# 3. Validar integridade
./scripts/validate-s3-data-integrity.sh
```

### **Cenário 3: Airbyte Platform Failure**
```bash
# 1. Verificar status dos workers
kubectl get pods -n plataforma | grep airbyte

# 2. Restart platform se necessário
kubectl rollout restart deployment/airbyte-server -n plataforma
kubectl rollout restart deployment/airbyte-worker -n plataforma

# 3. Re-trigger syncs perdidos
./scripts/retrigger-failed-syncs.sh
```

## 📊 RTO/RPO Targets

| Sistema | RTO (Recovery Time) | RPO (Recovery Point) | Estratégia |
|---------|-------------------|---------------------|------------|
| ACODE | < 2 horas | < 1 hora | Failover automático + Backup horário |
| Radar | < 4 horas | < 2 horas | Restart platform + Re-sync |
| Google Drive | < 8 horas | < 24 horas | Re-coleta manual + Backup files |

## 🔄 Testes de Recovery

### **Schedule de Testes**
```yaml
monthly_tests:
  - ACODE failover test (1st Monday)
  - S3 restoration test (2nd Monday)
  - Airbyte recovery test (3rd Monday)
  - Full disaster recovery drill (4th Monday)
```

### **Validação Pós-Recovery**
```bash
# Script de validação completa
./scripts/post-recovery-validation.sh

# Checklist manual:
# ✅ Conectividade com todas as fontes
# ✅ Dados sendo coletados corretamente
# ✅ Dashboards atualizando
# ✅ Alertas funcionando
# ✅ Stakeholders notificados
```

## 📞 Contatos de Emergência

### **Escalation Matrix**
- **L1**: Data Engineering Team (24/7)
- **L2**: DevOps Team + Infrastructure
- **L3**: CTO + VP Engineering

### **Canais de Comunicação**
- **Slack**: #data-engineering-alerts
- **PagerDuty**: CDP Critical Alerts
- **Email**: dataeng-oncall@farmarcas.com

---

**Relacionado**: [Troubleshooting](./troubleshooting.md) | [Monitoramento](./monitoramento.md)
