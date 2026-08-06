# Cálculo de Deducibles — Por Asegurado y a Nivel de Póliza

**Proyecto:** BISA Seguros - Salud Internacional
**Módulo:** Detalle de Póliza (Portal de Agentes)
**Componentes afectados:** Backend API (Spring Boot) + Webapp Agentes (Angular 21)
**Versión:** 1.0
**Última actualización:** 2026-04-28

---

## 1. Contexto

En la pantalla "Detalle de Póliza" del portal de agentes, se muestran deducibles en dos lugares:

1. **Header card de la póliza** — Dos donuts agregados a nivel de póliza ("Deducible Bolivia" y "Deducible Internacional").
2. **Tabla de Asegurados** — Por cada asegurado (titular + dependientes), dos columnas con monto y, al expandir la fila, dos donuts ("Bolivia D1" e "Internacional D2") con el detalle individual.

Los datos vienen de dos endpoints REST de BISA expuestos por el equipo de seguros:

- **Endpoint 1**: `GET /claim/deductible/policyDetail?policyNo=X&startDate=YYYY-MM-DD&endDate=YYYY-MM-DD` — agregado por nodo (asegurado).
- **Endpoint 2**: `GET /claim/deductible/deductibleDetail?id=N` — desglose ítem-por-ítem para un nodo específico.

Ambos requieren los parámetros indicados y se autentican con el service token BISA. La fecha mínima/máxima debe coincidir con la vigencia de la póliza (o con un período histórico si se quiere consultar uno anterior).

---

## 2. La data que llega de BISA

### 2.1 Endpoint 1 — Agregado por nodo

**Request:**
```
GET http://dev.graphapibisaseg.com:8081/claim/deductible/policyDetail
    ?policyNo=P2010000476
    &startDate=2024-04-01
    &endDate=2025-04-01
```

**Response de ejemplo:**
```json
{
  "value": {
    "decuctibleAgrNode": [
      {
        "nodeNo": "1-1-1",
        "limitDeductibleAmountNational":      2000.0,
        "limitDeductibleAmountInternational": 2000.0,
        "usedDeductibleAmount": 646.56,
        "partyName": "MARIA FERNANDA  ALVAREZ ALVIS",
        "partyRole": "Titular",
        "id": 9
      }
    ],
    "totalDeductibleUsed": 646.56,
    "policyNo": "P2010000476"
  }
}
```

Campos relevantes por nodo:

| Campo | Significado |
|-------|-------------|
| `nodeNo` | Identificador jerárquico BISA del asegurado (e.g., `1-1-1` titular, `1-1-2` cónyuge, `1-1-3` hijo). Hace match con `Asegurado.nodeDep` del lado interno. |
| `limitDeductibleAmountNational` | Tope (cap) Bolivia para ese asegurado, USD. |
| `limitDeductibleAmountInternational` | Tope Internacional para ese asegurado, USD. |
| `usedDeductibleAmount` | Total **combinado** consumido (Bolivia + Internacional sumados). |
| `id` | Identificador interno BISA del registro agregado. Se usa como parámetro del endpoint 2 para obtener el desglose. |

**Observación clave:** el endpoint 1 no separa el consumido por país, solo da un total combinado. Para mostrar Bolivia vs Internacional por separado necesitamos el endpoint 2.

### 2.2 Endpoint 2 — Desglose ítem-por-ítem

**Request:**
```
GET http://dev.graphapibisaseg.com:8081/claim/deductible/deductibleDetail?id=9
```

**Response de ejemplo:**
```json
{
  "value": [
    {
      "deductibleAmountUsd": 323.28,
      "isInternational": 0,
      "country": "BOLIVIA",
      "id": 264,
      "deductibleAggregateId": 9
    },
    {
      "deductibleAmountUsd": 323.28,
      "isInternational": 1,
      "country": "INTERNACIONAL",
      "id": 265,
      "deductibleAggregateId": 9
    }
  ],
  "length": 2
}
```

Campos relevantes por ítem:

| Campo | Significado |
|-------|-------------|
| `deductibleAmountUsd` | Monto del ítem aplicado al deducible, USD. |
| `isInternational` | `0` = Bolivia, `1` = Internacional. **Se usa este flag numérico** en lugar del string `country` por confiabilidad. |
| `deductibleAggregateId` | FK al `id` del nodo agregado del endpoint 1. |

**Verificación de consistencia:** sumando los ítems del endpoint 2 obtenemos el total del endpoint 1. En el ejemplo: `323.28 + 323.28 = 646.56` ✓.

---

## 3. Flujo en el backend

