
ver el tipo de plan
vigencia poliza en la UI de polizas/web Agentes

considerar que el agente pueda hacer el pago en repre del cliente

*Siniestros / deducibles
Filtros en la parte de clientes

mas informacion en la parte de clientes, fecha nacimiento

### 2026-02-02

polizas activas
polizas por vencer

en el login me llega codigoAgente

las del mes actual
Prima total

la misma respuesta que en el de clientes para la polizas

---
tengo este documento:
  @requerimientos_a_bisa/APIs_Requeridas_Portal_Agentes.md

  ayer nos reunimos con el equipo de Bisa y quedamos lo siguiente:
  - quita los puntos 1.2 y 1.3, todo el punto 2
  - en el punto 3 Autenticacion y autorizacion, el login se hara de la forma que esta en este documento:

  @requerimientos_a_bisa/Autenticación token.docx

  por favor poner un comentarion que en el login deberia llegarnos el campo "codigoAgente"

  - para el punto 4 APIs del Modulo de Agentes, en la seccion 4.1.1 Resumen de cartera, pasando el codigoAgente deberia llegarnos: polizasActivas, Polizas por vencer (tomando 1 mes como
  referencia), el total de las primas
  - Tiene que haber un web service para que en base al cogidoAgente se obtengan el listado de Terceros (los clientes que tiene asignado el agente) asi como tambien la informacion de la
  poliza activa, la repuesta deberia ser un listado de elementos como estan descritos en este documento:
  @requerimientos_a_bisa/SERVICIO_INFORMACION_USUARIO_POLIZA.docx

  pero en una sola llamada que me llegue toda esa informacion

  - aumentar un webservice para obtener los deducibles, en realidad deberiamos usar este web service:
  @requerimientos_a_bisa/SERVICIO DEDUCIBLES COBERTURAS.docx
  pasando el codigo de la poliza

  - quitar los puntos 4.1.2, 4.1.3, 4.1.4, quitar tambien toda la seccion 4.2, 4.3, 5, 6, 7, 8, 9

  mantener el documento actual y crear uno nuevo en formato markdown con todos esos cambios
