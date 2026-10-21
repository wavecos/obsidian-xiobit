# Endpoints a revisar — Módulo de Reembolsos

**Para:** equipo de Ethical Hacking · agosto 2026
**Base URLs:** 
`https://bisa-api.xiobit.com`
`https://servcon.atlasglobalhealth.com`
`http://10.10.1.14:8080`

52 endpoints en 6 controladores. Autenticación JWT Bearer salvo donde se indique.

---

## 1. Portal de agentes — `/api/v1/reembolsos`

Clase: `ROLE_AGENT, ROLE_ADMIN`

| Método | Ruta                                              | Rol          |     |
| ------ | ------------------------------------------------- | ------------ | --- |
| GET    | `/api/v1/reembolsos/validacion-poliza/{policyNo}` | clase        |     |
| POST   | `/api/v1/reembolsos`                              | `ROLE_AGENT` |     |
| PUT    | `/api/v1/reembolsos/{id}`                         | `ROLE_AGENT` |     |
| POST   | `/api/v1/reembolsos/{id}/documentos`              | `ROLE_AGENT` |     |
| DELETE | `/api/v1/reembolsos/{id}/documentos/{docId}`      | `ROLE_AGENT` |     |
| POST   | `/api/v1/reembolsos/{id}/enviar`                  | `ROLE_AGENT` |     |
| GET    | `/api/v1/reembolsos`                              | clase        |     |
| GET    | `/api/v1/reembolsos/{id}`                         | clase        |     |
| GET    | `/api/v1/reembolsos/{id}/documentos`              | clase        |     |
| GET    | `/api/v1/reembolsos/{id}/documentos/{docId}`      | clase        |     |
| GET    | `/api/v1/reembolsos/{id}/historial`               | clase        |     |
| GET    | `/api/v1/reembolsos/{id}/comprobante`             | clase        |     |

## 2. Portal de clientes y apps — `/api/v1/mis-reembolsos`

Clase: `ROLE_AFFILIATE`. Solo lectura.

| Método | Ruta                                             | Rol   |     |
| ------ | ------------------------------------------------ | ----- | --- |
| GET    | `/api/v1/mis-reembolsos`                         | clase |     |
| GET    | `/api/v1/mis-reembolsos/{id}`                    | clase |     |
| GET    | `/api/v1/mis-reembolsos/{id}/documentos`         | clase |     |
| GET    | `/api/v1/mis-reembolsos/{id}/documentos/{docId}` | clase |     |
| GET    | `/api/v1/mis-reembolsos/{id}/historial`          | clase |     |
| GET    | `/api/v1/mis-reembolsos/{id}/comprobante`        | clase |     |

## 3. Backoffice — `/api/v1/backoffice/reembolsos`

Clase: `ROLE_ADMIN, ROLE_REEMBOLSOS, ROLE_REEMBOLSOS_OPS, ROLE_CALL_CENTER`

| Método | Ruta | Rol | |
|---|---|---|---|
| GET | `/api/v1/backoffice/reembolsos` | clase | 🔴 |
| GET | `/api/v1/backoffice/reembolsos/agentes` | clase | |
| GET | `/api/v1/backoffice/reembolsos/{id}` | clase | 🔴 |
| GET | `/api/v1/backoffice/reembolsos/{id}/documentos` | clase | |
| GET | `/api/v1/backoffice/reembolsos/{id}/documentos/{docId}/archivo` | `ROLE_ADMIN, ROLE_REEMBOLSOS, ROLE_REEMBOLSOS_OPS` | 🔴 📎 |
| GET | `/api/v1/backoffice/reembolsos/{id}/historial` | clase | |
| GET | `/api/v1/backoffice/reembolsos/{id}/documentos-faltantes` | clase | |
| PATCH | `/api/v1/backoffice/reembolsos/{id}/estado` | `ROLE_ADMIN, ROLE_REEMBOLSOS` | |
| PATCH | `/api/v1/backoffice/reembolsos/{id}/documentos/{docId}` | `ROLE_ADMIN, ROLE_REEMBOLSOS` | |
| POST | `/api/v1/backoffice/reembolsos/{id}/notas` | `ROLE_ADMIN, ROLE_REEMBOLSOS` | |
| POST | `/api/v1/backoffice/reembolsos/{id}/reabrir` | `ROLE_ADMIN, ROLE_REEMBOLSOS_OPS` | |
| POST | `/api/v1/backoffice/reembolsos/{id}/pago` | `ROLE_ADMIN, ROLE_REEMBOLSOS_OPS` | 🔴 |
| POST | `/api/v1/backoffice/reembolsos/{id}/comprobante` | `ROLE_ADMIN, ROLE_REEMBOLSOS_OPS` | ⬆️ |
| GET | `/api/v1/backoffice/reembolsos/{id}/comprobante` | `ROLE_ADMIN, ROLE_REEMBOLSOS, ROLE_REEMBOLSOS_OPS` | 🔴 📎 |
| GET | `/api/v1/backoffice/reembolsos/{id}/freshdesk` | clase | |
| POST | `/api/v1/backoffice/reembolsos/{id}/freshdesk/reintentar` | `ROLE_ADMIN, ROLE_REEMBOLSOS` | |

