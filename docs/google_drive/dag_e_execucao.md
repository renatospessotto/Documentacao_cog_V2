# 🚀 DAG e Execução - Google Drive Collector

## 📋 Visão Geral

O Google Drive Collector é orquestrado via Apache Airflow, executando diariamente em ambiente Kubernetes (EKS) com containers Docker. Este documento detalha a configuração completa da DAG, deploy, monitoramento e execução.

### **Arquitetura de Execução**
```mermaid
graph TB
    subgraph "🚀 Orchestration Layer"
        AIRFLOW[Apache Airflow<br/>Scheduler & WebUI]
        DAG[DAG: gdrive_collector<br/>Schedule: Daily 07:30]
    end
    
    subgraph "☸️ Kubernetes Cluster (EKS)"
        POD[Fargate Pod<br/>gdrive-collector]
        CONTAINER[Docker Container<br/>Python 3.9 + Dependencies]
    end
    
    subgraph "🔐 Credentials & Config"
        K8S_SECRET[Kubernetes Secret<br/>google-service-account]
        IAM_ROLE[IAM Role<br/>gdrive-collector-role]
        CONFIG_MAP[ConfigMap<br/>files-configs]
    end
    
    subgraph "📊 Monitoring"
        CLOUDWATCH[CloudWatch Logs<br/>/aws/fargate/gdrive-collector]
        METRICS[Custom Metrics<br/>FilesProcessed, ProcessingTime]
        ALERTS[Slack Notifications<br/>#data-platform]
    end
    
    subgraph "🗄️ Data Flow"
        GDRIVE[Google Drive<br/>Source Files]
        S3[S3 Bronze Layer<br/>Processed Data]
        GLUE[AWS Glue Catalog<br/>Metadata Registry]
    end
    
    AIRFLOW --> DAG
    DAG --> POD
    POD --> CONTAINER
    
    K8S_SECRET --> CONTAINER
    IAM_ROLE --> CONTAINER
    CONFIG_MAP --> CONTAINER
    
    CONTAINER --> CLOUDWATCH
    CONTAINER --> METRICS
    METRICS --> ALERTS
    
    CONTAINER --> GDRIVE
    GDRIVE --> CONTAINER
    CONTAINER --> S3
    S3 --> GLUE
    
    style AIRFLOW fill:#e1f5fe
    style CONTAINER fill:#e8f5e8
    style CLOUDWATCH fill:#fff3e0
    style S3 fill:#f3e5f5
```

---

## 🛠️ 1. Configuração da DAG Airflow

### **1.1 DAG Principal - gdrive_collector.py**

