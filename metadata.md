---
title: Metadatos operativos del agente comercial
version: "6.0"
updated_at: "2026-10-14"
dataset: current.parquet
column_count: 60
agent_usable_true: 41
agent_usable_false: 19
classification: uso interno confidencial
---

# Metadatos operativos del agente comercial

## Estado y alcance

El encabezado recuperado declaraba 42/18, pero las filas del catálogo contienen 41/19.
Esta edición corrige los contadores sin cambiar los flags de las columnas.


Propuesta final de edición para sustituir el archivo existente. Corresponde a mappings.json 3.0.
No acredita despliegue, permisos, motor de ejecución ni inspección de los bytes actuales.
El catálogo conserva 60 nombres físicos y 41 campos comerciales habilitados. Los 19 campos
restantes son técnicos. No se crean fuentes ni se modifican Parquet.

## Política de cálculo y calidad

### calculation

Se proponen filtros combinados, agrupaciones, ordenación, pivotes, estadísticas, diferencias, ratios y columnas temporales sobre campos habilitados. No se exige una certificación externa para describir valores registrados. Las etiquetas de negocio y conversiones requieren definición conocida.

### access

agent_usable describe uso analítico, no concede permisos. Aplicar acceso efectivo del usuario. No exponer secretos ni evaluar rendimiento de personas.

### categorical

Campos categóricos con definición pendiente admiten análisis literal. Identificadores no se suman ni promedian.

### rows

Contar filas no exige sales_document. Contar documentos distintos excluye documentos nulos sin eliminar sus filas de otros análisis. No excluir registros por una regla de invalidación no documentada.

### joins

Enriquecer sin multiplicar ni perder filas. Resolver una tupla única de atributos por clave; las coincidencias contradictorias quedan sin asignación. Conservar el orden durante el enriquecimiento; el resultado agregado puede ordenarse libremente.

### keys

Comparar IDs mediante claves auxiliares exactas y validadas sin alterar originales. Conservar ceros y alfanuméricos. No redondear, truncar ni completar nulos. Mantener joins textuales exactos; la búsqueda de etiquetas no sustituye la clave.

### coverage

Un conflicto de una dimensión solo afecta a resultados que la utilizan. Incluir categorías sin asignación y reconciliar desgloses. Distinguir falta de acceso, conjunto vacío y ausencia de valores válidos.

### currency

Separar importes por moneda. Moneda vacía queda como no identificada, nunca como EUR supuesto. No sumar monedas distintas ni convertir sin tipo de cambio documentado.

### weight

Estadísticas del peso registrado disponibles. La unidad de sales_unit no se presume unidad del peso. Toneladas y conversiones solo con relación de unidades documentada para el alcance.

### amount

SUM(net_value_of_the_order) se etiqueta suma de valores registrados por moneda. No se declara ingreso o ventas certificadas mientras no se conozca si el importe se repite por documento.

### price

Precio: media simple y estadísticas, no suma como ingreso. Importe derivado y precio ponderado requieren precio y peso de bases compatibles. Numerador y denominador utilizan el mismo conjunto de pares válidos.

### nulls

COUNT de filas incluye nulos; COUNT de valores no. Estadísticas omiten valores ausentes y reportan cobertura. Todos nulos producen resultado no disponible. Denominador cero o nulo no produce ratio. Conservar negativos; no sustituirlos por valores absolutos.

### dates

Usar date_on_which_record para fechas comerciales. No asumir UTC ni desplazar horas hasta inspeccionar el tipo real. MTD/YTD y comparaciones declaran corte y períodos equivalentes. Ausencia de registros no acredita cobertura temporal completa.

### kkc

Convertir solo valores numéricos inequívocos para estadísticas descriptivas de KKC registrado. Separar N/A, nulos e inválidos. No sumar coeficientes como cantidades ni derivar emisiones o clinker equivalente sin unidad, escala y fórmula documentadas.

### factory

Fábrica de producción procede de la dimensión material. plant es expedición. Las clasificaciones heredadas no reemplazan la oficial y no se activan por defecto.

### execution

Toda cifra final requiere cálculo real sobre todas las filas del alcance. Una configuración no implementa un motor. Si falta ejecución o lectura, informar la limitación sin fabricar cifras.

## Métricas propuestas