> En este controlador **no hay verificación por registro**: el backoffice ve toda la cartera por
> diseño, así que el rol del `@PreAuthorize` es el único control de acceso. La prueba que
> corresponde acá es de escalada vertical entre roles, no de IDOR entre expedientes.
>
> En Spring Security 6 la anotación de método **sustituye** a la de clase, no se suman.

## 4. Catálogos — `/api/v1/catalogos/reembolsos`

Clase: `ROLE_ADMIN, ROLE_REEMBOLSOS, ROLE_REEMBOLSOS_OPS, ROLE_CALL_CENTER, ROLE_AGENT, ROLE_AFFILIATE`

| Método | Ruta                                                     |     |
| ------ | -------------------------------------------------------- | --- |
| GET    | `/api/v1/catalogos/reembolsos/bancos`                    |     |
| GET    | `/api/v1/catalogos/reembolsos/tipos-documento`           |     |
| GET    | `/api/v1/catalogos/reembolsos/estados`                   |     |
| GET    | `/api/v1/catalogos/reembolsos/tipos-atencion`            |     |
| GET    | `/api/v1/catalogos/reembolsos/lugares-accidente`         |     |
| GET    | `/api/v1/catalogos/reembolsos/tipos-cuenta`              |     |
| GET    | `/api/v1/catalogos/reembolsos/monedas`                   |     |
| GET    | `/api/v1/catalogos/reembolsos/tipos-documento-identidad` |     |
| GET    | `/api/v1/catalogos/reembolsos/departamentos`             |     |
| GET    | `/api/v1/catalogos/reembolsos/tipos-gestor`              |     |

## 5. Catálogos de backoffice — `/api/v1/backoffice/catalogos/reembolsos`

Clase: `ROLE_ADMIN`

| Método | Ruta |
|---|---|
| GET | `/api/v1/backoffice/catalogos/reembolsos/bancos` |
| POST | `/api/v1/backoffice/catalogos/reembolsos/bancos` |
| PUT | `/api/v1/backoffice/catalogos/reembolsos/bancos/{id}` |
| DELETE | `/api/v1/backoffice/catalogos/reembolsos/bancos/{id}` |
| GET | `/api/v1/backoffice/catalogos/reembolsos/freshdesk-estados` |
| PUT | `/api/v1/backoffice/catalogos/reembolsos/freshdesk-estados` |

## 6. Webhook de Freshdesk —  sin JWT

| Método | Ruta                                        | Autenticación                                      |     |
| ------ | ------------------------------------------- | -------------------------------------------------- | --- |
| POST   | `/api/v1/public/webhooks/freshdesk/tickets` | Secreto compartido en `X-Freshdesk-Webhook-Secret` |     |

Único endpoint del módulo alcanzable sin sesión. Cuerpo: `{"ticketId": <long>}`.
Respuestas: 202 aplicado · 204 el ticket no es de un reembolso · 401 secreto ausente o inválido ·
503 integración deshabilitada o sin configurar.

---
