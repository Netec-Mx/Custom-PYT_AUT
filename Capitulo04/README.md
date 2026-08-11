# Práctica 4 — Automatización de respaldos y limpieza de archivos

| Campo | Detalle |
|-------|---------|
| **Duración** | 38 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción general

En este laboratorio desarrollarás un script completo de automatización llamado `backup_manager.py` que integra tres tareas críticas de mantenimiento: creación de respaldos comprimidos con marca de tiempo, limpieza de archivos antiguos y verificación de espacio en disco mediante `subprocess`. El script implementará logging profesional con `RotatingFileHandler`, manejo robusto de excepciones y reutilizará funciones de la biblioteca modular construida en el Lab 3.

## Objetivos de aprendizaje

- [ ] Implementar un script de respaldo automático que comprima directorios usando `shutil` y `zipfile` con nombres basados en timestamp
- [ ] Manipular rutas y directorios de forma portable usando `pathlib.Path` para operaciones de creación, listado, movimiento y eliminación
- [ ] Ejecutar procesos del sistema operativo con `subprocess`, capturando salida y manejando códigos de retorno
- [ ] Implementar manejo de excepciones robusto con bloques `try/except/finally` y excepciones personalizadas
- [ ] Configurar logging estructurado con `RotatingFileHandler` que registre todas las operaciones en `logs/backup_manager.log`

## Prerrequisitos

### Conocimientos requeridos

- Laboratorio 03-00-01 completado: biblioteca modular con `file_utils.py` y `system_utils.py` funcionales en `~/automation_project/src/`
- Comprensión de lectura/escritura de archivos con `open()` y el gestor de contexto `with`
- Conocimiento básico de excepciones en Python (`try/except`)
- Familiaridad con modos de apertura de archivos (`'r'`, `'w'`, `'a'`, `'rb'`, `'wb'`)

### Acceso requerido

- Entorno virtual activo en `~/automation_project/.venv`
- Archivo `~/automation_project/data/input/employees.csv` creado en Lab 2
- Permisos de lectura/escritura sobre `~/automation_project/`

## Entorno del laboratorio

### Software necesario

| Componente | Versión |
|-----------|---------|
| Python | 3.12.1 |
| Sistema Operativo | Windows 10/11, macOS 12+, Ubuntu 22.04 LTS |
| Visual Studio Code | 1.88.1 |
| Git | 2.44.0 |

### Verificación del entorno

```bash
# Navegar al proyecto
cd ~/automation_project

# Activar entorno virtual
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate

# Verificar versión de Python
python --version
# Esperado: Python 3.12.1
```

### Preparación de archivos de prueba

Para que la limpieza de archivos tenga datos sobre los cuales operar, crearemos archivos temporales en `data/output/`:

```bash
# Crear archivos de prueba en data/output/ (ejecutar desde ~/automation_project)
python -c "
from pathlib import Path
import time, os

output_dir = Path('data/output')
output_dir.mkdir(parents=True, exist_ok=True)

# Crear archivos con diferentes antigüedades
for i in range(5):
    archivo = output_dir / f'reporte_antiguo_{i}.txt'
    archivo.write_text(f'Contenido de prueba {i}', encoding='utf-8')
    # Modificar fecha de acceso/modificación a 10 días atrás
    antiguo = time.time() - (10 * 86400)
    os.utime(archivo, (antiguo, antiguo))

# Crear archivos recientes
for i in range(3):
    archivo = output_dir / f'reporte_reciente_{i}.txt'
    archivo.write_text(f'Contenido reciente {i}', encoding='utf-8')

print(f'Archivos creados en {output_dir}:')
for f in sorted(output_dir.iterdir()):
    print(f'  {f.name}')
"
```

**Salida esperada:**

```
Archivos creados en data/output:
  reporte_antiguo_0.txt
  reporte_antiguo_1.txt
  reporte_antiguo_2.txt
  reporte_antiguo_3.txt
  reporte_antiguo_4.txt
  reporte_reciente_0.txt
  reporte_reciente_1.txt
  reporte_reciente_2.txt
```

---

## Paso 1: Definir excepciones personalizadas y configurar logging

### Objetivo

Crear el módulo de excepciones personalizadas y configurar el sistema de logging con `RotatingFileHandler` que registrará todas las operaciones del script.

### Instrucciones

1. Abre tu editor y crea el archivo `~/automation_project/src/backup_exceptions.py`:

```python
"""
Excepciones personalizadas para el módulo de respaldos.
Proporciona mensajes descriptivos para cada tipo de fallo en operaciones de archivos.
"""


class BackupError(Exception):
    """Excepción base para errores de respaldo."""

    def __init__(self, mensaje: str, ruta: str = "") -> None:
        self.ruta = ruta
        super().__init__(f"[BackupError] {mensaje}" + (f" — Ruta: {ruta}" if ruta else ""))


class DirectorioNoEncontrado(BackupError):
    """Se lanza cuando un directorio origen no existe."""

    def __init__(self, ruta: str) -> None:
        super().__init__(f"El directorio no existe: {ruta}", ruta)


class EspacioInsuficiente(BackupError):
    """Se lanza cuando no hay espacio suficiente en disco."""

    def __init__(self, espacio_disponible_mb: float, espacio_requerido_mb: float) -> None:
        self.espacio_disponible_mb = espacio_disponible_mb
        self.espacio_requerido_mb = espacio_requerido_mb
        super().__init__(
            f"Espacio insuficiente: {espacio_disponible_mb:.1f} MB disponibles, "
            f"{espacio_requerido_mb:.1f} MB requeridos"
        )


class LimpiezaError(BackupError):
    """Se lanza cuando falla la eliminación de un archivo durante la limpieza."""

    def __init__(self, ruta: str, motivo: str) -> None:
        super().__init__(f"No se pudo eliminar '{ruta}': {motivo}", ruta)
```

