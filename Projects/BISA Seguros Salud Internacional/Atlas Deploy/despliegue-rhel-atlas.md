# Guía de despliegue — Plataforma Atlas en servidor RHEL 9 / 10 (ambiente BISA)

> Guía paso a paso para dejar operativa **toda la parte de servidor** de la plataforma Atlas
> (Global Health) partiendo de un **servidor Red Hat Enterprise Linux recién instalado (solo SO)**.
> Válida para **RHEL 9 y RHEL 10**; las diferencias de RHEL 10 (repo de Docker y persistencia de
> iptables) están marcadas en los §2 y §3. Pensada para ejecutarse **presencialmente en BISA**.
>
> **Alcance:** Backend API (Spring Boot / Java 21) + las 3 SPA Angular (clientes, agentes, backoffice)
> en Docker Compose sobre el servidor de aplicaciones. El reverse proxy + TLS de borde corre en el
> Nginx que BISA ya tiene (host separado, §9), no lo instalamos nosotros.
>
> **Escenario acordado con BISA:**
> - BISA entrega el **servidor RHEL 10** (esta guía cubre también RHEL 9) con red/firewall y **un
>   datasource** apuntando a la base de datos
>   (SQL Server que **BISA administra** — nosotros NO instalamos el motor).
> - Runtime de contenedores: **Docker CE + `docker compose`**.
> - **Reverse proxy + TLS: se reutiliza el Nginx que BISA ya tiene, en un host separado** (proxy/balanceador
>   delante). No instalamos un Nginx de borde en el servidor de apps. TLS con **CA corporativa de BISA**.
> - El código se **clona desde GitHub en sitio**.
>
> Requerimientos completos: `Requerimientos_Tecnicos_Infraestructura_Despliegue.xlsx`.
> Documentos base (Ubuntu, aquí traducidos a RHEL): `Guia_Deployment_Docker.md`,
> `DEPLOYMENT_VPS.md`, `nginx-ssl-setup-guide.md`, `gmail-smtp-configuration.md`,
> `apple-wallet-setup.md`, `google-wallet-setup.md`.

---

## Arquitectura del despliegue

```
                Internet / red corporativa BISA
                              │  (443/80)
                 ┌────────────▼─────────────┐
                 │   Nginx de BISA           │  ← host SEPARADO; termina TLS
                 │   (proxy / balanceador)   │     (cert CA corporativa BISA)
                 └──┬────┬────┬────┬─────────┘
      api.<dom> ────┘    │    │    └──────── backoffice.<dom>
                         │    └───────────── agentes.<dom>
                         └────────────────── clientes.<dom>
                         │  proxy_pass a la IP INTERNA del servidor de apps
                         │  (8080 / 4201 / 4202 / 4203)  — red interna, no Internet
  ┌──────────────────────▼──────────────────────────────────────┐
  │  SERVIDOR DE APPS (RHEL 9/10) · Docker (red bisa_network)     │
  │                                                              │
  │  <ip-interna>:8080  backend-api  (Spring Boot / Java 21)      │
  │  <ip-interna>:4201  agentes-web  (Angular + Nginx interno)    │
  │  <ip-interna>:4202  backoffice-web                            │
  │  <ip-interna>:4203  clientes-web                              │
  │        │  cada web-container proxea /api/ → backend-api       │
  └────────┼─────────────────────────────────────────────────────┘
           │  JDBC 1433 (encrypt=true)
    ┌──────▼───────────────┐
    │  SQL Server de BISA   │  ← datasource provisto por BISA (externo)
    └───────────────────────┘
```

> **Dos capas de Nginx — no confundir:**
> 1. **Nginx interno de cada contenedor web** (parte de la imagen Docker): sirve el SPA Angular y
>    proxea `/api/` al backend por la red interna de Docker. **No se toca; va dentro de cada web.**
> 2. **Nginx de borde = el de BISA** (host separado): termina TLS y enruta cada dominio al puerto del
>    contenedor. **Aquí NO instalamos Nginx; solo entregamos a BISA los `server blocks` del §9.**

Puntos clave que hacen especial este despliegue frente a la doc existente:

1. **La base de datos es externa** (datasource de BISA). Hay que **desactivar el contenedor `mssql`**
   que trae el `docker-compose.yml` y apuntar el backend al host/puerto que dé BISA.
