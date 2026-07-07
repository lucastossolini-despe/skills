---
name: nps-analyst
description: >
  Análisis de NPS (Net Promoter Score) y experiencia del cliente para Despegar/Decolar. Se activa
  cuando el usuario pide: cortes de NPS, NPS ponderado, NPS real, métricas de promotores/detractores,
  promoter rate, detractor rate, queries sobre encuestas de satisfacción, satisfacción del cliente,
  experiencia del cliente, score de satisfacción, score de experiencia, posventa, análisis de
  posventa, verbatims, opiniones de clientes, encuestas, tablas del data lake relacionadas con NPS
  (asrpt_clusters_hmc, asci_nps_secuence_metrics, bi_nps_fact_surveys, bi_requests_fact_header,
  dp_checkout_fact_recovery, asci_motivos_detraccion_gpt, as_call_fact_detail, bi_message_fact_header,
  entre otras), journey NPS, buy failures, análisis de cancelaciones, análisis de reprogramaciones,
  segmentos A1-C3, contact center, PCRC, TMO, tasa de contacto, Service Level, motivos de detracción,
  cumplimiento de targets NPS, o cualquier consulta sobre el KPI NPS de Despegar/Decolar. También se
  activa cuando el analista menciona explícitamente esta skill, el Knowledge Base NPS, o pide usar el
  KB de NPS para cualquier análisis, aunque no sea exclusivamente de NPS.
  NO se activa para SQL genérico, Excel, o presentaciones sin contexto de NPS o experiencia del cliente.
---

# NPS Analyst Skill

## Activación

Esta skill se activa en cualquiera de estos casos:

1. **Pedido explícito de NPS:** cortes de NPS, NPS ponderado, NPS real, promoter/detractor rate, encuestas NPS, journey NPS, targets NPS, cumplimiento NPS.
2. **Satisfacción y experiencia del cliente:** satisfacción del cliente, experiencia del cliente, score de satisfacción, score de experiencia, verbatims, opiniones de clientes, posventa.
3. **Análisis relacionados con las tablas del KB:** buy failures, cancelaciones, reprogramaciones, contact center, PCRC, TMO, tasa de contacto, Service Level, segmentos A1-C3, motivos de detracción.
4. **Activación explícita por el analista:** cuando el analista menciona esta skill, el Knowledge Base NPS, o pide explícitamente "usá el KB de NPS" / "activá el NPS analyst" / "consultá el KB" — aunque el análisis no sea de NPS puro. En ese caso, leer el KB completo antes de responder.

**NO se activa** para SQL genérico sin contexto de NPS o experiencia del cliente, ni para Excel o presentaciones standalone.


## Flujo de trabajo

```
Paso 1 — Consultar la tabla de ruteo compacta (abajo, en esta misma SKILL).
Paso 2 — Si el caso coincide INEQUÍVOCAMENTE con una fila de la tabla:
          → leer solo la sección indicada del KB.
Paso 3 — En CUALQUIER otro caso (ambigüedad, múltiples dominios, metodología,
          caso no mapeado, primera consulta sin contexto previo, o la menor duda):
          → leer el KB COMPLETO.

⚠️ REGLA DE ORO: ante la duda, SIEMPRE leer el KB completo.
   Es mejor leer de más que construir un query incorrecto.

Knowledge Base NPS (documento completo):
  GoogleWorkspace_get_doc_as_markdown
    document_id: 1Ml4u029GmDBc68x8FvOYBfxOkkyLGqeOVASRxDTlXSc

Guía de ruteo detallada (referencia opcional, NO es paso obligatorio):
  GoogleWorkspace_get_doc_as_markdown
    document_id: 1JAov4S3SWvsxcHeqrC-snI2Mf19D5-GJHv_CeOCba18
```

## Tabla de ruteo compacta

