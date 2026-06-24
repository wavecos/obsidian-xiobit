# Google Wallet — Estado actual y pendientes

> Última actualización: 2026-06-16
> Alcance: integración "Agregar credencial a Google Wallet" para afiliados de BISA Salud Internacional.

---

## 1. Resumen ejecutivo

La integración técnica con Google Wallet está **implementada extremo a extremo en backend y Android**, incluyendo ahora el **QR firmado + endpoint scanner**, el **callback de resultado en Android** y los **tests del backend**. Lo único que falta para que funcione en producción es **provisionamiento operativo en Google Pay & Wallet Console** y **configuración de variables de entorno** (guía paso a paso: `bisa-seg-salud-intl-tech-docs/google-wallet-setup.md`). Pendiente de roadmap: **paridad en iOS y Web** y **observabilidad**.

| Capa | Estado |
| --- | --- |
| Backend API (generación de JWT firmado, endpoint REST) | ✅ Implementado |
| Android (Pay SDK + UI Credenciales + savePassesJwt) | ✅ Implementado |
| Configuración runtime (variables de entorno) | ⚠️ Pendiente — `application.yml` listo, faltan valores reales |
| Provisión en Google Pay & Wallet Console | ❌ Pendiente — ver `tech-docs/google-wallet-setup.md` |
| QR firmado (JWT corto) + endpoint scanner `/wallet/verify` | ✅ Implementado |
| Manejo del resultado de `savePassesJwt` en Android | ✅ Implementado — bus + callback en MainActivity |
| iOS — Apple Wallet (`.pkpass`) | ✅ Backend + iOS listos end-to-end; falta provisión Apple (cert) + env (ver `apple-wallet-setup.md`) |
| Web (clientes-web) — "Enviar pase a móvil" | ❌ No iniciado |
| Tests automatizados (backend) | ✅ Implementado — QR token, Google pass, ambos controllers |

---

## 2. Lo implementado

### 2.1 Backend API (`bisa-seg-salud-intl-backend-api`)

**Archivos:**

- `src/main/java/.../config/GoogleWalletProperties.java`
  Properties externalizadas bajo `app.integration.google-wallet`: `issuerId`, `classId`, `serviceAccountPath`, `origin`.

- `src/main/java/.../service/GoogleWalletPassService.java`
  Construye y firma el JWT que el cliente Android pasa a `PayClient.savePassesJwt(...)`.
  - Carga la service account desde el JSON path al startup (`@PostConstruct`).
  - Firma RS256 con la clave privada de la service account (`google-auth-library-oauth2-http`).
  - JWT envelope: `iss=service account email`, `aud=google`, `typ=savetowallet`, `origins=[origin configurado]`, `payload.genericObjects=[objeto del pase]`.
  - GenericObject incluye: `cardTitle` ("BISA Salud Internacional"), `subheader` ("AFILIADO"), `header` (nombre del dependiente), `hexBackgroundColor=#003DA5`, logo, barcode QR, y `textModulesData`: documento, plan, número de póliza, vigencia desde/hasta, cobertura, moneda, relación.
  - Devuelve `WalletPassResponse { jwt, saveUrl }`.

- `src/main/java/.../controller/WalletPassController.java`
  `GET /api/v1/wallet/passes/{policyNumber}/{dependentCode}` autenticado por JWT de cliente.
  - Resuelve el usuario desde el `SecurityContext`.
  - Obtiene la póliza desde `BisaAffiliatePolicyService` (servicio externo de BISA Corporativo).
  - Valida que la póliza pertenece al usuario y que el dependiente existe.
  - Llama a `GoogleWalletPassService.generatePass(...)`.
  - Errores: 401 (sin usuario), 403 (sin partyCode), 404 (póliza/dependiente no encontrados).

- `src/main/java/.../model/dto/WalletPassResponse.java`
  DTO con `jwt` y `saveUrl` (`https://pay.google.com/gp/v/save/<jwt>`).

