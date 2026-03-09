# Solicitud de Servicios Web - BISA Seguros Salud Internacional

## 1. INTRODUCCIÓN

El presente documento detalla los servicios web (APIs REST) requeridos para implementar el Portal de Asegurados de BISA Seguros Salud Internacional, tanto en su versión web (Angular) como móvil (Android/iOS).

Se requiere que todos los servicios:

* Utilicen protocolo **HTTPS**
* Retornen respuestas en formato **JSON**
* Implementen autenticación mediante **JWT (JSON Web Tokens)**
* Sigan estándares REST para métodos HTTP (GET, POST, PUT, DELETE)
* Incluyan manejo de errores con códigos HTTP estándar

***

## 2. SERVICIOS DE AUTENTICACIÓN Y SESIÓN

### 2.1. Login de Usuario

**Endpoint:** `POST /security/auth/authenticateBasic`

**Proporcionado**

### 2.2. Reset Password

Se requiere endpoint para reestablecer la contraseña

### 2.3. Refresh Token

Se requiere endpoint para hacer un "refresh token" (definir si tambien se lo hará en la App)

***

\* Los siguientes servicios son aproximaciones a lo que se requiere y se entiende que podrian varias tipos de propiedades, valores, nombres y nomenclatura.

## 3. SERVICIOS DE PERFIL DE USUARIO

### 3.1. Obtener Información del Usuario

