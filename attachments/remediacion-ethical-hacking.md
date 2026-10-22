# Remediación Ethical Hacking — Atlas Global Health

**Alcance de este documento:** dos rondas.

- **Secciones 1 a 8** — qué se corrigió de los hallazgos CRÍTICOS y ALTOS del informe de
  Ethical Hacking (julio 2026, Weimar Jhonathan Gonzales Choque), qué queda pendiente de
  confirmar antes de desplegar, y cómo verificar punto por punto que cada hallazgo quedó
  cerrado.
- **Secciones 9 a 13** — segunda ronda (agosto 2026), sobre el módulo de **Reembolsos**:
  verificación de que la ronda anterior no se rompió, auto-auditoría del módulo nuevo, y el
  re-test correspondiente. Empieza en «Segunda ronda — módulo de Reembolsos».

**Estado:** implementado y verificado en código. **No desplegado.**

**Confidencial** — contiene detalle de vulnerabilidades y pasos de explotación.

---

## 1. Resumen

| ID | Sev | Hallazgo | Estado |
|---|---|---|---|
| VUL-001 | 🔴 CRÍTICA | BOLA/IDOR en endpoints de pólizas | ✅ Corregido |
| VUL-002 | 🔴 CRÍTICA | BOLA/IDOR en pagos SWIFT | ✅ Corregido |
| VUL-003 | 🔴 CRÍTICA | Oráculo de identidad + account takeover vía Google | ✅ Corregido |
| VUL-004 | 🟠 ALTA | `bisaToken` expone IP interna y email corporativo | ✅ Corregido |
| VUL-005 | 🟠 ALTA | MFA no obligatorio para ningún rol | ✅ Corregido (roles de backoffice) |
| VUL-006 | 🟡 MEDIA | IDOR en historial de siniestros | ✅ Corregido (extra) |
| VUL-009 | 🟡 MEDIA | Sin rate limiting en login | ✅ Corregido (extra) |
| VUL-016 | 🔵 BAJA | Actuator público | ✅ Corregido (extra) |

Cerrados después, en la segunda ronda (ver sección 10):

| ID | Sev | Hallazgo | Estado |
|---|---|---|---|
| VUL-017/018 | 🔵 BAJA | CSP / cabeceras de seguridad en los portales | ✅ Corregido (W-05) |
| VUL-012/013 | 🟡 MEDIA | Validación de uploads | 🟡 Parcial — reembolsos sí; siniestros, cartas y SWIFT no |
| VUL-015 | 🔵 BAJA | Errores verbosos | ✅ Corregido (R-15) — con un matiz que la sección 10 precisa |

Verificación automatizada: backend **902 tests, 0 fallos** (87 nuevos, escritos
específicamente para probar estos hallazgos). Detalle de la verificación por componente en
la sección 7.

### Dos correcciones al diagnóstico del informe

Vale registrarlas porque cambian cómo se debe re-testear:

**1. La asimetría de validación de fecha de nacimiento entre hosts no existía.** El informe
concluye que `servcon` validaba la fecha y `bisa-api` no, y recomienda "igualar el
comportamiento". En realidad era **un solo bug de código**: `verifyAffiliateIdentity` tenía
un *early return* que, si el CI ya existía en la BD local con contraseña, devolvía `200 OK`
con el nombre completo **sin mirar la fecha**. `servcon` "sí validaba" solo porque ese CI no
estaba aún registrado en su base. Con una BD poblada, los 5 hosts se comportaban igual.

→ **Implicancia para el re-test:** hay que probar con un CI **ya registrado** en la BD del
host. Con un CI nuevo el comportamiento vulnerable no se reproduce.

**2. El backoffice no consumía `/api/v1/policies/**`.** Ya estaba migrado a un espejo propio
en `/api/v1/backoffice/call-center/**`, con `@PreAuthorize` correcto. El único acceso cruzado
legítimo que había que preservar era el del **portal de agentes** entrando a la póliza de un
cliente de su cartera.

---

## 2. Qué se corrigió

### VUL-001 y VUL-002 — BOLA/IDOR en pólizas y pagos SWIFT (CRÍTICAS)

**El problema:** todos los datos de póliza se piden a la API corporativa de BISA con un
**token de cuenta de servicio**, por lo que cualquier `policyNo`, `partyCode`, `documento` o
`idPrint` que llegara en la URL funcionaba. `PolicyController` no tenía **ni un solo**
`@PreAuthorize`, y `PagoSwiftController` tenía gate de rol pero ninguna verificación de que
la póliza fuera del solicitante.

**La corrección:** un componente centralizado `PolicyAccessGuard` que resuelve el conjunto de
recursos permitidos para el usuario autenticado y lanza `AccessDeniedException` (→ **403**) si
el identificador pedido no está en él.

El eje de decisión es la **capacidad**, no el rol —así sigue siendo correcto para usuarios con
más de un rol, y las cuentas heredadas `ROLE_USER` (sin `partyCode`) caen en denegado:

| Capacidad del usuario | Alcance permitido |
|---|---|
| `ROLE_ADMIN` | sin restricción, sin llamada a BISA |
| tiene `partyCode` (afiliado) | solo su propia póliza y su `idPrint` |
| tiene `bisaThirdPartyCode` + `bisaToken` (agente) | toda su cartera |
| ninguna | denegado |

Endpoints protegidos:

| Endpoint | Verificación |
|---|---|
| `GET /api/v1/policies/{policyNo}/detail` | `policyNo` |
| `GET /api/v1/policies/deductibles/{policyNo}` | `policyNo` |
| `GET /api/v1/policies/coverages/{policyNo}/{node}` | `policyNo` |
| `GET /api/v1/policies/invoices/{policyNo}` | `policyNo` |
| `GET /api/v1/policies/print/{idPrint}` | `idPrint` |
| `GET /api/v1/policies/user/profile/{documento}` | documento propio o admin |
| `GET /api/v1/policies/data/{partyCode}` | partyCode propio o admin |
| `POST /api/v1/policies/{policyNo}/pagos-swift` | `policyNo` |
| `GET /api/v1/policies/{policyNo}/pagos-swift/periodo-actual` | `policyNo` |
| `GET /api/v1/policies/{policyNo}/pagos-swift/cuotas` | `policyNo` |
| `GET /api/v1/policies/{policyNo}/pagos-swift/latest` | `policyNo` |
| `GET /api/v1/policies/{policyNo}/pagos-swift` (listado) | `policyNo` |
| `GET /api/v1/policies/me/data` | sin cambios — ya derivaba el `partyCode` del JWT |

**Decisión de diseño relevante:** el informe/plan sugería *eliminar* los 4 endpoints sin
consumidores (`user/profile`, `data/{partyCode}`, `pagos-swift/latest` y el listado). Se
optó por **protegerlos en lugar de borrarlos**: las builds móviles ya publicadas
(Android 1.0.11, iOS 1.0.7) están fuera de la visibilidad del repositorio y podrían
llamarlas. Proteger cierra la vulnerabilidad igual y no rompe una app que ya está en manos de
usuarios.

**Caché y disponibilidad.** El guard agregaría un round-trip a BISA por request (la pantalla
de detalle del agente dispara 1 llamada de detalle + N de coberturas). `PolicyOwnershipCache`
resuelve esto con tres niveles:

- **Fresco (3 min):** se sirve del caché.
- **Nunca deniega con datos cacheados:** si el identificador no está en el snapshot, fuerza
  **una** recarga antes de responder 403 — así una póliza renovada (cambia el `policyNo`) o
  recién asignada a una cartera es visible de inmediato. Esa recarga tiene *cooldown* de 30 s
  por sujeto, que es lo que impide que una enumeración convierta cada intento en una llamada a
  BISA.
- **Rancio (24 h), solo si BISA es inalcanzable:** ver abajo.

**Fail-closed, con 503 y no 403.** Si BISA no responde y no hay snapshot usable, se deniega
con **503** — no probamos una denegación, y confundir una caída con un problema de permisos
entrena a soporte a diagnosticar mal. Caso aparte: agente sin `bisaToken` (sesión expirada)
→ **401**.

El nivel rancio de 24 h existe por una razón concreta: `PagoPeriodoResolver` degrada a
propósito en lugar de bloquear, para que una caída de BISA no impida subir el comprobante
SWIFT. Un fail-closed ingenuo habría revertido esa decisión de producto.

**Archivos:** `security/PolicyAccessGuard.java` (nuevo),
`security/PolicyOwnershipCache.java` (nuevo), `controller/PolicyController.java`,
`controller/PagoSwiftController.java`.

**No se agregó verificación dentro de `PolicyDetailService`** a propósito:
`CallCenterController` comparte ese bean y lo invoca legítimamente sin contexto de propiedad,
bajo su propio gate de rol. Cualquier assert de propiedad en el servicio dejaría el detalle de
póliza del call center en 403. El controller es el único límite correcto.

---

### VUL-003 — Oráculo de identidad y account takeover (CRÍTICA)

Tres correcciones independientes.

**a) Se eliminó el oráculo.** Se borró el early return que devolvía el nombre completo sin
validar la fecha. Ahora la verificación contra BISA se ejecuta **siempre**, y el mensaje "su
cuenta ya está configurada" se emite **después** de validar.

