# 1. Bloqueantes

### Correo (SMTP corporativo). 

Se debe proporcionar:
- Host, puerto (587 o 465), usuario y contraseña. Si su servidor de correo acepta envíos por IP sin usuario, que lo digan.
- El remitente oficial (noreply@…), y que el usuario SMTP tenga permiso para enviar con ese remitente.
- SPF, DKIM y DMARC del dominio configurados para su servidor de correo. Si no, los OTP terminan en spam o se pierden sin aviso.
- El límite de envíos por hora o por día.
- Los dominios internos para ALERTAS_INTERNAS_DOMINIOS y los buzones del equipo que recibe las Alertas Internas de Cartas de Garantía.
### Almacenamiento. 

Actualmente es un directorio local. FileStorageService solo sabe escribir en disco local; no hay soporte para S3 ni NAS. Lo acordado en la guía es un disco dedicado. Pedir:
- Disco de 500 GB a 1 TB montado (por ejemplo /data/atlas/uploads), que quede persistente.
- Qué tipo de disco es: local, SAN o NFS. Si es NFS, que permita escribir al uid 1001 y funcione con SELinux.
- Política de backup de ese disco (frecuencia, retención, quién restaura). Ahí van documentos médicos.

### Base de datos (SQL Server de BISA).
- Backups de la base, que los administra BISA.
- Certificado del SQL: hoy usamos trustServerCertificate=true. Preguntar si lo aceptan o si nos dan su CA.

## 2. Integraciones

### Freshdesk:
  - Una API key de un usuario de servicio (no de una persona) y los valores finales de los
  - Revisar los campos de  Cartas de Garantía: tipo de ticket, grupo, claves cf_* y el valor de cf_operadores.
  - Que configuren el webhook según freshdesk-webhook-reembolsos.md * Revisar
  - Confirmar el SLA de 72 h de Cartas de Garantía.

### Confirmar de quién son las cuentas:
  - El Google Wallet usa una cuenta de servicio del proyecto bisa-salud-dev. Hace falta uno de producción, con Issuer ID y Class ID.
  - Firebase está en el proyecto bisa-salud-intl. Confirmar si ese proyecto es de BISA.
  - Los client IDs de Google Sign-In que vienen por defecto en application.yml:192-195.
  - Apple Wallet: Pass Type ID, Team ID y el certificado .p12.
  - Las cuentas de Play Console y App Store Connect, que deberían estar a nombre de BISA.
- Otros valores:
  - GOOGLE_WALLET_ORIGIN hoy es salud.bisaseguros.com. Preguntar si es el dominio correcto.
  - La página https://www.atlasglobalhealth.com/wallet/ (la que lleva el QR) tiene que exi

## 3. Operación después de la salida

- Monitoreo: el backend expone /actuator/prometheus por si lo quieren conectar.
- Ventana de mantenimiento y contactos de guardia de cada lado.
- Quién renueva los certificados TLS y quién vigila el espacio del disco de archivos.   