**Endpoint:** `GET /api/v1/usuarios/perfil`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "id": "uuid",
  "nombre": "Juan",
  "apellido": "Salas",
  "nombreCompleto": "Juan Salas",
  "email": "juan.salas@email.com",
  "tipoDocumento": "CI",
  "numeroDocumento": "1234567",
  "telefono": "+591 7123 4567",
  "fechaNacimiento": "1985-03-15",
  "direccion": "Calle Principal #123",
  "ciudad": "La Paz",
  "pais": "Bolivia"
}
```

### 3.2. Actualizar Información del Usuario

**Endpoint:** `PUT /api/v1/usuarios/perfil`

**Headers:** `Authorization: Bearer {token}`

**Request Body:**

```json
{
  "email": "nuevo@email.com",
  "telefono": "+591 7123 4567",
  "direccion": "Nueva dirección",
  "ciudad": "La Paz"
}
```

***

## 4. SERVICIOS DE PÓLIZAS

### 4.1. Listar Pólizas del Usuario

**Endpoint:** `GET /api/v1/polizas`

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:**

* `estado` (opcional): `ACTIVA|VENCIDA|CANCELADA|TODAS`
* `page` (opcional): número de página (default: 0)
* `size` (opcional): tamaño de página (default: 10)

**Response exitoso (200 OK):**

```json
{
  "content": [
    {
      "id": "uuid",
      "numeroPoliza": "BSI-2024-125",
      "tipo": "Salud Internacional",
      "plan": "Premium Plus",
      "estado": "ACTIVA",
      "fechaInicio": "2024-01-05",
      "fechaFin": "2025-01-04",
      "primaAnual": 2300.00,
      "moneda": "USD",
      "asegurado": {
        "nombre": "Juan Pérez Rodríguez",
        "documento": "CI 1234567"
      }
    }
  ],
  "totalElements": 1,
  "totalPages": 1,
  "currentPage": 0
}
```

### 4.2. Obtener Detalle de Póliza

**Endpoint:** `GET /api/v1/polizas/{numeroPoliza}`

**Headers:** `Authorization: Bearer {token}`

**Path Parameters:**

* `numeroPoliza`: Número de póliza (ej: "BSI-2024-125")

**Response exitoso (200 OK):**

```json
{
  "id": "uuid",
  "numeroPoliza": "BSI-2024-125",
  "tipo": "Salud Internacional",
  "plan": "Premium Plus",
  "estado": "ACTIVA",
  "fechaInicio": "2024-01-05",
  "fechaFin": "2025-01-04",
  "primaAnual": 2500.00,
  "moneda": "USD",
  "asegurado": {
    "nombre": "Juan Pérez Rodríguez",
    "tipoDocumento": "CI",
    "numeroDocumento": "1234567",
    "email": "juan.perez@email.com",
    "telefono": "+591 7123 4567"
  },
  "coberturas": [
    {
      "nombre": "Consultas médicas",
      "montoMaximo": 5000.00,
      "montoUtilizado": 240.00,
      "montoDisponible": 4760.00,
      "porcentajeUtilizado": 4.8
    },
    {
      "nombre": "Medicamentos",
      "montoMaximo": 3000.00,
      "montoUtilizado": 180.00,
      "montoDisponible": 2820.00,
      "porcentajeUtilizado": 6.0
    },
    {
      "nombre": "Exámenes",
      "montoMaximo": 2000.00,
      "montoUtilizado": 180.00,
      "montoDisponible": 1820.00,
      "porcentajeUtilizado": 9.0
    }
  ],
  "deducible": {
    "montoTotal": 1000.00,
    "montoUtilizado": 600.00,
    "montoDisponible": 400.00,
    "porcentajeUtilizado": 60.0
  },
  "beneficiarios": [
    {
      "nombre": "María Rodríguez",
      "parentesco": "Cónyuge",
      "porcentaje": 100
    }
  ]
}
```

### 4.3. Descargar Condicionado de Póliza (PDF)

**Endpoint:** `GET /api/v1/polizas/{numeroPoliza}/condicionado`

**Headers:** `Authorization: Bearer {token}`

**Path Parameters:**

* `numeroPoliza`: Número de póliza

**Response exitoso (200 OK):**

* Content-Type: `application/pdf`
* Retorna el archivo PDF del condicionado

**Nota:** Alternativamente puede retornar una URL pre-firmada:

```json
{
  "url": "https://storage.bisa.com/documentos/condicionado-BSI-2024-125.pdf?token=...",
  "expiresIn": 3600
}
```

***

## 5. SERVICIOS DE DEDUCIBLES Y USO

### 5.1. Obtener Desglose de Deducibles

**Endpoint:** `GET /api/v1/polizas/{numeroPoliza}/deducibles`

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:**

* `fechaInicio` (opcional): Fecha inicio filtro (formato: YYYY-MM-DD)
* `fechaFin` (opcional): Fecha fin filtro (formato: YYYY-MM-DD)

**Response exitoso (200 OK):**

```json
{
  "numeroPoliza": "BSI-2024-125",
  "periodo": "2024-01-05 a 2025-01-04",
  "resumen": {
    "montoTotal": 1000.00,
    "montoUtilizado": 600.00,
    "montoDisponible": 400.00,
    "porcentajeUtilizado": 60.0
  },
  "desglosePorServicio": [
    {
      "servicio": "Consultas",
      "cantidad": 4,
      "descripcion": "4 visitas",
      "montoUtilizado": 240.00
    },
    {
      "servicio": "Medicamentos",
      "cantidad": 3,
      "descripcion": "3 recetas",
      "montoUtilizado": 180.00
    },
    {
      "servicio": "Exámenes",
      "cantidad": 2,
      "descripcion": "2 estudios",
      "montoUtilizado": 180.00
    }
  ],
  "detalleTransacciones": [
    {
      "id": "uuid",
      "fecha": "2024-10-15",
      "servicio": "Consultas",
      "proveedor": "Clínica Los Andes",
      "descripcion": "Consulta general",
      "monto": 60.00
    }
  ]
}
```

***

## 6. SERVICIOS DE FACTURAS

### 6.1. Listar Facturas

**Endpoint:** `GET /api/v1/facturas`

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:**

* `numeroPoliza` (opcional): Filtrar por número de póliza
* `fechaInicio` (opcional): Fecha inicio (YYYY-MM-DD)
* `fechaFin` (opcional): Fecha fin (YYYY-MM-DD)
* `estado` (opcional): `PAGADA|PENDIENTE|VENCIDA`
* `page` (opcional): número de página
* `size` (opcional): tamaño de página

**Response exitoso (200 OK):**

```json
{
  "content": [
    {
      "id": "uuid",
      "numeroFactura": "FAC-2024-103",
      "numeroPoliza": "BSI-2024-125",
      "fecha": "2024-10-15",
      "fechaVencimiento": "2024-10-30",
      "concepto": "Prima Trimestral",
      "monto": 625.00,
      "moneda": "USD",
      "estado": "PAGADA",
      "metodoPago": "Tarjeta de crédito",
      "urlPDF": "https://..."
    },
    {
      "id": "uuid",
      "numeroFactura": "FAC-2024-98",
      "numeroPoliza": "BSI-2024-125",
      "fecha": "2024-08-08",
      "fechaVencimiento": "2024-08-23",
      "concepto": "Prima Trimestral",
      "monto": 450.00,
      "moneda": "USD",
      "estado": "PAGADA",
      "metodoPago": "Transferencia bancaria",
      "urlPDF": "https://..."
    }
  ],
  "totalElements": 15,
  "totalPages": 2,
  "currentPage": 0
}
```

### 6.2. Descargar Factura (PDF)

**Endpoint:** `GET /api/v1/facturas/{numeroFactura}/pdf`

**Headers:** `Authorization: Bearer {token}`

**Path Parameters:**

* `numeroFactura`: Número de factura (ej: "FAC-2024-103")

**Response exitoso (200 OK):**

* Content-Type: `application/pdf`
* Retorna el archivo PDF de la factura

**Alternativa - URL pre-firmada:**

```json
{
  "url": "https://storage.bisa.com/facturas/FAC-2024-103.pdf?token=...",
  "expiresIn": 3600
}
```

***

## 7. SERVICIOS DE PAGOS

### 7.1. Obtener Métodos de Pago Disponibles

**Endpoint:** `GET /api/v1/pagos/metodos`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "metodos": [
    {
      "id": "tarjeta_credito",
      "nombre": "Tarjeta de Crédito/Débito",
      "descripcion": "Visa, Mastercard, American Express",
      "habilitado": true,
      "icono": "https://..."
    },
    {
      "id": "transferencia",
      "nombre": "Transferencia Bancaria",
      "descripcion": "Transferencia directa a cuenta BISA",
      "habilitado": true,
      "datosBancarios": {
        "banco": "Banco BISA S.A.",
        "titular": "BISA Seguros",
        "numeroCuenta": "1234567890",
        "swift": "BISABOLPXXX"
      }
    },
    {
      "id": "qr_simple",
      "nombre": "Pago QR Simple",
      "descripcion": "Código QR para pago con apps bancarias",
      "habilitado": true
    }
  ]
}
```

