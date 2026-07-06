---
name: nps-analyst
description: >
  Análisis de NPS (Net Promoter Score) para Despegar/Decolar. Se activa cuando el usuario pide:
  cortes de NPS, métricas de promotores/detractores, promoter rate, detractor rate, queries sobre
  encuestas de satisfacción, tablas del data lake relacionadas con NPS (asrpt_clusters_hmc,
  asci_nps_secuence_metrics, bi_nps_fact_surveys, bi_requests_fact_header, dp_checkout_fact_recovery,
  asci_motivos_detraccion_gpt, as_call_fact_detail, bi_message_fact_header, entre otras), journey NPS,
  buy failures NPS, segmentos A1-C3, contact center y NPS (PCRC, Service Level), motivos de
  detracción, cumplimiento de targets NPS, o cualquier consulta sobre el KPI NPS de Despegar/Decolar.
  NO se activa para SQL genérico, Excel, presentaciones, o métricas distintas a NPS.
---

# NPS Analyst Skill

## Activación

Esta skill se activa cuando el analista pide análisis de NPS, cortes de NPS, queries sobre encuestas de satisfacción, tablas del data lake relacionadas con NPS, métricas de promotores/detractores, journey NPS, buy failures NPS, segmentos A1-C3, contact center y NPS, o cualquier consulta que involucre el KPI NPS de Despegar/Decolar.

## Flujo de trabajo: Lógica de dos niveles

### Nivel 1 — Leer siempre primero la Guía de Ruteo

Al activarse, **leer PRIMERO la guía de ruteo** usando:

```
GoogleWorkspace_get_doc_as_markdown
  document_id: 1JAov4S3SWvsxcHeqrC-snI2Mf19D5-GJHv_CeOCba18
```

La guía de ruteo indica:
- Qué **sección** del Knowledge Base NPS leer según el caso de uso
- Si la consulta es simple (un solo caso mapeado) → leer solo esa sección
- Si la consulta es compleja, cruza múltiples dominios, pide metodología, o no está mapeada → leer el KB completo

### Nivel 2 — Lectura selectiva del Knowledge Base NPS

Una vez identificada la sección correcta en la guía de ruteo, leer SOLO esa sección del Knowledge Base:

```
GoogleWorkspace_get_doc_as_markdown
  document_id: 1Ml4u029GmDBc68x8FvOYBfxOkkyLGqeOVASRxDTlXSc
```

**Cuándo leer solo una sección específica:**
- La consulta coincide con un caso de uso mapeado en la guía (ej: "NPS por journey", "NPS de buy failures", "NPS por agente")
- La consulta es directa y predecible

**Cuándo leer el KB completo:**
- La consulta cruza múltiples dominios simultáneamente
- La consulta pide metodología, fórmulas, o el "por qué" de un cálculo
- La consulta es ambigua o puede interpretarse de múltiples formas
- El caso de uso no está mapeado en la guía
- El analista dice: "no estoy seguro de qué tabla usar"
- Es la primera consulta del analista en una sesión

## Formatos de reporte por caso de uso

Algunos casos de uso tienen un formato de output estándar definido. Cuando aplique, ejecutar el formato indicado **además** del SQL — no en lugar de él.

### Journey NPS → Reporte HTML

**Cuándo aplica:** el analista pide análisis, apertura o corte de NPS por journey (con o sin especificar región). No aplica si el analista pide solo el SQL o los datos crudos.

**Qué hacer:**
1. Ejecutar el SQL de la sección 2.6.1 del KB para obtener los datos.
2. Generar un archivo HTML self-contained con el reporte visual. El archivo de referencia de estilo está en `data/nps-journey/tabla_nps_brasil_q2_2026.html` — leerlo para replicar el CSS y la estructura exacta.
3. Guardar el archivo en `data/nps-journey/tabla_nps_<region>_<meses>_<año>.html`.

