# Alcance de Ethical Hacking — Módulo de Reembolsos e integración con Freshdesk

**Para:** equipo de Ethical Hacking
**Versión:** agosto 2026 · segunda ronda
**Confidencial** — describe superficie de ataque y flujos de datos sensibles.

---

## 0. Lo que hay que leer primero

Este módulo es **el conjunto de datos más sensible de la plataforma**. Un expediente de
reembolso reúne, en una sola entidad:

- **Diagnóstico clínico** en texto libre.
- **Datos bancarios completos** del titular de la cuenta (banco, tipo, número, nombre, documento).
- **Evidencia médica adjunta**, hasta 10 archivos por expediente. Ocho tipos:
  `FACTURA`, `INFORME_MEDICO`, `RECETA_MEDICA`, `RESULTADOS_LABORATORIO`, `EPICRISIS`,
  `INFORME_SOAT`, `REPORTE_SINIESTRALIDAD_VEHICULAR`, `LIQUIDACION_OTRA_ASEGURADORA`.
- **Identidad del asegurado**: nombre, tipo y número de documento, correo, teléfono.

Ningún otro endpoint de Atlas combina las tres primeras. Si algo de este alcance se prioriza,
que sea esto.

**Tres modelos de autorización conviven en el mismo módulo**, y confundirlos hace perder tiempo:

| Superficie | Modelo | Qué probar |
|---|---|---|
| `/api/v1/reembolsos/**` (agentes) | Rol **+ verificación de propiedad** por registro | IDOR horizontal entre agentes |
| `/api/v1/mis-reembolsos/**` (asegurados) | Rol **+ verificación de propiedad** por registro | IDOR horizontal entre asegurados |
| `/api/v1/backoffice/reembolsos/**` | **Solo rol.** Sin comprobación por registro, a propósito | **Escalada vertical** entre roles |
| `/api/v1/public/webhooks/freshdesk/**` | Secreto compartido en cabecera, sin JWT | Fuerza bruta del secreto, replay |

> **El backoffice ve toda la cartera por diseño.** `esBackoffice = true` salta la comprobación
> de propiedad deliberadamente, porque operaciones necesita ver cualquier expediente. Buscar
> IDOR ahí no tiene sentido: lo que importa es **qué puede hacer cada rol**. Para eso hacen
> falta cuentas con un solo rol cada una (ver §8).

---

## 1. Arquitectura del módulo

### Actores

| Actor | Rol | Qué hace |
|---|---|---|
| Agente / corredor | `ROLE_AGENT` | **Único que crea.** Registra la solicitud en nombre del asegurado, adjunta documentos y la envía |
| Asegurado | `ROLE_AFFILIATE` | **Solo lectura** de sus propios expedientes (web y apps) |
| Gestión de reembolsos | `ROLE_REEMBOLSOS` | Revisa documentos, cambia estados, agrega notas |
| Operaciones | `ROLE_REEMBOLSOS_OPS` | Reapertura, registro del pago, carga del comprobante |
| Call center | `ROLE_CALL_CENTER` | Consulta el trámite. **Sin acceso a los adjuntos ni al comprobante** |
| Administrador | `ROLE_ADMIN` | Todo lo anterior, más el ABM de catálogos |

Todos los roles de backoffice tienen **MFA obligatoria** (remediación de julio, VUL-005): cada
login exige el OTP que llega al correo del usuario.

### Máquina de estados

```
BORRADOR ─→ REGISTRADO ─→ EN_REVISION ─┬─→ PENDIENTE_INFO ─→ EN_REVISION
                                        ├─→ EN_EVALUACION ─┬─→ APROBADO ─→ PAGADO
                                        ├─→ APROBADO        └─→ RECHAZADO
                                        └─→ RECHAZADO
```

- **Solo avanza.** `POST /{id}/reabrir` (exclusivo de `ROLE_REEMBOLSOS_OPS`) es el único camino
  que retrocede, exige motivo y queda en el historial.
- `PAGADO`, `RECHAZADO` y `ANULADO` son terminales.
- `BORRADOR` es interno: **el asegurado no lo ve**. Para él el expediente existe recién cuando
  el agente lo envía.