Toda la orquestación vive en `bisa-seg-salud-intl-backend-api` y se expone vía el endpoint existente:

```
GET /api/v1/policies/{policyNo}/detail
```

(opcionalmente acepta `?deductibleStart=...&deductibleEnd=...` para sobrescribir las fechas — útil para testing con vigencias anteriores).

### 3.1 Componentes involucrados

| Archivo | Responsabilidad |
|---------|-----------------|
| `controller/PolicyController.java` | Recibe la request HTTP, propaga overrides opcionales. |
| `integration/bisa/BisaAffiliatePolicyService.java` | Llama BISA endpoints 1 y 2 con service token. |
| `integration/bisa/AbstractBisaApiClient.java` | Cliente WebClient base, valida response (`error: true` lanza). |
| `integration/bisa/dto/Deductible*.java` | DTOs para deserializar las respuestas BISA. |
| `service/PolicyDetailService.java` | Orquesta: fetch + agregación + mapeo a `PolicyDetail`. |

### 3.2 Paso 1 — `fetchPerNodeDeductibles()` construye el mapa por nodo

```java
private Map<String, NodeDeducibles> fetchPerNodeDeductibles(
        String policyNo, PolicyDeductibles deductibles,
        String startOverride, String endOverride) {

    String startIso = isIsoDate(startOverride) ? startOverride : toIsoDate(deductibles.getVigenciaInicial());
    String endIso   = isIsoDate(endOverride)   ? endOverride   : toIsoDate(deductibles.getVigenciaFinal());

    // 1. Llama endpoint 1 con el rango resuelto
    DeductibleAggregateResponse aggregateResponse =
        bisaAffiliatePolicyService.getDeductibleAggregatesWithServiceToken(policyNo, startIso, endIso);

    Map<String, NodeDeducibles> result = new HashMap<>();
    for (DeductibleAggregateNode node : aggregateResponse.getValue().getDecuctibleAgrNode()) {

        Double limitBol = node.getLimitDeductibleAmountNational();
        Double limitInt = node.getLimitDeductibleAmountInternational();

        double consumedBol = 0.0;
        double consumedInt = 0.0;

        // 2. Llama endpoint 2 con node.id para obtener desglose
        DeductibleDetailResponse detail =
            bisaAffiliatePolicyService.getDeductibleDetailWithServiceToken(node.getId());

        // 3. Suma cada ítem en el cubo correcto según isInternational
        for (DeductibleDetailItem item : detail.getValue()) {
            double amount = item.getDeductibleAmountUsd();
            if (item.getIsInternational() != null && item.getIsInternational() == 1) {
                consumedInt += amount;
            } else {
                consumedBol += amount;   // default cuando es 0 o null
            }
        }

        // 4. Construye DeducibleConsumo para Bolivia y para Internacional
        result.put(node.getNodeNo().trim(), new NodeDeducibles(
            buildDeducibleConsumo(limitBol, consumedBol),    // monto, consumido, restante = max(monto-consumido, 0)
            buildDeducibleConsumo(limitInt, consumedInt)
        ));
    }
    return result;
}
```

Para la póliza del ejemplo, el mapa resultante es:

```
Map<String, NodeDeducibles> = {
  "1-1-1": (
    bolivia       = { monto: 2000, consumido: 323.28, restante: 1676.72, porcentaje: null },
    internacional = { monto: 2000, consumido: 323.28, restante: 1676.72, porcentaje: null }
  )
}
```

### 3.3 Paso 2 — `buildAsegurados()` distribuye por persona

Por cada asegurado de la póliza, busca su entrada en el mapa por `nodeDep`:

```java
NodeDeducibles perNode = dep.getNodeDep() != null
        ? deductiblesByNode.get(dep.getNodeDep().trim())
        : null;

DeducibleConsumo bolForNode = perNode != null && perNode.bolivia() != null
        ? perNode.bolivia()
        : bol;     // fallback: cap estático de la póliza (consumido=0)

DeducibleConsumo intlForNode = perNode != null && perNode.internacional() != null
        ? perNode.internacional()
        : intl;
```

Si la lista de dependientes viene vacía (e.g., el agente no tiene credenciales BISA para `/intermediary/policies`), se sintetiza el titular desde `nombreTitular + apellidoTitular` y los dependientes desde el campo `dependientesPoliza`. En ese path se usa la **convención posicional BISA**:

- Titular → `1-1-1`
- 1er dependiente → `1-1-2`
- 2do dependiente → `1-1-3`, etc.

Cada uno hace lookup en `deductiblesByNode` con esa key. Esto garantiza que el donut por-asegurado se llene aún cuando el `nodeDep` real no esté disponible.

