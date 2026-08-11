# Práctica 3 — Biblioteca reutilizable para automatización

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 32 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

## Descripción general

En este laboratorio transformarás el código monolítico de procesamiento de datos del Lab 2 (`data_processor.py`) en una biblioteca modular compuesta por tres submódulos especializados: `file_utils.py`, `data_utils.py` y `system_utils.py`. Cada módulo expondrá funciones con anotaciones de tipo completas, docstrings estilo Google, uso de `*args`/`**kwargs` y retornos múltiples tipados. Verificarás la corrección de tipos con `mypy` y validarás la funcionalidad con `pytest`.

## Objetivos de aprendizaje

- [ ] Refactorizar código existente en funciones con anotaciones de tipo y docstrings completos siguiendo PEP 257
- [ ] Implementar funciones que acepten argumentos variables usando `*args` y `**kwargs`
- [ ] Diseñar funciones con retornos múltiples tipados usando `Tuple` y `TypedDict`
- [ ] Modularizar la biblioteca en submódulos dentro de `src/` siguiendo responsabilidad única
- [ ] Utilizar `os`, `pathlib` y `sys` para operaciones portables de sistema de archivos

## Prerrequisitos

### Conocimientos previos

- Lab 02-00-01 completado: `data_processor.py` funcional y `employees.csv` en `data/input/`
- Comprensión de funciones Python (definición, parámetros, retorno)
- Noción básica de importación de módulos

### Acceso requerido

- Terminal con acceso al directorio `~/automation_project/`
- Entorno virtual `.venv` configurado con Python 3.12.1
- Conexión a Internet no requerida (todo es local)

## Entorno del laboratorio

### Software necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Python | 3.12.1 | Lenguaje principal |
| mypy | 1.9.0 | Verificación estática de tipos |
| pytest | 8.1.1 | Pruebas unitarias |
| black | 24.3.0 | Formateo de código |

### Preparación inicial

```bash
cd ~/automation_project
source .venv/bin/activate   # Windows: .venv\Scripts\activate
python --version            # Debe mostrar Python 3.12.1
pip show mypy pytest black | grep -E "^(Name|Version)"
```

Verifica que el archivo de datos existe:

```bash
head -3 data/input/employees.csv
```

---

## Paso 1 — Crear la estructura de paquete en `src/`

### Objetivo

Establecer la estructura de directorios y archivos `__init__.py` necesarios para que `src/` funcione como un paquete Python importable con submódulos.

### Instrucciones

1. Navega al directorio raíz del proyecto:

```bash
cd ~/automation_project
```

2. Crea los archivos de módulo vacíos (si `src/` ya existe del Lab 2, solo añade los nuevos):

```bash
touch src/__init__.py
touch src/file_utils.py
touch src/data_utils.py
touch src/system_utils.py
```

En Windows (PowerShell):

```powershell
New-Item -ItemType File -Path src\__init__.py -Force
New-Item -ItemType File -Path src\file_utils.py -Force
New-Item -ItemType File -Path src\data_utils.py -Force
New-Item -ItemType File -Path src\system_utils.py -Force
```

3. Verifica la estructura resultante:

```bash
find src/ -type f | sort
```

### Salida esperada

```
src/__init__.py
src/data_processor.py
src/data_utils.py
src/file_utils.py
src/system_utils.py
```

### Verificación

```bash
python -c "import src; print('Paquete src importado correctamente')"
```

---

## Paso 2 — Implementar `file_utils.py` con `pathlib` y retornos tipados

### Objetivo

Crear funciones de lectura y escritura de archivos usando `pathlib.Path`, con anotaciones de tipo completas, docstrings estilo Google y uso de `**kwargs` para parámetros opcionales de configuración.

### Instrucciones

1. Abre `src/file_utils.py` en tu editor y escribe el siguiente contenido:

```python
"""Módulo de utilidades para operaciones de lectura/escritura de archivos.

Proporciona funciones portables basadas en pathlib.Path para
leer CSVs, escribir archivos de texto y verificar rutas.
"""

import csv
from pathlib import Path
from typing import TypedDict


class ResultadoLectura(TypedDict):
    """Estructura tipada para el resultado de lectura de un CSV."""

    registros: list[dict[str, str]]
    total_filas: int
    columnas: list[str]


def leer_csv(ruta: Path, **kwargs: str) -> ResultadoLectura:
    """Lee un archivo CSV y retorna los registros con metadatos.

    Utiliza pathlib.Path para garantizar portabilidad entre sistemas
    operativos. Acepta parámetros opcionales de configuración via kwargs.

    Args:
        ruta: Ruta al archivo CSV (objeto Path).
        **kwargs: Parámetros opcionales:
            - encoding (str): Codificación del archivo. Por defecto 'utf-8'.
            - delimiter (str): Separador de columnas. Por defecto ','.

    Returns:
        TypedDict con 'registros' (lista de dicts), 'total_filas' (int)
        y 'columnas' (lista de nombres de columna).

    Raises:
        FileNotFoundError: Si la ruta no existe.
        ValueError: Si el archivo está vacío.
    """
    if not ruta.exists():
        raise FileNotFoundError(f"El archivo no existe: {ruta}")

    encoding = kwargs.get("encoding", "utf-8")
    delimiter = kwargs.get("delimiter", ",")

    with open(ruta, mode="r", encoding=encoding, newline="") as archivo:
        lector = csv.DictReader(archivo, delimiter=delimiter)
        registros: list[dict[str, str]] = list(lector)

    if not registros:
        raise ValueError(f"El archivo está vacío: {ruta}")

    columnas = list(registros[0].keys())

    return ResultadoLectura(
        registros=registros,
        total_filas=len(registros),
        columnas=columnas,
    )


def escribir_texto(
    ruta: Path,
    contenido: str,
    modo: str = "w",
    encoding: str = "utf-8",
) -> int:
    """Escribe contenido de texto en un archivo, creando directorios si es necesario.

    Args:
        ruta: Ruta destino del archivo (objeto Path).
        contenido: Texto a escribir.
        modo: Modo de apertura ('w' para sobrescribir, 'a' para agregar).
        encoding: Codificación del archivo.

    Returns:
        Número de caracteres escritos.

    Raises:
        ValueError: Si el modo no es 'w' ni 'a'.
    """
    if modo not in ("w", "a"):
        raise ValueError(f"Modo inválido: '{modo}'. Use 'w' o 'a'.")

    ruta.parent.mkdir(parents=True, exist_ok=True)

    with open(ruta, mode=modo, encoding=encoding) as archivo:
        caracteres = archivo.write(contenido)

    return caracteres


def verificar_ruta(ruta: Path) -> tuple[bool, str]:
    """Verifica si una ruta existe y retorna su tipo.

    Args:
        ruta: Ruta a verificar.

    Returns:
        Tupla (existe: bool, tipo: str) donde tipo es
        'archivo', 'directorio' o 'inexistente'.
    """
    if not ruta.exists():
        return (False, "inexistente")
    if ruta.is_file():
        return (True, "archivo")
    if ruta.is_dir():
        return (True, "directorio")
    return (True, "otro")
```

2. Guarda el archivo.

3. Verifica la sintaxis ejecutando:

```bash
python -c "from src.file_utils import leer_csv, escribir_texto, verificar_ruta; print('file_utils OK')"
```

### Salida esperada

```
file_utils OK
```

### Verificación

```bash
python -c "
from pathlib import Path
from src.file_utils import leer_csv

resultado = leer_csv(Path('data/input/employees.csv'))
print(f'Filas leídas: {resultado[\"total_filas\"]}')
print(f'Columnas: {resultado[\"columnas\"]}')
"
```

Debe mostrar 50 filas y las columnas del CSV (`id`, `nombre`, `departamento`, `salario`, `fecha_ingreso`).

---

## Paso 3 — Implementar `data_utils.py` con `*args`, `**kwargs` y `TypedDict`

### Objetivo

Crear funciones de transformación y validación de datos que demuestren el uso de argumentos variables, retornos múltiples tipados y el principio de responsabilidad única.

### Instrucciones

1. Abre `src/data_utils.py` y escribe:

```python
"""Módulo de utilidades para transformación y validación de datos.

Contiene funciones puras para filtrar, transformar y resumir
registros de empleados con tipado estático completo.
"""

from typing import Any, TypedDict


class ResumenEstadistico(TypedDict):
    """Estructura tipada para resumen estadístico de salarios."""

    total: float
    promedio: float
    minimo: float
    maximo: float
    cantidad: int


def filtrar_registros(
    registros: list[dict[str, str]], **criterios: str
) -> list[dict[str, str]]:
    """Filtra registros que coincidan con todos los criterios proporcionados.

    Acepta criterios de filtrado como kwargs donde la clave es el nombre
    de la columna y el valor es el valor esperado (comparación exacta,
    insensible a mayúsculas).

    Args:
        registros: Lista de diccionarios con los datos.
        **criterios: Pares columna=valor para filtrar.
            Ejemplo: departamento="Tecnología", activo="True".

    Returns:
        Lista de registros que cumplen todos los criterios.

    Raises:
        ValueError: Si no se proporciona al menos un criterio.
    """
    if not criterios:
        raise ValueError("Debe proporcionar al menos un criterio de filtrado.")

    resultados: list[dict[str, str]] = []
    for registro in registros:
        coincide = all(
            registro.get(campo, "").strip().lower() == valor.strip().lower()
            for campo, valor in criterios.items()
        )
        if coincide:
            resultados.append(registro)

    return resultados


def transformar_campos(
    registro: dict[str, str], *campos: str, transformacion: str = "titulo"
) -> dict[str, str]:
    """Aplica una transformación de texto a los campos especificados.

    Args:
        registro: Diccionario con los datos de un registro.
        *campos: Nombres de los campos a transformar (argumentos posicionales variables).
        transformacion: Tipo de transformación a aplicar.
            Opciones: 'titulo', 'mayusculas', 'minusculas'. Por defecto 'titulo'.

    Returns:
        Nuevo diccionario con los campos transformados.

    Raises:
        ValueError: Si la transformación no es válida.
        KeyError: Si algún campo no existe en el registro.
    """
    transformaciones_validas = ("titulo", "mayusculas", "minusculas")
    if transformacion not in transformaciones_validas:
        raise ValueError(
            f"Transformación '{transformacion}' no válida. "
            f"Use: {transformaciones_validas}"
        )

    resultado = registro.copy()

    for campo in campos:
        if campo not in resultado:
            raise KeyError(f"El campo '{campo}' no existe en el registro.")
        valor = resultado[campo]
        if transformacion == "titulo":
            resultado[campo] = valor.strip().title()
        elif transformacion == "mayusculas":
            resultado[campo] = valor.strip().upper()
        elif transformacion == "minusculas":
            resultado[campo] = valor.strip().lower()

    return resultado


def calcular_resumen_salarios(
    registros: list[dict[str, str]], campo_salario: str = "salario"
) -> ResumenEstadistico:
    """Calcula estadísticas básicas del campo de salario.

    Args:
        registros: Lista de registros con datos numéricos.
        campo_salario: Nombre del campo que contiene el salario.

    Returns:
        TypedDict con total, promedio, mínimo, máximo y cantidad.

    Raises:
        ValueError: Si no hay registros o el campo no es numérico.
    """
    if not registros:
        raise ValueError("La lista de registros está vacía.")

    salarios: list[float] = []
    for registro in registros:
        try:
            salarios.append(float(registro[campo_salario]))
        except (ValueError, KeyError) as e:
            raise ValueError(
                f"Error al procesar salario en registro: {e}"
            ) from e

    return ResumenEstadistico(
        total=sum(salarios),
        promedio=sum(salarios) / len(salarios),
        minimo=min(salarios),
        maximo=max(salarios),
        cantidad=len(salarios),
    )


def validar_registro(
    registro: dict[str, Any], *campos_requeridos: str
) -> tuple[bool, list[str]]:
    """Valida que un registro contenga los campos requeridos no vacíos.

    Args:
        registro: Diccionario a validar.
        *campos_requeridos: Nombres de campos que deben existir y no estar vacíos.

    Returns:
        Tupla (es_valido: bool, errores: list[str]) donde errores contiene
        mensajes descriptivos de cada campo faltante o vacío.
    """
    errores: list[str] = []

    for campo in campos_requeridos:
        if campo not in registro:
            errores.append(f"Campo '{campo}' no encontrado.")
        elif not str(registro[campo]).strip():
            errores.append(f"Campo '{campo}' está vacío.")

    return (len(errores) == 0, errores)
```

2. Guarda el archivo y verifica la importación:

```bash
python -c "from src.data_utils import filtrar_registros, transformar_campos, calcular_resumen_salarios, validar_registro; print('data_utils OK')"
```

### Salida esperada

```
data_utils OK
```

### Verificación

```bash
python -c "
from pathlib import Path
from src.file_utils import leer_csv
from src.data_utils import calcular_resumen_salarios

datos = leer_csv(Path('data/input/employees.csv'))
resumen = calcular_resumen_salarios(datos['registros'])
print(f'Promedio salarial: {resumen[\"promedio\"]:.2f}')
print(f'Cantidad empleados: {resumen[\"cantidad\"]}')
"
```

