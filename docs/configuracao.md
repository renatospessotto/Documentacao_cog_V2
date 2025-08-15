# ⚙️ Configuração - Cognitivo Data Platform

## 📋 Configurações por Sistema

### **ACODE + Redundância**
```yaml
# config/acode.yaml
acode:
  mysql:
    host: "db-hsp-farmarcas.acode.com.br"
    port: 3306
    database: "acode_farmarcas"
    user: "userfarmarcasac02"
    password: "${ACODE_PASS}"
  
  airbyte:
    connection_id: "connection_mysql_s3_acode"
    sync_mode: "full_refresh"
    
  redundancy:
    enabled: true
    backup_connection: "connection_mysql_s3_acode_backup"
    health_check_interval: 300  # 5 minutes
```

### **Radar Collector**
```yaml
# config/radar.yaml
radar:
  mysql:
    host: "db-mysql-radar-production"
    port: 3306
    user: "bi-cognitivo-read"
    password: "${RADAR_PASS}"
    
  tables:
    - store
    - store_metrics
    - product
    - contest
    - user_access
    # ... 80+ tables total
```

### **Google Drive Collector**
```yaml
# config/gdrive.yaml
google_drive:
  credentials_path: "${GOOGLE_CREDENTIALS_PATH}"
  folder_id: "${GOOGLE_DRIVE_FOLDER_ID}"
  
  files:
    - name: "base"
      file_id: "1xyz..."
      sheet_names: ["base"]
    - name: "loja" 
      file_id: "1abc..."
      sheet_names: ["loja"]
```

## 🌐 Configurações por Ambiente

### **Development**
```bash
export ENV="development"
export AWS_PROFILE="farmarcas-dev"
export S3_BUCKET="farmarcas-dev-bronze"
```

### **Production**
```bash
export ENV="production"
export AWS_PROFILE="farmarcas-production"
export S3_BUCKET="farmarcas-production-bronze"
```

---

**Próximo**: [Troubleshooting](./troubleshooting.md)