Además se **unificaron todas las respuestas de rechazo** a un único mensaje genérico,
`"Datos de identidad inválidos"`. Antes se distinguía `"No se encontró perfil para este
documento"` (CI inexistente) de `"Credenciales inválidas"` (CI válido, fecha incorrecta) —
un oráculo más fino que confirmaba la existencia de un CI sin conocer la fecha. La distinción
queda solo en el log del servidor y la auditoría.

También se hizo tolerante el parseo de fecha (`1990-1-15` ya no se rechaza frente al
`1990-01-15` de BISA), que antes podía rechazar usuarios legítimos.

**b) Se cortó la cadena de account takeover.** El ataque era: con CI + fecha de nacimiento
—ambos obtenibles por OSINT— `google-register` vinculaba la cuenta de Google del atacante al
perfil de la víctima y devolvía una sesión funcional, sin contraseña y sin aviso alguno.

La corrección introduce un **código de un solo uso enviado al email ya registrado del
afiliado**, que es el único factor que un atacante suplantando a alguien no puede recibir:

1. `verify-identity` genera el `setupToken` **y** un código de 6 dígitos que se envía al
   email del perfil BISA. La respuesta agrega `otpRequired` y `maskedEmail`.
2. **Nuevo** `POST /api/v1/auth/affiliate/verify-setup-otp` `{setupToken, otpCode}` → marca
   el token como verificado. Máximo 3 intentos; al agotarlos el token se quema.
3. **Nuevo** `POST /api/v1/auth/affiliate/resend-setup-otp` `{setupToken}`.
4. `set-password`, `link-google` y `google-register` **rechazan** si el token no tiene el
   código confirmado.
5. `google-register` además **rechaza cuentas que ya tienen contraseña o Google vinculado**
   (antes sobreescribía `google_id` sin ninguna condición — eso es lo que permitía secuestrar
   una cuenta activa) y exige que el `setupToken` pertenezca a ese documento.
6. **Notificación al titular** por email cada vez que se establece una contraseña o se
   vincula Google. Es lo que convierte un takeover silencioso en un incidente detectable.

**De dónde sale el email — punto crítico.** `createAffiliateUser` usaba un email sintético
`{documento}@affiliate.bisaseguros.com` cuando BISA no tenía uno. El gate lee el email del
**perfil fresco de BISA**, no el local, y si no hay uno real **bloquea el registro** con un
mensaje que deriva al call center. Ver sección 4.1: esto es un bloqueante operativo.

**c) Rate limiting** en `verify-identity` — ver VUL-009.

**Archivos:** `service/AffiliateAuthService.java`, `controller/AuthAffiliateController.java`,
`model/AffiliateSetupToken.java`, `integration/notification/EmailService.java`,
`model/dto/AffiliateSetupOtpRequest.java` (nuevo), `AffiliateSetupOtpResendRequest.java`
(nuevo), `GoogleRegisterRequest.java`, `AffiliateIdentityVerifyResponse.java`,
migración `db/changelog/changes/035-affiliate-setup-otp.sql` (nueva).

---

### VUL-004 — `bisaToken` en la respuesta de login (ALTA)

Se eliminó el campo de `BisaLoginResponse` y `LoginResponse`. Se sigue persistiendo en BD:
los ~10 consumidores del backend leen `user.getBisaToken()` de la base, nunca del request, y
ningún cliente lo enviaba de vuelta.

**Bug adyacente corregido en el mismo cambio:** `BisaAuthService` asignaba `.accessToken()`
**dos veces** en la ruta post-OTP, y la segunda ganaba — la respuesta del agente con 2FA salía
con el literal `"Bearer"` como access token. **El login de agente con 2FA estaba roto.**
Relevante porque si mañana se extiende el MFA obligatorio a agentes, los habría dejado sin
poder entrar.

**Archivos:** `model/dto/BisaLoginResponse.java`, `model/dto/LoginResponse.java`,
`integration/bisa/BisaAuthService.java`.

---

### VUL-005 — MFA obligatorio (ALTA)

El mecanismo de 2FA funcionaba; faltaba obligatoriedad. Todas las rutas de creación de usuario
fijaban `twoFactorEnabled = false` y nada lo forzaba, así que un administrador con acceso a
toda la base de asegurados estaba protegido solo por contraseña.

Nuevo `security/MfaPolicy.java`: el paso de OTP es obligatorio para los **6 roles de
backoffice** (`ROLE_ADMIN`, `ROLE_SINIESTROS`, `ROLE_RENOVACIONES`, `ROLE_REVISION_PAGOS`,
`ROLE_CARTAS_GARANTIA`, `ROLE_CALL_CENTER`), independientemente del flag por usuario. Se
aplica en los **cuatro** servicios de login, para que la política valga por usuario y no por
endpoint.

Configurable sin recompilar, para extenderlo a agentes más adelante:

```yaml
app.security.mfa.mandatory-roles: ${MFA_MANDATORY_ROLES:}   # vacío = los 6 de backoffice
```

**Sin riesgo de bloqueo y sin cambios de frontend:** el segundo factor llega por email, así
que no hace falta enrolamiento previo, y backoffice-web ya tenía la pantalla de OTP
funcionando.

**Pendiente antes de extenderlo a agentes:** `TwoFactorService.enable2fa/disable2fa` exige
`passwordEncoder.matches(password, user.getPassword())`, y para agentes BISA y afiliados con
Google vinculado `password` es `null` — hoy **no pueden activar 2FA en absoluto**. No bloquea
esta fase (los usuarios de backoffice sí tienen contraseña).

#### Seguimiento posterior

Dos consecuencias que esta fase dejó abiertas y que ya están resueltas:

**El pendiente de los usuarios federados.** `POST /2fa/enable/initiate` + `/enable/confirm`
confirman con un OTP al correo en lugar de la contraseña, y `disable2fa` solo verifica la
contraseña de cuentas que tienen una. Con eso los agentes BISA pueden activar y desactivar su
2FA desde `Mi Perfil › Seguridad` en agentes-web.

**"Sin cambios de frontend" era cierto para el login, no para el perfil.** La política se
aplicó server-side, pero `GET /2fa/status` seguía devolviendo solo el flag del usuario, así que
la tarjeta de 2FA del perfil de backoffice-web mostraba "Deshabilitado" y ofrecía un botón de
desactivar que respondía 200 sin desactivar nada. Ahora `/2fa/status` devuelve también
`mandatory` (desde `MfaPolicy.isRequired`), `/2fa/disable/*` responde **409** para los roles
cubiertos, y el perfil muestra el estado real con el distintivo *Obligatorio* y sin botón de
desactivar. Un bug de tipado agravaba el síntoma: el frontend leía `twoFactorEnabled` donde el
JSON trae `enabled`, de modo que el estado era siempre `undefined`.

**Archivos:** `security/MfaPolicy.java` (nuevo), `service/AuthService.java`,
`service/BackofficeAuthService.java`, `service/AffiliateAuthService.java`,
`integration/bisa/BisaAuthService.java`.

---

### Extras incluidos

**VUL-006 — IDOR en sub-recursos de siniestros (MEDIA).** `GET /siniestros/{id}/historial`
recibía `Authentication` y la descartaba. Se corrigió, y también los tres endpoints de
documentos (`upload`, `list`, `download`) que tenían el mismo hueco y **no estaban en el
informe**. Los cuatro pasan ahora por el recurso padre, que sí valida propiedad.

Además, `SiniestroService.validateAccess` y `CartaGarantiaService.validateAccess` lanzaban
`RuntimeException` pelada, que el handler global mapeaba a **400** — de ahí el
"400 No tiene permisos" que vio el consultor. Ahora lanzan `AccessDeniedException` → **403**.

**VUL-009 — Rate limiting (MEDIA).** No existía ningún throttling ni bloqueo de cuenta en
todo el backend: `CustomUserDetailsService` fijaba `accountLocked(false)` de forma
incondicional, no había contador de intentos fallidos, y `resend-otp` permitía acuñar códigos
sin límite (reiniciando además el contador de `verify-otp`).

Nuevo `security/RateLimitFilter.java`, responde **429** con `Retry-After`:

| Grupo de rutas | Límite por defecto |
|---|---|
| `verify-identity`, `google-register`, `verify-setup-otp`, `set-password`, `forgot-password` | 5 / minuto / IP |
| `*/login`, `*/verify-otp` | 10 / minuto / IP |
| `*/resend-otp`, `*/resend-setup-otp` | 3 / 5 minutos / IP |

Lee la IP de `X-Forwarded-For` (los 5 hosts están detrás de nginx), y solo la respeta si el
peer inmediato es un proxy de confianza —si no, cualquiera podría rotar la cabecera para
evadir el límite. Configuración en `app.security.rate-limit.*`, incluida
`trusted-proxies`.

> ⚠️ **Configurar `trusted-proxies` con la IP de nginx antes de desplegar.** Si se deja
> vacío se confía en la cabecera de cualquier peer; correcto solo si la app es
> inalcanzable salvo a través del proxy.

**VUL-016 — Actuator (BAJA).** Solo `/actuator/health` queda público (lo usa el healthcheck
de Docker); el resto del árbol exige `ROLE_ADMIN`. Se limpiaron además tres `requestMatchers`
muertos que estaban sombreados por el comodín de `/api/v1/auth/**`.