---

## Paso 4 — Implementar `system_utils.py` con `os`, `pathlib` y `sys`

### Objetivo

Crear funciones de utilidad del sistema que usen las librerías estándar `os`, `pathlib` y `sys` para operaciones portables de archivos y diagnóstico del entorno.

### Instrucciones

1. Abre `src/system_utils.py` y escribe:

```python
"""Módulo de utilidades del sistema operativo.

Proporciona funciones portables para gestión de directorios,
información del entorno y operaciones de respaldo usando
os, pathlib y sys.
"""

import os
import sys
import shutil
from pathlib import Path
from typing import TypedDict


class InfoEntorno(TypedDict):
    """Información del entorno de ejecución."""

    python_version: str
    plataforma: str
    directorio_trabajo: str
    directorio_proyecto: str
    encoding_sistema: str


def obtener_info_entorno(directorio_proyecto: Path | None = None) -> InfoEntorno:
    """Recopila información del entorno de ejecución actual.

    Args:
        directorio_proyecto: Ruta al directorio raíz del proyecto.
            Si es None, usa el directorio de trabajo actual.

    Returns:
        TypedDict con versión de Python, plataforma, directorios y encoding.
    """
    if directorio_proyecto is None:
        directorio_proyecto = Path.cwd()

    return InfoEntorno(
        python_version=sys.version.split()[0],
        plataforma=sys.platform,
        directorio_trabajo=str(Path.cwd()),
        directorio_proyecto=str(directorio_proyecto.resolve()),
        encoding_sistema=sys.getdefaultencoding(),
    )


def crear_directorio_seguro(ruta: Path, *subdirectorios: str) -> list[Path]:
    """Crea un directorio y opcionalmente subdirectorios dentro de él.

    Args:
        ruta: Directorio base a crear.
        *subdirectorios: Nombres de subdirectorios adicionales a crear
            dentro de la ruta base.

    Returns:
        Lista de todas las rutas (Path) creadas o que ya existían.
    """
    rutas_creadas: list[Path] = []

    ruta.mkdir(parents=True, exist_ok=True)
    rutas_creadas.append(ruta)

    for subdir in subdirectorios:
        ruta_subdir = ruta / subdir
        ruta_subdir.mkdir(parents=True, exist_ok=True)
        rutas_creadas.append(ruta_subdir)

    return rutas_creadas


def listar_archivos(
    directorio: Path, extension: str | None = None, recursivo: bool = False
) -> list[Path]:
    """Lista archivos en un directorio, opcionalmente filtrados por extensión.

    Args:
        directorio: Directorio a explorar.
        extension: Extensión para filtrar (ej: '.csv', '.txt'). Sin filtro si None.
        recursivo: Si True, busca también en subdirectorios.

    Returns:
        Lista de rutas Path a los archivos encontrados.

    Raises:
        NotADirectoryError: Si la ruta no es un directorio válido.
    """
    if not directorio.is_dir():
        raise NotADirectoryError(f"No es un directorio válido: {directorio}")

    patron = "**/*" if recursivo else "*"

    archivos = [
        p for p in directorio.glob(patron)
        if p.is_file()
    ]

    if extension is not None:
        ext = extension if extension.startswith(".") else f".{extension}"
        archivos = [a for a in archivos if a.suffix.lower() == ext.lower()]

    return sorted(archivos)


def obtener_tamano_directorio(directorio: Path) -> tuple[int, str]:
    """Calcula el tamaño total de un directorio en bytes.

    Args:
        directorio: Ruta al directorio.

    Returns:
        Tupla (tamano_bytes: int, tamano_legible: str) donde tamano_legible
        es una representación formateada (ej: '2.5 MB').
    """
    total_bytes = 0
    for ruta in directorio.rglob("*"):
        if ruta.is_file():
            total_bytes += ruta.stat().st_size

    # Formato legible
    if total_bytes < 1024:
        legible = f"{total_bytes} B"
    elif total_bytes < 1024**2:
        legible = f"{total_bytes / 1024:.1f} KB"
    elif total_bytes < 1024**3:
        legible = f"{total_bytes / 1024**2:.1f} MB"
    else:
        legible = f"{total_bytes / 1024**3:.1f} GB"

    return (total_bytes, legible)
```

2. Guarda y verifica:

```bash
python -c "from src.system_utils import obtener_info_entorno, crear_directorio_seguro, listar_archivos, obtener_tamano_directorio; print('system_utils OK')"
```

