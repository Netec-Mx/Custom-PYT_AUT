# Práctica 6 — Generación Automática de Reportes

| Campo | Detalle |
|-------|---------|
| **Duración** | 38 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio construirás un pipeline completo de generación automática de reportes a partir de datos de ventas en formato CSV. Partiendo de un archivo con ~500 registros ficticios, realizarás la carga, limpieza, validación, agregación y exportación de los datos a formatos Excel (.xlsx) y HTML usando pandas, openpyxl y Jinja2. Los archivos generados serán la entrada del Lab 07.

## Objetivos de Aprendizaje

- [ ] Leer, validar y limpiar un dataset CSV usando pandas, identificando valores nulos, duplicados y tipos incorrectos
- [ ] Generar reportes tabulares agregados con `groupby()` y `pivot_table()` calculando KPIs de negocio
- [ ] Exportar reportes a Excel (.xlsx) con formato aplicado (filtros automáticos, anchos de columna) usando openpyxl
- [ ] Renderizar un reporte HTML dinámico con plantillas Jinja2 alimentadas por los datos procesados

## Prerrequisitos

### Conocimiento requerido

- Estructuras de datos Python (listas, diccionarios, comprensiones)
- Lectura y escritura de archivos CSV con el módulo estándar `csv` y `pd.read_csv()`
- Conceptos básicos de DataFrames en pandas (columnas, índices, tipos)

### Acceso y software

| Software | Versión |
|----------|---------|
| Python | 3.11.9 |
| pip | 24.0 |
| pandas | 2.2.2 |
| openpyxl | 3.1.2 |
| Jinja2 | 3.1.4 |
| VS Code (recomendado) | 1.89.1 |

## Entorno del Laboratorio

### Estructura de directorios objetivo

```
~/proyectos/auto_reporter/
├── data/
│   ├── input/
│   │   └── ventas_2024.csv
│   └── output/
│       ├── ventas_limpias.csv
│       ├── reporte_ventas.xlsx
│       └── reporte_ventas.html
├── templates/
│   └── reporte.html.j2
├── src/
│   └── generar_reporte.py
├── requirements.txt
└── .venv/
```

### Comandos de configuración inicial

```bash
# Crear directorio raíz del proyecto
mkdir -p ~/proyectos/auto_reporter
cd ~/proyectos/auto_reporter

# Crear estructura de directorios
mkdir -p data/input data/output templates src

# Crear y activar entorno virtual
python3.11 -m venv .venv
source .venv/bin/activate    # macOS/Linux
# .venv\Scripts\activate     # Windows

# Verificar versión de Python
python --version
# Esperado: Python 3.11.9

# Crear requirements.txt
cat > requirements.txt << 'EOF'
pandas==2.2.2
openpyxl==3.1.2
Jinja2==3.1.4
EOF

# Instalar dependencias
pip install -r requirements.txt
```

---

## Paso a Paso

### Paso 1 — Generar el dataset de ventas ficticias

**Objetivo:** Crear el archivo `ventas_2024.csv` con ~500 registros que incluyan datos sucios (nulos, duplicados, tipos incorrectos) para simular un escenario real.

**Instrucciones:**

1. Crea el archivo `data/input/generar_datos.py`:

```python
"""Genera un CSV de ventas ficticias con datos sucios para el laboratorio."""
import csv
import random
from datetime import date, timedelta
from pathlib import Path

random.seed(42)

PRODUCTOS = {
    "Laptop Pro": ("Electrónica", 1200.00),
    "Monitor 27''": ("Electrónica", 450.00),
    "Teclado Mecánico": ("Periféricos", 89.99),
    "Mouse Ergonómico": ("Periféricos", 55.00),
    "Webcam HD": ("Periféricos", 120.00),
    "Silla Gamer": ("Mobiliario", 380.00),
    "Escritorio Ajustable": ("Mobiliario", 620.00),
    "Audífonos BT": ("Audio", 199.99),
    "Parlante Portátil": ("Audio", 75.00),
    "Hub USB-C": ("Accesorios", 42.00),
}

VENDEDORES = ["Ana García", "Carlos López", "María Torres", "Juan Pérez", "Laura Díaz"]
REGIONES = ["Norte", "Sur", "Centro", "Este", "Oeste"]

output_path = Path(__file__).parent / "ventas_2024.csv"

with open(output_path, "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["fecha", "producto", "categoria", "cantidad", "precio_unitario", "vendedor", "region"])

    inicio = date(2024, 1, 1)
    filas = []

    for i in range(500):
        dia = inicio + timedelta(days=random.randint(0, 364))
        producto = random.choice(list(PRODUCTOS.keys()))
        categoria, precio = PRODUCTOS[producto]
        cantidad = random.randint(1, 20)
        vendedor = random.choice(VENDEDORES)
        region = random.choice(REGIONES)

        # Introducir datos sucios intencionalmente
        fecha_str = dia.isoformat()
        precio_str = f"{precio:.2f}"
        cantidad_str = str(cantidad)

        # ~3% de filas con fecha mal formateada
        if random.random() < 0.03:
            fecha_str = dia.strftime("%d/%m/%Y")

        # ~4% de filas con precio nulo
        if random.random() < 0.04:
            precio_str = ""

        # ~3% de filas con cantidad negativa
        if random.random() < 0.03:
            cantidad_str = str(-cantidad)

        # ~2% de filas con vendedor vacío
        if random.random() < 0.02:
            vendedor = ""

        filas.append([fecha_str, producto, categoria, cantidad_str, precio_str, vendedor, region])

    # Agregar ~10 duplicados exactos
    for _ in range(10):
        filas.append(random.choice(filas[:490]))

    random.shuffle(filas)
    writer.writerows(filas)

print(f"Archivo generado: {output_path} ({len(filas)} filas)")
```