---

## 3. Cambios que afectan a los clientes

### 3.1 El registro de afiliados tiene un paso nuevo

Éste es el cambio de comportamiento más importante. El flujo pasa de dos pasos a tres:

```
antes:  verify-identity  →  set-password / link-google
ahora:  verify-identity  →  verify-setup-otp  →  set-password / link-google
```

Sin ese paso intermedio, `set-password` empieza a rechazar y **el registro queda roto**. Por
eso se implementó también en los tres clientes:

| Cliente | Cambio |
|---|---|
| clientes-web | Nueva ruta `/verify-setup-otp` + componente; `verify-identity` redirige ahí; el flujo "Continuar con Google" ahora verifica identidad primero y termina con `link-google` |
| Android | Nueva pantalla `SetupOtpScreen` + ViewModel + ruta `Screen.SetupOtp`; `RegisterViewModel` y el flujo de Google enrutan ahí |
| iOS | Nueva `SetupOtpView` + ViewModel + `Route.setupOtp`; `VerifyIdentityViewModel` y `GoogleLinkViewModel` enrutan ahí |

Los tres respetan el flag `otpRequired` de la respuesta, así que si un host todavía no tiene
el backend nuevo el cliente sigue funcionando con el flujo viejo. Eso es lo que permite un
rollout escalonado entre los 5 hosts.

### 3.2 `bisaToken` fuera de la respuesta

- **Los 3 portales web:** campo eliminado del modelo y del almacenamiento. Se conserva el
  `removeItem` en el logout para que se purguen los valores que dejaron las sesiones
  anteriores al cambio.
- **Android e iOS:** se eliminó el `isAffiliate` derivado de `bisaToken != null` (era código
  muerto que habría pasado a ser permanentemente `false`). El campo queda **nullable** a
  propósito: los 5 hosts se despliegan por separado, así que un cliente puede ver el campo a
  mitad del rollout y debe tolerarlo.

### 3.3 Nuevos códigos de respuesta a manejar

| Código | Cuándo | Nota para el cliente |
|---|---|---|
| **403** | Póliza/documento/idPrint ajeno | Antes de este cambio el backend devolvía 200 con los datos |
| **429** | Rate limit excedido | Trae `Retry-After` en segundos |
| **503** | No se pudo verificar titularidad (BISA caído) | Es reintentable; **no** es un problema de permisos |
| **401** | Agente sin `bisaToken` (sesión expirada) | Debe re-loguear |

---

## 4. Qué falta confirmar antes de desplegar

### 4.1 🔴 Bloqueante — emails de afiliados en BISA

El código de verificación se envía al email del perfil BISA. Si no hay uno real (o es el
sintético `{documento}@affiliate.bisaseguros.com`), **el registro se bloquea** y se deriva al
call center.

Es la decisión correcta en seguridad —el sentido del código es que el atacante no pueda
recibirlo— pero significa que **un afiliado sin email en BISA no podrá auto-registrarse.**

**Acción:** medir cuántos asegurados están en esa situación. Si el número es material, decidir
entre: (a) carga previa de emails, (b) canal alternativo verificado —`UserProfile` tiene
`telefono`, se podría usar SMS—, o (c) un procedimiento de call center documentado.

Consulta orientativa sobre la BD de la app (los que ya tienen cuenta con email sintético son
un indicador del tamaño del problema):

```sql
SELECT COUNT(*) AS con_email_sintetico
FROM app_users
WHERE auth_provider = 'AFFILIATE'
  AND email LIKE '%@affiliate.bisaseguros.com';
```

El dato definitivo, sin embargo, está del lado de BISA: cuántos registros de party de Salud
Internacional tienen el campo `email` vacío.

### 4.2 🔴 Bloqueante — emails de usuarios de backoffice

Con MFA obligatorio, un usuario de backoffice sin email entregable **no puede iniciar
sesión**. Verificar antes de activar:

```sql
SELECT u.username, u.email
FROM app_users u
JOIN app_user_roles ur ON ur.user_id = u.id
JOIN roles r ON r.id = ur.role_id
WHERE r.name IN ('ROLE_ADMIN','ROLE_SINIESTROS','ROLE_RENOVACIONES',
                 'ROLE_REVISION_PAGOS','ROLE_CARTAS_GARANTIA','ROLE_CALL_CENTER')
  AND (u.email IS NULL OR u.email = '' OR u.email LIKE '%@affiliate.bisaseguros.com');
```

Y confirmar que el SMTP entrega efectivamente a esos dominios (probar con un envío real, no
solo revisar configuración).

### 4.3 🟠 Configurar antes de desplegar

```yaml
app:
  security:
    rate-limit:
      trusted-proxies: <IP de nginx>     # ver advertencia en VUL-009
    mfa:
      mandatory-roles: ""                # vacío = los 6 de backoffice
```

### 4.4 🟠 Los 5 hosts se verifican por separado

El informe insiste en esto y sigue vigente: comparten backend pero son endpoints de red
independientes, con DNS, TLS y política de WAF propias, y pueden tener despliegues o cachés
distintos. **Un fix verificado en `servcon` no implica `bisa-api`, `app`, `asesores` ni
`coi`.** La migración Liquibase 035 debe correr en cada base.

### 4.5 🟡 Latencia del detalle de póliza del agente

Es el flujo que más llamadas dispara. Medir que el caché de 3 minutos deja los tiempos donde
estaban (estimado: ~2 llamadas a BISA en lugar de 7 para una póliza con 5 asegurados).

---

## 5. Qué queda fuera de este alcance

### Hallazgos no abordados

Los 8 MEDIA y 8 BAJA/INFO restantes: VUL-007 (`IntermediaryController` sin rol explícito),
VUL-008 (Stored XSS en perfil), VUL-010 (política de contraseñas solo cliente), VUL-011 (GET
que envía correo), VUL-012/013 (validación de uploads), VUL-014 (rango de fechaEvento),
VUL-015 (errores verbosos), VUL-017/018 (CSP/HSTS, BREACH), VUL-019 (nombres duplicados),
VUL-020 (certificate pinning), VUL-021 (IP interna en historial), VUL-022 a VUL-029.

> **Actualización (agosto 2026).** La segunda ronda cerró VUL-015 y VUL-017/018, y la mitad de
> VUL-012/013 (los uploads de reembolsos). Ver sección 12 para el estado real de cada uno.

> **VUL-010 merece atención temprana:** el informe reporta que se acepta `qwertyui`. La
> validación de complejidad es solo del lado del cliente. El rate limiting de VUL-009 reduce
> la explotabilidad pero **no corrige la debilidad de fondo.**

### Requiere infraestructura, no código

- **Segmentación de portales por host.** Que `POST /api/v1/auth/agent/login` funcione desde
  cualquier dominio es correcto como observación, y no lo resuelve este cambio: requiere
  segmentación a nivel de gateway/WAF.
- **`bisa-api.xiobit.com` expuesto a Internet sin WAF** (host de staging, nginx 1.18.0
  desactualizado — VUL-029). Decidir si debe seguir siendo público es la mitigación más barata
  de varios hallazgos a la vez.

---

## 6. Cómo testear que los puntos quedaron resueltos

### 6.1 Preparación

Se necesita lo mismo que usó el consultor: **dos cuentas de afiliado reales** (A y B) con
pólizas distintas, una cuenta de agente y una de backoffice.

```bash
HOST=https://servcon.atlasglobalhealth.com     # repetir con los otros 4 hosts

# Token de la cuenta B (el "atacante")
TOKEN_B=$(curl -s -X POST "$HOST/api/v1/auth/affiliate/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"<CI_B>","password":"<pass_B>"}' | jq -r '.accessToken')

# Datos de la cuenta A (la "víctima"): sacar policyNo e idPrint logueando como A
POLICY_A=<policyNo de A>
IDPRINT_A=<idPrint de A>
```

Repetir todo contra los **5 hosts**: `servcon.atlasglobalhealth.com`,
`bisa-api.xiobit.com`, `app.atlasglobalhealth.com`, `asesores.atlasglobalhealth.com`,
`coi.atlasglobalhealth.com`.

---

### 6.2 VUL-001 — BOLA/IDOR en pólizas

**Esperado: `403` en los 7. Antes devolvían `200` con los datos de A.**

```bash
for p in \
  "policies/$POLICY_A/detail?deductibleStart=2026-01-01&deductibleEnd=2026-12-31" \
  "policies/deductibles/$POLICY_A" \
  "policies/coverages/$POLICY_A/1-1-1" \
  "policies/invoices/$POLICY_A" \
  "policies/print/$IDPRINT_A" \
  "policies/user/profile/<CI_A>" \
  "policies/data/<partyCode_A>" ; do
  echo -n "$p → "
  curl -s -o /dev/null -w "%{http_code}\n" "$HOST/api/v1/$p" -H "Authorization: Bearer $TOKEN_B"
done
```

**Verificaciones adicionales, todas necesarias:**

1. **Bidireccionalidad:** repetir con el token de A y los identificadores de B.
2. **La propia póliza sigue funcionando** (esto es lo que detecta un fix demasiado agresivo):
   con `TOKEN_B` y el `policyNo` de **B**, los 7 deben dar `200`.
