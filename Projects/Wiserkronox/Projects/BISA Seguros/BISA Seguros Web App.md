### Descripción General
   - **Lenguajes Frameworks y Tecnologías**: 
	   - Java
	   - Spring Boot
	   - Maven
	   - Angular
	   - TypeScript
	   - Node.js
	   - npm
	   - ElasticSearch
   - **Componentes Principales**: 
	   - Backend (Spring Boot)
	   - Frontend (Angular)
	   - Base de Datos
	   - Servicios Externos
### Arquitectura
   - **Frontend**: Angular
     - Comunicación con el backend a través de HTTP/REST.
   - **Backend**: Spring Boot
     - Controladores REST.
     - Servicios.
     - Repositorios.
     - Seguridad (JWT).
   - **Base de Datos**: 
	- MS SQL Server
   - **Servicios Externos**: 
	   - Integración con servicios externos como Firebase para notificaciones push
	   - Busqueda con Elasticsearch
### Detalles de Componentes
   - **Frontend (Angular)**:
     - Componentes.
     - Servicios.
     - Módulos.
   - **Backend (Spring Boot)**:
     - Controladores.
     - Servicios.
     - Repositorios.
     - Configuración de Seguridad.
   - **Base de Datos**:
     - Entidades.
     - Repositorios.
   - **Servicios Externos**:
     - Integración con APIs externas.
### Flujo de Datos
   - **Usuario** interactúa con la **Aplicación Angular**.
   - **Angular** realiza peticiones HTTP al **Backend Spring Boot**.
   - **Spring Boot** procesa las peticiones, interactúa con la **Base de Datos** y otros **Servicios Externos**.
   - **Spring Boot** responde a **Angular** con los datos solicitados.
   - **Angular** actualiza la interfaz de usuario.
### Seguridad
   - Autenticación y Autorización con JWT.
   - Configuración de CORS para permitir peticiones desde orígenes específicos.

### Diagrama de Arquitectura

```plaintext
+-------------------+        +-------------------+        +-------------------+
|                   |        |                   |        |                   |
|    Usuario        |        |    Angular        |        |    Spring Boot    |
|                   |        |                   |        |                   |
+-------------------+        +-------------------+        +-------------------+
         |                           |                           |
         |                           |                           |
         |                           |                           |
         v                           v                           v
+-------------------+        +-------------------+        +-------------------+
|                   |        |                   |        |                   |
|    Navegador      |        |    Componentes    |        |    Controladores  |
|                   |        |                   |        |                   |
+-------------------+        +-------------------+        +-------------------+
         |                           |                           |
         |                           |                           |
         v                           v                           v
+-------------------+        +-------------------+        +-------------------+
|                   |        |                   |        |                   |
|    HTTP Request   | -----> |    Servicios      | -----> |    Servicios      |
|                   |        |                   |        |                   |
+-------------------+        +-------------------+        +-------------------+
         |                           |                           |
         |                           |                           |
         v                           v                           v
+-------------------+        +-------------------+        +-------------------+
|                   |        |                   |        |                   |
|    HTTP Response  | <----- |    Módulos        | <----- |    Repositorios   |
|                   |        |                   |        |                   |
+-------------------+        +-------------------+        +-------------------+
         |                           |                           |
         |                           |                           |
         v                           v                           v
+-------------------+        +-------------------+        +-------------------+
|                   |        |                   |        |                   |
|    Renderizado    |        |    Actualización  |        |    Base de Datos  |
|                   |        |    de UI          |        |                   |
+-------------------+        +-------------------+        +-------------------+
```
### Descripción de Componentes

- **Frontend (Angular)**:
  - **Componentes**: Partes de la interfaz de usuario.
  - **Servicios**: Lógica de negocio y comunicación con el backend.
  - **Módulos**: Agrupación de componentes y servicios.

- **Backend (Spring Boot)**:
  - **Controladores**: Manejan las peticiones HTTP.
  - **Servicios**: Contienen la lógica de negocio.
  - **Repositorios**: Interactúan con la base de datos.
  - **Seguridad**: Configuración de JWT y CORS.

- **Base de Datos**:
  - **Entidades**: Representación de las tablas de la base de datos.
  - **Repositorios**: Acceso a los datos.

- **Servicios Externos**:
  - **Firebase**: Notificaciones push.
  - **Elasticsearch**: Busqueda en servicios medicos
### Diagrama de BD

![[bisa_seguros_web_db_diagram.png]]