### Salida esperada

```
system_utils OK
```

### Verificación

```bash
python -c "
from pathlib import Path
from src.system_utils import obtener_info_entorno, listar_archivos

info = obtener_info_entorno(Path('.'))
print(f'Python: {info[\"python_version\"]}')
print(f'Plataforma: {info[\"plataforma\"]}')

csvs = listar_archivos(Path('data/input'), extension='.csv')
print(f'Archivos CSV en data/input: {len(csvs)}')
"
```

---

## Paso 5 — Configurar `__init__.py` como API pública del paquete

### Objetivo

Definir el módulo `__init__.py` para exportar las funciones principales de la biblioteca, facilitando importaciones limpias desde código externo.

### Instrucciones

1. Abre `src/__init__.py` y escribe:

```python
"""Biblioteca de automatización — paquete principal.

Exporta las funciones más utilizadas de los submódulos
file_utils, data_utils y system_utils para acceso directo.

Uso:
    from src import leer_csv, filtrar_registros, obtener_info_entorno
"""

from src.file_utils import leer_csv, escribir_texto, verificar_ruta
from src.data_utils import (
    filtrar_registros,
    transformar_campos,
    calcular_resumen_salarios,
    validar_registro,
)
from src.system_utils import (
    obtener_info_entorno,
    crear_directorio_seguro,
    listar_archivos,
    obtener_tamano_directorio,
)

__all__ = [
    "leer_csv",
    "escribir_texto",
    "verificar_ruta",
    "filtrar_registros",
    "transformar_campos",
    "calcular_resumen_salarios",
    "validar_registro",
    "obtener_info_entorno",
    "crear_directorio_seguro",
    "listar_archivos",
    "obtener_tamano_directorio",
]
```

2. Verifica que las importaciones directas funcionen:

```bash
python -c "from src import leer_csv, filtrar_registros, obtener_info_entorno; print('API pública OK')"
```

### Salida esperada

```
API pública OK
```

---

## Paso 6 — Formatear con `black` y verificar tipos con `mypy`

### Objetivo

Aplicar formateo automático con `black` y verificar que todas las anotaciones de tipo sean correctas con `mypy` sin errores.

### Instrucciones

1. Ejecuta `black` sobre todo el paquete `src/`:

```bash
black src/
```

2. Ejecuta `mypy` en modo estricto sobre los módulos:

```bash
mypy src/file_utils.py src/data_utils.py src/system_utils.py --strict
```

> **Nota:** Si `mypy` reporta errores relacionados con `TypedDict` y el uso de `list[dict]` (sintaxis 3.9+), asegúrate de estar usando Python 3.12.1. Si hay advertencias sobre módulos faltantes, puedes agregar `--ignore-missing-imports`.

3. Si `mypy` reporta errores, corrígelos según las indicaciones. Los errores más comunes son:
   - Falta de anotación de retorno en alguna función
   - Uso de `Any` sin importar explícitamente
   - Incompatibilidad entre tipo declarado y valor retornado

### Salida esperada (black)

```
reformatted src/file_utils.py
reformatted src/data_utils.py
reformatted src/system_utils.py
All done! ✨ 🍰 ✨
3 files reformatted.
```

### Salida esperada (mypy)

```
Success: no issues found in 3 source files
```

### Verificación

Si `mypy` muestra `Success: no issues found`, los tipos están correctos. Si muestra errores, revisa la línea indicada y ajusta la anotación.

---

## Paso 7 — Escribir pruebas unitarias con `pytest`

### Objetivo

Crear un archivo de pruebas que valide las funciones críticas de los tres módulos usando `pytest`.

### Instrucciones

1. Crea el archivo de pruebas:

```bash
touch tests/__init__.py
touch tests/test_biblioteca.py
```

2. Abre `tests/test_biblioteca.py` y escribe:

```python
"""Pruebas unitarias para la biblioteca de automatización (src/)."""

import pytest
from pathlib import Path
from src.file_utils import leer_csv, escribir_texto, verificar_ruta
from src.data_utils import (
    filtrar_registros,
    transformar_campos,
    calcular_resumen_salarios,
    validar_registro,
)
from src.system_utils import (
    obtener_info_entorno,
    crear_directorio_seguro,
    listar_archivos,
)


# === Tests para file_utils ===


class TestLeerCsv:
    """Pruebas para la función leer_csv."""

    def test_lectura_exitosa(self) -> None:
        """Verifica lectura correcta del CSV de empleados."""
        resultado = leer_csv(Path("data/input/employees.csv"))
        assert resultado["total_filas"] == 50
        assert "nombre" in resultado["columnas"]
        assert "salario" in resultado["columnas"]

    def test_archivo_inexistente(self) -> None:
        """Verifica que lanza FileNotFoundError con ruta inválida."""
        with pytest.raises(FileNotFoundError):
            leer_csv(Path("ruta/inexistente.csv"))

    def test_kwargs_encoding(self) -> None:
        """Verifica que acepta encoding como kwarg."""
        resultado = leer_csv(Path("data/input/employees.csv"), encoding="utf-8")
        assert resultado["total_filas"] > 0


class TestEscribirTexto:
    """Pruebas para la función escribir_texto."""

    def test_escritura_basica(self, tmp_path: Path) -> None:
        """Verifica escritura de texto en archivo temporal."""
        ruta = tmp_path / "test.txt"
        caracteres = escribir_texto(ruta, "Hola mundo")
        assert caracteres == 10
        assert ruta.read_text() == "Hola mundo"

    def test_modo_invalido(self, tmp_path: Path) -> None:
        """Verifica que modo inválido lanza ValueError."""
        with pytest.raises(ValueError, match="Modo inválido"):
            escribir_texto(tmp_path / "test.txt", "contenido", modo="x")


class TestVerificarRuta:
    """Pruebas para la función verificar_ruta."""

    def test_archivo_existente(self) -> None:
        """Verifica detección de archivo existente."""
        existe, tipo = verificar_ruta(Path("data/input/employees.csv"))
        assert existe is True
        assert tipo == "archivo"

    def test_ruta_inexistente(self) -> None:
        """Verifica detección de ruta inexistente."""
        existe, tipo = verificar_ruta(Path("no/existe/nada.txt"))
        assert existe is False
        assert tipo == "inexistente"


# === Tests para data_utils ===


class TestFiltrarRegistros:
    """Pruebas para la función filtrar_registros."""

    def test_filtrado_por_departamento(self) -> None:
        """Verifica filtrado con un criterio."""
        datos = leer_csv(Path("data/input/employees.csv"))
        # Obtener un departamento real del primer registro
        dept = datos["registros"][0]["departamento"]
        resultado = filtrar_registros(datos["registros"], departamento=dept)
        assert len(resultado) >= 1
        assert all(r["departamento"] == dept for r in resultado)

    def test_sin_criterios_lanza_error(self) -> None:
        """Verifica que sin criterios lanza ValueError."""
        with pytest.raises(ValueError):
            filtrar_registros([{"a": "1"}])


class TestTransformarCampos:
    """Pruebas para la función transformar_campos."""

    def test_transformacion_titulo(self) -> None:
        """Verifica transformación a título."""
        registro = {"nombre": "ana gómez", "departamento": "tecnología"}
        resultado = transformar_campos(registro, "nombre", transformacion="titulo")
        assert resultado["nombre"] == "Ana Gómez"

    def test_transformacion_mayusculas(self) -> None:
        """Verifica transformación a mayúsculas."""
        registro = {"nombre": "ana gómez"}
        resultado = transformar_campos(registro, "nombre", transformacion="mayusculas")
        assert resultado["nombre"] == "ANA GÓMEZ"

    def test_campo_inexistente(self) -> None:
        """Verifica que campo inexistente lanza KeyError."""
        with pytest.raises(KeyError):
            transformar_campos({"a": "1"}, "campo_falso")


class TestValidarRegistro:
    """Pruebas para la función validar_registro."""

    def test_registro_valido(self) -> None:
        """Verifica validación exitosa."""
        registro = {"nombre": "Ana", "salario": "50000"}
        es_valido, errores = validar_registro(registro, "nombre", "salario")
        assert es_valido is True
        assert errores == []

    def test_campo_faltante(self) -> None:
        """Verifica detección de campo faltante."""
        registro = {"nombre": "Ana"}
        es_valido, errores = validar_registro(registro, "nombre", "email")
        assert es_valido is False
        assert len(errores) == 1


# === Tests para system_utils ===


class TestObtenerInfoEntorno:
    """Pruebas para la función obtener_info_entorno."""

    def test_info_basica(self) -> None:
        """Verifica que retorna información del entorno."""
        info = obtener_info_entorno()
        assert info["python_version"].startswith("3.12")
        assert info["plataforma"] in ("linux", "darwin", "win32")


class TestCrearDirectorioSeguro:
    """Pruebas para la función crear_directorio_seguro."""

    def test_creacion_con_subdirectorios(self, tmp_path: Path) -> None:
        """Verifica creación de directorio con subdirectorios."""
        base = tmp_path / "nuevo"
        rutas = crear_directorio_seguro(base, "sub1", "sub2")
        assert len(rutas) == 3
        assert (base / "sub1").is_dir()
        assert (base / "sub2").is_dir()


class TestListarArchivos:
    """Pruebas para la función listar_archivos."""

    def test_listar_csv(self) -> None:
        """Verifica listado de archivos CSV."""
        archivos = listar_archivos(Path("data/input"), extension=".csv")
        assert len(archivos) >= 1
        assert all(a.suffix == ".csv" for a in archivos)

    def test_directorio_invalido(self) -> None:
        """Verifica error con directorio inválido."""
        with pytest.raises(NotADirectoryError):
            listar_archivos(Path("ruta/falsa"))
```

