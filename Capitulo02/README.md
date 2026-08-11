# Procesamiento de archivos JSON y CSV

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 32 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio implementarás un script completo de procesamiento de datos que lee un archivo CSV de empleados y un archivo JSON de configuración, aplica transformaciones usando comprehensions y estructuras de datos (listas, tuplas, sets y diccionarios), y exporta los resultados filtrados y procesados en formato JSON. El script `data_processor.py` será la base sobre la cual el Lab 3 construirá una biblioteca reutilizable.

## Objetivos de Aprendizaje

- [ ] Leer y parsear archivos CSV utilizando el módulo `csv` de la librería estándar
- [ ] Leer, transformar y serializar datos en formato JSON utilizando el módulo `json`
- [ ] Aplicar listas, tuplas, sets y diccionarios para organizar y deduplicar datos procesados
- [ ] Implementar list comprehensions y dict comprehensions para transformar datasets de forma eficiente
- [ ] Utilizar condicionales y bucles para validar y filtrar registros según parámetros de configuración

## Prerrequisitos

### Conocimientos Requeridos

- Laboratorio 01-00-01 completado (entorno virtual activo y estructura de proyecto creada)
- Tipos de datos primitivos en Python (strings, integers, floats, booleans)
- Comprensión básica de bucles `for` y condicionales `if/else`
- Conceptos de listas, tuplas y sets vistos en la Lección 2.1

### Acceso Necesario

- Terminal/línea de comandos con acceso al directorio `~/automation_project/`
- Entorno virtual `.venv` funcional con Python 3.12.1
- Editor Visual Studio Code (o equivalente)

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Python | 3.12.1 | Intérprete principal |
| pip | 24.0 | Gestión de paquetes |
| black | 24.3.0 | Formateo de código |
| VS Code | 1.88.1 | Editor de código |

### Estructura de Directorios Esperada

```
~/automation_project/
├── .venv/
├── src/
│   └── data_processor.py       ← script principal (crearás aquí)
├── data/
│   ├── input/
│   │   ├── employees.csv       ← archivo de datos (crearás aquí)
│   │   └── config.json         ← configuración de filtrado (crearás aquí)
│   └── output/
│       ├── employees_filtered.json  ← salida generada
│       └── department_summary.json  ← salida generada
├── tests/
├── logs/
├── backups/
└── requirements.txt
```

### Comandos de Preparación

```bash
# Navegar al directorio del proyecto
cd ~/automation_project

# Activar el entorno virtual
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate

# Verificar versión de Python
python --version
# Debe mostrar: Python 3.12.1

# Verificar que black está instalado
black --version
# Debe mostrar: black, 24.3.0
```

---

## Paso a Paso

### Paso 1: Crear el Archivo CSV de Empleados

**Objetivo:** Generar el archivo `employees.csv` con 50 registros que servirá como fuente de datos principal para este laboratorio y los siguientes.

**Instrucciones:**

1. Asegúrate de que el entorno virtual esté activo (el prompt debe mostrar `(.venv)`).

2. Crea un script auxiliar para generar los datos. Crea el archivo `~/automation_project/src/generate_data.py`:

```python
"""Generador de datos de prueba para employees.csv."""

import csv
import os
from pathlib import Path

# Ruta de salida
OUTPUT_DIR = Path(__file__).resolve().parent.parent / "data" / "input"
OUTPUT_FILE = OUTPUT_DIR / "employees.csv"


def generate_employees() -> list[dict[str, str]]:
    """Genera 50 registros de empleados con datos variados."""
    departamentos = ("Ingeniería", "Marketing", "Ventas", "RRHH", "Finanzas")
    nombres = [
        "Ana García", "Carlos López", "María Rodríguez", "Juan Martínez",
        "Laura Hernández", "Pedro Sánchez", "Sofía Díaz", "Diego Torres",
        "Valentina Ruiz", "Andrés Morales", "Camila Ortega", "Felipe Castro",
        "Isabella Vargas", "Mateo Romero", "Luciana Flores", "Sebastián Ríos",
        "Gabriela Mendoza", "Nicolás Herrera", "Paula Jiménez", "Daniel Acosta",
        "Mariana Silva", "Alejandro Peña", "Carolina Reyes", "Tomás Guerrero",
        "Valeria Medina", "Ricardo Campos", "Fernanda Delgado", "Emilio Navarro",
        "Daniela Vega", "Hugo Ramírez", "Natalia Aguilar", "Martín Córdoba",
        "Catalina Molina", "Santiago Paredes", "Andrea Guzmán", "Pablo Figueroa",
        "Renata Salazar", "Joaquín Espinoza", "Camilo Contreras", "Lorena Fuentes",
        "Adrián Soto", "Mónica Bravo", "Iván Pacheco", "Teresa Ramos",
        "Óscar Miranda", "Elena Suárez", "Roberto Lara", "Patricia Ibáñez",
        "Fernando Rojas", "Claudia Vera",
    ]
    salarios_base = [
        45000, 52000, 48000, 61000, 55000, 72000, 43000, 67000, 50000, 58000,
        46000, 63000, 49000, 71000, 54000, 68000, 47000, 59000, 51000, 64000,
        44000, 73000, 53000, 62000, 56000, 69000, 48000, 57000, 52000, 66000,
        45000, 70000, 50000, 60000, 55000, 65000, 47000, 58000, 53000, 61000,
        46000, 72000, 49000, 63000, 54000, 67000, 51000, 59000, 56000, 68000,
    ]
    fechas = [
        "2019-03-15", "2020-07-22", "2018-11-01", "2021-01-10", "2019-06-30",
        "2017-09-14", "2022-02-28", "2018-04-05", "2020-12-18", "2019-08-25",
        "2021-05-12", "2017-11-30", "2022-07-08", "2018-01-20", "2020-03-14",
        "2019-10-07", "2021-08-22", "2018-06-11", "2020-09-03", "2017-12-25",
        "2022-01-15", "2019-04-28", "2021-11-09", "2018-08-17", "2020-05-21",
        "2017-07-03", "2022-04-30", "2019-01-08", "2021-03-19", "2018-10-12",
        "2020-06-27", "2017-05-16", "2022-09-01", "2019-12-04", "2021-06-15",
        "2018-02-22", "2020-11-10", "2019-07-18", "2021-09-25", "2018-03-30",
        "2020-01-05", "2017-08-20", "2022-06-14", "2019-11-28", "2021-04-07",
        "2018-09-09", "2020-08-16", "2019-02-11", "2021-12-03", "2018-05-24",
    ]

    empleados = []
    for i in range(50):
        empleados.append(
            {
                "id": str(i + 1),
                "nombre": nombres[i],
                "departamento": departamentos[i % 5],
                "salario": str(salarios_base[i]),
                "fecha_ingreso": fechas[i],
            }
        )
    return empleados


def write_csv(empleados: list[dict[str, str]]) -> None:
    """Escribe los empleados en formato CSV."""
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
    fieldnames = ["id", "nombre", "departamento", "salario", "fecha_ingreso"]

    with open(OUTPUT_FILE, mode="w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(empleados)

    print(f"✅ Archivo generado: {OUTPUT_FILE}")
    print(f"   Total de registros: {len(empleados)}")


if __name__ == "__main__":
    empleados = generate_employees()
    write_csv(empleados)
```

3. Ejecuta el script generador:

```bash
cd ~/automation_project
python src/generate_data.py
```

**Resultado Esperado:**

```
✅ Archivo generado: /home/<usuario>/automation_project/data/input/employees.csv
   Total de registros: 50
```

**Verificación:**

```bash
# Verificar que el archivo existe y tiene contenido
head -6 data/input/employees.csv
```

Salida esperada (primeras 6 líneas):

```
id,nombre,departamento,salario,fecha_ingreso
1,Ana García,Ingeniería,45000,2019-03-15
2,Carlos López,Marketing,52000,2020-07-22
3,María Rodríguez,Ventas,48000,2018-11-01
4,Juan Martínez,RRHH,61000,2021-01-10
5,Laura Hernández,Finanzas,55000,2019-06-30
```

```bash
# Contar líneas (debe ser 51: 1 header + 50 registros)
wc -l data/input/employees.csv
```

---

### Paso 2: Crear el Archivo JSON de Configuración

**Objetivo:** Crear un archivo `config.json` que contenga parámetros de filtrado que el script principal utilizará para determinar qué registros procesar.

**Instrucciones:**

1. Crea el archivo `~/automation_project/data/input/config.json` con el siguiente contenido:

```json
{
    "filtros": {
        "salario_minimo": 50000,
        "departamentos_incluidos": ["Ingeniería", "Finanzas", "Ventas"],
        "fecha_ingreso_desde": "2019-01-01"
    },
    "salida": {
        "formato": "json",
        "incluir_resumen": true,
        "campos_exportar": ["id", "nombre", "departamento", "salario", "antiguedad_anios"]
    },
    "metadata": {
        "version": "1.0",
        "autor": "automation_project",
        "descripcion": "Configuración de filtrado para procesamiento de empleados"
    }
}
```

2. Verifica la validez del JSON:

```bash
python -c "import json; json.load(open('data/input/config.json')); print('✅ JSON válido')"
```

**Resultado Esperado:**