### 7.2. Generar Orden de Pago

**Endpoint:** `POST /api/v1/pagos/generar-orden`

**Headers:** `Authorization: Bearer {token}`

**Request Body:**

```json
{
  "numeroPoliza": "BSI-2024-125",
  "numeroFactura": "FAC-2024-103",
  "monto": 625.00,
  "metodoPago": "tarjeta_credito",
  "urlRetorno": "https://app.bisa.com/pagos/resultado",
  "urlNotificacion": "https://api.waveonix.com/webhook/pago"
}
```

**Response exitoso (201 Created):**

```json
{
  "ordenPagoId": "uuid",
  "numeroOrden": "ORD-2024-5678",
  "monto": 625.00,
  "moneda": "USD",
  "estado": "PENDIENTE",
  "metodoPago": "tarjeta_credito",
  "pasarelaPago": {
    "proveedor": "Stripe|PayPal|PagoSeguro",
    "urlPago": "https://pasarela.com/checkout/...",
    "token": "payment_token_123",
    "expiresIn": 1800
  },
  "qrCode": "data:image/png;base64,..." // Si aplica para QR
}
```

### 7.3. Consultar Estado de Pago

**Endpoint:** `GET /api/v1/pagos/{ordenPagoId}/estado`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "ordenPagoId": "uuid",
  "numeroOrden": "ORD-2024-5678",
  "estado": "PAGADO|PENDIENTE|RECHAZADO|CANCELADO",
  "monto": 625.00,
  "fechaPago": "2024-10-15T14:30:00Z",
  "metodoPago": "tarjeta_credito",
  "numeroTransaccion": "TXN-123456",
  "numeroFactura": "FAC-2024-103",
  "recibo": {
    "url": "https://...",
    "numeroRecibo": "REC-2024-890"
  }
}
```

### 7.4. Historial de Pagos

**Endpoint:** `GET /api/v1/pagos/historial`

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:**

* `numeroPoliza` (opcional)
* `fechaInicio` (opcional)
* `fechaFin` (opcional)
* `page`, `size`

**Response exitoso (200 OK):**

```json
{
  "content": [
    {
      "id": "uuid",
      "fecha": "2024-10-15T14:30:00Z",
      "numeroFactura": "FAC-2024-103",
      "numeroPoliza": "BSI-2024-125",
      "concepto": "Prima Trimestral",
      "monto": 625.00,
      "metodoPago": "Tarjeta de crédito",
      "estado": "PAGADO",
      "numeroTransaccion": "TXN-123456",
      "numeroRecibo": "REC-2024-890"
    }
  ],
  "totalElements": 8,
  "totalPages": 1,
  "currentPage": 0
}
```

***

## 8. SERVICIOS DE CENTROS MÉDICOS

### 8.1. Listar Centros Médicos (Red de Proveedores)

**Endpoint:** `GET /api/v1/centros-medicos`

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:**

* `ciudad` (opcional): Filtrar por ciudad
* `especialidad` (opcional): Filtrar por especialidad médica
* `nombre` (opcional): Búsqueda por nombre
* `latitud` (opcional): Coordenada para búsqueda por cercanía
* `longitud` (opcional): Coordenada para búsqueda por cercanía
* `radio` (opcional): Radio en km para búsqueda (default: 5)
* `page`, `size`

**Response exitoso (200 OK):**

```json
{
  "content": [
    {
      "id": "uuid",
      "nombre": "Clínica Los Andes",
      "tipo": "Clínica|Hospital|Laboratorio|Farmacia",
      "direccion": "Av. Arce #123",
      "ciudad": "La Paz",
      "pais": "Bolivia",
      "telefono": "+591 2 2123456",
      "email": "info@clinicalosandes.com",
      "horarioAtencion": "Lun-Vie: 8:00-18:00, Sáb: 8:00-13:00",
      "emergencias24h": true,
      "coordenadas": {
        "latitud": -16.5000,
        "longitud": -68.1500
      },
      "especialidades": [
        "Medicina General",
        "Pediatría",
        "Cardiología"
      ],
      "servicios": [
        "Consulta externa",
        "Emergencias",
        "Hospitalización",
        "Laboratorio"
      ],
      "imagenes": [
        "https://..."
      ],
      "distancia": 2.5 // Si se busca por ubicación
    }
  ],
  "totalElements": 45,
  "totalPages": 5,
  "currentPage": 0
}
```

### 8.2. Obtener Detalle de Centro Médico

**Endpoint:** `GET /api/v1/centros-medicos/{id}`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "id": "uuid",
  "nombre": "Clínica Los Andes",
  "tipo": "Clínica",
  "descripcion": "Clínica con más de 20 años de experiencia...",
  "direccion": "Av. Arce #123",
  "ciudad": "La Paz",
  "pais": "Bolivia",
  "telefono": "+591 2 2123456",
  "email": "info@clinicalosandes.com",
  "sitioWeb": "https://www.clinicalosandes.com",
  "horarioAtencion": "Lun-Vie: 8:00-18:00, Sáb: 8:00-13:00",
  "emergencias24h": true,
  "coordenadas": {
    "latitud": -16.5000,
    "longitud": -68.1500
  },
  "especialidades": [...],
  "servicios": [...],
  "medicos": [
    {
      "nombre": "Dr. Juan Pérez",
      "especialidad": "Cardiología",
      "titulo": "Médico Cardiólogo",
      "telefono": "+591 7123456"
    }
  ],
  "convenio": {
    "tipoConvenio": "DIRECTO",
    "requiereAutorizacion": false,
    "porcentajeCobertura": 100,
    "observaciones": "Cobertura total en consultas"
  },
  "imagenes": [...],
  "valoraciones": {
    "promedio": 4.5,
    "totalReviews": 123
  }
}
```