3. Ejecuta las pruebas:

```bash
pytest tests/test_biblioteca.py -v
```

### Salida esperada

```
tests/test_biblioteca.py::TestLeerCsv::test_lectura_exitosa PASSED
tests/test_biblioteca.py::TestLeerCsv::test_archivo_inexistente PASSED
tests/test_biblioteca.py::TestLeerCsv::test_kwargs_encoding PASSED
tests/test_biblioteca.py::TestEscribirTexto::test_escritura_basica PASSED
tests/test_biblioteca.py::TestEscribirTexto::test_modo_invalido PASSED
tests/test_biblioteca.py::TestVerificarRuta::test_archivo_existente PASSED
tests/test_biblioteca.py::TestVerificarRuta::test_ruta_inexistente PASSED
tests/test_biblioteca.py::TestFiltrarRegistros::test_filtrado_por_departamento PASSED
tests/test_biblioteca.py::TestFiltrarRegistros::test_sin_criterios_lanza_error PASSED
tests/test_biblioteca.py::TestTransformarCampos::test_transformacion_titulo PASSED
tests/test_biblioteca.py::TestTransformarCampos::test_transformacion_mayusculas PASSED
tests/test_biblioteca.py::TestTransformarCampos::test_campo_inexistente PASSED
tests/test_biblioteca.py::TestValidarRegistro::test_registro_valido PASSED
tests/test_biblioteca.py::TestValidarRegistro::test_campo_faltante PASSED
tests/test_biblioteca.py::TestObtenerInfoEntorno::test_info_basica PASSED
tests/test_biblioteca.py::TestCrearDirectorioSeguro::test_creacion_con_subdirectorios PASSED
tests/test_biblioteca.py::TestListarArchivos::test_listar_csv PASSED
tests/test_biblioteca.py::TestListarArchivos::test_directorio_invalido PASSED

==================== 18 passed in 0.XXs ====================
```

### Verificación

Todos los tests deben pasar (18 passed). Si alguno falla, revisa el mensaje de error y corrige el módulo correspondiente.

---

## Validación y pruebas finales

Ejecuta la validación completa del laboratorio con estos tres comandos en secuencia:

```bash
# 1. Formateo
black src/ tests/ --check

# 2. Verificación de tipos
mypy src/file_utils.py src/data_utils.py src/system_utils.py --strict

# 3. Pruebas completas
pytest tests/test_biblioteca.py -v --tb=short
```

**Criterios de éxito:**

| Verificación | Resultado esperado |
|---|---|
| `black --check` | Sin archivos por reformatear |
| `mypy --strict` | `Success: no issues found in 3 source files` |
| `pytest` | 18 tests passed, 0 failed |
| Importación directa | `from src import leer_csv` funciona sin error |

Prueba de integración final:

```bash
python -c "
from pathlib import Path
from src import leer_csv, filtrar_registros, calcular_resumen_salarios, obtener_info_entorno

# Leer datos
datos = leer_csv(Path('data/input/employees.csv'))
print(f'✓ CSV leído: {datos[\"total_filas\"]} registros')

# Calcular resumen
resumen = calcular_resumen_salarios(datos['registros'])
print(f'✓ Resumen: promedio={resumen[\"promedio\"]:.2f}, total={resumen[\"total\"]:.2f}')

# Info del entorno
info = obtener_info_entorno(Path('.'))
print(f'✓ Entorno: Python {info[\"python_version\"]} en {info[\"plataforma\"]}')

print('\n🎉 Biblioteca de automatización funcional y verificada.')
"
```

