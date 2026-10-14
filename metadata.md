---
title: Metadatos operativos del agente comercial
version: "5.0"
updated_at: "2026-10-14"
dataset: current.parquet
column_count: 60
agent_usable_true: 42
agent_usable_false: 18
classification: uso interno confidencial
---

# Alcance operativo

`current.parquet` es la fuente de observaciones; `mappings.json` define siete dimensiones oficiales,
claves, atributos permitidos, estados y validaciones ejecutables. Este catálogo sustituye v4 sin
conservar apéndices históricos, campos eliminados ni alias operativos. No eleva ningún flag.

## Libertad de cálculo

Los campos con agent_usable=true pueden utilizarse libremente para seleccionar, filtrar, ordenar, agrupar, contar, sumar, calcular medias, mínimos, máximos, percentiles, diferencias, variaciones, porcentajes, ratios y comparaciones, siempre que la operación sea matemáticamente compatible con el tipo del campo.

El agente puede combinar varios campos utilizables en una misma operación, crear cálculos temporales durante la consulta y encadenar agregaciones. No necesita una autorización específica para cada fórmula.

Las columnas calculadas creadas durante la consulta son temporales y no modifican los archivos originales.

Cuando una unidad o significado estén documentados en este archivo, el agente los utilizará directamente. No exigirá certificados, aprobaciones o evidencias adicionales distintas de esta definición vigente.

## Reglas de cruce: no canonización física

- Mantener identificadores originales. Clave auxiliar compatible de obra/pagador: ancho 10;
  material: 18. Aceptar enteros exactos no negativos y texto ASCII numérico íntegro (también .0),
  sin truncar; rechazar bool, float, fracción no cero y longitud excedida. Decimal íntegro es admisible.
  Texto puramente numérico Unicode se rechaza; alfanumérico legítimo permanece exacto.
  Nulo/blanco nunca se rellena con cero. Reportar claves inválidas.
- No modificar DIM, no limpiar sus filas ni materializar un catálogo de aliases. Resolver en consulta
  el conjunto único de VALORES requeridos para la clave auxiliar: repetición de igual asignación
  puede resolverse; valores distintos producen `ambiguo`. Contar colisiones de representación
  solo en claves efectivamente consultadas; conflictos ajenos no invalidan toda la dimensión.
- Conservar todas las observaciones y su orden; no hacer merge expansor, escoger primera fila ni
  `drop_duplicates` global. Conservar sin coincidencia, ambiguo, inválido y asignación ausente.
- Geografía y CP: igualdad de texto exacta, sin trim, casefold, tildes, padding ni alias. Un valor
  con espacios no equivale al mismo sin espacios. Cobertura parcial no implica inexistencia comercial.
- No deduplicar raw por documento, hashes o contadores técnicos. El grano comercial sigue pendiente.
  Excluir de consulta comercial observaciones con documento ausente/invalidado, informando cuántas.

## Dimensiones oficiales y alcance

| DIM operativo | Clave raw → DIM | Atributos y uso permitido |
|---|---|---|
| DIM_Material.parquet | material_number → material_number (auxiliar 18) | fabrica (producción), embalaje, calidad, tipo_de_cemento, prod_kkc, kkc como etiqueta; cálculo KKC bloqueado |
| DIM_GrupoClientes.parquet | customers_account_n → customers_account_n (auxiliar 10) | denomin_central; consolidar todos los pagadores asignados unívocamente a la central; no nombre_del_cliente |
| DIM_Distribuidores.parquet | customers_account_n → customers_account_n (auxiliar 10) | pertenencia a lista; presencia = distribuidor documentado; ausencia = no figura, no prueba de no distribuidor; no cliente_ajustado |
| DIM_Geografia.parquet | region_description → region_description (exacta) | provincia_ajustada, ccaa, mercado_oficemen, pain, zona_comercial, mercado |
| DIM_OrgVnt.parquet | sales_organization → sales_office (exacta, vínculo explícito) | region_comercial_documentada oficial; raw no tiene sales_office |
| DIM_SalesDistrict.parquet | postal_code → postal_code (exacta) | sales_district; territorio/distrito/cluster comercial, no límites administrativos |
| DIM_CarteraClientes.parquet | organización+canal+obra; después organización+canal+pagador | gven para identificar responsable/asignación autorizada; no desempeño, competencia ni rankings |