2. **SELinux está activo (enforcing)** en RHEL 9/10. Requiere ajustes que la doc para Ubuntu no menciona.
3. **TLS con CA corporativa** (no ACME): solo instalamos los certificados que entregue BISA.
4. Las **SPA en modo Docker usan API relativa `/api/v1`**; el Nginx de cada contenedor ya la enruta al
   backend por la red interna. **No hay que reconstruir las webs por cambio de dominio del API.**

---

## 0. Antes de empezar — checklist de entrada (verificar el primer día)

Sin estos insumos el despliegue se bloquea. Confírmalos con BISA apenas llegues
(equivale a la hoja «Checklist Provisión» del xlsx):

| # | Insumo que debe dar BISA | Detalle / por qué |
|---|---|---|
| 1 | **Acceso al servidor** | Usuario con `sudo`, por SSH o consola local. RHEL 9/10. |
| 2 | **Datasource de la BD** | `host`, `puerto` (1433), `nombre de BD`, `usuario`, `contraseña`. |
| 3 | **Permisos DDL del usuario de BD** | ⚠️ **Crítico:** al arrancar, Liquibase **crea el esquema** (tablas + `DATABASECHANGELOG`). El usuario debe poder crear objetos en esa BD (idealmente `db_owner` sobre esa base). Si solo tiene lectura/escritura de datos, el backend **no arranca**. |
| 4 | **Dominios + DNS** | `api.<dominio>`, `clientes.<dominio>`, `agentes.<dominio>`, `backoffice.<dominio>` resolviendo a la IP del servidor. |
| 5 | **Certificados TLS (CA corporativa)** | Por dominio o wildcard `*.<dominio>`: **certificado + cadena intermedia (bundle) + llave privada**. Se instalan en **el proxy de BISA** (host separado). |
| 6 | **Red proxy → servidor de apps** | IP del host del **Nginx de BISA**; que su firewall/red alcance el servidor de apps en 8080/4201/4202/4203 (interno). Público (443/80) lo maneja el proxy de BISA. 22 al servidor de apps desde IPs de admin. |
| 7 | **Allowlist saliente** | El backend debe alcanzar: API Corporativo BISA, `oauth2.googleapis.com`/`accounts.google.com`, `walletobjects.googleapis.com`, SMTP, y **`github.com`** (para clonar). |
| 8 | **SMTP corporativo** | `host`, `puerto` (587/465), `usuario`, `contraseña`, remitente (`noreply@<dominio-bisa>`). |
| 9 | **Credenciales API Corporativo BISA** | `BISA_API_BASE_URL`, `BISA_API_AUTH_URL` y token/usuario/clave del servicio. |
| 10 | **Almacenamiento para archivos** | Disco/volumen para `uploads` (PDFs/imágenes de siniestros, credenciales, pagos). |

> **Opcionales / fase posterior (no bloquean el arranque):** credenciales de Google Wallet, Apple Wallet
> y Firebase (push). Si aún no están provisionadas, se despliega sin ellas y esas funciones quedan
> diferidas; el backend arranca igual (solo loguea un *warning*). Ver §7.

---

## 1. Preparación base del sistema operativo (RHEL 9 / 10)

```bash
# Actualizar el sistema
sudo dnf -y update

# Zona horaria y sincronización de reloj (NTP con chrony)
sudo timedatectl set-timezone America/La_Paz        # ajustar a la zona de BISA
sudo dnf -y install chrony
sudo systemctl enable --now chronyd
timedatectl                                          # verificar "System clock synchronized: yes"

# Herramientas base
sudo dnf -y install git curl wget nano tar openssl policycoreutils-python-utils

# Confirmar que SELinux está activo (lo dejamos en enforcing y lo adaptamos; NO lo desactivamos)
getenforce                                           # debe decir: Enforcing
```

> Si el equipo trae un usuario genérico, crea uno de despliegue con sudo:
> `sudo useradd -m -G wheel deploy && sudo passwd deploy` (en RHEL, el grupo `wheel` da sudo).

---

## 2. Instalar Docker CE + Docker Compose

RHEL no trae Docker CE en sus repos (trae Podman). Agregamos el repositorio oficial de Docker.

> ⚠️ **RHEL 10:** el repo oficial de Docker **aún no publica paquetes para RHEL 10** — usar
> `.../linux/rhel/docker-ce.repo` da **404** al resolver `$releasever=10`. Usa una de estas dos
> alternativas (ambas válidas):