3. **Desde las apps nativas**, reutilizando el `accessToken` real de sesión, que es como el
   consultor confirmó el hallazgo: iOS contra `servcon`, Android contra `bisa-api`.
4. **Mayúsculas/minúsculas:** el `policyNo` propio en minúsculas debe seguir dando `200` (la
   comparación es case-insensitive a propósito, igual que en el resto del sistema).

El cuerpo del 403 debe ser `{"message":"No tiene permisos para acceder a esta póliza."}`.

---

### 6.3 VUL-002 — BOLA/IDOR en pagos SWIFT

**Esperado: `403`.**

```bash
for p in \
  "policies/$POLICY_A/pagos-swift/latest" \
  "policies/$POLICY_A/pagos-swift/periodo-actual" \
  "policies/$POLICY_A/pagos-swift/cuotas" \
  "policies/$POLICY_A/pagos-swift" ; do
  echo -n "$p → "
  curl -s -o /dev/null -w "%{http_code}\n" "$HOST/api/v1/$p" -H "Authorization: Bearer $TOKEN_B"
done
```

Y la subida de comprobante a una póliza ajena (esperado `403`):

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  "$HOST/api/v1/policies/$POLICY_A/pagos-swift" \
  -H "Authorization: Bearer $TOKEN_B" -F "file=@comprobante.pdf"
```

**Regresión obligatoria:** subir un comprobante a la póliza **propia** debe seguir dando
`200`.

---

### 6.4 VUL-003 — Oráculo de identidad y account takeover

**a) El oráculo.** ⚠️ Usar un **CI que ya exista en la BD del host** —con un CI nuevo el
comportamiento vulnerable no se reproduce (ver sección 1).

```bash
# CI registrado + fecha INCORRECTA
curl -s -X POST "$HOST/api/v1/auth/affiliate/verify-identity" \
  -H 'Content-Type: application/json' \
  -d '{"documento":"<CI_registrado>","fechaNacimiento":"2001-05-05"}'

# CI inexistente
curl -s -X POST "$HOST/api/v1/auth/affiliate/verify-identity" \
  -H 'Content-Type: application/json' \
  -d '{"documento":"99999999","fechaNacimiento":"2001-05-05"}'
```

**Esperado en ambos: el mismo status y el mismo cuerpo,**
`{"message":"Datos de identidad inválidos"}`. **No debe aparecer `nombreCompleto`.**

Que las dos respuestas sean **idénticas** es el criterio: si difieren en algo —status, texto,
longitud— el oráculo sigue existiendo en forma más fina.

**b) Fecha correcta sí funciona:**

```bash
curl -s -X POST "$HOST/api/v1/auth/affiliate/verify-identity" \
  -H 'Content-Type: application/json' \
  -d '{"documento":"<CI>","fechaNacimiento":"<fecha_correcta>"}'
```

Esperado: `200` con `setupToken`, `otpRequired: true`, `maskedEmail`, `nombreCompleto`. **Debe
llegar un correo con el código de 6 dígitos.**

**c) El takeover.** Es la prueba que cierra el hallazgo. Con un ID Token de Google real de una
cuenta cualquiera y el CI + fecha **correctos** de un afiliado:

```bash
# 1) Sin el código confirmado → debe RECHAZAR
curl -s -X POST "$HOST/api/v1/auth/affiliate/google-register" \
  -H 'Content-Type: application/json' \
  -d '{"documento":"<CI>","fechaNacimiento":"<fecha_correcta>",
       "idToken":"<google_id_token>","setupToken":"<setupToken>"}'
# Esperado: "Debe verificar el código enviado a su correo registrado antes de continuar."

# 2) Sobre una cuenta que YA tiene contraseña → debe RECHAZAR incluso con el código
#    confirmado. Esperado: "Su cuenta ya está configurada."
```

**El criterio de éxito es que no se obtenga `accessToken` en ninguno de los dos casos.**

**d) El flujo legítimo completo:**

```bash
# verify-identity → llega el código al correo → confirmarlo
curl -s -X POST "$HOST/api/v1/auth/affiliate/verify-setup-otp" \
  -H 'Content-Type: application/json' \
  -d '{"setupToken":"<setupToken>","otpCode":"<codigo_del_correo>"}'
# → 200

# recién ahora set-password funciona
curl -s -X POST "$HOST/api/v1/auth/affiliate/set-password" \
  -H 'Content-Type: application/json' \
  -d '{"setupToken":"<setupToken>","newPassword":"...","confirmPassword":"..."}'
# → 200 con accessToken, y llega un correo de aviso de cambio de credenciales
```

**e) Intentos del código:** 3 códigos incorrectos deben quemar el token
("Demasiados intentos"), y a partir de ahí hay que reiniciar desde `verify-identity`.

---

### 6.5 VUL-004 — `bisaToken`

```bash
curl -s -X POST "$HOST/api/v1/auth/agent/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"<agente>","password":"<pass>"}' | jq
```

**Esperado:** **no** aparece `bisaToken` en la respuesta, y `accessToken` contiene un JWT
real (no el literal `"Bearer"` — ese era el bug adyacente). Decodificar el `accessToken` y
confirmar que su payload **no** contiene `192.168.230.5`, códigos internos de compañía ni
email corporativo.

Repetir con login de afiliado y de backoffice.

**Regresión obligatoria:** el portal de agentes debe seguir funcionando completo —cartera,
detalle de póliza, descarga de PDF—, porque el backend sigue usando el `bisaToken` desde la
BD.

---

### 6.6 VUL-005 — MFA obligatorio

```bash
curl -s -X POST "$HOST/api/v1/auth/backoffice/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<pass>"}' | jq
```

**Esperado:** `{"otpRequired":true,"otpToken":"...","maskedEmail":"..."}` y **ningún
`accessToken`**. Debe llegar el correo con el código.

Verificar los **6 roles** de backoffice, no solo `admin`, y con cuentas que tengan
`twoFactorEnabled = false` —el punto es justamente que ya no importa ese flag:

```sql
SELECT username, two_factor_enabled FROM app_users WHERE username = 'admin';
-- two_factor_enabled = 0 y aun así debe exigir OTP
```

Luego completar con `POST /api/v1/auth/backoffice/verify-otp` y confirmar que sí entrega el
`accessToken`.

**Contraste:** el login de **afiliado** con `twoFactorEnabled = false` debe seguir entregando
`accessToken` directo (no se agregó fricción a clientes finales).

---

### 6.7 VUL-006 — IDOR en siniestros

```bash
# id de un siniestro de la cuenta A, con token de B
for p in "siniestros/<id_A>/historial" "siniestros/<id_A>/documentos" ; do
  echo -n "$p → "
  curl -s -o /dev/null -w "%{http_code}\n" "$HOST/api/v1/$p" -H "Authorization: Bearer $TOKEN_B"
done
```

**Esperado: `403`** (antes el historial devolvía `200` con el historial completo, el nombre
del agente gestor y las observaciones).

Verificar también `GET /siniestros/{id_A}/documentos/{docId}` → `403`.

**Regresión obligatoria:** un agente consultando el historial de un siniestro **de su
cliente** debe seguir dando `200`. Nota: los siniestros creados por el afiliado desde la app
tienen `agente = null` y el agente recibirá `403` — eso es correcto y coincide con el
comportamiento que ya tenía `GET /siniestros/{id}`.

---

### 6.8 VUL-009 — Rate limiting

```bash
for i in $(seq 1 7); do
  echo -n "intento $i → "
  curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    "$HOST/api/v1/auth/affiliate/verify-identity" \
    -H 'Content-Type: application/json' \
    -d '{"documento":"1234567","fechaNacimiento":"2001-01-01"}'
done
```

**Esperado:** los primeros 5 responden normal, del 6.º en adelante **`429`** con cabecera
`Retry-After: 60`.

Repetir con `*/login` (límite 10/min) y `*/resend-otp` (3 cada 5 min).

**Verificación importante de que `trusted-proxies` está bien configurado:** repetir los 7
intentos rotando la cabecera `X-Forwarded-For` en cada uno. Si **igual llega el 429**, la
configuración es correcta. Si los 7 pasan, la app está confiando en una cabecera que el
cliente controla y el límite es evadible.

```bash
for i in $(seq 1 7); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST \
    "$HOST/api/v1/auth/affiliate/verify-identity" \
    -H "X-Forwarded-For: 10.0.0.$i" -H 'Content-Type: application/json' \
    -d '{"documento":"1234567","fechaNacimiento":"2001-01-01"}'
