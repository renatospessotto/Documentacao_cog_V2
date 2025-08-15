# ⚙️ Pré-requisitos - Google Drive Collector

## 📋 Visão Geral

Este documento detalha todos os pré-requisitos necessários para configurar e executar o Google Drive Collector, incluindo autenticação Google Drive API, permissões AWS, dependências Python e configurações de ambiente.

---

## 🔐 1. Autenticação Google Drive API

### **1.1 Configuração Service Account**

#### **Passo 1: Criar Projeto no Google Cloud Console**
```bash
# 1. Acesse https://console.cloud.google.com/
# 2. Crie novo projeto ou selecione existente
# 3. Nome sugerido: "farmarcas-data-platform"
# 4. Anote o PROJECT_ID para referência
```

#### **Passo 2: Habilitar Google Drive API**
```bash
# Via Cloud Console:
# 1. Vá para "APIs & Services" > "Library"
# 2. Busque por "Google Drive API"
# 3. Clique em "Enable"

# Via gcloud CLI (opcional):
gcloud services enable drive.googleapis.com --project=farmarcas-data-platform
```

#### **Passo 3: Criar Service Account**
```bash
# Via Cloud Console:
# 1. Vá para "IAM & Admin" > "Service Accounts"
# 2. Clique "Create Service Account"
# 3. Nome: "gdrive-collector"
# 4. Descrição: "Service account para coleta de arquivos do Google Drive"
# 5. Não atribuir roles no projeto (desnecessário)

# Via gcloud CLI:
gcloud iam service-accounts create gdrive-collector \
    --description="Service account para coleta Google Drive" \
    --display-name="Google Drive Collector" \
    --project=farmarcas-data-platform
```

#### **Passo 4: Gerar Chave JSON**
```bash
# Via Cloud Console:
# 1. Clique no Service Account criado
# 2. Vá para aba "Keys"
# 3. Clique "Add Key" > "Create new key"
# 4. Selecione "JSON"
# 5. Faça download do arquivo

# Via gcloud CLI:
gcloud iam service-accounts keys create google_service_account.json \
    --iam-account=gdrive-collector@farmarcas-data-platform.iam.gserviceaccount.com \
    --project=farmarcas-data-platform
```

### **1.2 Estrutura do Arquivo Credentials**

O arquivo JSON deve ter a seguinte estrutura:

```json
{
  "type": "service_account",
  "project_id": "farmarcas-data-platform",
  "private_key_id": "abc123def456...",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQC...\n-----END PRIVATE KEY-----\n",
  "client_email": "gdrive-collector@farmarcas-data-platform.iam.gserviceaccount.com",
  "client_id": "123456789012345678901",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/gdrive-collector%40farmarcas-data-platform.iam.gserviceaccount.com"
}
```

### **1.3 Configuração de Permissões no Google Drive**

#### **Compartilhamento de Pastas**
Para cada pasta que será monitorada pelo collector:

```bash
# 1. Abra o Google Drive no navegador
# 2. Localize a pasta que contém os arquivos
# 3. Clique com botão direito > "Compartilhar"
# 4. Adicione o email do Service Account:
#    gdrive-collector@farmarcas-data-platform.iam.gserviceaccount.com
# 5. Defina permissão como "Viewer" (somente leitura)
# 6. Anote o ID da pasta (visível na URL)
```

#### **Obtenção de IDs de Pastas e Arquivos**
```bash
# Para pasta: https://drive.google.com/drive/folders/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms
# ID da pasta: 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms

# Para arquivo: https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms/edit
# ID do arquivo: 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms
```

### **1.4 Teste de Conectividade**

Script para validar a autenticação:

```python
#!/usr/bin/env python3
"""
Script de teste para validar autenticação Google Drive
"""

from google.oauth2 import service_account
from googleapiclient.discovery import build
import json

def test_google_drive_auth(credentials_path):
    """
    Testa autenticação com Google Drive API
    """
    
    try:
        # Carregar credenciais
        credentials = service_account.Credentials.from_service_account_file(
            credentials_path,
            scopes=['https://www.googleapis.com/auth/drive.readonly']
        )
        
        # Criar cliente da API
        service = build('drive', 'v3', credentials=credentials)
        
        # Testar acesso básico
        about = service.about().get(fields="user").execute()
        user_email = about.get('user', {}).get('emailAddress')
        
        print(f"✅ Autenticação bem-sucedida!")
        print(f"📧 Service Account: {user_email}")
        
        # Testar listagem de arquivos (primeiros 10)
        results = service.files().list(pageSize=10).execute()
        files = results.get('files', [])
        
        print(f"📁 Arquivos visíveis: {len(files)}")
        
        if files:
            print("📋 Primeiros arquivos encontrados:")
            for file in files[:3]:
                print(f"   - {file['name']} (ID: {file['id']})")
        
        return True
        
    except Exception as e:
        print(f"❌ Erro na autenticação: {str(e)}")
        return False

if __name__ == "__main__":
    credentials_path = "google_service_account.json"
    test_google_drive_auth(credentials_path)
```

