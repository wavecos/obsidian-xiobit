# Especificación de APIs Requeridas - Portal de Agentes
## BISA Seguros - Salud Internacional

**Documento:** Especificación Técnica de APIs
**Versión:** 1.0
**Fecha:** 26 de Enero, 2026
**Destinatario:** BISA Seguros y Reaseguros S.A.
**Preparado por:** Xiobit Solutions

---

## Tabla de Contenidos

1. [Introducción](#1-introducción)
2. [Consideraciones Generales](#2-consideraciones-generales)
3. [Autenticación y Autorización](#3-autenticación-y-autorización)
4. [APIs del Módulo de Agentes](#4-apis-del-módulo-de-agentes)
5. [APIs del Módulo Backoffice](#5-apis-del-módulo-backoffice)
6. [APIs Compartidas](#6-apis-compartidas)
7. [Modelos de Datos](#7-modelos-de-datos)
8. [Códigos de Respuesta](#8-códigos-de-respuesta)
9. [Consideraciones de Seguridad](#9-consideraciones-de-seguridad)
10. [Anexo: Ejemplos de Request/Response](#10-anexo-ejemplos-de-requestresponse)

---

## 1. Introducción

### 1.1 Propósito del Documento

Este documento especifica las APIs REST necesarias para implementar el **Portal de Agentes/Intermediarios** de BISA Seguros Salud Internacional. El objetivo es que el equipo técnico de BISA pueda validar la disponibilidad de estos servicios o identificar gaps que requieran desarrollo.

### 1.2 Alcance

El documento cubre las APIs necesarias para las siguientes funcionalidades:

- **Must Have (Críticas):** Cartera, Administración de Negocios, Configuración
- **Should Have (Importantes):** Comisiones, Reclamos
- **Could Have (Deseables):** Gráficas, Listas Útiles

### 1.3 Base URL

Todas las APIs seguirán el siguiente patrón:

```
https://{environment}.bisaseguros.com/api/v1/
```

Ejemplos:
- Desarrollo: `https://dev-api.bisaseguros.com/api/v1/`
- Producción: `https://api.bisaseguros.com/api/v1/`

---

## 2. Consideraciones Generales

### 2.1 Protocolo y Formato

- **Protocolo:** HTTPS (TLS 1.2 o superior)
- **Formato de datos:** JSON (application/json)
- **Encoding:** UTF-8
- **Versionado:** En la URL (`/api/v1/`)

### 2.2 Headers Comunes

Todas las peticiones deben incluir:

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {jwt_token}
X-Request-ID: {uuid}
X-Client-Version: {version}
```

### 2.3 Paginación

Para endpoints que retornan listas, usar query parameters:

```
?page=1&size=20&sort=createdDate,desc
```

Respuesta incluye metadata:

```json
{
  "content": [...],
  "page": {
    "number": 1,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  }
}
```

### 2.4 Filtrado

Usar query parameters para filtrado:

```
?status=ACTIVE&startDate=2026-01-01&endDate=2026-12-31
```

---

## 3. Autenticación y Autorización

### 3.1 Login de Agente

**Endpoint:** `POST /auth/agent/login`

**Descripción:** Autenticación de agente con usuario y contraseña.

**Request Body:**
```json
{
  "username": "string",
  "password": "string",
  "deviceId": "string (opcional)",
  "deviceInfo": {
    "platform": "WEB",
    "browser": "Chrome",
    "version": "120.0"
  }
}
```

**Response (200 OK):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1...",
  "refreshToken": "string",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "agentProfile": {
    "agentId": "string",
    "agentCode": "string",
    "fullName": "string",
    "email": "string",
    "role": "AGENT|BROKER|MASTER_AGENT",
    "status": "ACTIVE",
    "permissions": ["VIEW_PORTFOLIO", "CREATE_BUSINESS", ...]
  },
  "requires2FA": false
}
```

### 3.2 Verificación 2FA (si aplica)

**Endpoint:** `POST /auth/agent/verify-2fa`

**Request Body:**
```json
{
  "sessionToken": "string",
  "otpCode": "string"
}
```

**Response (200 OK):**
```json
{
  "accessToken": "string",
  "refreshToken": "string",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

### 3.3 Refresh Token

**Endpoint:** `POST /auth/refresh`

**Request Body:**
```json
{
  "refreshToken": "string"
}
```

**Response (200 OK):**
```json
{
  "accessToken": "string",
  "refreshToken": "string",
  "expiresIn": 3600
}
```

### 3.4 Logout

**Endpoint:** `POST /auth/logout`

**Request Body:**
```json
{
  "refreshToken": "string"
}
```

**Response (204 No Content)**

### 3.5 Cambio de Contraseña

**Endpoint:** `POST /auth/agent/change-password`

**Request Body:**
```json
{
  "currentPassword": "string",
  "newPassword": "string",
  "confirmPassword": "string"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Contraseña actualizada exitosamente"
}
```

---

## 4. APIs del Módulo de Agentes

### 4.1 Gestión de Cartera

#### 4.1.1 Resumen de Cartera (Must Have)

**Endpoint:** `GET /agents/{agentId}/portfolio/summary`

**Descripción:** Obtiene el resumen total de la cartera del agente.

**Path Parameters:**
- `agentId`: ID del agente (string)

**Query Parameters:**
- `startDate`: Fecha inicio (opcional, ISO 8601)
- `endDate`: Fecha fin (opcional, ISO 8601)

**Response (200 OK):**
```json
{
  "agentId": "string",
  "agentCode": "string",
  "summary": {
    "totalPolicies": 150,
    "activePolicies": 120,
    "gracePeriodPolicies": 15,
    "expiredPolicies": 15,
    "totalPremium": {
      "amount": 450000.00,
      "currency": "USD"
    },
    "monthlyPremium": {
      "amount": 37500.00,
      "currency": "USD"
    },
    "yearlyPremium": {
      "amount": 450000.00,
      "currency": "USD"
    }
  },
  "statusBreakdown": [
    {
      "status": "ACTIVE",
      "count": 120,
      "percentage": 80.0,
      "totalPremium": {
        "amount": 380000.00,
        "currency": "USD"
      }
    },
    {
      "status": "GRACE_PERIOD",
      "count": 15,
      "percentage": 10.0,
      "totalPremium": {
        "amount": 50000.00,
        "currency": "USD"
      }
    },
    {
      "status": "EXPIRED",
      "count": 15,
      "percentage": 10.0,
      "totalPremium": {
        "amount": 20000.00,
        "currency": "USD"
      }
    }
  ],
  "periodStart": "2026-01-01T00:00:00Z",
  "periodEnd": "2026-12-31T23:59:59Z",
  "lastUpdated": "2026-01-28T10:30:00Z"
}
```

#### 4.1.2 Detalle de Cartera por Cliente (Must Have)

**Endpoint:** `GET /agents/{agentId}/portfolio/clients`

**Descripción:** Lista detallada de clientes con sus pólizas.

**Path Parameters:**
- `agentId`: ID del agente

**Query Parameters:**
- `page`: Número de página (default: 0)
- `size`: Tamaño de página (default: 20)
- `status`: Filtro por estado (ACTIVE|GRACE_PERIOD|EXPIRED)
- `search`: Búsqueda por nombre/documento
- `sort`: Campo de ordenamiento (default: clientName,asc)

**Response (200 OK):**
```json
{
  "content": [
    {
      "clientId": "string",
      "clientCode": "string",
      "fullName": "Juan Pérez García",
      "documentType": "PASSPORT",
      "documentNumber": "AB123456",
      "email": "juan.perez@email.com",
      "phone": "+591 70123456",
      "policies": [
        {
          "policyId": "string",
          "policyNumber": "POL-2026-00001",
          "productType": "SALUD_INTERNACIONAL",
          "productName": "Plan Premium",
          "status": "ACTIVE",
          "startDate": "2026-01-01",
          "endDate": "2026-12-31",
          "premium": {
            "amount": 2500.00,
            "currency": "USD",
            "frequency": "MONTHLY"
          },
          "coverageAmount": {
            "amount": 500000.00,
            "currency": "USD"
          },
          "nextDueDate": "2026-02-01",
          "daysToExpire": 337
        }
      ],
      "totalPremium": {
        "amount": 2500.00,
        "currency": "USD"
      },
      "activePoliciesCount": 1
    }
  ],
  "page": {
    "number": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  }
}
```

#### 4.1.3 Detalle de Póliza Individual (Must Have)

**Endpoint:** `GET /agents/{agentId}/policies/{policyId}`

**Descripción:** Obtiene el detalle completo de una póliza específica.

**Response (200 OK):**
```json
{
  "policyId": "string",
  "policyNumber": "POL-2026-00001",
  "status": "ACTIVE",
  "client": {
    "clientId": "string",
    "fullName": "Juan Pérez García",
    "documentType": "PASSPORT",
    "documentNumber": "AB123456",
    "email": "juan.perez@email.com",
    "phone": "+591 70123456",
    "birthDate": "1985-05-15",
    "nationality": "Boliviana"
  },
  "product": {
    "productId": "string",
    "productCode": "PREM-INT-001",
    "productName": "Plan Premium Internacional",
    "category": "SALUD_INTERNACIONAL",
    "coverageType": "WORLDWIDE"
  },
  "coverage": {
    "coverageAmount": {
      "amount": 500000.00,
      "currency": "USD"
    },
    "deductible": {
      "amount": 1000.00,
      "currency": "USD"
    },
    "coinsurance": 20.0,
    "maxOutOfPocket": {
      "amount": 5000.00,
      "currency": "USD"
    }
  },
  "premium": {
    "amount": 2500.00,
    "currency": "USD",
    "frequency": "MONTHLY",
    "nextDueDate": "2026-02-01",
    "lastPaymentDate": "2026-01-01"
  },
  "dates": {
    "issueDate": "2025-12-15",
    "startDate": "2026-01-01",
    "endDate": "2026-12-31",
    "daysToExpire": 337
  },
  "beneficiaries": [
    {
      "fullName": "María García",
      "relationship": "SPOUSE",
      "percentage": 50.0
    },
    {
      "fullName": "Carlos Pérez",
      "relationship": "CHILD",
      "percentage": 50.0
    }
  ],
  "agent": {
    "agentId": "string",
    "agentCode": "AGT-001",
    "fullName": "Pedro Rodríguez",
    "email": "pedro.rodriguez@agent.com"
  }
}
```

#### 4.1.4 Gráficas de Producción (Could Have)

**Endpoint:** `GET /agents/{agentId}/portfolio/production-charts`

**Descripción:** Datos para gráficas de producción filtables.

**Query Parameters:**
- `period`: MONTHLY|QUARTERLY|YEARLY
- `startDate`: Fecha inicio
- `endDate`: Fecha fin
- `groupBy`: STATUS|PRODUCT|MONTH

**Response (200 OK):**
```json
{
  "agentId": "string",
  "period": "MONTHLY",
  "charts": {
    "productionByStatus": [
      {
        "status": "ACTIVE",
        "count": 120,
        "premium": {
          "amount": 380000.00,
          "currency": "USD"
        }
      }
    ],
    "productionByProduct": [
      {
        "productName": "Plan Premium",
        "count": 80,
        "premium": {
          "amount": 250000.00,
          "currency": "USD"
        }
      }
    ],
    "monthlyTrend": [
      {
        "month": "2026-01",
        "newPolicies": 15,
        "renewals": 5,
        "cancellations": 2,
        "netGrowth": 13,
        "totalPremium": {
          "amount": 35000.00,
          "currency": "USD"
        }
      }
    ]
  },
  "generatedAt": "2026-01-28T10:30:00Z"
}
```

### 4.2 Administración de Negocios

#### 4.2.1 Crear Solicitud de Nuevo Negocio (Must Have)

**Endpoint:** `POST /agents/{agentId}/business/new-policy-request`

**Descripción:** Crea una solicitud de nueva póliza.

**Request Body:**
```json
{
  "clientInfo": {
    "isExistingClient": false,
    "clientId": "string (si existe)",
    "documentType": "PASSPORT",
    "documentNumber": "AB123456",
    "firstName": "Juan",
    "lastName": "Pérez García",
    "birthDate": "1985-05-15",
    "gender": "M",
    "nationality": "Boliviana",
    "email": "juan.perez@email.com",
    "phone": "+591 70123456",
    "address": {
      "street": "Av. Principal 123",
      "city": "La Paz",
      "state": "La Paz",
      "country": "Bolivia",
      "zipCode": "00000"
    }
  },
  "policyInfo": {
    "productId": "string",
    "coverageAmount": 500000.00,
    "startDate": "2026-02-01",
    "paymentFrequency": "MONTHLY",
    "beneficiaries": [
      {
        "fullName": "María García",
        "documentNumber": "CD789012",
        "relationship": "SPOUSE",
        "percentage": 50.0,
        "birthDate": "1987-08-20"
      }
    ]
  },
  "medicalInfo": {
    "hasPreexistingConditions": false,
    "preexistingConditions": [],
    "isSmoker": false,
    "height": 175,
    "weight": 75
  },
  "attachments": [
    {
      "type": "DOCUMENT_ID",
      "fileName": "passport.pdf",
      "fileUrl": "string",
      "fileSize": 1024000
    }
  ],
  "notes": "Observaciones adicionales"
}
```

**Response (201 Created):**
```json
{
  "requestId": "string",
  "requestNumber": "REQ-2026-00001",
  "status": "PENDING_REVIEW",
  "createdAt": "2026-01-28T10:30:00Z",
  "estimatedProcessingTime": "2-3 días hábiles",
  "nextSteps": [
    "Revisión de documentación",
    "Evaluación médica",
    "Aprobación de suscripción"
  ]
}
```

#### 4.2.2 Listar Solicitudes de Negocio

**Endpoint:** `GET /agents/{agentId}/business/requests`

**Query Parameters:**
- `status`: PENDING_REVIEW|APPROVED|REJECTED|IN_PROGRESS
- `page`, `size`, `sort`

**Response (200 OK):**
```json
{
  "content": [
    {
      "requestId": "string",
      "requestNumber": "REQ-2026-00001",
      "status": "PENDING_REVIEW",
      "clientName": "Juan Pérez García",
      "productName": "Plan Premium Internacional",
      "requestedCoverage": {
        "amount": 500000.00,
        "currency": "USD"
      },
      "estimatedPremium": {
        "amount": 2500.00,
        "currency": "USD",
        "frequency": "MONTHLY"
      },
      "createdAt": "2026-01-28T10:30:00Z",
      "lastUpdated": "2026-01-28T10:30:00Z"
    }
  ],
  "page": {...}
}
```

#### 4.2.3 Detalle de Solicitud

**Endpoint:** `GET /agents/{agentId}/business/requests/{requestId}`

**Response (200 OK):**
```json
{
  "requestId": "string",
  "requestNumber": "REQ-2026-00001",
  "status": "PENDING_REVIEW",
  "clientInfo": {...},
  "policyInfo": {...},
  "medicalInfo": {...},
  "attachments": [...],
  "timeline": [
    {
      "date": "2026-01-28T10:30:00Z",
      "status": "SUBMITTED",
      "description": "Solicitud enviada",
      "performedBy": "Pedro Rodríguez (Agente)"
    }
  ],
  "notes": "string"
}
```

#### 4.2.4 Gestión de Renovaciones (Must Have)

**Endpoint:** `GET /agents/{agentId}/renewals/upcoming`

**Descripción:** Lista de pólizas próximas a vencer.

**Query Parameters:**
- `daysAhead`: Días hacia adelante (default: 60)
- `status`: Filtro por estado
- `page`, `size`, `sort`

**Response (200 OK):**
```json
{
  "content": [
    {
      "policyId": "string",
      "policyNumber": "POL-2025-00100",
      "clientName": "Juan Pérez García",
      "productName": "Plan Premium",
      "expirationDate": "2026-03-31",
      "daysToExpire": 62,
      "currentPremium": {
        "amount": 2500.00,
        "currency": "USD",
        "frequency": "MONTHLY"
      },
      "renewalStatus": "PENDING",
      "notificationsSent": 1,
      "lastNotificationDate": "2026-01-15"
    }
  ],
  "page": {...}
}
```

#### 4.2.5 Enviar Aviso de Renovación (Must Have)

**Endpoint:** `POST /agents/{agentId}/renewals/{policyId}/send-notice`

**Descripción:** Envía aviso de renovación al asegurado.

**Request Body:**
```json
{
  "notificationType": "EMAIL|SMS|BOTH",
  "customMessage": "Estimado cliente, su póliza está próxima a vencer...",
  "includeQuote": true
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "notificationId": "string",
  "sentAt": "2026-01-28T10:30:00Z",
  "recipients": {
    "email": "juan.perez@email.com",
    "phone": "+591 70123456"
  }
}
```

#### 4.2.6 Visualización de Comisiones (Should Have)

**Endpoint:** `GET /agents/{agentId}/commissions`

**Query Parameters:**
- `startDate`: Fecha inicio
- `endDate`: Fecha fin
- `status`: PENDING|PAID|CANCELLED
- `page`, `size`, `sort`

**Response (200 OK):**
```json
{
  "summary": {
    "totalEarned": {
      "amount": 25000.00,
      "currency": "USD"
    },
    "totalPaid": {
      "amount": 20000.00,
      "currency": "USD"
    },
    "totalPending": {
      "amount": 5000.00,
      "currency": "USD"
    }
  },
  "content": [
    {
      "commissionId": "string",
      "policyNumber": "POL-2026-00001",
      "clientName": "Juan Pérez García",
      "productName": "Plan Premium",
      "commissionType": "NEW_BUSINESS|RENEWAL",
      "premium": {
        "amount": 2500.00,
        "currency": "USD"
      },
      "commissionRate": 15.0,
      "commissionAmount": {
        "amount": 375.00,
        "currency": "USD"
      },
      "status": "PAID",
      "earnedDate": "2026-01-01",
      "paidDate": "2026-01-15",
      "paymentReference": "PAY-2026-001"
    }
  ],
  "page": {...}
}
```

#### 4.2.7 Estado de Reclamos Asociados (Should Have)

**Endpoint:** `GET /agents/{agentId}/claims`

**Query Parameters:**
- `status`: SUBMITTED|IN_REVIEW|APPROVED|REJECTED|PAID
- `policyId`: Filtro por póliza
- `clientId`: Filtro por cliente
- `page`, `size`, `sort`

**Response (200 OK):**
```json
{
  "content": [
    {
      "claimId": "string",
      "claimNumber": "CLM-2026-00001",
      "policyNumber": "POL-2026-00001",
      "clientName": "Juan Pérez García",
      "claimType": "MEDICAL_EXPENSE",
      "claimAmount": {
        "amount": 5000.00,
        "currency": "USD"
      },
      "approvedAmount": {
        "amount": 4000.00,
        "currency": "USD"
      },
      "status": "APPROVED",
      "submittedDate": "2026-01-15",
      "lastUpdated": "2026-01-25",
      "description": "Hospitalización por cirugía"
    }
  ],
  "page": {...}
}
```

#### 4.2.8 Listas Útiles (Could Have)

**Endpoint:** `GET /agents/{agentId}/useful-lists/birthdays`

**Descripción:** Lista de cumpleaños de clientes.

**Query Parameters:**
- `month`: Mes específico (1-12)
- `daysAhead`: Próximos N días

**Response (200 OK):**
```json
{
  "content": [
    {
      "clientId": "string",
      "clientName": "Juan Pérez García",
      "birthDate": "1985-05-15",
      "upcomingBirthday": "2026-05-15",
      "age": 41,
      "daysUntilBirthday": 107,
      "email": "juan.perez@email.com",
      "phone": "+591 70123456",
      "activePolicies": 1
    }
  ]
}
```

**Endpoint:** `GET /agents/{agentId}/useful-lists/expiration-alerts`

**Response (200 OK):**
```json
{
  "content": [
    {
      "policyId": "string",
      "policyNumber": "POL-2025-00100",
      "clientName": "María González",
      "expirationDate": "2026-02-28",
      "daysToExpire": 31,
      "alertLevel": "WARNING|CRITICAL",
      "lastContactDate": "2026-01-15",
      "renewalStatus": "PENDING"
    }
  ]
}
```

### 4.3 Configuración y Perfil

#### 4.3.1 Obtener Perfil del Agente (Must Have)

**Endpoint:** `GET /agents/{agentId}/profile`

**Response (200 OK):**
```json
{
  "agentId": "string",
  "agentCode": "AGT-001",
  "personalInfo": {
    "firstName": "Pedro",
    "lastName": "Rodríguez",
    "fullName": "Pedro Rodríguez",
    "documentType": "CI",
    "documentNumber": "1234567",
    "birthDate": "1980-03-20",
    "gender": "M",
    "nationality": "Boliviana"
  },
  "contactInfo": {
    "email": "pedro.rodriguez@agent.com",
    "alternateEmail": "p.rodriguez@personal.com",
    "phone": "+591 70987654",
    "alternatePhone": "+591 77123456",
    "address": {
      "street": "Calle Los Pinos 456",
      "city": "La Paz",
      "state": "La Paz",
      "country": "Bolivia",
      "zipCode": "00000"
    }
  },
  "professionalInfo": {
    "agentType": "BROKER|AGENT|MASTER_AGENT",
    "licenseNumber": "LIC-2020-001",
    "licenseExpiryDate": "2027-12-31",
    "startDate": "2020-01-15",
    "status": "ACTIVE",
    "specializations": ["SALUD_INTERNACIONAL", "VIDA"],
    "languages": ["es", "en"]
  },
  "bankInfo": {
    "bankName": "Banco Nacional",
    "accountType": "SAVINGS",
    "accountNumber": "****5678",
    "currency": "USD"
  },
  "statistics": {
    "totalPolicies": 150,
    "activePolicies": 120,
    "totalPremium": {
      "amount": 450000.00,
      "currency": "USD"
    },
    "lifetimeCommissions": {
      "amount": 75000.00,
      "currency": "USD"
    },
    "joinDate": "2020-01-15",
    "yearsOfService": 6
  }
}
```

#### 4.3.2 Actualizar Perfil (Must Have)

**Endpoint:** `PUT /agents/{agentId}/profile`

**Request Body:**
```json
{
  "contactInfo": {
    "alternateEmail": "p.rodriguez@personal.com",
    "phone": "+591 70987654",
    "alternatePhone": "+591 77123456",
    "address": {
      "street": "Calle Los Pinos 456",
      "city": "La Paz",
      "state": "La Paz",
      "country": "Bolivia",
      "zipCode": "00000"
    }
  },
  "professionalInfo": {
    "specializations": ["SALUD_INTERNACIONAL", "VIDA", "ACCIDENTES"],
    "languages": ["es", "en", "pt"]
  }
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Perfil actualizado exitosamente",
  "updatedAt": "2026-01-28T10:30:00Z"
}
```

---

## 5. APIs del Módulo Backoffice

### 5.1 Gestión de Usuarios

#### 5.1.1 Listar Agentes

**Endpoint:** `GET /backoffice/agents`

**Query Parameters:**
- `status`: ACTIVE|INACTIVE|SUSPENDED
- `agentType`: BROKER|AGENT|MASTER_AGENT
- `search`: Búsqueda por nombre/código
- `page`, `size`, `sort`

**Response (200 OK):**
```json
{
  "content": [
    {
      "agentId": "string",
      "agentCode": "AGT-001",
      "fullName": "Pedro Rodríguez",
      "email": "pedro.rodriguez@agent.com",
      "agentType": "BROKER",
      "status": "ACTIVE",
      "activePolicies": 120,
      "totalPremium": {
        "amount": 450000.00,
        "currency": "USD"
      },
      "joinDate": "2020-01-15",
      "lastLogin": "2026-01-28T09:00:00Z"
    }
  ],
  "page": {...}
}
```

#### 5.1.2 Crear Agente

**Endpoint:** `POST /backoffice/agents`

**Request Body:**
```json
{
  "personalInfo": {
    "firstName": "Carlos",
    "lastName": "Mendoza",
    "documentType": "CI",
    "documentNumber": "9876543",
    "birthDate": "1985-07-10",
    "gender": "M",
    "nationality": "Boliviana"
  },
  "contactInfo": {
    "email": "carlos.mendoza@agent.com",
    "phone": "+591 71234567",
    "address": {...}
  },
  "professionalInfo": {
    "agentType": "AGENT",
    "licenseNumber": "LIC-2026-010",
    "licenseExpiryDate": "2029-12-31",
    "startDate": "2026-02-01",
    "specializations": ["SALUD_INTERNACIONAL"]
  },
  "credentials": {
    "username": "carlos.mendoza",
    "temporaryPassword": "Temp@2026",
    "requirePasswordChange": true
  }
}
```

**Response (201 Created):**
```json
{
  "agentId": "string",
  "agentCode": "AGT-152",
  "message": "Agente creado exitosamente",
  "credentials": {
    "username": "carlos.mendoza",
    "temporaryPassword": "Temp@2026"
  },
  "activationLink": "https://portal.bisaseguros.com/activate?token=..."
}
```

#### 5.1.3 Actualizar Estado de Agente

**Endpoint:** `PATCH /backoffice/agents/{agentId}/status`

**Request Body:**
```json
{
  "status": "ACTIVE|INACTIVE|SUSPENDED",
  "reason": "Suspensión temporal por auditoría",
  "effectiveDate": "2026-02-01"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "agentId": "string",
  "newStatus": "SUSPENDED",
  "effectiveDate": "2026-02-01"
}
```

### 5.2 Gestión de Comisiones

#### 5.2.1 Configurar Comisiones por Agente

**Endpoint:** `POST /backoffice/commissions/config`

**Request Body:**
```json
{
  "agentId": "string",
  "productId": "string (opcional, null = todos los productos)",
  "commissionRules": [
    {
      "type": "NEW_BUSINESS",
      "rate": 15.0,
      "minPremium": 0,
      "maxPremium": null
    },
    {
      "type": "RENEWAL",
      "rate": 10.0,
      "minPremium": 0,
      "maxPremium": null
    }
  ],
  "effectiveDate": "2026-02-01",
  "expiryDate": null
}
```

**Response (200 OK):**
```json
{
  "configId": "string",
  "success": true,
  "message": "Configuración de comisiones actualizada"
}
```

#### 5.2.2 Procesar Pago de Comisiones

**Endpoint:** `POST /backoffice/commissions/payments`

**Request Body:**
```json
{
  "paymentPeriod": {
    "startDate": "2026-01-01",
    "endDate": "2026-01-31"
  },
  "agentIds": ["agent-1", "agent-2", "..."],
  "paymentDate": "2026-02-15",
  "paymentMethod": "BANK_TRANSFER",
  "notes": "Pago comisiones enero 2026"
}
```

**Response (200 OK):**
```json
{
  "batchId": "string",
  "totalAgents": 50,
  "totalAmount": {
    "amount": 125000.00,
    "currency": "USD"
  },
  "payments": [
    {
      "agentId": "string",
      "agentName": "Pedro Rodríguez",
      "amount": {
        "amount": 5000.00,
        "currency": "USD"
      },
      "commissionCount": 25,
      "status": "PROCESSED"
    }
  ],
  "processedAt": "2026-01-28T10:30:00Z"
}
```

### 5.3 Gestión de FAQ y Contenidos

#### 5.3.1 Listar FAQs

**Endpoint:** `GET /backoffice/faqs`

**Query Parameters:**
- `category`: Categoría de FAQ
- `status`: PUBLISHED|DRAFT|ARCHIVED
- `search`: Búsqueda en pregunta/respuesta

**Response (200 OK):**
```json
{
  "content": [
    {
      "faqId": "string",
      "category": "POLICIES",
      "question": "¿Cómo renovar mi póliza?",
      "answer": "Para renovar su póliza...",
      "order": 1,
      "status": "PUBLISHED",
      "views": 150,
      "helpful": 120,
      "notHelpful": 5,
      "createdBy": "admin@bisaseguros.com",
      "createdAt": "2026-01-15T10:00:00Z",
      "lastModified": "2026-01-20T15:30:00Z"
    }
  ],
  "page": {...}
}
```

#### 5.3.2 Crear/Actualizar FAQ

**Endpoint:** `POST /backoffice/faqs`

**Request Body:**
```json
{
  "category": "POLICIES",
  "question": "¿Qué cubre mi seguro internacional?",
  "answer": "Su seguro internacional cubre...",
  "order": 5,
  "status": "PUBLISHED",
  "tags": ["cobertura", "internacional", "beneficios"]
}
```

**Response (201 Created):**
```json
{
  "faqId": "string",
  "message": "FAQ creada exitosamente"
}
```

### 5.4 Reportes y Trazabilidad

#### 5.4.1 Log de Accesos

**Endpoint:** `GET /backoffice/audit/access-logs`

**Query Parameters:**
- `userId`: ID de usuario (agente/cliente)
- `userType`: AGENT|CLIENT|ADMIN
- `action`: LOGIN|LOGOUT|VIEW|CREATE|UPDATE|DELETE
- `resourceType`: POLICY|CLAIM|PROFILE|COMMISSION
- `startDate`, `endDate`
- `page`, `size`, `sort`

**Response (200 OK):**
```json
{
  "content": [
    {
      "logId": "string",
      "timestamp": "2026-01-28T10:30:00Z",
      "userId": "string",
      "userName": "Pedro Rodríguez",
      "userType": "AGENT",
      "action": "VIEW",
      "resourceType": "POLICY",
      "resourceId": "POL-2026-00001",
      "ipAddress": "192.168.1.100",
      "userAgent": "Mozilla/5.0...",
      "status": "SUCCESS",
      "details": {
        "method": "GET",
        "endpoint": "/api/v1/agents/123/policies/456"
      }
    }
  ],
  "page": {...}
}
```

#### 5.4.2 Reporte de Producción

**Endpoint:** `GET /backoffice/reports/production`

**Query Parameters:**
- `startDate`, `endDate`
- `agentId`: Filtro por agente
- `productId`: Filtro por producto
- `groupBy`: AGENT|PRODUCT|MONTH|STATUS

**Response (200 OK):**
```json
{
  "reportPeriod": {
    "startDate": "2026-01-01",
    "endDate": "2026-01-31"
  },
  "summary": {
    "totalPolicies": 250,
    "newBusiness": 180,
    "renewals": 70,
    "totalPremium": {
      "amount": 625000.00,
      "currency": "USD"
    }
  },
  "breakdown": [
    {
      "dimension": "Pedro Rodríguez (AGT-001)",
      "policiesCount": 50,
      "premium": {
        "amount": 125000.00,
        "currency": "USD"
      },
      "newBusiness": 35,
      "renewals": 15
    }
  ],
  "generatedAt": "2026-01-28T10:30:00Z"
}
```

### 5.5 Configuración de Parámetros

#### 5.5.1 Obtener Parámetros del Sistema

**Endpoint:** `GET /backoffice/config/parameters`

**Query Parameters:**
- `category`: COMMISSION|NOTIFICATION|SECURITY|GENERAL

**Response (200 OK):**
```json
{
  "parameters": [
    {
      "key": "commission.default.new_business_rate",
      "value": "15.0",
      "dataType": "DECIMAL",
      "category": "COMMISSION",
      "description": "Tasa de comisión por defecto para nuevos negocios (%)",
      "editable": true,
      "lastModified": "2026-01-15T10:00:00Z",
      "modifiedBy": "admin@bisaseguros.com"
    },
    {
      "key": "notification.renewal.days_ahead",
      "value": "60",
      "dataType": "INTEGER",
      "category": "NOTIFICATION",
      "description": "Días de anticipación para avisos de renovación",
      "editable": true,
      "lastModified": "2025-12-01T10:00:00Z",
      "modifiedBy": "admin@bisaseguros.com"
    }
  ]
}
```

#### 5.5.2 Actualizar Parámetro

**Endpoint:** `PUT /backoffice/config/parameters/{key}`

**Request Body:**
```json
{
  "value": "20.0",
  "reason": "Actualización de política de comisiones 2026"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "parameter": {
    "key": "commission.default.new_business_rate",
    "oldValue": "15.0",
    "newValue": "20.0",
    "updatedAt": "2026-01-28T10:30:00Z",
    "updatedBy": "admin@bisaseguros.com"
  }
}
```

---

## 6. APIs Compartidas

### 6.1 Catálogos y Referencias

#### 6.1.1 Productos Disponibles

**Endpoint:** `GET /catalog/products`

**Query Parameters:**
- `category`: SALUD_INTERNACIONAL|VIDA|ACCIDENTES
- `status`: ACTIVE|INACTIVE

**Response (200 OK):**
```json
{
  "products": [
    {
      "productId": "string",
      "productCode": "PREM-INT-001",
      "productName": "Plan Premium Internacional",
      "category": "SALUD_INTERNACIONAL",
      "description": "Cobertura médica internacional premium",
      "coverageType": "WORLDWIDE",
      "minCoverageAmount": {
        "amount": 100000.00,
        "currency": "USD"
      },
      "maxCoverageAmount": {
        "amount": 1000000.00,
        "currency": "USD"
      },
      "features": [
        "Cobertura mundial",
        "Atención 24/7",
        "Telemedicina incluida"
      ],
      "status": "ACTIVE"
    }
  ]
}
```

#### 6.1.2 Centros Médicos

**Endpoint:** `GET /catalog/medical-centers`

**Query Parameters:**
- `network`: INTERNATIONAL|LOCAL
- `country`: Código ISO del país
- `city`: Ciudad
- `speciality`: Especialidad médica
- `search`: Búsqueda por nombre

**Response (200 OK):**
```json
{
  "centers": [
    {
      "centerId": "string",
      "name": "Hospital Internacional Premium",
      "network": "INTERNATIONAL",
      "address": {
        "street": "5th Avenue 100",
        "city": "Miami",
        "state": "Florida",
        "country": "USA",
        "zipCode": "33101",
        "coordinates": {
          "latitude": 25.7617,
          "longitude": -80.1918
        }
      },
      "contact": {
        "phone": "+1 305 123 4567",
        "email": "info@hospitalpremium.com",
        "website": "https://hospitalpremium.com"
      },
      "specialities": [
        "CARDIOLOGY",
        "ONCOLOGY",
        "SURGERY"
      ],
      "services": [
        "EMERGENCY_24_7",
        "HOSPITALIZATION",
        "OUTPATIENT"
      ],
      "accreditation": "JCI",
      "acceptsDirectBilling": true
    }
  ]
}
```

### 6.2 Notificaciones

#### 6.2.1 Enviar Notificación

**Endpoint:** `POST /notifications/send`

**Request Body:**
```json
{
  "recipientType": "AGENT|CLIENT",
  "recipientId": "string",
  "notificationType": "EMAIL|SMS|PUSH|IN_APP",
  "template": "RENEWAL_REMINDER",
  "parameters": {
    "clientName": "Juan Pérez",
    "policyNumber": "POL-2026-00001",
    "expirationDate": "2026-03-31"
  },
  "priority": "HIGH|NORMAL|LOW",
  "scheduledFor": "2026-02-01T09:00:00Z"
}
```

**Response (200 OK):**
```json
{
  "notificationId": "string",
  "status": "SENT|SCHEDULED",
  "sentAt": "2026-01-28T10:30:00Z"
}
```

### 6.3 Documentos

#### 6.3.1 Subir Documento

**Endpoint:** `POST /documents/upload`

**Headers:**
```
Content-Type: multipart/form-data
```

**Form Data:**
- `file`: Archivo binario
- `documentType`: POLICY_DOCUMENT|ID_DOCUMENT|MEDICAL_REPORT|PAYMENT_PROOF
- `referenceType`: POLICY|CLAIM|AGENT|CLIENT
- `referenceId`: ID de referencia
- `description`: Descripción (opcional)

**Response (201 Created):**
```json
{
  "documentId": "string",
  "fileName": "passport.pdf",
  "fileSize": 1024000,
  "mimeType": "application/pdf",
  "documentType": "ID_DOCUMENT",
  "uploadedAt": "2026-01-28T10:30:00Z",
  "uploadedBy": "string",
  "url": "https://cdn.bisaseguros.com/documents/...",
  "expiresAt": "2026-02-28T10:30:00Z"
}
```

#### 6.3.2 Descargar Documento

**Endpoint:** `GET /documents/{documentId}/download`

**Response (200 OK):**
- Binary file stream con headers apropiados

#### 6.3.3 Listar Documentos

**Endpoint:** `GET /documents`

**Query Parameters:**
- `referenceType`: POLICY|CLAIM|AGENT
- `referenceId`: ID de referencia
- `documentType`: Tipo de documento

**Response (200 OK):**
```json
{
  "documents": [
    {
      "documentId": "string",
      "fileName": "condicionado_general.pdf",
      "fileSize": 2048000,
      "documentType": "POLICY_DOCUMENT",
      "uploadedAt": "2026-01-15T10:00:00Z",
      "uploadedBy": "system",
      "downloadUrl": "https://cdn.bisaseguros.com/documents/..."
    }
  ]
}
```

---

## 7. Modelos de Datos

### 7.1 Enumeraciones Comunes

```typescript
// Estados de Póliza
enum PolicyStatus {
  ACTIVE = "ACTIVE",
  GRACE_PERIOD = "GRACE_PERIOD",
  EXPIRED = "EXPIRED",
  CANCELLED = "CANCELLED",
  SUSPENDED = "SUSPENDED"
}

// Tipos de Agente
enum AgentType {
  AGENT = "AGENT",
  BROKER = "BROKER",
  MASTER_AGENT = "MASTER_AGENT"
}

// Estados de Agente
enum AgentStatus {
  ACTIVE = "ACTIVE",
  INACTIVE = "INACTIVE",
  SUSPENDED = "SUSPENDED"
}

// Tipos de Comisión
enum CommissionType {
  NEW_BUSINESS = "NEW_BUSINESS",
  RENEWAL = "RENEWAL",
  UPSELL = "UPSELL"
}

// Estados de Comisión
enum CommissionStatus {
  PENDING = "PENDING",
  APPROVED = "APPROVED",
  PAID = "PAID",
  CANCELLED = "CANCELLED"
}

// Estados de Reclamo
enum ClaimStatus {
  SUBMITTED = "SUBMITTED",
  IN_REVIEW = "IN_REVIEW",
  APPROVED = "APPROVED",
  REJECTED = "REJECTED",
  PAID = "PAID"
}

// Frecuencia de Pago
enum PaymentFrequency {
  MONTHLY = "MONTHLY",
  QUARTERLY = "QUARTERLY",
  SEMI_ANNUAL = "SEMI_ANNUAL",
  ANNUAL = "ANNUAL"
}

// Tipos de Documento
enum DocumentType {
  PASSPORT = "PASSPORT",
  CI = "CI",
  RUC = "RUC",
  DRIVER_LICENSE = "DRIVER_LICENSE"
}
```

### 7.2 Estructura de Montos

```typescript
interface Money {
  amount: number;      // Monto numérico
  currency: string;    // Código ISO 4217 (USD, BOB, etc.)
}
```

### 7.3 Estructura de Dirección

```typescript
interface Address {
  street: string;
  city: string;
  state: string;
  country: string;     // Código ISO 3166-1
  zipCode: string;
  coordinates?: {
    latitude: number;
    longitude: number;
  };
}
```

---

## 8. Códigos de Respuesta

### 8.1 Códigos HTTP Estándar

| Código | Descripción | Uso |
|--------|-------------|-----|
| 200 | OK | Operación exitosa (GET, PUT, PATCH) |
| 201 | Created | Recurso creado exitosamente (POST) |
| 204 | No Content | Operación exitosa sin contenido (DELETE) |
| 400 | Bad Request | Request inválido (validación fallida) |
| 401 | Unauthorized | No autenticado |
| 403 | Forbidden | No autorizado (sin permisos) |
| 404 | Not Found | Recurso no encontrado |
| 409 | Conflict | Conflicto (ej. duplicado) |
| 422 | Unprocessable Entity | Error de negocio |
| 429 | Too Many Requests | Rate limit excedido |
| 500 | Internal Server Error | Error del servidor |
| 503 | Service Unavailable | Servicio temporalmente no disponible |

### 8.2 Estructura de Respuestas de Error

```json
{
  "timestamp": "2026-01-28T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validación fallida",
  "errors": [
    {
      "field": "email",
      "message": "Email inválido",
      "code": "INVALID_FORMAT"
    }
  ],
  "path": "/api/v1/agents/123/profile",
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

### 8.3 Códigos de Error de Negocio

```typescript
enum BusinessErrorCode {
  // Agente
  AGENT_NOT_FOUND = "AGENT_NOT_FOUND",
  AGENT_INACTIVE = "AGENT_INACTIVE",
  AGENT_SUSPENDED = "AGENT_SUSPENDED",

  // Póliza
  POLICY_NOT_FOUND = "POLICY_NOT_FOUND",
  POLICY_EXPIRED = "POLICY_EXPIRED",
  POLICY_NOT_OWNED = "POLICY_NOT_OWNED",

  // Comisiones
  COMMISSION_ALREADY_PAID = "COMMISSION_ALREADY_PAID",
  COMMISSION_CALCULATION_ERROR = "COMMISSION_CALCULATION_ERROR",

  // Autenticación
  INVALID_CREDENTIALS = "INVALID_CREDENTIALS",
  TOKEN_EXPIRED = "TOKEN_EXPIRED",
  INSUFFICIENT_PERMISSIONS = "INSUFFICIENT_PERMISSIONS",

  // Validación
  INVALID_INPUT = "INVALID_INPUT",
  MISSING_REQUIRED_FIELD = "MISSING_REQUIRED_FIELD",
  DUPLICATE_ENTRY = "DUPLICATE_ENTRY"
}
```

---

## 9. Consideraciones de Seguridad

### 9.1 Autenticación

- **JWT (JSON Web Tokens)** para autenticación stateless
- Tokens de acceso de corta duración (1 hora)
- Refresh tokens de larga duración (30 días)
- 2FA opcional para agentes con permisos elevados

### 9.2 Autorización

- **RBAC (Role-Based Access Control)** basado en roles y permisos
- Validación de permisos en cada endpoint
- Los agentes solo pueden acceder a datos de sus propios clientes
- Backoffice tiene acceso administrativo completo

### 9.3 Protección de Datos

- **HTTPS obligatorio** en todos los endpoints
- **Encriptación de datos sensibles** (contraseñas, datos bancarios)
- **Ofuscación de datos** en logs (no registrar datos sensibles)
- **PII (Personal Identifiable Information)** protegida según normativas

### 9.4 Rate Limiting

Límites sugeridos por tipo de usuario:

| Tipo de Usuario | Requests por minuto |
|-----------------|---------------------|
| Agente | 100 |
| Backoffice | 500 |
| Servicio Interno | Sin límite |

### 9.5 Auditoría

- **Log completo** de todas las operaciones de escritura (CREATE, UPDATE, DELETE)
- **Log de acceso** a datos sensibles (pólizas, comisiones)
- **Retención de logs** por al menos 2 años
- **Trazabilidad** de cambios con usuario, fecha y hora

### 9.6 CORS (Cross-Origin Resource Sharing)

Configuración sugerida:

```
Access-Control-Allow-Origin: https://portal.bisaseguros.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

---

## 10. Anexo: Ejemplos de Request/Response

### Ejemplo 1: Login y Consulta de Cartera

#### Request: Login
```http
POST /api/v1/auth/agent/login HTTP/1.1
Host: api.bisaseguros.com
Content-Type: application/json

{
  "username": "pedro.rodriguez",
  "password": "SecureP@ss2026"
}
```

#### Response: Login
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "550e8400-e29b-41d4-a716-446655440000",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "agentProfile": {
    "agentId": "agt-12345",
    "agentCode": "AGT-001",
    "fullName": "Pedro Rodríguez",
    "email": "pedro.rodriguez@agent.com",
    "role": "BROKER",
    "status": "ACTIVE",
    "permissions": [
      "VIEW_PORTFOLIO",
      "CREATE_BUSINESS",
      "MANAGE_RENEWALS"
    ]
  },
  "requires2FA": false
}
```

#### Request: Consulta de Cartera
```http
GET /api/v1/agents/agt-12345/portfolio/summary HTTP/1.1
Host: api.bisaseguros.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
```

#### Response: Resumen de Cartera
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "agentId": "agt-12345",
  "agentCode": "AGT-001",
  "summary": {
    "totalPolicies": 150,
    "activePolicies": 120,
    "gracePeriodPolicies": 15,
    "expiredPolicies": 15,
    "totalPremium": {
      "amount": 450000.00,
      "currency": "USD"
    }
  },
  "lastUpdated": "2026-01-28T10:30:00Z"
}
```

### Ejemplo 2: Crear Solicitud de Nueva Póliza

#### Request
```http
POST /api/v1/agents/agt-12345/business/new-policy-request HTTP/1.1
Host: api.bisaseguros.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "clientInfo": {
    "isExistingClient": false,
    "documentType": "PASSPORT",
    "documentNumber": "AB123456",
    "firstName": "Juan",
    "lastName": "Pérez García",
    "birthDate": "1985-05-15",
    "email": "juan.perez@email.com",
    "phone": "+591 70123456"
  },
  "policyInfo": {
    "productId": "prod-premium-001",
    "coverageAmount": 500000.00,
    "startDate": "2026-02-01",
    "paymentFrequency": "MONTHLY"
  }
}
```

#### Response
```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/agents/agt-12345/business/requests/req-789

{
  "requestId": "req-789",
  "requestNumber": "REQ-2026-00001",
  "status": "PENDING_REVIEW",
  "createdAt": "2026-01-28T10:30:00Z",
  "estimatedProcessingTime": "2-3 días hábiles"
}
```

### Ejemplo 3: Error de Validación

#### Request
```http
POST /api/v1/agents/agt-12345/profile HTTP/1.1
Host: api.bisaseguros.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "contactInfo": {
    "email": "invalid-email",
    "phone": "123"
  }
}
```

#### Response
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "timestamp": "2026-01-28T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validación fallida",
  "errors": [
    {
      "field": "contactInfo.email",
      "message": "Formato de email inválido",
      "code": "INVALID_FORMAT",
      "rejectedValue": "invalid-email"
    },
    {
      "field": "contactInfo.phone",
      "message": "Número de teléfono debe tener al menos 8 dígitos",
      "code": "INVALID_LENGTH",
      "rejectedValue": "123"
    }
  ],
  "path": "/api/v1/agents/agt-12345/profile",
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

## Resumen de Endpoints por Módulo

### Módulo de Agentes
Total de endpoints: **25**

| Categoría | Endpoints | Prioridad |
|-----------|-----------|-----------|
| Autenticación | 5 | Must Have |
| Cartera | 4 | Must Have |
| Negocios | 8 | Must Have / Should Have |
| Perfil | 2 | Must Have |
| Listas Útiles | 2 | Could Have |
| Comisiones | 2 | Should Have |
| Reclamos | 2 | Should Have |