```python
"""
DAG para coleta automática de arquivos do Google Drive
Data Platform Farmarcas

Executa diariamente às 07:30 (UTC-3)
Processamento de arquivos críticos de negócio
"""

from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
from airflow.operators.python import PythonOperator
from airflow.models import Variable
from datetime import datetime, timedelta
import logging

# Configurações da DAG
DAG_ID = 'gdrive_collector'
SCHEDULE_INTERVAL = '30 7 * * *'  # 07:30 todos os dias (UTC-3)
DESCRIPTION = 'Coleta automática de arquivos do Google Drive para Bronze Layer'

# Configurações padrão
default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'email': ['data-platform@farmarcas.com'],
    'retries': 2,
    'retry_delay': timedelta(minutes=15),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(hours=1)
}

# Criar DAG
dag = DAG(
    DAG_ID,
    default_args=default_args,
    description=DESCRIPTION,
    schedule_interval=SCHEDULE_INTERVAL,
    catchup=False,  # Não executar backfill automático
    max_active_runs=1,  # Apenas uma execução por vez
    tags=['data-platform', 'google-drive', 'bronze-layer', 'farmarcas']
)

# =====================================
# TASK 1: Validação de Pré-requisitos
# =====================================

def validate_prerequisites(**context):
    """
    Valida se todos os pré-requisitos estão atendidos
    """
    import boto3
    from google.oauth2 import service_account
    from googleapiclient.discovery import build
    
    logger = logging.getLogger(__name__)
    
    try:
        # 1. Validar acesso AWS S3
        s3_client = boto3.client('s3')
        bucket_name = Variable.get('GDRIVE_S3_BUCKET', 'farmarcas-production-bronze')
        
        s3_client.head_bucket(Bucket=bucket_name)
        logger.info(f"✅ S3 Bucket acessível: {bucket_name}")
        
        # 2. Validar Service Account Google (via variável Airflow)
        google_creds_json = Variable.get('GOOGLE_SERVICE_ACCOUNT_JSON')
        
        # Parse credentials
        import json
        creds_data = json.loads(google_creds_json)
        
        logger.info(f"✅ Google Service Account: {creds_data.get('client_email')}")
        
        # 3. Verificar conectividade Google Drive API
        credentials = service_account.Credentials.from_service_account_info(
            creds_data,
            scopes=['https://www.googleapis.com/auth/drive.readonly']
        )
        
        service = build('drive', 'v3', credentials=credentials)
        about_result = service.about().get(fields="user").execute()
        
        logger.info(f"✅ Google Drive API conectada")
        
        context['task_instance'].xcom_push(
            key='validation_status',
            value='success'
        )
        
        return True
        
    except Exception as e:
        logger.error(f"❌ Falha na validação: {str(e)}")
        raise

validate_task = PythonOperator(
    task_id='validate_prerequisites',
    python_callable=validate_prerequisites,
    dag=dag
)

# =====================================
# TASK 2: Execução Principal
# =====================================

# Configuração do Pod Kubernetes
k8s_pod_config = {
    # Imagem do container
    'image': 'farmarcas/gdrive-collector:latest',
    'image_pull_policy': 'Always',
    
    # Recursos computacionais
    'resources': {
        'requests': {
            'memory': '512Mi',
            'cpu': '0.5'
        },
        'limits': {
            'memory': '1Gi',
            'cpu': '1'
        }
    },
    
    # Configurações de rede e segurança
    'namespace': 'data-platform',
    'service_account_name': 'gdrive-collector',
    'security_context': {
        'run_as_non_root': True,
        'run_as_user': 1000,
        'fs_group': 1000
    },
    
    # Política de restart
    'restart_policy': 'Never',
    
    # Timeout da execução
    'startup_timeout_seconds': 300,
    'get_logs': True,
    
    # Labels para identificação
    'labels': {
        'app': 'gdrive-collector',
        'version': 'v1',
        'environment': 'production',
        'team': 'data-platform'
    }
}

# Variáveis de ambiente para o container
env_vars = {
    # Configurações AWS (via IAM Role)
    'AWS_DEFAULT_REGION': 'us-east-2',
    'S3_BUCKET_NAME': Variable.get('GDRIVE_S3_BUCKET', 'farmarcas-production-bronze'),
    
    # Configurações da aplicação
    'LOG_LEVEL': 'INFO',
    'ENVIRONMENT': 'production',
    'TZ': 'America/Sao_Paulo',
    
    # Configurações de performance
    'MAX_CONCURRENT_DOWNLOADS': '5',
    'DOWNLOAD_CHUNK_SIZE_MB': '10',
    'API_RATE_LIMIT': '10',
    
    # CloudWatch Logs
    'CLOUDWATCH_LOG_GROUP': '/aws/fargate/gdrive-collector',
    
    # Notifications
    'SLACK_WEBHOOK_URL': Variable.get('SLACK_WEBHOOK_URL', ''),
    'NOTIFY_ON_SUCCESS': 'false',
    'NOTIFY_ON_ERROR': 'true'
}

# Volumes e mounts para credenciais
volume_mounts = [
    {
        'name': 'google-service-account',
        'mount_path': '/app/credentials',
        'read_only': True
    },
    {
        'name': 'files-config',
        'mount_path': '/app/config',
        'read_only': True
    }
]

volumes = [
    {
        'name': 'google-service-account',
        'secret': {
            'secret_name': 'google-service-account'
        }
    },
    {
        'name': 'files-config',
        'config_map': {
            'name': 'gdrive-files-config'
        }
    }
]

# Task principal de execução
collect_gdrive_files = KubernetesPodOperator(
    task_id='collect_gdrive_files',
    name='gdrive-collector-{{ ds_nodash }}-{{ ts_nodash_with_tz }}',
    
    # Configurações do Pod
    **k8s_pod_config,
    
    # Variáveis de ambiente
    env_vars=env_vars,
    
    # Volumes
    volume_mounts=volume_mounts,
    volumes=volumes,
    
    # Configurações Airflow
    dag=dag,
    
    # Logs e debugging
    do_xcom_push=True,
    is_delete_operator_pod=True  # Limpar pod após execução
)

# =====================================
# TASK 3: Validação de Resultados
# =====================================

def validate_processing_results(**context):
    """
    Valida se o processamento foi executado corretamente
    """
    import boto3
    from datetime import datetime
    
    logger = logging.getLogger(__name__)
    
    try:
        # Obter data de execução
        execution_date = context['ds']  # YYYY-MM-DD
        
        # Verificar arquivos no S3
        s3_client = boto3.client('s3')
        bucket_name = Variable.get('GDRIVE_S3_BUCKET', 'farmarcas-production-bronze')
        
        # Prefixo esperado para arquivos do dia
        prefix = f"origin=eks/database=bronze_gdrive/"
        
        # Listar objetos criados hoje
        response = s3_client.list_objects_v2(
            Bucket=bucket_name,
            Prefix=prefix
        )
        
        if 'Contents' not in response:
            raise ValueError("Nenhum arquivo encontrado no S3")
        
        # Filtrar apenas arquivos de hoje
        today_files = []
        for obj in response['Contents']:
            if execution_date in obj['Key']:
                today_files.append(obj)
        
        if not today_files:
            raise ValueError(f"Nenhum arquivo processado para a data {execution_date}")
        
        # Validar tamanhos mínimos
        total_size = sum(obj['Size'] for obj in today_files)
        min_expected_size = 1024 * 1024  # 1MB mínimo
        
        if total_size < min_expected_size:
            logger.warning(f"Tamanho total muito pequeno: {total_size} bytes")
        
        # Log dos resultados
        logger.info(f"✅ Validação concluída:")
        logger.info(f"   📁 Arquivos processados: {len(today_files)}")
        logger.info(f"   💾 Tamanho total: {total_size / 1024 / 1024:.2f} MB")
        
        # Armazenar métricas no XCom
        context['task_instance'].xcom_push(
            key='processing_metrics',
            value={
                'files_count': len(today_files),
                'total_size_mb': round(total_size / 1024 / 1024, 2),
                'execution_date': execution_date,
                'files_list': [obj['Key'] for obj in today_files]
            }
        )
        
        return True
        
    except Exception as e:
        logger.error(f"❌ Falha na validação de resultados: {str(e)}")
        raise

validate_results = PythonOperator(
    task_id='validate_results',
    python_callable=validate_processing_results,
    dag=dag
)

# =====================================
# TASK 4: Notificação de Sucesso
# =====================================

def format_success_message(**context):
    """
    Formata mensagem de sucesso para Slack
    """
    # Obter métricas do XCom
    metrics = context['task_instance'].xcom_pull(
        task_ids='validate_results',
        key='processing_metrics'
    )
    
    execution_date = context['ds']
    
    message = f"""
🎉 *Google Drive Collector - Sucesso*

📅 *Data*: {execution_date}
📁 *Arquivos processados*: {metrics.get('files_count', 0)}
💾 *Tamanho total*: {metrics.get('total_size_mb', 0)} MB
⏱️ *Execução*: {{ ts }}

✅ Dados disponíveis na Bronze Layer
📊 Próxima execução: Amanhã às 07:30
    """
    
    return message.strip()

notify_success = SlackWebhookOperator(
    task_id='notify_success',
    http_conn_id='slack_webhook',
    message=format_success_message(),
    channel='#data-platform',
    username='Airflow Bot',
    icon_emoji=':robot_face:',
    dag=dag,
    trigger_rule='all_success'  # Só executa se todas as tasks anteriores tiveram sucesso
)

# =====================================
# TASK 5: Notificação de Falha
# =====================================

def format_failure_message(**context):
    """
    Formata mensagem de falha para Slack
    """
    failed_task = context.get('task_instance')
    execution_date = context['ds']
    
    message = f"""
🚨 *Google Drive Collector - FALHA*

📅 *Data*: {execution_date}
❌ *Task que falhou*: {failed_task.task_id if failed_task else 'Desconhecida'}
⏱️ *Timestamp*: {{ ts }}

🔍 *Ações necessárias*:
• Verificar logs no CloudWatch
• Validar conectividade Google Drive
• Confirmar acesso AWS S3
• Executar manualmente se necessário

📖 *Logs*: CloudWatch `/aws/fargate/gdrive-collector`
🔧 *Airflow*: [Ver DAG]({{ var.value.AIRFLOW_BASE_URL }}/tree?dag_id=gdrive_collector)
    """
    
    return message.strip()

notify_failure = SlackWebhookOperator(
    task_id='notify_failure',
    http_conn_id='slack_webhook',
    message=format_failure_message(),
    channel='#data-platform',
    username='Airflow Bot',
    icon_emoji=':rotating_light:',
    dag=dag,
    trigger_rule='one_failed'  # Executa se qualquer task anterior falhar
)

# =====================================
# DEPENDÊNCIAS DA DAG
# =====================================

# Fluxo principal
validate_task >> collect_gdrive_files >> validate_results >> notify_success

# Notificação de falha em paralelo
[validate_task, collect_gdrive_files, validate_results] >> notify_failure
```

