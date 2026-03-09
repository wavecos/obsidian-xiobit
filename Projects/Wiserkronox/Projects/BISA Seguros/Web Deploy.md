# BISA SEGUROS - Web Deply

Documento para poder realizar el cambio de variables de configuración para realizar un deploy.
## 1. Detener el servicio

```
sudo systemctl stop bisa_seguros_web.service 
```
## 2. Editar las variables de ambiente
```
sudo systemctl edit bisa_seguros_web.service
```
Ejemplo de Variables de ambiente, que se pueden editar los valores:
```
[Service]
Environment="BISA_API_BASE_URL=https://dev.graphapibisaseg.com"
Environment="BISA_API_CLIENT_USER=wiserwebapp"
Environment="BISA_API_CLIENT_PASSWORD=wiserwebapp"
Environment="BISA_API_DEMO_MODE=true"
Environment="DATASOURCE_URL=jdbc:sqlserver://localhost:1433;database=bisa_seguros_web;trustServerCertificate=true"
Environment="DATASOURCE_USERNAME=SA"
Environment="DATASOURCE_PASSWORD="yourStrong(!)Password""
Environment="SERVER_PORT=8080"
Environment="HOST_BASE_URL=https://bisaseg.xiobit.com"
Environment="EXTERNAL_IMAGE_STORAGE=/home/ubuntu/bisa_seguros/external/images/"
Environment="ELASTICSEARCH_URL=http://localhost:9200"
Environment="BISA_API_WEBHOOK_APIKEY=af19f25e010a60f08f3746250e9e36cacc8c95f300ba94fcd2ffce83a9cf5fd2"
```
Desarrollo:

```
[Service]
Environment="DATASOURCE_URL=jdbc:sqlserver://10.10.1.185:1433;database=SB_CERTPC004_DEV;trustServerCertificate=true"
Environment="DATASOURCE_USERNAME=UsrSqlWiserRX"
Environment="DATASOURCE_PASSWORD=BisaSeguros.2024"
Environment="SERVER_PORT=8080"
Environment="HOST_BASE_URL=http://10.10.1.96:8080"
Environment="EXTERNAL_IMAGE_STORAGE=/media/images/"
```
## 3. Levantar el servicio
```
sudo systemctl start bisa_seguros_web.service
```
## 4. Monitorear el Servicio (ver logs)
```
journalctl -u bisa_seguros_web.service -f 
```
Salir con CRTL-C
