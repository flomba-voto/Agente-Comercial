---
title: "Base de conocimiento de metadatos del consolidado de ventas y entregas"
version: "4.0"
updated_at: "2026-10-13"
dataset: "current.parquet"
dataset_folder: "Parquets/Raw_2024-2026"
material_lookup: "Tabla de materiales.xlsx"
column_count: 60
row_count_verified: 336601
classification: "uso interno confidencial"
---

# Base de conocimiento de metadatos

## Alcance y estado de la migración

Este documento sustituye a `metadata_v3.md` para el esquema nuevo de `current.parquet`.
El catálogo operativo contiene **exactamente 60 columnas reales**, con los nombres del archivo nuevo.
Los nombres antiguos aparecen exclusivamente en el apéndice de auditoría, no son alias operativos.
Las reglas de negocio proceden de la versión anterior; las equivalencias, tipos y observaciones
se contrastaron con ambos archivos completos. No se han añadido autorizaciones por inferencia.

| Fuente | Carpeta SharePoint | Columnas | Filas verificadas |
|---|---|---:|---:|
| Esquema antiguo | `Parquets/V2` | 125 | 380.480 |
| Esquema nuevo, vigente para este documento | `Parquets/Raw_2024-2026` | 60 | 336.601 |

- Correspondencias confiables: **53**, incluidas las columnas técnicas conservadas con el mismo nombre y tipo.
- Columnas técnicas nuevas sin autorización: **6**.
- Campo nuevo con correspondencia funcional no resuelta: **`date_on_which_record`**.
- Columnas antiguas sin correspondencia vigente identificada: **70**; quedan excluidas del catálogo.
  Además, dos campos de fecha antiguos son candidatos indistinguibles para el campo de fecha nuevo.
- La diferencia de filas entre los archivos no demuestra por sí sola borrados, pérdida de datos ni cambios comerciales:
  son instantáneas con alcance distinto. No se supuso igualdad de orden ni correspondencia de todas las filas.
- `Tabla de materiales.xlsx` no fue aportada ni inspeccionada: su estructura y contenidos descritos por la versión
  anterior se conservan como contrato pendiente de validación, no como hechos comprobados en esta revisión.

## Política de uso

El flag `agent_usable` es obligatorio. El agente solo puede utilizar una columna cuando
su valor sea `true`. Si es `false`, puede mencionar su existencia y su restricción,
pero no utilizarla para calcular, filtrar, segmentar, resolver identidades, explicar ni concluir.
Una definición descriptiva o una equivalencia técnica **no** sustituyen una autorización.

Se preservan los flags de las columnas con equivalencia confiable. Ningún `false` se eleva a `true`.
Los campos nuevos sin antecedente autorizado y el campo de fecha ambiguo se mantienen en `false`.
Los derivados autorizados tampoco pueden eludir las restricciones de sus fuentes ni las condiciones de unidad o lookup.

## Resolución inteligente de clientes

### Estado operativo y precedencia de permisos

- Identificador canónico del cliente/pagador: **`customers_account_n`**, autorizado.
- Campo de nombre asociado: **`customer`**, existente pero **no autorizado** (`agent_usable: false`).
- **La resolución por nombre está deshabilitada en esta versión.** No leer valores únicos de `customer`,
  no aplicar búsquedas exactas, alias ni coincidencia difusa sobre ese campo y no mostrar sus nombres.
- Si el usuario aporta un identificador canónico inequívoco, resolver mediante coincidencia exacta con
  `customers_account_n`, conservando su representación textual y sus ceros iniciales.
  Si aporta solo un nombre, solicitar el identificador o la autorización explícita del propietario
  y la actualización del flag de `customer` antes de continuar. No devolver cifras para una identidad no resuelta.
- `sold_to_party` y `sold_to_party_descript` conservan la definición heredada de obra/solicitante;
  no sustituirlos por el identificador o el nombre del cliente/pagador.

### Protocolo conservado para una futura autorización del nombre

El siguiente protocolo se conserva como regla de negocio, pero **no puede ejecutarse**
mientras `customer` continúe en `false`. Cuando exista autorización y se actualicen primero los metadatos:

1. No filtrar directamente `current.parquet` por el texto introducido. Resolverlo primero contra un catálogo
   autorizado de pares `customers_account_n` / `customer` y obtener su identificador canónico.
2. Usar para desambiguar únicamente atributos autorizados y con vínculo confirmado al cliente.
   `country_key`, `city` y `postal_code` describen la obra, no automáticamente el domicilio del pagador.
3. Una vez confirmado el cliente, aplicar filtros y cálculos mediante `customers_account_n`.

### Normalización de nombres (protocolo condicionado)