- `pom.xml`
  Dependencias `io.jsonwebtoken:jjwt-api/impl/jackson` (firma) y `com.google.auth:google-auth-library-oauth2-http` (carga de credenciales).

- `src/main/resources/application.yml`
  ```yaml
  app:
    integration:
      google-wallet:
        issuer-id: ${GOOGLE_WALLET_ISSUER_ID:}
        class-id: ${GOOGLE_WALLET_CLASS_ID:}
        service-account-path: ${GOOGLE_WALLET_SA_PATH:}
        origin: ${GOOGLE_WALLET_ORIGIN:https://salud.bisaseguros.com}
  ```
  Las cuatro variables están vacías por defecto y el servicio loguea un warning si no están configuradas.

### 2.2 Android (`bisa-seg-salud-intl-android`)

**Capa data:**

- `data/remote/api/WalletApiService.kt` — Retrofit interface, `GET api/v1/wallet/passes/{policyNumber}/{dependentCode}`.
- `data/remote/dto/WalletDtos.kt` — `WalletPassResponseDto(jwt, saveUrl)` con Moshi.
- `data/remote/source/WalletRemoteDataSource.kt` — Manejo de errores por status code (401/404/500) con mensajes traducidos al español.
- `data/repository/WalletRepositoryImpl.kt` — Mapea `WalletPassResponseDto -> String` (sólo el JWT).

**Capa domain:**

- `domain/repository/WalletRepository.kt`
- `domain/usecase/GetWalletPassJwtUseCase.kt` — Ejecuta en `IoDispatcher`.

**Capa presentation (pantalla Credenciales):**

- `presentation/credencial/CredencialContract.kt` — `CredencialUiState(walletAvailable, loadingDependentCode, ...)`, `CredencialUiEvent.CheckWalletAvailability/AddToWalletClicked`, `CredencialUiEffect.LaunchWalletSave/ShowError`.
- `presentation/credencial/CredencialViewModel.kt`
  - `checkAvailability(activity)`: usa `Pay.getClient(activity).getPayApiAvailabilityStatus(PayClient.RequestType.SAVE_PASSES)` y actualiza `walletAvailable`.
  - `requestPassJwt(...)`: invoca el usecase y emite `LaunchWalletSave(jwt)`.
- `presentation/credencial/CredencialScreen.kt`
  - `LaunchedEffect(activity) { viewModel.onEvent(CheckWalletAvailability, activity) }` al entrar.
  - Recolecta `uiEffect` y llama `Pay.getClient(activity).savePassesJwt(jwt, activity, ADD_TO_GOOGLE_WALLET_REQUEST_CODE)`.
  - Botón "Agregar a Google Wallet" se renderiza por dependiente sólo si `walletAvailable == true` y hay `dependentCode`.

**Dependencias:** `com.google.android.gms:play-services-pay` en `gradle/libs.versions.toml`.

### 2.3 Web clientes / iOS

No tienen implementación. El ícono `lucideWallet` en `policy-detail.component.ts` es decorativo.

---

## 3. Pendientes para llegar a producción

### 3.1 Provisionamiento en Google Pay & Wallet Console *(operativo, no código)*

Antes de configurar el backend, el equipo operativo de BISA debe:

1. **Crear/obtener cuenta Issuer** en https://pay.google.com/business/console.
   - Anotar el `issuerId` (formato numérico de ~16 dígitos).
2. **Solicitar acceso al Wallet API** (puede tardar 1-3 días hábiles).
3. **Crear un GenericClass** (la plantilla del pase). Definir:
   - `id`: convención `<issuerId>.bisa-salud-credencial-v1`.
   - `classTemplateInfo` (encabezado, lista de campos, layout) si se quiere ir más allá del default.
   - `securityAnimation` (opcional, sello de seguridad).
   - `multipleDevicesAndHoldersAllowedStatus: ONE_USER_ALL_DEVICES`.
   - Una vez creada, anotar el `classId` completo (`<issuerId>.<classSuffix>`).
