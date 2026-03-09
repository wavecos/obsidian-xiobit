## Usuario 071931
### Producción

| Usuario          | Notas AppWeb      | Notas Android                   | Observaciones                                         |
| ---------------- | ----------------- | ------------------------------- | ----------------------------------------------------- |
| user: ==071931== | Sale un error 500 | La aplicación sale abruptamente | para este caso por favor enviar el json de respuesta. |
### Ambiente Pruebas

| Usuario      | Notas AppWeb                                | Notas Android                               | Observaciones                                          |
| ------------ | ------------------------------------------- | ------------------------------------------- | ------------------------------------------------------ |
| user: 071931 | Sale un mensaje: "No hay primas pendientes" | Sale un mensaje: "No hay primas pendientes" | el comportamiento en ambas aplicaciones es el correcto |
Para este usuario en el ambiente de Pruebas no se evidencio el error reportado que si esta ocurriendo en Producción, en este ambiente de Producción no se puede saber exactamente que respuesta esta llegando, para lo cual se pide que **==nos envien el JSON de respuesta del servicio 300==**
Desde la app se manda esto:
```
{
    "parametros": {
        "tercero": "071931",
        "pagosincluidos": "true"
    },
    "identificador": "wiserwebapp",
    "metodoCodigo": 300,
    "password": "wiserwebapp"
}
```
## Usuario 6730687
### Producción

El usuario no presenta problemas en este ambiente, el login y el ingreso a la pantalla principal se ven normales.
### Ambiente Pruebas
El usuario no presenta problemas en este ambiente, el login y el ingreso a la pantalla principal se ven normales.

## Usuario 2690520
### Producción

El usuario no presenta problemas en este ambiente, el login y el ingreso a la pantalla principal se ven normales.
### Ambiente Pruebas
El usuario no presenta problemas pero si esta en la acción de Cambio de contraseña, las pantallas se muestran bien en la webapp y en android.

## Usuario 2314254
### Producción

El usuario no presenta problemas en este ambiente, el login y el ingreso a la pantalla principal se ven normales.
### Ambiente Pruebas
El usuario no presenta problemas en este ambiente, el login y el ingreso a la pantalla principal se ven normales.