Normalizar texto y nombres antes de compararlos: minúsculas; eliminar tildes y diacríticos;
sustituir puntuación y caracteres especiales por espacios; eliminar espacios duplicados;
normalizar símbolos como `&` y `+`; eliminar sufijos societarios no distintivos
(`S.L.`, `SL`, `S.A.`, `SA`, `Ltda.`, `Inc.`, `Corp.` y equivalentes);
conservar números, localidades y términos distintivos. No inventar un registro de alias.

### Orden de resolución (protocolo condicionado)

1. Coincidencia exacta con el nombre original.
2. Coincidencia exacta con el nombre normalizado.
3. Coincidencia con un alias registrado y autorizado, si existe.
4. Coincidencia difusa mediante `RapidFuzz`, combinando `fuzz.WRatio` y `fuzz.token_set_ratio`.
5. Ajuste por contexto mediante atributos autorizados con vínculo confirmado al cliente.

### Nivel de confianza (protocolo condicionado)

Comparar siempre el primer candidato con el segundo mejor resultado:

- **Alta:** puntuación igual o superior a 90 y ventaja mínima de 8 puntos frente al segundo candidato.
  Selección automática solo si no hay ambigüedad entre nombres similares.
- **Media:** puntuación entre 75 y 89, o diferencia inferior a 8 puntos entre los mejores candidatos.
  Solicitar confirmación mostrando como máximo tres nombres canónicos, únicamente cuando su uso esté autorizado.
- **Baja:** puntuación inferior a 75. Indicar que no se encontró una coincidencia fiable y solicitar más información.

### Reglas de seguridad

- No combinar registros de varios clientes como si fueran uno solo.
- No interpretar ausencia de coincidencia como ausencia de datos comerciales.
- No mostrar cifras hasta haber resuelto la identidad de forma fiable.
- Conservar el identificador confirmado durante la conversación hasta que cambie el cliente.
- Usar el nombre canónico en la respuesta solo cuando `customer` esté autorizado.
- Cuando una resolución no sea exacta, indicar brevemente el cliente interpretado sin divulgar campos restringidos.

## Representación de identificadores, fechas y unidades

### Identificadores

`sales_document`, `sold_to_party`, `customers_account_n` y `material_number` son ahora texto UTF8.
En esta instantánea, los tres primeros tienen longitud 10
y `material_number` longitud 18.
Los identificadores de obra, cliente y material contienen ceros iniciales. No sumarlos,
no convertirlos a flotante y no eliminar esos ceros en filtros o claves canónicas.
La normalización numérica usada para el cotejo histórico no define su representación operativa.

Para enlazar con un lookup cuyo código sea numérico o textual sin padding, puede generarse una clave
auxiliar de material compatible con el texto canónico, conservando `material_number` intacto.
Antes, verificar que el código es íntegro, no fraccionario, no nulo y no alfanumérico;
no convertir códigos arbitrarios ni completar datos ausentes.

### Fecha y hora

`date_on_which_record` es `TIMESTAMP[µs]` almacenado como `INT64`, con
`isAdjustedToUTC=false`: no declara una zona horaria ni debe interpretarse como UTC.
**No es un serial Excel ni una fracción de día.** Un lector Parquet debe respetar el tipo lógico.
En esta instantánea hay 8 nulos; todos los valores no nulos están a medianoche,
con rango 2024-01-01 a 2026-10-08.
La representación no determina qué evento comercial fecha: su significado y equivalencia están pendientes.
No utilizarla para periodificar ventas, salidas, pedidos ni tiempos de entrega mientras permanezca en `false`.
No existen columnas independientes de hora operativas en el esquema nuevo.

### Unidades y controles previos a cálculos

- `net_price`, `net_weight_1`, `net_value_of_the_order` y `distance` son `DOUBLE`.
  La igualdad de valores entre archivos no prueba por sí sola una unidad física.
- La regla heredada define `net_price` por tonelada y `net_weight_1` como peso entregado en toneladas.
  Se conserva ese contrato de negocio, **pero no se certifica homogeneidad de unidades en todas las filas**:
  el cotejo encontró una unidad histórica `KG` y el esquema nuevo no trae un campo explícito de unidad de peso.
- `sales_unit` contiene varios códigos de unidad comercial y permanece en `false`.
  No usarlo como unidad de peso, no convertirlo por inferencia y no elevar su flag para sortear esta limitación.
- Antes de etiquetar sumas de `net_weight_1` como toneladas o calcular `valor_neto_entrega`,
  exigir un contrato de normalización validado por el propietario para el subconjunto calculado.
  Si falta esa garantía, informar la limitación y no presentar una magnitud en toneladas ni un importe derivado fiable.
  No efectuar conversiones automáticas, no sustituir nulos por cero y conservar el signo de los valores.
- `currency` declara `EUR` en esta instantánea; no suponer que futuros archivos tengan una sola moneda.
- `um_for_distance` contiene `KM` o cadena vacía. Los registros con unidad vacía tienen `distance` igual a cero.
  Una unidad vacía no confirma una distancia real nula: no completar unidad ni interpretar automáticamente cero como ausencia de trayecto.