done
```

---

### 6.9 VUL-016 — Actuator

```bash
curl -s -o /dev/null -w "health: %{http_code}\n"      "$HOST/actuator/health"
curl -s -o /dev/null -w "env: %{http_code}\n"         "$HOST/actuator/env"
curl -s -o /dev/null -w "configprops: %{http_code}\n" "$HOST/actuator/configprops"
curl -s -o /dev/null -w "beans: %{http_code}\n"       "$HOST/actuator/beans"
```

**Esperado:** `health` → `200` (lo usa el healthcheck de Docker); el resto → `401`/`403`.

---

### 6.10 Regresión funcional obligatoria

Los 403 nuevos son el riesgo principal de este cambio. Hay que recorrer los flujos completos,
no solo los casos de ataque.

**clientes-web / Android / iOS (afiliado, sobre su propia póliza):**
- [ ] Home
- [ ] Detalle de póliza, deducibles, facturas
- [ ] Descarga del PDF de póliza
- [ ] Credencial / wallet (Apple y Google)
- [ ] Pagos SWIFT: periodo actual, cuotas, **subida de comprobante**
- [ ] Siniestros: listado, detalle, historial, documentos
- [ ] Cartas de garantía
- [ ] **Registro completo de un afiliado nuevo** (el flujo de 3 pasos)
- [ ] **Registro con Google** de un afiliado nuevo

**agentes-web — ⚠️ el flujo con más riesgo de regresión de todo el cambio:**
- [ ] Login (y login con 2FA activado, que estaba roto y ahora debería funcionar)
- [ ] Cartera → listado de clientes
- [ ] Detalle de póliza **de un cliente de su cartera**
- [ ] **Expandir cada asegurado** (dispara una llamada de coberturas por fila)
- [ ] Descargar el PDF de la póliza de un cliente
- [ ] Historial de un siniestro de su cliente
- [ ] Y el caso negativo: una póliza **fuera** de su cartera debe dar `403`

**backoffice-web:**
- [ ] Login **con el paso de OTP nuevo**, para los 6 roles
- [ ] Call center: búsqueda por documento y por póliza
- [ ] Detalle de póliza del call center
- [ ] Pagos SWIFT (backoffice)

---

## 7. Estado de la verificación automatizada

| Componente | Resultado |
|---|---|
| Backend | `./mvnw test` → **902 tests, 0 fallos** (87 nuevos) |
| clientes-web | typecheck limpio, 1/1 tests |
| agentes-web | typecheck limpio, 133/135 ⚠️ |
| backoffice-web | typecheck limpio, 79/79 |
| Android | `compileDevDebugKotlin` limpio |
| iOS | `BUILD SUCCEEDED`, 74 tests, 0 fallos ⚠️ |

Los 87 tests nuevos prueban específicamente cada hallazgo: acceso cruzado → 403, propia
póliza → 200, cartera del agente → 200, fuera de cartera → 403, BISA caído → 503, orden del
guard *antes* de la llamada a BISA, CI+fecha insuficientes para tomar una cuenta, y el filter
chain de seguridad construible.

⚠️ **Dos matices, dichos con precisión:**

- **agentes-web 133/135:** los 2 fallos son de `siniestro.model.spec.ts` y
  `renovacion.model.spec.ts` (clases CSS de badges). Verificado con git que esos archivos no
  se tocaron — son preexistentes.
- **iOS:** los 74 tests pasan y todas las suites reportan `passed`, pero `xcodebuild` imprime
  `** TEST FAILED **`. La causa es un fallo al escribir el bundle `.xcresult` en DerivedData
  (`mkstemp: No such file or directory`), no un test fallido.

### Deuda técnica encontrada de paso

**Los test source sets de Android e iOS ya estaban rotos antes de este trabajo** (verificado
con `git stash`). Ni `./gradlew test` ni la suite de iOS podían estar pasando en CI:

- Android: `NotificationRepositoryImplTest`, `ForgotPasswordViewModelTest`,
  `LoginViewModelTest` — **siguen rotos**, son ajenos a esta remediación.
- iOS: `MockAuthRepository` sin `deleteAccount()`, `LoginViewModelTests` con
  `navigateToRegister` desactualizado — **reparados**, para poder verificar este trabajo.
- Android `GoogleLinkViewModelTest` — reparado y reescrito para el flujo nuevo.

**Ningún test cargaba `SecurityConfig`.** Los ~30 tests de controller son slices
`@WebMvcTest` con `addFilters = false`, así que toda la configuración de seguridad estaba sin
probar y un error ahí solo aparecía al arrancar la aplicación. De hecho ocurrió durante este
trabajo: un filtro registrado contra una clase de referencia sin orden registrado en Spring
Security compilaba bien y reventaba en el boot. Se agregó
`SecurityConfigFilterChainTest`, que construye el filter chain real y verifica el orden
(throttling después de CORS, antes de la autenticación). Se validó reintroduciendo el bug a
propósito y confirmando que el test falla con el mismo mensaje.

---

## 8. Secuencia de despliegue sugerida

1. Resolver los bloqueantes 4.1 y 4.2 (emails).
2. Configurar `trusted-proxies` y, si aplica, `mandatory-roles` (4.3).
3. Desplegar en **un** host (sugerido: `bisa-api.xiobit.com`, staging) y correr la sección 6
   completa ahí.
4. Regresión funcional (6.10), empezando por **agentes-web**.
5. Desplegar en los 4 restantes, **verificando cada uno por separado** (4.4).
6. Solicitar el **re-test** al consultor y actualizar la Matriz de Trazabilidad.
7. Configurar las alertas SIEM que recomienda el informe: un JWT consultando un
   `policyNo`/`partyCode` que no coincide con su `sub` debería ser **cero** en operación
   normal. Vigilar también el volumen de 429 y de 503 del guard —un pico de 503 indica una
   caída de BISA, no un problema de permisos.

---

# Segunda ronda — módulo de Reembolsos (agosto 2026)

**Contexto.** Después de la remediación de julio se construyó el módulo de Reembolsos
(backend + los tres portales web). BISA solicitó una nueva ronda de Ethical Hacking sobre
backend y webapps, así que se hizo primero una auto-auditoría del módulo nuevo y una
verificación de que nada de la ronda anterior se hubiera roto.

**Criterio:** OWASP Top 10 2021 más los patrones ya remediados en el repositorio (BOLA/IDOR,
rate limiting, MFA, validación de subidas).

**Entregable al auditor en esta ronda:** `endpoints-eh-reembolsos.md` — los 52 endpoints del
módulo con su rol requerido. El alcance completo (modelo de autorización, integración con
Freshdesk, casos de prueba y reglas del engagement) está en
`alcance-eh-reembolsos-freshdesk.md`, todavía interno.

**Por qué este módulo importa más que los otros:** transporta **diagnóstico clínico** y
**datos bancarios** del asegurado. Es el conjunto de datos más sensible de la plataforma.

## 9. Verificación de no-regresión de la ronda anterior

Lo primero, porque es lo que la pregunta original quería saber: **la remediación de julio
está intacta.**

- `origin/ethical-hacking` está contenido en `reembolsos` en los cuatro repositorios
  (`git log --oneline origin/ethical-hacking ^reembolsos` está vacío en todos).
- `PolicyAccessGuard`, `PolicyOwnershipCache`, `RateLimitFilter` y `MfaPolicy` siguen en pie
  y con sus tests pasando.
- `ROLE_REEMBOLSOS` y `ROLE_REEMBOLSOS_OPS` ya estaban en `BackofficeRoles.ALL`, así que
  heredan MFA obligatorio sin trabajo extra.
- `ReembolsoPolicyValidator` delega en `PolicyAccessGuard.requirePolicy`, con lo que hereda
  la semántica 403/401/503 verbatim en lugar de reimplementar la búsqueda en cartera.
- `ReembolsoController` ya estaba inventariado en `PolicyEndpointGuardInventoryTest`.
- **Línea base antes de tocar nada: 1136 tests, 0 fallos.** El módulo se construyó siguiendo
  los patrones; lo que sigue son huecos concretos, no un rediseño.

## 10. Hallazgos de esta ronda

| ID | Sev | Hallazgo | Estado |
|---|---|---|---|
| R-01 | 🟠 ALTA | `Content-Type` del multipart persistido sin validar ⇒ XSS almacenado vía `blob:` | ✅ Corregido |
| R-02 | 🟠 ALTA | Avisos y push de un expediente dirigidos a una dirección elegida por el agente | ✅ Corregido |
| R-03 | 🟠 ALTA | Webhook de Freshdesk sin rate limit ni auditoría, con oráculo de longitud del secreto | ✅ Corregido |
| R-04 | 🟡 MEDIA | `ROLE_CALL_CENTER` podía descargar todo documento médico y comprobante de la cartera | ✅ Corregido |
| R-05 | 🟡 MEDIA | `ROLE_REEMBOLSOS_OPS` ausente en el `@PreAuthorize` de clase: pagaba a ciegas | ✅ Corregido |
| R-06 | 🟡 MEDIA | Subida de documentos sin control de estado ⇒ inyección de evidencia post-envío | ✅ Corregido |
| R-07 | 🟡 MEDIA | CORS comodín credenciado en `WebConfig`; la lista de producción era letra muerta | ✅ Corregido |
| R-08 | 🟡 MEDIA | `page`/`size` sin acotar: `size=100000` vuelca la tabla entera | ✅ Corregido |
| R-09 | 🟡 MEDIA | Oráculo de existencia: 400 para un id inexistente vs 403 para uno ajeno | ✅ Corregido |
| R-10 | 🔵 BAJA | `Content-Disposition` con el nombre original sin sanear | ✅ Corregido |
| R-11 | 🔵 BAJA | Comprobante de pago sin comprobación de magic bytes | ✅ Corregido |
| R-12 | 🔵 BAJA | `gestorAgenteId` permitía enumerar `app_users` y leer nombres ajenos | ✅ Corregido |
| R-13 | 🔵 BAJA | IP de la declaración jurada no consciente de proxy: sellaba la IP de nginx | ✅ Corregido |
| R-14 | 🔵 BAJA | PII en logs (correo completo, CI, monto) y `syncError` sin recortar | ✅ Corregido |
| R-15 | 🔵 BAJA | `GlobalExceptionHandler` hacía eco de `ex.getMessage()` en el 500 | ✅ Corregido |
| R-16 | 🔵 BAJA | `application-staging.yml` sin endurecer: heredaba trazas y `/v3/api-docs` público | ✅ Corregido |
| R-17 | 🔵 BAJA | Ninguna denegación de acceso dejaba fila de auditoría | ✅ Corregido |
| W-01 | 🟠 ALTA | (3 portales) `mimeType` del blob sin allowlist — la otra mitad de R-01 | ✅ Corregido |
| W-02 | 🟡 MEDIA | backoffice: `/reembolsos*` sin guard de rol (protección solo del menú) | ✅ Corregido |
| W-03 | 🟡 MEDIA | backoffice: controles de escritura renderizados para cualquier rol | ✅ Corregido |
| W-04 | 🔵 BAJA | agentes: borrador con PII del asegurado sobrevivía al logout | ✅ Corregido |
| W-05 | 🟡 MEDIA | (3 portales) sin CSP, y cabeceras perdidas en los `location` de assets | ✅ Corregido |
| W-06 | 🔵 BAJA | agentes/backoffice: el interceptor adjuntaba el Bearer a cualquier URL | ✅ Corregido |
| W-07 | 🔵 BAJA | backoffice: validación del comprobante solo por sufijo del nombre | ✅ Corregido |

**Lo que salió limpio, y conviene decirlo:** la trampa de autenticación documentada en
`CLAUDE.md` (pasar una URL privada a `<img src>` / `window.open`) **no aparece** — los seis
caminos de descarga usan correctamente blob vía `HttpClient`. Tampoco hay sinks de XSS en el
código de reembolsos (todo `{{ }}`), ni URLs cableadas, ni peticiones fuera del `HttpClient`,
ni concatenación de SQL/JPQL: la única consulta nativa recibe un prefijo construido en el
servidor. La validación de subidas de agentes-web es sólida (tabla por tipo, tope de 10,
duplicados, cruce MIME↔extensión declarada) y el backend ya tenía la única comprobación de
magic bytes de la plataforma.

### R-01 — El `Content-Type` del multipart llegaba a la cabecera de respuesta (ALTA)

`ReembolsoDocumentoService` guardaba `.mimeType(file.getContentType())`: el valor que declara
quien sube el archivo. La extensión y los magic bytes sí se comprobaban, el MIME no, y
`ReembolsoComprobanteService.archivo()` lo replicaba con `MediaType.parseMediaType(...)`.

Los tres portales reenvuelven esos bytes y los abren con `window.open` sobre una URL `blob:`.
**Un documento `blob:` hereda el origen de quien lo abre**, así que un agente que sube un
políglota (`%PDF-…<script>`, que pasa los magic bytes a propósito) declarando
`Content-Type: text/html` ejecuta script en el origen del **backoffice**, donde el JWT queda
en `localStorage` cuando el usuario marcó «recordarme». El `Content-Disposition: attachment`
del servidor no protege: el SPA descarta la cabecera al reconstruir el blob.

Corregido en las dos capas, a propósito:

- Backend: nuevo `service/ContenidoArchivo.java` deriva el tipo de la **extensión ya
  validada** (`mimeDeExtension`) y normaliza cualquier tipo ya almacenado antes de emitirlo
  (`mimeSeguro`, que cubre las filas escritas antes de esta corrección).
- Frontend: allowlist `application/pdf | image/jpeg | image/png` antes de `createObjectURL`
  en `agentes/shared/utils/blob-download.ts` y en los nuevos
  `{clientes,backoffice}/shared/utils/blob-viewer.ts`; cualquier otro valor se degrada a
  `application/octet-stream`.

La allowlist se queda en el cliente **aunque el servidor ya normalice**, porque es la capa
que decide qué se ejecuta y no debe depender de que el otro lado siga haciendo lo correcto.

### R-02 — Datos del expediente a una dirección que elige el agente (ALTA)

`correoContacto` y `cuentaTitularEmail` son texto libre del formulario (`@Email` y nada más)
y nadie los coteja contra el titular de la póliza. Consecuencias:

- `contextoBase` ponía **`diagnostico`** en el contexto de las tres plantillas de correo.
  Ninguna plantilla lo usaba todavía —el riesgo era latente, no activo—, pero el destinatario
  es esa dirección de texto libre: cualquier edición futura de la plantilla habría publicado
  el diagnóstico clínico de un tercero. **Quitado del contexto.**
- `DomainNotificationService.reembolsoRecipient`, sin `asegurado` vinculado, caía a
  `resolveByEmail(correoContacto, cuentaTitularEmail)`: el **push** —con nombre del asegurado,
  número de reembolso y monto pagado— se entregaba a cualquier usuario local cuyo correo
  coincidiera. Bastaba escribir el correo de otro usuario de la app. **Fallback eliminado:**
  sin asegurado vinculado no hay push, solo correo. Enrutar una notificación por un dato que
  elige el emisor no tiene arreglo parcial.
- El correo de contacto **sí** puede legítimamente ser el del agente, así que no se rechaza;
  lo que no podía pasar es que la elección no quedara registrada. Ahora el historial del envío
  anota la dirección (enmascarada) y si difiere de la cuenta del asegurado.

> **Decisión pendiente de BISA.** Los avisos siguen llevando nombre del asegurado, número de
> póliza e importe pagado a esa dirección, porque es el flujo de negocio: el agente gestiona
> en nombre del asegurado. Si se quiere cerrar del todo, la corrección es validar
> `correoContacto` contra el correo del titular en BISA y rechazar lo que no coincida — es un
> cambio funcional, no técnico, y necesita su aprobación.

### R-03 — El webhook de Freshdesk (ALTA)

El diseño de fondo es correcto y conviene decirlo: secreto compartido, cerrado por defecto
(sin `FRESHDESK_WEBHOOK_SECRET` no atiende a nadie), y `aplicarDesdeWebhook` **no confía en el
`status` del cuerpo** sino que re-consulta Freshdesk, así que lo peor que consigue quien tenga
el secreto es forzar una resincronización contra el dato real. Los huecos eran tres:

- `RateLimitFilter.bucketFor` solo cubría rutas de autenticación, así que la ruta quedaba con
  **adivinación ilimitada del secreto y sin bloqueo**. Nuevo bucket `WEBHOOK`
  (`app.security.rate-limit.webhook-*`, 30/min/IP por defecto) que cubre
  `/api/v1/public/webhooks/**` y `/api/v1/public/wallet/verify`.
- `constantTimeEquals` no cortocircuitaba por carácter pero **retornaba temprano si las
  longitudes diferían**, regalando la longitud del secreto. Reemplazado por
  `MessageDigest.isEqual` sobre bytes UTF-8. Mismo cambio en `WalletVerifyController`, que
  comparte el esquema.
- El 401 no escribía nada. Ahora hace `log.warn` y una fila de auditoría
  (`ACCESO_DENEGADO`, canal `INTEGRACION`, nuevo valor de `CanalOrigen`).

### R-04 y R-05 — El reparto de roles del backoffice (MEDIA)

Ningún handler de `BackofficeReembolsoController` hace comprobación por registro:
`esBackoffice = true` salta `requireAcceso` **a propósito**, porque operaciones ve toda la
cartera. Eso significa que el rol del `@PreAuthorize` es el único control de acceso, y ahí
había dos cosas mal:

- La anotación de clase concedía lectura a `ROLE_CALL_CENTER`, así que un rol de soporte de
  solo lectura podía recorrer `{id}` y bajarse el informe médico, la receta y la factura de
  **toda la cartera**. La descarga de adjuntos y de comprobantes pasa a exigir
  `ROLE_ADMIN | ROLE_REEMBOLSOS | ROLE_REEMBOLSOS_OPS`; el listado y el detalle siguen
  disponibles para call center, que es lo que necesita para atender una llamada.
- `ROLE_REEMBOLSOS_OPS` **faltaba en la anotación de clase** mientras las de método sí le
  daban reabrir, pagar y subir el comprobante. En Spring Security 6 la anotación de método
  **sustituye** a la de clase, no se suman, así que operaciones podía registrar un pago pero
  recibía 403 al abrir el detalle o al descargar el comprobante que acababa de subir.

### R-06 — Inyección de evidencia post-envío (MEDIA)

`uploadDocumentos` validaba cantidad, tipo, tamaño y magic bytes, pero **no el estado** —
mientras `deleteDocumento` sí exigía `BORRADOR` desde el principio. Se podían adjuntar
archivos a una solicitud enviada, aprobada o **pagada**; y como nada los empujaba al ticket,
el archivo quedaba en la base y en disco sin llegar nunca a Freshdesk: Operaciones resolvía
sobre un expediente distinto del que veía el agente.

Ahora solo se admite en `BORRADOR` y en `PENDIENTE_INFO` (que es exactamente el estado en que
Operaciones pidió información que falta), y en `PENDIENTE_INFO` se encola
`UPLOAD_ATTACHMENTS` en el outbox — un tipo de evento que ya tenía handler y que nadie
disparaba — para que el ticket no se desincronice.

### R-07 — Dos configuraciones de CORS compitiendo (MEDIA)

`config/WebConfig.java` registraba un mapeo MVC sobre `/**` con `allowedOriginPatterns("*")`,
`allowedHeaders("*")` y `allowCredentials(true)`: un comodín credenciado. En paralelo,
`SecurityConfig.corsConfigurationSource()` tenía la lista **escrita en el código**, con los
`localhost` de desarrollo y un `https://*.waveonix.com` que `setAllowedOrigins` nunca hace
coincidir (los comodines exigen `setAllowedOriginPatterns`), e **ignoraba** la propiedad
`app.cors.allowed-origins` que sí existe por perfil. Es decir: la configuración de producción
era letra muerta y el efectivo era el comodín.

La explotabilidad es limitada porque la autenticación es Bearer en cabecera y no cookie, pero
es un hallazgo que el auditor va a reportar. `WebConfig.addCorsMappings` se eliminó —
`SecurityConfig` es la única autoridad — y los orígenes salen de la propiedad, sin ningún
`localhost` en prod ni en staging.

### R-09 — El oráculo de 400 vs 403 (MEDIA)

`requireAcceso` lanzaba `RuntimeException("Reembolso no encontrado")` → **400** para un id
inexistente y `AccessDeniedException` → **403** para uno ajeno. Comparando códigos se podía
mapear qué ids existen sin tener acceso a ninguno. Para el agente y el asegurado los dos casos
son ahora el mismo 403; el backoffice, que ve toda la cartera por diseño, sigue distinguiendo.

### R-15 — Un matiz sobre VUL-015 que conviene precisar

El diagnóstico habitual es que `server.error.include-stacktrace: on-param` deja `?trace=true`
devolviendo la traza. En producción **eso ya estaba desactivado** (`application-prod.yml`
fija los tres en `never`). Lo que no está gobernado por `server.error.*` es
`GlobalExceptionHandler`, que construye sus propios cuerpos: el texto de una
`NullPointerException`, de un fallo de JDBC o de una respuesta de un tercero salía al cliente
**incluso con el perfil endurecido**. El handler de `Exception` pasa a devolver un mensaje
fijo y a registrar el detalle en el log. `application-staging.yml`, que no sobreescribía nada
y heredaba `on-param` más `/v3/api-docs/**` público, ahora replica el bloque de prod (R-16).

### W-05 — CSP: cierra VUL-017/018

Los tres `nginx.conf` traían `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`
(no-op en cualquier navegador actual) y `Referrer-Policy`, pero **ninguna CSP**. Además
`add_header` **no se hereda** en cuanto un `location` declara uno propio, así que los bloques
de caché de assets servían `.js`, `.css`, imágenes y fuentes sin ninguna cabecera de
seguridad. Es un fallo que solo se ve empíricamente con `curl -I` sobre un `.js`.

Se añadió CSP (`default-src 'self'; script-src 'self'; object-src 'none';
frame-ancestors 'self'`, más `style-src 'self' 'unsafe-inline'` porque Angular inyecta
estilos de componente en tiempo de ejecución), `Permissions-Policy` y
`Cross-Origin-Opener-Policy`, y el bloque se **repite a propósito** en cada `location` que
declara su propio `add_header`, con una nota explicando por qué no se debe «limpiar» la
duplicación. `script-src 'self'` es además la mitigación de defensa en profundidad de
R-01/W-01: un documento `blob:` hereda la CSP del origen que lo crea.

`Strict-Transport-Security` **no** va en el contenedor: la termina el nginx del host (ver
`nginx-ssl-setup-guide.md`) y emitirla dos veces es peor que emitirla una.

#### ⚠️ La CSP obligó a desactivar `inlineCritical` en los tres `angular.json`

Al desplegar, los tres portales salieron **sin estilos**. La causa: con la optimización activa
(configuraciones `production` y `docker`, no `development`), Angular inlinea el CSS crítico y
difiere la hoja completa así:

```html
<link rel="stylesheet" href="styles-XXXX.css" media="print" onload="this.media='all'">
```

Ese `onload` es un manejador **inline**, y `script-src 'self'` lo bloquea: nunca se ejecuta, el
`media` se queda en `print` y la hoja no se aplica jamás. Lo único que se ve es el CSS crítico
inlineado —fondo y tipografías— sobre HTML sin maquetar.

La corrección es `"optimization": { "styles": { "inlineCritical": false } }` en las
configuraciones `production` y `docker` de los tres `angular.json`, con lo que Angular emite un
`<link rel="stylesheet">` normal. Se paga un primer pintado marginalmente más tardío; la
alternativa era `script-src 'unsafe-inline'`, que es exactamente la directiva que sostiene la
mitigación de R-01.

> **Lección para el próximo cambio de CSP.** El fallo no aparece con
> `ng build --configuration development`, que desactiva la optimización entera. **Verificar
> siempre con `--configuration docker`**, que es lo que compila el `Dockerfile`, y revisar el
> `index.html` generado:
>
> ```bash
> ng build --configuration docker
> grep -o 'onload=[^>]*' dist/*/browser/index.html   # tiene que no devolver nada
> ```

#### Google Identity Services en clientes-web

La CSP de `clientes-web` incluye `https://accounts.google.com` en `script-src` y en `frame-src`
porque `GoogleAuthService` inyecta el script de GIS en tiempo de ejecución. Hoy no se carga
—`environment.googleClientId` está vacío en los tres entornos— pero sin esas dos entradas el
«Iniciar sesión con Google» rompería en silencio el día que se configure el client id.

