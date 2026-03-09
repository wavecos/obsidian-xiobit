# Especificación de APIs Requeridas - Portal de Agentes
## BISA Seguros - Salud Internacional

**Documento:** Especificación Técnica de APIs
**Versión:** 2.0
**Fecha:** 3 de Febrero, 2026
**Destinatario:** BISA Seguros y Reaseguros S.A.
**Preparado por:** Xiobit Solutions

---

## Tabla de Contenidos

1. [Introducción](#1-introducción)
2. [Autenticación y Autorización](#2-autenticación-y-autorización)
3. [APIs del Módulo de Agentes](#3-apis-del-módulo-de-agentes)
4. [Resumen de Endpoints](#4-resumen-de-endpoints)

---

## 1. Introducción

### 1.1 Propósito del Documento

Este documento especifica las APIs REST necesarias para implementar el **Portal de Agentes/Intermediarios** de BISA Seguros Salud Internacional. El objetivo es que el equipo técnico de BISA pueda validar la disponibilidad de estos servicios o identificar gaps que requieran desarrollo.

**Cambios en esta versión:**
- Actualización del método de autenticación según especificaciones de BISA
- Simplificación de endpoints basado en servicios existentes
- Enfoque en funcionalidades core del portal de agentes

---

## 2. Autenticación y Autorización

### 2.1 Login de Agente

**Endpoint:** `GET http://dev.graphapibisaseg.com:8081/security/auth/authenticateBasic`

**Método:** GET

**Tipo de Autenticación:** Basic Authentication

**Headers:**
```http
Authorization: Basic {base64(username:password)}
```

**Descripción:** Autenticación de agente con usuario y contraseña usando Basic Auth.

**Ejemplo de credenciales:**
- USERNAME: `paovando`
- PASSWORD: `abc.123`

**Response (200 OK):**
```json
{
  "id": 47062121643429,
  "userCode": "paovando",
  "firstName": "Patricia",
  "lastName": "Ovando Peres",
  "description": "",
  "companyId": 2008051211300407600,
  "companynodeId": 2008051214222701000,
  "thirdPartyCode": "97891",
  "emailCorp": "paovando@grupobisa.com",
  "agent": "",
  "token": "eyJhbGciOiJIUzUxMiJ9.eyJkYXRhIjoie1wibGFzdE5hbWVcIjpcIk92YW5kbyBIZXJyZXJhXCIsXCJhZ2VudFwiOlwiXCIsXCJkZXNjcmlwdGlvblwiOlwiXCIsXCJ1c2VyQ29kZVwiOlwicGFvdmFuZG9cIixcImNvbXBhbnlub2RlSWRcIjoyMDA4MDUxMjE0MjIyNzAwOTYwLFwiZmlyc3ROYW1lXCI6XCJQYXRyaWNpYVwiLFwiY29tcGFueUlkXCI6MjAwODA1MTIxMTMwMDQwNzQ3MCxcIm1hY0FkZHJlc3NcIjpudWxsLFwiY2xpZW50SXBcIjpcIjE5Mi4xNjguMjUuNVwiLFwidGhpcmRQYXJ0eUNvZGVcIjpcIjk3ODkxXCIsXCJpZFwiOjQ3MDYyMTIxNjQzNDI5LFwiZW1haWxDb3JwXCI6XCJwYW92YW5kb0BncnVwb2Jpc2EuY29tXCJ9IiwidG9rZW5fZXhwaXJhdGlvbl9kYXRlIjoxNzY5MTA1NTgyNTY0LCJ0b2tlbl9jcmVhdGVfZGF0ZSI6MTc2NjUxMzU4MjU2NH0.WCzojWa7DEuxPgJyOzBN4YQ6HmDC6VIzMzksI2toXYUMi3eTAylv0M0dso3SD3v3m6PfEabhDFdUW-707Sdlig",
  "macAddress": null,
  "clientIp": "192.168.25.5"
}
```

**IMPORTANTE:**
> **Nota:** En la respuesta del login debería incluirse el campo `codigoAgente` para identificar al agente en las operaciones subsiguientes. Este campo es necesario para las consultas de cartera y comisiones.


---

## 3. APIs del Módulo de Agentes

### 3.1 Gestión de Cartera

#### 3.1.1 Resumen de Cartera (Must Have)

**Endpoint:** `GET /agents/portfolio/summary`

**Descripción:** Obtiene el resumen total de la cartera del agente, incluyendo pólizas activas, pólizas por vencer y total de primas.

**Query Parameters:**
- `codigoAgente`: Código del agente (string, requerido)

**Headers:**
```http
Authorization: Bearer {token}
```

**Ejemplo de Request:**
```http
GET /agents/portfolio/summary?codigoAgente=98086
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

**Response (200 OK):**
```json
{
  "codigoAgente": "98086",
  "nombreAgente": "Asescor S.R.L. Corredores De Seguros",
  "resumen": {
    "polizasActivas": 45,
    "polizasPorVencer": 8,
    "totalPrimas": {
      "amount": 185000.00,
      "currency": "USD"
    }
  },
  "periodoReferencia": {
    "fechaConsulta": "2026-02-03",
    "mesesVencimiento": 1
  },
  "ultimaActualizacion": "2026-02-03T10:30:00Z"
}
```

**Notas:**
- Las pólizas por vencer se calculan tomando 1 mes como referencia desde la fecha actual
- El total de primas representa la suma de todas las primas anuales de las pólizas activas

---

### 3.2 Listado de Terceros (Clientes) y sus Pólizas

#### 3.2.1 Obtener Terceros con Información de Pólizas (Must Have)

**Endpoint:** `GET /agents/clients/policies`

**Descripción:** Obtiene el listado completo de terceros (clientes) asignados al agente junto con la información detallada de sus pólizas activas en una sola llamada.

**Query Parameters:**
- `codigoAgente`: Código del agente (string, requerido)
- `page`: Número de página (opcional, default: 0)
- `size`: Tamaño de página (opcional, default: 20)

**Headers:**
```http
Authorization: Bearer {token}
```

**Servicios Base Utilizados:**
- Usuario: `GET http://dev.graphapibisaseg.com:8081/party/parties/saludInt/user/profile/{documento}`
- Póliza: `GET http://dev.graphapibisaseg.com:8081/contract/issues/policiesInt/{partyCode}`

**Ejemplo de Request:**
```http
GET /agents/clients/policies?codigoAgente=98086&page=0&size=20
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

**Response (200 OK):**
```json
{
  "codigoAgente": "98086",
  "content": [
    {
      "tercero": {
        "partyCode": "25003930",
        "numeroDocumento": "1511972",
        "tipoDocumento": "CI",
        "nombreCompleto": "PAUKER URMAN JOSE",
        "nombre": "JOSE",
        "apellido": "PAUKER URMAN",
        "fechaNacimiento": "1965-10-20",
        "email": "josepauker@gmail.com",
        "telefono": "",
        "direccion": "BARRIO LAS PALMAS CALLE QUIMOME",
        "ciudad": "10201",
        "pais": "2220130"
      },
      "poliza": {
        "contractNo": "C2010001273",
        "policyNo": "P2010001259",
        "productName": "Asistencia Medica",
        "productNodeName": "Global Health Premier",
        "startDate": "2025-09-01",
        "endDate": "2026-09-01",
        "premiumAnual": "7868.09",
        "currency": "USD",
        "dependientes": [
          {
            "codeDep": "25003930",
            "nameDep": "PAUKER URMAN JOSE",
            "ciDep": "1511972",
            "relacion": "TITULAR"
          },
          {
            "codeDep": "25003931",
            "nameDep": "ARCE DE PAUKER ROSARIO",
            "ciDep": "3280332",
            "relacion": "CONYUGUE"
          }
        ]
      }
    },
    {
      "tercero": {
        "partyCode": "25004125",
        "numeroDocumento": "2345678",
        "tipoDocumento": "CI",
        "nombreCompleto": "GARCIA LOPEZ MARIA",
        "nombre": "MARIA",
        "apellido": "GARCIA LOPEZ",
        "fechaNacimiento": "1978-03-15",
        "email": "maria.garcia@email.com",
        "telefono": "+591 70123456",
        "direccion": "Av. Principal 456",
        "ciudad": "10201",
        "pais": "2220130"
      },
      "poliza": {
        "contractNo": "C2010001280",
        "policyNo": "P2010001265",
        "productName": "Asistencia Medica",
        "productNodeName": "Global Health Standard",
        "startDate": "2025-10-01",
        "endDate": "2026-10-01",
        "premiumAnual": "5200.00",
        "currency": "USD",
        "dependientes": [
          {
            "codeDep": "25004125",
            "nameDep": "GARCIA LOPEZ MARIA",
            "ciDep": "2345678",
            "relacion": "TITULAR"
          }
        ]
      }
    }
  ],
  "page": {
    "number": 0,
    "size": 20,
    "totalElements": 45,
    "totalPages": 3
  },
  "timestamp": "2026-02-03T10:30:00Z"
}
```

**Notas:**
- Este endpoint combina la información de dos servicios en una sola llamada
- Cada elemento del array `content` contiene tanto los datos del tercero como su póliza activa
- Si un tercero no tiene póliza activa, el campo `poliza` será `null`
- La información de dependientes está incluida dentro de cada póliza

---

### 3.3 Deducibles y Coberturas

#### 3.3.1 Obtener Deducibles de una Póliza (Must Have)

**Endpoint:** `GET http://dev.graphapibisaseg.com:8081/contract/issues/policiesDed/{policyNo}`

**Descripción:** Obtiene la información de deducibles y datos generales de una póliza específica.

**Path Parameters:**
- `policyNo`: Número de póliza (string, requerido). Ejemplo: `P2010001259`

**Headers:**
```http
Authorization: Bearer {token}
```

**Ejemplo de Request:**
```http
GET http://dev.graphapibisaseg.com:8081/contract/issues/policiesDed/P2010001259
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

**Response (200 OK):**
```json
{
  "error": false,
  "errorMessage": "",
  "result": {
    "numeroPoliza": "P2010001259",
    "nombreProducto": "Global Health Premier",
    "nombreTitular": "Laboratorios Y Optica Pauker Ltda.",
    "apellidoTitular": "Laboratorios Y Optica Pauker Ltda.",
    "direccionTitular": "Casco Viejo,Calle 24 De Septiembre,#201",
    "ciudadTitular": "Santa Cruz",
    "ciudadPoliza": "Santa Cruz",
    "dependientesPoliza": "Arce De Pauker Rosario",
    "codigoAgente": "98086",
    "nombreAgente": "Asescor S.R.L. Corredores De Seguros",
    "anioPoliza": "2025",
    "fechaEmision": "23/09/2025",
    "fechaLargaEmision": "23 de septiembre de 2025",
    "vigenciaInicial": "01/09/2025",
    "vigenciaFinal": "01/09/2026",
    "fechaVencimientoPago": "01/09/2025",
    "fechaCorte": "28/01/2026",
    "formaPagoPoliza": "Semestral – 2 cuotas",
    "valorAseguradoPoliza": "6,500,000.00",
    "deduciblePolizaInt": " US$ 10.000",
    "deduciblePolizaBol": " US$ 10.000",
    "montoPrimaTotal": "15,650.16",
    "totalPagar": "7,825.08",
    "cantidadRecibida": "0.00",
    "balanceAnterior": "7,973.70",
    "balanceActual": "15,947.40"
  }
}
```

**Campos Importantes:**
- `deduciblePolizaInt`: Deducible para servicios internacionales
- `deduciblePolizaBol`: Deducible para servicios en Bolivia
- `valorAseguradoPoliza`: Monto máximo de cobertura
- `balanceActual`: Balance actual del deducible
- `formaPagoPoliza`: Frecuencia de pago de la prima

---

#### 3.3.2 Obtener Coberturas de una Póliza (Must Have)

**Endpoint:** `GET http://dev.graphapibisaseg.com:8081/contract/issues/policiesCob/{policyNo}/{nodo}`

**Descripción:** Obtiene el detalle de coberturas de una póliza específica por nodo.

**Path Parameters:**
- `policyNo`: Número de póliza (string, requerido). Ejemplo: `P2010001259`
- `nodo`: Nodo de cobertura (string, requerido). Ejemplo: `1-1-1`

**Headers:**
```http
Authorization: Bearer {token}
```

**Ejemplo de Request:**
```http
GET http://dev.graphapibisaseg.com:8081/contract/issues/policiesCob/P2010001259/1-1-1
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

**Response (200 OK):**
```json
{
  "error": false,
  "errorMessage": "",
  "result": [
    {
      "VALOR_ASEGURADO": "6500000.00",
      "PRIMA": "5828.21",
      "PRIMA_REASEGURO": "-349.6031746032",
      "TASA": "0.0",
      "TASA_REASEGURO": "0.0",
      "CANTIDAD": "0.0",
      "PORCENTAJE_DEDUCIBLE": "0.0",
      "PORCENTAJE_COPAGO": "0.0",
      "PORCENTAJE_COASEGURO": "0.0",
      "COPAGO_CANTIDAD": "0.0",
      "LIMITE_ASEGURADO_VIGENCIA": "0.000000",
      "PERIODO_VIGENCIA": "0",
      "PERIODO_ESPERA": "0"
    }
  ]
}
```

**Campos Importantes:**
- `VALOR_ASEGURADO`: Monto asegurado para esta cobertura
- `PRIMA`: Prima correspondiente a esta cobertura
- `PORCENTAJE_DEDUCIBLE`: Porcentaje de deducible aplicable
- `PORCENTAJE_COPAGO`: Porcentaje de copago
- `PORCENTAJE_COASEGURO`: Porcentaje de coaseguro
- `PERIODO_ESPERA`: Período de espera en días

---

## 4. Resumen de Endpoints

### Resumen por Módulo

| # | Endpoint | Método | Descripción | Prioridad |
|---|----------|--------|-------------|-----------|
| 1 | `/security/auth/authenticateBasic` | GET | Login de agente con Basic Auth | Must Have |
| 2 | `/agents/portfolio/summary` | GET | Resumen de cartera del agente | Must Have |
| 3 | `/agents/clients/policies` | GET | Listado de terceros con pólizas | Must Have |
| 4 | `/contract/issues/policiesDed/{policyNo}` | GET | Deducibles de una póliza | Must Have |
| 5 | `/contract/issues/policiesCob/{policyNo}/{nodo}` | GET | Coberturas de una póliza | Must Have |

### Total de Endpoints: 5

---

## Notas Importantes

### Cambios Respecto a Versión Anterior

1. **Autenticación Simplificada**: Se usa el servicio de autenticación existente de BISA con Basic Auth y tokens JWT.

2. **Servicios Integrados**: Se aprovechan los servicios existentes de BISA para obtener información de usuarios, pólizas, deducibles y coberturas.

3. **Endpoint Combinado**: El nuevo endpoint `/agents/clients/policies` combina la información de terceros y pólizas en una sola llamada para optimizar el rendimiento.

4. **Campo Requerido**: Es necesario que el login retorne el campo `codigoAgente` para las operaciones subsiguientes.

5. **Período de Referencia**: Las pólizas por vencer se calculan con 1 mes de anticipación desde la fecha actual.

### Campos de Usuario Requeridos

Para el correcto funcionamiento del portal, el servicio de login debe retornar:

- `userCode`: Código de usuario
- `firstName`: Nombre
- `lastName`: Apellido
- `emailCorp`: Email corporativo
- `token`: Token JWT para autenticación
- **`codigoAgente`**: Código del agente (NUEVO - REQUERIDO)

---

**Documento preparado por:** Xiobit Solutions
**Fecha:** 3 de Febrero, 2026
**Versión:** 2.0

Este documento ha sido actualizado basándose en la reunión de discovery del 2 de febrero de 2026 con el equipo de BISA Seguros.