## Reglas de negocio confirmadas


### Región comercial por `sales_organization`

| Código | Región |
|---|---|
| EO11 | Norte |
| EO21 | Norte |
| EO55 | Sur |
| EO56 | Madeira |
| EO57 | Canarias |
| EO79 | Canarias |
| EO65 | Centro |
| EO66 | Centro |
| EO53 | Sur |
| EO54 | Export |

### Mercado por `sales_organization`

| Código | Mercado |
|---|---|
| EO11 | Doméstico |
| EO21 | Export |
| EO55 | Doméstico |
| EO56 | Export |
| EO57 | Doméstico |
| EO79 | Doméstico |
| EO65 | Doméstico |
| EO66 | Export |
| EO53 | Doméstico |
| EO54 | Export |

### Formato por `distribution_channel`

| Código | Formato |
|---|---|
| E3 | Granel |
| E4 | Ensacado |

### Gestión de entrega por `price_group`

| Código | Resultado |
|---|---|
| CE | Entrega gestionada por Votorantim |
| CP | El cliente recoge o gestiona el pedido |

### Valor neto de entrega

```text
valor_neto_entrega = net_price * net_weight_1
```

La fórmula conserva la regla autorizada: precio neto por tonelada multiplicado por peso neto entregado
en toneladas. Aplicar el control de unidades anterior antes de calcular. El resultado es un campo
**derivado**; no sustituirlo automáticamente por `net_value_of_the_order` ni sumar importes de distintas monedas.

### Fábrica derivada

Usar `region_comercial` (derivada de `sales_organization`), `plant`, `material_number`
y `formato_producto` (derivada de `distribution_channel`). Aplicar en este orden:

1. Región comercial Canarias → Tenerife.
2. `plant` = COUR y formato/embalaje normalizado = Bulk → Oural.
3. `material_number` = `000000000000200212` → Oural.
4. `plant` = CSCB → Córdoba.
5. `plant` = CCAL o CCAC → Alconera.
6. `plant` = CSBB → Bobadilla.
7. `plant` = CSNB → Niebla.
8. `plant` = CSHU → Niebla-Madeira.
9. `plant` = CSMA, CSGR, CSPM, CMAL, CMGR o CMPM → Málaga.
10. Cualquier otro caso válido → Toral.

> Para COUR, crear explícitamente un atributo auxiliar de formato/embalaje a partir del
> `formato_producto` autorizado: Granel → Bulk; Ensacado → no Bulk.
> No hay una columna física Bulk ni una columna de hoja de cálculo que pueda asumirse como fuente.
> Si el canal, la región o la clave de material no son válidos o están ausentes, no aplicar el fallback
> Toral para ocultar el dato faltante; informar que no se puede derivar de forma fiable.

### Calidad de material

Cruzar `material_number` con `Material` de `Tabla de materiales.xlsx` y devolver `Calidad`,
**solo después de obtener y validar la tabla auxiliar** y la equivalencia de claves.
El código del material en el dataset nuevo es textual y conserva padding; normalizar una clave auxiliar
compatible en ambos lados, sin modificar los identificadores originales ni asumir tipos del Excel no inspeccionado.
Validar unicidad y cardinalidad del cruce para evitar multiplicar registros.

La regla heredada clasifica el producto como clinker cuando no existe calidad o no es numérica.

> Advertencia conservada de la versión anterior, no comprobación del archivo auxiliar actual:
> las etiquetas no numéricas `Clinker`, `Cto.Blanco` y `Porte` quedarían todas clasificadas como clinker.
> Por ello la regla también clasificaría cemento blanco y portes como clinker.
> Mostrar esta advertencia cuando corresponda y no inventar excepciones antes de que se definan.
> Sin acceso al lookup no atribuir calidad, no inventar coincidencias y no presentar como verificadas
> sus estadísticas históricas de materiales o calidades.

## Campos derivados autorizados

Estos nombres son derivados de negocio, no columnas físicas de `current.parquet`.
La autorización no elimina los controles de fuentes, claves, monedas, nulos y unidades.

| Campo | agent_usable | Fuentes actuales y condiciones |
|---|---|---|
| `region_comercial` | true | `sales_organization`; mapa documentado |
| `mercado` | true | `sales_organization`; mapa documentado |
| `formato_producto` | true | `distribution_channel`; mapa documentado |
| `gestion_entrega` | true | `price_group`; mapa documentado |
| `valor_neto_entrega` | true, condicionado | `net_price`, `net_weight_1`; peso confirmado en toneladas y moneda homogénea |
| `fabrica_derivada` | true, condicionado | `region_comercial`, `plant`, `material_number`, `formato_producto`; normalización Bulk explícita |
| `calidad_material` | true, condicionado | `material_number` y lookup `Material` → `Calidad`; Excel obtenido y validado |
| `tipo_producto_calidad` | true con advertencia, condicionado | `calidad_material`; regla numérica/no numérica y advertencia de etiquetas |