2. Ejecuta el script generador:

```bash
cd ~/proyectos/auto_reporter
python data/input/generar_datos.py
```

**Salida esperada:**

```
Archivo generado: /home/<usuario>/proyectos/auto_reporter/data/input/ventas_2024.csv (510 filas)
```

**Verificación:**

```bash
head -5 data/input/ventas_2024.csv
wc -l data/input/ventas_2024.csv
```

Debes ver la fila de encabezado seguida de datos, y un total de ~511 líneas (encabezado + 510 filas de datos).

---

### Paso 2 — Carga y exploración inicial del dataset

**Objetivo:** Cargar el CSV con pandas y realizar una exploración rápida para identificar problemas de calidad de datos.

**Instrucciones:**

1. Crea el archivo `src/generar_reporte.py` con la sección de carga:

```python
"""Pipeline de generación automática de reportes de ventas."""
from pathlib import Path

import pandas as pd

# === CONFIGURACIÓN DE RUTAS ===
BASE_DIR = Path(__file__).resolve().parent.parent
INPUT_DIR = BASE_DIR / "data" / "input"
OUTPUT_DIR = BASE_DIR / "data" / "output"
TEMPLATES_DIR = BASE_DIR / "templates"

# === PASO 1: CARGA Y EXPLORACIÓN ===
def cargar_datos(ruta_csv: Path) -> pd.DataFrame:
    """Carga el archivo CSV de ventas y muestra información exploratoria."""
    print(f"Cargando datos desde: {ruta_csv}")
    df = pd.read_csv(ruta_csv, encoding="utf-8")

    print(f"\n{'='*60}")
    print("EXPLORACIÓN INICIAL DEL DATASET")
    print(f"{'='*60}")
    print(f"Dimensiones: {df.shape[0]} filas × {df.shape[1]} columnas")
    print(f"\nColumnas: {list(df.columns)}")
    print(f"\nPrimeras 5 filas:")
    print(df.head().to_string(index=False))
    print(f"\nInformación de tipos:")
    print(df.dtypes.to_string())
    print(f"\nValores nulos por columna:")
    print(df.isnull().sum().to_string())
    print(f"\nDuplicados exactos: {df.duplicated().sum()}")

    return df


if __name__ == "__main__":
    ruta_ventas = INPUT_DIR / "ventas_2024.csv"
    df_raw = cargar_datos(ruta_ventas)
```

2. Ejecuta para verificar la exploración:

```bash
cd ~/proyectos/auto_reporter
python src/generar_reporte.py
```

**Salida esperada (aproximada):**

```
Cargando datos desde: .../data/input/ventas_2024.csv

============================================================
EXPLORACIÓN INICIAL DEL DATASET
============================================================
Dimensiones: 510 filas × 7 columnas

Columnas: ['fecha', 'producto', 'categoria', 'cantidad', 'precio_unitario', 'vendedor', 'region']

Primeras 5 filas:
       fecha           producto    categoria cantidad precio_unitario     vendedor  region
  2024-03-15       Laptop Pro   Electrónica       12         1200.00  Ana García   Norte
...

Información de tipos:
fecha              object
producto           object
categoria          object
cantidad           object
precio_unitario    object
vendedor           object
region             object

Valores nulos por columna:
fecha              0
producto           0
categoria          0
cantidad           0
precio_unitario    ~20
vendedor           ~10
region             0

Duplicados exactos: ~10
```

**Verificación:** Confirma que `precio_unitario` y `vendedor` muestran valores nulos, y que existen duplicados detectados. Los tipos son todos `object` porque pandas no pudo inferir correctamente los numéricos (hay valores vacíos y formatos mixtos).

---

### Paso 3 — Limpieza y validación de datos

**Objetivo:** Eliminar duplicados, corregir tipos de datos, manejar nulos y validar reglas de negocio.

**Instrucciones:**