4. **Crear Service Account** en Google Cloud, agregarle el rol **Wallet Object Issuer** desde la consola de Google Pay & Wallet.
5. **Descargar el JSON** de la service account. Tratarlo como secreto (no commitear).
6. **Whitelist del `origin`** en la configuración del Issuer: `https://salud.bisaseguros.com` (y dev: `http://localhost:8080` mientras se prueba).

> Referencia: https://developers.google.com/wallet/generic/web

### 3.2 Configuración runtime del backend

Una vez completado 3.1, definir en el host de producción (y staging):

```bash
GOOGLE_WALLET_ISSUER_ID=<numérico>
GOOGLE_WALLET_CLASS_ID=<issuerId>.bisa-salud-credencial-v1
GOOGLE_WALLET_SA_PATH=/etc/bisa-secrets/google-wallet-sa.json
GOOGLE_WALLET_ORIGIN=https://salud.bisaseguros.com
```

- Subir el JSON al servidor con permisos `0600` y owner del proceso de la app.
- Para `docker compose`, montar el archivo como secret/volumen y exportar las variables en `.env`.
- Reiniciar el backend. Verificar log de arranque: `"Loaded Google Wallet service account: <email>"`.

**Validación rápida:**
```bash
curl -H "Authorization: Bearer <token-cliente>" \
  https://salud.bisaseguros.com/api/v1/wallet/passes/<policyNo>/<depCode>
# Debe devolver { "jwt": "...", "saveUrl": "https://pay.google.com/gp/v/save/..." }
```

### 3.3 Endurecer el QR del pase (seguridad) — ✅ IMPLEMENTADO

**Implementado:** el barcode (Google y Apple) ahora contiene un JWT corto firmado
HS256 generado por `WalletQrTokenService.issue(policyNo, depCode)` (claims `pol`,
`dep`, `iat`, `exp` ~30 días). El endpoint scanner
`POST /api/v1/public/wallet/verify` (controller `WalletVerifyController`, protegido
por header `X-Wallet-Api-Key`) valida firma + expiración y devuelve
`{ valid, policyNo, dependentCode, expiresAt }`. Un código falso o vencido es
rechazado. Config: `app.integration.wallet-qr.*` (`WALLET_QR_SECRET`,
`WALLET_PARTNER_API_KEY`).

> Nota de diseño: el QR lleva sólo `pol`/`dep` (no PII) — la clínica contrasta
> contra el nombre/plan impreso en la cara del pase. Enriquecer la respuesta con
> datos vivos del afiliado requeriría un lookup BISA por número de póliza (hoy sólo
> existe por `partyCode`); queda como mejora futura.

**Propuesta original (referencia histórica):**

1. **Endpoint scanner** para clínicas/farmacias: `POST /api/v1/wallet/passes/verify` (autenticado por API key del partner o JWT de personal médico autorizado).
   - Body: `{ "code": "<contenido del QR>" }`.
   - Respuesta: datos del titular válidos para validación (nombre, póliza, vigencia, cobertura) + flag `valid: true|false`.
2. **Cambiar el contenido del QR** a un JWT corto firmado por el backend:
   - Claims: `pol` (policyNo), `dep` (dependentCode), `iat`, `exp` (~30 días) — refrescable.
   - Firma con la misma clave del JWT de Wallet o una clave dedicada.
   - Al escanear, el partner llama al endpoint scanner que valida la firma + expiración + estado del afiliado.
3. **Rotación del pase:** cuando expire el JWT del QR (o cambien datos del afiliado), el cliente Android debe **actualizar el objeto** vía Wallet REST API (`PATCH /walletobjects/v1/genericObject/{id}`) o regenerar el pase. Esto requiere agregar lógica de actualización en `GoogleWalletPassService` (hoy sólo crea, no actualiza).

### 3.4 Resultado de `savePassesJwt` en Android — ✅ IMPLEMENTADO

`MainActivity.onActivityResult` captura `ADD_TO_GOOGLE_WALLET_REQUEST_CODE`
(constante compartida en `WalletConstants.kt`) y publica el resultado en
`WalletSaveResultBus` (`@Singleton`). `CredencialViewModel` colecta el bus y emite
`CredencialUiEffect.ShowMessage("Credencial agregada a Google Wallet.")` en éxito;
`CredencialScreen` lo muestra como Toast. Cancelación = silencioso.