### **1.2 Configuração de Conexões Airflow**

```python
# airflow_connections.py
"""
Script para configurar conexões necessárias no Airflow
"""

from airflow.models import Connection
from airflow.utils.db import provide_session

@provide_session
def create_connections(session=None):
    """
    Cria conexões necessárias para o Google Drive Collector
    """
    
    # Conexão Slack Webhook
    slack_conn = Connection(
        conn_id='slack_webhook',
        conn_type='http',
        host='hooks.slack.com',
        password='{{ var.value.SLACK_WEBHOOK_TOKEN }}',  # Token do webhook
        extra=json.dumps({
            'webhook_token': '{{ var.value.SLACK_WEBHOOK_TOKEN }}'
        })
    )
    
    # Adicionar conexões se não existirem
    existing_slack = session.query(Connection).filter(
        Connection.conn_id == 'slack_webhook'
    ).first()
    
    if not existing_slack:
        session.add(slack_conn)
        session.commit()
        print("✅ Conexão Slack criada")
    else:
        print("⚠️ Conexão Slack já existe")

if __name__ == "__main__":
    create_connections()
```

### **1.3 Variáveis Airflow**

```bash
# Definir variáveis via CLI
airflow variables set GDRIVE_S3_BUCKET "farmarcas-production-bronze"
airflow variables set SLACK_WEBHOOK_URL "https://hooks.slack.com/services/..."
airflow variables set SLACK_WEBHOOK_TOKEN "T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
airflow variables set AIRFLOW_BASE_URL "https://airflow.farmarcas.com"

# Variável com credenciais Google (JSON)
airflow variables set GOOGLE_SERVICE_ACCOUNT_JSON '{
  "type": "service_account",
  "project_id": "farmarcas-data-platform",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "client_email": "gdrive-collector@farmarcas-data-platform.iam.gserviceaccount.com",
  "client_id": "...",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}'
```