| Métrica | Etiqueta | Operación | Condiciones |
|---|---|---|---|
| `conteo_observaciones` | Observaciones del alcance | `count_rows` | Valores válidos y alcance explícito. |
| `documentos_distintos` | Códigos de documento distintos no nulos | `count_distinct` | Valores válidos y alcance explícito. |
| `pagadores_distintos` | Códigos de pagador distintos no nulos | `count_distinct` | Valores válidos y alcance explícito. |
| `obras_distintas` | Códigos de obra distintos no nulos | `count_distinct` | Valores válidos y alcance explícito. |
| `peso_registrado` | Suma del peso registrado; unidad no confirmada | `sum` | No presentar como toneladas sin unidad documentada. |
| `peso_toneladas` | Peso en toneladas cuando la unidad esté documentada | `sum` | Unidad y conversión verificables para el alcance. |
| `suma_valor_registrado` | Suma de valores registrados por moneda | `sum` | No equivale automáticamente a ingresos o valor de entregas. |
| `valor_neto_entrega` | Importe derivado de precio por peso compatible | `sum_product` | Unidad de peso y base de precio compatibles y documentadas.; Moneda identificada; pares válidos; grano de entrega conocido para usar esa etiqueta. |
| `precio_medio` | Media simple del precio registrado por moneda y base compatible | `mean` | Valores válidos y alcance explícito. |
| `precio_ponderado` | Precio ponderado por peso compatible | `sum_product_div_sum` | Mismo conjunto de pares válidos en numerador y denominador.; Denominador distinto de cero; base de precio/peso compatible. |
| `distancia_registrada` | Estadísticas de distancia por unidad registrada | `descriptive` | Valores válidos y alcance explícito. |
| `kkc_registrado` | Estadísticas de KKC registrado, no indicador de sostenibilidad | `descriptive_numeric_parse` | Conversión numérica inequívoca; no reemplazar N/A o nulos por cero. |

## Dimensiones y enlaces

| Dimensión | Archivo | Enlace | Atributos |
|---|---|---|---|
| material | `DIM_Material.parquet` | material_number → material_number | fabrica, embalaje, calidad, tipo_de_cemento, prod_kkc, kkc |
| grupo_clientes | `DIM_GrupoClientes.parquet` | customers_account_n → customers_account_n | denomin_central |
| distribuidores | `DIM_Distribuidores.parquet` | customers_account_n → customers_account_n | figura_en_mapping |
| geografia | `DIM_Geografia.parquet` | region_description → region_description | ccaa, provincia_ajustada, mercado_oficemen, pain, zona_comercial, mercado |
| organizacion | `DIM_OrgVnt.parquet` | sales_organization → sales_office | region_comercial_documentada |
| territorio | `DIM_SalesDistrict.parquet` | postal_code → postal_code | sales_district |
| cartera | `DIM_CarteraClientes.parquet` | sales_organization → orgvt, distribution_channel → cdis, sold_to_party → solic | gven, asignada, fuente, estado, nivel_consultado |

La cartera consulta primero organización, canal y obra, comprobando pagador. Si no hay clave
utilizable o coincidencia, consulta organización, canal y pagador. Una asignación contradictoria
no se sustituye por otra vía. Los campos SAP de vendedor siguen siendo descriptivos independientes.
Los atributos excluidos en mappings.json no se incorporan automáticamente a la proyección.
La ausencia en la lista de distribuidores no demuestra que una entidad no sea distribuidora.
Geografía del pagador y territorio de obra son ámbitos diferentes.

## Información todavía no confirmada

- Unidad efectiva del peso y relación con sales_unit.
- Grano de cada observación y posible repetición del importe por documento.
- Atributo UTC del timestamp: las fuentes recuperadas discrepan.
- Unidad, escala y fórmula de KKC.

Estas ausencias limitan etiquetas y fórmulas dependientes, no todo el análisis.
No se rellenan con supuestos ni se convierten en certificados ficticios.
Los mapas heredados y derivados no se reconstruyen desde instrucciones textuales de origen.
No se habilitan clasificaciones inferidas. La fábrica oficial sigue disponible en DIM_Material.

## Catálogo de columnas