**Estructura del reporte HTML:**
- **Header:** fondo violeta (#5B21B6), título "NPS POR JOURNEY — [REGIÓN]", subtítulo con rango de meses y región.
- **Columnas por mes:** dos sub-columnas por mes: `Total` (fondo blanco) y `Top` (fondo amarillo #FFFBEB). Encabezado del mes incluye nombre abreviado + encuestas totales.
- **Celdas NPS:** valor grande con signo (+/-); verde (#16A34A) si positivo, rojo (#DC2626) si negativo, gris (#6b7280) si entre -1 y +1. Delta debajo con ▲ verde o ▼ rojo vs mes anterior. El mes más antiguo mostrado usa el mes previo como referencia de delta (traer ese mes en el SQL aunque no se muestre).
- **Secciones:** filas de encabezado en morado oscuro (#4C1D95) agrupando los buckets por journey.
- **Nombres de display para buckets:**
  - `machine_puro` → "Machine Puro" / subtítulo "sofia + solo_miv"
  - `HUMAN` → "Human" / subtítulo "Agente humano"
  - `MACHINE` → "Machine" / subtítulo "Bot / máquina"
  - `friccion` → "Fricción" / subtítulo "Con fricción"
  - `sin_friccion_asistido` → "Sin fric. Asistido" / subtítulo "Sin fricción — asistido"
  - `sin_friccion_no_asistido` → "Sin fric. No Asistido" / subtítulo "Sin fricción — no asist."
- **Orden de secciones:** MIV_SOFIA_PURO (Machine Puro) → BACK (Human, Machine) → CONTATO (Fricción, Sin fric. Asistido, Sin fric. No Asistido).
- **Columna Share (última):** barras de progreso, azul = Total%, naranja/amarillo = Top%. Valores del último mes completo.
- **Footer:** fuente `asci_nps_secuence_metrics` + nota de delay si el último mes es parcial.

> ℹ️ En el futuro se agregarán formatos estándar para otros casos de uso (buy failures, detractores, etc.). Esta sección se irá completando.

## Tablas clave de referencia

| Tabla | Uso |
|-------|-----|
| `data.lake.asrpt_clusters_hmc` | Tabla principal NPS (mes en curso y 2025+). Campos: flg_nps_result, flg_promoter, flg_detractor, flg_passive, cluster_human_machine, flg_top_user |
| `data.lake.asci_nps_secuence_metrics` | NPS por journey y journey_l2. ⚠️ Delay de 6 días |
| `data.lake.bi_nps_fact_surveys` | Tabla de encuestas NPS |
| `data.lake.bi_customer_dim_segmentation_pv` | Segmentación A1-C3 (join con from/to_datetime) |
| `data.lake.bi_transactional_fact_transactions` | Tabla transaccional madre |
| `data.lake.bi_transactional_fact_products` | Productos: hotel_name, destination_city_code, intrip_flag, flg_lowcost |
| `data.lake.bi_requests_fact_header` | Requests: request_type, request_state, creation_date |
| `data.lake.bi_requests_fact_cancellations` | Cancelaciones. ⚠️ Campo cancel_method incluye NO_ISSUE |
| `data.lake.clerk_rs_rescheduling` | Reprogramaciones: wake_up_date |
| `data.lake.dp_checkout_fact_recovery` | Buy failures: buy_failure_segment |
| `data.lake.asci_motivos_detraccion_gpt` | Clasificación GPT de verbatims de detracción |
| `data.lake.asrpt_nps_agente_detallado` | NPS a nivel agente individual |
| `data.lake.as_call_fact_detail` | Detalle de llamadas: acd_sec, call_incoming_flg |
| `data.lake.bi_message_fact_header` | Canales digitales (WhatsApp, chat) |
| `data.lake.asci_targets_npspv` | Targets NPS y cumplimiento |
| `data.lake.apocalypse_transactions` | Transacciones de contingencia (flg_contingencia) |
| `data.lake.genesys_conversation_v2` | Conversaciones telephony para Service Level |

## Comportamiento esperado al responder

1. **Generar el SQL correcto** con las tablas, joins, particiones y flags indicados en el KB
2. **Incluir advertencias críticas** cuando corresponda:
   - ⚠️ Buy failures: excluir `NO_ISSUE` (no son cancelaciones reales)
   - ⚠️ Journey: usar `asci_nps_secuence_metrics` con delay de 6 días
   - ⚠️ NPS ponderado vs real: conocer la diferencia según el caso
   - ⚠️ Partición obligatoria: `nps_answer_yearmonth >= '2024-01'` + `flg_nps_result IS NOT NULL`
   - ⚠️ Response rate benchmark mínimo: 10%
   - ⚠️ Significancia estadística: mínimo 30 respuestas por corte
3. **Si hay duda sobre qué tabla usar** → referir a Sección 0 del KB "Cómo elegir la tabla NPS correcta"
4. **Indicar siempre qué sección del KB consultó** (transparencia)

## Formato de respuesta sugerido

```
## Caso identificado
Esto es un corte de NPS por [caso] → Sección [X.X] del KB

## SQL
```sql
-- [SQL generado]
```

## Advertencias del KB
- ⚠️ [Advertencia 1]
- ⚠️ [Advertencia 2]

## Sección consultada
Sección [X.X] del Knowledge Base NPS v3.2

## Sugerencias adicionales
- [Variante o corte adicional relacionado]
```

## Cómo actualizar el Knowledge Base NPS

Esta skill también se activa cuando el usuario pide modificar, corregir, agregar o actualizar contenido del Knowledge Base NPS.

### Flujo de actualización

**Paso 1 — Leer el KB completo**
Antes de cualquier edición, leer el documento completo:
```
GoogleWorkspace_get_doc_as_markdown
  document_id: 1Ml4u029GmDBc68x8FvOYBfxOkkyLGqeOVASRxDTlXSc
```
Esto es obligatorio aunque el usuario indique una sección específica: es necesario para validar consistencia.

**Paso 2 — Validar consistencia antes de editar**
Antes de escribir cualquier cambio, verificar:
- Que los nombres de tablas y campos referenciados en el cambio existen en el diccionario de datos (secciones 3.x).
- Que los filtros de partición y flags son consistentes con los definidos en las secciones 1.2 y 1.3.
- Que el cambio no contradice ni duplica contenido de otras secciones.
- Que las advertencias críticas relevantes (delay, NO_ISSUE, BAU, etc.) están presentes donde corresponde.
Si hay inconsistencias, reportarlas al usuario antes de proceder.

**Paso 3 — Aplicar los cambios**
Usar exclusivamente `GoogleWorkspace_find_and_replace_doc` con `tab_id: "t.0"`:
- Buscar el texto existente a reemplazar con la mayor especificidad posible (frases únicas, no palabras sueltas).
- Si el texto exacto no existe, buscar una frase característica de esa misma oración y adaptar el find manteniendo el espíritu del cambio.
- Usar `match_case: true`.
- Para inserciones de texto nuevo (no reemplazos), anclar la búsqueda en el texto inmediatamente anterior al punto de inserción.
- NO usar `batch_update_doc` ni `modify_doc_text` en este documento: el tamaño y la complejidad del KB hacen que los índices de posición sean inestables.

**Paso 4 — Confirmar el resultado**
Después de cada operación de reemplazo, reportar:
- Qué texto se buscó y qué se insertó/reemplazó.
- Si se requirió adaptar la frase buscada y por qué.
- Que no se modificó contenido fuera del alcance del cambio pedido.

### Criterios de calidad para ediciones del KB
- Toda advertencia crítica debe estar marcada con ⚠️ y ubicada en el lugar donde la IA podría cometer el error, no solo en una sección general.
- Los snippets SQL deben usar Trino y seguir el patrón de CTEs del resto del documento.
- Las referencias a secciones deben usar la numeración existente (ej: "ver sección 1.3").
- El lenguaje debe ser imperativo y directo, igual que el resto del KB.

## Notas técnicas

- **Filtros obligatorios de partición:** `nps_answer_yearmonth >= '2024-01'` y `flg_nps_result IS NOT NULL`
- **Fechas:** usar `nps_answer_date` para análisis; `sended_date` para tasa de respuesta
- **FLag flg_top_user:** 1 = Pasaporte Global, 0/NULL = BAU. NO filtrar con `= 0` para BAU
- **FLag flg_contingencia:** join con `apocalypse_transactions` para excluir eventos extraordinarios
- **Carriers lowcost válidos:** FO, Y4, JA, B6, TS, WJ, VB, P5
