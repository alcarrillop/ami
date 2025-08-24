# Guía de Contribución - Ami Assistant

## 🤝 Bienvenido a Ami

¡Gracias por tu interés en contribuir al proyecto Ami! Esta guía te ayudará a entender cómo puedes colaborar efectivamente.

## 📋 Índice
- [Código de Conducta](#código-de-conducta)
- [Tipos de Contribución](#tipos-de-contribución)
- [Configuración del Entorno](#configuración-del-entorno)
- [Proceso de Desarrollo](#proceso-de-desarrollo)
- [Estándares de Código](#estándares-de-código)
- [Testing](#testing)
- [Documentación](#documentación)
- [Pull Requests](#pull-requests)

## 🌟 Código de Conducta

### Nuestros Valores
- **Respeto**: Tratamos a todos con dignidad y consideración
- **Inclusión**: Valoramos la diversidad de perspectivas
- **Colaboración**: Trabajamos juntos hacia objetivos comunes
- **Transparencia**: Comunicamos abierta y honestamente

### Comportamientos Esperados
- Usar lenguaje inclusivo y respetuoso
- Ser receptivo a comentarios constructivos
- Mostrar empatía hacia otros colaboradores
- Centrarse en lo que es mejor para la comunidad

## 🎯 Tipos de Contribución

### 1. 🐛 Reportar Bugs
**Antes de reportar:**
- Busca si el bug ya fue reportado
- Reproduce el problema en la última versión
- Reúne información del sistema y logs

**Template de Bug Report:**
```markdown
**Descripción del Bug**
Descripción clara y concisa del problema.

**Pasos para Reproducir**
1. Ir a '...'
2. Hacer clic en '....'
3. Hacer scroll hasta '....'
4. Ver error

**Comportamiento Esperado**
Descripción clara de lo que esperabas que pasara.

**Screenshots**
Si aplica, añadir screenshots para explicar el problema.

**Información del Sistema:**
- OS: [ej. iOS]
- Browser [ej. chrome, safari]
- Versión [ej. 22]
- n8n Version: [ej. 1.15.0]

**Logs Adicionales**
```

### 2. 💡 Sugerir Mejoras
**Template de Feature Request:**
```markdown
**¿Esta feature request está relacionada a un problema?**
Descripción clara y concisa del problema. Ej. Me frustra cuando [...]

**Describe la solución que te gustaría**
Descripción clara y concisa de lo que quieres que pase.

**Describe alternativas consideradas**
Descripción clara y concisa de cualquier solución alternativa.

**Contexto Adicional**
Añadir cualquier contexto o screenshots sobre la feature request.
```

### 3. 🔧 Contribuir con Código
- Nuevas características
- Corrección de bugs
- Mejoras de performance
- Refactoring de código

### 4. 📚 Mejorar Documentación
- Corregir typos
- Mejorar claridad
- Añadir ejemplos
- Traducir contenido

## ⚙️ Configuración del Entorno

### 1. Fork y Clone
```bash
# Fork el repositorio en GitHub
# Luego clonar tu fork
git clone https://github.com/TU_USERNAME/ami-assistant.git
cd ami-assistant

# Añadir upstream
git remote add upstream https://github.com/habi/ami-assistant.git
```

### 2. Configurar Entorno Local
```bash
# Copiar variables de entorno
cp .env.example .env.local

# Editar con tus configuraciones
nano .env.local

# Iniciar servicios de desarrollo
docker-compose -f docker-compose.dev.yml up -d
```

### 3. Verificar Instalación
```bash
# Test de conectividad
./scripts/test-setup.sh

# Verificar n8n
curl http://localhost:5678/healthz

# Ejecutar tests
npm test
```

## 🔄 Proceso de Desarrollo

### 1. Workflow de Git

```bash
# Siempre trabajar desde main actualizado
git checkout main
git pull upstream main

# Crear rama para tu feature/fix
git checkout -b feature/nueva-funcionalidad
# O para bugs:
git checkout -b fix/descripcion-bug
```

### 2. Convención de Ramas

| Prefijo | Descripción | Ejemplo |
|---------|-------------|---------|
| `feature/` | Nueva funcionalidad | `feature/whatsapp-integration` |
| `fix/` | Corrección de bug | `fix/validation-timeout` |
| `docs/` | Cambios en documentación | `docs/api-examples` |
| `refactor/` | Refactoring de código | `refactor/state-management` |
| `test/` | Añadir o mejorar tests | `test/document-validation` |

### 3. Convención de Commits

Usamos [Conventional Commits](https://www.conventionalcommits.org/):

```bash
# Formato: tipo(scope): descripción
git commit -m "feat(validation): add support for new document type"
git commit -m "fix(webhook): handle timeout errors gracefully"
git commit -m "docs(api): add examples for document upload"
```

**Tipos válidos:**
- `feat`: Nueva funcionalidad
- `fix`: Corrección de bug
- `docs`: Cambios en documentación
- `style`: Cambios de formato (no afectan funcionalidad)
- `refactor`: Refactoring de código
- `test`: Añadir o modificar tests
- `chore`: Tareas de mantenimiento

## 📏 Estándares de Código

### 1. Workflow de n8n

#### Naming Convention
- **Nodos**: PascalCase descriptivo
  - ✅ `ValidateDocument`, `SendNotification`
  - ❌ `node1`, `validate_doc`

- **Variables**: camelCase
  - ✅ `leadData`, `documentType`
  - ❌ `lead_data`, `DocumentType`

#### Estructura de Nodos
```javascript
// Template para nodos de JavaScript
const nodeConfig = {
  name: 'ValidateLeadData',
  type: 'function',
  code: `
    // === CONFIGURACIÓN ===
    const requiredFields = ['full_name', 'email', 'phone_e164'];
    
    // === VALIDACIÓN ===
    const inputData = $input.all();
    const leadData = inputData[0].json;
    
    // === LÓGICA PRINCIPAL ===
    const validationResult = validateFields(leadData, requiredFields);
    
    // === RETORNO ===
    return [{
      json: {
        valid: validationResult.isValid,
        errors: validationResult.errors,
        lead_id: leadData.lead_id
      }
    }];
  `
};
```

### 2. Documentación en Código

```javascript
// === HEADER OBLIGATORIO ===
/**
 * Nodo: ValidateDocument
 * Propósito: Validar documentos usando MCP API
 * Autor: [Tu nombre]
 * Fecha: 2024-01-15
 * Versión: 1.0.0
 */

// === CONFIGURACIÓN ===
const CONFIG = {
  timeout: 30000,
  retries: 3,
  supportedTypes: ['cedula', 'servicio_publico', 'ctl', 'paz_y_salvo_admin']
};

// === FUNCIONES AUXILIARES ===
function validateDocumentType(type) {
  return CONFIG.supportedTypes.includes(type);
}

// === LÓGICA PRINCIPAL ===
// [código del nodo]
```

### 3. Manejo de Errores

```javascript
// Template para manejo de errores
try {
  // Lógica principal
  const result = await processDocument(document);
  
  return [{
    json: {
      success: true,
      data: result
    }
  }];
  
} catch (error) {
  // Log del error
  console.error(`Error en ${node.name}:`, error);
  
  // Retorno de error estructurado
  return [{
    json: {
      success: false,
      error: {
        type: error.type || 'unknown',
        message: error.message,
        code: error.code || 'ERR_UNKNOWN',
        node: node.name,
        timestamp: new Date().toISOString()
      }
    }
  }];
}
```

## 🧪 Testing

### 1. Tests Requeridos

Para cualquier contribución de código:

- [ ] **Unit tests** para funciones individuales
- [ ] **Integration tests** para flujo completo
- [ ] **Tests de validación** para documentos
- [ ] **Tests de error handling**

### 2. Ejecutar Tests

```bash
# Tests completos
npm test

# Tests específicos
npm test -- --grep "document validation"

# Tests con coverage
npm run test:coverage

# Tests de carga
npm run test:load
```

### 3. Template de Test

```javascript
// test/validation.test.js
describe('Document Validation', () => {
  describe('Cedula Validation', () => {
    it('should accept valid cedula document', async () => {
      // Arrange
      const mockDocument = {
        doc_type: 'cedula',
        file: 'base64_content_here'
      };
      
      // Act
      const result = await validateDocument(mockDocument);
      
      // Assert
      expect(result.status).toBe('auto_ok');
      expect(result.confidence).toBeGreaterThan(0.8);
    });
    
    it('should reject invalid format', async () => {
      // Test para formato inválido
    });
  });
});
```

## 📚 Documentación

### 1. Documentación Requerida

Para nuevas features:

- [ ] **README actualizado** si cambia funcionalidad principal
- [ ] **API docs** si añades nuevos endpoints
- [ ] **Workflow docs** si modificas flujo
- [ ] **Ejemplos de uso** para nuevas funcionalidades

### 2. Estilo de Documentación

```markdown
# Título Principal

## Descripción Breve
Una o dos oraciones explicando qué hace.

## Uso Básico
\`\`\`javascript
// Ejemplo simple y claro
const result = await ami.validateDocument(document);
\`\`\`

## Parámetros

| Parámetro | Tipo | Descripción | Requerido |
|-----------|------|-------------|-----------|
| `document` | Object | Documento a validar | ✅ |

## Respuesta
\`\`\`json
{
  "status": "auto_ok",
  "confidence": 0.95
}
\`\`\`

## Ejemplos Avanzados
[Ejemplos más complejos]
```

## 🔍 Pull Requests

### 1. Checklist Pre-PR

Antes de crear tu PR:

- [ ] **Tests pasan**: `npm test`
- [ ] **Linting OK**: `npm run lint`
- [ ] **Documentación actualizada**
- [ ] **Commits bien formateados**
- [ ] **Rama actualizada con main**

### 2. Template de PR

```markdown
## 📋 Descripción
Descripción clara de los cambios realizados.

## 🔗 Issue Relacionado
Closes #123

## 🧪 Tipo de Cambio
- [ ] Bug fix (cambio que corrige un issue)
- [ ] Nueva feature (cambio que añade funcionalidad)
- [ ] Breaking change (fix o feature que causa cambios incompatibles)
- [ ] Documentación

## ✅ Testing
- [ ] Tests unitarios añadidos/actualizados
- [ ] Tests de integración verificados
- [ ] Tests manuales realizados

## 📸 Screenshots (si aplica)
[Añadir screenshots para cambios de UI]

## 📝 Checklist
- [ ] Mi código sigue las convenciones del proyecto
- [ ] He realizado una auto-revisión de mi código
- [ ] He comentado mi código en áreas complejas
- [ ] He actualizado la documentación correspondiente
- [ ] Mis cambios no generan nuevas warnings
- [ ] He añadido tests que prueban mi fix/feature
- [ ] Tests nuevos y existentes pasan localmente
```

### 3. Proceso de Review

1. **Auto-revisión**: Revisa tu propio PR antes de enviarlo
2. **Automated checks**: CI/CD debe pasar
3. **Peer review**: Al menos 1 aprobación requerida
4. **Testing**: Reviewer debe validar funcionalidad
5. **Merge**: Squash and merge preferido

## 🚀 Deployment

### 1. Ambiente de Desarrollo

```bash
# Deploy a desarrollo
git checkout main
git pull upstream main
./scripts/deploy-dev.sh
```

### 2. Ambiente de Staging

```bash
# Deploy a staging (solo maintainers)
git checkout main
git tag v1.x.x
./scripts/deploy-staging.sh
```

### 3. Producción

Solo maintainers del proyecto pueden hacer deploy a producción.

## 🏷️ Release Process

### 1. Versionado

Seguimos [Semantic Versioning](https://semver.org/):

- **MAJOR** (1.0.0): Cambios incompatibles
- **MINOR** (0.1.0): Nueva funcionalidad compatible
- **PATCH** (0.0.1): Bug fixes compatibles

### 2. Changelog

Mantenemos un `CHANGELOG.md` actualizado:

```markdown
## [1.2.0] - 2024-01-15

### Added
- Nueva validación para documentos CTL
- Soporte para WhatsApp webhooks

### Changed
- Mejorado tiempo de respuesta de validación
- Actualizada documentación de API

### Fixed
- Corregido timeout en validación de documentos grandes
- Arreglado manejo de errores en MCP API
```

## 🆘 Obtener Ayuda

### Canales de Comunicación

- **GitHub Issues**: Para bugs y feature requests
- **GitHub Discussions**: Para preguntas generales
- **Email**: tech@habi.co para temas sensibles

### Recursos Útiles

- [Documentación de n8n](https://docs.n8n.io/)
- [API Reference de Ami](docs/api-documentation.md)
- [Workflow Guide](docs/workflow-technical-guide.md)

---

**¡Gracias por contribuir a Ami! 🎉**

Tu contribución ayuda a hacer que el proceso de venta de apartamentos sea más fácil y eficiente para miles de usuarios.