| Nº | Columna | agent_usable | Tipo de referencia | Definición |
|---:|---|:---:|---|---|
| 001 | `id` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 002 | `business_key` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 003 | `content_hash` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 004 | `record_status` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 005 | `version_number` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 006 | `first_seen_run_id` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 007 | `last_seen_run_id` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 008 | `valid_from_run_id` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 009 | `valid_to_run_id` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 010 | `first_seen_at` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 011 | `last_seen_at` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 012 | `stale_at` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 013 | `source_deleted_flag` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 014 | `sales_document` | true | STRING | Código de documento registrado. Admite recuento distinto; no demuestra que cada fila sea una entrega o un pedido diferente. |
| 015 | `sales_document_type` | true | STRING | Categoría registrada de sales_document_type. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 016 | `order_reason` | true | STRING | Categoría registrada de order_reason. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 017 | `sales_organization` | true | STRING | Código de organización de ventas; enlace a sales_office en la dimensión de organización. |
| 018 | `distribution_channel` | true | STRING | Código del canal registrado. Su interpretación comercial debe corresponder a las etiquetas documentadas. |
| 019 | `sales_group` | true | STRING | Código SAP registrado; disponible para descripción, no como sustituto de la asignación de cartera. |
| 020 | `sales_group_descript` | true | STRING | Descripción SAP del vendedor asociado. Uso descriptivo o de asignación; no evaluación de personas. |
| 021 | `sold_to_party` | true | STRING | Código de obra o solicitante; entidad distinta del pagador. |
| 022 | `sold_to_party_descript` | true | STRING | Etiqueta de la obra o solicitante. No identifica por sí sola un pagador. |
| 023 | `postal_code` | true | STRING | Código postal de la obra, almacenado como identificador textual. |
| 024 | `city` | true | STRING | Población registrada de la obra. |
| 025 | `country_key` | true | STRING | Código de país registrado de la obra. |
| 026 | `customers_account_n` | true | STRING | Código de cliente o pagador; clave para agrupar y enlazar dimensiones. |
| 027 | `customer` | true | STRING | Etiqueta registrada del pagador. La búsqueda por nombre devuelve códigos candidatos dentro del acceso autorizado; nombres repetidos no son claves únicas. |
| 028 | `material_number` | true | STRING | Identificador de material para enlazar la dimensión material. |
| 029 | `material_description` | true | STRING | Categoría registrada de material_description. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 030 | `material_group` | true | STRING | Categoría registrada de material_group. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 031 | `material_group_descript` | true | STRING | Categoría registrada de material_group_descript. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 032 | `sales_document_item_1` | true | STRING | Categoría registrada cuyo significado funcional no está confirmado. No usar como número de posición ni como clave de deduplicación. |
| 033 | `reason_for_rejection` | true | STRING | Categoría registrada de reason_for_rejection. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 034 | `sales_unit` | true | STRING | Código de unidad comercial registrado. No se ha confirmado que determine la unidad de net_weight_1. |
| 035 | `plant` | true | STRING | Centro de expedición registrado; no equivale a fábrica de producción. |
| 036 | `route` | true | STRING | Categoría registrada de route. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 037 | `route_description` | true | STRING | Categoría registrada de route_description. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 038 | `delivery_priority` | true | STRING | Categoría registrada de delivery_priority. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 039 | `net_price` | true | DOUBLE | Precio neto registrado. La referencia recuperada menciona precio por tonelada, pero la compatibilidad con el peso debe comprobarse antes de calcular un importe derivado. |
| 040 | `net_value_of_the_order` | true | DOUBLE | Importe registrado. Su suma por moneda es una suma de valores de filas; no acredita ingresos ni elimina posibles repeticiones de importe documental. |
| 041 | `tax_amount` | true | DOUBLE | Valor numérico registrado con significado funcional pendiente. Estadísticas descriptivas disponibles; no presentar como impuesto liquidado. |
| 042 | `currency` | true | STRING | Código de moneda del registro. Los cálculos monetarios se separan por moneda; moneda ausente se informa por separado. |
| 043 | `price_group` | true | STRING | Código registrado relativo a la gestión de entrega. |
| 044 | `price_group_descript` | true | STRING | Categoría registrada de price_group_descript. Se permite análisis por valor literal sin atribuir significados funcionales no confirmados. |
| 045 | `customer_group` | true | STRING | Código registrado del grupo del cliente. |
| 046 | `customer_group_descript` | true | STRING | Descripción registrada del grupo del cliente. |
| 047 | `terms_of_payment_key` | true | STRING | Código registrado de condiciones de pago. |
| 048 | `terms_of_payment_descript` | true | STRING | Descripción registrada de condiciones de pago. |
| 049 | `region` | true | STRING | Código de provincia del pagador; no región comercial ni provincia de la obra. |
| 050 | `region_description` | true | STRING | Etiqueta de provincia del pagador, utilizada para el enlace geográfico. |
| 051 | `net_weight_1` | true | DOUBLE | Peso neto registrado. La unidad física no queda acreditada por el tipo DOUBLE ni por sales_unit. Estadísticas en base registrada disponibles; toneladas requieren unidad documentada. |
| 052 | `date_on_which_record` | true | TIMESTAMP_US | Fecha comercial de referencia para filtros y agrupaciones. El atributo UTC discrepa entre las fuentes recuperadas; inspeccionar el tipo real y no desplazar horas por suposición. |
| 053 | `distance` | true | DOUBLE | Distancia registrada; etiquetar kilómetros únicamente cuando um_for_distance indique KM. |
| 054 | `um_for_distance` | true | STRING | Unidad de distancia registrada; un valor vacío no define una unidad. |
| 055 | `artifact_key` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 056 | `source_file` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 057 | `dom_index` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 058 | `_dg_equivalence_key` | false | STRING | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 059 | `_dg_occurrence_count` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |
| 060 | `_dg_occurrence_index` | false | INT64 | Campo técnico excluido de la proyección comercial; no utilizar como cantidad, ponderador o clave de deduplicación comercial. |

## Verificación antes de sustituir

Comprobar lectura JSON, nombres de campos, correspondencia de versiones y proyección de 41/19.
Probar en el consumidor existente: conteo de filas, suma registrada por moneda, filtro temporal,
cruce sin expansión y precio ponderado con pares compatibles. Las nuevas propiedades no garantizan
que el consumidor implemente las operaciones. No declarar probado lo que no se haya ejecutado.
Los recuentos y hashes históricos del JSON no se usan como resultados de la carga actual.