***

## 9. SERVICIOS DE NOTIFICACIONES

### 9.1. Listar Notificaciones del Usuario

**Endpoint:** `GET /api/v1/notificaciones`

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:**

* `leida` (opcional): `true|false` - Filtrar por estado de lectura
* `tipo` (opcional): `PAGO|POLIZA|SISTEMA|RECORDATORIO`
* `page`, `size`

**Response exitoso (200 OK):**

```json
{
  "content": [
    {
      "id": "uuid",
      "tipo": "PAGO",
      "titulo": "Pago recibido",
      "mensaje": "Se ha registrado el pago de la factura FAC-2024-103",
      "fecha": "2024-10-15T14:30:00Z",
      "leida": false,
      "accion": {
        "tipo": "NAVEGAR",
        "destino": "/facturas/FAC-2024-103"
      },
      "icono": "payment",
      "prioridad": "NORMAL|ALTA|URGENTE"
    }
  ],
  "notificacionesNoLeidas": 3,
  "totalElements": 25,
  "totalPages": 3,
  "currentPage": 0
}
```

### 9.2. Marcar Notificación como Leída

**Endpoint:** `PUT /api/v1/notificaciones/{id}/marcar-leida`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "message": "Notificación marcada como leída"
}
```

### 9.3. Obtener Contador de Notificaciones No Leídas

**Endpoint:** `GET /api/v1/notificaciones/no-leidas/contador`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "cantidad": 3
}
```