## 11. Red de regresión añadida

El inventario de endpoints es la pieza más valiosa de la ronda anterior, y solo cubría tres
controladores. Ahora `PolicyEndpointGuardInventoryTest` cubre también `MisReembolsosController`,
`BackofficeReembolsoController`, `CatalogoReembolsoController` y
`BackofficeCatalogoReembolsoController`, con una categoría nueva y explícita:

> **`ROLE_GATED_BACKOFFICE_ENDPOINTS`** — endpoints donde el rol **es** la puerta, porque
> `esBackoffice = true` salta la comprobación por registro. Se listan aparte y no como
> «deliberadamente sin guard» porque son la clase de endpoint en la que aflojar un rol es
> exactamente lo mismo que quitar el control de acceso.

Tests nuevos, uno por hallazgo:

| Archivo | Fija |
|---|---|
| `service/ContenidoArchivoTest` | MIME derivado de la extensión; políglota `%PDF`+`text/html` servido como `octet-stream`; `Content-Disposition` con comillas y CRLF (R-01, R-10) |
| `service/ReembolsoDocumentoServiceTest` (ampliado) | Subida rechazada en `REGISTRADO`/`PAGADO`; en `PENDIENTE_INFO` encola el reenvío; el tipo guardado no es el que declara el cliente (R-01, R-06) |
| `service/ReembolsoComprobanteValidacionTest` | PNG renombrado a `.pdf` rechazado, también sin `Content-Type` (R-11) |
| `security/RateLimitFilterTest` (ampliado) | 429 en el webhook y en wallet; tope por IP; otros `/public/**` sin tope (R-03) |
| `controller/FreshdeskWebhookControllerTest` (ampliado) | Secreto de otra longitud indistinguible; cada 401 auditado; un aviso válido no genera ruido (R-03) |
| `controller/ReembolsoControllerAccessTest` (ampliado) | Call center 403 en descargas, `ROLE_REEMBOLSOS` 200, OPS puede leer lo que paga (R-04, R-05) |
| `config/CorsAndProfileHardeningTest` | Falla si reaparece un `CorsRegistry` MVC o si vuelven los orígenes al código; staging y prod endurecidos (R-07, R-16) |
| `service/ReembolsoServiceTest` (ampliado) | Id inexistente e id ajeno indistinguibles; `size` acotado a 100, `page` negativo sin reventar (R-08, R-09) |
| `service/ReembolsoGestorEnumeracionTest` | `gestorAgenteId` acotado a `ROLE_AGENT` activo, con mensaje idéntico exista o no (R-12) |
| `shared/utils/blob-download.spec.ts` (agentes), `blob-viewer.spec.ts` (clientes, backoffice) | Allowlist de MIME, incluido `text/html; charset=utf-8` (W-01) |
| `core/guards/auth.guard.spec.ts` (backoffice, ampliado) | `reembolsosGuard` deja pasar los tres roles del módulo y rechaza los de otros; `/reembolsos` como destino por defecto (W-02) |
| `core/interceptors/auth.interceptor.spec.ts` (agentes, backoffice, ampliado) | El Bearer no sale hacia un host de terceros (W-06) |
| `core/services/auth.service.spec.ts` (agentes, ampliado) | El borrador con PII del asegurado se olvida al cerrar sesión (W-04) |