| Caso de uso | Sección KB | Notas |
|---|---|---|
| NPS por journey / journey_l2 (formato estándar) | 2.6.1 | ⚠️ Usar asci_nps_secuence_metrics. Delay 6 días. Ver SQL embebido abajo. |
| NPS por journey L2 desagregado (sofia y solo_miv por separado) | 2.6.2 | Solo cuando se pide explícitamente sin Machine Puro |
| NPS ponderado y real (últimos N meses) | SQL embebido abajo | ⭐ PEDIDO RECURRENTE. Ejecutar SQL, mostrar tabla HTML. Escala -1 a 1. |
| NPS general (últimos N meses, sin journey) | 2.1.1 | Tabla asrpt_clusters_hmc por defecto |
| NPS por mes con delta | 2.1.2 + 1.2-A | Requiere LAG |
| NPS por región/país | 2.1.4 | Campo country_code en tabla principal |
| NPS por cluster (Human vs Machine) | 2.1.5 | Campo cluster_human_machine |
| NPS por segmento A1-C3 | 2.6.3 + 1.2-F | Join con bi_customer_dim_segmentation_pv |
| NPS Top User / Pasaporte Global | 2.6.4 | flg_top_user = 1 |
| NPS buy failures | 2.3.3 | ⚠️ Excluir NO_ISSUE |
| NPS cancelaciones | 2.3.4 | ⚠️ Excluir cancel_method = NO_ISSUE |
| NPS reprogramaciones | 2.4.2 | Join con clerk_rs_rescheduling |
| NPS por aérea/carrier | 2.6.11 | Join con bi_transactional_fact_products |
| NPS por hotel o destino | 2.6.9 / 2.6.10 | Mínimo 30 respuestas |
| NPS por agente | 3.22 | Tabla asrpt_nps_agente_detallado |
| PCRC y Service Level | 2.5 | Tabla genesys_conversation_v2 |
| Motivos de detracción (GPT) | 3.21 | Tabla asci_motivos_detraccion_gpt |
| Cumplimiento de targets | 3.18 | Join con asci_targets_npspv |
| Promoter rate / detractor rate / response rate | 1.2-A | Fórmulas derivadas |
| Cómo elegir la tabla NPS correcta | Sección 0 | Árbol de decisión de tablas |


**Si el caso no está en esta tabla, o si el pedido combina más de un caso, o si hay alguna ambigüedad: leer el KB COMPLETO.**


## SQL embebido: NPS Ponderado y Real — Pedido recurrente

SQL para responder el pedido recurrente "dame el NPS ponderado y real de los últimos N meses".
Fuente: `data.lake.asrpt_clusters_hmc`. Escala: -1 a 1 (igual que los targets). NO multiplicar por 100.

**⚠️ Cómo calcular las fechas:**
- FECHA_INICIO = primer día del primer mes solicitado
- FECHA_FIN    = primer día del mes siguiente al último solicitado
- Ejemplo para ene–jun 2026: FECHA_INICIO = '2026-01-01', FECHA_FIN = '2026-07-01'

```sql
WITH rta AS (
    -- NPS real por type (sobre respondidas)
    SELECT
        nps_answer_yearmonth                              AS periodo,
        type,
        CAST(SUM(flg_answered)   AS DOUBLE)              AS respondidos,
        CAST(SUM(flg_nps_result) AS DOUBLE)              AS resultado
    FROM data.lake.asrpt_clusters_hmc
    WHERE (sended_date     BETWEEN DATE 'FECHA_INICIO' AND DATE 'FECHA_FIN')
       OR (nps_answer_date BETWEEN DATE 'FECHA_INICIO' AND DATE 'FECHA_FIN')
      AND nps_answer_yearmonth >= '2024-01'
    GROUP BY 1, 2
    HAVING SUM(CASE WHEN nps_answer_date IS NOT NULL THEN 1 ELSE 0 END) > 0
),
env AS (
    -- Share de envíos por type (sobre enviadas con flg_sent)
    SELECT
        date_format(sended_date, '%Y-%m')                AS periodo,
        type,
        CAST(SUM(flg_sent) AS DOUBLE)
            / CAST(SUM(SUM(flg_sent)) OVER (
                PARTITION BY date_format(sended_date, '%Y-%m')
              ) AS DOUBLE)                               AS share
    FROM data.lake.asrpt_clusters_hmc
    WHERE (sended_date     BETWEEN DATE 'FECHA_INICIO' AND DATE 'FECHA_FIN')
       OR (nps_answer_date BETWEEN DATE 'FECHA_INICIO' AND DATE 'FECHA_FIN')
      AND nps_answer_yearmonth >= '2024-01'
    GROUP BY 1, 2
)
SELECT
    g.periodo                                AS Periodo,
    SUM(g.resultado_ponderado)               AS NPS_Ponderado,
    SUM(g.resultado) / SUM(g.respondidos)    AS NPS_Real
FROM (
    SELECT
        r.periodo,
        r.type,
        r.resultado,
        r.respondidos,
        e.share,
        (r.resultado / r.respondidos) * e.share AS resultado_ponderado
    FROM rta r
    LEFT JOIN env e ON r.periodo = e.periodo AND r.type = e.type
) g
GROUP BY 1
ORDER BY 1
```