2. Crea el archivo `~/automation_project/src/backup_logger.py` con la configuración de logging:

```python
"""
Configuración de logging para backup_manager.
Usa RotatingFileHandler para evitar que el archivo de log crezca indefinidamente.
"""

import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path


def configurar_logger(
    nombre: str = "backup_manager",
    directorio_logs: Path | None = None,
    nivel: int = logging.INFO,
    max_bytes: int = 5 * 1024 * 1024,  # 5 MB
    backup_count: int = 3,
) -> logging.Logger:
    """
    Configura y retorna un logger con RotatingFileHandler y StreamHandler.

    Args:
        nombre: Nombre del logger.
        directorio_logs: Directorio donde se almacenará el archivo de log.
        nivel: Nivel mínimo de logging (default: INFO).
        max_bytes: Tamaño máximo del archivo de log antes de rotar.
        backup_count: Número de archivos de respaldo del log.

    Returns:
        Logger configurado con handlers de archivo y consola.
    """
    if directorio_logs is None:
        directorio_logs = Path(__file__).resolve().parent.parent / "logs"

    # Crear directorio de logs si no existe
    directorio_logs.mkdir(parents=True, exist_ok=True)

    ruta_log = directorio_logs / f"{nombre}.log"

    logger = logging.getLogger(nombre)
    logger.setLevel(nivel)

    # Evitar duplicar handlers si se llama múltiples veces
    if logger.handlers:
        return logger

    # Formato del log
    formato = logging.Formatter(
        fmt="%(asctime)s | %(levelname)-8s | %(funcName)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )

    # Handler de archivo con rotación
    file_handler = RotatingFileHandler(
        filename=ruta_log,
        maxBytes=max_bytes,
        backupCount=backup_count,
        encoding="utf-8",
    )
    file_handler.setLevel(nivel)
    file_handler.setFormatter(formato)

    # Handler de consola
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.WARNING)
    console_handler.setFormatter(formato)

    logger.addHandler(file_handler)
    logger.addHandler(console_handler)

    return logger
```

3. Verifica la creación de ambos archivos:

```bash
ls -la src/backup_exceptions.py src/backup_logger.py
```

### Salida esperada

```
-rw-r--r--  1 user  staff  1234  ... src/backup_exceptions.py
-rw-r--r--  1 user  staff  1567  ... src/backup_logger.py
```

### Verificación

```bash
python -c "
from src.backup_exceptions import BackupError, DirectorioNoEncontrado, EspacioInsuficiente, LimpiezaError
from src.backup_logger import configurar_logger
from pathlib import Path

# Probar excepciones
try:
    raise DirectorioNoEncontrado('/tmp/no_existe')
except BackupError as e:
    print(f'Excepción capturada: {e}')

# Probar logger
logger = configurar_logger()
logger.info('Logger configurado correctamente')
print(f'Log creado en: {Path(\"logs/backup_manager.log\").resolve()}')
print(f'Contenido del log:')
print(Path('logs/backup_manager.log').read_text(encoding='utf-8'))
"
```

**Salida esperada:**

```
Excepción capturada: [BackupError] El directorio no existe: /tmp/no_existe — Ruta: /tmp/no_existe
Log creado en: /home/<usuario>/automation_project/logs/backup_manager.log
Contenido del log:
2025-XX-XX XX:XX:XX | INFO     | <module> | Logger configurado correctamente
```

---

## Paso 2: Implementar la función de respaldo comprimido

### Objetivo

Crear la función principal de respaldo que comprime el directorio `data/` en un archivo `.zip` con marca de tiempo y lo almacena en `backups/`.

### Instrucciones

1. Crea el archivo `~/automation_project/src/backup_manager.py` con el siguiente contenido inicial:

```python
"""
backup_manager.py — Script de automatización de respaldos y limpieza.

Funcionalidades:
1. Crear respaldos comprimidos (.zip) del directorio data/ con timestamp.
2. Limpiar archivos antiguos en data/output/.
3. Verificar espacio en disco y generar reporte de estado.

Uso:
    python -m src.backup_manager [--dias-antiguedad N] [--min-espacio-mb N]
"""

import shutil
import zipfile
import subprocess
import platform
from pathlib import Path
from datetime import datetime, timedelta

from src.backup_exceptions import (
    BackupError,
    DirectorioNoEncontrado,
    EspacioInsuficiente,
    LimpiezaError,
)
from src.backup_logger import configurar_logger

# Constantes de configuración
PROYECTO_RAIZ = Path(__file__).resolve().parent.parent
DIRECTORIO_DATA = PROYECTO_RAIZ / "data"
DIRECTORIO_BACKUPS = PROYECTO_RAIZ / "backups"
DIRECTORIO_OUTPUT = PROYECTO_RAIZ / "data" / "output"
DIRECTORIO_LOGS = PROYECTO_RAIZ / "logs"

# Inicializar logger
logger = configurar_logger(directorio_logs=DIRECTORIO_LOGS)


def crear_respaldo(
    directorio_origen: Path | None = None,
    directorio_destino: Path | None = None,
    prefijo: str = "backup_data",
) -> Path:
    """
    Crea un archivo ZIP comprimido del directorio origen con timestamp en el nombre.

    Args:
        directorio_origen: Directorio a respaldar (default: data/).
        directorio_destino: Directorio donde almacenar el ZIP (default: backups/).
        prefijo: Prefijo para el nombre del archivo ZIP.

    Returns:
        Path al archivo ZIP creado.

    Raises:
        DirectorioNoEncontrado: Si el directorio origen no existe.
        BackupError: Si ocurre un error durante la compresión.
    """
    if directorio_origen is None:
        directorio_origen = DIRECTORIO_DATA
    if directorio_destino is None:
        directorio_destino = DIRECTORIO_BACKUPS

    logger.info(f"Iniciando respaldo de '{directorio_origen}'")

    # Validar que el directorio origen existe
    if not directorio_origen.exists():
        raise DirectorioNoEncontrado(str(directorio_origen))

    if not directorio_origen.is_dir():
        raise BackupError(f"La ruta no es un directorio: {directorio_origen}")

    # Crear directorio de destino si no existe
    directorio_destino.mkdir(parents=True, exist_ok=True)

    # Generar nombre con timestamp
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    nombre_zip = f"{prefijo}_{timestamp}.zip"
    ruta_zip = directorio_destino / nombre_zip

    try:
        # Crear archivo ZIP con compresión
        archivos_incluidos = 0
        tamano_total = 0

        with zipfile.ZipFile(ruta_zip, "w", zipfile.ZIP_DEFLATED) as zf:
            for archivo in directorio_origen.rglob("*"):
                if archivo.is_file():
                    # Ruta relativa dentro del ZIP
                    ruta_relativa = archivo.relative_to(directorio_origen.parent)
                    zf.write(archivo, ruta_relativa)
                    archivos_incluidos += 1
                    tamano_total += archivo.stat().st_size

        tamano_zip = ruta_zip.stat().st_size
        ratio = (1 - tamano_zip / tamano_total) * 100 if tamano_total > 0 else 0

        logger.info(
            f"Respaldo creado exitosamente: '{nombre_zip}' | "
            f"Archivos: {archivos_incluidos} | "
            f"Tamaño original: {tamano_total / 1024:.1f} KB | "
            f"Tamaño ZIP: {tamano_zip / 1024:.1f} KB | "
            f"Compresión: {ratio:.1f}%"
        )

        return ruta_zip

    except PermissionError as e:
        logger.error(f"Permiso denegado al crear respaldo: {e}")
        # Limpiar archivo parcial si existe
        if ruta_zip.exists():
            ruta_zip.unlink()
        raise BackupError(f"Permiso denegado: {e}", str(ruta_zip))

    except OSError as e:
        logger.error(f"Error del sistema al crear respaldo: {e}")
        if ruta_zip.exists():
            ruta_zip.unlink()
        raise BackupError(f"Error del sistema: {e}", str(ruta_zip))
```

2. Verifica que la función de respaldo funciona correctamente:

```bash
python -c "
from src.backup_manager import crear_respaldo
from pathlib import Path

ruta_zip = crear_respaldo()
print(f'Respaldo creado: {ruta_zip}')
print(f'Tamaño: {ruta_zip.stat().st_size / 1024:.1f} KB')

# Verificar contenido del ZIP
import zipfile
with zipfile.ZipFile(ruta_zip, 'r') as zf:
    print(f'Archivos en el ZIP ({len(zf.namelist())}):')
    for nombre in zf.namelist()[:5]:
        print(f'  {nombre}')
    if len(zf.namelist()) > 5:
        print(f'  ... y {len(zf.namelist()) - 5} más')
"
```

### Salida esperada

```
Respaldo creado: /home/<usuario>/automation_project/backups/backup_data_20250XXX_XXXXXX.zip
Tamaño: X.X KB
Archivos en el ZIP (X):
  data/input/employees.csv
  data/output/reporte_antiguo_0.txt
  ...
```

### Verificación

```bash
ls -la backups/
# Debe mostrar al menos un archivo .zip con timestamp en el nombre
```

---

## Paso 3: Implementar la función de limpieza de archivos antiguos

### Objetivo

Añadir al módulo `backup_manager.py` la función que identifica y elimina archivos con más de N días de antigüedad en `data/output/`.

### Instrucciones

1. Abre `~/automation_project/src/backup_manager.py` y añade la siguiente función después de `crear_respaldo()`:

```python
def limpiar_archivos_antiguos(
    directorio: Path | None = None,
    dias_antiguedad: int = 7,
    extensiones: list[str] | None = None,
    simulacion: bool = False,
) -> dict[str, list[str]]:
    """
    Elimina archivos con más de N días de antigüedad en el directorio especificado.

    Args:
        directorio: Directorio a limpiar (default: data/output/).
        dias_antiguedad: Número de días para considerar un archivo como antiguo.
        extensiones: Lista de extensiones a considerar (e.g., ['.txt', '.csv']).
                     Si es None, se consideran todos los archivos.
        simulacion: Si es True, solo reporta sin eliminar (dry-run).

    Returns:
        Diccionario con listas de archivos 'eliminados' y 'errores'.

    Raises:
        DirectorioNoEncontrado: Si el directorio no existe.
    """
    if directorio is None:
        directorio = DIRECTORIO_OUTPUT

    logger.info(
        f"Iniciando limpieza en '{directorio}' | "
        f"Antigüedad: {dias_antiguedad} días | "
        f"Simulación: {simulacion}"
    )

    if not directorio.exists():
        raise DirectorioNoEncontrado(str(directorio))

    resultado: dict[str, list[str]] = {"eliminados": [], "errores": [], "omitidos": []}
    fecha_limite = datetime.now() - timedelta(days=dias_antiguedad)

    try:
        for archivo in directorio.iterdir():
            if not archivo.is_file():
                continue

            # Filtrar por extensión si se especificó
            if extensiones and archivo.suffix.lower() not in extensiones:
                resultado["omitidos"].append(str(archivo.name))
                continue

            # Verificar antigüedad basada en fecha de modificación
            fecha_modificacion = datetime.fromtimestamp(archivo.stat().st_mtime)

            if fecha_modificacion < fecha_limite:
                if simulacion:
                    logger.info(f"[SIMULACIÓN] Se eliminaría: '{archivo.name}' "
                               f"(modificado: {fecha_modificacion.strftime('%Y-%m-%d')})")
                    resultado["eliminados"].append(str(archivo.name))
                else:
                    try:
                        tamano = archivo.stat().st_size
                        archivo.unlink()
                        logger.info(
                            f"Eliminado: '{archivo.name}' | "
                            f"Tamaño: {tamano} bytes | "
                            f"Modificado: {fecha_modificacion.strftime('%Y-%m-%d %H:%M')}"
                        )
                        resultado["eliminados"].append(str(archivo.name))
                    except PermissionError as e:
                        msg = f"Permiso denegado para '{archivo.name}': {e}"
                        logger.warning(msg)
                        resultado["errores"].append(msg)
                    except OSError as e:
                        msg = f"Error al eliminar '{archivo.name}': {e}"
                        logger.warning(msg)
                        resultado["errores"].append(msg)

    except PermissionError as e:
        raise LimpiezaError(str(directorio), f"Sin permisos para listar directorio: {e}")

    logger.info(
        f"Limpieza completada | "
        f"Eliminados: {len(resultado['eliminados'])} | "
        f"Errores: {len(resultado['errores'])} | "
        f"Omitidos: {len(resultado['omitidos'])}"
    )

    return resultado
```

2. Prueba la función en modo simulación primero:

```bash
python -c "
from src.backup_manager import limpiar_archivos_antiguos

# Modo simulación (no elimina nada)
resultado = limpiar_archivos_antiguos(dias_antiguedad=7, simulacion=True)
print('=== Modo Simulación ===')
print(f'Archivos que se eliminarían: {len(resultado[\"eliminados\"])}')
for f in resultado['eliminados']:
    print(f'  - {f}')
"
```

### Salida esperada

```
=== Modo Simulación ===
Archivos que se eliminarían: 5
  - reporte_antiguo_0.txt
  - reporte_antiguo_1.txt
  - reporte_antiguo_2.txt
  - reporte_antiguo_3.txt
  - reporte_antiguo_4.txt
```

3. Ejecuta la limpieza real:

```bash
python -c "
from src.backup_manager import limpiar_archivos_antiguos

# Ejecución real
resultado = limpiar_archivos_antiguos(dias_antiguedad=7, simulacion=False)
print(f'Archivos eliminados: {len(resultado[\"eliminados\"])}')
print(f'Errores: {len(resultado[\"errores\"])}')

# Verificar que los archivos recientes permanecen
from pathlib import Path
restantes = list(Path('data/output').iterdir())
print(f'Archivos restantes en data/output/: {len(restantes)}')
for f in restantes:
    print(f'  {f.name}')
"
```

### Salida esperada

```
Archivos eliminados: 5
Errores: 0
Archivos restantes en data/output/: 3
  reporte_reciente_0.txt
  reporte_reciente_1.txt
  reporte_reciente_2.txt
```

### Verificación

```bash
# Los archivos antiguos deben haber sido eliminados
ls data/output/
# Solo deben aparecer los archivos reporte_reciente_*.txt
```

---

## Paso 4: Implementar verificación de espacio en disco con subprocess

### Objetivo

Añadir una función que ejecute comandos del sistema operativo para verificar el espacio disponible en disco, usando `subprocess` con captura de salida y manejo de códigos de retorno.

### Instrucciones

1. Añade la siguiente función a `~/automation_project/src/backup_manager.py`:

```python
def verificar_espacio_disco(
    ruta: Path | None = None,
    espacio_minimo_mb: float = 100.0,
) -> dict[str, float | bool | str]:
    """
    Verifica el espacio disponible en disco usando subprocess.

    Ejecuta comandos del sistema operativo (df en Unix, wmic en Windows)
    para obtener información de espacio en disco.

    Args:
        ruta: Ruta del filesystem a verificar (default: raíz del proyecto).
        espacio_minimo_mb: Espacio mínimo requerido en MB.

    Returns:
        Diccionario con información de espacio en disco.

    Raises:
        EspacioInsuficiente: Si el espacio disponible es menor al mínimo.
        BackupError: Si falla la ejecución del comando del sistema.
    """
    if ruta is None:
        ruta = PROYECTO_RAIZ

    logger.info(f"Verificando espacio en disco para '{ruta}'")

    sistema = platform.system()

    try:
        if sistema in ("Linux", "Darwin"):  # Linux o macOS
            resultado = subprocess.run(
                ["df", "-BM", str(ruta)],
                capture_output=True,
                text=True,
                timeout=10,
            )

            if resultado.returncode != 0:
                logger.error(f"Error al ejecutar 'df': {resultado.stderr}")
                raise BackupError(
                    f"Comando 'df' falló con código {resultado.returncode}: {resultado.stderr}"
                )

            # Parsear salida de df
            lineas = resultado.stdout.strip().split("\n")
            if len(lineas) >= 2:
                # La segunda línea contiene los datos
                partes = lineas[1].split()
                # df -BM muestra: Filesystem Size Used Avail Use% Mounted
                disponible_mb = float(partes[3].replace("M", ""))
                total_mb = float(partes[1].replace("M", ""))
                usado_mb = float(partes[2].replace("M", ""))
            else:
                raise BackupError("Salida inesperada del comando 'df'")

        elif sistema == "Windows":
            # Usar shutil.disk_usage como alternativa portable
            uso = shutil.disk_usage(ruta)
            total_mb = uso.total / (1024 * 1024)
            usado_mb = uso.used / (1024 * 1024)
            disponible_mb = uso.free / (1024 * 1024)

            # También ejecutar subprocess para demostrar la técnica
            resultado = subprocess.run(
                ["wmic", "logicaldisk", "get", "freespace,size,caption"],
                capture_output=True,
                text=True,
                timeout=10,
            )
            logger.info(f"Salida de wmic:\n{resultado.stdout[:200]}")

        else:
            # Fallback usando shutil para cualquier otro sistema
            uso = shutil.disk_usage(ruta)
            total_mb = uso.total / (1024 * 1024)
            usado_mb = uso.used / (1024 * 1024)
            disponible_mb = uso.free / (1024 * 1024)

    except subprocess.TimeoutExpired:
        logger.error("Timeout al verificar espacio en disco")
        raise BackupError("Timeout al ejecutar comando de verificación de espacio")

    except FileNotFoundError as e:
        logger.warning(f"Comando no encontrado: {e}. Usando shutil.disk_usage como fallback.")
        uso = shutil.disk_usage(ruta)
        total_mb = uso.total / (1024 * 1024)
        usado_mb = uso.used / (1024 * 1024)
        disponible_mb = uso.free / (1024 * 1024)

    porcentaje_usado = (usado_mb / total_mb) * 100 if total_mb > 0 else 0
    suficiente = disponible_mb >= espacio_minimo_mb

    info_disco = {
        "total_mb": round(total_mb, 1),
        "usado_mb": round(usado_mb, 1),
        "disponible_mb": round(disponible_mb, 1),
        "porcentaje_usado": round(porcentaje_usado, 1),
        "suficiente": suficiente,
        "sistema": sistema,
    }

    logger.info(
        f"Espacio en disco — Total: {info_disco['total_mb']} MB | "
        f"Usado: {info_disco['porcentaje_usado']}% | "
        f"Disponible: {info_disco['disponible_mb']} MB | "
        f"Suficiente: {suficiente}"
    )

    if not suficiente:
        raise EspacioInsuficiente(disponible_mb, espacio_minimo_mb)

    return info_disco
```

2. Prueba la función:

```bash
python -c "
from src.backup_manager import verificar_espacio_disco

info = verificar_espacio_disco(espacio_minimo_mb=50.0)
print('=== Información de Espacio en Disco ===')
for clave, valor in info.items():
    print(f'  {clave}: {valor}')
"
```

### Salida esperada

```
=== Información de Espacio en Disco ===
  total_mb: XXXXX.X
  usado_mb: XXXXX.X
  disponible_mb: XXXXX.X
  porcentaje_usado: XX.X
  suficiente: True
  sistema: Linux
```

### Verificación

```bash
# Probar que la excepción se lanza correctamente con un umbral imposible
python -c "
from src.backup_manager import verificar_espacio_disco
from src.backup_exceptions import EspacioInsuficiente

try:
    verificar_espacio_disco(espacio_minimo_mb=999999999)
except EspacioInsuficiente as e:
    print(f'Excepción capturada correctamente: {e}')
"
```

---

## Paso 5: Implementar la función de reporte de estado

### Objetivo

Crear una función que genere un reporte consolidado de las operaciones realizadas, escribiéndolo tanto en el log como en un archivo de texto en `logs/`.

### Instrucciones

1. Añade la siguiente función a `~/automation_project/src/backup_manager.py`:

```python
def generar_reporte_estado(
    resultado_respaldo: Path | None = None,
    resultado_limpieza: dict[str, list[str]] | None = None,
    info_disco: dict | None = None,
    ruta_reporte: Path | None = None,
) -> Path:
    """
    Genera un reporte de estado con los resultados de todas las operaciones.

    Args:
        resultado_respaldo: Ruta al archivo ZIP creado.
        resultado_limpieza: Diccionario con resultados de limpieza.
        info_disco: Información de espacio en disco.
        ruta_reporte: Ruta donde guardar el reporte (default: logs/).

    Returns:
        Path al archivo de reporte generado.
    """
    if ruta_reporte is None:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        ruta_reporte = DIRECTORIO_LOGS / f"reporte_estado_{timestamp}.txt"

    logger.info("Generando reporte de estado...")

    lineas: list[str] = []
    lineas.append("=" * 60)
    lineas.append("  REPORTE DE ESTADO — BACKUP MANAGER")
    lineas.append("=" * 60)
    lineas.append(f"  Fecha: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    lineas.append(f"  Proyecto: {PROYECTO_RAIZ}")
    lineas.append("")

    # Sección: Respaldo
    lineas.append("-" * 60)
    lineas.append("  RESPALDO")
    lineas.append("-" * 60)
    if resultado_respaldo and resultado_respaldo.exists():
        tamano = resultado_respaldo.stat().st_size
        lineas.append(f"  Estado: EXITOSO")
        lineas.append(f"  Archivo: {resultado_respaldo.name}")
        lineas.append(f"  Tamaño: {tamano / 1024:.1f} KB")
        lineas.append(f"  Ruta: {resultado_respaldo}")
    else:
        lineas.append(f"  Estado: NO EJECUTADO o FALLIDO")
    lineas.append("")

    # Sección: Limpieza
    lineas.append("-" * 60)
    lineas.append("  LIMPIEZA DE ARCHIVOS")
    lineas.append("-" * 60)
    if resultado_limpieza:
        lineas.append(f"  Archivos eliminados: {len(resultado_limpieza['eliminados'])}")
        for f in resultado_limpieza["eliminados"]:
            lineas.append(f"    - {f}")
        if resultado_limpieza["errores"]:
            lineas.append(f"  Errores: {len(resultado_limpieza['errores'])}")
            for e in resultado_limpieza["errores"]:
                lineas.append(f"    ! {e}")
    else:
        lineas.append("  Estado: NO EJECUTADO")
    lineas.append("")

    # Sección: Disco
    lineas.append("-" * 60)
    lineas.append("  ESPACIO EN DISCO")
    lineas.append("-" * 60)
    if info_disco:
        lineas.append(f"  Total: {info_disco['total_mb']} MB")
        lineas.append(f"  Usado: {info_disco['usado_mb']} MB ({info_disco['porcentaje_usado']}%)")
        lineas.append(f"  Disponible: {info_disco['disponible_mb']} MB")
        lineas.append(f"  Suficiente: {'SÍ' if info_disco['suficiente'] else 'NO'}")
    else:
        lineas.append("  Estado: NO VERIFICADO")
    lineas.append("")
    lineas.append("=" * 60)

    # Escribir reporte a archivo
    contenido = "\n".join(lineas)

    with open(ruta_reporte, "w", encoding="utf-8") as f:
        f.write(contenido)

    logger.info(f"Reporte generado: '{ruta_reporte}'")

    return ruta_reporte
```