### Resultado

| Componente | Antes | Después |
|---|---|---|
| Backend | 1136 tests, 0 fallos | **1180 tests, 0 fallos** (`BUILD SUCCESS`) |
| clientes-web | — | build limpio, 20/20 |
| agentes-web | — | build limpio, 201/203 ⚠️ |
| backoffice-web | — | build limpio, 125/125 |

⚠️ **agentes-web 201/203:** los 2 fallos son los mismos de la ronda anterior
(`siniestro.model.spec.ts` y `renovacion.model.spec.ts`, clases CSS de badges). Verificado con
`git status` que esos archivos no se tocaron: siguen siendo preexistentes.

## 12. Qué queda abierto

**De esta ronda, pendiente de decisión de negocio:**

- **R-02 (parcial).** Los avisos siguen llevando nombre del asegurado, póliza e importe al
  `correoContacto` que escribe el agente. Cerrarlo del todo exige validar esa dirección contra
  el correo del titular en BISA, que es un cambio funcional.
- **R-04 (verificar con BISA).** Se restringió la descarga de adjuntos fuera de
  `ROLE_CALL_CENTER` bajo el supuesto de que atender una llamada no necesita el archivo. Si
  call center sí lo necesita, revertir la anotación y cubrirlo con auditoría de descargas
  (la infraestructura para eso ya está: R-17 dejó `ACCESO_DENEGADO` emitiéndose, y falta
  añadir el evento de descarga).