⚠️ NPS Ponderado y NPS Real en escala -1 a 1. Los targets también están en esa escala (ej: 0.42 = 42%).
⚠️ El share se calcula con `SUM(flg_sent)` (enviadas), no con `COUNT(DISTINCT survey_id)`.
⚠️ El NPS por type se calcula con `SUM(flg_nps_result)/SUM(flg_answered)`, no con `AVG`.

## Formato de salida: tabla HTML para NPS Ponderado y Real

Cuando el resultado sea el pedido recurrente "NPS ponderado y real", o el KPI general de NPS, mostrar siempre una tabla HTML con:
- Una fila por mes
- Columnas: Periodo | NPS Ponderado | NPS Real | Delta Ponderado (vs mes anterior)
- NPS formateado como porcentaje con 1 decimal (ej: 43.5%)
- Delta con flecha ▲/▼ y color verde/rojo
- Guardar en `data/nps-general/tabla_nps_ponderado_YYYY-MM_YYYY-MM.html`
**CSS mínimo:**
```css
table { border-collapse: collapse; width: 100%; font-family: Arial, sans-serif; }
th, td { border: 1px solid #ddd; padding: 8px; text-align: center; }
th { background-color: #4472C4; color: white; }
tr:nth-child(even) { background-color: #f2f2f2; }
.delta-pos { color: #28a745; font-weight: bold; }
.delta-neg { color: #dc3545; font-weight: bold; }
```

## SQL embebido: NPS por Journey (sección 2.6.1)

SQL completo para NPS por journey + journey_l2 con Machine Puro, deltas y share.


