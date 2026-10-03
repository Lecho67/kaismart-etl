# Laboratorio práctico de ETL — Kaismart Solutions S.A.S.

Maestría en Ciencia de Datos e IA · Universidad Autónoma de Occidente (UAO)



Pipeline ETL que extrae las dos fuentes de información de la empresa, las guarda en una arquitectura Medallion y perfila la calidad de los datos. Todo el desarrollo está en el cuaderno [lab_etl_kaismart.ipynb](lab_etl_kaismart.ipynb).

| Fuente | Origen | Registros |
|---|---|---|
| Sistema comercial | Base de datos MySQL `clientes`, tabla `ventas` | 5.000 |
| Sistema logístico | Archivo Excel `kaismart_eventos_logisticos.xlsx` | 50.000 |

## Estado del proyecto

| Parte | Contenido | Estado |
|---|---|---|
| 1 | Extracción desde MySQL | Completa |
| 2 | Extracción desde Excel | Completa |
| 3 | Comprensión inicial de los datasets | Completa |
| 4 | Perfil inicial de calidad del dato | Completa |
| 5 y 6 | — | Pendientes |
| 7 | Limpieza (Silver) e integración (Gold) | Pendiente |
| 8 | Automatización del pipeline con `schedule` | Completa (Silver y Gold quedan como funciones por completar) |

En las Partes 1 a 4 **no se convierten tipos, no se limpia y no se imputa nada**: los datos se guardan y se analizan tal como llegan.

## Arquitectura Medallion

| Capa | Contenido | Carpeta |
|---|---|---|
| **Raw** | Archivo original del sistema logístico | `data/raw/` |
| **Bronze** | Datos tal como se extrajeron de la fuente, sin transformar | `data/bronze/` |
| **Silver** | Datos depurados: nulos tratados, duplicados eliminados, categorías estandarizadas, tipos corregidos | `data/silver/` |
| **Gold** | Ventas + logística integradas por `pedido_id`, listas para análisis | `data/gold/` |

Archivos generados hasta ahora: `data/bronze/ventas_bronze.csv` y `data/bronze/logistica_bronze.csv`.

## Estructura del proyecto

```
kaismart_etl/
├── lab_etl_kaismart.ipynb   # Cuaderno con todas las partes
├── config.yaml              # Fuentes, rutas y autores
├── .env                     # Contraseña de la BD (no se versiona)
├── .env.example             # Plantilla del .env
├── requirements.txt         # Dependencias
└── data/
    ├── raw/                 # kaismart_eventos_logisticos.xlsx
    ├── bronze/              # CSV sin transformar
    ├── silver/              # (Parte 7)
    └── gold/                # (Parte 7)
```

## Requisitos

- Python 3.10 o superior
- Acceso de red al servidor MySQL configurado en `config.yaml`
- Dependencias: `pandas`, `openpyxl`, `mysql-connector-python`, `python-dotenv`, `PyYAML`, `schedule`, `jupyter`, `ipykernel`

## Instalación y ejecución

1. Crear y activar un entorno virtual:

   ```powershell
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   ```

2. Instalar las dependencias:

   ```powershell
   pip install -r requirements.txt
   ```

3. Crear el archivo `.env` a partir de la plantilla y poner la contraseña de la base de datos:

   ```powershell
   Copy-Item .env.example .env
   ```

   ```
   DB_PASSWORD=<contraseña>
   ```

4. Verificar que el Excel esté en `data/raw/kaismart_eventos_logisticos.xlsx`.

5. Abrir el cuaderno **desde la raíz del proyecto** (el cuaderno toma la carpeta actual como raíz y exige encontrar `config.yaml` allí) y ejecutar las celdas en orden:

   ```powershell
   jupyter notebook lab_etl_kaismart.ipynb
   ```

## Configuración

[config.yaml](config.yaml) define los parámetros de las fuentes (host, base de datos, tabla, hoja del Excel, registros esperados) y las rutas de cada capa, todas relativas a la raíz del proyecto. La contraseña **nunca** se escribe en el código ni en el YAML: se lee de la variable `DB_PASSWORD` del archivo `.env`, que está en `.gitignore`.

## Automatización (Parte 8)

El orquestador usa la librería `schedule` y agrupa el proceso en funciones:

| Función | Qué hace |
|---|---|
| `extraer_ventas()` | MySQL → `data/bronze/ventas_bronze.csv` |
| `extraer_logistica()` | Excel → `data/bronze/logistica_bronze.csv` |
| `transformar_silver()` | Definida, se completa en la Parte 7 |
| `integrar_gold()` | Definida, se completa en la Parte 7 |
| `ejecutar_pipeline()` | Ejecuta los pasos en orden; si uno falla, no corre los siguientes y el error se informa sin detener el orquestador |

Los archivos de cada capa se sobrescriben en cada ejecución, así que repetir el pipeline no duplica registros. Para ejecutarlo todos los días a las 6:00 a. m. en un script independiente:

```python
schedule.every().day.at("06:00").do(ejecutar_pipeline)

while True:
    schedule.run_pending()
    time.sleep(60)
```

En el cuaderno solo se hace una demostración: se programa cada minuto y se detiene tras dos ejecuciones.

## Principales hallazgos de calidad

**`df_ventas`** (5.000 filas, 17 variables, 1 ene – 30 jun 2026)
- Montos coherentes (bruto, descuento y neto), sin valores en cero ni negativos, pero guardados como `Decimal` (`object`).
- Nulos normales: `id_tienda` (70 %, solo aplica a tienda física) y `calificacion_cliente` (43 %, es opcional).
- Sin duplicados.

**`df_logistica`** (50.000 filas, 14 variables, 1 ene – 3 jul 2026)
- 100 eventos duplicados (99 copias exactas y 1 con `centro_logistico` faltante en una copia); 98 pedidos con etapas faltantes y 100 `evento_id` ausentes, cifras que coinciden.
- Nulos que son problema: `centro_logistico` (90), `fecha_evento` (65), `fecha_prometida_entrega` (55), `ciudad_destino` (45), y 80 `transportadora` / 60 `numero_guia` ausentes en eventos posteriores al despacho.
- Las fechas están como texto y existe una columna sobrante `Unnamed: 13` del Excel.
- Una misma guía aparece en dos pedidos distintos.

Ningún problema se corrigió en esta entrega: la limpieza corresponde a la capa Silver (Parte 7).
