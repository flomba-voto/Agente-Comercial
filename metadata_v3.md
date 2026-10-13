---
title: "Base de conocimiento de metadatos del consolidado de ventas y entregas"
version: "3.0"
updated_at: "2026-10-5"
dataset: "current.parquet"
material_lookup: "Tabla de materiales.xlsx"
column_count: 125
classification: "uso interno confidencial"
---

# Base de conocimiento de metadatos

## Política de uso

El flag `agent_usable` es obligatorio. El agente solo puede utilizar una columna cuando el valor sea `true`. Si es `false`, puede mencionar que el campo existe y está pendiente de definición, pero no puede usarlo para calcular, filtrar, segmentar, explicar ni concluir.

## Resolución inteligente de clientes

### Principio general

Cuando el usuario mencione un cliente, no filtres directamente `current.parquet`
por el texto introducido. Primero resuelve ese texto contra el catálogo de clientes
del dataset y obtén su identificador canónico.

La coincidencia debe realizarse sobre los valores únicos de:

- identificador único de cliente: `[COLUMNA_ID_CLIENTE]`
- nombre canónico del cliente: `[COLUMNA_NOMBRE_CLIENTE]`
- país, zona, canal u otras columnas útiles para desambiguar, si existen

Una vez identificado el cliente, aplica los filtros y cálculos utilizando
`[COLUMNA_ID_CLIENTE]`.

### Normalización de nombres

Normaliza tanto el texto del usuario como los nombres del dataset antes de compararlos:

1. convertir a minúsculas;
2. eliminar tildes y diacríticos;
3. sustituir puntuación y caracteres especiales por espacios;
4. eliminar espacios duplicados;
5. normalizar símbolos como `&` y `+`;
6. eliminar sufijos societarios cuando no sean distintivos, por ejemplo:
   `S.L.`, `SL`, `S.A.`, `SA`, `Ltda.`, `Inc.`, `Corp.` y equivalentes;
7. conservar números, localidades y términos distintivos.

### Orden de resolución

Busca candidatos en este orden:

1. coincidencia exacta con el nombre original;
2. coincidencia exacta con el nombre normalizado;
3. coincidencia con un alias registrado;
4. coincidencia difusa mediante `RapidFuzz`, combinando:
   - `fuzz.WRatio`;
   - `fuzz.token_set_ratio`;
5. ajuste por contexto usando país, zona, canal u otros atributos mencionados.

### Nivel de confianza

Calcula los candidatos y compáralos también con el segundo mejor resultado.

- **Confianza alta:** puntuación igual o superior a 90 y ventaja mínima de
  8 puntos frente al segundo candidato. Selecciona automáticamente.
- **Confianza media:** puntuación entre 75 y 89, o diferencia inferior a
  8 puntos entre los mejores candidatos. Solicita confirmación mostrando
  como máximo tres nombres canónicos.
- **Confianza baja:** puntuación inferior a 75. Indica que no se ha encontrado
  una coincidencia fiable y solicita más información.

No selecciones automáticamente cuando varios clientes tengan nombres similares,
aunque el primer resultado supere el umbral.

### Reglas de seguridad

- No combines registros de varios clientes como si fueran uno solo.
- No interpretes ausencia de coincidencia como ausencia de datos comerciales.
- No muestres cifras hasta haber resuelto el cliente de forma fiable.
- Si el usuario confirma un candidato, conserva su identificador canónico
  durante la conversación hasta que cambie de cliente.
- En la respuesta, utiliza el nombre canónico del dataset.
- Cuando una resolución no sea exacta, indica brevemente el cliente interpretado.

## Reglas de negocio confirmadas

### Región comercial por organización de ventas

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

### Mercado por organización de ventas

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

### Formato por canal de distribución

| Código | Formato |
|---|---|
| E3 | Granel |
| E4 | Ensacado |

### Gestión de entrega por grupo de precios

| Código | Resultado |
|---|---|
| CE | Entrega gestionada por Votorantim |
| CP | El cliente recoge o gestiona el pedido |

### Valor neto de entrega

```text
valor_neto_entrega = precio_neto * peso_neto_entrega
```

El resultado usa precio neto por tonelada y peso neto entregado en toneladas. Es un campo derivado y no debe sustituirse automáticamente por `valor_neto`.

### Fábrica derivada

Aplicar en este orden:

1. Región comercial Canarias → Tenerife.
2. Centro COUR y formato/embalaje Bulk → Oural.
3. Material 200212 → Oural.
4. Centro CSCB → Córdoba.
5. Centro CCAL o CCAC → Alconera.
6. Centro CSBB → Bobadilla.
7. Centro CSNB → Niebla.
8. Centro CSHU → Niebla-Madeira.
9. Centro CSMA, CSGR, CSPM, CMAL, CMGR o CMPM → Málaga.
10. Cualquier otro caso → Toral.

> La fórmula original usa una columna derivada DD con el literal `Bulk`, aunque los datos fuente terminan en DC. La implementación debe crear o normalizar explícitamente ese atributo antes de aplicar la regla COUR.

### Calidad de material

Cruzar `material` con la columna `Material` de `Tabla de materiales.xlsx` y devolver `Calidad`. Si no existe calidad o no es numérica, la regla indicada clasifica el producto como clinker.

> Control de coherencia: la tabla auxiliar contiene valores no numéricos `Clinker`, `Cto.Blanco` y `Porte`. Aplicar «no numérico = clinker» también clasificaría cemento blanco y portes como clinker. El agente debe mostrar esta advertencia en esos casos hasta que se definan excepciones.

La tabla auxiliar contiene 100 materiales, 100 códigos únicos, 85 calidades numéricas y 15 no numéricas.

## Campos derivados autorizados

| Campo | agent_usable | Fuentes |
|---|---|---|
| `region_comercial` | true | `organizacion_ventas` |
| `mercado` | true | `organizacion_ventas` |
| `formato_producto` | true | `canal_distribucion` |
| `gestion_entrega` | true | `grupo_de_precios` |
| `valor_neto_entrega` | true | `precio_neto`, `peso_neto_entrega` |
| `fabrica_derivada` | true | región, centro, material y formato/embalaje |
| `calidad_material` | true | lookup por material |
| `tipo_producto_calidad` | true con advertencia | calidad numérica/no numérica |

## Metadatos por columna

### 001. `id`

- **agent_usable:** `false`
- **Definición:** Identificador técnico de fila.
- **Estado:**

### 002. `business_key`

- **agent_usable:** `false`
- **Definición:** Hash técnico de la clave de negocio.
- **Estado:**

### 003. `content_hash`

- **agent_usable:** `false`
- **Definición:** Hash técnico del contenido.
- **Estado:**

### 004. `record_status`

- **agent_usable:** `false`
- **Definición:** Estado técnico del registro.
- **Estado:**

### 005. `version_number`

- **agent_usable:** `false`
- **Definición:** Versión técnica.
- **Estado:**

### 006. `first_seen_run_id`

- **agent_usable:** `false`
- **Definición:** Primera carga en que se observó.
- **Estado:**

### 007. `last_seen_run_id`

- **agent_usable:** `false`
- **Definición:** Última carga en que se observó.
- **Estado:**

### 008. `valid_from_run_id`

- **agent_usable:** `false`
- **Definición:** Carga inicial de vigencia.
- **Estado:**

### 009. `valid_to_run_id`

- **agent_usable:** `false`
- **Definición:** Carga final de vigencia.
- **Estado:**

### 010. `first_seen_at`

- **agent_usable:** `false`
- **Definición:** Fecha-hora técnica de primera observación.
- **Estado:**

### 011. `last_seen_at`

- **agent_usable:** `false`
- **Definición:** Fecha-hora técnica de última observación.
- **Estado:**

### 012. `stale_at`

- **agent_usable:** `false`
- **Definición:** Fecha-hora técnica de obsolescencia.
- **Estado:**

### 013. `source_deleted_flag`

- **agent_usable:** `false`
- **Definición:** Flag de borrado en origen.
- **Estado:**

### 014. `documento_de_ventas`

- **agent_usable:** `true`
- **Definición:** Identificador del documento de ventas.
- **Estado:** Permitida para el agente.

### 015. `sociedad_facturadora`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 016. `creado_el`

- **agent_usable:** `true`
- **Definición:** Fecha de creación como serial Excel.
- **Estado:** Permitida para el agente.

### 017. `hora`

- **agent_usable:** `true`
- **Definición:** Hora de creación como fracción de día.
- **Estado:** Permitida para el agente.

### 018. `creado_por`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 019. `clase_doc_ventas`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 020. `motivo_pedido`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 021. `organizacion_ventas`

- **agent_usable:** `true`
- **Definición:** Identifica la región comercial de la transacción mediante el mapa documentado.
- **Estado:** Permitida para el agente.
- **Derivaciones permitidas:** `region_comercial`, `mercado`.
- **Derivación permitida:** `fabrica_derivada`.