```sql
-- NPS por journey + journey_l2 | Formato estándar con Machine Puro
-- CÓMO CALCULAR LAS FECHAS (siempre relativo al rango solicitado):
--   FECHA_INICIO_N1 = primer día del mes ANTERIOR al primero solicitado
--                     (necesario para calcular el delta del primer mes mostrado)
--   FECHA_FIN       = primer día del mes SIGUIENTE al último mes solicitado
--   MES_SHARE       = primer día del último mes solicitado (mes de referencia para share)
--
-- Ejemplo — si el usuario pide abril, mayo, junio 2026:
--   FECHA_INICIO_N1 = '2026-03-01'
--   FECHA_FIN       = '2026-07-01'
--   MES_SHARE       = '2026-06-01'
--
-- REGION_FILTRO:
--   Hispa   → AND region = 'Hispa'
--   Brasil  → AND region = 'Brasil'
--   Total   → (omitir la condición)
--
WITH base AS (
    SELECT
        journey, journey_l2,
        DATE_TRUNC('month', nps_answer_date) AS mes,
        flg_nps_result, flg_top_user, survey_id
    FROM data.lake.asci_nps_secuence_metrics
    WHERE nps_answer_date >= DATE 'FECHA_INICIO_N1'
      AND nps_answer_date <  DATE 'FECHA_FIN'
      AND flg_nps_result  IS NOT NULL
      -- AND REGION_FILTRO  -- ← descomentar y ajustar arriba
),
total AS (
    SELECT journey, journey_l2, mes,
        ROUND(100.0 * AVG(flg_nps_result), 1) AS nps,
        COUNT(DISTINCT survey_id)              AS encuestas
    FROM base GROUP BY journey, journey_l2, mes
),
top_u AS (
    SELECT journey, journey_l2, mes,
        ROUND(100.0 * AVG(flg_nps_result), 1) AS nps_top,
        COUNT(DISTINCT survey_id)              AS encuestas_top
    FROM base WHERE flg_top_user = 1
    GROUP BY journey, journey_l2, mes
),
share_ref AS (
    SELECT journey, journey_l2,
        COUNT(DISTINCT survey_id)                               AS enc_jl2,
        SUM(COUNT(DISTINCT survey_id)) OVER ()                  AS enc_total,
        SUM(CASE WHEN flg_top_user = 1 THEN 1 ELSE 0 END)      AS enc_top_jl2,
        SUM(SUM(CASE WHEN flg_top_user = 1 THEN 1 ELSE 0 END)) OVER () AS enc_top_total
    FROM base WHERE mes = DATE 'MES_SHARE'
    GROUP BY journey, journey_l2
),
joined AS (
    SELECT t.journey, t.journey_l2, t.mes, t.nps, t.encuestas,
        tp.nps_top, COALESCE(tp.encuestas_top, 0) AS encuestas_top,
        ROUND(100.0 * sr.enc_jl2     / NULLIF(sr.enc_total, 0), 1) AS share_total_pct,
        ROUND(100.0 * sr.enc_top_jl2 / NULLIF(sr.enc_top_total, 0), 1) AS share_top_pct
    FROM total t
    LEFT JOIN top_u tp ON t.journey=tp.journey AND t.journey_l2=tp.journey_l2 AND t.mes=tp.mes
    LEFT JOIN share_ref sr ON t.journey=sr.journey AND t.journey_l2=sr.journey_l2
    WHERE t.journey_l2 IS NOT NULL AND t.journey_l2 != 'NA'
),
machine_puro AS (
    SELECT 'MIV_SOFIA_PURO' AS journey, 'machine_puro' AS journey_l2, mes,
        ROUND(SUM(nps * encuestas) / NULLIF(SUM(encuestas), 0), 1)             AS nps,
        SUM(encuestas)                                                          AS encuestas,
        ROUND(SUM(nps_top * encuestas_top) / NULLIF(SUM(encuestas_top), 0), 1) AS nps_top,
        SUM(encuestas_top)    AS encuestas_top,
        SUM(share_total_pct)  AS share_total_pct,
        SUM(share_top_pct)    AS share_top_pct
    FROM joined WHERE journey = 'MIV_SOFIA_PURO'
    GROUP BY mes
),
final AS (
    SELECT * FROM joined WHERE journey != 'MIV_SOFIA_PURO'
    UNION ALL
    SELECT * FROM machine_puro
)
SELECT * FROM final
ORDER BY journey, journey_l2, mes
```

> ⚠️ El delta se calcula en el cliente comparando cada fila con el mes N-1 del mismo `journey_l2`.
> ⚠️ Validar mínimo 30 respuestas por bucket antes de mostrar NPS Top en Machine Puro.

## Formato HTML para Journey NPS

### Cuándo aplica

Cuando el analista pide análisis, apertura o corte de NPS por journey para presentar o compartir (no solo SQL o datos crudos).

### Pasos

1. Ejecutar el SQL embebido arriba (o la sección 2.6.1 del KB si hay dudas).
2. Generar el HTML con el CSS y estructura documentados abajo.
3. Guardar en `data/nps-journey/tabla_nps_<region>_<meses>_<año>.html`.
4. El archivo `data/nps-journey/tabla_nps_brasil_q2_2026.html` puede usarse como validación visual pero **NO es la fuente de verdad del estilo** — este documento lo es.