Cartera: reconstruir claves por componentes físicos `orgvt`, `cdis`, `solic`/`pagador`; atributo autorizado `gven`.
No usar `busqueda_obra`/`busqueda_pagador` concatenadas.
En obra, comparar pagador mediante clave auxiliar compatible antes de resolver conjuntos de
asignaciones; representaciones íntegras equivalentes no son conflictos. No alterar valores originales.
Obra tiene prioridad. Si existe una asignación de obra contradictoria, ausente o con pagador distinto,
conservar estado y no buscar pagador. Pagador solo si obra no coincide o está ausente, y solo con
una asignación observada única. No existe tercer nivel por sales_group. `fuente` conserva el origen de la asignación válida;
`nivel_consultado` registra obra/pagador incluso en conflicto, y `estado` conserva el resultado. La etiqueta no certifica una
identidad laboral ni un crosswalk con descriptivos raw.

Geografía clasifica provincia del pagador (`region_description`), no provincia de obra.
CP, `city`, `country_key` pertenecen a obra. `pain` es el país etiquetado en DIM_Geografia:
no sustituir `country_key` ni unir dimensiones por campos técnicos. CCAA, mercado Oficemen,
zona y mercado son clasificaciones distintas. CP→territorio no establece CP→provincia ni país:
DIM_SalesDistrict no tiene país; no inferir jurisdicción por una coincidencia textual.

## Material, organizaciones y reglas heredadas

`fabrica` de DIM_Material es la clasificación oficial de producción. `plant` es el centro de expedición;
son conceptos separados. No igualarlos. La fábrica heredada difiere en 8.377 observaciones comparables
según auditoría, no es corrección automática del mapping. `region_comercial_documentada` de DIM_OrgVnt
prima para etiquetar organizaciones; no equiparar Norte Export/Norte, Sur Export/Madeira o Centro Export/Centro.
`mercado_organizacion`, `formato_canal` y `gestion_entrega` conservan los mapas explícitos del JSON,
con desconocido para código no contemplado; no equivalen al mercado geográfico ni al embalaje oficial.

Calidad y tipo se muestran como etiquetas exactas del mapping. No convertir no numérico/ausente en
clinker, no clasificar cemento blanco o material desconocido como clinker. `kkc` es STRING:
parseo explícito separado mediante Decimal, conservar literal N/A, nulo, inválido y número.

**Fábrica legacy opcional**, solo solicitud explícita y atributo `fabrica_legacy`, nunca default:
requiere organización/canal/material/plant válidos. Orden: Canarias→Tenerife; COUR+E3→Oural;
material auxiliar 000000000000200212→Oural; CSCB→Córdoba; CCAL/CCAC→Alconera;
CSBB→Bobadilla; CSNB→Niebla; CSHU→Niebla-Madeira; CSMA/CSGR/CSPM/CMAL/CMGR/CMPM→Málaga;
otro caso válido→Toral. E3=Granel y E4=Ensacado; otro canal impide fallback. Región legacy:
EO11/EO21 Norte, EO55/EO53 Sur, EO56 Madeira, EO57/EO79 Canarias, EO65/EO66 Centro, EO54 Export.
Esta regla conserva un concepto heredado separado; no la presentar como fábrica oficial de producción.

## Métricas comerciales

- net_weight_1 es una medida numérica de peso. Puede agregarse mediante suma, media, mínimo, máximo, mediana y percentiles.
- La unidad se obtiene de sales_unit. Cuando existan varias unidades, el agente debe separar el resultado por unidad o convertirlas solo si la equivalencia está definida en estos metadatos.
- `net_price` es un precio unitario y no debe sumarse directamente. Puede calcularse su media simple o ponderada.
- `net_value_of_the_order` es un importe monetario sumable por observación.
- Los importes deben separarse por currency cuando haya más de una moneda.
- `valor_neto_entrega` puede calcularse como `net_price * net_weight_1` cuando ambos valores sean válidos.
- El precio ponderado puede calcularse como: `SUM(net_price * net_weight_1) / SUM(net_weight_1)`.
- Los negativos se conservan.
- Los nulos se excluyen de la operación matemática y se informa de su número.
- Un conjunto sin filas produce “sin observaciones”, no un cero inventado.

