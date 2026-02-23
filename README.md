# sm_finalproject

## 📋 Descripción del Proyecto

Este proyecto implementa un flujo ETL completo que procesa datos del sistema historiador **Exaquantum** de la planta, los transforma aplicando los cálculos de eficiencia termodinámica del reporte de performance, y entrega un dashboard interactivo en Power BI para el seguimiento semanal de KPIs de equipos críticos.

El objetivo es reemplazar el proceso manual de generación de reportes (Excel + Google Colab) por un pipeline automatizado, reproducible y gobernado dentro del ecosistema Azure.

---

## 🏭 Origen de los Datos

### Fuente Principal — Historiador Exaquantum
Los datos provienen del sistema **SCADA/Historian Exaquantum** de la central termoeléctrica. El historiador registra señales de proceso cada **5 minutos** de los equipos principales:

| Equipo | Tags | Variables |
|---|---|---|
| Turbina de Gas TG11 | `G1_DWATT`, `G1_FQG`, `G1_TTXM`, `G1_CTIM`, `G1_CPR`, `G1_CTD` | Potencia (MW), Flujo gas, Temp. exhaust, Compresor |
| Turbina de Gas TG12 | `G2_DWATT`, `G2_FQG`, `G2_TTXM`, `G2_CTIM`, `G2_CPR`, `G2_CTD` | Potencia (MW), Flujo gas, Temp. exhaust, Compresor |
| Turbina de Vapor TV | `S1_DWATT` | Potencia generada (MW) |
| HRSG 11 | `G1_TTXM`, `11TI1870` | Temp. exhaust entrada / stack salida |
| HRSG 12 | `G2_TTXM`, `12TI1870` | Temp. exhaust entrada / stack salida |
| Condensador Box 1 | `10TI6591A`, `10TI6595A`, `10TI3005` | Temperatura entrada / salida / agua fría |
| Condensador Box 2 | `10TI6591B`, `10TI6595B` | Temperatura entrada / salida |
| Auxiliares | `10JI8128`, `10JI8129` | Consumo auxiliar (MW) |
| Ambiente | `00TI8002` | Temperatura ambiente (°F) |

Los archivos son exportados en formato **CSV con separador `;`** y cargados manualmente al contenedor `raw` del Data Lake.

---

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                         AZURE CLOUD                             │
│                                                                 │
│  ┌──────────┐    ┌─────────────────────────────────────────┐   │
│  │ GitHub   │    │        ADLS Gen2 (adlssmartdatajpc)      │   │
│  │  CI/CD   │    │  ┌────────┐ ┌────────┐ ┌────────┐       │   │
│  └────┬─────┘    │  │  raw   │ │ bronze │ │ silver │       │   │
│       │          │  │  CSV   │ │ Delta  │ │ Delta  │       │   │
│       ▼          │  └───┬────┘ └───┬────┘ └───┬────┘       │   │
│  ┌────────────┐  │      │          │           │            │   │
│  │  Unity     │  │  ┌───┴──────────┴───────────┴────────┐  │   │
│  │  Catalog   │  │  │         golden / Delta             │  │   │
│  └────┬───────┘  │  └───────────────────┬────────────────┘  │   │
│       │          └──────────────────────┼───────────────────┘   │
│       │                                 │                        │
│  ┌────┴─────────────────────────────────┴──────┐                │
│  │              Azure Databricks               │                │
│  │                                             │                │
│  │  01_bronze_ingesta.py  ──▶  Bronze Layer    │                │
│  │  02_silver_transform.py ──▶ Silver Layer    │                │
│  │  03_golden_kpis.py     ──▶  Golden Layer    │                │
│  └──────────────────────────────┬──────────────┘                │
│                                 │                                │
│                    ┌────────────▼──────────┐                    │
│                    │   Power BI Dashboard   │                    │
│                    │  KPIs Semanales        │                    │
│                    └───────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🥉🥈🥇 Arquitectura Medallion

### RAW
Archivos CSV originales exportados del historiador Exaquantum. Se almacenan sin modificación en `abfss://raw@adlssmartdatajpc.dfs.core.windows.net/`. La conexión se realiza vía **Managed Identity** sin exponer credenciales.