1. Agrega la función de limpieza en `src/generar_reporte.py` (después de `cargar_datos`):

```python
def limpiar_datos(df: pd.DataFrame) -> pd.DataFrame:
    """Limpia y valida el DataFrame de ventas."""
    print(f"\n{'='*60}")
    print("LIMPIEZA DE DATOS")
    print(f"{'='*60}")

    filas_iniciales = len(df)

    # 1. Eliminar duplicados exactos
    df = df.drop_duplicates()
    duplicados_eliminados = filas_iniciales - len(df)
    print(f"  Duplicados eliminados: {duplicados_eliminados}")

    # 2. Normalizar columna 'fecha' a formato datetime
    #    Manejar formatos mixtos: ISO (2024-01-15) y europeo (15/01/2024)
    df["fecha"] = pd.to_datetime(df["fecha"], format="mixed", dayfirst=True)

    # 3. Convertir 'precio_unitario' a numérico (coerce convierte errores a NaN)
    df["precio_unitario"] = pd.to_numeric(df["precio_unitario"], errors="coerce")

    # 4. Convertir 'cantidad' a numérico
    df["cantidad"] = pd.to_numeric(df["cantidad"], errors="coerce")

    # 5. Rellenar precios nulos con la mediana del producto correspondiente
    nulos_precio_antes = df["precio_unitario"].isnull().sum()
    df["precio_unitario"] = df.groupby("producto")["precio_unitario"].transform(
        lambda x: x.fillna(x.median())
    )
    print(f"  Precios nulos rellenados (mediana por producto): {nulos_precio_antes}")

    # 6. Rellenar vendedor vacío con "Sin asignar"
    nulos_vendedor = df["vendedor"].isnull().sum() + (df["vendedor"] == "").sum()
    df["vendedor"] = df["vendedor"].replace("", pd.NA)
    df["vendedor"] = df["vendedor"].fillna("Sin asignar")
    print(f"  Vendedores vacíos rellenados: {nulos_vendedor}")

    # 7. Validar reglas de negocio
    # Cantidades deben ser > 0
    filas_cantidad_invalida = (df["cantidad"] <= 0).sum()
    df = df[df["cantidad"] > 0]
    print(f"  Filas con cantidad <= 0 eliminadas: {filas_cantidad_invalida}")

    # Precios deben ser > 0
    filas_precio_invalido = (df["precio_unitario"] <= 0).sum()
    df = df[df["precio_unitario"] > 0]
    print(f"  Filas con precio <= 0 eliminadas: {filas_precio_invalido}")

    # Fechas deben estar en 2024
    fecha_inicio = pd.Timestamp("2024-01-01")
    fecha_fin = pd.Timestamp("2024-12-31")
    fuera_rango = ((df["fecha"] < fecha_inicio) | (df["fecha"] > fecha_fin)).sum()
    df = df[(df["fecha"] >= fecha_inicio) & (df["fecha"] <= fecha_fin)]
    print(f"  Filas con fecha fuera de 2024 eliminadas: {fuera_rango}")

    # 8. Resetear índice
    df = df.reset_index(drop=True)

    # 9. Agregar columna calculada: total_venta
    df["total_venta"] = df["cantidad"] * df["precio_unitario"]

    print(f"\n  Resultado final: {len(df)} filas limpias (de {filas_iniciales} originales)")
    print(f"  Tipos de datos finales:")
    print(f"    fecha: {df['fecha'].dtype}")
    print(f"    cantidad: {df['cantidad'].dtype}")
    print(f"    precio_unitario: {df['precio_unitario'].dtype}")
    print(f"    total_venta: {df['total_venta'].dtype}")

    return df
```

2. Actualiza el bloque `__main__`:

```python
if __name__ == "__main__":
    ruta_ventas = INPUT_DIR / "ventas_2024.csv"
    df_raw = cargar_datos(ruta_ventas)
    df_limpio = limpiar_datos(df_raw)

    # Guardar dataset limpio
    ruta_limpio = OUTPUT_DIR / "ventas_limpias.csv"
    df_limpio.to_csv(ruta_limpio, index=False, encoding="utf-8")
    print(f"\n  Dataset limpio guardado en: {ruta_limpio}")
```

3. Ejecuta el script:

```bash
python src/generar_reporte.py
```

**Salida esperada:**

```
============================================================
LIMPIEZA DE DATOS
============================================================
  Duplicados eliminados: 10
  Precios nulos rellenados (mediana por producto): ~20
  Vendedores vacíos rellenados: ~10
  Filas con cantidad <= 0 eliminadas: ~15
  Filas con precio <= 0 eliminadas: 0
  Filas con fecha fuera de 2024 eliminadas: 0

  Resultado final: ~485 filas limpias (de 510 originales)
  Tipos de datos finales:
    fecha: datetime64[ns]
    cantidad: int64
    precio_unitario: float64
    total_venta: float64

  Dataset limpio guardado en: .../data/output/ventas_limpias.csv
```