---

## ☸️ 2. Configuração Kubernetes

### **2.1 Namespace e RBAC**

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: data-platform
  labels:
    name: data-platform
    team: data-engineering

---
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gdrive-collector
  namespace: data-platform
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/gdrive-collector-role

---
# rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: data-platform
  name: gdrive-collector-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gdrive-collector-binding
  namespace: data-platform
subjects:
- kind: ServiceAccount
  name: gdrive-collector
  namespace: data-platform
roleRef:
  kind: Role
  name: gdrive-collector-role
  apiGroup: rbac.authorization.k8s.io
```

### **2.2 Secrets e ConfigMaps**

```yaml
# google-credentials-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: google-service-account
  namespace: data-platform
type: Opaque
data:
  # Base64 encoded do arquivo google_service_account.json
  google_service_account.json: ewogICJ0eXBlIjogInNlcnZpY2VfYWNjb3VudCIsCiAgInByb2plY3RfaWQiOi...

---
# files-config-configmap.yaml  
apiVersion: v1
kind: ConfigMap
metadata:
  name: gdrive-files-config
  namespace: data-platform
data:
  files_configs.yaml: |
    # Base de produtos
    base:
      name: "Base de Produtos"
      gdrive_file_id: "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms"
      table_name: "base"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"
    
    # Cadastro de lojas
    loja:
      name: "Cadastro de Lojas"
      gdrive_file_id: "1hsJJT5SVaE8bNcBGFstO7f0Lh8v8QnHG8v7f9G2mF4"
      table_name: "loja"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"
    
    # Distribuidores
    distribuidor:
      name: "Distribuidores"
      gdrive_file_id: "1mGQR7gH8vK3PlxY9N0sE2uZ6xW4qR8mF9N2sE3uZ7"
      table_name: "distribuidor"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"
    
    # Verbas dashboard
    verbas_base_industria:
      name: "Verbas Dashboard"
      gdrive_file_id: "1qR2sT4uV5wX6yZ7aB8cD9eF0gH1iJ2kL3mN4oP5qR6"
      table_name: "verbas_base_industria"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"
    
    # Pasta rebates
    rebates_template:
      name: "rebates_template"
      gdrive_folder_id: "1_4xO-dsMBrCOcr8uNdFHFhrmP7Lh2Y8_"
      table_name: "rebates_template"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"
    
    # Geografia
    geografia:
      name: "Geografia"
      gdrive_file_id: "1sT3uV4wX5yZ6aB7cD8eF9gH0iJ1kL2mN3oP4qR5sT6"
      table_name: "geografia"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"
    
    # Alíquotas ICMS
    aliquota_icms:
      name: "Alíquotas ICMS"
      gdrive_file_id: "1uV2wX3yZ4aB5cD6eF7gH8iJ9kL0mN1oP2qR3sT4uV5"
      table_name: "aliquota_icms"
      datalake:
        bucket: "farmarcas-production-bronze"
        database: "bronze_gdrive"

