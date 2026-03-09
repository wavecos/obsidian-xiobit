### Descripción General del Proyecto
La aplicación **BISA Seguros** es una aplicación móvil para gestionar seguros, realizar cotizaciones y acceder a servicios relacionados.
### Arquitectura General
La aplicación sigue la arquitectura **MVVM (Model-View-ViewModel)**.
### Componentes Principales
- **App / AppDelegate**: Punto de entrada de la UI.
- **Swift UI View**s: Componentes de UI reutilizables.
- **ViewModel**: Maneja la lógica de negocio y la comunicación con el Repository.
- **Repository**: Fuente única de verdad para los datos.
### Dependencias y Librerías
- **Factory**: Inyección de dependencias.
- **Alamofire**: Llamadas a la API.
- **Realm**: Persistencia de datos.
- **Swift UI**: Construcción de la UI.
- **Firebase**: Mensajería y servicios en la nube.
- **Kingfisher**: Carga de imágenes.
- **Lottie**: Animaciones.
### Flujo de Datos
El flujo de datos sigue el patrón unidireccional:
1. La UI (Views) interactúa con el ViewModel.
2. El ViewModel solicita datos al Repository.
3. El Repository obtiene datos de la red o la base de datos.
4. Los datos fluyen de vuelta al ViewModel y luego a la UI.
### Gestión de Dependencias
Se utiliza **Factory** para la inyección de dependencias, facilitando la gestión y el ciclo de vida de las dependencias.
### Persistencia de Datos
Se utiliza **Realm** para la persistencia de datos locales.
### Red y API
Se utiliza **Alamofire** para realizar llamadas a la API y manejar las respuestas de manera eficiente.
### UI y Navegación
La UI se construye utilizando **Swift UI** y la navegación se maneja con **Navigation Destination**.