**Verificación:**

```bash
head -3 data/output/ventas_limpias.csv
python -c "import pandas as pd; df=pd.read_csv('data/output/ventas_limpias.csv'); print(df.dtypes); print(df.isnull().sum())"
```

Confirma que no hay valores nulos y que los tipos numéricos son correctos.

---

### Paso 4 — Generación de reporte agregado con KPIs

**Objetivo:** Calcular métricas de negocio usando `groupby()` y `pivot_table()` para generar un resumen ejecutivo.

**Instrucciones:**

1. Agrega la función de agregación en `src/generar_reporte.py`:

```python
def generar_kpis(df: pd.DataFrame) -> dict:
    """Calcula KPIs de negocio a partir del DataFrame limpio."""
    print(f"\n{'='*60}")
    print("GENERACIÓN DE KPIs")
    print(f"{'='*60}")

    kpis = {}

    # KPI 1: Total de ventas
    kpis["total_ventas"] = df["total_venta"].sum()
    print(f"  Total ventas: ${kpis['total_ventas']:,.2f}")

    # KPI 2: Número de transacciones
    kpis["num_transacciones"] = len(df)
    print(f"  Transacciones: {kpis['num_transacciones']}")

    # KPI 3: Ticket promedio
    kpis["ticket_promedio"] = df["total_venta"].mean()
    print(f"  Ticket promedio: ${kpis['ticket_promedio']:,.2f}")

    # KPI 4: Top 5 productos por ingreso total
    top_productos = (
        df.groupby("producto")["total_venta"]
        .sum()
        .sort_values(ascending=False)
        .head(5)
        .reset_index()
    )
    top_productos.columns = ["Producto", "Ingreso Total"]
    kpis["top_5_productos"] = top_productos
    print(f"\n  Top 5 productos:")
    print(top_productos.to_string(index=False))

    # KPI 5: Ventas por región
    ventas_region = (
        df.groupby("region")["total_venta"]
        .agg(["sum", "count", "mean"])
        .round(2)
        .reset_index()
    )
    ventas_region.columns = ["Región", "Total", "Transacciones", "Promedio"]
    ventas_region = ventas_region.sort_values("Total", ascending=False)
    kpis["ventas_por_region"] = ventas_region
    print(f"\n  Ventas por región:")
    print(ventas_region.to_string(index=False))

    # KPI 6: Pivot table — ventas por categoría y mes
    df_pivot = df.copy()
    df_pivot["mes"] = df_pivot["fecha"].dt.month_name(locale="es_ES.UTF-8") if hasattr(df_pivot["fecha"].dt, "month_name") else df_pivot["fecha"].dt.strftime("%B")
    df_pivot["mes_num"] = df_pivot["fecha"].dt.month

    pivot = pd.pivot_table(
        df_pivot,
        values="total_venta",
        index="categoria",
        columns="mes_num",
        aggfunc="sum",
        fill_value=0,
    ).round(2)
    kpis["pivot_categoria_mes"] = pivot
    print(f"\n  Pivot categoría × mes (primeras 4 columnas):")
    print(pivot.iloc[:, :4].to_string())

    return kpis
```

2. Actualiza el bloque `__main__`:

```python
if __name__ == "__main__":
    ruta_ventas = INPUT_DIR / "ventas_2024.csv"
    df_raw = cargar_datos(ruta_ventas)
    df_limpio = limpiar_datos(df_raw)

    # Guardar dataset limpio
    ruta_limpio = OUTPUT_DIR / "ventas_limpias.csv"
    df_limpio.to_csv(ruta_limpio, index=False, encoding="utf-8")
    print(f"\n  Dataset limpio guardado en: {ruta_limpio}")

    # Generar KPIs
    kpis = generar_kpis(df_limpio)
```

3. Ejecuta:

```bash
python src/generar_reporte.py
```

**Salida esperada:** Verás los KPIs impresos en consola con totales, top 5 productos y tabla por región.

**Verificación:** Confirma que `total_ventas` es un valor positivo razonable (debería estar entre $500,000 y $3,000,000 dado el rango de precios y cantidades) y que las 5 regiones aparecen en el desglose.

---

### Paso 5 — Exportación a Excel con formato aplicado

**Objetivo:** Generar un archivo `.xlsx` con múltiples hojas, filtros automáticos y anchos de columna ajustados.

**Instrucciones:**

1. Agrega la función de exportación Excel en `src/generar_reporte.py`:

```python
from openpyxl.utils import get_column_letter


def exportar_excel(df: pd.DataFrame, kpis: dict, ruta_salida: Path) -> None:
    """Exporta el reporte a un archivo Excel con formato profesional."""
    print(f"\n{'='*60}")
    print("EXPORTACIÓN A EXCEL")
    print(f"{'='*60}")

    with pd.ExcelWriter(ruta_salida, engine="openpyxl") as writer:
        # Hoja 1: Resumen de KPIs
        resumen_data = {
            "Métrica": ["Total Ventas", "Transacciones", "Ticket Promedio"],
            "Valor": [
                f"${kpis['total_ventas']:,.2f}",
                str(kpis['num_transacciones']),
                f"${kpis['ticket_promedio']:,.2f}",
            ],
        }
        df_resumen = pd.DataFrame(resumen_data)
        df_resumen.to_excel(writer, sheet_name="Resumen", index=False, startrow=1)

        # Hoja 2: Top 5 Productos
        kpis["top_5_productos"].to_excel(
            writer, sheet_name="Top Productos", index=False, startrow=1
        )

        # Hoja 3: Ventas por Región
        kpis["ventas_por_region"].to_excel(
            writer, sheet_name="Por Región", index=False, startrow=1
        )

        # Hoja 4: Detalle completo
        df_export = df.copy()
        df_export["fecha"] = df_export["fecha"].dt.strftime("%Y-%m-%d")
        df_export.to_excel(
            writer, sheet_name="Detalle", index=False, startrow=0
        )

        # Hoja 5: Pivot categoría × mes
        kpis["pivot_categoria_mes"].to_excel(
            writer, sheet_name="Pivot Categoría-Mes", startrow=1
        )

        # === APLICAR FORMATO ===
        workbook = writer.book

        for sheet_name in workbook.sheetnames:
            ws = workbook[sheet_name]

            # Auto-filtro en la fila de encabezado
            if ws.max_row > 1:
                ws.auto_filter.ref = ws.dimensions

            # Ajustar ancho de columnas
            for col_idx in range(1, ws.max_column + 1):
                col_letter = get_column_letter(col_idx)
                max_length = 0
                for row in ws.iter_rows(
                    min_col=col_idx, max_col=col_idx, values_only=True
                ):
                    for cell_value in row:
                        if cell_value is not None:
                            max_length = max(max_length, len(str(cell_value)))
                adjusted_width = min(max_length + 2, 40)
                ws.column_dimensions[col_letter].width = adjusted_width

    print(f"  Archivo Excel generado: {ruta_salida}")
    print(f"  Hojas creadas: {list(workbook.sheetnames)}")
```

2. Agrega al final del import (parte superior del archivo):

```python
from openpyxl.utils import get_column_letter
```

3. Actualiza `__main__`:

```python
if __name__ == "__main__":
    ruta_ventas = INPUT_DIR / "ventas_2024.csv"
    df_raw = cargar_datos(ruta_ventas)
    df_limpio = limpiar_datos(df_raw)

    ruta_limpio = OUTPUT_DIR / "ventas_limpias.csv"
    df_limpio.to_csv(ruta_limpio, index=False, encoding="utf-8")
    print(f"\n  Dataset limpio guardado en: {ruta_limpio}")

    kpis = generar_kpis(df_limpio)

    # Exportar a Excel
    ruta_excel = OUTPUT_DIR / "reporte_ventas.xlsx"
    exportar_excel(df_limpio, kpis, ruta_excel)
```

4. Ejecuta:

```bash
python src/generar_reporte.py
```

**Salida esperada:**

```
============================================================
EXPORTACIÓN A EXCEL
============================================================
  Archivo Excel generado: .../data/output/reporte_ventas.xlsx
  Hojas creadas: ['Resumen', 'Top Productos', 'Por Región', 'Detalle', 'Pivot Categoría-Mes']
```

**Verificación:**

```bash
ls -lh data/output/reporte_ventas.xlsx
python -c "
import openpyxl
wb = openpyxl.load_workbook('data/output/reporte_ventas.xlsx')
print('Hojas:', wb.sheetnames)
ws = wb['Detalle']
print(f'Detalle: {ws.max_row} filas × {ws.max_column} columnas')
print(f'Auto-filtro: {ws.auto_filter.ref}')
"
```

Confirma que el archivo existe, tiene 5 hojas y que la hoja "Detalle" tiene auto-filtro aplicado.

---

### Paso 6 — Generación de reporte HTML con Jinja2

**Objetivo:** Crear una plantilla Jinja2 y renderizar un reporte HTML dinámico con los KPIs y datos procesados.

**Instrucciones:**