***

## 10. SERVICIOS DE SOPORTE Y FAQ

### 10.1. Listar Preguntas Frecuentes

**Endpoint:** `GET /api/v1/faq`

**Query Parameters:**

* `categoria` (opcional): Filtrar por categoría
* `busqueda` (opcional): Búsqueda por texto

**Response exitoso (200 OK):**

```json
{
  "categorias": [
    {
      "id": "polizas",
      "nombre": "Pólizas y Coberturas",
      "icono": "policy",
      "preguntas": [
        {
          "id": "uuid",
          "pregunta": "¿Cómo activo mi póliza?",
          "respuesta": "Para activar tu póliza debes...",
          "orden": 1,
          "util": {
            "votosPositivos": 45,
            "votosNegativos": 2
          }
        }
      ]
    },
    {
      "id": "pagos",
      "nombre": "Pagos y Facturación",
      "icono": "payment",
      "preguntas": [...]
    }
  ]
}
```

### 10.2. Enviar Consulta de Soporte

**Endpoint:** `POST /api/v1/soporte/consulta`

**Headers:** `Authorization: Bearer {token}`

**Request Body:**

```json
{
  "asunto": "Consulta sobre mi póliza",
  "mensaje": "Tengo una duda sobre...",
  "categoria": "POLIZAS|PAGOS|TECNICO|OTRO",
  "prioridad": "BAJA|MEDIA|ALTA",
  "numeroPoliza": "BSI-2024-125" // Opcional
}
```

**Response exitoso (201 Created):**

```json
{
  "ticketId": "uuid",
  "numeroTicket": "TKT-2024-789",
  "estado": "ABIERTO",
  "fechaCreacion": "2024-10-15T14:30:00Z",
  "tiempoRespuestaEstimado": "24 horas"
}
```

***

## 11. SERVICIOS DE DOCUMENTOS

### 11.1. Subir Documento

**Endpoint:** `POST /api/v1/documentos/upload`

**Headers:**

* `Authorization: Bearer {token}`
* `Content-Type: multipart/form-data`

**Request Body (multipart/form-data):**

* `file`: Archivo (PDF, JPG, PNG - max 10MB)
* `tipo`: Tipo de documento (ej: "RECLAMACION", "REEMBOLSO", "IDENTIFICACION")
* `numeroPoliza`: Número de póliza asociada
* `descripcion`: Descripción opcional

**Response exitoso (201 Created):**

```json
{
  "documentoId": "uuid",
  "nombreArchivo": "reclamacion.pdf",
  "tipoArchivo": "application/pdf",
  "tamano": 1024000,
  "url": "https://storage.bisa.com/documentos/uuid.pdf",
  "fechaCarga": "2024-10-15T14:30:00Z"
}
```

***

## 12. CONFIGURACIÓN Y PREFERENCIAS

### 12.1. Obtener Configuración de Usuario

**Endpoint:** `GET /api/v1/usuarios/configuracion`

**Headers:** `Authorization: Bearer {token}`

**Response exitoso (200 OK):**

```json
{
  "notificaciones": {
    "email": true,
    "push": true,
    "sms": false
  },
  "idioma": "es",
  "moneda": "USD",
  "tema": "light|dark|auto"
}
```

### 12.2. Actualizar Configuración de Usuario

**Endpoint:** `PUT /api/v1/usuarios/configuracion`

**Headers:** `Authorization: Bearer {token}`

**Request Body:**

```json
{
  "notificaciones": {
    "email": true,
    "push": true,
    "sms": false
  },
  "idioma": "es",
  "tema": "dark"
}
```

***

## 13. CÓDIGOS DE ERROR ESPERADOS

### Códigos HTTP Estándar:

* `200 OK` - Solicitud exitosa
* `201 Created` - Recurso creado exitosamente
* `400 Bad Request` - Datos de entrada inválidos
* `401 Unauthorized` - Token inválido o expirado
* `403 Forbidden` - Sin permisos para acceder al recurso
* `404 Not Found` - Recurso no encontrado
* `409 Conflict` - Conflicto con el estado actual
* `422 Unprocessable Entity` - Validación de negocio fallida
* `429 Too Many Requests` - Límite de rate limiting excedido
* `500 Internal Server Error` - Error interno del servidor
* `503 Service Unavailable` - Servicio temporalmente no disponible

### Formato de Respuesta de Error:

```json
{
  "error": {
    "codigo": "POLIZA_NO_ENCONTRADA",
    "mensaje": "No se encontró la póliza BSI-2024-125",
    "timestamp": "2024-10-15T14:30:00Z",
    "path": "/api/v1/polizas/BSI-2024-125",
    "detalles": {
      "campo": "numeroPoliza",
      "valorRecibido": "BSI-2024-125"
    }
  }
}
```

***

## 14. CONSIDERACIONES DE SEGURIDAD

### 14.1. Autenticación

* Todos los endpoints (excepto `/auth/login`) requieren JWT válido
* Los tokens deben enviarse en el header: `Authorization: Bearer {token}`
* Tokens expiran en 1 hora (configurable)
* Implementar refresh tokens para renovación automática

### 14.2. Rate Limiting

* Límite sugerido: 100 requests por minuto por usuario
* Límite para endpoints de login: 5 intentos cada 15 minutos

### 14.3. CORS

* Permitir orígenes desde:
  * `https://app.bisaseguros.com` (producción web)
  * `https://staging.bisaseguros.com` (staging web)
  * Aplicaciones móviles (Android/iOS)

### 14.4. Encriptación

* Todas las comunicaciones deben ser sobre HTTPS/TLS 1.2+
* Datos sensibles (contraseñas) deben estar hasheados con bcrypt
* Información personal debe cumplir con normativas de protección de datos

***

## 15. REQUISITOS NO FUNCIONALES

### 15.1. Performance

* Tiempo de respuesta promedio: < 500ms
* Tiempo de respuesta máximo: < 2 segundos
* Disponibilidad: 99.9% uptime

### 15.2. Documentación

* OpenAPI/Swagger para documentación interactiva
* Ejemplos de request/response para cada endpoint
* Códigos de error documentados

### 15.3. Versionado

* Utilizar versionado en la URL: `/api/v1/...`
* Mantener compatibilidad hacia atrás por al menos 6 meses

### 15.4. Logs y Monitoreo

* Logging de todas las transacciones críticas (pagos, cambios de póliza)
* Trazabilidad con transaction IDs en headers
* Métricas de performance disponibles

***

## 16. AMBIENTE DE PRUEBAS

### 16.1. Datos de Prueba Requeridos

Solicitamos que se proporcionen:

* Al menos 3 usuarios de prueba con diferentes perfiles
* 5-10 pólizas de prueba (activas, vencidas, canceladas)
* 10-15 facturas de prueba con diferentes estados
* 5-10 centros médicos de prueba
* Datos de deducibles con transacciones simuladas

### 16.2. Ambientes

* **Desarrollo:** `https://api-dev.bisaseguros.com`
* **Staging:** `https://api-staging.bisaseguros.com`
* **Producción:** `https://api.bisaseguros.com`

***

## 17. APÉNDICES

### Apéndice A - Catálogos y Valores Constantes

**Estados de Póliza:**

* `ACTIVA`
* `VENCIDA`
* `CANCELADA`
* `SUSPENDIDA`

**Tipos de Documento:**

* `CI` - Cédula de Identidad
* `PASSPORT` - Pasaporte
* `RUC` - Registro Único de Contribuyentes

**Estados de Factura:**

* `PENDIENTE`
* `PAGADA`
* `VENCIDA`
* `CANCELADA`

**Estados de Pago:**

* `PENDIENTE`
* `PROCESANDO`
* `PAGADO`
* `RECHAZADO`
* `CANCELADO`
* `REEMBOLSADO`

**Monedas:**

* `USD` - Dólares Americanos
* `BOB` - Bolivianos
* `EUR` - Euros

###