---

## ☁️ 2. Configuração AWS

### **2.1 Credenciais AWS**

#### **IAM User para Desenvolvimento**
```bash
# 1. Acesse AWS Console > IAM > Users
# 2. Create User: "gdrive-collector-dev"
# 3. Attach policies directly
# 4. Create Access Key
# 5. Anote Access Key ID e Secret Access Key
```

#### **IAM Role para Produção (EKS/Fargate)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks-fargate-pods.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### **2.2 Políticas IAM Necessárias**

#### **Política S3 (Mínima)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3BronzeLayerAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl", 
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-production-bronze/origin=eks/database=bronze_gdrive/*"
      ]
    },
    {
      "Sid": "S3BucketLevelAccess",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::farmarcas-production-bronze"
      ],
      "Condition": {
        "StringLike": {
          "s3:prefix": ["origin=eks/database=bronze_gdrive/*"]
        }
      }
    }
  ]
}
```

#### **Política CloudWatch Logs**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream", 
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": [
        "arn:aws:logs:us-east-2:*:log-group:/aws/fargate/gdrive-collector*"
      ]
    }
  ]
}
```

#### **Política Glue (Opcional para Catalogação)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GlueCatalogAccess",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:CreateTable",
        "glue:UpdateTable",
        "glue:BatchCreatePartition"
      ],
      "Resource": [
        "arn:aws:glue:us-east-2:*:catalog",
        "arn:aws:glue:us-east-2:*:database/bronze_gdrive",
        "arn:aws:glue:us-east-2:*:table/bronze_gdrive/*"
      ]
    }
  ]
}
```

### **2.3 Configuração de Credenciais**

#### **Via Variáveis de Ambiente**
```bash
# Desenvolvimento Local
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-2"
export AWS_SESSION_TOKEN=""  # Se usando STS

# Verificar configuração
aws sts get-caller-identity
aws s3 ls farmarcas-production-bronze --prefix "origin=eks/database=bronze_gdrive/"
```

#### **Via AWS Credentials File**
```ini
# ~/.aws/credentials
[gdrive-collector]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
region = us-east-2

# ~/.aws/config  
[profile gdrive-collector]
region = us-east-2
output = json
```

#### **Via IAM Role (Produção)**
```yaml
# EKS Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gdrive-collector
  namespace: data-platform
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/gdrive-collector-role
```

### **2.4 Teste de Conectividade AWS**

```python
#!/usr/bin/env python3
"""
Script de teste para validar acesso AWS
"""

import boto3
import json
from datetime import datetime

def test_aws_access():
    """
    Testa acesso aos serviços AWS necessários
    """
    
    try:
        # Teste STS (identidade)
        sts = boto3.client('sts')
        identity = sts.get_caller_identity()
        
        print(f"✅ AWS Autenticação bem-sucedida!")
        print(f"👤 Account: {identity['Account']}")
        print(f"🆔 User/Role: {identity['Arn']}")
        
        # Teste S3
        s3 = boto3.client('s3')
        bucket_name = "farmarcas-production-bronze"
        prefix = "origin=eks/database=bronze_gdrive/"
        
        # Testar listagem
        response = s3.list_objects_v2(Bucket=bucket_name, Prefix=prefix, MaxKeys=5)
        object_count = response.get('KeyCount', 0)
        
        print(f"📦 S3 Bucket: {bucket_name}")
        print(f"📁 Objetos no prefixo: {object_count}")
        
        # Testar upload
        test_key = f"{prefix}test/{datetime.now().isoformat()}.txt"
        s3.put_object(
            Bucket=bucket_name,
            Key=test_key,
            Body=b"Test upload from gdrive-collector",
            Metadata={'source': 'test'}
        )
        
        print(f"✅ Upload teste bem-sucedido: {test_key}")
        
        # Limpar arquivo teste
        s3.delete_object(Bucket=bucket_name, Key=test_key)
        print(f"🗑️ Arquivo teste removido")
        
        # Teste CloudWatch Logs (opcional)
        try:
            logs = boto3.client('logs')
            log_groups = logs.describe_log_groups(logGroupNamePrefix='/aws/fargate/gdrive')
            print(f"📊 CloudWatch Log Groups: {len(log_groups['logGroups'])}")
        except Exception as e:
            print(f"⚠️ CloudWatch Logs: {str(e)}")
        
        return True
        
    except Exception as e:
        print(f"❌ Erro AWS: {str(e)}")
        return False

if __name__ == "__main__":
    test_aws_access()
```

---

## 🐍 3. Dependências Python

### **3.1 requirements.txt**

```txt
# Google APIs
google-auth==2.17.3
google-auth-oauthlib==1.0.0
google-auth-httplib2==0.1.0
google-api-python-client==2.88.0

# AWS Services
boto3==1.26.137
awswrangler==3.2.0

# Data Processing
pandas==2.0.2
openpyxl==3.1.2           # Excel reading
xlrd==2.0.1               # Legacy Excel support

# Utilities
PyYAML==6.0
pytz==2023.3
python-dateutil==2.8.2

# Logging and Monitoring
structlog==23.1.0
psutil==5.9.5             # System monitoring

# Development/Testing (optional)
pytest==7.3.1
pytest-cov==4.1.0
black==23.3.0
flake8==6.0.0
```

### **3.2 Instalação do Ambiente**

#### **Via pip (Desenvolvimento)**
```bash
# Criar ambiente virtual
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# ou
venv\Scripts\activate     # Windows

# Instalar dependências
pip install --upgrade pip
pip install -r requirements.txt

# Verificar instalação
pip list | grep -E "(google|boto3|pandas|awswrangler)"
```

#### **Via conda (Alternativo)**
```bash
# Criar ambiente conda
conda create -n gdrive-collector python=3.9
conda activate gdrive-collector

# Instalar principais dependências via conda
conda install pandas openpyxl xlrd pyyaml boto3

# Instalar Google APIs via pip
pip install google-auth google-api-python-client awswrangler
```

#### **Dockerfile para Produção**
```dockerfile
FROM python:3.9-slim

# Instalar dependências do sistema
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copiar e instalar requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código da aplicação
COPY collector_gdrive/ ./collector_gdrive/
COPY files_configs.yaml .

# Configurar usuário não-root
RUN useradd -m -u 1000 collector
USER collector

# Definir ponto de entrada
ENTRYPOINT ["python", "collector_gdrive/main.py"]
```

### **3.3 Validação das Dependências**

```python
#!/usr/bin/env python3
"""
Script para validar todas as dependências necessárias
"""

import sys
import importlib

def check_dependencies():
    """
    Verifica se todas as dependências estão instaladas
    """
    
    required_packages = {
        # Google APIs
        'google.auth': 'google-auth',
        'google.oauth2': 'google-auth', 
        'googleapiclient.discovery': 'google-api-python-client',
        
        # AWS
        'boto3': 'boto3',
        'awswrangler': 'awswrangler',
        
        # Data Processing
        'pandas': 'pandas',
        'openpyxl': 'openpyxl',
        'xlrd': 'xlrd',
        
        # Utilities
        'yaml': 'PyYAML',
        'pytz': 'pytz',
        'dateutil': 'python-dateutil',
    }
    
    print("🔍 Verificando dependências...")
    
    missing_packages = []
    
    for module_name, package_name in required_packages.items():
        try:
            importlib.import_module(module_name)
            print(f"✅ {package_name}")
        except ImportError:
            print(f"❌ {package_name}")
            missing_packages.append(package_name)
    
    if missing_packages:
        print(f"\n⚠️ Pacotes ausentes: {', '.join(missing_packages)}")
        print("💡 Execute: pip install " + " ".join(missing_packages))
        return False
    
    print("\n✅ Todas as dependências estão instaladas!")
    
    # Verificar versões críticas
    print("\n📋 Versões das dependências principais:")
    
    try:
        import pandas as pd
        print(f"   pandas: {pd.__version__}")
    except:
        pass
        
    try:
        import boto3
        print(f"   boto3: {boto3.__version__}")
    except:
        pass
        
    try:
        import awswrangler as wr
        print(f"   awswrangler: {wr.__version__}")
    except:
        pass
    
    return True

if __name__ == "__main__":
    success = check_dependencies()
    sys.exit(0 if success else 1)
```

---

## 🔧 4. Variáveis de Ambiente

### **4.1 Variáveis Obrigatórias**

```bash
# Google Drive Authentication
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/google_service_account.json"

# AWS Configuration
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-2"

# Application Configuration
export S3_BUCKET_NAME="farmarcas-production-bronze"
export LOG_LEVEL="INFO"
export ENVIRONMENT="production"
```

### **4.2 Variáveis Opcionais**

```bash
# Performance Tuning
export MAX_CONCURRENT_DOWNLOADS="5"
export DOWNLOAD_CHUNK_SIZE_MB="10"
export API_RATE_LIMIT="10"

# Monitoring and Alerts
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
export CLOUDWATCH_LOG_GROUP="/aws/fargate/gdrive-collector"

# Development/Debug
export DEBUG_MODE="false"
export CACHE_ENABLED="true"
export CACHE_DIR="/tmp/gdrive_cache"

# Timezone
export TZ="America/Sao_Paulo"
```

### **4.3 Arquivo .env para Desenvolvimento**

```bash
# .env file (para desenvolvimento local)
# NÃO COMMITAR ESTE ARQUIVO

# Google Drive
GOOGLE_APPLICATION_CREDENTIALS=./credentials/google_service_account.json

# AWS
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-2

# Application
S3_BUCKET_NAME=farmarcas-development-bronze
LOG_LEVEL=DEBUG
ENVIRONMENT=development

# Optional
DEBUG_MODE=true
CACHE_ENABLED=false
MAX_CONCURRENT_DOWNLOADS=3
```

### **4.4 Configuração Docker/Kubernetes**

```yaml
# docker-compose.yml (desenvolvimento)
version: '3.8'
services:
  gdrive-collector:
    build: .
    environment:
      - GOOGLE_APPLICATION_CREDENTIALS=/app/credentials/google_service_account.json
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_DEFAULT_REGION=us-east-2
      - S3_BUCKET_NAME=farmarcas-development-bronze
      - LOG_LEVEL=INFO
    volumes:
      - ./credentials:/app/credentials:ro
    networks:
      - data-platform

# kubernetes-deployment.yaml (produção)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gdrive-collector
spec:
  template:
    spec:
      serviceAccountName: gdrive-collector  # Com IAM Role
      containers:
      - name: collector
        image: farmarcas/gdrive-collector:latest
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /app/credentials/google_service_account.json
        - name: S3_BUCKET_NAME
          value: farmarcas-production-bronze
        - name: LOG_LEVEL
          value: INFO
        - name: ENVIRONMENT
          value: production
        volumeMounts:
        - name: google-credentials
          mountPath: /app/credentials
          readOnly: true
      volumes:
      - name: google-credentials
        secret:
          secretName: google-service-account
```

---

## 🌐 5. Configuração de Rede

### **5.1 Firewall e Conectividade**

#### **URLs que Precisam de Acesso**
```bash
# Google APIs
https://www.googleapis.com          # Google Drive API
https://accounts.google.com         # OAuth2 authentication
https://oauth2.googleapis.com       # Token refresh

# AWS Services
https://s3.us-east-2.amazonaws.com  # S3 operations
https://logs.us-east-2.amazonaws.com # CloudWatch Logs
https://sts.amazonaws.com           # Identity verification

# Portas necessárias
443/tcp  # HTTPS para todas as APIs
53/tcp   # DNS resolution
53/udp   # DNS resolution
```

#### **Configuração Proxy (se necessário)**
```bash
# Via variáveis de ambiente
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,.internal.domain"

# Via pip (para instalação de dependências)
pip install --proxy http://proxy.company.com:8080 -r requirements.txt
```

### **5.2 Segurança**

#### **TLS/SSL Verification**
```python
import ssl
import urllib3

# Para ambientes corporativos com certificados próprios
# APENAS SE ABSOLUTAMENTE NECESSÁRIO
def configure_ssl_context():
    """
    Configuração SSL para ambientes corporativos
    """
    
    # Opção 1: Usar certificados customizados
    ssl_context = ssl.create_default_context()
    ssl_context.load_verify_locations('/path/to/corporate-ca.pem')
    
    # Opção 2: Desabilitar verificação (NÃO RECOMENDADO)
    # ssl_context.check_hostname = False
    # ssl_context.verify_mode = ssl.CERT_NONE
    # urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
    
    return ssl_context
```

---

## ✅ 6. Checklist de Validação

### **6.1 Google Drive API**
```bash
✅ Projeto criado no Google Cloud Console
✅ Google Drive API habilitada
✅ Service Account criado
✅ Arquivo JSON de credenciais baixado
✅ Pastas compartilhadas com Service Account
✅ IDs de pastas/arquivos coletados
✅ Teste de conectividade executado com sucesso
```

### **6.2 AWS**
```bash
✅ IAM User/Role configurado
✅ Políticas S3 anexadas
✅ Políticas CloudWatch Logs anexadas
✅ Credenciais AWS configuradas
✅ Acesso ao bucket S3 validado
✅ Teste de upload/download S3 executado
```

### **6.3 Python Environment**
```bash
✅ Python 3.9+ instalado
✅ Ambiente virtual criado
✅ requirements.txt instalado
✅ Todas as dependências validadas
✅ Teste de importação executado
```

### **6.4 Configurações**
```bash
✅ Variáveis de ambiente definidas
✅ files_configs.yaml configurado
✅ Credenciais Google Drive no local correto
✅ Logs de teste executados com sucesso
✅ Conectividade de rede validada
```

### **6.5 Script de Validação Completa**

```python
#!/usr/bin/env python3
"""
Script completo de validação de pré-requisitos
"""

import os
import sys
import json
from pathlib import Path

def validate_environment():
    """
    Executa validação completa do ambiente
    """
    
    print("🔍 Iniciando validação completa do ambiente...")
    
    checks = []
    
    # 1. Variáveis de ambiente
    required_env_vars = [
        'GOOGLE_APPLICATION_CREDENTIALS',
        'AWS_ACCESS_KEY_ID', 
        'AWS_SECRET_ACCESS_KEY',
        'AWS_DEFAULT_REGION'
    ]
    
    for var in required_env_vars:
        if os.getenv(var):
            print(f"✅ {var}")
            checks.append(True)
        else:
            print(f"❌ {var} não definida")
            checks.append(False)
    
    # 2. Arquivo de credenciais Google
    google_creds = os.getenv('GOOGLE_APPLICATION_CREDENTIALS')
    if google_creds and Path(google_creds).exists():
        try:
            with open(google_creds) as f:
                creds_data = json.load(f)
                if 'client_email' in creds_data:
                    print(f"✅ Credenciais Google: {creds_data['client_email']}")
                    checks.append(True)
                else:
                    print("❌ Arquivo de credenciais Google inválido")
                    checks.append(False)
        except Exception as e:
            print(f"❌ Erro ao ler credenciais Google: {e}")
            checks.append(False)
    else:
        print("❌ Arquivo de credenciais Google não encontrado")
        checks.append(False)
    
    # 3. Dependências Python
    try:
        import google.auth
        import boto3
        import pandas as pd
        import awswrangler as wr
        print("✅ Dependências Python instaladas")
        checks.append(True)
    except ImportError as e:
        print(f"❌ Dependências Python ausentes: {e}")
        checks.append(False)
    
    # 4. Conectividade (testes básicos)
    try:
        # Teste Google Drive
        from google.oauth2 import service_account
        from googleapiclient.discovery import build
        
        credentials = service_account.Credentials.from_service_account_file(
            google_creds,
            scopes=['https://www.googleapis.com/auth/drive.readonly']
        )
        service = build('drive', 'v3', credentials=credentials)
        service.about().get(fields="user").execute()
        
        print("✅ Conectividade Google Drive")
        checks.append(True)
    except Exception as e:
        print(f"❌ Conectividade Google Drive: {e}")
        checks.append(False)
    
    try:
        # Teste AWS
        import boto3
        sts = boto3.client('sts')
        sts.get_caller_identity()
        
        print("✅ Conectividade AWS")
        checks.append(True)
    except Exception as e:
        print(f"❌ Conectividade AWS: {e}")
        checks.append(False)
    
    # Resultado final
    success_rate = sum(checks) / len(checks) * 100
    
    print(f"\n📊 Resultado: {success_rate:.1f}% dos checks aprovados")
    
    if success_rate == 100:
        print("🎉 Ambiente completamente configurado!")
        return True
    elif success_rate >= 80:
        print("⚠️ Ambiente quase pronto - verifique itens pendentes")
        return True
    else:
        print("❌ Ambiente precisa de configuração adicional")
        return False

if __name__ == "__main__":
    success = validate_environment()
    sys.exit(0 if success else 1)
```

---

**Última Atualização**: 07/08/2025 - Pré-requisitos validados para ambiente de produção
