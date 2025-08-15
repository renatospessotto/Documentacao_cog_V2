# 🏗️ Cognitivo Data Platform - Documentação Técnica

[![Status](https://img.shields.io/badge/Status-Ativo-brightgreen.svg)](https://github.com/cognitivo/cdp-docs)
[![Environment](https://img.shields.io/badge/Environment-Production-red.svg)](docs/environments.md)
[![Coverage](https://img.shields.io/badge/Cobertura-100%25-green.svg)](docs/coverage.md)
[![SLA](https://img.shields.io/badge/SLA-99.9%25-blue.svg)](docs/sla.md)

## 📋 Descrição

A **Cognitivo Data Platform (CDP)** é uma plataforma abrangente de ingestão, processamento e distribuição de dados para a Farmarcas, implementando pipelines robustos e automatizados para coleta de informações críticas de múltiplas fontes. A plataforma orquestra a ingestão de dados fiscais, comerciais, operacionais e de terceiros, garantindo alta disponibilidade, integridade e conformidade regulatória.

Este repositório centraliza a documentação técnica completa de todos os sistemas da CDP, incluindo coletores especializados (ACODE, Radar, Google Drive), mecanismos de redundância, configurações operacionais e procedimentos de manutenção. A arquitetura suporta processamento de milhões de registros diários com SLA de 99.9% de disponibilidade, atendendo às necessidades críticas de business intelligence, compliance fiscal e tomada de decisão estratégica da organização.

## 🚀 Quick Start

### Instalação
```bash
# Clone o repositório
git clone <repository-url>
cd cdp-documentacao

# Configure as credenciais necessárias
cp .env.example .env
vim .env  # Configure suas variáveis

# Verificar conectividade com sistemas
./scripts/health-check-all.sh
```

### Uso Básico
```bash
# Verificar status de todos os sistemas
./scripts/status-dashboard.sh

# Executar diagnóstico completo
./scripts/diagnostic-full.sh

# Acessar logs em tempo real
./scripts/logs-monitor.sh
```

### Acesso Rápido aos Sistemas
```bash
# ACODE - Sistema fiscal
cd docs/acode_redundancia/ && ./quick-start.sh

# Radar - Sistema farmacêutico  
cd docs/radar/ && ./quick-start.sh

# Google Drive - Collector de arquivos
cd docs/google_drive/ && ./quick-start.sh
```

## ✨ Principais Funcionalidades

### 🔄 **Sistemas de Ingestão**
- **ACODE + Redundância**: Pipeline crítico MySQL → S3 com failover automático
- **Radar Collector**: Sincronização Airbyte de dados farmacêuticos e métricas
- **Google Drive Collector**: Automação de coleta de planilhas e documentos

### 🛡️ **Alta Disponibilidade**
- **Redundância Multi-Sistema**: Backup automático e recovery procedures
- **Monitoramento 24/7**: Health checks, alertas em tempo real
- **SLA 99.9%**: Garantia de disponibilidade com RTO < 2 horas

### 📊 **Processamento de Dados**
- **Volume Massivo**: 50M+ registros fiscais processados diariamente
- **Multi-Formato**: Excel, CSV, Parquet com schema validation
- **Particionamento Inteligente**: Otimização S3 por data e origem

### 🔐 **Compliance e Segurança**
- **Auditoria Completa**: Logs estruturados e rastreabilidade
- **Credenciais Seguras**: AWS Secrets Manager e rotação automática
- **Validação de Dados**: Quality checks e data profiling

### 🎯 **Orquestração Avançada**
- **Airflow Integration**: DAGs automatizadas com dependências
- **Kubernetes Native**: Containerização e auto-scaling
- **API Management**: Endpoints padronizados e versionamento

### 📈 **Observabilidade**
- **Métricas Detalhadas**: Performance, throughput e error rates
- **Dashboards Operacionais**: Grafana + CloudWatch integration
- **Alertas Inteligentes**: Slack, PagerDuty e email notifications

## 📚 Documentação Completa

Para informações detalhadas sobre cada sistema, consulte nossa **[documentação modular](./docs/)**:

### 🔄 **Sistemas de Ingestão**
- 📖 **[ACODE + Redundância](./docs/acode_redundancia/)** - Pipeline fiscal crítico com backup automático
- 📊 **[Radar Collector](./docs/radar/)** - Dados farmacêuticos via Airbyte
- 📁 **[Google Drive Collector](./docs/google_drive/)** - Automação de planilhas e documentos

### 🛠️ **Operações e Configuração**
- ⚙️ **[Guia de Instalação](./docs/instalacao.md)** - Setup completo da plataforma
- 🏗️ **[Arquitetura Geral](./docs/arquitetura.md)** - Visão técnica consolidada
- 🔧 **[Configurações](./docs/configuracao.md)** - Parâmetros e variáveis por sistema

### 🚨 **Manutenção e Suporte**
- ⚠️ **[Troubleshooting](./docs/troubleshooting.md)** - Solução de problemas comuns
- 💡 **[Boas Práticas](./docs/boas-praticas.md)** - Recomendações operacionais
- 📊 **[Monitoramento](./docs/monitoramento.md)** - Métricas, alertas e observabilidade

### 📋 **Procedimentos**
- 🔄 **[Backup & Recovery](./docs/backup-recovery.md)** - Procedimentos de contingência
- 🔒 **[Segurança](./docs/seguranca.md)** - Credenciais, permissões e compliance
- 📈 **[Performance](./docs/performance.md)** - Otimização e scaling guidelines

---

## 📊 Métricas da Plataforma

### **Volume de Dados Processados**
- **ACODE**: ~50M+ registros fiscais/dia | ~4-8GB
- **Radar**: ~1M+ registros farmacêuticos/dia | ~500MB-2GB  
- **Google Drive**: ~200+ arquivos/dia | ~100-500MB

### **SLAs Operacionais**
- **Disponibilidade**: 99.9% (target) | 99.95% (atual)
- **RTO (Recovery Time)**: < 2 horas
- **RPO (Recovery Point)**: < 1 hora
- **Data Freshness**: < 4 horas para dashboards

### **Cobertura de Monitoramento**
- **Health Checks**: 100% dos sistemas críticos
- **Alertas Ativos**: 50+ regras configuradas
- **Dashboards**: 15+ painéis operacionais
- **Logs Centralizados**: 100% dos componentes

---

**Suporte**: Data Engineering Team | **Criticidade**: Alta | **Última Atualização**: 15/08/2025
