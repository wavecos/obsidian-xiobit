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
> 2. **Nginx de borde:** termina TLS y enruta cada dominio al puerto del contenedor. En **producción** es
>    el **Nginx de BISA** (host separado) y solo le entregamos los `server blocks` (§9.B); en **staging**
>    instalamos un Nginx en el mismo servidor (§9.A). Ver «Ambientes» abajo.

---

## Ambientes: staging vs producción

La plataforma se despliega en dos ambientes con **topología de red distinta**. Los pasos **1, 2, 4, 6, 7, 8
son idénticos**; solo cambian el **firewall (§3)**, el **binding de puertos (§5.3)** y el
**reverse proxy + TLS (§9)**. El diagrama de arriba es la topología de **producción**.

| Aspecto | 🟠 **Staging** (todo en 1 servidor) | 🔵 **Producción** |
|---|---|---|
| Reverse proxy + TLS | **Nginx propio en el MISMO servidor** (lo instalamos, §9.A) | **Nginx de BISA** en host separado (§9.B) |
| `APP_BIND_IP` (binding) | `127.0.0.1` (el nginx local llega por loopback) | IP interna del servidor (el proxy de BISA llega por red) |
| Firewall entrante | **443/80 públicos** en el propio server | Solo el proxy de BISA alcanza 8080/4201-4203; 443/80 los sirve BISA |
| Regla `DOCKER-USER` | **No hace falta** (contenedores en 127.0.0.1) | Sí (restringe al host del proxy) |
| Certificados | CA corporativa o self-signed (uso interno) | CA corporativa de BISA |
| Base de datos | Datasource de BISA en otro servidor (**igual que prod**) | Datasource de BISA |
| Almacenamiento de archivos | Directorio en disco local del server (menor) | **Disco dedicado 500 GB–1 TB** montado (BISA) |

> En cada paso marcado sigue la caja **🟠 Staging** o **🔵 Producción** según corresponda.

Puntos clave que hacen especial este despliegue frente a la doc existente:

1. **La base de datos es externa** (datasource de BISA, en otro servidor) **en ambos ambientes**. Hay que
   **desactivar el contenedor `mssql`** que trae el `docker-compose.yml` y apuntar el backend al host/puerto de BISA.
2. **SELinux está activo (enforcing)** en RHEL 9/10. Requiere ajustes que la doc para Ubuntu no menciona.
3. **TLS sin ACME:** en producción con la CA corporativa de BISA; en staging puede usarse self-signed.
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
| 5 | **Certificados TLS** | Por dominio o wildcard `*.<dominio>`: **certificado + cadena intermedia (bundle) + llave privada**. 🔵 Prod: se instalan en el proxy de BISA. 🟠 Staging: en el Nginx local del servidor (CA corporativa o self-signed). |
| 6 | 🔵 **Red proxy → servidor de apps** (solo prod) | IP del host del **Nginx de BISA**; que su firewall/red alcance el servidor de apps en 8080/4201/4202/4203 (interno). Público (443/80) lo maneja el proxy de BISA. 22 al servidor de apps desde IPs de admin. |
| 7 | **Allowlist saliente** | El backend debe alcanzar: API Corporativo BISA, `oauth2.googleapis.com`/`accounts.google.com`, `walletobjects.googleapis.com`, SMTP, y **`github.com`** (para clonar). |
| 8 | **SMTP corporativo** | `host`, `puerto` (587/465), `usuario`, `contraseña`, remitente (`noreply@<dominio-bisa>`). |
| 9 | **Credenciales API Corporativo BISA** | `BISA_API_BASE_URL`, `BISA_API_AUTH_URL` y token/usuario/clave del servicio. |
| 10 | **Disco dedicado para archivos** | 🔵 Prod: disco persistente **500 GB–1 TB** montado en el SO (p. ej. `/data/atlas/uploads`), en `/etc/fstab`; ~25 GB iniciales, crece con uso. 🟠 Staging: directorio en el disco local. Para `uploads` (PDFs/imágenes de siniestros, credenciales, pagos). |

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