```
✅ JSON válido
```

**Verificación:**

```bash
python -c "
import json
with open('data/input/config.json') as f:
    config = json.load(f)
print(f'Filtros definidos: {list(config[\"filtros\"].keys())}')
print(f'Departamentos: {config[\"filtros\"][\"departamentos_incluidos\"]}')
"
```

Salida esperada:

```
Filtros definidos: ['salario_minimo', 'departamentos_incluidos', 'fecha_ingreso_desde']
Departamentos: ['Ingeniería', 'Finanzas', 'Ventas']
```

---

### Paso 3: Implementar el Script Principal — Lectura de Datos

**Objetivo:** Crear la primera parte del script `data_processor.py` que lee el CSV y el JSON, almacenando los datos en estructuras apropiadas (listas de diccionarios para empleados, diccionario para configuración).

**Instrucciones:**

1. Crea el archivo `~/automation_project/src/data_processor.py` con el siguiente contenido inicial:

```python
"""
data_processor.py — Procesamiento de archivos JSON y CSV.

Lee empleados desde CSV, aplica filtros desde config.json,
transforma datos y exporta resultados a JSON.
"""

import csv
import json
from collections import Counter
from pathlib import Path

# Rutas base del proyecto
PROJECT_ROOT = Path(__file__).resolve().parent.parent
INPUT_DIR = PROJECT_ROOT / "data" / "input"
OUTPUT_DIR = PROJECT_ROOT / "data" / "output"

# Archivos de entrada
CSV_FILE = INPUT_DIR / "employees.csv"
CONFIG_FILE = INPUT_DIR / "config.json"


def leer_csv(ruta: Path) -> list[dict[str, str]]:
    """Lee un archivo CSV y retorna una lista de diccionarios.

    Cada diccionario representa una fila, con las claves tomadas del header.
    Usa una lista porque necesitamos preservar el orden y permitir duplicados.

    Args:
        ruta: Path al archivo CSV.

    Returns:
        Lista de diccionarios con los datos del CSV.
    """
    empleados: list[dict[str, str]] = []

    with open(ruta, mode="r", encoding="utf-8") as archivo:
        lector = csv.DictReader(archivo)
        for fila in lector:
            empleados.append(dict(fila))

    return empleados


def leer_configuracion(ruta: Path) -> dict:
    """Lee y parsea un archivo JSON de configuración.

    Retorna un diccionario con los parámetros de filtrado y salida.

    Args:
        ruta: Path al archivo JSON.

    Returns:
        Diccionario con la configuración completa.
    """
    with open(ruta, mode="r", encoding="utf-8") as archivo:
        configuracion = json.load(archivo)

    return configuracion


# === Punto de entrada para prueba parcial ===
if __name__ == "__main__":
    # Leer datos
    empleados_raw = leer_csv(CSV_FILE)
    config = leer_configuracion(CONFIG_FILE)

    print(f"📄 Empleados leídos: {len(empleados_raw)}")
    print(f"⚙️  Configuración cargada: {list(config.keys())}")
    print(f"   Primer empleado: {empleados_raw[0]}")
```

2. Ejecuta para verificar la lectura:

```bash
cd ~/automation_project
python src/data_processor.py
```

**Resultado Esperado:**

```
📄 Empleados leídos: 50
⚙️  Configuración cargada: ['filtros', 'salida', 'metadata']
   Primer empleado: {'id': '1', 'nombre': 'Ana García', 'departamento': 'Ingeniería', 'salario': '45000', 'fecha_ingreso': '2019-03-15'}
```

**Verificación:** Confirma que se leen exactamente 50 registros y que las claves del diccionario coinciden con los headers del CSV.

---

### Paso 4: Implementar Transformaciones con Comprehensions

**Objetivo:** Agregar funciones de transformación que utilicen list comprehensions, dict comprehensions y sets para filtrar, deduplicar y enriquecer los datos.

**Instrucciones:**

1. Agrega las siguientes funciones al archivo `data_processor.py`, **antes** del bloque `if __name__ == "__main__"`:

```python
def convertir_tipos(empleados: list[dict[str, str]]) -> list[dict[str, str | int | float]]:
    """Convierte campos numéricos de string a int/float.

    Usa una list comprehension con un dict comprehension anidado para
    transformar cada registro de forma eficiente.

    Args:
        empleados: Lista de diccionarios con todos los valores como strings.

    Returns:
        Lista de diccionarios con salario como int e id como int.
    """
    return [
        {
            "id": int(emp["id"]),
            "nombre": emp["nombre"],
            "departamento": emp["departamento"],
            "salario": int(emp["salario"]),
            "fecha_ingreso": emp["fecha_ingreso"],
        }
        for emp in empleados
    ]


def obtener_departamentos_unicos(empleados: list[dict]) -> tuple[str, ...]:
    """Extrae departamentos únicos usando un set, retorna como tupla inmutable.

    El set elimina duplicados automáticamente. Convertimos a tupla porque
    la lista de departamentos es un dato de referencia que no debe modificarse.

    Args:
        empleados: Lista de diccionarios de empleados.

    Returns:
        Tupla ordenada con nombres de departamentos únicos.
    """
    departamentos_set: set[str] = {emp["departamento"] for emp in empleados}
    return tuple(sorted(departamentos_set))


def filtrar_empleados(
    empleados: list[dict], config: dict
) -> list[dict]:
    """Filtra empleados según los criterios definidos en config.json.

    Aplica tres filtros combinados:
    1. Salario mínimo (condicional numérica)
    2. Departamentos incluidos (pertenencia a set para O(1) lookup)
    3. Fecha de ingreso desde (comparación de strings ISO 8601)

    Args:
        empleados: Lista de empleados con tipos convertidos.
        config: Diccionario de configuración con filtros.

    Returns:
        Lista filtrada de empleados que cumplen TODOS los criterios.
    """
    filtros = config["filtros"]
    salario_min: int = filtros["salario_minimo"]
    # Usar set para búsqueda O(1) en lugar de lista O(n)
    deptos_incluidos: set[str] = set(filtros["departamentos_incluidos"])
    fecha_desde: str = filtros["fecha_ingreso_desde"]

    empleados_filtrados = [
        emp
        for emp in empleados
        if emp["salario"] >= salario_min
        and emp["departamento"] in deptos_incluidos
        and emp["fecha_ingreso"] >= fecha_desde
    ]

    return empleados_filtrados


def calcular_antiguedad(empleados: list[dict], anio_actual: int = 2024) -> list[dict]:
    """Agrega campo 'antiguedad_anios' a cada empleado.

    Usa una list comprehension con operador de desempaquetado (**) para
    crear nuevos diccionarios sin mutar los originales.

    Args:
        empleados: Lista de empleados filtrados.
        anio_actual: Año de referencia para el cálculo.

    Returns:
        Lista de empleados con el campo antiguedad_anios agregado.
    """
    return [
        {
            **emp,
            "antiguedad_anios": anio_actual - int(emp["fecha_ingreso"][:4]),
        }
        for emp in empleados
    ]


def generar_resumen_departamentos(empleados: list[dict]) -> dict[str, dict]:
    """Genera estadísticas por departamento usando collections.Counter y comprehensions.

    Calcula para cada departamento: cantidad de empleados, salario promedio,
    salario máximo y salario mínimo.

    Args:
        empleados: Lista de empleados procesados.

    Returns:
        Diccionario con resumen por departamento.
    """
    # Agrupar empleados por departamento usando un diccionario de listas
    por_departamento: dict[str, list[dict]] = {}
    for emp in empleados:
        depto = emp["departamento"]
        if depto not in por_departamento:
            por_departamento[depto] = []
        por_departamento[depto].append(emp)

    # Generar resumen con dict comprehension
    resumen = {
        depto: {
            "cantidad_empleados": len(lista),
            "salario_promedio": round(
                sum(e["salario"] for e in lista) / len(lista), 2
            ),
            "salario_maximo": max(e["salario"] for e in lista),
            "salario_minimo": min(e["salario"] for e in lista),
            "antiguedad_promedio": round(
                sum(e["antiguedad_anios"] for e in lista) / len(lista), 1
            ),
        }
        for depto, lista in por_departamento.items()
    }

    return resumen
```

2. Actualiza el bloque `if __name__ == "__main__"` para incluir las transformaciones:

```python
if __name__ == "__main__":
    # === LECTURA ===
    empleados_raw = leer_csv(CSV_FILE)
    config = leer_configuracion(CONFIG_FILE)

    print(f"📄 Empleados leídos: {len(empleados_raw)}")
    print(f"⚙️  Configuración cargada: {list(config.keys())}")

    # === TRANSFORMACIÓN ===
    # Paso 1: Convertir tipos
    empleados = convertir_tipos(empleados_raw)

    # Paso 2: Obtener departamentos únicos (demostración de set → tupla)
    deptos = obtener_departamentos_unicos(empleados)
    print(f"\n🏢 Departamentos únicos (tupla inmutable): {deptos}")

    # Paso 3: Filtrar según configuración
    empleados_filtrados = filtrar_empleados(empleados, config)
    print(f"🔍 Empleados después del filtrado: {len(empleados_filtrados)}")

    # Paso 4: Calcular antigüedad
    empleados_procesados = calcular_antiguedad(empleados_filtrados)

    # Paso 5: Generar resumen por departamento
    resumen = generar_resumen_departamentos(empleados_procesados)
    print(f"\n📊 Resumen por departamento:")
    for depto, stats in resumen.items():
        print(f"   {depto}: {stats['cantidad_empleados']} empleados, "
              f"promedio ${stats['salario_promedio']:,.0f}")
```