---
# network-policy.yaml (Opcional para segurança)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gdrive-collector-netpol
  namespace: data-platform
spec:
  podSelector:
    matchLabels:
      app: gdrive-collector
  policyTypes:
  - Egress
  egress:
  - to: []  # Permitir todo tráfego de saída
    ports:
    - protocol: TCP
      port: 443  # HTTPS para APIs
    - protocol: TCP
      port: 53   # DNS
    - protocol: UDP
      port: 53   # DNS
```

### **2.3 Comandos de Deploy**

```bash
#!/bin/bash
# deploy-k8s.sh

echo "🚀 Deploying Google Drive Collector to Kubernetes..."

# Aplicar namespace e RBAC
kubectl apply -f namespace.yaml
kubectl apply -f service-account.yaml
kubectl apply -f rbac.yaml

# Criar secrets (se não existem)
if ! kubectl get secret google-service-account -n data-platform &>/dev/null; then
    echo "📋 Criando secret para Google Service Account..."
    kubectl create secret generic google-service-account \
        --from-file=google_service_account.json=./credentials/google_service_account.json \
        -n data-platform
else
    echo "✅ Secret google-service-account já existe"
fi

# Aplicar ConfigMaps
kubectl apply -f files-config-configmap.yaml

# Verificar recursos criados
echo "📊 Verificando recursos criados:"
kubectl get serviceaccount,secrets,configmaps -n data-platform