### 022. `canal_distribucion`

- **agent_usable:** `true`
- **Definición:** Identifica el formato del producto: E3=Granel; E4=Ensacado.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `formato_producto`.

### 023. `sector`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 024. `descripcion_sector`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 025. `grupo_de_vendedores`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 026. `denomin_grvendedor`

- **agent_usable:** `true`
- **Definición:** Vendedor bajo el que está registrado el cliente en SAP.
- **Estado:** Permitida para el agente.

### 027. `oficina_de_ventas`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 028. `denomin_ofic_vta`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 029. `division`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 030. `referencia`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 031. `solicitante`

- **agent_usable:** `true`
- **Definición:** Número de obra.
- **Estado:** Permitida para el agente.

### 032. `nombre_solic`

- **agent_usable:** `true`
- **Definición:** Nombre asociado al número de solicitante u obra.
- **Estado:** Permitida para el agente.

### 033. `codigo_postal`

- **agent_usable:** `true`
- **Definición:** Código postal de la obra.
- **Estado:** Permitida para el agente.

### 034. `poblacion`

- **agent_usable:** `true`
- **Definición:** Población de la obra.
- **Estado:** Permitida para el agente.

### 035. `pais`

- **agent_usable:** `true`
- **Definición:** País de la obra.
- **Estado:** Permitida para el agente.

### 036. `pagador`

- **agent_usable:** `true`
- **Definición:** Hace referencia al número de cliente.
- **Estado:**

### 037. `cliente`

- **agent_usable:** `false`
- **Definición:** Hace referencia al nombre de cliente asociado a un determinado numero de cliente (columna pagador).
- **Estado:**

### 038. `grupo_de_clientes_1`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 039. `grupo_de_clientes_2`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 040. `grupo_de_clientes_3`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 041. `grupo_de_clientes_4`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 042. `grupo_de_clientes_5`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 043. `area_ctrl_creditos`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 044. `no_pedido_cliente`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 045. `posicion`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 046. `material`

- **agent_usable:** `true`
- **Definición:** Código de material usado para cruzar con `Tabla de materiales.xlsx`.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `fabrica_derivada`.
- **Lookup permitido:** `Tabla de materiales.xlsx` para calidad y atributos auxiliares.

### 047. `denomin_material`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 048. `grupo_de_articulos`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 049. `denom_gr_articulos`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 050. `tipo_de_posicion`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 051. `pos_suscept_entrega`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 052. `relevante_p_factura`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 053. `motivo_de_rechazo`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 054. `jerarquia_productos`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 055. `un_medida_venta`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 056. `cantidad_de_pedido`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 057. `cantidad_acum_confir`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 058. `peso_bruto`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 059. `peso_neto_pedido`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 060. `unidad_de_peso`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 061. `unidad_de_volumen`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 062. `centro`

- **agent_usable:** `true`
- **Definición:** Centro de expedición del cemento.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `fabrica_derivada`.

### 063. `almacen`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 064. `pto_expedic_pto_recepcion`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 065. `denomin_puentexped`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 066. `ruta`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 067. `nombre_ruta`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 068. `prioridad_de_entrega`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 069. `precio_neto`

- **agent_usable:** `true`
- **Definición:** Precio neto por tonelada del albarán.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `valor_neto_entrega`.

### 070. `cantidad_base`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 071. `unidad_de_medida`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 072. `devolucion`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 073. `grupo_de_materiales`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 074. `gr_imputacion_mat`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 075. `grupo_de_reappels`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 076. `grupo_de_comisiones`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 077. `grupo_de_material_1`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 078. `grupo_de_material_2`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 079. `grupo_de_material_3`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 080. `grupo_de_material_4`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 081. `grupo_de_material_5`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 082. `valor_neto`

- **agent_usable:** `true`
- **Definición:** Valor neto registrado en la fuente; no confundir con el valor neto de entrega derivado.
- **Estado:** Permitida para el agente.

### 083. `importe_del_impuesto`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 084. `moneda_del_documento`

- **agent_usable:** `true`
- **Definición:** Moneda del documento.
- **Estado:** Permitida para el agente.

### 085. `posicion_sd`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 086. `grupo_de_precios`

- **agent_usable:** `true`
- **Definición:** Identifica quién gestiona la entrega: CE=Votorantim; CP=cliente recoge o gestiona el pedido.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `gestion_entrega`.