## Catálogo completo de columnas físicas

| Nº | Columna | agent_usable | Tipo observado | Definición operativa |
|---:|---|:---:|---|---|
| 001 | `id` | false | INT64 | Identificador técnico de fila. |
| 002 | `business_key` | false | STRING (BYTE_ARRAY, UTF8) | Hash técnico de la clave de negocio. |
| 003 | `content_hash` | false | STRING (BYTE_ARRAY, UTF8) | Hash técnico del contenido. |
| 004 | `record_status` | false | STRING (BYTE_ARRAY, UTF8) | Estado técnico del registro. |
| 005 | `version_number` | false | INT64 | Versión técnica. |
| 006 | `first_seen_run_id` | false | INT64 | Primera carga en que se observó. |
| 007 | `last_seen_run_id` | false | INT64 | Última carga en que se observó. |
| 008 | `valid_from_run_id` | false | INT64 | Carga inicial de vigencia. |
| 009 | `valid_to_run_id` | false | INT64 | Carga final de vigencia. |
| 010 | `first_seen_at` | false | STRING (BYTE_ARRAY, UTF8) | Fecha-hora técnica de primera observación. |
| 011 | `last_seen_at` | false | STRING (BYTE_ARRAY, UTF8) | Fecha-hora técnica de última observación. |
| 012 | `stale_at` | false | STRING (BYTE_ARRAY, UTF8) | Fecha-hora técnica de obsolescencia. |
| 013 | `source_deleted_flag` | false | INT64 | Flag de borrado en origen. |
| 014 | `sales_document` | true | STRING (BYTE_ARRAY, UTF8) | Identificador del documento de ventas, representado como texto canónico; conservar ceros iniciales. |
| 015 | `sales_document_type` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 016 | `order_reason` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 017 | `sales_organization` | true | STRING (BYTE_ARRAY, UTF8) | Identifica la región comercial de la transacción mediante el mapa documentado. |
| 018 | `distribution_channel` | true | STRING (BYTE_ARRAY, UTF8) | Identifica el formato del producto: E3=Granel; E4=Ensacado. |
| 019 | `sales_group` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 020 | `sales_group_descript` | true | STRING (BYTE_ARRAY, UTF8) | Vendedor bajo el que está registrado el cliente en SAP. |
| 021 | `sold_to_party` | true | STRING (BYTE_ARRAY, UTF8) | Número de obra/solicitante según la definición heredada; representado como texto canónico. No reinterpretar como cliente/pagador por su nombre inglés. |
| 022 | `sold_to_party_descript` | true | STRING (BYTE_ARRAY, UTF8) | Nombre asociado a la obra/solicitante identificada por `sold_to_party`; no sustituye al nombre restringido del cliente/pagador. |
| 023 | `postal_code` | true | STRING (BYTE_ARRAY, UTF8) | Código postal de la obra. |
| 024 | `city` | true | STRING (BYTE_ARRAY, UTF8) | Población de la obra. |
| 025 | `country_key` | true | STRING (BYTE_ARRAY, UTF8) | País de la obra. |
| 026 | `customers_account_n` | true | STRING (BYTE_ARRAY, UTF8) | Número de cliente/pagador, identificador canónico textual; conservar ceros iniciales. |
| 027 | `customer` | true | STRING (BYTE_ARRAY, UTF8) | Nombre de cliente asociado a `customers_account_n`. La definición existe, pero el flag no autoriza usar el nombre ni resolver identidades con él. |
| 028 | `material_number` | true | STRING (BYTE_ARRAY, UTF8) | Código de material. Enlace auxiliar a DIM_Material.parquet sin alterar el original. |
| 029 | `material_description` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 030 | `material_group` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 031 | `material_group_descript` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 032 | `sales_document_item_1` | true | STRING (BYTE_ARRAY, UTF8) | Correspondencia de datos con el antiguo campo de tipo de posición; no es una equivalencia con un número de posición. La definición funcional continúa pendiente y el campo no está autorizado. |
| 033 | `reason_for_rejection` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 034 | `sales_unit` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 035 | `plant` | true | STRING (BYTE_ARRAY, UTF8) | Centro de expedición del cemento. |
| 036 | `route` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 037 | `route_description` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 038 | `delivery_priority` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 039 | `net_price` | true | DOUBLE | Precio neto del albarán, por tonelada según el contrato funcional heredado; aplicar controles de unidad antes del valor de entrega derivado. |
| 040 | `net_value_of_the_order` | true | DOUBLE | Valor neto registrado en la fuente; no confundir con `valor_neto_entrega` derivado. |
| 041 | `tax_amount` | true | DOUBLE | Definición funcional pendiente de confirmación con el propietario del dato. |
| 042 | `currency` | true | STRING (BYTE_ARRAY, UTF8) | Moneda del documento; EUR en la instantánea revisada, sin garantía de moneda única en futuras cargas. |
| 043 | `price_group` | true | STRING (BYTE_ARRAY, UTF8) | Identifica quién gestiona la entrega: CE=Votorantim; CP=cliente recoge o gestiona el pedido. |
| 044 | `price_group_descript` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 045 | `customer_group` | true | STRING (BYTE_ARRAY, UTF8) | Definición funcional pendiente de confirmación con el propietario del dato. |
| 046 | `customer_group_descript` | true | STRING (BYTE_ARRAY, UTF8) | Identifica el grupo al que pertenece el cliente. |
| 047 | `terms_of_payment_key` | true | STRING (BYTE_ARRAY, UTF8) | Código relativo al tipo de pago. |
| 048 | `terms_of_payment_descript` | true | STRING (BYTE_ARRAY, UTF8) | Descripción de las condiciones de pago. |
| 049 | `region` | true | STRING (BYTE_ARRAY, UTF8) | Provincia del cliente/pagador según la codificación INE o numeración alfabética de provincias de España heredada; no es la región comercial derivada. |
| 050 | `region_description` | true | STRING (BYTE_ARRAY, UTF8) | Nombre de la provincia asociado a `region`. |
| 051 | `net_weight_1` | true | DOUBLE | Peso neto entregado al cliente, medida de volumen comercial según la regla heredada. El contrato lo expresa en toneladas, pero el esquema no declara unidad de peso; aplicar los controles de unidad antes de cálculos o etiquetas. |
| 052 | `date_on_which_record` | true | TIMESTAMP[µs] sin zona horaria (INT64; isAdjustedToUTC=true) | Fecha de la operación comercial de la observación.
operaciones: filtrar, agrupar, ordenar, mínimo, máximo, año, trimestre, mes,
semana, día, MTD, YTD, comparación interanual y variación entre períodos. |
| 053 | `distance` | true | DOUBLE | Distancia entre expedición y entrega, en kilómetros cuando `um_for_distance` declara KM; no interpretar unidad vacía y cero como un trayecto real nulo. |
| 054 | `um_for_distance` | true | STRING (BYTE_ARRAY, UTF8) | Unidad declarada de distancia: KM o cadena vacía en la instantánea revisada; el vacío no acredita una unidad. |
| 055 | `artifact_key` | false | STRING (BYTE_ARRAY, UTF8) | Campo técnico nuevo. Su semántica exacta, alcance y estabilidad están pendientes de confirmación; no se presume que reemplace un identificador antiguo. |
| 056 | `source_file` | false | STRING (BYTE_ARRAY, UTF8) | Campo técnico nuevo de nombre indicativo de archivo fuente; su contrato no está confirmado y no equivale automáticamente a una hoja de origen. |
| 057 | `dom_index` | false | INT64 | Campo técnico nuevo, entero; el nombre no permite confirmar su significado ni equipararlo a un número de fila previo. |
| 058 | `_dg_equivalence_key` | false | STRING (BYTE_ARRAY, UTF8) | Campo técnico nuevo de equivalencia; construcción, unicidad y uso como clave no confirmados. |
| 059 | `_dg_occurrence_count` | false | INT64 | Campo técnico nuevo de ocurrencias; no asumir que sea un factor de expansión, una cantidad comercial ni un multiplicador. |
| 060 | `_dg_occurrence_index` | false | INT64 | Campo técnico nuevo de ocurrencias; orden y significado pendientes de confirmación. |

**Total verificado:** 60 columnas; 42 true, 18 false. Ningún permiso heredado elevado.