### 3.4 Paso 3 — `buildHeader()` agrega a nivel de póliza

```java
private AggregatedPolicyDeducible aggregatePolicyDeducible(
        Map<String, NodeDeducibles> deductiblesByNode, boolean bolivia) {

    double cap = 0.0;
    double consumido = 0.0;
    boolean hasAny = false;

    for (NodeDeducibles pair : deductiblesByNode.values()) {
        DeducibleConsumo side = bolivia ? pair.bolivia() : pair.internacional();
        if (side == null) continue;
        hasAny = true;
        if (side.getMonto() != null)     cap       += side.getMonto();
        if (side.getConsumido() != null) consumido += side.getConsumido();
    }

    return hasAny
        ? new AggregatedPolicyDeducible(cap, consumido)
        : new AggregatedPolicyDeducible(null, null);   // fallback al cap estático
}
```

Es decir: el donut del **header** muestra la suma agregada del cap y del consumido entre TODOS los asegurados. Cuando hay un solo asegurado, el header coincide con la fila individual; cuando hay varios, el header refleja el "cuánto del deducible total se ha gastado en toda la póliza".

---

## 4. Cálculo paso a paso con el ejemplo (un asegurado)

Datos de entrada (`P2010000476`, ventana `2024-04-01 → 2025-04-01`):

| Origen | Campo | Valor |
|--------|-------|-------|
| Endpoint 1, nodo `1-1-1` | `limitDeductibleAmountNational` | 2000.00 |
| Endpoint 1, nodo `1-1-1` | `limitDeductibleAmountInternational` | 2000.00 |
| Endpoint 1, nodo `1-1-1` | `usedDeductibleAmount` | 646.56 |
| Endpoint 1, nodo `1-1-1` | `id` | 9 |
| Endpoint 2 (id=9), ítem 1 | `deductibleAmountUsd` / `isInternational` | 323.28 / 0 |
| Endpoint 2 (id=9), ítem 2 | `deductibleAmountUsd` / `isInternational` | 323.28 / 1 |

### Cálculo per-asegurado (Titular)

```
consumedBol = 323.28        (suma de ítems con isInternational == 0)
consumedInt = 323.28        (suma de ítems con isInternational == 1)

Asegurado.deducibleBolivia = {
  monto:      2000,
  consumido:  323.28,
  restante:   max(2000 - 323.28, 0)  = 1676.72,
  porcentaje: null
}

Asegurado.deducibleInternacional = {
  monto:      2000,
  consumido:  323.28,
  restante:   1676.72,
  porcentaje: null
}
```

→ Donut Bolivia (D1): 16% consumido, `USD 323.28 / USD 2,000.00`, restante `USD 1,676.72`.
→ Donut Internacional (D2): idéntico.

### Cálculo agregado a nivel de póliza (header)

```
header.deduciblePolizaBol            = 2000        (suma de monto Bolivia entre nodos = 1 nodo)
header.deduciblePolizaBolConsumido   = 323.28      (suma de consumido Bolivia)
header.deduciblePolizaInt            = 2000
header.deduciblePolizaIntConsumido   = 323.28
```

→ Donut Header Bolivia: 16%. → Donut Header Internacional: 16%. Coinciden con el per-asegurado **porque hay un solo asegurado**.

---

## 5. Cálculo con múltiples asegurados (escenario hipotético)

Misma póliza con Titular + Cónyuge + Hijo. BISA devolvería:

| nodeNo | partyRole | limitNat | limitInt | usedTotal | id |
|--------|-----------|----------|----------|-----------|-----|
| 1-1-1  | Titular   | 2000     | 2000     | 800       | 9  |
| 1-1-2  | Cónyuge   | 2000     | 2000     | 200       | 10 |
| 1-1-3  | Hijo(a)   | 2000     | 2000     | 0         | 11 |

Tres llamadas al endpoint 2 (una por id) que hipotéticamente devuelven, descontando el split:

| Nodo | consumedBol | consumedInt |
|------|-------------|-------------|
| 1-1-1 | 600 | 200 |
| 1-1-2 | 100 | 100 |
| 1-1-3 | 0   | 0   |

### Tabla de asegurados (donut por persona)

| Asegurado | Bolivia (consumido / cap) | Internacional |
|-----------|---------------------------|---------------|
| Titular   | 600 / 2000 = **30%**      | 200 / 2000 = **10%** |
| Cónyuge   | 100 / 2000 = **5%**       | 100 / 2000 = **5%**  |
| Hijo(a)   | 0 / 2000 = **0%**         | 0 / 2000 = **0%**    |

