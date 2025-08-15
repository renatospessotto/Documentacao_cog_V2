# 💡 Boas Práticas - Cognitivo Data Platform

## 🔐 Segurança e Credenciais

### **Gerenciamento de Secrets**
```bash
# Use AWS Secrets Manager ao invés de variáveis de ambiente
aws secretsmanager create-secret \
  --name "cdp/acode/mysql" \
  --description "ACODE MySQL credentials" \
  --secret-string '{"username":"user","password":"pass"}'

# Rotação automática a cada 90 dias
aws secretsmanager update-secret \
  --secret-id "cdp/acode/mysql" \
  --rotation-rules '{"AutomaticallyAfterDays":90}'
```

### **Permissões IAM**
- Use roles específicos para cada sistema
- Princípio do menor privilégio
- Auditoria regular de permissões

## 📊 Monitoramento

### **Métricas Essenciais**
- **Data Freshness**: < 4 horas para dashboards
- **Success Rate**: > 99% para pipelines críticos
- **Duration**: Monitorar tendências de crescimento
- **Error Rate**: < 1% para todos os sistemas

### **Alertas Recomendados**
```yaml
# config/alerts.yaml
alerts:
  critical:
    - acode_sync_failure
    - s3_access_denied
    - mysql_connection_timeout
  
  warning:
    - radar_sync_slow
    - gdrive_rate_limit
    - disk_space_low
```

## 🔄 Operações

### **Backup e Recovery**
- Backup diário de configurações
- Teste de recovery mensal
- Documentar procedimentos de emergência

### **Manutenção Preventiva**
- Limpeza de logs antiga semanalmente
- Atualização de dependências mensalmente
- Review de performance trimestralmente

---

**Próximo**: [Monitoramento](./monitoramento.md)