1. Crea la plantilla `templates/reporte.html.j2`:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Reporte de Ventas 2024</title>
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .container { max-width: 1200px; margin: 0 auto; background: white; padding: 30px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        h1 { color: #2c3e50; border-bottom: 3px solid #3498db; padding-bottom: 10px; }
        h2 { color: #34495e; margin-top: 30px; }
        .kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; margin: 20px 0; }
        .kpi-card { background: #ecf0f1; padding: 20px; border-radius: 8px; text-align: center; }
        .kpi-card .valor { font-size: 1.8em; font-weight: bold; color: #2980b9; }
        .kpi-card .etiqueta { color: #7f8c8d; margin-top: 5px; }
        table { border-collapse: collapse; width: 100%; margin: 15px 0; }
        th { background: #3498db; color: white; padding: 12px 8px; text-align: left; }
        td { padding: 10px 8px; border-bottom: 1px solid #ecf0f1; }
        tr:hover { background: #f8f9fa; }
        .footer { margin-top: 40px; padding-top: 20px; border-top: 1px solid #ecf0f1; color: #95a5a6; font-size: 0.9em; }
    </style>
</head>
<body>
    <div class="container">
        <h1>📊 Reporte de Ventas 2024</h1>
        <p>Generado automáticamente el {{ fecha_generacion }}</p>

        <h2>Indicadores Clave (KPIs)</h2>
        <div class="kpi-grid">
            <div class="kpi-card">
                <div class="valor">${{ total_ventas }}</div>
                <div class="etiqueta">Total Ventas</div>
            </div>
            <div class="kpi-card">
                <div class="valor">{{ num_transacciones }}</div>
                <div class="etiqueta">Transacciones</div>
            </div>
            <div class="kpi-card">
                <div class="valor">${{ ticket_promedio }}</div>
                <div class="etiqueta">Ticket Promedio</div>
            </div>
        </div>

        <h2>Top 5 Productos por Ingreso</h2>
        <table>
            <thead>
                <tr>
                    <th>#</th>
                    <th>Producto</th>
                    <th>Ingreso Total</th>
                </tr>
            </thead>
            <tbody>
                {% for producto in top_productos %}
                <tr>
                    <td>{{ loop.index }}</td>
                    <td>{{ producto.nombre }}</td>
                    <td>${{ producto.ingreso }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>

        <h2>Ventas por Región</h2>
        <table>
            <thead>
                <tr>
                    <th>Región</th>
                    <th>Total</th>
                    <th>Transacciones</th>
                    <th>Promedio</th>
                </tr>
            </thead>
            <tbody>
                {% for region in ventas_region %}
                <tr>
                    <td>{{ region.nombre }}</td>
                    <td>${{ region.total }}</td>
                    <td>{{ region.transacciones }}</td>
                    <td>${{ region.promedio }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>

        <div class="footer">
            <p>Reporte generado por auto_reporter | Datos procesados: {{ num_filas_procesadas }} registros</p>
        </div>
    </div>
</body>
</html>
```

2. Agrega la función de renderizado HTML en `src/generar_reporte.py`:

```python
from datetime import datetime

from jinja2 import Environment, FileSystemLoader


def generar_html(df: pd.DataFrame, kpis: dict, ruta_salida: Path) -> None:
    """Renderiza el reporte HTML usando la plantilla Jinja2."""
    print(f"\n{'='*60}")
    print("GENERACIÓN DE REPORTE HTML")
    print(f"{'='*60}")

    # Configurar Jinja2
    env = Environment(
        loader=FileSystemLoader(str(TEMPLATES_DIR)),
        autoescape=True,
    )
    template = env.get_template("reporte.html.j2")

    # Preparar datos para la plantilla
    top_productos = [
        {"nombre": row["Producto"], "ingreso": f"{row['Ingreso Total']:,.2f}"}
        for _, row in kpis["top_5_productos"].iterrows()
    ]

    ventas_region = [
        {
            "nombre": row["Región"],
            "total": f"{row['Total']:,.2f}",
            "transacciones": int(row["Transacciones"]),
            "promedio": f"{row['Promedio']:,.2f}",
        }
        for _, row in kpis["ventas_por_region"].iterrows()
    ]

    # Renderizar
    html_content = template.render(
        fecha_generacion=datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        total_ventas=f"{kpis['total_ventas']:,.2f}",
        num_transacciones=kpis["num_transacciones"],
        ticket_promedio=f"{kpis['ticket_promedio']:,.2f}",
        top_productos=top_productos,
        ventas_region=ventas_region,
        num_filas_procesadas=len(df),
    )

    # Escribir archivo
    ruta_salida.write_text(html_content, encoding="utf-8")
    print(f"  Reporte HTML generado: {ruta_salida}")
    print(f"  Tamaño: {ruta_salida.stat().st_size / 1024:.1f} KB")
```

3. Actualiza los imports al inicio del archivo y el bloque `__main__` final:

```python
"""Pipeline de generación automática de reportes de ventas."""
from datetime import datetime
from pathlib import Path

import pandas as pd
from jinja2 import Environment, FileSystemLoader
from openpyxl.utils import get_column_letter

# ... (mantener CONFIGURACIÓN DE RUTAS y todas las funciones) ...

if __name__ == "__main__":
    ruta_ventas = INPUT_DIR / "ventas_2024.csv"

    # Carga
    df_raw = cargar_datos(ruta_ventas)

    # Limpieza
    df_limpio = limpiar_datos(df_raw)

    # Guardar dataset limpio
    ruta_limpio = OUTPUT_DIR / "ventas_limpias.csv"
    df_limpio.to_csv(ruta_limpio, index=False, encoding="utf-8")
    print(f"\n  Dataset limpio guardado en: {ruta_limpio}")

    # KPIs
    kpis = generar_kpis(df_limpio)

    # Excel
    ruta_excel = OUTPUT_DIR / "reporte_ventas.xlsx"
    exportar_excel(df_limpio, kpis, ruta_excel)

    # HTML
    ruta_html = OUTPUT_DIR / "reporte_ventas.html"
    generar_html(df_limpio, kpis, ruta_html)

    print(f"\n{'='*60}")
    print("✅ PIPELINE COMPLETADO EXITOSAMENTE")
    print(f"{'='*60}")
    print(f"  Archivos generados:")
    print(f"    - {ruta_limpio}")
    print(f"    - {ruta_excel}")
    print(f"    - {ruta_html}")
```

4. Ejecuta el pipeline completo:

```bash
python src/generar_reporte.py
```

**Salida esperada:**

```
============================================================
GENERACIÓN DE REPORTE HTML
============================================================
  Reporte HTML generado: .../data/output/reporte_ventas.html
  Tamaño: ~4.2 KB

============================================================
✅ PIPELINE COMPLETADO EXITOSAMENTE
============================================================
  Archivos generados:
    - .../data/output/ventas_limpias.csv
    - .../data/output/reporte_ventas.xlsx
    - .../data/output/reporte_ventas.html
```

**Verificación:**

```bash
# Verificar que el HTML es válido y contiene datos
python -c "
from pathlib import Path
html = Path('data/output/reporte_ventas.html').read_text(encoding='utf-8')
assert '<!DOCTYPE html>' in html
assert 'Total Ventas' in html
assert 'Top 5 Productos' in html
assert 'Ventas por Región' in html
print('✅ HTML válido con todas las secciones esperadas')
print(f'   Longitud: {len(html)} caracteres')
"
```

Opcionalmente, abre el archivo en un navegador:

```bash
# macOS
open data/output/reporte_ventas.html
# Linux
xdg-open data/output/reporte_ventas.html
# Windows
start data/output/reporte_ventas.html
```

---

## Validación y Pruebas

Ejecuta el siguiente script de validación integral para confirmar que todos los artefactos se generaron correctamente:

```bash
python -c "
from pathlib import Path
import pandas as pd
import openpyxl

base = Path('data/output')

# 1. Verificar existencia de archivos
archivos = ['ventas_limpias.csv', 'reporte_ventas.xlsx', 'reporte_ventas.html']
for archivo in archivos:
    ruta = base / archivo
    assert ruta.exists(), f'❌ No encontrado: {ruta}'
    print(f'✅ {archivo} ({ruta.stat().st_size / 1024:.1f} KB)')

# 2. Validar CSV limpio
df = pd.read_csv(base / 'ventas_limpias.csv')
assert df.isnull().sum().sum() == 0, '❌ Hay valores nulos en el CSV limpio'
assert (df['cantidad'] > 0).all(), '❌ Hay cantidades <= 0'
assert (df['precio_unitario'] > 0).all(), '❌ Hay precios <= 0'
assert 'total_venta' in df.columns, '❌ Falta columna total_venta'
assert len(df) >= 450, f'❌ Muy pocas filas: {len(df)}'
print(f'✅ CSV limpio válido: {len(df)} filas, sin nulos, reglas OK')

# 3. Validar Excel
wb = openpyxl.load_workbook(base / 'reporte_ventas.xlsx')
hojas_esperadas = {'Resumen', 'Top Productos', 'Por Región', 'Detalle', 'Pivot Categoría-Mes'}
assert set(wb.sheetnames) == hojas_esperadas, f'❌ Hojas incorrectas: {wb.sheetnames}'
ws_detalle = wb['Detalle']
assert ws_detalle.auto_filter.ref is not None, '❌ Sin auto-filtro en Detalle'
print(f'✅ Excel válido: {len(wb.sheetnames)} hojas, auto-filtro activo')

# 4. Validar HTML
html = (base / 'reporte_ventas.html').read_text(encoding='utf-8')
assert 'kpi-card' in html, '❌ Faltan tarjetas KPI en HTML'
assert html.count('<tr>') >= 7, '❌ Pocas filas en tablas HTML'
print(f'✅ HTML válido: contiene KPIs y tablas')

print('\n🎉 TODAS LAS VALIDACIONES PASARON EXITOSAMENTE')
"
```

**Resultado esperado:**

```
✅ ventas_limpias.csv (X.X KB)
✅ reporte_ventas.xlsx (X.X KB)
✅ reporte_ventas.html (X.X KB)
✅ CSV limpio válido: ~485 filas, sin nulos, reglas OK
✅ Excel válido: 5 hojas, auto-filtro activo
✅ HTML válido: contiene KPIs y tablas

🎉 TODAS LAS VALIDACIONES PASARON EXITOSAMENTE
```

---

## Solución de Problemas

### Problema 1: `locale.Error` al usar `month_name(locale="es_ES.UTF-8")`

**Síntomas:** Error `locale.Error: unsupported locale setting` o nombres de mes en inglés en lugar de español.

**Causa:** El locale `es_ES.UTF-8` no está instalado en el sistema operativo. Esto es común en contenedores Docker y en sistemas con configuración mínima.

**Solución:** Reemplaza el uso de `month_name(locale=...)` por `strftime("%B")` o usa un mapeo manual de números a nombres de mes. En la función `generar_kpis`, la columna `mes_num` (numérica) es suficiente para el pivot table. Si necesitas nombres en español para el HTML:

```python
MESES_ES = {
    1: "Enero", 2: "Febrero", 3: "Marzo", 4: "Abril",
    5: "Mayo", 6: "Junio", 7: "Julio", 8: "Agosto",
    9: "Septiembre", 10: "Octubre", 11: "Noviembre", 12: "Diciembre"
}
df_pivot["mes"] = df_pivot["fecha"].dt.month.map(MESES_ES)
```

---

### Problema 2: `PermissionError` al escribir el archivo Excel en Windows

**Síntomas:** `PermissionError: [Errno 13] Permission denied: 'data/output/reporte_ventas.xlsx'` al ejecutar el script por segunda vez.

**Causa:** El archivo `.xlsx` está abierto en Microsoft Excel u otro programa que mantiene un bloqueo de escritura sobre el archivo.

**Solución:** Cierra el archivo Excel antes de re-ejecutar el script. Alternativamente, agrega un manejo defensivo:

```python
import os
import sys

ruta_excel = OUTPUT_DIR / "reporte_ventas.xlsx"
if ruta_excel.exists():
    try:
        # Intentar abrir en modo escritura para verificar que no está bloqueado
        with open(ruta_excel, "a"):
            pass
    except PermissionError:
        print(f"❌ ERROR: El archivo {ruta_excel} está abierto en otro programa.")
        print("   Ciérrelo e intente de nuevo.")
        sys.exit(1)

exportar_excel(df_limpio, kpis, ruta_excel)
```

---

## Limpieza

Para este laboratorio no es necesario eliminar los archivos generados, ya que serán la entrada del Lab 07. Sin embargo, si deseas reiniciar el ejercicio:

```bash
cd ~/proyectos/auto_reporter

# Eliminar solo los archivos de salida (mantener el CSV de entrada)
rm -f data/output/ventas_limpias.csv
rm -f data/output/reporte_ventas.xlsx
rm -f data/output/reporte_ventas.html

# Para un reinicio completo (incluyendo datos generados)
rm -f data/input/ventas_2024.csv
```

Para desactivar el entorno virtual:

```bash
deactivate
```

---

## Resumen

En este laboratorio completaste un pipeline end-to-end de generación automática de reportes:

| Fase | Herramienta | Resultado |
|------|-------------|-----------|
| Carga y exploración | `pd.read_csv()`, `info()`, `describe()` | Diagnóstico de calidad |
| Limpieza | `drop_duplicates()`, `fillna()`, `pd.to_datetime()`, `astype()` | Dataset sin nulos ni anomalías |
| Validación | Aserciones y filtros booleanos | Reglas de negocio cumplidas |
| Agregación | `groupby()`, `pivot_table()` | KPIs calculados |
| Exportación Excel | `pd.ExcelWriter()` + openpyxl | Archivo `.xlsx` con formato |
| Reporte HTML | Jinja2 `Environment` + `render()` | HTML visual con KPIs |

**Archivos generados (entrada para Lab 07):**
- `data/output/ventas_limpias.csv` — dataset limpio y validado
- `data/output/reporte_ventas.xlsx` — reporte Excel con 5 hojas formateadas
- `data/output/reporte_ventas.html` — reporte visual renderizado

### Recursos adicionales

- [Documentación de pandas — GroupBy](https://pandas.pydata.org/docs/user_guide/groupby.html)
- [Documentación de openpyxl — Working with styles](https://openpyxl.readthedocs.io/en/stable/styles.html)
- [Documentación de Jinja2 — Template Designer](https://jinja.palletsprojects.com/en/3.1.x/templates/)
- [pandas — ExcelWriter](https://pandas.pydata.org/docs/reference/api/pandas.ExcelWriter.html)