3. Ejecuta el script actualizado:

```bash
python src/data_processor.py
```

**Resultado Esperado:**

```
📄 Empleados leídos: 50
⚙️  Configuración cargada: ['filtros', 'salida', 'metadata']

🏢 Departamentos únicos (tupla inmutable): ('Finanzas', 'Ingeniería', 'Marketing', 'RRHH', 'Ventas')
🔍 Empleados después del filtrado: 14
   
📊 Resumen por departamento:
   Ingeniería: 5 empleados, promedio $64,400
   Finanzas: 5 empleados, promedio $63,200
   Ventas: 4 empleados, promedio $57,500
```

> **Nota:** Los números exactos pueden variar ligeramente según los datos generados, pero el total de filtrados debe estar entre 12 y 16.

**Verificación:**

```bash
python -c "
from src.data_processor import convertir_tipos, leer_csv, obtener_departamentos_unicos
from pathlib import Path

emps = convertir_tipos(leer_csv(Path('data/input/employees.csv')))
deptos = obtener_departamentos_unicos(emps)

assert len(emps) == 50, f'Se esperaban 50, se obtuvieron {len(emps)}'
assert isinstance(deptos, tuple), 'departamentos debe ser tupla'
assert isinstance(emps[0]['salario'], int), 'salario debe ser int'
print('✅ Transformaciones verificadas correctamente')
"
```

---

### Paso 5: Implementar la Exportación a JSON

**Objetivo:** Agregar funciones para escribir los resultados procesados en archivos JSON en el directorio de salida.

**Instrucciones:**

1. Agrega las siguientes funciones al archivo `data_processor.py`, antes del bloque `if __name__`:

```python
def exportar_json(datos: list[dict] | dict, ruta: Path, descripcion: str = "") -> None:
    """Serializa datos a formato JSON con formato legible.

    Args:
        datos: Estructura de datos a serializar (lista o diccionario).
        ruta: Path de destino para el archivo JSON.
        descripcion: Descripción opcional para el mensaje de confirmación.
    """
    # Asegurar que el directorio de salida existe
    ruta.parent.mkdir(parents=True, exist_ok=True)

    with open(ruta, mode="w", encoding="utf-8") as archivo:
        json.dump(datos, archivo, ensure_ascii=False, indent=2)

    print(f"💾 Exportado: {ruta.name} ({descripcion})")


def preparar_salida(
    empleados: list[dict], campos: list[str]
) -> list[dict]:
    """Selecciona solo los campos especificados en la configuración.

    Usa dict comprehension para crear registros con solo los campos deseados.

    Args:
        empleados: Lista completa de empleados procesados.
        campos: Lista de nombres de campos a incluir en la salida.

    Returns:
        Lista de diccionarios con solo los campos seleccionados.
    """
    # Convertir a set para verificación O(1) de existencia
    campos_disponibles: set[str] = set(empleados[0].keys()) if empleados else set()
    campos_validos: list[str] = [c for c in campos if c in campos_disponibles]

    return [
        {campo: emp[campo] for campo in campos_validos}
        for emp in empleados
    ]
```

2. Actualiza el bloque `if __name__ == "__main__"` agregando la sección de exportación al final:

```python
    # === EXPORTACIÓN ===
    print("\n" + "=" * 50)
    print("EXPORTACIÓN DE RESULTADOS")
    print("=" * 50)

    # Preparar datos con solo los campos configurados
    campos_exportar = config["salida"]["campos_exportar"]
    datos_salida = preparar_salida(empleados_procesados, campos_exportar)

    # Exportar empleados filtrados
    exportar_json(
        datos_salida,
        OUTPUT_DIR / "employees_filtered.json",
        descripcion=f"{len(datos_salida)} empleados filtrados",
    )

    # Exportar resumen por departamento (si está habilitado en config)
    if config["salida"]["incluir_resumen"]:
        exportar_json(
            resumen,
            OUTPUT_DIR / "department_summary.json",
            descripcion="resumen estadístico por departamento",
        )

    # Mostrar IDs procesados como verificación (usando set para unicidad)
    ids_procesados: set[int] = {emp["id"] for emp in empleados_procesados}
    print(f"\n✅ Procesamiento completo. IDs exportados: {len(ids_procesados)}")
    print(f"   Verificación de unicidad: {'Sin duplicados' if len(ids_procesados) == len(empleados_procesados) else '⚠️ Duplicados detectados'}")
```

