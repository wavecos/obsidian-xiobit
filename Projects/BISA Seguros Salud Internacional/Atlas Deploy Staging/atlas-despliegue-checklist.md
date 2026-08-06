# ✅ Checklist de despliegue — Atlas en RHEL 9 / 10 (BISA)

> Hoja de ruta para el día del despliegue. Detalle completo en **`despliegue-rhel-atlas.md`**.
> Config del proxy en **`atlas-nginx-bisa-proxy.conf`**.
>
> **Dos ambientes** — 🟠 **Staging** (todo en 1 server, Nginx local) · 🔵 **Producción** (Nginx de BISA
> en host separado). Los pasos sin marca aplican a ambos.

---

## A. Insumos de BISA (verificar ANTES de arrancar)

- [x] Acceso al servidor RHEL 9/10 (usuario con `sudo`)
- [x] Datasource BD (externo, otro servidor — **uno por ambiente**): **host · puerto · nombre · usuario · contraseña**
- [x] ⚠️ Usuario de BD con **permisos DDL** (db_owner) — Liquibase crea el esquema al arrancar
- [ ] Dominios + DNS: `api.` / `clientes.` / `agentes.` / `backoffice.`
- [ ] Certificado TLS (**con cadena intermedia**) — 🔵 prod: en el proxy de BISA · 🟠 staging: en el Nginx local (CA corp. o self-signed)
- [ ] 🔵 (prod) IP del **host del proxy Nginx** de BISA
- [ ] SMTP: host · puerto · usuario · contraseña · remitente
- [ ] Credenciales API Corporativo BISA (base URL, auth URL, token)
- [ ] Allowlist saliente: API BISA · Google · SMTP · **github.com**
- [x] IP interna del servidor de apps (`APP_BIND_IP`)

---

## B. Servidor de apps (RHEL 9/10) — paso a paso

**1. Base**
- [x] `sudo dnf -y update`
- [x] Zona horaria + NTP (`chronyd` activo)
- [x] `git curl openssl policycoreutils-python-utils` instalados · `getenforce` = Enforcing

**2. Docker CE**
- [x] Repo Docker (⚠️ RHEL 10: el repo `rhel` da 404 → fijar rama 9 con `sed .../rhel/9/...` o usar repo `centos`)
- [x] `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
- [x] `systemctl enable --now docker` · usuario en grupo `docker`
- [x] `docker run --rm hello-world` OK

**3. Firewall (firewalld)**
- [ ] Base: `ssh`
- [ ] 🟠 Staging: abrir `http` + `https` (443/80 públicos, el Nginx local termina TLS)
- [ ] 🔵 Prod: NO abrir 443/80; regla `DOCKER-USER` permite IP del proxy → 8080/4201/4202/4203, DROP el resto
- [ ] 🔵 Prod: persistir DOCKER-USER — RHEL 9 `iptables-services`+`service iptables save` · RHEL 10 systemd oneshot

**4. Código**
- [x] `git clone` + `git submodule update --init --recursive` + checkout rama prod

**4b. Almacenamiento (disco dedicado de archivos)**
- [ ] Disco montado y en `/etc/fstab` (🔵 prod **500 GB–1 TB** · 🟠 staging dir en disco local)
- [ ] `sudo mkdir -p /data/atlas/uploads` + `sudo chown -R 1001:1001 /data/atlas/uploads` (uid del contenedor)

**5. Editar `docker/docker-compose.yml`**
- [x] Eliminar servicio `mssql` completo
- [x] `backend-api`: quitar `depends_on: mssql`; `DB_HOST=${DB_HOST}`, `DB_PORT=${DB_PORT:-1433}`
- [x] Puertos vía `${APP_BIND_IP}` (🟠 staging=`127.0.0.1` · 🔵 prod=IP interna): 8080/4201/4202/4203
- [x] Uploads a disco dedicado: `${UPLOADS_HOST_DIR}:/app/uploads:Z` · quitar volumen nombrado `uploads_data`
- [x] Bind mount secrets con SELinux: `./secrets:/app/secrets:ro,Z`
- [x] Quitar volumen `mssql_data` (dejar solo `logs_data`)

**6. `.env` de producción** (`cp .env.example .env`, `chmod 600 .env`)
- [ ] ⚠️ NO reutilizar el `.env` del repo (tiene secretos vivos)
- [x] `APP_BIND_IP` + `UPLOADS_HOST_DIR` + `DB_HOST` (agregar línea) + `DB_PORT/NAME/USERNAME/PASSWORD`
- [ ] Regenerar: `JWT_SECRET`, `WALLET_QR_SECRET` (`openssl rand -base64 48`), `WALLET_PARTNER_API_KEY` (`openssl rand -hex 24`)
- [ ] SMTP · `NOTIFICATION_EMAIL_FROM` · `EMAIL_REDIRECT_ENABLED=false`
- [ ] `BISA_API_BASE_URL` / `_AUTH_URL` / `_SERVICE_TOKEN`
- [ ] `CORS_ALLOWED_ORIGINS` · `GOOGLE_WALLET_ORIGIN` · `PASSWORD_RESET_BASE_URL` (dominios prod)
- [ ] `JAVA_OPTS=-Xms1g -Xmx2g`

**7. Secretos wallet/push** (solo si ya provisionados) → `docker/secrets/` con `chmod 600`

**8. Levantar**
- [ ] `docker compose build --no-cache`
- [ ] `docker compose up -d backend-api agentes-web backoffice-web clientes-web`
- [ ] Logs: Liquibase aplica changelogs + `Started BackendApiApplication`
- [ ] `curl http://<ip-interna>:8080/actuator/health` → `{"status":"UP"}`