## Metadatos por columna

Los tipos y la nulabilidad describen el esquema físico/lógico leído, no una autorización adicional.
Todas las columnas son opcionales en el esquema; un campo sin nulos en esta instantánea sigue siendo nullable.

### 001. `id`

- **agent_usable:** `false`
- **Definición:** Identificador técnico de fila.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 002. `business_key`

- **agent_usable:** `false`
- **Definición:** Hash técnico de la clave de negocio.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 003. `content_hash`

- **agent_usable:** `false`
- **Definición:** Hash técnico del contenido.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 004. `record_status`

- **agent_usable:** `false`
- **Definición:** Estado técnico del registro.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 005. `version_number`

- **agent_usable:** `false`
- **Definición:** Versión técnica.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 006. `first_seen_run_id`

- **agent_usable:** `false`
- **Definición:** Primera carga en que se observó.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 007. `last_seen_run_id`

- **agent_usable:** `false`
- **Definición:** Última carga en que se observó.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 008. `valid_from_run_id`

- **agent_usable:** `false`
- **Definición:** Carga inicial de vigencia.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 009. `valid_to_run_id`

- **agent_usable:** `false`
- **Definición:** Carga final de vigencia.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 010. `first_seen_at`

- **agent_usable:** `false`
- **Definición:** Fecha-hora técnica de primera observación.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 011. `last_seen_at`

- **agent_usable:** `false`
- **Definición:** Fecha-hora técnica de última observación.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 012. `stale_at`

- **agent_usable:** `false`
- **Definición:** Fecha-hora técnica de obsolescencia.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 013. `source_deleted_flag`

- **agent_usable:** `false`
- **Definición:** Flag de borrado en origen.
- **Estado:** Campo técnico no autorizado.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 014. `sales_document`

- **agent_usable:** `true`
- **Definición:** Identificador del documento de ventas, representado como texto canónico; conservar ceros iniciales.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 015. `sales_document_type`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 016. `order_reason`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 017. `sales_organization`

- **agent_usable:** `true`
- **Definición:** Identifica la región comercial de la transacción mediante el mapa documentado.
- **Estado:** Permitida para el agente.
- **Derivaciones permitidas:** `region_comercial`, `mercado`.
- **Derivación permitida:** `fabrica_derivada`.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 018. `distribution_channel`

- **agent_usable:** `true`
- **Definición:** Identifica el formato del producto: E3=Granel; E4=Ensacado.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `formato_producto`.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 019. `sales_group`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 020. `sales_group_descript`

- **agent_usable:** `true`
- **Definición:** Vendedor bajo el que está registrado el cliente en SAP.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 021. `sold_to_party`

- **agent_usable:** `true`
- **Definición:** Número de obra/solicitante según la definición heredada; representado como texto canónico. No reinterpretar como cliente/pagador por su nombre inglés.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 022. `sold_to_party_descript`

- **agent_usable:** `true`
- **Definición:** Nombre asociado a la obra/solicitante identificada por `sold_to_party`; no sustituye al nombre restringido del cliente/pagador.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 023. `postal_code`

- **agent_usable:** `true`
- **Definición:** Código postal de la obra.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 024. `city`

- **agent_usable:** `true`
- **Definición:** Población de la obra.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 025. `country_key`

- **agent_usable:** `true`
- **Definición:** País de la obra.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 026. `customers_account_n`

- **agent_usable:** `true`
- **Definición:** Número de cliente/pagador, identificador canónico textual; conservar ceros iniciales.
- **Estado:** Permitida para identificar y filtrar por cliente/pagador inequívoco.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 027. `customer`

- **agent_usable:** `false`
- **Definición:** Nombre de cliente asociado a `customers_account_n`. La definición existe, pero el flag no autoriza usar el nombre ni resolver identidades con él.
- **Estado:** Uso bloqueado, incluida la resolución por nombre; requiere autorización explícita y actualización del flag.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 028. `material_number`

- **agent_usable:** `true`
- **Definición:** Código de material, texto canónico con ceros iniciales, para el cruce condicionado con `Tabla de materiales.xlsx`.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `fabrica_derivada`.
- **Lookup permitido:** `Tabla de materiales.xlsx` para calidad y atributos auxiliares.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 029. `material_description`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 030. `material_group`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 031. `material_group_descript`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 032. `sales_document_item_1`

- **agent_usable:** `false`
- **Definición:** Correspondencia de datos con el antiguo campo de tipo de posición; no es una equivalencia con un número de posición. La definición funcional continúa pendiente y el campo no está autorizado.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Precaución:** No inferir semántica numérica de posición a partir del sufijo del nombre nuevo.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 033. `reason_for_rejection`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 034. `sales_unit`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 035. `plant`

- **agent_usable:** `true`
- **Definición:** Centro de expedición del cemento.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `fabrica_derivada`.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 036. `route`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 037. `route_description`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 038. `delivery_priority`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 039. `net_price`