# Verificar IAM Role association
echo "🔐 Verificando IAM Role do Service Account:"
kubectl describe serviceaccount gdrive-collector -n data-platform

echo "✅ Deploy concluído!"
echo "💡 Para testar execução manual:"
echo "   kubectl run gdrive-collector-test --image=farmarcas/gdrive-collector:latest --serviceaccount=gdrive-collector -n data-platform --rm -it --command -- python collector_gdrive/main.py"
```

---

## 🐳 3. Containerização e Deploy

### **3.1 Dockerfile Otimizado**

```dockerfile
# Dockerfile
FROM python:3.9-slim

# Metadados
LABEL maintainer="data-platform@farmarcas.com"
LABEL version="1.0"
LABEL description="Google Drive Collector for Farmarcas Data Platform"

# Instalar dependências do sistema
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Criar usuário não-root
RUN useradd -m -u 1000 collector && \
    mkdir -p /app /app/credentials /app/config && \
    chown -R collector:collector /app

WORKDIR /app

# Copiar e instalar requirements
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copiar código da aplicação
COPY collector_gdrive/ ./collector_gdrive/

# Mudar para usuário não-root
USER collector

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import google.auth; import boto3; print('Health check OK')" || exit 1

# Configurar ponto de entrada
ENTRYPOINT ["python", "collector_gdrive/main.py"]
```

### **3.2 Build e Push**

```bash
#!/bin/bash
# build-and-push.sh

# Configurações
IMAGE_NAME="farmarcas/gdrive-collector"
VERSION=${1:-"latest"}
REGISTRY="123456789012.dkr.ecr.us-east-2.amazonaws.com"

echo "🐳 Building Docker image: ${IMAGE_NAME}:${VERSION}"

# Build da imagem
docker build -t ${IMAGE_NAME}:${VERSION} .

# Tag para ECR
docker tag ${IMAGE_NAME}:${VERSION} ${REGISTRY}/${IMAGE_NAME}:${VERSION}

# Login no ECR
echo "🔐 Logging into ECR..."
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${REGISTRY}

# Push para ECR
echo "⬆️ Pushing to ECR..."
docker push ${REGISTRY}/${IMAGE_NAME}:${VERSION}

# Atualizar tag latest
if [ "$VERSION" != "latest" ]; then
    docker tag ${IMAGE_NAME}:${VERSION} ${REGISTRY}/${IMAGE_NAME}:latest
    docker push ${REGISTRY}/${IMAGE_NAME}:latest
fi

echo "✅ Image pushed successfully!"
echo "📝 Update Airflow DAG with: ${REGISTRY}/${IMAGE_NAME}:${VERSION}"
```

### **3.3 CI/CD Pipeline (GitHub Actions)**

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy Google Drive Collector

on:
  push:
    branches: [main]
    paths: 
      - 'collector_gdrive/**'
      - 'requirements.txt'
      - 'Dockerfile'
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-2
  ECR_REPOSITORY: farmarcas/gdrive-collector
  EKS_CLUSTER_NAME: farmarcas-data-platform

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-cov flake8
    
    - name: Lint with flake8
      run: |
        flake8 collector_gdrive/ --count --select=E9,F63,F7,F82 --show-source --statistics
    
    - name: Test with pytest
      run: |
        pytest collector_gdrive/tests/ -v --cov=collector_gdrive
  
  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
    
    - name: Update Kubernetes config
      run: |
        aws eks update-kubeconfig --region ${{ env.AWS_REGION }} --name ${{ env.EKS_CLUSTER_NAME }}
    
    - name: Deploy to Kubernetes
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        # Atualizar ConfigMap se necessário
        kubectl apply -f k8s/files-config-configmap.yaml
        
        # Trigger rolling update da DAG (se necessário)
        echo "✅ Deploy completed. Image: $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
```

---

## 📊 4. Monitoramento e Logs

### **4.1 CloudWatch Logs Configuration**