**Sigue abierto de la ronda de julio, ajeno a Reembolsos:**

- Bloqueo de cuenta persistente. `CustomUserDetailsService` sigue con `accountLocked(false)`
  fijo; solo hay throttling por IP, y es estado por instancia, así que N réplicas multiplican
  el límite efectivo.
- VUL-010 (política de contraseñas solo en cliente), VUL-008 (Stored XSS en perfil),
  VUL-029 (segmentación de portales / WAF).
- Magic bytes en siniestros, cartas de garantía y comprobantes SWIFT: el helper
  `ContenidoArchivo` ya está listo para reutilizar, pero son módulos distintos. Esto es la
  otra mitad de VUL-012/013.
- `swift-payment-review.component.ts` es el único `bypassSecurityTrustResourceUrl` de los tres
  portales y comparte el patrón de `mimeType` sin validar de R-01. Conviene **probarlo junto a
  R-01** en el re-test, aunque corregirlo sea otro módulo.

## 13. Re-test de esta ronda

Mismo formato que la sección 6. `$TOKEN_AGENTE` es un agente con al menos un reembolso;
`$TOKEN_CC` un usuario con solo `ROLE_CALL_CENTER`; `$ID` un reembolso existente.

```bash
# R-01 — subir un políglota declarando text/html y comprobar cómo se sirve
printf '%%PDF-1.4\n<script>alert(1)</script>' > /tmp/poliglota.pdf
curl -s -X POST "$BASE/api/v1/reembolsos/$ID/documentos" \
  -H "Authorization: Bearer $TOKEN_AGENTE" \
  -F "files=@/tmp/poliglota.pdf;type=text/html" -F "tipos=FACTURA"
# Luego descargarlo y mirar la cabecera: debe decir application/pdf, nunca text/html
curl -sI "$BASE/api/v1/reembolsos/$ID/documentos/$DOC_ID" \
  -H "Authorization: Bearer $TOKEN_AGENTE" | grep -i content-type

# R-01/R-10 — nombre hostil en Content-Disposition
cp /tmp/poliglota.pdf '/tmp/x";attachment;filename="y.html.pdf'
# Esperado al descargar: filename entrecomillado sin romperse, más filename*=UTF-8''

# R-03 — el webhook ya tiene tope y audita el fallo
for i in $(seq 1 35); do
  curl -s -o /dev/null -w "%{http_code} " -X POST \
    "$BASE/api/v1/public/webhooks/freshdesk/tickets" \
    -H 'X-Freshdesk-Webhook-Secret: incorrecto' \
    -H 'Content-Type: application/json' -d '{"ticketId":1}'
done
# Esperado: 401 hasta agotar el bucket y luego 429 con Retry-After.
# Y en la auditoría del backoffice: filas ACCESO_DENEGADO, canal INTEGRACION.

# R-03 — la longitud del secreto ya no se filtra: ambos deben tardar y responder igual
curl -s -o /dev/null -w "%{http_code}\n" -X POST "$BASE/api/v1/public/webhooks/freshdesk/tickets" \
  -H 'X-Freshdesk-Webhook-Secret: x' -H 'Content-Type: application/json' -d '{"ticketId":1}'
curl -s -o /dev/null -w "%{http_code}\n" -X POST "$BASE/api/v1/public/webhooks/freshdesk/tickets" \
  -H "X-Freshdesk-Webhook-Secret: $(head -c 64 /dev/zero | tr '\0' 'x')" \
  -H 'Content-Type: application/json' -d '{"ticketId":1}'

# R-04 — call center no baja la evidencia clínica
curl -s -o /dev/null -w "%{http_code}\n" \
  "$BASE/api/v1/backoffice/reembolsos/$ID/documentos/$DOC_ID/archivo" \
  -H "Authorization: Bearer $TOKEN_CC"                                    # → 403
curl -s -o /dev/null -w "%{http_code}\n" \
  "$BASE/api/v1/backoffice/reembolsos/$ID/comprobante" \
  -H "Authorization: Bearer $TOKEN_CC"                                    # → 403
curl -s -o /dev/null -w "%{http_code}\n" \
  "$BASE/api/v1/backoffice/reembolsos/$ID" -H "Authorization: Bearer $TOKEN_CC"  # → 200

# R-05 — operaciones puede leer el expediente que paga
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/api/v1/backoffice/reembolsos/$ID" \
  -H "Authorization: Bearer $TOKEN_OPS"                                   # → 200

# R-06 — no se adjunta a un expediente ya enviado
curl -s -X POST "$BASE/api/v1/reembolsos/$ID_ENVIADO/documentos" \
  -H "Authorization: Bearer $TOKEN_AGENTE" \
  -F "files=@/tmp/poliglota.pdf" -F "tipos=FACTURA"
# Esperado: 400 "no admite nuevos documentos en estado ..."

# R-07 — CORS ya no refleja un origen arbitrario
curl -sI -X OPTIONS "$BASE/api/v1/reembolsos" \
  -H 'Origin: https://atacante.example' \
  -H 'Access-Control-Request-Method: GET' | grep -i access-control-allow-origin
# Esperado: sin cabecera (o el origen configurado), nunca https://atacante.example

# R-08 — la paginación está acotada
curl -s "$BASE/api/v1/backoffice/reembolsos?size=100000" \
  -H "Authorization: Bearer $TOKEN_ADMIN" | head -c 300     # size efectivo 100
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/api/v1/backoffice/reembolsos?page=-1" \
  -H "Authorization: Bearer $TOKEN_ADMIN"                   # → 200, no 500

# R-09 — un id inexistente y uno ajeno son indistinguibles
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/api/v1/reembolsos/99999999" \
  -H "Authorization: Bearer $TOKEN_AGENTE"                  # → 403
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/api/v1/reembolsos/$ID_DE_OTRO_AGENTE" \
  -H "Authorization: Bearer $TOKEN_AGENTE"                  # → 403, mismo cuerpo

# R-16 — staging ya no filtra trazas ni publica el inventario
curl -s "$BASE_STAGING/api/v1/reembolsos/abc?trace=true" \
  -H "Authorization: Bearer $TOKEN_AGENTE" | grep -ci "at com.bisaseguros"   # → 0
curl -s -o /dev/null -w "%{http_code}\n" "$BASE_STAGING/v3/api-docs"         # → 401/403/404

# W-05 — las cabeceras llegan también en un asset con caché
curl -sI "$PORTAL/index.html" | grep -i "content-security-policy"
curl -sI "$PORTAL/$(curl -s $PORTAL/index.html | grep -oE 'main-[a-f0-9]+\.js' | head -1)" \
  | grep -i "content-security-policy"
# Las dos tienen que devolver la CSP. Si la segunda no la trae, se rompió la
# repetición del bloque en el location de assets.
```

### Regresión funcional obligatoria

Recorrido completo por la máquina de estados, que es donde se cruzan R-04, R-05 y R-06:

1. **agentes-web:** crear borrador → subir documentos → validar póliza → enviar.
2. **backoffice-web** como `ROLE_REEMBOLSOS`: revisar, aprobar/rechazar documentos, cambiar
   estado, pasar a `PENDIENTE_INFO`.
3. **agentes-web:** subir el documento que falta en `PENDIENTE_INFO` y confirmar que **llega
   al ticket de Freshdesk** (es el camino nuevo de R-06).
4. **backoffice-web** como `ROLE_REEMBOLSOS_OPS`: registrar el pago, subir el comprobante y
   **volver a abrir el detalle** (es lo que R-05 desbloqueó).
5. **backoffice-web** como `ROLE_CALL_CENTER`: confirmar que ve el listado y el detalle, y que
   los botones de descarga de adjuntos ya no se le ofrecen.
6. **clientes-web:** ver detalle, historial, y descargar documento y comprobante.
7. Confirmar que el webhook sigue aplicando el cambio de estado (**202**).

> **Qué no tocar durante el EH.** El webhook de Freshdesk contra el tenant real y el envío
> efectivo de tickets: contaminan la cola de BISA. Hay marcado de ticket de prueba (ver
> `ReembolsoTicketMarcadoPruebaTest`); usarlo, o apuntar `FRESHDESK_ENABLED=false`, que deja
> el endpoint devolviendo 503 sin tocar el tercero.