### 087. `denomin_gr_precios`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 088. `grupo_de_clientes`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 089. `denomin_gr_client`

- **agent_usable:** `true`
- **Definición:** Identifica el grupo al que pertenece el cliente.
- **Estado:** Permitida para el agente.

### 090. `zona_de_ventas`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 091. `denomzvtas_zona_vta`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 092. `lista_de_precios`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 093. `texto_listaprec`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 094. `incoterms`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 095. `incoterm_parte_2`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 096. `condiciones_de_pago`

- **agent_usable:** `true`
- **Definición:** Código relativo al tipo de pago.
- **Estado:** Permitida para el agente.

### 097. `explic_cond_pago`

- **agent_usable:** `true`
- **Definición:** Descripción de las condiciones de pago.
- **Estado:** Permitida para el agente.

### 098. `via_de_pago`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 099. `signific_via_pago`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 100. `region`

- **agent_usable:** `true`
- **Definición:** Provincia del pagador según codificación INE o numeración alfabética de provincias de España.
- **Estado:** Permitida para el agente.

### 101. `denomin_region`

- **agent_usable:** `true`
- **Definición:** Nombre de la provincia asociado a `region`.
- **Estado:** Permitida para el agente.

### 102. `fecha_de_pedido`

- **agent_usable:** `true`
- **Definición:** Columna vacía en el conjunto actual.
- **Estado:** Permitida para el agente.

### 103. `status_entrega`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 104. `status_total_credito`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 105. `status_rechazo`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 106. `status_global`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 107. `fecha_de_pedido2`

- **agent_usable:** `true`
- **Definición:** Fecha alternativa de pedido como serial Excel.
- **Estado:** Permitida para el agente.

### 108. `hora_de_entrega`

- **agent_usable:** `true`
- **Definición:** Hora de entrega como fracción de día.
- **Estado:** Permitida para el agente.

### 109. `dom_com_subsig`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 110. `posicion_siguiente`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 111. `cantidad_entrega`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 112. `peso_neto_entrega`

- **agent_usable:** `true`
- **Definición:** Peso de cemento entregado al cliente en toneladas; es la medida usada cuando se habla de volumen.
- **Estado:** Permitida para el agente.
- **Derivación permitida:** `valor_neto_entrega`.

### 113. `fe_mov_mcia_real`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 114. `fecha_reg_sal_merc`

- **agent_usable:** `true`
- **Definición:** Fecha de salida de la mercancía del punto de expedición.
- **Estado:** Permitida para el agente.

### 115. `hora_sal_mercancias`

- **agent_usable:** `true`
- **Definición:** Hora de salida como fracción de día.
- **Estado:** Permitida para el agente.

### 116. `tp_doc_subsiguiente`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 117. `stat_tot_mov_mcia`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 118. `status_movimiento_de_mercancias`

- **agent_usable:** `false`
- **Definición:** Definición funcional pendiente de confirmación con el propietario del dato.
- **Estado:** No permitida hasta confirmar la definición funcional.

### 119. `distance`

- **agent_usable:** `true`
- **Definición:** Distancia en kilómetros entre el punto de expedición y el punto de entrega.
- **Estado:** Permitida para el agente.

### 120. `um_for_distance`

- **agent_usable:** `true`
- **Definición:** Unidad declarada de distancia.
- **Estado:** Permitida para el agente.

### 121. `source_sheet`

- **agent_usable:** `false`
- **Definición:** Año u hoja de origen.
- **Estado:**

### 122. `_dg_source_row_number`

- **agent_usable:** `false`
- **Definición:** Número técnico de fila en la fuente.
- **Estado:**

### 123. `_dg_row_id`

- **agent_usable:** `false`
- **Definición:** ID técnico de fila.
- **Estado:**

### 124. `_dg_source_id`

- **agent_usable:** `false`
- **Definición:** ID técnico de fuente.
- **Estado:**

### 125. `_dg_load_id`

- **agent_usable:** `false`
- **Definición:** ID técnico de carga.
- **Estado:**

## Resumen de flags

- **Usables:** 48.
- **No usables pendientes de definición:** 77.

## Reglas operativas finales

- No sumar identificadores.
- Diferenciar nulo, cadena vacía y cero.
- Excluir la fila residual sin documento de ventas en análisis comerciales.
- No inventar el significado de códigos no documentados.
- Proteger información de clientes, vendedores, precios y condiciones comerciales.
- Si una futura instrucción contradice este documento, actualizar primero los metadatos y el flag antes de usar la columna.
