# API Documentation - Ami Assistant

## 📋 Índice
- [Introducción](#introducción)
- [Autenticación](#autenticación)
- [Endpoints Principales](#endpoints-principales)
- [Herramientas MCP](#herramientas-mcp)
- [Webhook Configuration](#webhook-configuration)
- [Schemas de Datos](#schemas-de-datos)
- [Códigos de Error](#códigos-de-error)
- [Ejemplos de Uso](#ejemplos-de-uso)

## 🚀 Introducción

La API de Ami proporciona endpoints para la integración con el asistente de recolección de documentos. El sistema está construido sobre n8n y utiliza herramientas MCP (Model Context Protocol) para la validación de documentos.

### URL Base
```
https://api.habi.co/ami/v1
```

### Formatos Soportados
- **Request**: `application/json`, `multipart/form-data`
- **Response**: `application/json`

## 🔐 Autenticación

### API Key Authentication
```http
Authorization: Bearer <api_key>
```

### Webhook Signature Verification
```javascript
// Verificación de signature en webhooks
const crypto = require('crypto');

function verifyWebhookSignature(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  return signature === `sha256=${expectedSignature}`;
}
```

## 🌐 Endpoints Principales

### 1. Iniciar Conversación

**POST** `/conversation/start`

Inicia una nueva conversación con Ami.

#### Request
```json
{
  "user_id": "string",
  "channel": "whatsapp|web|telegram",
  "initial_message": "string (opcional)"
}
```

#### Response
```json
{
  "lead_id": "uuid",
  "conversation_id": "uuid", 
  "status": "started",
  "initial_response": {
    "message": "¡Hola! Soy Ami 👋, tu asistente de Habi...",
    "next_action": "collect_basic_data"
  }
}
```

### 2. Enviar Mensaje

**POST** `/conversation/{lead_id}/message`

Envía un mensaje a la conversación existente.

#### Request
```json
{
  "message": "string",
  "message_type": "text|file",
  "file_data": {
    "filename": "string",
    "content": "base64_encoded_content",
    "mime_type": "string"
  }
}
```

#### Response
```json
{
  "lead_id": "uuid",
  "ami_response": {
    "message": "string",
    "next_action": "string",
    "progress": {
      "current_step": "doc_cedula",
      "completed": 1,
      "total": 4,
      "percentage": 25
    }
  },
  "validation_required": boolean
}
```

### 3. Obtener Estado del Lead

**GET** `/lead/{lead_id}/status`

Obtiene el estado actual de un lead específico.

#### Response
```json
{
  "lead_id": "uuid",
  "status": "doc_cedula",
  "basic_data": {
    "full_name": "string",
    "phone_e164": "string", 
    "email": "string",
    "address": "string"
  },
  "documents_status": {
    "cedula": "pending|ok|needs_review|rejected",
    "servicio_publico": "pending|ok|needs_review|rejected",
    "ctl": "pending|ok|needs_review|rejected",
    "paz_y_salvo_admin": "pending|ok|needs_review|rejected"
  },
  "progress": {
    "percentage": 25,
    "completed_documents": 1,
    "total_documents": 4
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:35:00Z"
}
```

### 4. Actualizar Lead Data

**PUT** `/lead/{lead_id}`

Actualiza los datos de un lead existente.

#### Request
```json
{
  "full_name": "string (opcional)",
  "phone_e164": "string (opcional)",
  "email": "string (opcional)",
  "address": "string (opcional)",
  "status": "string (opcional)",
  "documents_status": "object (opcional)"
}
```

### 5. Subir Documento

**POST** `/lead/{lead_id}/document`

Sube un documento para validación.

#### Request (multipart/form-data)
```
doc_type: "cedula|servicio_publico|ctl|paz_y_salvo_admin"
file: [archivo binario]
```

#### Response
```json
{
  "document_id": "uuid",
  "doc_type": "cedula",
  "validation_status": "pending",
  "message": "Documento recibido, validando..."
}
```

## 🛠️ Herramientas MCP

### 1. Validate Lead Fields

Valida los datos básicos del lead.

#### Endpoint
**POST** `/mcp/validate_lead_fields`

#### Request
```json
{
  "lead_id": "uuid",
  "fields": {
    "full_name": "string",
    "phone_e164": "string",
    "email": "string", 
    "address": "string"
  }
}
```

#### Response
```json
{
  "valid": boolean,
  "errors": [
    {
      "field": "email",
      "message": "Formato de email inválido",
      "code": "INVALID_FORMAT"
    }
  ],
  "suggestions": {
    "phone_e164": "+57 300 123 4567"
  }
}
```

### 2. Validate Document

Valida un documento subido usando IA.

#### Endpoint
**POST** `/mcp/validate_doc`

#### Request
```json
{
  "doc_type": "cedula|servicio_publico|ctl|paz_y_salvo_admin",
  "file": "base64_encoded_content",
  "file_name": "string",
  "mime_type": "string",
  "lead_id": "uuid"
}
```

#### Response
```json
{
  "status": "auto_ok|needs_review|rejected",
  "confidence": 0.95,
  "message": "Documento válido y legible",
  "details": {
    "extracted_data": {
      "document_number": "12345678",
      "expiry_date": "2030-12-31",
      "name": "Juan Pérez"
    },
    "validation_checks": [
      {
        "check": "readable_text",
        "passed": true
      },
      {
        "check": "valid_format", 
        "passed": true
      }
    ]
  },
  "issues": [
    {
      "type": "warning",
      "message": "Imagen ligeramente borrosa en esquina inferior"
    }
  ]
}
```

### 3. Save Data Lead

Persiste los datos validados del lead.

#### Endpoint  
**POST** `/mcp/save_data_lead`

#### Request
```json
{
  "lead_id": "uuid",
  "full_name": "string",
  "phone_e164": "string",
  "email": "string",
  "address": "string", 
  "status": "string",
  "documents_status": {
    "cedula": "ok|needs_review|rejected|pending",
    "servicio_publico": "ok|needs_review|rejected|pending",
    "ctl": "ok|needs_review|rejected|pending",
    "paz_y_salvo_admin": "ok|needs_review|rejected|pending"
  },
  "completion_timestamp": "string (opcional)",
  "ready_for_human_review": boolean
}
```

#### Response
```json
{
  "success": true,
  "lead_id": "uuid",
  "updated_fields": ["status", "documents_status"],
  "timestamp": "2024-01-15T10:35:00Z"
}
```

## 🔗 Webhook Configuration

### Webhook de Entrada

Configura tu sistema para recibir notificaciones de Ami.

#### URL del Webhook
```
POST https://your-system.com/webhooks/ami
```

#### Headers
```
Content-Type: application/json
X-Ami-Signature: sha256=<signature>
X-Ami-Event: <event_type>
```

#### Payload
```json
{
  "event": "document_validated|lead_completed|validation_failed",
  "lead_id": "uuid",
  "timestamp": "2024-01-15T10:35:00Z",
  "data": {
    "document_type": "cedula",
    "validation_result": "ok",
    "confidence": 0.95
  }
}
```

### Eventos de Webhook

| Evento | Descripción | Datos Incluidos |
|--------|-------------|-----------------|
| `document_validated` | Documento validado exitosamente | `doc_type`, `validation_result` |
| `validation_failed` | Validación de documento falló | `doc_type`, `error_reason` |
| `lead_completed` | Lead completó todos los documentos | `completion_time`, `all_documents` |
| `state_changed` | Estado del lead cambió | `old_state`, `new_state` |

## 📊 Schemas de Datos

### Lead Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "lead_id": {
      "type": "string",
      "format": "uuid"
    },
    "full_name": {
      "type": "string",
      "minLength": 2,
      "maxLength": 100
    },
    "phone_e164": {
      "type": "string",
      "pattern": "^\\+[1-9]\\d{1,14}$"
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "address": {
      "type": "string",
      "minLength": 10,
      "maxLength": 500
    },
    "status": {
      "type": "string",
      "enum": ["start", "lead_basic", "doc_cedula", "doc_servicio", "doc_ctl", "doc_pazsalvo", "review", "done"]
    }
  },
  "required": ["lead_id", "status"]
}
```

### Document Schema
```json
{
  "type": "object",
  "properties": {
    "doc_type": {
      "type": "string",
      "enum": ["cedula", "servicio_publico", "ctl", "paz_y_salvo_admin"]
    },
    "file": {
      "type": "string",
      "contentEncoding": "base64"
    },
    "validation_status": {
      "type": "string", 
      "enum": ["pending", "ok", "needs_review", "rejected"]
    }
  },
  "required": ["doc_type", "file"]
}
```

## ❌ Códigos de Error

### Errores HTTP Standard

| Código | Descripción | Ejemplo |
|--------|-------------|---------|
| `400` | Bad Request | Datos inválidos en request |
| `401` | Unauthorized | API key inválida |
| `404` | Not Found | Lead no encontrado |
| `429` | Rate Limited | Demasiadas requests |
| `500` | Internal Error | Error del servidor |

### Errores Específicos de Ami

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Document validation failed",
    "details": {
      "doc_type": "cedula",
      "issues": ["unreadable_text", "invalid_format"]
    },
    "retry_allowed": true,
    "max_retries": 3
  }
}
```

#### Códigos de Error Custom

| Código | Descripción | Acción Recomendada |
|--------|-------------|--------------------|
| `INVALID_DOC_TYPE` | Tipo de documento no soportado | Verificar tipos válidos |
| `FILE_TOO_LARGE` | Archivo excede límite de tamaño | Comprimir archivo |
| `UNSUPPORTED_FORMAT` | Formato de archivo no soportado | Convertir a PDF/JPG |
| `LEAD_NOT_FOUND` | Lead ID no existe | Verificar ID o crear nuevo lead |
| `INVALID_STATE` | Operación no válida para estado actual | Verificar flujo de estados |
| `RATE_LIMIT_EXCEEDED` | Límite de requests excedido | Esperar antes de retry |
| `MCP_TIMEOUT` | Timeout en validación MCP | Reintentar request |

## 💡 Ejemplos de Uso

### Flujo Completo - JavaScript

```javascript
class AmiClient {
  constructor(apiKey, baseURL) {
    this.apiKey = apiKey;
    this.baseURL = baseURL;
  }

  async startConversation(userId) {
    const response = await fetch(`${this.baseURL}/conversation/start`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ user_id: userId, channel: 'web' })
    });
    
    return response.json();
  }

  async sendMessage(leadId, message) {
    const response = await fetch(`${this.baseURL}/conversation/${leadId}/message`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ message })
    });
    
    return response.json();
  }

  async uploadDocument(leadId, docType, file) {
    const formData = new FormData();
    formData.append('doc_type', docType);
    formData.append('file', file);

    const response = await fetch(`${this.baseURL}/lead/${leadId}/document`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      },
      body: formData
    });
    
    return response.json();
  }
}

// Uso del cliente
const ami = new AmiClient('your-api-key', 'https://api.habi.co/ami/v1');

// Iniciar conversación
const conversation = await ami.startConversation('user-123');
console.log('Lead ID:', conversation.lead_id);

// Enviar datos básicos
await ami.sendMessage(conversation.lead_id, 'Juan Pérez');
await ami.sendMessage(conversation.lead_id, '+57 300 123 4567');
await ami.sendMessage(conversation.lead_id, 'juan@email.com');

// Subir documento
const file = document.getElementById('file-input').files[0];
const result = await ami.uploadDocument(conversation.lead_id, 'cedula', file);
```

### Validación de Webhook - Node.js

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();

app.post('/webhooks/ami', express.raw({type: 'application/json'}), (req, res) => {
  const signature = req.headers['x-ami-signature'];
  const payload = req.body;
  
  // Verificar signature
  const expectedSignature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');
  
  if (signature !== `sha256=${expectedSignature}`) {
    return res.status(401).send('Invalid signature');
  }
  
  const event = JSON.parse(payload);
  
  // Procesar evento
  switch(event.event) {
    case 'document_validated':
      console.log(`Documento ${event.data.doc_type} validado para lead ${event.lead_id}`);
      break;
    case 'lead_completed':
      console.log(`Lead ${event.lead_id} completó todos los documentos`);
      // Notificar a equipo de ventas
      break;
  }
  
  res.status(200).send('OK');
});
```

### Integración con React

```jsx
import React, { useState, useEffect } from 'react';

function AmiChat({ userId }) {
  const [conversation, setConversation] = useState(null);
  const [messages, setMessages] = useState([]);
  const [inputMessage, setInputMessage] = useState('');

  useEffect(() => {
    // Iniciar conversación al montar componente
    startConversation();
  }, []);

  const startConversation = async () => {
    try {
      const response = await fetch('/api/ami/start', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      
      const data = await response.json();
      setConversation(data);
      setMessages([{
        type: 'ami',
        content: data.initial_response.message
      }]);
    } catch (error) {
      console.error('Error starting conversation:', error);
    }
  };

  const sendMessage = async () => {
    if (!inputMessage.trim() || !conversation) return;
    
    // Agregar mensaje del usuario
    setMessages(prev => [...prev, { type: 'user', content: inputMessage }]);
    
    try {
      const response = await fetch(`/api/ami/message/${conversation.lead_id}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: inputMessage })
      });
      
      const data = await response.json();
      
      // Agregar respuesta de Ami
      setMessages(prev => [...prev, {
        type: 'ami',
        content: data.ami_response.message
      }]);
      
      setInputMessage('');
    } catch (error) {
      console.error('Error sending message:', error);
    }
  };

  const handleFileUpload = async (file, docType) => {
    const formData = new FormData();
    formData.append('doc_type', docType);
    formData.append('file', file);

    try {
      const response = await fetch(`/api/ami/document/${conversation.lead_id}`, {
        method: 'POST',
        body: formData
      });
      
      const result = await response.json();
      
      setMessages(prev => [...prev, {
        type: 'user',
        content: `📄 Documento subido: ${file.name}`
      }, {
        type: 'ami', 
        content: result.message
      }]);
    } catch (error) {
      console.error('Error uploading document:', error);
    }
  };

  return (
    <div className="ami-chat">
      <div className="messages">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.type}`}>
            {msg.content}
          </div>
        ))}
      </div>
      
      <div className="input-area">
        <input
          value={inputMessage}
          onChange={(e) => setInputMessage(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="Escribe tu mensaje..."
        />
        <button onClick={sendMessage}>Enviar</button>
      </div>
    </div>
  );
}
```

---

**📧 Para obtener tu API key o soporte técnico, contacta al equipo de desarrollo de Habi.**