- **agent_usable:** `true`
- **Definición:** Precio neto del albarán, por tonelada según el contrato funcional heredado; aplicar controles de unidad antes del valor de entrega derivado.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `valor_neto_entrega`.
- **Control obligatorio:** Verificar unidades para el cálculo o etiquetado; no usar campos restringidos como sustitutos de unidad.
- **Tipo actual:** `DOUBLE`.
- **Nulabilidad de esquema:** Sí.

### 040. `net_value_of_the_order`

- **agent_usable:** `true`
- **Definición:** Valor neto registrado en la fuente; no confundir con `valor_neto_entrega` derivado.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `DOUBLE`.
- **Nulabilidad de esquema:** Sí.

### 041. `tax_amount`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `DOUBLE`.
- **Nulabilidad de esquema:** Sí.

### 042. `currency`

- **agent_usable:** `true`
- **Definición:** Moneda del documento; EUR en la instantánea revisada, sin garantía de moneda única en futuras cargas.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 043. `price_group`

- **agent_usable:** `true`
- **Definición:** Identifica quién gestiona la entrega: CE=Votorantim; CP=cliente recoge o gestiona el pedido.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `gestion_entrega`.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 044. `price_group_descript`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 045. `customer_group`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 046. `customer_group_descript`

- **agent_usable:** `true`
- **Definición:** Identifica el grupo al que pertenece el cliente.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 047. `terms_of_payment_key`

- **agent_usable:** `true`
- **Definición:** Código relativo al tipo de pago.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 048. `terms_of_payment_descript`

- **agent_usable:** `true`
- **Definición:** Descripción de las condiciones de pago.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 049. `region`

- **agent_usable:** `true`
- **Definición:** Provincia del cliente/pagador según la codificación INE o numeración alfabética de provincias de España heredada; no es la región comercial derivada.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 050. `region_description`

- **agent_usable:** `true`
- **Definición:** Nombre de la provincia asociado a `region`.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 051. `net_weight_1`

- **agent_usable:** `true`
- **Definición:** Peso neto entregado al cliente, medida de volumen comercial según la regla heredada. El contrato lo expresa en toneladas, pero el esquema no declara unidad de peso; aplicar los controles de unidad antes de cálculos o etiquetas.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `valor_neto_entrega`.
- **Control obligatorio:** Verificar unidades para el cálculo o etiquetado; no usar campos restringidos como sustitutos de unidad.
- **Tipo actual:** `DOUBLE`.
- **Nulabilidad de esquema:** Sí.

### 052. `date_on_which_record`

- **agent_usable:** `false`
- **Definición:** Campo de fecha-hora con evento funcional no confirmado. Su representación está verificada; no permite concluir que sea creación, pedido, entrega o salida.
- **Estado:** No permitido hasta resolver la equivalencia ambigua y autorizar expresamente su uso.
- **Representación observada:** TIMESTAMP en microsegundos sin zona horaria declarada; valores a medianoche y nulos. No serial Excel ni hora fraccionaria.
- **Tipo actual:** `TIMESTAMP[µs] sin zona horaria (INT64; isAdjustedToUTC=false)`.
- **Nulabilidad de esquema:** Sí.

### 053. `distance`

- **agent_usable:** `true`
- **Definición:** Distancia entre expedición y entrega, en kilómetros cuando `um_for_distance` declara KM; no interpretar unidad vacía y cero como un trayecto real nulo.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `DOUBLE`.
- **Nulabilidad de esquema:** Sí.

### 054. `um_for_distance`

- **agent_usable:** `true`
- **Definición:** Unidad declarada de distancia: KM o cadena vacía en la instantánea revisada; el vacío no acredita una unidad.
- **Estado:** Permitida para el agente.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 055. `artifact_key`

- **agent_usable:** `false`
- **Definición:** Campo técnico nuevo. Su semántica exacta, alcance y estabilidad están pendientes de confirmación; no se presume que reemplace un identificador antiguo.
- **Estado:** Campo nuevo sin autorización; pendiente de definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 056. `source_file`

- **agent_usable:** `false`
- **Definición:** Campo técnico nuevo de nombre indicativo de archivo fuente; su contrato no está confirmado y no equivale automáticamente a una hoja de origen.
- **Estado:** Campo nuevo sin autorización; pendiente de definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 057. `dom_index`

- **agent_usable:** `false`
- **Definición:** Campo técnico nuevo, entero; el nombre no permite confirmar su significado ni equipararlo a un número de fila previo.
- **Estado:** Campo nuevo sin autorización; pendiente de definición funcional.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 058. `_dg_equivalence_key`

- **agent_usable:** `false`
- **Definición:** Campo técnico nuevo de equivalencia; construcción, unicidad y uso como clave no confirmados.
- **Estado:** Campo nuevo sin autorización; pendiente de definición funcional.
- **Tipo actual:** `STRING (BYTE_ARRAY, UTF8)`.
- **Nulabilidad de esquema:** Sí.