```bash
sudo dnf -y install dnf-plugins-core

# --- OPCIÓN A (recomendada en RHEL 10): fijar el repo de Docker a la rama RHEL 9 ---
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo sed -i 's|/rhel/\$releasever/|/rhel/9/|g' /etc/yum.repos.d/docker-ce.repo

# --- OPCIÓN B: usar el repo de CentOS de Docker (compatible con RHEL) ---
# sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# (En RHEL 9 basta con el .repo de rhel sin el sed anterior.)
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Arrancar y habilitar Docker al inicio
sudo systemctl enable --now docker

# Permitir usar docker sin sudo (cerrar y reabrir sesión después)
sudo usermod -aG docker $USER
newgrp docker    # aplica el grupo en la sesión actual sin re-loguear

# Verificar
docker --version
docker compose version
docker run --rm hello-world
```

> **Si hay conflicto con Podman/Buildah** (paquetes preinstalados que chocan):
> `sudo dnf -y remove podman buildah` y reintenta la instalación. Coordina con BISA antes de
> remover Podman si tienen políticas al respecto.

---

## 3. Firewall (firewalld)

En RHEL el firewall es **firewalld** (equivalente a `ufw` de Ubuntu). Como el TLS público lo maneja el
**Nginx de BISA (host separado)**, el servidor de apps **no necesita 443/80 al público**: solo SSH de
administración y que los puertos de los contenedores sean alcanzables **desde el host del proxy de BISA**.

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=ssh     # 22 (idealmente restringido a IPs de admin)
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

> ⚠️ **Gotcha importante (Docker + firewalld en RHEL):** Docker inserta sus propias reglas de iptables
> para los puertos que publica, y **saltan las zonas de firewalld**. Es decir, agregar/quitar puertos con
> `firewall-cmd` **no** controla lo que Docker expone. Para restringir de verdad quién alcanza
> 8080/4201/4202/4203, usa la cadena **`DOCKER-USER`** (se evalúa antes que las reglas de Docker).
> Sustituye `<IP_PROXY_BISA>` por la IP del host donde corre el Nginx de BISA:

```bash
# Permitir solo al proxy de BISA hacia los puertos de contenedor; rechazar el resto
sudo iptables -I DOCKER-USER -p tcp -s <IP_PROXY_BISA> \
     -m multiport --dports 8080,4201,4202,4203 -j ACCEPT
sudo iptables -I DOCKER-USER -p tcp \
     -m multiport --dports 8080,4201,4202,4203 -j DROP
```

> **RHEL 10 — nota sobre iptables:** en RHEL 10 el backend es **nftables** y el comando `iptables` es en
> realidad `iptables-nft` (capa de compatibilidad). La cadena `DOCKER-USER` **sigue existiendo** mientras
> Docker use su backend iptables por defecto (lo normal), así que las reglas de arriba funcionan igual.
> Verifica con `sudo iptables -S DOCKER-USER`.

**Persistir las reglas al reinicio:**

```bash
# RHEL 9: paquete clásico
sudo dnf -y install iptables-services && sudo service iptables save

# RHEL 10: si iptables-services no está disponible, persiste con un servicio systemd oneshot
# que reaplica la regla tras arrancar Docker (crear /etc/systemd/system/docker-user-fw.service
# con ExecStart= las 2 líneas iptables de arriba, After=docker.service, y habilitarlo).
```

> Este control es **defensa en profundidad**: el acceso real proxy→apps también lo restringe la
> **red/firewall de BISA** (allowlist del §0), que es el control primario. El puerto **1433 saliente**
> hacia el SQL de BISA lo gestiona igualmente la red de BISA.

---

## 4. Traer el código fuente

```bash
mkdir -p ~/atlas && cd ~/atlas
git clone <URL_DEL_MONOREPO> .
git submodule update --init --recursive
git checkout <rama-de-produccion>     # confirmar con el equipo (main / release)
git submodule update --init --recursive   # re-sincronizar submódulos tras el checkout
```

Estructura relevante tras clonar:

```
atlas/
├─ bisa-seg-salud-intl-backend-api/
│  ├─ Dockerfile
│  └─ docker/
│     ├─ docker-compose.yml     ← se edita en §5
│     ├─ .env.example           ← plantilla; se copia a .env en §6
│     └─ secrets/               ← certificados de wallet (§7)
├─ bisa-seg-salud-intl-agentes-web/
├─ bisa-seg-salud-intl-backoffice-web/
└─ bisa-seg-salud-intl-clientes-web/
```

---

## 5. Ajustar `docker-compose.yml` para usar la BD externa de BISA

El `docker-compose.yml` viene preparado para **desarrollo**: incluye un contenedor `mssql` y el
`backend-api` apunta a él con valores fijos (`DB_HOST=mssql`, `DB_PORT=1433`). Como BISA nos da la BD
por datasource, hay que **quitar el contenedor `mssql`** y **parametrizar la conexión**.

Edita `bisa-seg-salud-intl-backend-api/docker/docker-compose.yml` con estos 4 cambios:

**5.1 — Eliminar el servicio `mssql` completo.** Borra el bloque `mssql:` (líneas del servicio, desde
`mssql:` hasta antes de `# Backend API Spring Boot`).

**5.2 — En `backend-api`, quitar `depends_on` y parametrizar la BD.**

```yaml
# ── ANTES ─────────────────────────────
      # Base de datos
      - DB_HOST=mssql
      - DB_PORT=1433
      ...
    depends_on:
      mssql:
        condition: service_healthy

# ── DESPUÉS ───────────────────────────
      # Base de datos (datasource externo de BISA — se toma de .env)
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT:-1433}
      # (elimina por completo el bloque depends_on: mssql)
```

**5.3 — Publicar los puertos en la IP interna del servidor** (para que el Nginx de BISA, que está en
otro host, los alcance por la red interna — no en `127.0.0.1`, que solo serviría si el proxy estuviera
en la misma máquina). Define `APP_BIND_IP` en el `.env` (§6) con la IP interna del servidor de apps:

```yaml
# backend-api
    ports:
      - "${APP_BIND_IP}:${SERVER_PORT:-8080}:${SERVER_PORT:-8080}"
# agentes-web
    ports:
      - "${APP_BIND_IP}:${WEB_PORT:-4201}:80"
# backoffice-web
    ports:
      - "${APP_BIND_IP}:${BACKOFFICE_WEB_PORT:-4202}:80"
# clientes-web
    ports:
      - "${APP_BIND_IP}:${CLIENTES_WEB_PORT:-4203}:80"
```

> Atar a la IP interna (en vez de `0.0.0.0`) reduce la exposición, pero el control de acceso real lo
> da la cadena `DOCKER-USER` del §3 (que solo permite al host del proxy de BISA).

**5.4 — SELinux en el mount de secretos + quitar el volumen `mssql_data`.**

```yaml
# En backend-api.volumes, añade ",Z" al bind mount de secrets:
      - ./secrets:/app/secrets:ro,Z

# En la sección volumes: (al final del archivo), elimina la entrada mssql_data:
volumes:
  uploads_data:
    driver: local
  logs_data:
    driver: local
```

> **¿Por qué `:Z`?** SELinux (enforcing) bloquea que el contenedor lea un directorio del host sin la
> etiqueta correcta. `:Z` reetiqueta ese directorio para uso exclusivo del contenedor. Los volúmenes
> con nombre (`uploads_data`, `logs_data`) los gestiona Docker y **no** necesitan `:Z`.

> **Alternativa sin editar el compose:** crear un `docker-compose.override.yml` que redefina los
> `environment` de BD y los `ports`, y levantar con `docker compose up -d --no-deps backend-api ...`.
> Se recomienda la **edición directa** por claridad y para no arrastrar el contenedor `mssql`.

---

## 6. Configurar el `.env` de producción (regenerar secretos)

> ⚠️ **Seguridad — leer primero:** el repositorio incluye un `docker/.env` versionado con **secretos
> reales de desarrollo** (app-password de Gmail, token del API BISA, contraseñas de wallet, etc.).
> **No reutilices ese archivo.** Crea uno nuevo desde la plantilla y **regenera todos los secretos**.

```bash
cd ~/atlas/bisa-seg-salud-intl-backend-api/docker
cp .env.example .env
```

La plantilla **no trae la variable `DB_HOST`** (antes estaba fija en el compose): **hay que agregarla**.
Edita `.env` (`nano .env`) y completa así:

```bash
# ── Ejecución ─────────────────────────
SPRING_PROFILES_ACTIVE=prod
SERVER_PORT=8080
JAVA_OPTS=-Xms1g -Xmx2g            # para servidor de 8 GB RAM
APP_BIND_IP=<ip-interna-del-servidor-de-apps>   # IP que alcanza el proxy de BISA (ver §5.3)

# ── Base de datos (DATASOURCE DE BISA) ─
DB_HOST=<host-sql-de-bisa>          # ← AGREGAR esta línea (no viene en .env.example)
DB_PORT=1433
DB_NAME=<nombre-bd-de-bisa>
DB_USERNAME=<usuario-app-de-bisa>
DB_PASSWORD=<password-de-bisa>

# ── Secretos NUEVOS (generar en sitio) ─
JWT_SECRET=<pegar salida de: openssl rand -base64 48>
WALLET_QR_SECRET=<pegar salida de: openssl rand -base64 48>
WALLET_PARTNER_API_KEY=<pegar salida de: openssl rand -hex 24>

# ── SMTP corporativo de BISA ──────────
MAIL_HOST=<smtp-de-bisa>
MAIL_PORT=587
MAIL_USERNAME=<usuario-smtp>
MAIL_PASSWORD=<password-smtp>
NOTIFICATION_EMAIL_FROM=noreply@<dominio-bisa>
EMAIL_NOTIFICATIONS_ENABLED=true
EMAIL_REDIRECT_ENABLED=false        # ⚠️ SIEMPRE false en producción

# ── API Corporativo BISA ──────────────
BISA_API_BASE_URL=<url-api-bisa>
BISA_API_AUTH_URL=<url-auth-api-bisa>
BISA_SERVICE_TOKEN=<token-de-bisa>

# ── Dominios de producción ────────────
CORS_ALLOWED_ORIGINS=https://clientes.<dominio>,https://agentes.<dominio>,https://backoffice.<dominio>
GOOGLE_WALLET_ORIGIN=https://clientes.<dominio>
PASSWORD_RESET_BASE_URL=https://clientes.<dominio>

# ── Puertos de las webs (defaults ok) ─
WEB_PORT=4201
BACKOFFICE_WEB_PORT=4202
CLIENTES_WEB_PORT=4203
```

Genera los secretos así (copia cada salida al `.env`):

```bash
openssl rand -base64 48    # JWT_SECRET  y  WALLET_QR_SECRET
openssl rand -hex 24       # WALLET_PARTNER_API_KEY
```

Protege el archivo:

```bash
chmod 600 .env
```

> **Wallet / Google / Firebase:** deja `GOOGLE_WALLET_ISSUER_ID`, `APPLE_WALLET_*`, `PUSH_NOTIFICATIONS_ENABLED`
> etc. como en la plantilla (vacíos / `false`) mientras BISA no provisione esas cuentas. Ver §7.

---

## 7. Colocar secretos de archivo (solo si ya están provisionados)

Las credenciales de wallet/push son **archivos** que se montan en `/app/secrets` (directorio
`docker/secrets/`, en `.gitignore`). Solo aplican cuando BISA haya provisionado esas cuentas:

```bash
cd ~/atlas/bisa-seg-salud-intl-backend-api/docker/secrets

# Google Wallet (service account JSON)
cp <origen>/google-wallet-sa.json .

# Apple Wallet (.p12 + certificado WWDR en PEM)
cp <origen>/apple-wallet.p12 .
cp <origen>/AppleWWDRCAG4.pem .

# Firebase (push, fase futura)
cp <origen>/firebase-sa.json .

chmod 600 *
```

Luego completa en `.env` las variables correspondientes (`GOOGLE_WALLET_ISSUER_ID`, `GOOGLE_WALLET_CLASS_ID`,
`APPLE_WALLET_PASS_TYPE_ID`, `APPLE_WALLET_TEAM_ID`, `APPLE_WALLET_CERT_PASSWORD`, `PUSH_NOTIFICATIONS_ENABLED=true`).
Detalle en `apple-wallet-setup.md` y `google-wallet-setup.md`.

> El backend **arranca aunque estos archivos no existan** (se monta el directorio, no cada archivo);
> solo loguea un *warning* y las funciones de wallet quedan inactivas hasta reiniciar con los archivos puestos.

---

## 8. Construir y levantar los contenedores