- El envío sella las tres declaraciones juradas con **hora e IP del servidor**, no del cliente.

### Persistencia

Tablas creadas por Liquibase (`036`, `037`, `041`): `reembolsos`, `reembolso_atenciones`,
`reembolso_documentos`, `reembolso_historial`, `reembolso_freshdesk_eventos`,
`catalogo_bancos`, `catalogo_estados_reembolso`, `catalogo_freshdesk_estados`,
`catalogo_tipos_documento_reembolso`.

Los archivos **no** se guardan en base: van al disco del contenedor bajo
`{UPLOAD_DIR}/reembolsos/{reembolsoId}/{uuid}.{ext}`, y **no hay resource handler** que los
sirva estáticamente — solo se alcanzan por los endpoints autenticados.

---

## 2. Inventario de endpoints

### 2.1 Portal de agentes — `/api/v1/reembolsos`

Clase: `@PreAuthorize("hasAnyAuthority('ROLE_AGENT','ROLE_ADMIN')")`

| # | Método | Ruta | Rol | Cuerpo / parámetros | Devuelve |
|---|---|---|---|---|---|
| 1 | GET | `/validacion-poliza/{policyNo}` | clase | `policyNo` en la ruta | Asegurados de la póliza (nombre, CI), vigencia |
| 2 | POST | `` | `ROLE_AGENT` | `ReembolsoCreateRequest` | Expediente creado |
| 3 | PUT | `/{id}` | `ROLE_AGENT` | `ReembolsoCreateRequest` | Expediente |
| 4 | POST | `/{id}/documentos` | `ROLE_AGENT` | **multipart**: `files[]` + `tipos[]` | Metadatos |
| 5 | DELETE | `/{id}/documentos/{docId}` | `ROLE_AGENT` | — | 204 |
| 6 | POST | `/{id}/enviar` | `ROLE_AGENT` | `ReembolsoEnviarRequest` (3 booleanos) | Expediente |
| 7 | GET | `` | clase | `estado`, `search`, `fechaDesde`, `fechaHasta`, `page`, `size` | Página |
| 8 | GET | `/{id}` | clase | — | Expediente completo |
| 9 | GET | `/{id}/documentos` | clase | — | Metadatos |
| 10 | GET | `/{id}/documentos/{docId}` | clase | — | **Archivo** |
| 11 | GET | `/{id}/historial` | clase | — | Trazabilidad |
| 12 | GET | `/{id}/comprobante` | clase | — | **PDF del comprobante** |

**Puntos de interés.** (1) toma un `policyNo` del llamador y pasa por `PolicyAccessGuard`, el
mismo control que remedió VUL-001 — es el candidato natural a BOLA. Del (3) al (12) toman un
`{id}` ajeno y se autorizan contra el dueño. `crear` y `listar` no reciben identificadores de
terceros: `listar` filtra por el agente del token (salvo `ROLE_ADMIN`, que ve todo).

### 2.2 Portal de clientes y apps — `/api/v1/mis-reembolsos`

Clase: `@PreAuthorize("hasAuthority('ROLE_AFFILIATE')")`. **No hay verbos de escritura**, por diseño.

| Método | Ruta | Parámetros | Devuelve |
|---|---|---|---|
| GET | `` | `estado`, `search`, `page`, `size` | Página, filtrada por el id del token |
| GET | `/{id}` | — | Expediente completo |
| GET | `/{id}/documentos` | — | Metadatos |
| GET | `/{id}/documentos/{docId}` | — | **Archivo** |
| GET | `/{id}/historial` | — | Trazabilidad |
| GET | `/{id}/comprobante` | — | **PDF del comprobante** |

### 2.3 Backoffice — `/api/v1/backoffice/reembolsos`

Clase: `ROLE_ADMIN, ROLE_REEMBOLSOS, ROLE_REEMBOLSOS_OPS, ROLE_CALL_CENTER`.