En RHEL el firewall es **firewalld** (equivalente a `ufw` de Ubuntu). Base común (ambos ambientes):

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=ssh     # 22 (idealmente restringido a IPs de admin)
```

🟠 **Staging** — el Nginx local termina TLS en el propio servidor, así que abre 443/80 al público:

```bash
sudo firewall-cmd --permanent --add-service=http    # 80  → redirección a HTTPS
sudo firewall-cmd --permanent --add-service=https   # 443
```

🔵 **Producción** — el TLS lo sirve el Nginx de BISA (host separado); el servidor de apps **NO** abre
443/80. Solo hace falta que el host del proxy alcance los puertos de contenedor (regla `DOCKER-USER` abajo).

Aplica y verifica (en ambos casos):

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

> 🔵 ⚠️ **Solo producción — gotcha Docker + firewalld en RHEL** (en staging NO hace falta: los
> contenedores quedan en `127.0.0.1`, inaccesibles desde fuera). Docker inserta sus propias reglas de
> iptables para los puertos que publica, y **saltan las zonas de firewalld**: agregar/quitar puertos con
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

> Estos 4 cambios aplican **igual a staging y a producción** (la BD es datasource externo en ambos). La
> única diferencia entre ambientes es el valor de `APP_BIND_IP` en 5.3 (staging `127.0.0.1`, prod IP interna).

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

**5.3 — Publicar los puertos según el ambiente.** El compose usa una sola variable `APP_BIND_IP` (definida
en el `.env`, §6) para atar los puertos:
- 🟠 **Staging:** `APP_BIND_IP=127.0.0.1` → los contenedores solo son accesibles por loopback; el Nginx
  local (§9.A) los alcanza y nada queda expuesto a la red.
- 🔵 **Producción:** `APP_BIND_IP=<ip-interna-del-servidor>` → el Nginx de BISA (host separado) los alcanza
  por la red interna. El acceso se restringe con la regla `DOCKER-USER` del §3.

El bloque de `ports` es el mismo en ambos casos:

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

> Nunca uses `0.0.0.0` (expondría los contenedores en todas las interfaces). En producción, además del
> binding, el control de acceso real lo da la cadena `DOCKER-USER` del §3.

**5.4 — Volúmenes: uploads a disco dedicado, `:Z` en secretos, quitar `mssql_data`.**

```yaml
# backend-api.volumes → así queda (3 entradas):
      - ${UPLOADS_HOST_DIR}:/app/uploads:Z    # disco dedicado de archivos (§5.5)
      - logs_data:/var/log/bisa-seguros
      - ./secrets:/app/secrets:ro,Z           # ,Z para SELinux

# Sección volumes: (al final) → quita mssql_data Y uploads_data (uploads ahora es bind mount):
volumes:
  logs_data:
    driver: local
```

> **¿Por qué `:Z`?** SELinux (enforcing) bloquea que el contenedor use un directorio del host sin la
> etiqueta correcta. `:Z` reetiqueta el directorio para uso exclusivo del contenedor. `logs_data` es
> volumen Docker con nombre (lo gestiona Docker) y **no** necesita `:Z`.

**5.5 — Almacenamiento de archivos en disco dedicado.** Los `uploads` (PDFs/imágenes de siniestros,
credenciales, pagos) crecen con el uso (~25 GB iniciales; estimar **500 GB–1 TB**). Deben ir a un
**disco/volumen persistente dedicado**, nunca al disco del SO.

- 🔵 **Producción:** BISA provee un disco dedicado (500 GB–1 TB) montado en el SO, p. ej. `/data/atlas/uploads`.
- 🟠 **Staging:** un directorio en el disco del servidor basta (menor tamaño), mismo mecanismo.

Prepara el punto de montaje y los permisos (el backend corre como uid **1001** — usuario `appuser` del
contenedor, definido en el `Dockerfile`):

```bash
df -h /data/atlas/uploads          # confirma el disco montado (y que esté en /etc/fstab para el reboot)
sudo mkdir -p /data/atlas/uploads
sudo chown -R 1001:1001 /data/atlas/uploads   # el contenedor escribe como uid 1001
```

Define la ruta en el `.env` (§6): `UPLOADS_HOST_DIR=/data/atlas/uploads`. El compose la monta en
`/app/uploads` con `:Z`. (El contenedor sigue usando `UPLOAD_DIR=/app/uploads` internamente.)

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
APP_BIND_IP=<ip-interna>   # staging: 127.0.0.1 · prod: IP interna del servidor (ver §5.3)

# ── Base de datos (DATASOURCE DE BISA) ─
DB_HOST=<host-sql-de-bisa>          # ← AGREGAR esta línea (no viene en .env.example)
DB_PORT=1433
DB_NAME=<nombre-bd-de-bisa>
DB_USERNAME=<usuario-app-de-bisa>
DB_PASSWORD=<password-de-bisa>

# ── Almacenamiento de archivos ────────
UPLOADS_HOST_DIR=/data/atlas/uploads   # disco dedicado montado en el SO (ver §5.5)

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

# Levantar los 4 servicios (sin mssql, que ya eliminamos en §5) — igual en staging y prod
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

## 9. Reverse proxy + TLS

El mismo conjunto de `server blocks` sirve para ambos ambientes; solo cambia **dónde** corre el Nginx y el
valor de `<IP_APPS>`. El archivo `atlas-nginx-bisa-proxy.conf` (en esta carpeta) trae los 4 bloques
completos — un `server block` por dominio, cada uno con: redirección `80→443`, TLS 1.2/1.3, cabeceras de
seguridad (HSTS, X-Frame-Options, X-Content-Type-Options), `client_max_body_size 10M` (coincide con
`MAX_FILE_SIZE`) y `proxy_pass` al puerto del contenedor con las cabeceras `X-Forwarded-*`.

Mapa dominio → contenedor (idéntico en staging y prod):

| Dominio | `proxy_pass` |
|---|---|
| `api.<DOMINIO>`        | `http://<IP_APPS>:8080` (+ `location /.well-known/` → `:4203`, y `/actuator/health`) |
| `clientes.<DOMINIO>`   | `http://<IP_APPS>:4203` |
| `agentes.<DOMINIO>`    | `http://<IP_APPS>:4201` |
| `backoffice.<DOMINIO>` | `http://<IP_APPS>:4202` |

