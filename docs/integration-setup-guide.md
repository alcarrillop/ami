# Guía de Integración y Configuración - Ami Assistant

## 📋 Índice
- [Requisitos Previos](#requisitos-previos)
- [Instalación Paso a Paso](#instalación-paso-a-paso)
- [Configuración de n8n](#configuración-de-n8n)
- [Configuración de Base de Datos](#configuración-de-base-de-datos)
- [Configuración de Herramientas MCP](#configuración-de-herramientas-mcp)
- [Configuración de Webhooks](#configuración-de-webhooks)
- [Variables de Entorno](#variables-de-entorno)
- [Testing y Validación](#testing-y-validación)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)

## 🛠️ Requisitos Previos

### Software Requerido

| Componente | Versión Mínima | Recomendada | Notas |
|------------|----------------|-------------|-------|
| **n8n** | 1.0.0 | 1.15.0+ | Self-hosted o Cloud |
| **Node.js** | 18.0.0 | 20.0.0+ | Para n8n self-hosted |
| **PostgreSQL** | 12.0 | 15.0+ | Base de datos principal |
| **Redis** | 6.0 | 7.0+ | Cache y sesiones |
| **Docker** | 20.0 | 24.0+ | Para deployment |

### Servicios Externos

- **Herramientas MCP**: API de validación de documentos
- **Servicio de Archivos**: S3 o compatible para almacenamiento
- **SMTP**: Para notificaciones por email
- **Webhook URLs**: Para integraciones externas

### Accesos Requeridos

- [ ] Cuenta de n8n (self-hosted o cloud)
- [ ] Credenciales de base de datos PostgreSQL
- [ ] API keys para herramientas MCP
- [ ] Configuración de almacenamiento de archivos
- [ ] Dominios y certificados SSL

## 📦 Instalación Paso a Paso

### Opción 1: Docker Compose (Recomendado)

1. **Clonar el repositorio**
   ```bash
   git clone https://github.com/habi/ami-assistant.git
   cd ami-assistant
   ```

2. **Configurar variables de entorno**
   ```bash
   cp .env.example .env
   # Editar .env con tus configuraciones
   nano .env
   ```

3. **Iniciar servicios**
   ```bash
   docker-compose up -d
   ```

4. **Verificar instalación**
   ```bash
   docker-compose ps
   curl http://localhost:5678/healthz
   ```

### Opción 2: Instalación Manual

1. **Instalar n8n**
   ```bash
   npm install -g n8n
   # O usando yarn
   yarn global add n8n
   ```

2. **Configurar base de datos**
   ```bash
   # Crear base de datos
   createdb ami_assistant
   
   # Ejecutar migraciones
   psql ami_assistant < database/schema.sql
   ```

3. **Instalar dependencias adicionales**
   ```bash
   npm install pg redis dotenv
   ```

4. **Iniciar n8n**
   ```bash
   export DB_TYPE=postgresdb
   export DB_POSTGRESDB_HOST=localhost
   export DB_POSTGRESDB_DATABASE=ami_assistant
   n8n start
   ```

## ⚙️ Configuración de n8n

### 1. Configuración Inicial

**Acceder a la interfaz web:**
```
http://localhost:5678
```

**Crear usuario administrador:**
- Email: admin@habi.co
- Password: [contraseña segura]

### 2. Configuración de Credenciales

#### PostgreSQL Database
```json
{
  "name": "AmiDatabase",
  "type": "postgres",
  "host": "{{ $env.DB_HOST }}",
  "port": "{{ $env.DB_PORT }}",
  "database": "{{ $env.DB_NAME }}",
  "username": "{{ $env.DB_USER }}",
  "password": "{{ $env.DB_PASSWORD }}",
  "ssl": {
    "enabled": true,
    "rejectUnauthorized": false
  }
}
```

#### MCP API Credentials
```json
{
  "name": "MCPValidation",
  "type": "httpHeaderAuth",
  "headerAuth": {
    "name": "Authorization",
    "value": "Bearer {{ $env.MCP_API_KEY }}"
  }
}
```

#### File Storage (S3)
```json
{
  "name": "DocumentStorage",
  "type": "aws",
  "accessKeyId": "{{ $env.AWS_ACCESS_KEY_ID }}",
  "secretAccessKey": "{{ $env.AWS_SECRET_ACCESS_KEY }}",
  "region": "{{ $env.AWS_REGION }}"
}
```

### 3. Importar Workflow

1. **Descargar workflow JSON**
   ```bash
   curl -o ami-workflow.json https://raw.githubusercontent.com/habi/ami-assistant/main/workflows/ami-main-workflow.json
   ```

2. **Importar en n8n**
   - Ir a "Workflows" → "Import from file"
   - Seleccionar `ami-workflow.json`
   - Configurar credenciales en cada nodo

3. **Activar workflow**
   ```bash
   # Via API
   curl -X POST http://localhost:5678/rest/workflows/activate \
     -H "Content-Type: application/json" \
     -d '{"id": "workflow_id"}'
   ```

## 🗄️ Configuración de Base de Datos

### 1. Schema de Base de Datos

```sql
-- Crear extensiones necesarias
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Tabla principal de leads
CREATE TABLE leads (
  lead_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  full_name VARCHAR(255),
  phone_e164 VARCHAR(20),
  email VARCHAR(255),
  address TEXT,
  status VARCHAR(50) NOT NULL DEFAULT 'start',
  documents_status JSONB DEFAULT '{}',
  completion_timestamp TIMESTAMP,
  ready_for_human_review BOOLEAN DEFAULT FALSE,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Tabla de documentos
CREATE TABLE lead_documents (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  lead_id UUID REFERENCES leads(lead_id) ON DELETE CASCADE,
  doc_type VARCHAR(50) NOT NULL,
  file_path VARCHAR(500),
  file_name VARCHAR(255),
  file_size INTEGER,
  mime_type VARCHAR(100),
  validation_status VARCHAR(50) DEFAULT 'pending',
  validation_details JSONB,
  confidence_score DECIMAL(3,2),
  uploaded_at TIMESTAMP DEFAULT NOW(),
  validated_at TIMESTAMP
);

-- Tabla de interacciones
CREATE TABLE lead_interactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  lead_id UUID REFERENCES leads(lead_id) ON DELETE CASCADE,
  interaction_type VARCHAR(50) NOT NULL,
  message_content TEXT,
  response_content TEXT,
  file_attached BOOLEAN DEFAULT FALSE,
  processing_time INTEGER, -- en millisegundos
  timestamp TIMESTAMP DEFAULT NOW()
);

-- Índices para performance
CREATE INDEX idx_leads_status ON leads(status);
CREATE INDEX idx_leads_created_at ON leads(created_at);
CREATE INDEX idx_leads_phone ON leads(phone_e164);
CREATE INDEX idx_documents_lead_id ON lead_documents(lead_id);
CREATE INDEX idx_documents_type ON lead_documents(doc_type);
CREATE INDEX idx_interactions_lead_id ON lead_interactions(lead_id);

-- Trigger para updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_leads_updated_at 
  BEFORE UPDATE ON leads 
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 2. Datos de Configuración

```sql
-- Configuraciones del sistema
CREATE TABLE system_config (
  key VARCHAR(100) PRIMARY KEY,
  value JSONB NOT NULL,
  description TEXT,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Insertar configuraciones iniciales
INSERT INTO system_config (key, value, description) VALUES
('document_types', '["cedula", "servicio_publico", "ctl", "paz_y_salvo_admin"]', 'Tipos de documentos válidos'),
('max_file_size_mb', '10', 'Tamaño máximo de archivo en MB'),
('supported_formats', '["pdf", "jpg", "jpeg", "png"]', 'Formatos de archivo soportados'),
('ctv_max_age_days', '30', 'Edad máxima del CTL en días'),
('service_max_age_months', '2', 'Edad máxima del servicio público en meses');
```

### 3. Procedimientos Almacenados

```sql
-- Función para calcular progreso
CREATE OR REPLACE FUNCTION calculate_lead_progress(p_lead_id UUID)
RETURNS JSONB AS $$
DECLARE
  doc_status JSONB;
  completed_count INTEGER := 0;
  total_count INTEGER := 4;
  percentage INTEGER;
BEGIN
  SELECT documents_status INTO doc_status
  FROM leads WHERE lead_id = p_lead_id;
  
  -- Contar documentos completados
  IF (doc_status->>'cedula') = 'ok' THEN completed_count := completed_count + 1; END IF;
  IF (doc_status->>'servicio_publico') = 'ok' THEN completed_count := completed_count + 1; END IF;
  IF (doc_status->>'ctl') = 'ok' THEN completed_count := completed_count + 1; END IF;
  IF (doc_status->>'paz_y_salvo_admin') = 'ok' THEN completed_count := completed_count + 1; END IF;
  
  percentage := (completed_count * 100) / total_count;
  
  RETURN jsonb_build_object(
    'completed', completed_count,
    'total', total_count,
    'percentage', percentage
  );
END;
$$ LANGUAGE plpgsql;
```

## 🔧 Configuración de Herramientas MCP

### 1. Configuración del Cliente MCP

```javascript
// config/mcp-client.js
const MCPClient = {
  baseURL: process.env.MCP_API_ENDPOINT,
  apiKey: process.env.MCP_API_KEY,
  timeout: 30000,
  retries: 3,
  
  // Configuración por tipo de documento
  validationConfig: {
    cedula: {
      required_fields: ['front_side', 'back_side'],
      max_age_years: null, // Sin límite de edad
      confidence_threshold: 0.8
    },
    servicio_publico: {
      required_fields: ['address', 'account_holder'],
      max_age_months: 2,
      confidence_threshold: 0.7
    },
    ctl: {
      required_fields: ['property_folio', 'owner_info'],
      max_age_days: 30,
      confidence_threshold: 0.9
    },
    paz_y_salvo_admin: {
      required_fields: ['official_seal', 'signature'],
      max_age_months: 1,
      confidence_threshold: 0.8
    }
  }
};
```

### 2. Configuración de Endpoints MCP

```json
{
  "mcp_endpoints": {
    "validate_lead_fields": {
      "url": "/api/v1/validate/lead",
      "method": "POST",
      "timeout": 5000
    },
    "validate_document": {
      "url": "/api/v1/validate/document",
      "method": "POST", 
      "timeout": 30000
    },
    "extract_document_data": {
      "url": "/api/v1/extract/document",
      "method": "POST",
      "timeout": 25000
    }
  }
}
```

### 3. Configuración de Fallbacks

```javascript
// Configuración de fallbacks cuando MCP no está disponible
const fallbackConfig = {
  validation_fallback: 'human_review',
  notify_admin: true,
  max_fallback_time: 3600000, // 1 hora
  fallback_message: "Estoy experimentando problemas técnicos. Un asesor humano te contactará pronto."
};
```

## 🔗 Configuración de Webhooks

### 1. Webhook de Entrada (n8n)

**URL del webhook:**
```
https://your-n8n-instance.com/webhook/ami-input
```

**Configuración en n8n:**
```json
{
  "httpMethod": "POST",
  "path": "ami-input",
  "authentication": "headerAuth",
  "options": {
    "allowedOrigins": ["https://your-chat-app.com"],
    "rawBody": false
  }
}
```

### 2. Webhooks de Salida

**Para notificar a sistemas externos:**

```javascript
// Nodo HTTP Request en n8n
{
  "method": "POST",
  "url": "{{ $env.EXTERNAL_WEBHOOK_URL }}",
  "headers": {
    "Authorization": "Bearer {{ $env.EXTERNAL_API_KEY }}",
    "Content-Type": "application/json",
    "X-Ami-Event": "{{ $json.event_type }}"
  },
  "body": {
    "event": "{{ $json.event_type }}",
    "lead_id": "{{ $json.lead_id }}",
    "timestamp": "{{ $now }}",
    "data": "{{ $json }}"
  }
}
```

### 3. Configuración de Retry y Error Handling

```json
{
  "webhook_config": {
    "retry_policy": {
      "max_attempts": 3,
      "backoff": "exponential",
      "initial_delay": 1000
    },
    "error_handling": {
      "on_failure": "log_and_continue",
      "notify_admin": true,
      "fallback_action": "queue_for_manual_processing"
    }
  }
}
```

## 🌍 Variables de Entorno

### Archivo .env Completo

```bash
# === CONFIGURACIÓN GENERAL ===
NODE_ENV=production
LOG_LEVEL=info
DEBUG=false

# === N8N CONFIGURACIÓN ===
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_ENCRYPTION_KEY=your-very-secure-encryption-key-32-chars

# === BASE DE DATOS ===
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=localhost
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=ami_assistant
DB_POSTGRESDB_USER=ami_user
DB_POSTGRESDB_PASSWORD=secure_password
DB_POSTGRESDB_SSL_CA=/path/to/ca-cert.pem
DB_POSTGRESDB_SSL_CERT=/path/to/client-cert.pem
DB_POSTGRESDB_SSL_KEY=/path/to/client-key.pem

# === REDIS (CACHE Y SESIONES) ===
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis_password
REDIS_DB=0

# === MCP API ===
MCP_API_ENDPOINT=https://api.mcp-provider.com/v1
MCP_API_KEY=your-mcp-api-key
MCP_TIMEOUT=30000
MCP_MAX_RETRIES=3

# === ALMACENAMIENTO DE ARCHIVOS ===
STORAGE_TYPE=s3
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=us-east-1
AWS_S3_BUCKET=ami-documents
AWS_S3_PREFIX=documents/

# === WEBHOOKS ===
WEBHOOK_SECRET=your-webhook-secret-key
EXTERNAL_WEBHOOK_URL=https://your-system.com/webhooks/ami
EXTERNAL_API_KEY=your-external-api-key

# === NOTIFICACIONES ===
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=notifications@habi.co
SMTP_PASSWORD=smtp_password
ADMIN_EMAIL=admin@habi.co

# === SEGURIDAD ===
JWT_SECRET=your-jwt-secret-key
API_RATE_LIMIT=100
CORS_ORIGINS=https://your-frontend.com,https://admin.habi.co

# === MONITORING ===
SENTRY_DSN=https://your-sentry-dsn@sentry.io/project
METRICS_ENDPOINT=https://metrics.habi.co/ami
```

### Validación de Variables

```bash
# Script para validar configuración
#!/bin/bash

echo "🔍 Validando configuración de Ami..."

# Verificar variables críticas
check_var() {
  if [ -z "${!1}" ]; then
    echo "❌ Variable $1 no está configurada"
    exit 1
  else
    echo "✅ $1 configurada"
  fi
}

check_var "DB_POSTGRESDB_HOST"
check_var "MCP_API_ENDPOINT"
check_var "MCP_API_KEY"
check_var "WEBHOOK_SECRET"
check_var "N8N_ENCRYPTION_KEY"

echo "🎉 Configuración válida!"
```

## 🧪 Testing y Validación

### 1. Tests de Conectividad

```bash
# Test de base de datos
psql -h $DB_POSTGRESDB_HOST -U $DB_POSTGRESDB_USER -d $DB_POSTGRESDB_DATABASE -c "SELECT version();"

# Test de MCP API
curl -H "Authorization: Bearer $MCP_API_KEY" $MCP_API_ENDPOINT/health

# Test de Redis
redis-cli -h $REDIS_HOST -p $REDIS_PORT ping

# Test de S3
aws s3 ls s3://$AWS_S3_BUCKET --region $AWS_REGION
```

### 2. Tests de Workflow

```javascript
// test-workflow.js
const axios = require('axios');

async function testWorkflow() {
  const baseURL = 'http://localhost:5678';
  
  // Test 1: Iniciar conversación
  console.log('🧪 Testing conversation start...');
  const startResponse = await axios.post(`${baseURL}/webhook/ami-input`, {
    lead_id: 'test-lead-123',
    message: 'Hola'
  });
  
  console.log('✅ Start response:', startResponse.data);
  
  // Test 2: Enviar datos básicos
  console.log('🧪 Testing basic data collection...');
  const dataResponse = await axios.post(`${baseURL}/webhook/ami-input`, {
    lead_id: 'test-lead-123',
    message: 'Juan Pérez'
  });
  
  console.log('✅ Data response:', dataResponse.data);
  
  // Test 3: Simular validación de documento
  console.log('🧪 Testing document validation...');
  const docResponse = await axios.post(`${baseURL}/webhook/ami-input`, {
    lead_id: 'test-lead-123',
    agent_message: {
      status: 'auto_ok',
      doc_type: 'cedula',
      confidence: 0.95
    }
  });
  
  console.log('✅ Document response:', docResponse.data);
}

testWorkflow().catch(console.error);
```

### 3. Tests de Carga

```bash
# Usar Artillery para tests de carga
npm install -g artillery

# Crear configuración de test
cat > load-test.yml << EOF
config:
  target: 'http://localhost:5678'
  phases:
    - duration: 60
      arrivalRate: 10
scenarios:
  - name: "Ami workflow test"
    flow:
      - post:
          url: "/webhook/ami-input"
          json:
            lead_id: "load-test-{{ \$uuid }}"
            message: "Test message"
EOF

# Ejecutar test
artillery run load-test.yml
```

## 🚀 Deployment

### 1. Deployment con Docker

**docker-compose.prod.yml:**
```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=${DB_NAME}
      - DB_POSTGRESDB_USER=${DB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./workflows:/home/node/.n8n/workflows
    ports:
      - "5678:5678"
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/schema.sql:/docker-entrypoint-initdb.d/schema.sql

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl/certs
    depends_on:
      - n8n

volumes:
  n8n_data:
  postgres_data:
  redis_data:
```

### 2. Configuración de Nginx

```nginx
# nginx/nginx.conf
upstream n8n {
    server n8n:5678;
}

server {
    listen 443 ssl http2;
    server_name ami.habi.co;

    ssl_certificate /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/certs/privkey.pem;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=ami:10m rate=10r/s;

    location / {
        limit_req zone=ami burst=20 nodelay;
        
        proxy_pass http://n8n;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### 3. Scripts de Deployment

```bash
#!/bin/bash
# deploy.sh

set -e

echo "🚀 Deploying Ami Assistant..."

# Backup base de datos
echo "📦 Creating database backup..."
pg_dump $DB_POSTGRESDB_DATABASE > "backup-$(date +%Y%m%d-%H%M%S).sql"

# Pull latest images
echo "⬇️  Pulling latest images..."
docker-compose -f docker-compose.prod.yml pull

# Update workflows
echo "🔄 Updating workflows..."
curl -X POST http://localhost:5678/rest/workflows/import \
  -H "Content-Type: application/json" \
  -d @workflows/ami-main-workflow.json

# Restart services
echo "🔄 Restarting services..."
docker-compose -f docker-compose.prod.yml up -d

# Wait for services
echo "⏳ Waiting for services to be ready..."
sleep 30

# Health check
echo "🏥 Performing health check..."
curl -f http://localhost:5678/healthz || exit 1

echo "✅ Deployment completed successfully!"
```

## 🔧 Troubleshooting

### Problemas Comunes

#### 1. Error de Conexión a Base de Datos
**Síntoma:** `ECONNREFUSED` o `connection timeout`

**Solución:**
```bash
# Verificar conectividad
nc -zv $DB_POSTGRESDB_HOST $DB_POSTGRESDB_PORT

# Verificar credenciales
psql "postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST:$DB_PORT/$DB_NAME" -c "\dt"

# Verificar logs
docker logs postgres_container
```

#### 2. MCP API No Responde
**Síntoma:** Timeouts en validación de documentos

**Solución:**
```bash
# Test manual de MCP API
curl -v -H "Authorization: Bearer $MCP_API_KEY" \
  $MCP_API_ENDPOINT/health

# Verificar configuración de timeout
# Aumentar timeout en configuración de n8n si es necesario
```

#### 3. Workflows No Se Ejecutan
**Síntoma:** Webhooks recibidos pero sin respuesta

**Solución:**
```bash
# Verificar logs de n8n
docker logs n8n_container

# Verificar estado del workflow
curl -H "Authorization: Bearer $N8N_API_KEY" \
  http://localhost:5678/rest/workflows

# Reactivar workflow si es necesario
curl -X POST http://localhost:5678/rest/workflows/{id}/activate
```

### Logs y Monitoring

```bash
# Ver logs en tiempo real
docker-compose logs -f n8n

# Logs de base de datos
docker-compose logs postgres

# Métricas de sistema
docker stats

# Espacio en disco
df -h
docker system df
```

### Scripts de Mantenimiento

```bash
#!/bin/bash
# maintenance.sh

# Limpiar logs antiguos
find /var/log/ami -name "*.log" -mtime +7 -delete

# Limpiar documentos temporales
find /tmp/ami-uploads -mtime +1 -delete

# Optimizar base de datos
psql $DB_NAME -c "VACUUM ANALYZE;"

# Limpiar imágenes Docker no utilizadas
docker system prune -f

echo "🧹 Mantenimiento completado"
```

---

**📞 Para soporte durante la instalación, contacta al equipo técnico de Habi con los logs específicos del error.**