> `savePassesJwt` no expone un `IntentSender`, por eso se mantiene la ruta
> `onActivityResult` (anotada `@Deprecated`) en vez de `StartIntentSenderForResult`.

### 3.5 Auditoría y observabilidad

Hoy no hay registro persistente de cuántos usuarios agregaron el pase ni de su ciclo de vida.

Sugerencias:
1. **Log en `WalletPassController`** ya existe (línea 72-73). Mejorarlo con structured logging: usuario, póliza, dependiente, timestamp.
2. **Tabla `wallet_pass_event`** opcional:
   - Campos: `id`, `user_id`, `policy_no`, `dependent_code`, `event` (`GENERATED`, `SAVED`, `REMOVED`), `created_at`.
   - Liquibase changelog nuevo.
   - El evento `GENERATED` se inserta al llamar al endpoint REST; los eventos `SAVED`/`REMOVED` requieren configurar **Wallet Web Push Notifications** (un endpoint público que Google llama cuando cambia el estado del objeto).
3. **Webhook receiver:** `POST /api/v1/public/wallet/notifications` que reciba los pings de Google y registre los eventos en la tabla. Requiere validar la firma del payload contra la clave pública de Google.
4. **Métricas en Actuator:** `wallet.pass.generated`, `wallet.pass.saved` (contadores).

### 3.6 iOS — Apple Wallet (`.pkpass`)

**Estado:** ✅ **implementado extremo a extremo** (backend + iOS). Falta únicamente
**provisión en Apple** (Pass Type ID + certificado) y configurar variables de entorno.
Guía paso a paso: `bisa-seg-salud-intl-tech-docs/apple-wallet-setup.md`.

**Ya implementado:**