### Header (donut agregado)

| Donut | Cálculo | Resultado |
|-------|---------|-----------|
| Bolivia | (600 + 100 + 0) / (2000 + 2000 + 2000) | **700 / 6000 = 11.7%** |
| Internacional | (200 + 100 + 0) / (2000 + 2000 + 2000) | **300 / 6000 = 5%** |

---

## 6. Shape final de la response al frontend

`GET /api/v1/policies/{policyNo}/detail` devuelve el envelope estándar `{ result, error, errorMessage }`. Dentro de `result`:

```jsonc
{
  "header": {
    "policyNo": "P2010000476",
    "deduciblePolizaBol":           2000,        // suma de caps Bolivia (o cap estático en fallback)
    "deduciblePolizaBolConsumido":  323.28,      // suma de consumido Bolivia (null en fallback)
    "deduciblePolizaInt":           2000,
    "deduciblePolizaIntConsumido":  323.28,
    "vigenciaDesde":  "2025-04-01",
    "vigenciaHasta":  "2026-04-01",
    // ... resto del header (forma de pago, balance, etc.) ...
  },
  "asegurados": [
    {
      "nombre":  "Alvarez Alvis Maria Fernanda",
      "nodeDep": "1-1-1",
      "relacion": "TITULAR",
      "deducibleBolivia":      { "monto": 2000, "consumido": 323.28, "restante": 1676.72, "porcentaje": null },
      "deducibleInternacional":{ "monto": 2000, "consumido": 323.28, "restante": 1676.72, "porcentaje": null }
      // ... resto del asegurado ...
    }
  ]
  // ... titular, tomador, cuotas, facturas, centrosMedicos ...
}
```

El frontend (Angular) consume estas estructuras directamente en:

- `policy-header-card.component.ts` — donuts del header.
- `asegurados-table.component.ts` — columnas + donuts de la fila expandida.
- `deductible-donut.component.ts` — componente SVG genérico que dibuja el donut a partir de `consumido` y `total`.

---

## 7. Fallbacks (qué pasa cuando algo falla)

| Escenario | Qué hace el sistema |
|-----------|---------------------|
| Vigencia inicial/final no parseable o nula | Skip endpoint 1, mapa vacío, fallback al cap estático del header comercial con `consumido=0`. Log WARN. |
| Endpoint 1 falla (HTTP error, BISA caído, `error:true`) | Mapa vacío, mismo fallback. Log WARN con el mensaje del error. |
| Endpoint 1 devuelve `decuctibleAgrNode: []` | Mapa vacío, mismo fallback. Log INFO. |
| Endpoint 2 falla para un nodo concreto | Para ese nodo: se preserva el limit real, pero `consumedBol = node.usedDeductibleAmount` (combinado total) y `consumedInt = 0` — mejor mostrar algo no-cero que perder la información. Log WARN. |
| El `nodeDep` de un asegurado no matchea ninguna key del mapa | Ese asegurado usa el cap estático de póliza con `consumido=0` (fallback per-asegurado). |
| Lista de dependientes vacía (no hay credenciales agente) | Convención posicional BISA: titular = `1-1-1`, dependientes secuenciales `1-1-2`, `1-1-3`, ... |
| Header sin datos por nodo | `deduciblePolizaBolConsumido` y `IntConsumido` quedan `null`; el donut del header sigue mostrando el cap estático con 0% consumido (sin regresión vs comportamiento previo). |

---

## 8. Decisiones de diseño

1. **Usar `isInternational` (numérico) sobre `country` (string)** — más confiable que comparar literales `"BOLIVIA"`/`"INTERNACIONAL"` que pueden cambiar de localización.
2. **Defensivo con valores desconocidos** — si `isInternational` no es `1`, lo tratamos como Bolivia (default). No se pierde monto. Si BISA agregara un valor `2` para algún caso especial, terminaría sumándose a Bolivia; preferible a desaparecer.
3. **Atribución a Bolivia cuando endpoint 2 falla** — preserva el dato del usuario (sabe que consumió algo) aunque pierde el split exacto. Mejor que mostrar `0%`.
4. **Header agregado, no per-asegurado-replicado** — los donuts del header muestran la deuda total de la póliza (suma), no el cap individual de un titular. Le da al agente una vista de "cuánto del deducible total ya se gastó en esta póliza".
5. **`porcentaje = null` siempre** — BISA no expone porcentaje en estos endpoints. El donut lo deriva visualmente como `consumido/monto`. La tabla muestra `· X%` solo si `porcentaje` viene seteado, lo cual queda preparado para si BISA lo agrega más adelante.
6. **`@JsonIgnoreProperties(ignoreUnknown = true)` en todos los DTOs nuevos** — BISA puede agregar campos sin romper la deserialización (e.g., `displayName`, `subproductName`, `partyCode`, etc., que sí vienen en el response pero no usamos hoy).
7. **Mantener el typo `decuctibleAgrNode`** del payload BISA — el campo real viene mal escrito; mapear con `@JsonProperty("decuctibleAgrNode")` para no romper deserialización.
8. **`nodeNo.trim()` y `nodeDep.trim()`** al insertar/leer el mapa — defensivo contra whitespace inconsistente que BISA pudiera meter.