#### CSS embebido (copiar tal cual en el `<style>` del HTML)

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}
body {
    font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: #f3f4f6;
    padding: 20px;
    color: #1f2937;
}
.container {
    max-width: 1200px;
    margin: 0 auto;
    background: white;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 4px 20px rgba(0,0,0,0.08);
}
/* Header */
.header {
    background: linear-gradient(135deg, #5B21B6 0%, #4C1D95 100%);
    color: white;
    padding: 24px 28px;
    display: flex;
    align-items: center;
    gap: 16px;
}
.logo {
    width: 48px;
    height: 48px;
    background: #7C3AED;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.5rem;
    font-weight: 700;
    flex-shrink: 0;
}
.header-text h1 {
    font-size: 1.4rem;
    font-weight: 700;
    margin-bottom: 4px;
}
.header-text p {
    font-size: 0.9rem;
    opacity: 0.9;
}
/* Table */
.table-wrapper {
    overflow-x: auto;
}
table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.85rem;
}
/* Section headers */
.section-header {
    background: #4C1D95;
    color: white;
    font-weight: 700;
    text-transform: uppercase;
    font-size: 0.75rem;
    letter-spacing: 0.5px;
}
.section-header td {
    padding: 10px 12px;
}
/* Sub-section headers (journey_l2 name) */
.row-header {
    background: #f9fafb;
}
.row-header td {
    padding: 10px 12px;
    border-bottom: 1px solid #e5e7eb;
}
.bucket-name {
    font-weight: 700;
    font-size: 0.9rem;
    color: #1f2937;
}
.bucket-subtitle {
    font-size: 0.72rem;
    color: #6b7280;
    font-weight: 400;
    margin-top: 2px;
}
/* Month headers */
.month-header {
    background: #6B21A8;
    color: white;
    text-align: center;
    font-weight: 600;
}
.month-header td {
    padding: 8px 6px;
}
.month-name {
    font-size: 0.85rem;
    font-weight: 700;
}
.month-surveys {
    font-size: 0.65rem;
    opacity: 0.85;
    margin-top: 2px;
}
.sub-header {
    background: #7E22CE;
    color: white;
    text-align: center;
    font-size: 0.7rem;
    font-weight: 500;
}
.sub-header td {
    padding: 6px 4px;
}
/* Data rows */
.data-row td {
    padding: 10px 8px;
    border-bottom: 1px solid #e5e7eb;
    vertical-align: middle;
}
.cell-total {
    background: white;
    text-align: center;
}
.cell-top {
    background: #FFFBEB;
    text-align: center;
}
.cell-share {
    background: #f9fafb;
    padding: 8px 12px;
}
/* NPS values */
.nps-value {
    font-size: 1.4rem;
    font-weight: 700;
    line-height: 1.2;
}
.nps-positive {
    color: #16A34A;
}
.nps-negative {
    color: #DC2626;
}
.nps-neutral {
    color: #6b7280;
}
/* Delta values */
.delta {
    font-size: 0.72rem;
    margin-top: 2px;
    font-weight: 500;
}
.delta-up {
    color: #16A34A;
}
.delta-down {
    color: #DC2626;
}
.delta-flat {
    color: #6b7280;
}
/* Share bars */
.share-bar-container {
    margin-bottom: 6px;
}
.share-label {
    font-size: 0.65rem;
    color: #6b7280;
    margin-bottom: 2px;
    display: flex;
    align-items: center;
    gap: 4px;
}
.share-bar {
    height: 12px;
    border-radius: 6px;
    position: relative;
    display: flex;
    align-items: center;
}
.bar-total {
    background: #3B82F6;
    height: 100%;
    border-radius: 6px;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    padding-right: 6px;
    min-width: 30px;
}
.bar-top {
    background: #F59E0B;
    height: 100%;
    border-radius: 6px;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    padding-right: 6px;
    min-width: 30px;
}
.bar-value {
    font-size: 0.6rem;
    font-weight: 600;
    color: white;
}
.icon-square {
    display: inline-block;
    width: 8px;
    height: 8px;
    border: 1.5px solid currentColor;
    margin-right: 4px;
}
.icon-circle {
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: currentColor;
    margin-right: 4px;
}
/* Footer */
.footer {
    padding: 12px 28px;
    font-size: 0.7rem;
    color: #6b7280;
    background: #f9fafb;
    border-top: 1px solid #e5e7eb;
}
.footer-note {
    font-style: italic;
    margin-top: 4px;
}
/* Sticky first column */
.sticky-col {
    position: sticky;
    left: 0;
    z-index: 10;
    background: inherit;
}
.data-row .sticky-col {
    background: white;
}
.data-row:nth-child(even) .sticky-col {
    background: #f9fafb;
}
.row-header .sticky-col {
    background: #f9fafb;
}
/* Row label column styling */
.row-label {
    min-width: 160px;
    background: white;
}
.row-header .row-label {
    background: #f9fafb;
}
/* Survey count indicators */
.survey-count {
    font-size: 0.6rem;
    color: #9ca3af;
    margin-top: 2px;
}
```

#### Especificación de estructura HTML

**Header:**
```html
<div class="header">
  <div class="logo">D</div>
  <div class="header-text">
    <h1>HIGHLIGHTS DEL MES — NPS POST VENTA</h1>
    <p>Abr · May · Jun 2026* | Región Brasil</p>
  </div>