2. No pruebes esta función de forma aislada todavía — se integrará en el paso siguiente con la función principal.

---

## Paso 6: Crear la función principal con orquestación completa

### Objetivo

Implementar la función `ejecutar()` que orquesta todas las operaciones con manejo de excepciones robusto usando `try/except/finally`, y un bloque `if __name__ == "__main__"` con argumentos de línea de comandos.

### Instrucciones

1. Añade al final de `~/automation_project/src/backup_manager.py`:

```python
def ejecutar(
    dias_antiguedad: int = 7,
    espacio_minimo_mb: float = 100.0,
    simulacion_limpieza: bool = False,
) -> dict[str, any]:
    """
    Función principal que orquesta todas las operaciones de respaldo y limpieza.

    Ejecuta en orden:
    1. Verificación de espacio en disco.
    2. Creación de respaldo comprimido.
    3. Limpieza de archivos antiguos.
    4. Generación de reporte de estado.

    Args:
        dias_antiguedad: Días para considerar un archivo como antiguo.
        espacio_minimo_mb: Espacio mínimo requerido en disco (MB).
        simulacion_limpieza: Si es True, la limpieza no elimina archivos.

    Returns:
        Diccionario con resultados de cada operación.
    """
    logger.info("=" * 50)
    logger.info("INICIO DE EJECUCIÓN — BACKUP MANAGER")
    logger.info("=" * 50)
    logger.info(
        f"Parámetros: dias_antiguedad={dias_antiguedad}, "
        f"espacio_minimo_mb={espacio_minimo_mb}, "
        f"simulacion={simulacion_limpieza}"
    )

    resultados: dict[str, any] = {
        "exito": False,
        "respaldo": None,
        "limpieza": None,
        "disco": None,
        "reporte": None,
        "errores": [],
    }

    try:
        # Paso 1: Verificar espacio en disco
        logger.info("--- Paso 1/4: Verificando espacio en disco ---")
        try:
            info_disco = verificar_espacio_disco(espacio_minimo_mb=espacio_minimo_mb)
            resultados["disco"] = info_disco
        except EspacioInsuficiente as e:
            logger.warning(f"Espacio insuficiente, pero se continuará: {e}")
            resultados["errores"].append(str(e))
            # Usamos shutil como fallback para obtener info sin lanzar excepción
            import shutil as _shutil
            uso = _shutil.disk_usage(PROYECTO_RAIZ)
            info_disco = {
                "total_mb": round(uso.total / (1024**2), 1),
                "usado_mb": round(uso.used / (1024**2), 1),
                "disponible_mb": round(uso.free / (1024**2), 1),
                "porcentaje_usado": round(uso.used / uso.total * 100, 1),
                "suficiente": False,
                "sistema": platform.system(),
            }
            resultados["disco"] = info_disco

        # Paso 2: Crear respaldo
        logger.info("--- Paso 2/4: Creando respaldo comprimido ---")
        try:
            ruta_zip = crear_respaldo()
            resultados["respaldo"] = ruta_zip
        except (DirectorioNoEncontrado, BackupError) as e:
            logger.error(f"Error al crear respaldo: {e}")
            resultados["errores"].append(str(e))
            ruta_zip = None

        # Paso 3: Limpiar archivos antiguos
        logger.info("--- Paso 3/4: Limpiando archivos antiguos ---")
        try:
            resultado_limpieza = limpiar_archivos_antiguos(
                dias_antiguedad=dias_antiguedad,
                simulacion=simulacion_limpieza,
            )
            resultados["limpieza"] = resultado_limpieza
        except (DirectorioNoEncontrado, LimpiezaError) as e:
            logger.error(f"Error en limpieza: {e}")
            resultados["errores"].append(str(e))
            resultado_limpieza = None

        # Paso 4: Generar reporte
        logger.info("--- Paso 4/4: Generando reporte de estado ---")
        ruta_reporte = generar_reporte_estado(
            resultado_respaldo=ruta_zip,
            resultado_limpieza=resultado_limpieza,
            info_disco=info_disco,
        )
        resultados["reporte"] = ruta_reporte
        resultados["exito"] = len(resultados["errores"]) == 0

    except Exception as e:
        logger.critical(f"Error inesperado: {type(e).__name__}: {e}")
        resultados["errores"].append(f"Error crítico: {e}")

    finally:
        estado = "EXITOSO" if resultados["exito"] else "CON ERRORES"
        logger.info(f"FIN DE EJECUCIÓN — Estado: {estado}")
        logger.info(f"Errores totales: {len(resultados['errores'])}")
        logger.info("=" * 50)

    return resultados


if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(
        description="Automatización de respaldos y limpieza de archivos"
    )
    parser.add_argument(
        "--dias-antiguedad",
        type=int,
        default=7,
        help="Días de antigüedad para considerar un archivo como eliminable (default: 7)",
    )
    parser.add_argument(
        "--min-espacio-mb",
        type=float,
        default=100.0,
        help="Espacio mínimo requerido en disco en MB (default: 100)",
    )
    parser.add_argument(
        "--simulacion",
        action="store_true",
        help="Ejecutar limpieza en modo simulación (no elimina archivos)",
    )

    args = parser.parse_args()

    resultados = ejecutar(
        dias_antiguedad=args.dias_antiguedad,
        espacio_minimo_mb=args.min_espacio_mb,
        simulacion_limpieza=args.simulacion,
    )

    # Resumen final en consola
    print("\n" + "=" * 50)
    print("  RESUMEN DE EJECUCIÓN")
    print("=" * 50)
    print(f"  Estado: {'✓ EXITOSO' if resultados['exito'] else '✗ CON ERRORES'}")

    if resultados["respaldo"]:
        print(f"  Respaldo: {resultados['respaldo'].name}")

    if resultados["limpieza"]:
        print(f"  Archivos limpiados: {len(resultados['limpieza']['eliminados'])}")

    if resultados["disco"]:
        print(f"  Disco disponible: {resultados['disco']['disponible_mb']} MB")

    if resultados["reporte"]:
        print(f"  Reporte: {resultados['reporte'].name}")

    if resultados["errores"]:
        print(f"\n  Errores ({len(resultados['errores'])}):")
        for err in resultados["errores"]:
            print(f"    ! {err}")

    print("=" * 50)
```