| Método | Ruta | Rol efectivo | Notas |
|---|---|---|---|
| GET | `` | clase | Toda la cartera. Filtros: `agenteId`, `estado`, `search`, fechas, paginación |
| GET | `/agentes` | clase | Nombres de agentes con reembolsos |
| GET | `/{id}` | clase | Expediente completo |
| GET | `/{id}/documentos` | clase | Metadatos |
| GET | `/{id}/documentos/{docId}/archivo` | **ADMIN, REEMBOLSOS, OPS** | **Archivo** — sin call center |
| GET | `/{id}/historial` | clase | Trazabilidad |
| GET | `/{id}/documentos-faltantes` | clase | Obligatorios pendientes |
| PATCH | `/{id}/estado` | ADMIN, REEMBOLSOS | Transición + observaciones |
| PATCH | `/{id}/documentos/{docId}` | ADMIN, REEMBOLSOS | Aprueba / rechaza un adjunto |
| POST | `/{id}/notas` | ADMIN, REEMBOLSOS | Nota privada o respuesta pública al ticket |
| POST | `/{id}/reabrir` | **ADMIN, OPS** | Único retroceso de estado |
| POST | `/{id}/pago` | **ADMIN, OPS** | Importe, moneda, fecha, referencia |
| POST | `/{id}/comprobante` | **ADMIN, OPS** | **multipart**, solo PDF |
| GET | `/{id}/comprobante` | **ADMIN, REEMBOLSOS, OPS** | **PDF** — sin call center |
| GET | `/{id}/freshdesk` | clase | Diagnóstico de sincronización |
| POST | `/{id}/freshdesk/reintentar` | ADMIN, REEMBOLSOS | Reencola eventos en error |

> En Spring Security 6 la anotación de método **sustituye** a la de clase, no se suman. Vale la
> pena verificarlo endpoint por endpoint en lugar de asumir la intersección.

### 2.4 Catálogos

| Ruta | Rol | Notas |
|---|---|---|
| `GET /api/v1/catalogos/reembolsos/{bancos,tipos-documento,estados,tipos-atencion,lugares-accidente,tipos-cuenta,monedas,tipos-documento-identidad,departamentos,tipos-gestor}` | ADMIN, REEMBOLSOS, OPS, CALL_CENTER, AGENT, **AFFILIATE** | Catálogos cerrados |
| `GET /api/v1/catalogos/reembolsos/gestores` | ídem | ⚠️ Devuelve **id y nombre de todos los agentes habilitados**, abierto también a afiliados |
| `/api/v1/backoffice/catalogos/reembolsos/**` | `ROLE_ADMIN` | ABM de bancos y del mapeo de estados de Freshdesk |

---

## 3. Integración con Freshdesk

Freshdesk es la herramienta con la que el área de reembolsos de BISA **opera** el trámite. Al
enviarse un expediente se crea un ticket, y el ciclo de vida se sincroniza en ambos sentidos.

### 3.1 Qué datos salen de la plataforma

Esto es lo primero que conviene registrar en el informe: **hay PHI que sale hacia un SaaS de
terceros.** Es una decisión de negocio, no un defecto, pero debe quedar documentada.

**En el cuerpo del ticket (HTML):** número de reembolso, nombre y documento del asegurado,
número y estado de la póliza, **diagnóstico clínico**, fechas de síntoma y consulta, tipo de
atención, hospitalización, accidente y si hubo terceros, otra aseguradora, detalle de cada
atención (proveedor, factura, fecha, descripción, importe), correo y teléfono de contacto,
banco, tipo y moneda de cuenta, nombre y documento del titular, **IP y hora de las
declaraciones juradas**, y enlaces profundos a los portales.

**El número de cuenta va enmascarado** (`cuentaMascarada`); es el único campo que se ofusca.
El diagnóstico **no** se enmascara.

**En campos personalizados (`cf_*`), si están configurados:** número de reembolso, póliza,
nombre y documento del asegurado, código de asegurado, tipo de atención, canal de origen,
monto solicitado, servicio de salud y URL del backoffice.

**Como adjuntos:** los documentos del expediente — informe médico, receta, epicrisis, facturas.

**Como solicitante del ticket:** `correoContacto` del formulario (ver §6, riesgo aceptado),
salvo que `forceRequesterEmail` esté configurado.

### 3.2 Salida — patrón outbox

Nada se envía a Freshdesk dentro de la transacción del usuario: se encola en
`reembolso_freshdesk_eventos` y un job lo drena.

