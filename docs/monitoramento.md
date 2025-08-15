# 📊 Monitoramento e Observabilidade - CDP

## 📈 Dashboards Operacionais

### **Dashboard Principal - CDP Overview**
- **URL**: `https://grafana.farmarcas.com/dashboards/cdp-overview`
- **Métricas**: Success rate, duration, data volume
- **Alertas**: Integrados com Slack e PagerDuty

### **Dashboards por Sistema**
- **ACODE**: `https://grafana.farmarcas.com/dashboards/acode`
- **Radar**: `https://grafana.farmarcas.com/dashboards/radar`
- **Google Drive**: `https://grafana.farmarcas.com/dashboards/gdrive`

## 🚨 Alertas Configurados

### **Criticalidade P1**
```yaml
acode_sync_failure:
  condition: "ACODE sync fails"
  channels: ["pagerduty", "slack"]
  escalation: "CTO after 30min"

s3_access_denied:
  condition: "S3 upload fails with 403"
  channels: ["pagerduty", "slack"]
  escalation: "DevOps team"
```

### **Criticalidade P2**
```yaml
radar_sync_slow:
  condition: "Radar sync > 2 hours"
  channels: ["slack"]
  escalation: "Data Engineering team"

data_quality_issues:
  condition: "Soda Core quality checks fail"
  channels: ["slack", "email"]
  escalation: "Data team"
```

## 📊 Métricas SLA

| Sistema | Disponibilidade | RTO | RPO | Data Freshness |
|---------|-----------------|-----|-----|----------------|
| ACODE | 99.9% | 2h | 1h | 4h |
| Radar | 99.5% | 4h | 2h | 6h |
| Google Drive | 99.0% | 8h | 24h | 24h |

## 🔍 Logs Centralizados

### **Acesso aos Logs**
```bash
# Logs Airflow
kubectl logs -n plataforma -l app=airflow-scheduler --tail=100

# Logs Airbyte
kubectl logs -n plataforma -l app=airbyte-worker --tail=100

# Logs Coletores
kubectl logs -n collectors -l app=gdrive-collector --tail=100
```

### **Query Examples - CloudWatch**
```sql
-- Erros ACODE últimas 24h
fields @timestamp, @message
| filter @message like /ERROR/
| filter @message like /acode/
| sort @timestamp desc
| limit 100

-- Performance Radar
fields @timestamp, duration
| filter @message like /radar_sync_complete/
| stats avg(duration) by bin(5m)
```

---

**Relacionado**: [Troubleshooting](./troubleshooting.md) | [Boas Práticas](./boas-praticas.md)
