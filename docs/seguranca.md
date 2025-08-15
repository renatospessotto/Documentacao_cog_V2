# 🔐 Segurança - Cognitivo Data Platform

## 🛡️ Controles de Acesso

### **Autenticação Multi-Fator (MFA)**
```yaml
# Exigência para todas as contas críticas
required_mfa:
  - MySQL Production Databases
  - AWS Root/Admin Accounts  
  - Airbyte Admin Console
  - DataHub Admin Access
  - S3 Production Buckets
```

### **RBAC (Role-Based Access Control)**
```yaml
roles:
  data_engineer:
    permissions:
      - airbyte:read,write
      - s3:bronze,silver:read,write
      - mysql:read
      - datahub:metadata:write
      
  data_analyst:
    permissions:
      - s3:silver,gold:read
      - datahub:read
      - dashboards:read
      
  admin:
    permissions:
      - "*:*"
    mfa_required: true
    session_timeout: 30m
```

## 🔑 Gerenciamento de Credenciais

### **AWS Secrets Manager**
```bash
# Rotação automática de credenciais
aws secretsmanager rotate-secret \
  --secret-id "mysql-acode-credentials" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:account:function:SecretsManagerRDSPostgreSQLRotationSingleUser"

# Auditoria de acesso
aws secretsmanager describe-secret \
  --secret-id "mysql-acode-credentials" \
  --query 'LastAccessedDate'
```

### **Ambiente de Desenvolvimento**
```yaml
# .env.example
DATABASE_URL=mysql://user:pass@localhost:3306/dev_db
AIRBYTE_USERNAME=dev_user
AIRBYTE_PASSWORD=dev_pass
S3_BUCKET=farmarcas-dev-bronze
AWS_REGION=us-east-1

# NUNCA commitar credenciais reais
# Usar sempre variáveis de ambiente ou secrets manager
```

## 🚨 Monitoramento de Segurança

### **Alertas de Segurança**
```yaml
security_alerts:
  failed_login_attempts:
    threshold: 5
    window: 5m
    action: block_ip_30m
    
  unusual_data_access:
    threshold: 10x_normal_volume
    window: 1h
    action: alert_security_team
    
  privileged_access:
    events: ["admin_login", "schema_change", "user_creation"]
    action: log_and_notify
```

### **Auditoria de Acesso**
```sql
-- Query para auditoria MySQL (ACODE)
SELECT 
    USER() as current_user,
    CONNECTION_ID() as session_id,
    NOW() as access_time,
    DATABASE() as database_accessed
FROM DUAL;

-- Auditoria S3 via CloudTrail
SELECT 
    eventTime,
    eventName,
    userIdentity.userName,
    requestParameters.bucketName,
    sourceIPAddress
FROM cloudtrail_logs 
WHERE eventTime >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
  AND eventName LIKE 's3:%'
ORDER BY eventTime DESC;
```

## 🔒 Criptografia e Compliance

### **Dados em Trânsito**
```yaml
encryption_in_transit:
  mysql_connections:
    protocol: SSL/TLS 1.2+
    certificate_validation: required
    
  s3_transfers:
    protocol: HTTPS
    encryption: AES256
    
  api_communications:
    protocol: HTTPS
    authentication: OAuth2 + JWT
```

### **Dados em Repouso**
```yaml
encryption_at_rest:
  s3_buckets:
    encryption: SSE-S3 (AES256)
    key_management: AWS KMS
    
  mysql_databases:
    encryption: InnoDB tablespace encryption
    key_rotation: automatic_90_days
    
  backup_files:
    encryption: GPG
    key_storage: AWS KMS
```

### **Compliance LGPD**
```yaml
data_governance:
  pii_classification:
    - cpf: high_sensitivity
    - email: medium_sensitivity
    - phone: medium_sensitivity
    
  retention_policies:
    raw_data: 7_years
    processed_data: 5_years
    logs: 1_year
    
  anonymization:
    enabled: true
    fields: [cpf, rg, email]
    method: sha256_hash
```

## 🔥 Procedimentos de Incident Response

### **Breach Detection**
```bash
# Script de detecção de anomalias
./scripts/security-anomaly-detection.sh

# Verificações automáticas:
# - Acessos fora do horário comercial
# - Downloads massivos de dados
# - Tentativas de acesso negadas
# - Modificações em tabelas críticas
```

### **Containment & Remediation**
```bash
# 1. Isolamento imediato
./scripts/security-containment.sh

# 2. Preservação de evidências
./scripts/collect-security-evidence.sh

# 3. Notificação de stakeholders
./scripts/notify-security-incident.sh "SECURITY BREACH DETECTED"

# 4. Análise forense
./scripts/security-forensic-analysis.sh
```

## 👥 Treinamento e Awareness

### **Security Champions Program**
```yaml
quarterly_training:
  - OWASP Top 10 for Data Platforms
  - Secure Coding Practices
  - LGPD Compliance Updates
  - Incident Response Simulation

security_champions:
  - 1 por squad (Data Engineering, Analytics, DevOps)
  - Monthly security reviews
  - Threat modeling sessions
```

### **Phishing & Social Engineering**
```yaml
awareness_program:
  monthly_simulations:
    - Fake credential requests
    - Suspicious attachments
    - Social engineering calls
    
  metrics:
    click_rate: <5%
    report_rate: >90%
    time_to_report: <2_minutes
```

## 📋 Security Checklist

### **Daily Checks**
- [ ] Review failed login attempts
- [ ] Monitor unusual data access patterns  
- [ ] Verify backup encryption status
- [ ] Check security alert queue

### **Weekly Checks**
- [ ] Review user access permissions
- [ ] Audit privileged account usage
- [ ] Validate SSL certificate status
- [ ] Update security patches

### **Monthly Checks**
- [ ] Access review and cleanup
- [ ] Penetration testing results
- [ ] Security training completion
- [ ] Compliance audit preparation

---

**Relacionado**: [Boas Práticas](./boas-praticas.md) | [Troubleshooting](./troubleshooting.md)