3. Ejecuta el script completo:

```bash
python src/data_processor.py
```

**Resultado Esperado:**

```
📄 Empleados leídos: 50
⚙️  Configuración cargada: ['filtros', 'salida', 'metadata']

🏢 Departamentos únicos (tupla inmutable): ('Finanzas', 'Ingeniería', 'Marketing', 'RRHH', 'Ventas')
🔍 Empleados después del filtrado: 14

📊 Resumen por departamento:
   Ingeniería: 5 empleados, promedio $64,400
   Finanzas: 5 empleados, promedio $63,200
   Ventas: 4 empleados, promedio $57,500

==================================================
EXPORTACIÓN DE RESULTADOS
==================================================
💾 Exportado: employees_filtered.json (14 empleados filtrados)
💾 Exportado: department_summary.json (resumen estadístico por departamento)

✅ Procesamiento completo. IDs exportados: 14
   Verificación de unicidad: Sin duplicados
```

**Verificación:**

```bash
# Verificar que los archivos de salida existen
ls -la data/output/

# Inspeccionar el contenido del JSON filtrado (primeros 2 registros)
python -c "
import json
with open('data/output/employees_filtered.json') as f:
    datos = json.load(f)
print(f'Registros exportados: {len(datos)}')
print(f'Campos por registro: {list(datos[0].keys())}')
print(f'Ejemplo: {json.dumps(datos[0], ensure_ascii=False, indent=2)}')
"
```

Salida esperada:

```
Registros exportados: 14
Campos por registro: ['id', 'nombre', 'departamento', 'salario', 'antiguedad_anios']
Ejemplo: {
  "id": 6,
  "nombre": "Pedro Sánchez",
  "departamento": "Ingeniería",
  "salario": 72000,
  "antiguedad_anios": 7
}
```

---

### Paso 6: Formatear el Código con Black

**Objetivo:** Aplicar el formateador `black` para asegurar que el código cumple con el estándar de estilo del proyecto.

**Instrucciones:**

1. Ejecuta black en modo verificación primero (sin modificar):

```bash
black --check --diff src/data_processor.py
```

2. Si hay cambios sugeridos, aplica el formateo:

```bash
black src/data_processor.py
```

3. Formatea también el script generador:

```bash
black src/generate_data.py
```

**Resultado Esperado:**

```
reformatted src/data_processor.py

All done! ✨ 🍰 ✨
1 file reformatted.
```

O si ya estaba formateado:

```
All done! ✨ 🍰 ✨
1 file would be left unchanged.
```

**Verificación:**

```bash
black --check src/data_processor.py src/generate_data.py
```

Debe mostrar:

```
All done! ✨ 🍰 ✨
2 files would be left unchanged.
```

---

### Paso 7: Verificación Integral del Procesamiento

**Objetivo:** Ejecutar una verificación completa que valide la integridad de todo el pipeline: lectura → transformación → filtrado → exportación.

**Instrucciones:**

1. Crea un script de verificación `~/automation_project/tests/test_data_processor.py`:

```python
"""Verificación integral del pipeline de procesamiento de datos."""

import json
import sys
from pathlib import Path

# Agregar src al path para importar módulos
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "src"))

from data_processor import (
    leer_csv,
    leer_configuracion,
    convertir_tipos,
    obtener_departamentos_unicos,
    filtrar_empleados,
    calcular_antiguedad,
    generar_resumen_departamentos,
    preparar_salida,
    CSV_FILE,
    CONFIG_FILE,
    OUTPUT_DIR,
)


def verificar_pipeline() -> None:
    """Ejecuta verificaciones sobre todo el pipeline."""
    errores: list[str] = []

    # 1. Verificar lectura CSV
    empleados_raw = leer_csv(CSV_FILE)
    if len(empleados_raw) != 50:
        errores.append(f"CSV: se esperaban 50 registros, se obtuvieron {len(empleados_raw)}")

    # 2. Verificar lectura JSON
    config = leer_configuracion(CONFIG_FILE)
    if "filtros" not in config:
        errores.append("JSON: falta la clave 'filtros' en config.json")

    # 3. Verificar conversión de tipos
    empleados = convertir_tipos(empleados_raw)
    if not isinstance(empleados[0]["salario"], int):
        errores.append(f"Tipos: salario debería ser int, es {type(empleados[0]['salario'])}")

    # 4. Verificar departamentos únicos (debe ser tupla)
    deptos = obtener_departamentos_unicos(empleados)
    if not isinstance(deptos, tuple):
        errores.append(f"Departamentos: debería ser tuple, es {type(deptos)}")
    if len(deptos) != 5:
        errores.append(f"Departamentos: se esperaban 5, se obtuvieron {len(deptos)}")

    # 5. Verificar filtrado
    filtrados = filtrar_empleados(empleados, config)
    if len(filtrados) == 0:
        errores.append("Filtrado: no se obtuvo ningún resultado")
    if len(filtrados) >= len(empleados):
        errores.append("Filtrado: no se filtró ningún registro")

    # 6. Verificar antigüedad
    con_antiguedad = calcular_antiguedad(filtrados)
    if "antiguedad_anios" not in con_antiguedad[0]:
        errores.append("Antigüedad: falta el campo 'antiguedad_anios'")

    # 7. Verificar resumen
    resumen = generar_resumen_departamentos(con_antiguedad)
    for depto, stats in resumen.items():
        if stats["salario_promedio"] <= 0:
            errores.append(f"Resumen: salario promedio inválido para {depto}")

    # 8. Verificar archivos de salida
    archivo_filtrado = OUTPUT_DIR / "employees_filtered.json"
    archivo_resumen = OUTPUT_DIR / "department_summary.json"

    if not archivo_filtrado.exists():
        errores.append(f"Salida: no existe {archivo_filtrado.name}")
    else:
        with open(archivo_filtrado) as f:
            datos = json.load(f)
        if len(datos) != len(filtrados):
            errores.append(
                f"Salida: JSON tiene {len(datos)} registros, "
                f"se esperaban {len(filtrados)}"
            )

    if not archivo_resumen.exists():
        errores.append(f"Salida: no existe {archivo_resumen.name}")

    # === RESULTADO ===
    print("\n" + "=" * 60)
    print("VERIFICACIÓN INTEGRAL DEL PIPELINE")
    print("=" * 60)

    if errores:
        print(f"\n❌ Se encontraron {len(errores)} error(es):\n")
        for err in errores:
            print(f"   • {err}")
        sys.exit(1)
    else:
        print("\n✅ Todas las verificaciones pasaron correctamente:")
        print(f"   • CSV leído: 50 registros")
        print(f"   • JSON config: válido con 3 secciones")
        print(f"   • Conversión de tipos: int para salario e id")
        print(f"   • Departamentos únicos: {len(deptos)} (tupla inmutable)")
        print(f"   • Filtrado: {len(filtrados)} empleados seleccionados")
        print(f"   • Antigüedad calculada: campo agregado correctamente")
        print(f"   • Resumen: {len(resumen)} departamentos con estadísticas")
        print(f"   • Archivos exportados: 2 JSON en data/output/")
        print(f"\n{'=' * 60}")


if __name__ == "__main__":
    verificar_pipeline()
```

2. Ejecuta la verificación:

```bash
cd ~/automation_project
python tests/test_data_processor.py
```

**Resultado Esperado:**

```
============================================================
VERIFICACIÓN INTEGRAL DEL PIPELINE
============================================================

✅ Todas las verificaciones pasaron correctamente:
   • CSV leído: 50 registros
   • JSON config: válido con 3 secciones
   • Conversión de tipos: int para salario e id
   • Departamentos únicos: 5 (tupla inmutable)
   • Filtrado: 14 empleados seleccionados
   • Antigüedad calculada: campo agregado correctamente
   • Resumen: 3 departamentos con estadísticas
   • Archivos exportados: 2 JSON en data/output/

============================================================
```

---

## Validación y Testing

Ejecuta los siguientes comandos para confirmar que el laboratorio se completó exitosamente:

```bash
cd ~/automation_project

# 1. Verificar estructura de archivos generados
echo "=== Archivos de entrada ==="
ls data/input/employees.csv data/input/config.json

echo "=== Archivos de salida ==="
ls data/output/employees_filtered.json data/output/department_summary.json

# 2. Verificar formato con black
black --check src/data_processor.py src/generate_data.py

# 3. Ejecutar pipeline completo
python src/data_processor.py

# 4. Ejecutar verificación integral
python tests/test_data_processor.py

# 5. Verificar contenido JSON de salida
python -c "
import json
with open('data/output/employees_filtered.json') as f:
    emp = json.load(f)
with open('data/output/department_summary.json') as f:
    res = json.load(f)
assert len(emp) > 0, 'No hay empleados filtrados'
assert len(res) > 0, 'No hay resumen de departamentos'
assert 'antiguedad_anios' in emp[0], 'Falta campo antiguedad'
print(f'✅ VALIDACIÓN FINAL EXITOSA')
print(f'   Empleados filtrados: {len(emp)}')
print(f'   Departamentos en resumen: {len(res)}')
"
```