| Evento | Se encola cuando |
|---|---|
| `CREATE_TICKET` | El agente envía el expediente. **Idempotente**: no duplica si ya hay ticket o un evento pendiente |
| `UPLOAD_ATTACHMENTS` | Se adjunta un documento estando en `PENDIENTE_INFO`. Idempotente |
| `ADD_NOTE` / `ADD_REPLY` | Nota privada o respuesta pública desde el backoffice |
| `UPDATE_STATUS` | Cambio de estado que tiene mapeo configurado |

- Drenado: `@Scheduled` cada **60 s** (`outbox-interval-ms`), lotes de 50. Un fallo no arrastra
  al resto del lote.
- Reintentos: backoff exponencial `2^intentos × 30 s`, tope 1 h, hasta **8 intentos**
  (`outbox-max-attempts`). Agotados, el evento pasa a `ERROR` y el reembolso a
  `freshdesk_sync_estado = ERROR`.
- Los adjuntos se reparten en lotes de **20 MB** por petición (`FreshdeskAttachmentChunker`).

**Salientes:** `POST /api/v2/tickets`, `GET /api/v2/tickets/{id}`, `GET /api/v2/tickets`,
`GET /api/v2/search/tickets`, `POST /api/v2/tickets/{id}/notes`,
`POST /api/v2/tickets/{id}/reply`, `PUT /api/v2/tickets/{id}`.
Autenticación: **Basic `apiKey:X`**, calculada una vez y nunca registrada en el log.

### 3.3 Entrada A — polling

`@Scheduled` cada **5 min** (`poll-interval-ms`, lotes de 100) consulta los tickets abiertos y
refleja el estado. Es la red de seguridad: si el webhook falla o se pierde, el estado converge
igual en el siguiente ciclo.

### 3.4 Entrada B — webhook  🔴 **el único endpoint del módulo sin JWT**

```
POST /api/v1/public/webhooks/freshdesk/tickets
X-Freshdesk-Webhook-Secret: <64 hex>
Content-Type: application/json

{ "ticketId": 4321, "status": "Closed" }
```

| Código | Significado |
|---|---|
| 202 | Aplicado |
| 204 | El ticket no corresponde a un reembolso, o Freshdesk ya no lo devuelve |
| 401 | Falta la cabecera o el secreto no coincide |
| 503 | Integración apagada, secreto sin configurar, o fallo interno |

**Controles que ya tiene** — verificarlos, no asumirlos:

- **Cerrado por defecto.** Sin `FRESHDESK_WEBHOOK_SECRET` responde 503 a todo: una variable de
  entorno sin definir no puede dejarlo abierto.
- **El `status` del cuerpo no se usa.** Solo se lee `ticketId`; el estado se **re-consulta** a
  Freshdesk con `GET /api/v2/tickets/{id}`. Quien tenga el secreto no puede forzar una
  transición: lo peor que consigue es una resincronización contra el dato real.
- **Regla de solo avance**, también para lo que llega de Freshdesk.
- **Un `status` sin mapeo no se adivina**: se guarda la etiqueta y no se toca el estado interno.
- Comparación del secreto con `MessageDigest.isEqual` sobre bytes UTF-8 — sin cortocircuito ni
  por contenido ni por longitud.
- Dentro del bucket `WEBHOOK` de `RateLimitFilter`: **30 peticiones/min por IP**, 429 con
  `Retry-After`.
- Cada 401 escribe `log.warn` y una fila de auditoría (`ACCESO_DENEGADO`, canal `INTEGRACION`).

**Lo que el webhook no tiene, y es deliberado:** no hay firma HMAC del cuerpo, ni timestamp, ni
nonce, ni protección de replay, ni allowlist de IPs. El disparador de Freshdesk no sabe firmar,
así que el modelo de confianza es secreto-en-cabecera sobre TLS. Dado que el cuerpo no se
utiliza para decidir nada, un replay es inocuo. **Confirmarlo es un buen caso de prueba.**

`WalletVerifyController` (`POST /api/v1/public/wallet/verify`) usa el **mismo esquema** de
secreto compartido en cabecera: un hallazgo en uno aplica al otro.