2. Antes de ejecutar, recrea los archivos de prueba antiguos (fueron eliminados en el Paso 3):

```bash
python -c "
from pathlib import Path
import time, os

output_dir = Path('data/output')
for i in range(5):
    archivo = output_dir / f'reporte_antiguo_{i}.txt'
    archivo.write_text(f'Contenido de prueba {i}', encoding='utf-8')
    antiguo = time.time() - (10 * 86400)
    os.utime(archivo, (antiguo, antiguo))
print('Archivos de prueba recreados.')
"
```

3. Ejecuta el script completo:

```bash
cd ~/automation_project
python -m src.backup_manager --dias-antiguedad 7 --simulacion
```

### Salida esperada

```
==================================================
  RESUMEN DE EJECUCIÓN
==================================================
  Estado: ✓ EXITOSO
  Respaldo: backup_data_20250XXX_XXXXXX.zip
  Archivos limpiados: 5
  Disco disponible: XXXXX.X MB
  Reporte: reporte_estado_20250XXX_XXXXXX.txt
==================================================
```

4. Ahora ejecuta sin simulación para eliminar los archivos realmente:

```bash
python -m src.backup_manager --dias-antiguedad 7
```

### Verificación

```bash
# Verificar que el log contiene todas las operaciones
echo "=== Últimas 20 líneas del log ==="
tail -20 logs/backup_manager.log

# Verificar que existe el reporte
ls logs/reporte_estado_*.txt

# Verificar que los archivos antiguos fueron eliminados
ls data/output/

# Verificar que el backup existe
ls -la backups/
```

---

## Paso 7: Añadir el archivo `__init__.py` y verificar imports

### Objetivo

Asegurar que el módulo `src` es importable como paquete y que `backup_manager.py` puede importar correctamente las dependencias del Lab 3.

### Instrucciones

1. Verifica que existe `~/automation_project/src/__init__.py`. Si no existe, créalo:

```bash
touch src/__init__.py
```

2. Si no tienes `file_utils.py` y `system_utils.py` del Lab 3, crea stubs mínimos para que los imports no fallen. Verifica con:

```bash
python -c "
import src.backup_manager as bm
print('Módulo importado correctamente')
print(f'Funciones disponibles:')
for nombre in dir(bm):
    if not nombre.startswith('_') and callable(getattr(bm, nombre)):
        print(f'  - {nombre}')
"
```

### Salida esperada

```
Módulo importado correctamente
Funciones disponibles:
  - configurar_logger
  - crear_respaldo
  - ejecutar
  - generar_reporte_estado
  - limpiar_archivos_antiguos
  - verificar_espacio_disco
```

---

## Validación y pruebas

Ejecuta la siguiente secuencia completa de validación para confirmar que todo funciona correctamente:

```bash
cd ~/automation_project

# 1. Verificar estructura de archivos
echo "=== Estructura de archivos ==="
find src -name "backup_*" -type f | sort
echo ""

# 2. Ejecutar el script con ayuda
python -m src.backup_manager --help

# 3. Ejecutar una ejecución completa en modo simulación
echo ""
echo "=== Ejecución en modo simulación ==="
python -m src.backup_manager --dias-antiguedad 5 --simulacion

# 4. Verificar contenido del log
echo ""
echo "=== Verificación del log ==="
wc -l logs/backup_manager.log
grep "INICIO DE EJECUCIÓN" logs/backup_manager.log | tail -1
grep "FIN DE EJECUCIÓN" logs/backup_manager.log | tail -1

# 5. Verificar que el ZIP es válido
echo ""
echo "=== Verificación del ZIP ==="
python -c "
import zipfile
from pathlib import Path

zips = sorted(Path('backups').glob('*.zip'))
if zips:
    ultimo = zips[-1]
    with zipfile.ZipFile(ultimo, 'r') as zf:
        resultado = zf.testzip()
        if resultado is None:
            print(f'ZIP válido: {ultimo.name} ({len(zf.namelist())} archivos)')
        else:
            print(f'ZIP corrupto en: {resultado}')
else:
    print('No se encontraron archivos ZIP')
"

# 6. Verificar reporte de estado
echo ""
echo "=== Último reporte de estado ==="
cat $(ls -t logs/reporte_estado_*.txt | head -1)
```