### 059. `_dg_occurrence_count`

- **agent_usable:** `false`
- **Definición:** Campo técnico nuevo de ocurrencias; no asumir que sea un factor de expansión, una cantidad comercial ni un multiplicador.
- **Estado:** Campo nuevo sin autorización; pendiente de definición funcional.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

### 060. `_dg_occurrence_index`

- **agent_usable:** `false`
- **Definición:** Campo técnico nuevo de ocurrencias; orden y significado pendientes de confirmación.
- **Estado:** Campo nuevo sin autorización; pendiente de definición funcional.
- **Tipo actual:** `INT64`.
- **Nulabilidad de esquema:** Sí.

## Resumen de flags

Recuento calculado de los bloques del catálogo nuevo, no del resumen histórico:

- **Columnas físicas:** 60.
- **Usables (`true`):** 24.
- **No usables (`false`):** 36; incluyen campos técnicos, campos funcionales sin autorización y la fecha ambigua.
- **Cambios de `false` a `true`:** ninguno.
- Los derivados no se incluyen en el recuento de columnas físicas.

La versión anterior contenía realmente 31 flags `true` y 94 `false` en sus 125 bloques.
Su resumen publicado indicaba cifras distintas; este documento corrige el recuento sin alterar permisos por ese motivo.

## Reglas operativas finales

- Usar exclusivamente los nombres nuevos del catálogo operativo; el apéndice es evidencia histórica, no un catálogo de alias.
- No sumar identificadores ni usar claves técnicas restringidas para deduplicar o expandir filas.
- Diferenciar nulo, cadena vacía y cero; no completar silenciosamente valores faltantes.
- Excluir registros sin `sales_document` en análisis comerciales. En la instantánea nueva no se observan nulos ni cadenas vacías en ese campo.
- No asumir que varias filas del mismo documento son duplicados: no se ha confirmado una clave de granularidad comercial autorizada.
- No inventar el significado de códigos no documentados ni extrapolar cambios de etiquetas como cambios comerciales.
- Proteger información de clientes, vendedores, precios y condiciones comerciales. Este documento no contiene ejemplos de clientes ni sus datos.
- No usar `customer` para resolver nombres ni `date_on_which_record` para análisis temporal mientras sus flags sean `false`.
- No computar magnitudes en toneladas ni el derivado de importe sin satisfacer el control de unidades.
- No ejecutar el lookup de calidad sin disponer de la tabla auxiliar validada.
- Si una futura instrucción contradice este documento, actualizar primero los metadatos y el flag antes de usar la columna.

## Apéndice de auditoría: equivalencias, exclusiones y método

**Este apéndice no es operativo.** Los nombres antiguos sirven únicamente para demostrar la migración.
No habilitan columnas eliminadas, aliases de consulta ni campos con flag `false`.

### Método y alcance de evidencia

- Se leyó íntegramente el Markdown original y se decodificaron todas las columnas de ambos Parquet,
  validando sus recuentos contra los footers: 125 y 60 columnas, respectivamente.
- No había lectores Parquet de alto nivel disponibles. Se utilizó lectura local de metadatos Thrift Compact
  y páginas Parquet con Snappy, PLAIN y diccionario/RLE; se validaron tamaños descomprimidos y filas de cada grupo.
  No se instalaron paquetes ni se enviaron datos a servicios externos.
- Se alinearon 228.383 documentos presentes **una sola vez** en cada instantánea,
  sin usar el orden de fila ni las claves técnicas como clave comercial.
- El cotejo normalizó solo para auditoría números históricos y sus representaciones textuales nuevas
  (padding y sufijos decimales nulos); el texto descriptivo se comparó tras trim, sin traducción ni fuzzy.
- En las filas alineadas, la identidad compuesta de obra, cliente, material y centro coincide en
  228.383 casos. Eso respalda el emparejamiento, no una autorización de nuevas claves.
- Los porcentajes se calculan sobre pares no nulos. Las diferencias pueden incluir cambios de instantánea,
  actualización de textos o codificación; no prueban por sí solas una alteración del significado.
  La evidencia de datos respalda correspondencias, pero no sustituye una especificación de ETL del propietario.
- Las columnas técnicas conservadas se identifican por nombre y tipo; no se afirma igualdad de sus valores ni estabilidad de IDs entre cargas.
- Ningún dato concreto de cliente se incluye en la evidencia publicada.

### Equivalencias confiables