---

## 9. Override de fechas para testing

El endpoint `/api/v1/policies/{policyNo}/detail` acepta dos query params opcionales:

- `deductibleStart` — ISO `yyyy-MM-dd`
- `deductibleEnd` — ISO `yyyy-MM-dd`

Cuando se proporcionan y son válidos, **reemplazan** la vigencia de la póliza al consultar el endpoint 1 de BISA. Esto permite consultar vigencias anteriores con datos históricos.

En el frontend hay un flag dev:

```ts
private readonly DEV_DEDUCTIBLE_WINDOW: { start: string; end: string } | null = null;
```

Al setearlo (e.g., `{ start: '2024-04-01', end: '2025-04-01' }`), el frontend anexa los query params al request del backend. Útil para testing sin tocar la base de datos ni el código de producción.

Existe también `DEV_POLICY_OVERRIDE` que redirige toda navegación de detalle a una póliza específica (e.g., `'P2010000476'`).

Ambos flags se dejan en `null` para producción.

---

## 10. Observabilidad

`PolicyDetailService.fetchPerNodeDeductibles()` emite los siguientes logs:

- `INFO  Fetching per-asegurado deductibles for policy {} window {} → {}` — confirma la ventana usada.
- `INFO  Fetching per-asegurado deductibles for policy {} window {} → {} (override)` — cuando se usaron los query params.
- `WARN  Skipping per-node deductibles for policy {}: missing vigencia dates ...` — cuando no hay fecha parseable.
- `WARN  Could not fetch deductible aggregates for policy {}: {}` — cuando endpoint 1 falla.
- `INFO  BISA returned no deductible aggregates for policy {} (window ...)` — cuando devuelve array vacío.
- `INFO  BISA returned {} deductible aggregate node(s) for policy {}` — confirma cuántos nodos se procesaron.
- `WARN  Could not fetch deductible detail for node {} (id={}): {}` — cuando endpoint 2 falla para un nodo concreto.

Estos logs son suficientes para diagnosticar la mayoría de los problemas de integración sin necesidad de inspeccionar tráfico HTTP.

---

## 11. Archivos clave

### Backend

- `bisa-seg-salud-intl-backend-api/src/main/java/com/bisaseguros/salud/internacional/backend_api/controller/PolicyController.java`
- `bisa-seg-salud-intl-backend-api/src/main/java/com/bisaseguros/salud/internacional/backend_api/service/PolicyDetailService.java`
- `bisa-seg-salud-intl-backend-api/src/main/java/com/bisaseguros/salud/internacional/backend_api/integration/bisa/BisaAffiliatePolicyService.java`
- `bisa-seg-salud-intl-backend-api/src/main/java/com/bisaseguros/salud/internacional/backend_api/integration/bisa/AbstractBisaApiClient.java`
- `bisa-seg-salud-intl-backend-api/src/main/java/com/bisaseguros/salud/internacional/backend_api/integration/bisa/dto/DeductibleAggregateNode.java`
- `.../integration/bisa/dto/DeductibleAggregateValue.java`
- `.../integration/bisa/dto/DeductibleAggregateResponse.java`
- `.../integration/bisa/dto/DeductibleDetailItem.java`
- `.../integration/bisa/dto/DeductibleDetailResponse.java`
- `.../integration/bisa/dto/DeducibleConsumo.java`
- `.../integration/bisa/dto/PolicyHeader.java`

### Frontend

- `bisa-seg-salud-intl-agentes-web/src/app/features/policy-detail/services/policy.service.ts`
- `bisa-seg-salud-intl-agentes-web/src/app/features/policy-detail/components/policy-header-card.component.ts`
- `bisa-seg-salud-intl-agentes-web/src/app/features/policy-detail/components/asegurados-table.component.ts`
- `bisa-seg-salud-intl-agentes-web/src/app/features/policy-detail/components/deductible-donut.component.ts`
- `bisa-seg-salud-intl-agentes-web/src/app/features/policy-detail/models/policy.model.ts`