**Criterios de éxito:**
- ✅ `employees.csv` contiene exactamente 50 registros + header
- ✅ `config.json` es válido y contiene las 3 secciones requeridas
- ✅ `employees_filtered.json` contiene solo empleados que cumplen los 3 filtros
- ✅ `department_summary.json` contiene estadísticas por departamento
- ✅ Black no reporta cambios pendientes
- ✅ El script de verificación integral pasa sin errores

---

## Solución de Problemas

### Problema 1: `ModuleNotFoundError: No module named 'src'` al ejecutar tests

**Síntomas:** Al ejecutar `python tests/test_data_processor.py`, Python no encuentra el módulo `data_processor`.

**Causa:** Python no incluye automáticamente el directorio `src/` en su path de búsqueda de módulos. El script de test necesita agregar explícitamente la ruta.

**Solución:**

1. Verifica que estás ejecutando desde el directorio raíz del proyecto:

```bash
cd ~/automation_project
pwd
# Debe mostrar: /home/<usuario>/automation_project
```

2. Verifica que el script de test incluye la manipulación de `sys.path`:

```python
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "src"))
```

3. Alternativa: ejecuta con la variable `PYTHONPATH`:

```bash
PYTHONPATH=src python tests/test_data_processor.py
```

---

### Problema 2: `UnicodeDecodeError` al leer `employees.csv` en Windows

**Síntomas:** Al ejecutar `python src/data_processor.py` en Windows, aparece un error como:

```
UnicodeDecodeError: 'charmap' codec can't decode byte 0x9d in position 42
```

**Causa:** Windows usa por defecto la codificación `cp1252` en lugar de `utf-8`. Los caracteres especiales (tildes, eñes) en los nombres de empleados no se decodifican correctamente.

**Solución:**

1. Verifica que la función `leer_csv` especifica explícitamente `encoding="utf-8"`:

```python
with open(ruta, mode="r", encoding="utf-8") as archivo:
```

2. Si el archivo CSV fue creado sin encoding UTF-8, regenera el archivo:

```bash
python src/generate_data.py
```

3. Verifica la codificación del archivo existente:

```bash
python -c "
with open('data/input/employees.csv', 'rb') as f:
    raw = f.read(100)
print(raw)
# Debe mostrar caracteres UTF-8 correctos (b'\\xc3\\xa1' para 'á')
"
```

---

## Limpieza

No se requiere limpieza de archivos generados, ya que los archivos de salida en `data/output/` serán consumidos por el Lab 3. Sin embargo, si necesitas reiniciar el laboratorio:

```bash
cd ~/automation_project

# Eliminar solo archivos de salida (mantener inputs)
rm -f data/output/employees_filtered.json
rm -f data/output/department_summary.json

# Para reinicio completo (incluyendo datos de entrada)
# rm -f data/input/employees.csv
# rm -f data/input/config.json
```

---

## Resumen

### Conceptos Clave Aplicados

| Concepto | Aplicación en este Lab |
|----------|----------------------|
| **Listas** | Almacenar registros de empleados (orden y duplicados preservados) |
| **Tuplas** | Retornar departamentos únicos como dato inmutable |
| **Sets** | Deduplicar departamentos, búsqueda O(1) en filtrado, verificar unicidad de IDs |
| **Diccionarios** | Representar cada empleado y la configuración JSON |
| **List comprehension** | Convertir tipos, filtrar empleados, calcular antigüedad |
| **Dict comprehension** | Generar resumen por departamento, seleccionar campos de salida |
| **Módulo `csv`** | Lectura estructurada con `DictReader` |
| **Módulo `json`** | Parseo de configuración (`json.load`) y exportación (`json.dump`) |
| **`collections.Counter`** | Importado para uso futuro en conteos (disponible en el módulo) |
| **`pathlib.Path`** | Manejo multiplataforma de rutas de archivos |

### Archivos Generados para el Lab 3

- `~/automation_project/data/output/employees_filtered.json` — datos filtrados listos para consumo
- `~/automation_project/data/output/department_summary.json` — resumen estadístico por departamento
- `~/automation_project/src/data_processor.py` — script reutilizable con funciones modulares

### Recursos Adicionales

- [Documentación oficial: módulo csv](https://docs.python.org/3/library/csv.html)
- [Documentación oficial: módulo json](https://docs.python.org/3/library/json.html)
- [Real Python: Working with JSON Data in Python](https://realpython.com/python-json/)
- [PEP 274: Dict Comprehensions](https://peps.python.org/pep-0274/)
- [Black: The Uncompromising Code Formatter](https://black.readthedocs.io/en/stable/)