### 3.5 Configuración y secretos

Todo bajo `app.integration.freshdesk.*`. Los dos secretos:

| Variable | Uso |
|---|---|
| `FRESHDESK_API_KEY` | Basic auth saliente hacia Freshdesk |
| `FRESHDESK_WEBHOOK_SECRET` | Clave compartida del webhook entrante |

`FRESHDESK_ENABLED=false` activa `FreshdeskDisabledClient`: los trámites se registran igual y
quedan marcados como `DESHABILITADO`. **Es la forma recomendada de probar sin tocar el tenant
real** (ver §9).

---

## 4. Validación de entrada

### 4.1 Campos del formulario

Cubiertos con Bean Validation: `@NotNull` en enums y booleanos, `@NotBlank` + `@Size` en cada
cadena, `@Email` en los dos correos, `@PastOrPresent` en las fechas, `@Size(max=20)` + `@Valid`
en las atenciones, `@Pattern("^\\d{6,20}$")` en el número de cuenta,
`@Pattern("^[A-Za-z0-9\\-/]{1,60}$")` en el número de factura, `@DecimalMin`/`@Digits` en importes.

Reglas cruzadas en `ReembolsoValidator`: CI `^\d{5,10}$`, documento extranjero
`^[A-Za-z0-9-]{4,20}$`, orden de fechas, facturas duplicadas, campos condicionalmente
obligatorios o prohibidos.

**Texto libre acotado solo por longitud** — los candidatos a XSS almacenado y a inyección en el
ticket: `diagnostico` (4000), `sintomasAnterioresDetalle` (1000), `hospitalNombre` (200),
`lugarAccidenteOtro` (200), `otraAseguradoraNombre` (200), nombres del asegurado y del titular,
`telefonoContacto` (50, **sin patrón**), `proveedorNombre` (200), `descripcionServicio` (1000),
`moneda` y `cuentaMoneda` (10, **sin lista cerrada**), y desde el backoffice `mensaje` (4000),
`observaciones` (2000), `referencia` (100), `motivo` (2000).

Del lado servidor, las cuatro plantillas Thymeleaf usan **solo `th:text`, cero `th:utext`**, así
que el escapado es correcto al renderizar. El riesgo vive en los clientes que muestran esos
campos y en el cuerpo HTML del ticket de Freshdesk.

### 4.2 Subida de archivos

Es la validación más estricta de la plataforma. Vale la pena intentar romperla:

- Máximo **10 archivos** por expediente.
- **El lote entero se valida antes de escribir el primer byte** (nada de cargas parciales).
- Reglas **por tipo de documento, leídas de la base** (`catalogo_tipos_documento_reembolso`):
  extensiones permitidas y tamaño máximo. Ej.: factura pdf/jpg/jpeg/png 5 MB; informe médico
  solo PDF; receta 3 MB.
- **Comprobación de magic bytes**: `%PDF`, `FF D8 FF` (JPEG), `89 50 4E 47` (PNG). Una imagen
  renombrada a `.pdf` se rechaza.
- **El `Content-Type` del multipart no se persiste.** El tipo con el que se sirve el archivo se
  deriva de la extensión ya validada, y se normaliza otra vez al emitirlo.
