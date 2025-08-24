# Workflows de n8n - Ami Assistant

## 📋 Descripción

Esta carpeta contiene los workflows de n8n que implementan la lógica del asistente Ami. Los workflows están diseñados para ser importados directamente en una instancia de n8n.

## 📁 Estructura de Archivos

```
workflows/
├── README.md                           # Este archivo
├── hackathon-lead-habi.json           # Workflow principal de Ami 
├── mcp-documents.json                 # Workflow de validación de documentos MCP
├── hackathon-lead-habi-placeholder.json  # Placeholder (eliminar después)
└── mcp-documents-placeholder.json     # Placeholder (eliminar después)
```

## 🚨 Importante

**Los workflows de hackathon-lead-habi y mcp-documents están listos para ser añadidos.**

Para completar la documentación, necesitarás:

1. **Exportar el workflow `hackathon-lead-habi` desde tu instancia de n8n**
2. **Exportar el workflow `mcp-documents` desde tu instancia de n8n**
3. **Reemplazar los archivos placeholder con los workflows reales**

## 📥 Cómo Exportar Workflows desde n8n

1. **Acceder a tu instancia de n8n**
   ```
   https://your-n8n-instance.com
   ```

2. **Navegar al workflow de Ami**
   - Ir a "Workflows"
   - Seleccionar el workflow principal de Ami

3. **Exportar el workflow**
   - Hacer clic en el menú "..." (tres puntos)
   - Seleccionar "Download"
   - Guardar como `ami-main-workflow.json`

4. **Mover los archivos a este directorio**
   ```bash
   # Workflow principal de Ami
   mv ~/Downloads/hackathon-lead-habi.json ./workflows/hackathon-lead-habi.json
   
   # Workflow de validación de documentos MCP
   mv ~/Downloads/mcp-documents.json ./workflows/mcp-documents.json
   
   # Eliminar placeholders
   rm ./workflows/hackathon-lead-habi-placeholder.json
   rm ./workflows/mcp-documents-placeholder.json
   ```

## 📤 Cómo Importar Workflows en n8n

### Opción 1: Interfaz Web

1. **Acceder a n8n**
2. **Ir a "Workflows"**
3. **Hacer clic en "Import from file"**
4. **Seleccionar el archivo JSON**
5. **Configurar credenciales necesarias**
6. **Activar el workflow**

### Opción 2: API de n8n

```bash
# Importar workflow principal via API
curl -X POST http://localhost:5678/rest/workflows/import \
  -H "Content-Type: application/json" \
  -d @workflows/hackathon-lead-habi.json

# Importar workflow de validación MCP
curl -X POST http://localhost:5678/rest/workflows/import \
  -H "Content-Type: application/json" \
  -d @workflows/mcp-documents.json
```

### Opción 3: CLI (si tienes acceso)

```bash
# Usando n8n CLI para importar ambos workflows
n8n import:workflow --file=workflows/hackathon-lead-habi.json
n8n import:workflow --file=workflows/mcp-documents.json
```

## ⚙️ Configuración Post-Importación

Después de importar los workflows, necesitarás:

### 1. Configurar Credenciales

- **Database (PostgreSQL)**
- **MCP API Keys**
- **S3 Storage**
- **SMTP Settings**

### 2. Verificar URLs de Webhook

```bash
# Verificar que el webhook esté activo
curl -X POST https://your-n8n-instance.com/webhook/ami-input \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

### 3. Activar el Workflow

```bash
# Via interfaz web o API
curl -X POST http://localhost:5678/rest/workflows/{workflow_id}/activate
```

## 🧪 Testing de Workflows

### Test Básico
```json
{
  "lead_id": "test-123",
  "message": "Hola Ami"
}
```

### Test con Documento
```json
{
  "lead_id": "test-123",
  "message": "archivo_subido.pdf",
  "file_data": {
    "content": "base64_encoded_content",
    "mime_type": "application/pdf"
  }
}
```

### Test de Validación
```json
{
  "lead_id": "test-123",
  "agent_message": {
    "status": "auto_ok",
    "doc_type": "cedula",
    "confidence": 0.95
  }
}
```

## 📝 Notas para el Desarrollo

### Estructura Esperada del Workflow Principal

El workflow principal debe incluir estos nodos clave:

1. **Webhook Trigger** - Punto de entrada
2. **Input Parser** - Procesamiento de entrada
3. **State Manager** - Control de flujo
4. **Lead Data Manager** - Gestión de datos del lead
5. **MCP Validation** - Validación de documentos
6. **Response Generator** - Generación de respuestas
7. **Error Handler** - Manejo de errores

### Variables de Entorno Requeridas

El workflow debe usar estas variables:

```javascript
// Variables de entorno que el workflow debe referenciar
const requiredEnvVars = [
  'MCP_API_ENDPOINT',
  'MCP_API_KEY', 
  'DB_POSTGRESDB_HOST',
  'DB_POSTGRESDB_DATABASE',
  'DB_POSTGRESDB_USER',
  'DB_POSTGRESDB_PASSWORD',
  'AWS_S3_BUCKET',
  'WEBHOOK_SECRET'
];
```

## 🔄 Versionado de Workflows

### Convención de Nombres
```
ami-main-workflow-v1.0.0.json
ami-main-workflow-v1.1.0.json
ami-main-workflow-v2.0.0.json
```

### Changelog de Versiones

#### v1.0.0 (Pendiente)
- [ ] Workflow inicial completo
- [ ] Flujo de 4 documentos
- [ ] Integración con MCP
- [ ] Persistencia en PostgreSQL

#### v1.1.0 (Futuro)
- [ ] Mejoras en manejo de errores
- [ ] Optimizaciones de performance
- [ ] Logging mejorado

#### v2.0.0 (Futuro)
- [ ] Soporte para WhatsApp
- [ ] Validación ML mejorada
- [ ] Dashboard de métricas

## 🚀 Deploy en Producción

### Checklist Pre-Deploy

- [ ] Workflow exportado desde desarrollo
- [ ] Credenciales configuradas
- [ ] Variables de entorno validadas
- [ ] Tests básicos ejecutados
- [ ] Backup de workflow anterior (si existe)

### Comando de Deploy

```bash
#!/bin/bash
# deploy-workflow.sh

echo "🚀 Deploying Ami workflow..."

# Backup workflow actual
curl -s http://localhost:5678/rest/workflows > backup-workflows-$(date +%Y%m%d).json

# Importar nuevo workflow
curl -X POST http://localhost:5678/rest/workflows/import \
  -H "Content-Type: application/json" \
  -d @workflows/ami-main-workflow.json

# Activar workflow
curl -X POST http://localhost:5678/rest/workflows/{id}/activate

echo "✅ Workflow deployed successfully!"
```

---

**🔄 Recuerda actualizar este directorio cada vez que hagas cambios en el workflow de n8n!**