```python
# logging_config.py
"""
Configuração de logs para CloudWatch
"""

import logging
import watchtower
import boto3
from datetime import datetime

def setup_cloudwatch_logging():
    """
    Configura logging para CloudWatch
    """
    
    # Configurar handler CloudWatch
    cloudwatch_handler = watchtower.CloudWatchLogsHandler(
        log_group='/aws/fargate/gdrive-collector',
        stream_name=f'gdrive-collector-{datetime.now().strftime("%Y-%m-%d-%H-%M-%S")}',
        boto3_session=boto3.Session()
    )
    
    # Configurar formatação
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    cloudwatch_handler.setFormatter(formatter)
    
    # Configurar logger principal
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    logger.addHandler(cloudwatch_handler)
    
    # Configurar logger console (para Kubernetes logs)
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)
    
    return logger
```

### **4.2 Métricas Customizadas**

```python
# metrics.py
"""
Envio de métricas customizadas para CloudWatch
"""

import boto3
import time
from datetime import datetime

class MetricsCollector:
    """
    Classe para coletar e enviar métricas
    """
    
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.metrics_buffer = []
        
    def record_metric(self, name, value, unit='Count', dimensions=None):
        """
        Registra uma métrica
        """
        
        metric = {
            'MetricName': name,
            'Value': value,
            'Unit': unit,
            'Timestamp': datetime.utcnow()
        }
        
        if dimensions:
            metric['Dimensions'] = [
                {'Name': k, 'Value': v} for k, v in dimensions.items()
            ]
        
        self.metrics_buffer.append(metric)
    
    def send_metrics(self):
        """
        Envia métricas para CloudWatch
        """
        
        if not self.metrics_buffer:
            return
        
        try:
            self.cloudwatch.put_metric_data(
                Namespace='GDriveCollector',
                MetricData=self.metrics_buffer
            )
            
            print(f"📊 Enviadas {len(self.metrics_buffer)} métricas para CloudWatch")
            self.metrics_buffer = []
            
        except Exception as e:
            print(f"❌ Erro ao enviar métricas: {e}")

# Uso no main.py
def main():
    metrics = MetricsCollector()
    start_time = time.time()
    
    try:
        # ... processamento ...
        
        # Métricas de sucesso
        metrics.record_metric('FilesProcessed', files_count)
        metrics.record_metric('ProcessingTimeSeconds', time.time() - start_time, 'Seconds')
        metrics.record_metric('DataSizeBytes', total_bytes, 'Bytes')
        
    except Exception as e:
        # Métricas de erro
        metrics.record_metric('ErrorCount', 1)
        metrics.record_metric('ErrorType', 1, dimensions={'ErrorType': type(e).__name__})
        raise
    
    finally:
        metrics.send_metrics()
```

### **4.3 Dashboard CloudWatch**

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["GDriveCollector", "FilesProcessed"],
          [".", "ErrorCount"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-2",
        "title": "Google Drive Collector - Files Processed vs Errors"
      }
    },
    {
      "type": "metric", 
      "properties": {
        "metrics": [
          ["GDriveCollector", "ProcessingTimeSeconds"]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-2",
        "title": "Processing Time (seconds)"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["GDriveCollector", "DataSizeBytes"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-2",
        "title": "Data Volume (bytes)"
      }
    },
    {
      "type": "log",
      "properties": {
        "query": "SOURCE '/aws/fargate/gdrive-collector'\n| fields @timestamp, @message\n| filter @message like /ERROR/\n| sort @timestamp desc\n| limit 20",
        "region": "us-east-2",
        "title": "Recent Errors"
      }
    }
  ]
}
```

---

## 🔧 5. Execução e Operação

### **5.1 Execução Manual**

```bash
# Via Airflow UI
# 1. Acesse https://airflow.farmarcas.com
# 2. Localize DAG "gdrive_collector"
# 3. Clique "Trigger DAG"

# Via CLI Airflow
airflow dags trigger gdrive_collector