- Nombre en disco reemplazado por **UUID**; el original nunca toca el sistema de archivos.
  Rechazo de `..`, `/` y `\`, más una aserción de contención dentro del directorio raíz.
- `Content-Disposition` saneado (comillas y caracteres de control) y con `filename*=UTF-8''`.
- Solo se admite adjuntar en `BORRADOR` y `PENDIENTE_INFO`.
- Límites de servlet: 6 MB por archivo, 60 MB por petición.

El comprobante de pago del backoffice pasa por el mismo control de cabecera, y además exige
extensión `.pdf` y `Content-Type` coherente cuando el cliente lo envía.

### 4.3 Inyección SQL

Todas las consultas son derivadas de Spring Data o `@Query` con parámetros nombrados. Hay **una
sola consulta nativa** (`findMaxSequenceByPrefix`) y su parámetro se construye en el servidor a
partir de la fecha, nunca del cliente. **No hay ordenamiento dinámico**: el `Sort` está fijo en
código, así que no existe la superficie clásica de inyección por nombre de propiedad. La
paginación está acotada a `size ≤ 100` y `page ≥ 0`.

---

## 5. Casos de prueba sugeridos

Ordenados por dónde esperaríamos encontrar algo, no por facilidad.

### Autorización

1. **IDOR horizontal, agentes.** Con el token del agente A, recorrer los 10 endpoints `{id}` del
   §2.1 usando el id de un expediente del agente B. Esperado: **403** en todos, incluidos los
   sub-recursos (`/documentos`, `/documentos/{docId}`, `/historial`, `/comprobante`).
2. **IDOR horizontal, asegurados.** Lo mismo en `/api/v1/mis-reembolsos/**` entre dos afiliados.
3. **Oráculo de existencia.** Comparar la respuesta de un `{id}` inexistente con la de uno ajeno:
   deben ser **indistinguibles** (mismo código y mismo cuerpo) para agentes y asegurados.
4. **Sub-recurso cruzado.** Un `docId` que existe pero pertenece a **otro** expediente, contra un
   `{id}` propio. Esperado: 404/403, nunca el archivo.
5. **Borrador ajeno.** Con el token del asegurado, pedir un expediente en `BORRADOR`. Esperado: 403.
6. **Escalada vertical en backoffice** — el caso central. Con **una cuenta por rol**:
   - `ROLE_CALL_CENTER`: debe poder listar y ver el detalle; debe recibir **403** en
     `/{id}/documentos/{docId}/archivo` y en `/{id}/comprobante`, y en toda escritura.
   - `ROLE_REEMBOLSOS`: 403 en `reabrir`, `pago` y `comprobante`.
   - `ROLE_REEMBOLSOS_OPS`: 403 en `estado`, `notas` y `revisarDocumento`.
7. **BOLA en la validación de póliza.** `GET /reembolsos/validacion-poliza/{policyNo}` con una
   póliza fuera de la cartera del agente. Esperado: **403**; **503** si BISA no responde (fallo
   cerrado, no permisivo); **401** si el agente no trae credenciales federadas.
8. **Rutas del backoffice sin sesión del rol.** `/reembolsos` y `/reembolsos/:id` en el portal:
   confirmar que el guard de rol redirige y que el 403 del servidor sostiene el control aunque
   se fuerce la navegación.

### Subida y descarga de archivos

9. **Políglota.** PDF que empieza por `%PDF-` y contiene `<script>`, subido declarando
   `Content-Type: text/html`. Descargarlo y mirar la cabecera: debe volver como
   `application/pdf`, nunca como `text/html`.
10. **SVG.** Extensión `.png` con contenido SVG, y `.svg` a secas. Ambos deben caer.
11. **Nombre hostil.** `x";attachment;filename="y.html.pdf`, y otro con CRLF embebido.
    Comprobar que el `Content-Disposition` no se parte ni gana parámetros.
12. **Recorrido de rutas.** `../../etc/passwd.pdf` como nombre de archivo.
13. **Estado.** Adjuntar a un expediente en `REGISTRADO`, `APROBADO` y `PAGADO`. Esperado: rechazo.
14. **Límites.** Archivo 11; archivo por encima del tope del tipo; lote de 10 × 6 MB (justo en el
    tope de petición); lote con un archivo inválido en la última posición — no debe escribirse
    **ninguno**.
15. **Comprobante.** PNG renombrado a `.pdf`, con y sin `Content-Type`.

### Webhook y integración

16. **Fuerza bruta del secreto.** 35 intentos seguidos: 401 hasta agotar el bucket y luego **429**
    con `Retry-After`. Confirmar además que aparecen filas `ACCESO_DENEGADO` con canal
    `INTEGRACION` en el módulo de auditoría del backoffice.
17. **Oráculo de longitud.** Secretos de 1 y de 64 caracteres deben dar la misma respuesta, sin
    diferencia de tiempo medible.
18. **Confianza en el cuerpo.** Con el secreto válido, enviar un `status` que no corresponde al
    real del ticket. El estado interno **no** debe cambiar según el payload.
19. **Retroceso.** Provocar un mapeo que intente retroceder el estado: debe ignorarse.
20. **Replay.** Reenviar el mismo aviso N veces: sin efectos acumulativos.
21. **Enumeración de tickets.** Con el secreto, distinguir 202 de 204 permite mapear qué ids de
    ticket pertenecen a reembolsos. Valorar el impacto.

### Datos y otros

22. **Enumeración de usuarios.** `gestorAgenteId` en el alta apuntando a ids arbitrarios: no debe
    revelar nombres de cuentas que no sean agentes activos, ni distinguir «no existe» de
    «existe pero no es agente».
23. **`/catalogos/reembolsos/gestores` como afiliado** — lista de personal comercial.
24. **XSS almacenado.** Payload en `diagnostico`, `descripcionServicio`, `proveedorNombre` y en
    `observaciones`/`mensaje` del backoffice; revisar cómo se renderiza en los tres portales,
    en las apps y **en el cuerpo del ticket de Freshdesk**.
25. **Paginación.** `size=100000` y `page=-1` en las dos listas.
26. **Verbosidad de errores.** `?trace=true`, cuerpos malformados, tipos inesperados en enums.
27. **CORS.** `OPTIONS` con un `Origin` arbitrario sobre `/api/v1/reembolsos`.
28. **Cabeceras.** Verificar la CSP tanto en `/` como en un asset `.js` con caché — en nginx las
    cabeceras no se heredan en un `location` que declara las suyas, y ese fallo solo se ve con
    `curl -I`.

---

## 6. Riesgos aceptados y decisiones de negocio

Declarados por adelantado para no gastar tiempo en discutirlos como hallazgos nuevos. Se
pueden reportar, pero ya están identificados.

| # | Situación | Estado |
|---|---|---|
| 1 | **El backoffice ve toda la cartera sin comprobación por registro.** Es el modelo: operaciones necesita cualquier expediente | Por diseño |
| 2 | **PHI hacia un tercero.** El diagnóstico clínico y los adjuntos médicos viajan a Freshdesk (SaaS). El número de cuenta va enmascarado; el diagnóstico no | Decisión de negocio. Freshdesk es la herramienta con la que BISA opera el trámite |
| 3 | **`correoContacto` es texto libre del agente** y no se coteja contra el titular de la póliza. Los avisos (nombre del asegurado, póliza, importe pagado) van a esa dirección, que además queda como solicitante del ticket | **Riesgo aceptado, pendiente de decisión.** Ya se quitó el diagnóstico de los correos y se eliminó el enrutado de notificaciones push por ese campo |
| 4 | **Sin bloqueo de cuenta persistente.** Solo hay limitación por IP, y el estado es por instancia: N réplicas multiplican el límite efectivo | Abierto desde la ronda de julio |
| 5 | **El webhook no firma el cuerpo** (HMAC, timestamp, nonce) | Limitación del disparador de Freshdesk. Mitigado porque el cuerpo no decide nada |
| 6 | **Política de contraseñas solo en cliente** (VUL-010) | Abierto desde julio |
| 7 | **Magic bytes solo en reembolsos.** Siniestros, cartas de garantía y comprobantes SWIFT siguen aceptando cualquier contenido con extensión permitida | Mitad de VUL-012/013, todavía abierta. **Fuera de este alcance**, pero conviene contrastarlo |

---

## 7. Controles ya implementados

Para que la verificación sea de confirmación y no de descubrimiento. Todos tienen prueba
automatizada asociada; el detalle está en `remediacion-ethical-hacking.md`, secciones 9-13.

| Control | Dónde |
|---|---|
| Verificación de propiedad única para el expediente **y todos sus sub-recursos** | `ReembolsoService.requireAcceso` |
| Autorización de póliza reutilizando el guard de VUL-001 | `ReembolsoPolicyValidator` → `PolicyAccessGuard` |
| Id inexistente e id ajeno indistinguibles | ídem |
| MFA obligatoria para los roles del módulo | `MfaPolicy` + `BackofficeRoles.ALL` |
| Rate limiting del webhook, 30/min/IP | `RateLimitFilter`, bucket `WEBHOOK` |
| Comparación de secretos en tiempo constante | `MessageDigest.isEqual` |
| Auditoría de toda denegación de acceso | `GlobalExceptionHandler` → `AuditService` |
| MIME derivado de la extensión validada, no del cliente | `ContenidoArchivo` |
| Magic bytes en documentos y en el comprobante | ídem |
| `Content-Disposition` saneado, RFC 5987 | ídem |
| Paginación acotada, sin `sort` del cliente | `ReembolsoService.pageable` |
| CSP, `X-Frame-Options`, `nosniff`, `Referrer-Policy`, `Permissions-Policy` | `nginx.conf` de los tres portales |
| CORS por perfil, sin comodín ni `localhost` en prod | `SecurityConfig` |
| Sin trazas ni OpenAPI en `prod` ni en `staging` | `application-{prod,staging}.yml` |
| IP de la declaración jurada consciente de proxy | `ClientIpResolver` |

---

## 8. Qué necesitamos preparar antes de empezar

- [ ] **Cuentas de backoffice con un solo rol cada una**: `ROLE_REEMBOLSOS`,
      `ROLE_REEMBOLSOS_OPS` y `ROLE_CALL_CENTER` por separado. Con una cuenta `ROLE_ADMIN` no se
      puede probar la escalada vertical, que es la prueba central de esta superficie.
- [ ] **Acceso al buzón de correo de cada cuenta**: la MFA es obligatoria para todos los roles de
      backoffice y el OTP llega por correo.
- [ ] **Dos agentes distintos, cada uno con cartera propia**, para el IDOR horizontal.
- [ ] **Dos afiliados distintos**, cada uno con al menos un reembolso enviado.
- [ ] **Expedientes en varios estados** (`BORRADOR`, `REGISTRADO`, `PENDIENTE_INFO`, `APROBADO`,
      `PAGADO`), para los casos 13 y 19.
- [ ] **El secreto del webhook** para los casos 18-21, o confirmación de que solo se prueba la
      autenticación.
- [ ] Definir si el entorno de pruebas corre con `FRESHDESK_ENABLED=false` o contra un tenant de
      pruebas (ver §9).

Las credenciales se entregan **por canal seguro**, no en este documento, y se rotan al cerrar
el engagement.

---

## 9. Reglas de la prueba

- **No disparar el webhook contra el tenant real de Freshdesk.** Contamina la cola de soporte de
  BISA. Dos opciones: `FRESHDESK_ENABLED=false` (el endpoint responde 503 y el trámite se
  registra igual, marcado como `DESHABILITADO`), o el marcado de ticket de prueba
  (`FRESHDESK_SUBJECT_PREFIX='[PRUEBA - NO GESTIONAR]'` y `FRESHDESK_STATUS=5`, que hace nacer
  cerrados los tickets de ensayo).
- **La creación masiva de expedientes tiene coste externo.** Cada alta consulta la API de BISA y
  cada envío crea un ticket real. `POST /api/v1/reembolsos` no está limitado por
  `RateLimitFilter` (solo cubre autenticación y webhooks), así que la moderación es de ustedes.
- **Los límites que van a encontrar:** identidad 5/min/IP, login y `verify-otp` 10/min/IP,
  reenvío de OTP 3/5min/IP, webhooks 30/min/IP. Todos devuelven **429** con `Retry-After`.
- **Fallos por indisponibilidad de BISA** se responden con **503**, no 403. Un pico de 503 es una
  caída del proveedor, no un control de permisos: conviene distinguirlo en el informe.
- Avisar antes de cualquier prueba de carga o de denegación de servicio.

---

## 10. Documentos relacionados

| Documento | Contenido |
|---|---|
| `endpoints-api-ethical-hacking.md` | Inventario completo de la API, todos los módulos |
| `endpoints-sensibles-eh.md` | Alcance priorizado por sensibilidad; reembolsos en la sección 11 |
| `remediacion-ethical-hacking.md` | Hallazgos de las dos rondas y su remediación, con re-test |
| `freshdesk-webhook-reembolsos.md` | Configuración operativa del webhook, del lado de BISA |
| `Reembolsos/CONTRATO-REEMBOLSOS.md` | Contrato funcional entre las cinco aplicaciones |