| Columna antigua | Columna nueva | Tipo antiguo → nuevo | Pares no nulos | Concordancia normalizada | Alcance |
|---|---|---|---:|---:|---|
| `documento_de_ventas` | `sales_document` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `clase_doc_ventas` | `sales_document_type` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 99,9991 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `motivo_pedido` | `order_reason` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 223.814 | 99,9991 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `organizacion_ventas` | `sales_organization` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `canal_distribucion` | `distribution_channel` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `grupo_de_vendedores` | `sales_group` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `denomin_grvendedor` | `sales_group_descript` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 91,5874 % | Correspondencia del descriptivo respaldada por identificadores; texto no idéntico |
| `solicitante` | `sold_to_party` | DOUBLE → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `nombre_solic` | `sold_to_party_descript` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 90,1779 % | Correspondencia del descriptivo respaldada por identificadores; texto no idéntico |
| `codigo_postal` | `postal_code` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 99,5849 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `poblacion` | `city` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 99,2880 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `pais` | `country_key` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `pagador` | `customers_account_n` | DOUBLE → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `cliente` | `customer` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 98,4473 % | Correspondencia del descriptivo respaldada por identificadores; texto no idéntico |
| `material` | `material_number` | DOUBLE → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `denomin_material` | `material_description` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 46,4036 % | Correspondencia del descriptivo respaldada por identificadores; texto no idéntico |
| `grupo_de_articulos` | `material_group` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.382 | 100,0000 % | Concordancia de valores en el cotejo |
| `denom_gr_articulos` | `material_group_descript` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `motivo_de_rechazo` | `reason_for_rejection` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 224.906 | 99,9987 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `un_medida_venta` | `sales_unit` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 90,8929 % | Cambios de códigos documentados abajo; sigue no autorizado |
| `centro` | `plant` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `ruta` | `route` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.306 | 99,9974 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `nombre_ruta` | `route_description` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.306 | 99,9974 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `prioridad_de_entrega` | `delivery_priority` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `precio_neto` | `net_price` | DOUBLE → DOUBLE | 228.383 | 99,9991 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `valor_neto` | `net_value_of_the_order` | DOUBLE → DOUBLE | 228.383 | 99,9116 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `importe_del_impuesto` | `tax_amount` | DOUBLE → DOUBLE | 228.383 | 99,9142 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `moneda_del_documento` | `currency` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `grupo_de_precios` | `price_group` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `denomin_gr_precios` | `price_group_descript` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `grupo_de_clientes` | `customer_group` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `denomin_gr_client` | `customer_group_descript` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `condiciones_de_pago` | `terms_of_payment_key` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `explic_cond_pago` | `terms_of_payment_descript` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 100,0000 % | Concordancia de valores en el cotejo |
| `region` | `region` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.368 | 99,3734 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `denomin_region` | `region_description` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 99,3857 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `distance` | `distance` | DOUBLE → DOUBLE | 228.383 | 99,9982 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `um_for_distance` | `um_for_distance` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.219 | 99,9982 % | Correspondencia funcional respaldada; no igualdad absoluta |
| `tipo_de_posicion` | `sales_document_item_1` | STRING (BYTE_ARRAY, UTF8) → STRING (BYTE_ARRAY, UTF8) | 228.383 | 99,9991 % | Es tipo de posición, no número de posición; sigue no autorizado |
| `peso_neto_entrega` | `net_weight_1` | DOUBLE → DOUBLE | 228.383 | 99,9987 % | Peso entregado, confirmado en casos discriminantes; unidad sujeta a control |
| `id` | `id` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `business_key` | `business_key` | STRING (BYTE_ARRAY, UTF8) → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `content_hash` | `content_hash` | STRING (BYTE_ARRAY, UTF8) → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `record_status` | `record_status` | STRING (BYTE_ARRAY, UTF8) → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `version_number` | `version_number` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `first_seen_run_id` | `first_seen_run_id` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `last_seen_run_id` | `last_seen_run_id` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `valid_from_run_id` | `valid_from_run_id` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `valid_to_run_id` | `valid_to_run_id` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `first_seen_at` | `first_seen_at` | STRING (BYTE_ARRAY, UTF8) → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `last_seen_at` | `last_seen_at` | STRING (BYTE_ARRAY, UTF8) → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `stale_at` | `stale_at` | STRING (BYTE_ARRAY, UTF8) → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |
| `source_deleted_flag` | `source_deleted_flag` | INT64 → mismo tipo | — | — | Continuidad técnica por nombre/tipo; uso bloqueado |

### Equivalencias no obvias resueltas con datos

- **`sales_document_item_1`:** coincide con `tipo_de_posicion` en 228.381
  de 228.383 pares. Los candidatos `posicion`, `posicion_sd` y `posicion_siguiente`
  muestran concordancia nula; no inferir un número de posición por el nombre inglés.
- **`net_weight_1`:** se eligió `peso_neto_entrega`, no `peso_neto_pedido` ni `peso_bruto`.
  En 619 pares donde peso pedido y entregado difieren, el nuevo coincide con el entregado
  en 619 y con el pedido en 0.
- **Descriptivos:** las claves de obra, cliente y material coinciden tras normalizar representación,
  pero algunos textos cambian; particularmente `denomin_material` → `material_description`
  no es una sustitución literal de contenidos. No se inventa una transformación de esos textos.
