Para produccion se debe modificar el SP de NaturalRepository y JuridicoRepositorytural, por el siguiente:

producción: 
```sql
EXEC dbo.CUMPLIMIENTO_TERCEROS_PCC04
```
   
entorno local :
```sql
EXEC dbo.CUMPLIMIENTO_TERCERO_JURIDICO 
EXEC dbo.CUMPLIMIENTO_TERCERO_NATURAL
```
### domain/Pago.java
se debe aumentar : schema = "cumplimiento" como se ve en la siguiente linea

```java
@Table(name = "TRANSACCIONES_CTA" , schema = "cumplimiento")
```