# Via kubectl (teste direto)
kubectl run gdrive-collector-manual \
    --image=123456789012.dkr.ecr.us-east-2.amazonaws.com/farmarcas/gdrive-collector:latest \
    --serviceaccount=gdrive-collector \
    --namespace=data-platform \
    --rm -it \
    --command -- python collector_gdrive/main.py

# Via Docker local (desenvolvimento)
docker run --rm \
    -e GOOGLE_APPLICATION_CREDENTIALS=/app/credentials/google_service_account.json \
    -e AWS_ACCESS_KEY_ID=AKIA... \
    -e AWS_SECRET_ACCESS_KEY=... \
    -e S3_BUCKET_NAME=farmarcas-development-bronze \
    -v $(pwd)/credentials:/app/credentials \
    -v $(pwd)/files_configs.yaml:/app/files_configs.yaml \
    farmarcas/gdrive-collector:latest
```

### **5.2 Monitoramento de Execução**

```bash
# Verificar status da DAG
airflow dags state gdrive_collector $(date +%Y-%m-%d)

# Ver logs específicos da task
airflow tasks log gdrive_collector collect_gdrive_files $(date +%Y-%m-%d)

# Monitorar pods Kubernetes
kubectl get pods -n data-platform -l app=gdrive-collector

# Ver logs do pod em tempo real
kubectl logs -f -n data-platform -l app=gdrive-collector

# Verificar eventos do Kubernetes
kubectl get events -n data-platform --sort-by='.lastTimestamp'

# Verificar arquivos no S3
aws s3 ls s3://farmarcas-production-bronze/origin=eks/database=bronze_gdrive/ --recursive --human-readable

# Verificar métricas CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace GDriveCollector \
    --metric-name FilesProcessed \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 3600 \
    --statistics Sum
```

### **5.3 Troubleshooting**

```bash
# Verificar recursos Kubernetes
kubectl describe serviceaccount gdrive-collector -n data-platform
kubectl describe secret google-service-account -n data-platform
kubectl describe configmap gdrive-files-config -n data-platform

# Testar conectividade AWS
kubectl run aws-test --rm -it --image=amazon/aws-cli:latest --serviceaccount=gdrive-collector -n data-platform -- sts get-caller-identity

# Testar acesso S3
kubectl run s3-test --rm -it --image=amazon/aws-cli:latest --serviceaccount=gdrive-collector -n data-platform -- s3 ls farmarcas-production-bronze

# Verificar logs detalhados
kubectl logs -n data-platform -l app=gdrive-collector --previous --tail=100

# Debug de rede
kubectl run netshoot --rm -it --image=nicolaka/netshoot -n data-platform -- nslookup www.googleapis.com
```

---

## 📋 6. Checklist de Deploy

### **6.1 Pré-Deploy**
```bash
✅ IAM Role configurado com políticas corretas
✅ Service Account Google criado e compartilhado
✅ ECR repository criado
✅ EKS cluster configurado
✅ Namespace data-platform criado
✅ Secrets e ConfigMaps aplicados
✅ Imagem Docker buildada e pushed
✅ Variáveis Airflow configuradas
✅ Conexões Airflow criadas
```

### **6.2 Pós-Deploy**
```bash
✅ DAG visível no Airflow UI
✅ Primeira execução manual bem-sucedida
✅ Logs aparecem no CloudWatch
✅ Métricas sendo enviadas
✅ Arquivos aparecem no S3
✅ Alertas Slack funcionando
✅ Tabelas criadas no Glue Catalog
✅ Schedule automático funcionando
```

### **6.3 Validação Contínua**
```bash
✅ Execução diária às 07:30 (UTC-3)
✅ ~7-10 arquivos processados por dia
✅ Tempo de execução < 10 minutos
✅ Volume de dados 2-5 MB por dia
✅ Zero falhas por 7 dias consecutivos
✅ Alertas funcionando corretamente
✅ Dados consistentes no data lake
✅ Performance estável
```

---

**Última Atualização**: 07/08/2025 - Sistema em produção com execução diária automatizada