---

## Solución de problemas

### Problema 1: `mypy` reporta errores con sintaxis `list[dict[str, str]]`

**Síntomas:** mypy muestra `error: "list" is not subscriptable` o errores similares con tipos genéricos usando sintaxis nativa.

**Causa:** Se está ejecutando mypy con una versión de Python inferior a 3.9, o la configuración de mypy no reconoce la versión del intérprete. En Python 3.12.1 la sintaxis `list[dict]` es nativa, pero mypy necesita saber qué versión de Python interpretar.

**Solución:**

```bash
# Verificar que mypy usa el Python correcto
mypy --python-version 3.12 src/file_utils.py src/data_utils.py src/system_utils.py --strict
```

Alternativamente, crea un archivo `mypy.ini` en la raíz del proyecto:

```ini
[mypy]
python_version = 3.12
strict = True
```

Luego ejecuta simplemente `mypy src/`.

### Problema 2: `ModuleNotFoundError: No module named 'src'` al ejecutar pytest o scripts

**Síntomas:** Al ejecutar `pytest` o `python -c "from src import ..."` se obtiene un error de módulo no encontrado.

**Causa:** Python no encuentra el paquete `src` porque el directorio de trabajo actual no es `~/automation_project/` o porque falta el archivo `__init__.py`.

**Solución:**

```bash
# 1. Asegurarse de estar en el directorio correcto
cd ~/automation_project

# 2. Verificar que __init__.py existe
ls src/__init__.py

# 3. Si el problema persiste, instalar el paquete en modo desarrollo
pip install -e .
```

Si no tienes un `setup.py` o `pyproject.toml`, la solución más directa es siempre ejecutar desde la raíz del proyecto:

```bash
cd ~/automation_project
python -m pytest tests/test_biblioteca.py -v
```

---

## Limpieza

Este laboratorio no genera archivos temporales que deban eliminarse. Los módulos creados (`file_utils.py`, `data_utils.py`, `system_utils.py`) y las pruebas (`test_biblioteca.py`) son artefactos permanentes que se usarán en los Labs 4 y 5.

Si deseas verificar que no quedaron archivos de caché innecesarios:

```bash
# Eliminar caché de mypy y pytest
rm -rf .mypy_cache/ .pytest_cache/ src/__pycache__/ tests/__pycache__/
```

---

## Resumen

En este laboratorio has logrado:

- **Modularizar** código monolítico en tres submódulos con responsabilidad única: `file_utils.py` (E/S de archivos), `data_utils.py` (transformaciones de datos) y `system_utils.py` (operaciones del sistema)
- **Aplicar tipado estático** completo con `TypedDict`, `tuple[...]`, `list[...]` y tipos opcionales, verificado con `mypy --strict`
- **Implementar `*args` y `**kwargs`** en funciones reales: `transformar_campos(*campos)`, `filtrar_registros(**criterios)`, `crear_directorio_seguro(*subdirectorios)` y `leer_csv(**kwargs)`
- **Documentar** cada función con docstrings estilo Google incluyendo Args, Returns y Raises
- **Validar** la biblioteca con 18 pruebas unitarias usando pytest

### Estructura final del proyecto

```
~/automation_project/
├── src/
│   ├── __init__.py          ← API pública
│   ├── data_processor.py    ← Lab 2 (original)
│   ├── file_utils.py        ← Nuevo: E/S de archivos
│   ├── data_utils.py        ← Nuevo: transformaciones
│   └── system_utils.py      ← Nuevo: utilidades del sistema
├── tests/
│   ├── __init__.py
│   └── test_biblioteca.py   ← 18 tests
├── data/input/employees.csv
└── requirements.txt
```

### Próximos pasos

En el **Lab 4** importarás directamente `crear_directorio_seguro`, `listar_archivos` y `obtener_tamano_directorio` para construir un sistema automatizado de respaldos. En el **Lab 5**, usarás `leer_csv` y `filtrar_registros` como parte del cliente REST. La biblioteca creada aquí es el núcleo reutilizable de todo el proyecto.

### Recursos adicionales

- [Documentación oficial del módulo `typing`](https://docs.python.org/3/library/typing.html)
- [PEP 257 — Convenciones de Docstrings](https://peps.python.org/pep-0257/)
- [Documentación de `pathlib`](https://docs.python.org/3/library/pathlib.html)
- [Guía de mypy para TypedDict](https://mypy.readthedocs.io/en/stable/typed_dict.html)