### 🥉 Bronze — `catalog_termoplanta.bronze.historian_signals`
Ingesta cruda del CSV al Data Lake en formato **Delta**. Las únicas transformaciones son:
- Tipado del campo `HORA` a `timestamp`
- Renombre de columnas que inician con número (`11TI1870` → `TI1870_11`)
- Adición de metadata de trazabilidad: `source_file`, `ingestion_date`

### 🥈 Silver — `catalog_termoplanta.silver.historian_kpis`
Cálculo de todos los KPIs de performance en PySpark:

| KPI | Fórmula | Unidad |
|---|---|---|
| Heat Rate Neto | `((G1_FQG + G2_FQG) × PCI_GN) / (Potencia_neta) × factor` | BTU/kWh |
| Heat Rate Bruto | `((G1_FQG + G2_FQG) × PCI_GN) / (Potencia_bruta) × factor` | BTU/kWh |
| Eficiencia TG11 | `3600 × 100 / (flujo_energético / G1_DWATT)` | % |
| Eficiencia TG12 | `3600 × 100 / (flujo_energético / G2_DWATT)` | % |
| Eficiencia HRSG11 | `(T_exhaust - T_stack) / (T_exhaust - T_ambiente)` | % |
| Eficiencia HRSG12 | `(T_exhaust - T_stack) / (T_exhaust - T_ambiente)` | % |
| Efectividad Box 1 | `(T_entrada - T_salida) / (T_entrada - T_agua_fría)` | % |
| Efectividad Box 2 | `(T_entrada - T_salida) / (T_entrada - T_agua_fría)` | % |
| Eficiencia Compresor TG11 | `(T_salida_isentrópica - T_entrada) / (T_salida_real - T_entrada)` | % |
| Eficiencia Compresor TG12 | `(T_salida_isentrópica - T_entrada) / (T_salida_real - T_entrada)` | % |

Adicionalmente se realiza:
- **Clasificación de condición operativa** por rangos de potencia:
  - `Carga_Base_CC2x1` — ambas TGs > 170 MW + TV > 150 MW
  - `BL_1x1_TG11` — solo TG11 operando en ciclo combinado
  - `BL_1x1_TG12` — solo TG12 operando en ciclo combinado
  - `MT_CC2x1` — carga media ambas TGs (90–140 MW)
- **Detección de outliers** por método IQR por condición operativa y semana

### 🥇 Golden — `catalog_termoplanta.golden`
Tablas agregadas listas para consumo en Power BI:

| Tabla | Granularidad | Descripción |
|---|---|---|
| `kpi_semanal` | Semana × Condición | Media, min, max, std de todos los KPIs |
| `kpi_diario` | Día × Condición | Promedios diarios + energía generada (MWh) |

---

## 📁 Estructura del Repositorio

```
📦 thermal-plant-etl/
│
├── 📂 datasets/                    # Archivos fuente del historiador
│   └── BD_EXAQUAUNTUM.csv
│
├── 📂 PrepAmb/                     # Preparación del ambiente Unity Catalog
│   └── Preparacion_Ambiente.ipynb
│
├── 📂 proceso/                     # Notebooks del ETL
│   ├── 01_bronze_ingesta.ipynb     # RAW → Bronze
│   ├── 02_silver_transform.ipynb   # Bronze → Silver (KPIs)
│   └── 03_golden_kpis.ipynb        # Silver → Golden (agregaciones)
│
├── 📂 reversion/                   # Scripts de rollback
│   └── drop_catalog.py
│
├── 📂 seguridad/                   # GRANTs sobre tablas Golden
│   └── grants.sql
│
├── 📂 dashboard/                   # Dashboard Power BI
│   ├── performance_monitoring.pbix
│   └── preview.png
│
├── 📂 evidencias/                  # Capturas de ejecución
│   └── workflow_success.png
│
├── 📂 certificaciones/             # Certificados Azure
│   └── cert.png
│
├── 📂 .github/
│   └── 📂 workflows/
│       └── deploy.yml              # CI/CD dev → prod
│
└── 📄 README.md
```