Tokens a reemplazar en el `.conf`: `<IP_APPS>`, `<DOMINIO>`, `<RUTA_CERT>`, `<RUTA_KEY>`.

> ⚠️ En todos los bloques, `proxy_set_header X-Forwarded-Proto $scheme` es imprescindible: como el TLS
> termina en el proxy, sin esa cabecera el backend generaría los enlaces (reset de contraseña, wallet) en `http`.

---

### 9.A — 🟠 Staging: Nginx en el MISMO servidor

Instalamos y operamos nosotros el Nginx local. Aquí **`<IP_APPS>` = `127.0.0.1`**.

```bash
sudo dnf -y install nginx
sudo systemctl enable --now nginx
# SELinux: permitir que Nginx proxee a los puertos locales de Docker
sudo setsebool -P httpd_can_network_connect on
```

1. Copiar el certificado (CA corporativa o **self-signed** para uso interno) y la llave, p. ej.
   `/etc/pki/tls/certs/atlas.crt` y `/etc/pki/tls/private/atlas.key` (llave `chmod 600`).
2. Copiar `atlas-nginx-bisa-proxy.conf` a **`/etc/nginx/conf.d/atlas.conf`** y reemplazar tokens:
   `<IP_APPS>`→`127.0.0.1`, `<DOMINIO>`→dominio de staging, `<RUTA_CERT>`/`<RUTA_KEY>`→las rutas de arriba.
3. Validar y recargar:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

> Con self-signed los navegadores mostrarán advertencia (aceptable en staging interno); las apps móviles
> apuntando a staging pueden requerir confiar el certificado.

---

### 9.B — 🔵 Producción: Nginx de BISA (host separado)

BISA ya tiene su Nginx; **no instalamos Nginx aquí**. Solo **entregamos** `atlas-nginx-bisa-proxy.conf`
para que BISA lo agregue a su proxy. Aquí **`<IP_APPS>` = IP interna del servidor de apps**. TLS,
certificados y recarga los opera BISA en su host.

**Puntos a coordinar con BISA:**
- Que el firewall del servidor de apps (regla `DOCKER-USER`, §3) permita al **host del proxy** hacia 8080/4201/4202/4203.
- El certificado debe incluir **la cadena intermedia** de la CA corporativa (o Android/iOS lo rechazan);
  un wildcard `*.<dominio>` cubre los 4 subdominios.
- **`X-Forwarded-Proto`** debe llegar al backend (si hay un balanceador adicional, que preserve la cabecera).
- **Deep links:** `assetlinks.json` (huella SHA-256 del keystore Android) y `apple-app-site-association`
  (Team ID + Bundle ID) deben apuntar al dominio definitivo; coordinar con el equipo móvil.
- Si el proxy de BISA es RHEL con SELinux: `setsebool -P httpd_can_network_connect on` (lo aplica BISA).
- BISA valida con `nginx -t` y recarga en su host.

---

## 10. Verificación end-to-end (smoke tests)

Ejecutar desde una máquina con acceso a los dominios:

1. **Salud del API por HTTPS:**
   `curl -I https://api.<dominio>/actuator/health` → `200 OK`, certificado válido (sin `-k`).
2. **Portales web:** abrir `https://clientes.<dominio>`, `https://agentes.<dominio>`,
   `https://backoffice.<dominio>`; deben cargar y **no** mostrar errores CORS en la consola del navegador.
3. **Login + 2FA:** iniciar sesión en un portal; confirmar que llega el **OTP por correo**
   (valida SMTP de BISA) y que el token JWT funciona.
4. **Archivos privados:** subir y **descargar** un documento (siniestro / credencial). Valida el disco de
   uploads (`/data/atlas/uploads`) y el patrón de acceso autenticado a archivos (ver «File Storage» de `CLAUDE.md`).
5. **API Corporativo BISA:** ejecutar una operación que consulte pólizas/afiliados (valida la allowlist
   saliente y las credenciales del API interno).
6. **Logs y volúmenes:**
   ```bash
   docker compose logs --tail=100 backend-api      # sin ERROR
   du -sh /data/atlas/uploads                       # el disco de uploads crece al subir archivos
   docker volume ls | grep logs_data
   ```

---

## 11. Operación y post-despliegue

- **Reinicio automático:** los servicios tienen `restart: unless-stopped`; sobreviven reinicios del host.
- **Backups:**
  - Base de datos → **la respalda BISA** (retención 7–30 días fuera del servidor de BD).
  - Archivos subidos → respaldar el disco dedicado (o snapshot del volumen por BISA):
    ```bash
    tar czf /backup/uploads-$(date +%F).tar.gz -C /data/atlas/uploads .
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