```bash
cd ~/atlas/bisa-seg-salud-intl-backend-api/docker

# Construir imágenes (backend + 3 webs). Toma varios minutos la primera vez.
docker compose build --no-cache

# Levantar SOLO los 4 servicios (sin mssql, que ya eliminamos)
docker compose up -d backend-api agentes-web backoffice-web clientes-web

# Estado de los contenedores
docker compose ps

# Seguir el arranque del backend: buscar que Liquibase aplique los changelogs
# y la línea "Started BackendApiApplication"
docker compose logs -f backend-api
```

Verificación local (dentro del servidor; usa la IP interna que pusiste en `APP_BIND_IP`):

```bash
curl http://<ip-interna>:8080/actuator/health   # esperado: {"status":"UP"}
curl -I http://<ip-interna>:4203/                # clientes-web responde 200
```

> **Si el backend no arranca:**
> - `Login failed` / `permission denied` en logs → revisar credenciales del datasource (§6) o **permisos DDL**
>   del usuario (Liquibase no puede crear el esquema). Volver al punto 3 del checklist con BISA.
> - `Connection refused / timeout` a la BD → firewall/red entre el servidor de apps y el SQL de BISA (puerto 1433).
> - Verifica la conectividad cruda: `nc -zv <host-sql-de-bisa> 1433`.

---

## 9. Configuración para el Nginx de BISA (reverse proxy + TLS)

> **BISA ya tiene un Nginx en un host separado.** No instalamos Nginx en el servidor de apps: solo
> **entregamos a BISA la configuración** (los `server blocks`) para que la agregue a su proxy. TLS,
> certificados y recarga del servicio los opera **BISA en su host**.

Cada contenedor web ya sirve su SPA y proxea `/api/` al backend por la red interna de Docker. El Nginx de
BISA solo tiene que: (a) terminar TLS con los certificados de la CA corporativa y (b) enrutar cada dominio
al puerto del contenedor correspondiente **en la IP interna del servidor de apps** (`APP_BIND_IP`).

> 📎 **Archivo listo para entregar a BISA:** `atlas-nginx-bisa-proxy.conf` (en esta misma carpeta) trae los
> 4 `server blocks` completos; BISA solo reemplaza `<IP_APPS>`, `<DOMINIO>`, `<RUTA_CERT>` y `<RUTA_KEY>`.
> El detalle a continuación explica el contenido de ese archivo.

**9.1 — Certificados (los instala BISA en su proxy).** El bundle debe incluir **la cadena intermedia** de
la CA corporativa; si no, algunos clientes (Android/iOS incluidos) rechazan el certificado. Un wildcard
`*.<dominio>` cubre los 4 subdominios.

**9.2 — Contenido de la configuración.** El bloque completo está en `atlas-nginx-bisa-proxy.conf` (no se
duplica aquí para que no se desincronice). Trae un `server block` por dominio; cada uno hace: redirección
`80→443`, TLS 1.2/1.3 con el certificado de BISA, cabeceras de seguridad (HSTS, X-Frame-Options,
X-Content-Type-Options), `client_max_body_size 10M` (coincide con `MAX_FILE_SIZE` del backend) y
`proxy_pass` al puerto del contenedor en `<IP_APPS>` con las cabeceras `X-Forwarded-*`. Tokens a
reemplazar: `<IP_APPS>`, `<DOMINIO>`, `<RUTA_CERT>`, `<RUTA_KEY>`.

Mapa dominio → contenedor:

| Dominio | `proxy_pass` |
|---|---|
| `api.<DOMINIO>`        | `http://<IP_APPS>:8080` (+ `location /.well-known/` → `:4203`, y `/actuator/health`) |
| `clientes.<DOMINIO>`   | `http://<IP_APPS>:4203` |
| `agentes.<DOMINIO>`    | `http://<IP_APPS>:4201` |
| `backoffice.<DOMINIO>` | `http://<IP_APPS>:4202` |

> ⚠️ En todos los bloques, `proxy_set_header X-Forwarded-Proto $scheme` es imprescindible: como el TLS
> termina en el proxy, sin esa cabecera el backend generaría los enlaces (reset de contraseña, wallet) en `http`.

**9.3 — Puntos a coordinar con BISA sobre su proxy:**
- Que el firewall del servidor de apps (cadena `DOCKER-USER`, §3) permita al **host del proxy** hacia
  8080/4201/4202/4203.