</div>
```

**Fila de headers de mes (row 1):**
- Primera celda: sticky-col row-label con `background: #6B21A8` (vacía)
- Una celda por mes con `colspan="2"` + `border-right: 1px solid rgba(255,255,255,0.2)"`
- Contenido: nombre del mes en mayúsculas + contador de encuestas (□ Total · ● Top)
- Última celda: `rowspan="3"`, `background: #6B21A8`, texto "SHARE" + mes de referencia

**Fila sub-headers (row 2):**
- Primera celda: sticky-col row-label con `background: #7E22CE` (vacía)
- Por cada mes: celda "□ Total" + celda "● Top" con `border-right`


**Secciones (section-header):**
- Fila `<tr class="section-header">` con el nombre de sección en la primera celda
- Secciones en orden: MIV — MACHINE PURO → CONTACTO → BACK
- Primera celda: sticky-col row-label con `background: #4C1D95`
**Filas de datos (data-row):**
- Primera celda: sticky-col row-label con `.bucket-name` + `.bucket-subtitle`
- Cada mes: celda `.cell-total` + celda `.cell-top` (con `border-right`)
- NPS: `<div class="nps-value nps-positive|negative|neutral">+XX.X</div>`
- Delta: `<div class="delta delta-up|down|flat">▲|▼ X.X</div>`
- Celda Share: `.cell-share` con barras `.bar-total` (azul) y `.bar-top` (amarillo)


**Nombres de display para buckets:**
- `machine_puro` → "Machine Puro" / subtítulo "sofia + solo_miv"
- `HUMAN` → "Human" / subtítulo "Agente humano"
- `MACHINE` → "Machine" / subtítulo "Bot / máquina"
- `friccion` → "Fricción" / subtítulo "Con fricción"
- `sin_friccion_asistido` → "Sin fric. Asistido" / subtítulo "Sin fricción — asistido"
- `sin_friccion_no_asistido` → "Sin fric. No Asistido" / subtítulo "Sin fricción — no asist."

**Footer:**
```html
<div class="footer">
  Generado por ToqanClaw · Fuente: asci_nps_secuence_metrics · Brasil · Abr–Jun 2026
  <div class="footer-note">*Jun 2026 con datos al 06/07/2026</div>
</div>
```

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
   - ⚠️ NPS ponderado vs real: escala -1 a 1. Ponderado usa SUM(flg_sent) para share y SUM(flg_nps_result)/SUM(flg_answered) para NPS por type. NO usar AVG ni COUNT(DISTINCT survey_id).
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

## Instalación y distribución

Este skill está publicado en GitHub en `lucastossolini-despe/skills`.

### Cómo instalar (primera vez)
Abrir una conversación con ToqanClaw y pegar:


```
Instalá este skill: npx skills add lucastossolini-despe/skills --skill nps-analyst --agent '*' --yes --copy
```

### Cómo actualizar (cuando haya una versión nueva)

```
Actualizá mis skills: npx skills update
```


### Repositorio

https://github.com/lucastossolini-despe/skills

### Requisitos

- Acceso al data lake de Despegar
- Acceso a los Google Docs del Knowledge Base NPS (solicitar acceso a Lucas Tossolini si no los tenés)