**Criterios de éxito:**

- [x] El script se ejecuta sin errores
- [x] Se crea un archivo `.zip` válido en `backups/`
- [x] El log registra todas las operaciones con timestamps
- [x] Las excepciones personalizadas funcionan correctamente
- [x] El reporte de estado contiene las tres secciones (respaldo, limpieza, disco)
- [x] El modo simulación no elimina archivos

---

## Solución de problemas

### Problema 1: `ModuleNotFoundError: No module named 'src'`

**Síntomas:** Al ejecutar `python -m src.backup_manager`, Python no encuentra el módulo `src`.

**Causa:** El directorio de trabajo actual no es `~/automation_project/`, o falta el archivo `src/__init__.py`. Python necesita estar en el directorio padre del paquete `src` para que la resolución de módulos funcione correctamente.

**Solución:**

```bash
# Verificar directorio actual
pwd
# Debe mostrar: /home/<usuario>/automation_project

# Si no estás en el directorio correcto:
cd ~/automation_project

# Verificar que __init__.py existe
ls src/__init__.py

# Si no existe, crearlo:
touch src/__init__.py

# Alternativa: ejecutar con PYTHONPATH explícito
PYTHONPATH=. python -m src.backup_manager
```

### Problema 2: `PermissionError` al crear el archivo ZIP o eliminar archivos

**Síntomas:** El script falla con `PermissionError: [Errno 13] Permission denied` al intentar escribir en `backups/` o eliminar archivos en `data/output/`.

**Causa:** Los permisos del directorio o archivo no permiten escritura/eliminación al usuario actual. Esto puede ocurrir si los archivos fueron creados por otro usuario o proceso, o si el sistema de archivos tiene restricciones.

**Solución:**

```bash
# Verificar permisos de los directorios
ls -la backups/
ls -la data/output/

# Corregir permisos (macOS/Linux)
chmod 755 backups/
chmod 755 data/output/
chmod 644 data/output/*

# En Windows (PowerShell como administrador):
# icacls "backups" /grant "%USERNAME%:F"
# icacls "data\output" /grant "%USERNAME%:F" /T

# Verificar que el script maneja el error correctamente
python -c "
from src.backup_manager import ejecutar
resultados = ejecutar(simulacion_limpieza=True)
print(f'Errores capturados: {resultados[\"errores\"]}')
"
```

---

## Limpieza

Después de completar el laboratorio, puedes limpiar los archivos de prueba que ya no necesitas, pero **conserva** los archivos del módulo y al menos un respaldo para el Lab 5:

```bash
cd ~/automation_project

# Eliminar archivos de prueba temporales (conservar los recientes como ejemplo)
rm -f data/output/reporte_antiguo_*.txt

# Conservar solo el último respaldo (eliminar los de prueba)
ls -t backups/*.zip | tail -n +2 | xargs rm -f 2>/dev/null

# Verificar estado final
echo "=== Estado final del proyecto ==="
echo "Backups:"
ls backups/
echo ""
echo "Logs:"
ls logs/
echo ""
echo "Módulos creados:"
ls src/backup_*.py
```

**Archivos que deben permanecer para el Lab 5:**

- `src/backup_manager.py`
- `src/backup_exceptions.py`
- `src/backup_logger.py`
- `backups/` (con al menos un archivo `.zip`)
- `logs/backup_manager.log`

---

## Resumen

En este laboratorio has construido un sistema completo de automatización de respaldos que integra múltiples módulos de la biblioteca estándar de Python:

| Módulo | Uso en el laboratorio |
|--------|----------------------|
| `zipfile` | Creación de archivos comprimidos con `ZIP_DEFLATED` |
| `shutil` | Verificación de espacio en disco como fallback |
| `pathlib` | Manipulación portable de rutas y directorios |
| `subprocess` | Ejecución de comandos del SO (`df`, `wmic`) |
| `logging` | Registro estructurado con `RotatingFileHandler` |
| `datetime` | Timestamps y cálculo de antigüedad de archivos |

**Conceptos clave aplicados:**

- **Excepciones personalizadas** con herencia para categorizar errores de forma semántica
- **Logging con rotación** que evita el crecimiento indefinido de archivos de log
- **Patrón try/except/finally** para garantizar la ejecución del bloque de cierre independientemente del resultado
- **Modo simulación (dry-run)** como práctica de seguridad antes de operaciones destructivas
- **Portabilidad multiplataforma** mediante detección de `platform.system()` y fallbacks

### Recursos adicionales

- [Documentación oficial — módulo `zipfile`](https://docs.python.org/3/library/zipfile.html)
- [Documentación oficial — módulo `subprocess`](https://docs.python.org/3/library/subprocess.html)
- [Documentación oficial — `logging.handlers.RotatingFileHandler`](https://docs.python.org/3/library/logging.handlers.html#rotatingfilehandler)
- [Documentación oficial — módulo `pathlib`](https://docs.python.org/3/library/pathlib.html)