---

## C. Reverse proxy + TLS (§9)

**🟠 Staging — Nginx local en el mismo server:**
- [ ] `dnf install nginx` + `systemctl enable --now nginx` + `setsebool -P httpd_can_network_connect on`
- [ ] Cert (CA corp. o self-signed) + llave en `/etc/pki/tls/...`
- [ ] `atlas-nginx-bisa-proxy.conf` → `/etc/nginx/conf.d/atlas.conf`; `<IP_APPS>`=`127.0.0.1`, `<DOMINIO>`, rutas cert
- [ ] `nginx -t` · `systemctl reload nginx`

**🔵 Producción — Nginx de BISA (host separado):**
- [ ] Entregar `atlas-nginx-bisa-proxy.conf`; `<IP_APPS>`=IP interna, `<DOMINIO>`, `<RUTA_CERT>`, `<RUTA_KEY>`
- [ ] Certificado con cadena intermedia instalado (en el proxy de BISA)
- [ ] Proxy pasa `X-Forwarded-Proto` al backend
- [ ] BISA: `setsebool` si es RHEL · `nginx -t` · `reload`

---

## D. Smoke tests (Go / No-Go)

- [ ] `curl -I https://api.<dominio>/actuator/health` → 200, TLS válido (sin `-k`)
- [ ] Las 3 webs cargan por su dominio, sin errores CORS en consola
- [ ] Login + 2FA: llega el OTP por correo (valida SMTP + JWT)
- [ ] Subir y descargar un documento privado (valida el disco de uploads `/data/atlas/uploads`)
- [ ] Una operación que consulte el API Corporativo BISA responde
- [ ] `docker compose logs backend-api` sin ERROR

---

## Bloqueos típicos

| Síntoma | Causa probable |
|---|---|
| Backend no arranca, `permission denied` en logs | Usuario de BD sin permisos DDL (Liquibase) |
| `Connection refused` a la BD | Red/firewall al SQL de BISA (`nc -zv <host> 1433`) |
| Contenedor no lee `secrets/` | Falta `:Z` (SELinux) en el bind mount |
| Backend no guarda archivos / `AccessDenied` en uploads | Disco de uploads sin `chown 1001:1001` o falta `:Z` |
| Web da 502 desde el proxy de BISA | Falta regla `DOCKER-USER` o binding en `APP_BIND_IP` |
| Enlaces de correo/wallet salen en `http` | Proxy no envía `X-Forwarded-Proto` |
| TLS rechazado en móvil | Certificado sin cadena intermedia |