Backend (`ApplePassKitService`, `de.brendamour:jpasskit:0.5.0`):
- Genera y firma el `.pkpass` (ZIP con `pass.json` + assets + manifest + signature PKCS#7).
- Endpoint `GET /api/v1/wallet/passes/{policyNumber}/{dependentCode}/pkpass` →
  `Content-Type: application/vnd.apple.pkpass`, con validación usuario/póliza/dependiente.
  Devuelve **503** mientras el certificado no esté provisionado.
- QR = el mismo JWT firmado que Google (`WalletQrTokenService`).
- Assets de marca (`icon.png`/`logo.png` + @2x/@3x) empaquetados en
  `src/main/resources/apple-wallet/`.
- Tests: `ApplePassKitServiceTest`.

iOS (`AddToWalletButton` + `GetWalletPassUseCase` → `WalletRepositoryImpl` →
`WalletRemoteDataSource`):
- Botón "Añadir a Apple Wallet" en Credenciales (visible solo si `PKAddPassesViewController.canAddPasses()`).
- Descarga el `.pkpass` con la sesión Alamofire autenticada (`APIClient.download`).
- Lo presenta con `PKAddPassesViewController(pass: PKPass(data:))`.
- **No** requiere el entitlement `pass-type-identifiers` para el flujo de añadir.

**Pendiente operativo:** provisión Apple (cuenta personal para dev, migrar a org BISA
para prod) + variables de entorno + montar el `.p12`/WWDR en `docker/secrets/`.

**Para mantener paridad con Google Wallet:**
- Mismo conjunto de campos visibles (documento, plan, vigencia, cobertura, etc.).
- Mismo formato de QR (cuando se implemente 3.3, debe servir igual para ambos).
- Mismo `webServiceURL` apuntando a `/api/v1/wallet/apple/passes/{id}` para que Apple Wallet pueda actualizar el pase cuando cambien datos.

### 3.7 Web clientes — flujo "Enviar al móvil"

En desktop el `saveUrl` de Google Wallet redirige correctamente pero requiere que el usuario tenga el navegador con su cuenta Google y un Android cerca. Mejor UX:

Opción A — botón "Enviar pase a mi móvil por email":
- Endpoint backend `POST /api/v1/wallet/passes/{...}/send-email` que envía un correo al afiliado con el `saveUrl` del JWT.
- En el correo: instrucciones y dos botones (Google Wallet + Apple Wallet cuando exista).

Opción B — mostrar QR del `saveUrl` en pantalla:
- El usuario escanea con su móvil → abre Wallet → pase guardado.
- No requiere backend adicional; sólo añadir un componente QR (`angularx-qrcode` o similar) en `policy-detail.component.ts`.

Decidir con producto. Recomendación: **opción B** primero (sin trabajo de backend), opción A si se priorizan los usuarios que abren la web desde el móvil mismo.

### 3.8 Tests — ✅ Backend implementado

- `WalletQrTokenServiceTest`: round-trip issue→verify, rechazo de token expirado y de firma inválida, error cuando no hay secret.
- `GoogleWalletPassServiceTest`: estructura del JWT (`iss/aud/typ/origins/payload`), error sin SA, error sin issuer/class, sanitización del `objectId`, barcode = token del `WalletQrTokenService`.
- `@WebMvcTest(WalletPassController.class)`: 200, 401 (usuario no encontrado), 403 (sin partyCode), 404 (póliza/dependiente no coincide).
- `@WebMvcTest(WalletVerifyController.class)`: 200 válido, 200 inválido (expirado), 400 (token malformado), 401 (api key ausente/incorrecta), 503 (sin partner key).

> **Nota:** se agregó `@ExceptionHandler(ResponseStatusException.class)` en
> `GlobalExceptionHandler` — antes `ResponseStatusException` caía en el handler de
> `RuntimeException` y devolvía 400, anulando los 401/403/404 de los controllers.

**Pendiente:** Android instrumentation test del flow Credencial → Wallet (mock del PayClient).

### 3.9 Documentación operativa

Crear bajo `bisa-seg-salud-intl-tech-docs/`:

- `google-wallet-setup.md` — Pasos completos para registrar Issuer, crear class, service account, configurar variables.
- `google-wallet-rotation.md` — Cómo rotar la service account JSON sin downtime.
- `google-wallet-class-versioning.md` — Política para subir versiones del template (`v2`, `v3`) sin invalidar pases ya guardados.

---

## 4. Archivos críticos (referencia rápida)

| Archivo | Rol |
| --- | --- |
| `bisa-seg-salud-intl-backend-api/.../service/GoogleWalletPassService.java` | Firma del JWT. **TODO de seguridad del QR en líneas 107-114.** |
| `bisa-seg-salud-intl-backend-api/.../controller/WalletPassController.java` | Endpoint REST. |
| `bisa-seg-salud-intl-backend-api/.../config/GoogleWalletProperties.java` | Properties. |
| `bisa-seg-salud-intl-backend-api/src/main/resources/application.yml` (líneas 118-122) | Variables de entorno. |
| `bisa-seg-salud-intl-android/.../presentation/credencial/CredencialViewModel.kt` | Lógica del flow Wallet en Android. |
| `bisa-seg-salud-intl-android/.../presentation/credencial/CredencialScreen.kt` | `savePassesJwt(...)` (línea 88). |

---

## 5. Orden sugerido de trabajo

1. **Inmediato (operativo, sin código)** — Provisión en Google Pay & Wallet Console (3.1). Bloquea todo lo demás en ambiente real.
2. **Inmediato (código mínimo)** — Capturar resultado de `savePassesJwt` en Android (3.4) — quick win UX.
3. **Antes de release público** — Endurecer el QR + endpoint scanner (3.3). Sin esto, el pase no es seguro para validación en clínicas.
4. **Post-release iterativo** — Auditoría/observabilidad (3.5), tests (3.8), documentación operativa (3.9).
5. **Roadmap medio plazo** — iOS Apple Wallet (3.6) cuando la app iOS llegue a la pantalla de Credenciales.
6. **Roadmap largo plazo** — Flow web (3.7), si producto lo prioriza.