- **Unidades de venta:** se observaron transiciones `TO` → `TM`, `PAK` → `FAR` y `BAG` → `SAC`.
  Estas parejas son evidencia de codificación en los registros alineados, no equivalencias físicas autorizadas.
  `sales_unit` hereda el flag `false`; tampoco identifica la unidad de `net_weight_1`.

### Fecha ambigua: no atribuir un significado comercial

| Candidato antiguo | Campo nuevo | Pares no nulos | Concordancia de fecha tras conversión Excel → fecha timestamp |
|---|---|---:|---:|
| `fecha_reg_sal_merc` | `date_on_which_record` | 228.380 | 99,9733 % |
| `fe_mov_mcia_real` | `date_on_which_record` | 228.380 | 99,9733 % |
| `fecha_de_pedido2` | `date_on_which_record` | 228.110 | 65,6131 % |
| `creado_el` | `date_on_which_record` | 228.380 | 41,8080 % |

`fecha_reg_sal_merc` y `fe_mov_mcia_real` son idénticas en las 380.480
filas del archivo antiguo, incluidos sus nulos. Los datos no permiten decidir cuál originó
`date_on_which_record`. Sus flags anteriores eran distintos, por lo que no se transfiere una autorización
por similitud. El nuevo campo queda en `false` hasta confirmar el origen, evento funcional y autorización.
No se considera una equivalencia confirmada con creación ni con fecha de pedido.

### Columnas antiguas sin correspondencia vigente identificada

Las siguientes 70 columnas se excluyen del catálogo nuevo. «Sin correspondencia» significa
que no se ha identificado una equivalencia confiable en estos archivos; no demuestra su eliminación
física de todos los sistemas ni permite inventar reemplazos.

- `sociedad_facturadora`.
- `creado_el`.
- `hora`.
- `creado_por`.
- `sector`.
- `descripcion_sector`.
- `oficina_de_ventas`.
- `denomin_ofic_vta`.
- `division`.
- `referencia`.
- `grupo_de_clientes_1`.
- `grupo_de_clientes_2`.
- `grupo_de_clientes_3`.
- `grupo_de_clientes_4`.
- `grupo_de_clientes_5`.
- `area_ctrl_creditos`.
- `no_pedido_cliente`.
- `posicion`.
- `pos_suscept_entrega`.
- `relevante_p_factura`.
- `jerarquia_productos`.
- `cantidad_de_pedido`.
- `cantidad_acum_confir`.
- `peso_bruto`.
- `peso_neto_pedido`.
- `unidad_de_peso`.
- `unidad_de_volumen`.
- `almacen`.
- `pto_expedic_pto_recepcion`.
- `denomin_puentexped`.
- `cantidad_base`.
- `unidad_de_medida`.
- `devolucion`.
- `grupo_de_materiales`.
- `gr_imputacion_mat`.
- `grupo_de_reappels`.
- `grupo_de_comisiones`.
- `grupo_de_material_1`.
- `grupo_de_material_2`.
- `grupo_de_material_3`.
- `grupo_de_material_4`.
- `grupo_de_material_5`.
- `posicion_sd`.
- `zona_de_ventas`.
- `denomzvtas_zona_vta`.
- `lista_de_precios`.
- `texto_listaprec`.
- `incoterms`.
- `incoterm_parte_2`.
- `via_de_pago`.
- `signific_via_pago`.
- `fecha_de_pedido`.
- `status_entrega`.
- `status_total_credito`.
- `status_rechazo`.
- `status_global`.
- `fecha_de_pedido2`.
- `hora_de_entrega`.
- `dom_com_subsig`.
- `posicion_siguiente`.
- `cantidad_entrega`.
- `hora_sal_mercancias`.
- `tp_doc_subsiguiente`.
- `stat_tot_mov_mcia`.
- `status_movimiento_de_mercancias`.
- `source_sheet`.
- `_dg_source_row_number`.
- `_dg_row_id`.
- `_dg_source_id`.
- `_dg_load_id`.

Los dos candidatos de fecha descritos arriba tampoco se mantienen como bloques operativos.
Los campos técnicos nuevos `artifact_key`, `source_file`, `dom_index`, `_dg_equivalence_key`,
`_dg_occurrence_count` y `_dg_occurrence_index` no se equiparan automáticamente a
`source_sheet`, `_dg_source_row_number`, `_dg_row_id`, `_dg_source_id` ni `_dg_load_id`.

### Huellas de las fuentes comparadas

SHA-256 calculado sobre cada archivo completo:

- Antiguo, Parquets/V2: `5974dd4a434ca62bbc601d1f980ce6c0ae4849f78bbc4eb858cbc11d2c40cc0d`.
- Nuevo, Parquets/Raw_2024-2026: `6bb0a71e63315521d07b843e4ed3988e130a8537cd676d366103072fdb9d09a2`.