- **`X-Forwarded-Proto $scheme`** debe llegar al backend (arriba) para que los enlaces que genera
  (reset de contraseña, wallet) salgan en `https`. Si BISA usa un balanceador adicional, que preserve
  esa cabecera.
- **Deep links:** `assetlinks.json` (huella SHA-256 del keystore Android) y `apple-app-site-association`
  (Team ID + Bundle ID) deben apuntar al dominio definitivo; coordinar con el equipo móvil.
- Si el proxy de BISA es RHEL con SELinux: en **ese** host se requiere `setsebool -P httpd_can_network_connect on`
  (lo aplica BISA, no nosotros).
- BISA valida con `nginx -t` y recarga (`systemctl reload nginx`) en su host.

---

## 10. Verificación end-to-end (smoke tests)

Ejecutar desde una máquina con acceso a los dominios:

1. **Salud del API por HTTPS:**
   `curl -I https://api.<dominio>/actuator/health` → `200 OK`, certificado válido (sin `-k`).
2. **Portales web:** abrir `https://clientes.<dominio>`, `https://agentes.<dominio>`,
   `https://backoffice.<dominio>`; deben cargar y **no** mostrar errores CORS en la consola del navegador.
3. **Login + 2FA:** iniciar sesión en un portal; confirmar que llega el **OTP por correo**
   (valida SMTP de BISA) y que el token JWT funciona.
4. **Archivos privados:** subir y **descargar** un documento (siniestro / credencial). Valida el volumen
   `uploads_data` y el patrón de acceso autenticado a archivos (ver sección «File Storage» de `CLAUDE.md`).
5. **API Corporativo BISA:** ejecutar una operación que consulte pólizas/afiliados (valida la allowlist
   saliente y las credenciales del API interno).
6. **Logs y volúmenes:**
   ```bash
   docker compose logs --tail=100 backend-api      # sin ERROR
   docker volume ls | grep -E 'uploads_data|logs_data'
   ```

---

## 11. Operación y post-despliegue

- **Reinicio automático:** los servicios tienen `restart: unless-stopped`; sobreviven reinicios del host.
- **Backups:**
  - Base de datos → **la respalda BISA** (retención 7–30 días fuera del servidor de BD).
  - Archivos subidos → respaldar `uploads_data`:
    ```bash
    docker run --rm -v docker_uploads_data:/source:ro -v $(pwd):/backup alpine \
      tar czf /backup/uploads-$(date +%F).tar.gz -C /source .
    ```
- **Logs:** rotación ya configurada (100 MB × 30 archivos, volumen `logs_data`).
- **Actualizaciones futuras:**
  ```bash
  cd ~/atlas && git pull && git submodule update --remote --merge
  cd bisa-seg-salud-intl-backend-api/docker
  docker compose build --no-cache
  docker compose up -d backend-api agentes-web backoffice-web clientes-web
  ```
- **Renovación de certificados:** viven en el **proxy de BISA**; la renovación (CA corporativa) la opera
  BISA en su host. Definir con ellos el proceso y la ventana de recarga.
- **Higiene de secretos:** el `.env` de producción **no se versiona**; considera rotar los secretos que
  quedaron expuestos en el `docker/.env` del repositorio.

---

## Referencias

- `Requerimientos_Tecnicos_Infraestructura_Despliegue.xlsx` — requerimientos e insumos de BISA.
- `nginx-ssl-setup-guide.md` — bloques Nginx detallados (base del §9; aquí adaptados a `conf.d/` y CA corporativa).
- `Guia_Deployment_Docker.md`, `DEPLOYMENT_VPS.md` — runbooks originales (Ubuntu).
- `gmail-smtp-configuration.md` — referencia SMTP.
- `apple-wallet-setup.md`, `google-wallet-setup.md` — provisión y montaje de secretos de wallet.
- `CLAUDE.md` (raíz) — patrón de acceso a archivos privados (relevante para el smoke test 4).

## Fuera de alcance de esta guía

Instalación/administración del motor SQL Server (lo hace BISA), publicación de las apps iOS/Android en las
tiendas, y provisión de cuentas externas (Apple Developer, Google Play, Google Cloud/Wallet). Se referencian
pero no se ejecutan en este despliegue de servidor.
