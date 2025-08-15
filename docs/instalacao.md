# 🛠️ Guia de Instalação - Cognitivo Data Platform

## 📋 Pré-requisitos

### Requisitos de Sistema
- **Sistema Operacional**: Linux (Ubuntu 20.04+) ou macOS
- **Python**: 3.8+ com pip e virtualenv
- **Docker**: 20.10+ com Docker Compose
- **Kubernetes**: 1.20+ com kubectl configurado
- **AWS CLI**: v2.0+ com credenciais válidas

### Credenciais Necessárias

#### MySQL Connections
```bash
# ACODE
export ACODE_HOST="db-hsp-farmarcas.acode.com.br"
export ACODE_USER="userfarmarcasac02"  
export ACODE_PASS="<senha_acode>"

# Radar
export RADAR_HOST="db-mysql-radar-production"
export RADAR_USER="bi-cognitivo-read"
export RADAR_PASS="<senha_radar>"
```

#### AWS Configuration
```bash
export AWS_PROFILE="farmarcas-production"
export AWS_REGION="us-east-2"
export AWS_S3_BUCKET_BRONZE="farmarcas-production-bronze"
export AWS_S3_BUCKET_SILVER="farmarcas-production-silver"
export AWS_S3_BUCKET_GOLD="farmarcas-production-gold"
```

#### Google Drive API
```bash
export GOOGLE_CREDENTIALS_PATH="/path/to/service-account.json"
export GOOGLE_DRIVE_FOLDER_ID="<folder_id>"
```

## 🚀 Instalação Completa

### 1. Configuração do Ambiente

```bash
# Clone o repositório
git clone <repository-url>
cd cdp-platform

# Criar ambiente Python
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Configurar variáveis de ambiente
cp .env.example .env
vim .env  # Configure todas as variáveis necessárias
source .env
```

### 2. Verificação de Conectividade

```bash
# Testar conexões MySQL
./scripts/test-mysql-connections.sh

# Verificar acesso AWS S3
aws s3 ls s3://farmarcas-production-bronze/ --profile farmarcas-production

# Testar Google Drive API
python scripts/test-gdrive-connection.py
```

### 3. Setup dos Coletores

#### ACODE System
```bash
cd collectors/acode/
./setup.sh
./test-connection.sh
```

#### Radar System  
```bash
cd collectors/radar/
./setup.sh
./validate-tables.sh
```

#### Google Drive Collector
```bash
cd collectors/gdrive/
./setup.sh
./test-files-config.sh
```

### 4. Deploy em Kubernetes

```bash
# Aplicar namespaces
kubectl apply -f k8s/namespaces/

# Deploy secrets
kubectl apply -f k8s/secrets/

# Deploy applications
kubectl apply -f k8s/deployments/

# Verificar status
kubectl get pods -n cdp-platform
```

## ✅ Verificação da Instalação

### Health Check Completo
```bash
# Executar verificação geral
./scripts/health-check-all.sh

# Resultado esperado:
# ✅ ACODE Connection: OK
# ✅ Radar Connection: OK  
# ✅ Google Drive API: OK
# ✅ AWS S3 Access: OK
# ✅ Airbyte Platform: OK
# ✅ Airflow DAGs: OK
```

### Validação por Sistema
```bash
# ACODE - Testar sync manual
cd docs/acode/ && ./quick-test.sh

# Radar - Verificar tabelas
cd docs/radar/ && ./validate-sync.sh

# Google Drive - Coletar arquivo teste
cd docs/google-drive/ && ./test-collection.sh
```

## 🔧 Configuração de Monitoramento

### Logs Centralizados
```bash
# Configurar agregação de logs
kubectl apply -f monitoring/logging/

# Verificar coleta
kubectl logs -n monitoring -l app=log-aggregator
```

### Métricas e Alertas
```bash
# Deploy Prometheus/Grafana
kubectl apply -f monitoring/metrics/

# Configurar alertas Slack
kubectl apply -f monitoring/alerts/
```

## 📊 Validação Final

### Execução de Pipeline Completo
```bash
# Trigger manual de todos os sistemas
./scripts/trigger-full-pipeline.sh

# Monitorar execução
./scripts/monitor-pipeline.sh

# Validar dados em S3
./scripts/validate-s3-data.sh
```

### Métricas Esperadas
- **ACODE**: ~50M registros processados em 45-120min
- **Radar**: ~1M registros sincronizados em 45-60min  
- **Google Drive**: ~200 arquivos coletados em 15-30min

---

## 🚨 Troubleshooting de Instalação

### Problemas Comuns

#### Erro de Credenciais MySQL
```bash
# Verificar conectividade
telnet <mysql_host> 3306

# Testar credenciais
mysql -h <host> -u <user> -p<password> -e "SELECT 1"
```

#### Falha de Acesso S3
```bash
# Verificar AWS CLI
aws sts get-caller-identity --profile farmarcas-production

# Testar permissões
aws s3 ls s3://farmarcas-production-bronze/ --profile farmarcas-production
```

#### Google Drive API Error
```bash
# Validar service account
python -c "
import json
with open('service-account.json') as f:
    data = json.load(f)
    print(f'Project: {data[\"project_id\"]}')
    print(f'Client Email: {data[\"client_email\"]}')
"

# Testar permissões
python scripts/test-gdrive-permissions.py
```

---

**Próximo**: [Configuração Detalhada](./configuracao.md)
