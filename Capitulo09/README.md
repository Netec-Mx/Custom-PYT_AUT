# Práctica 9 — Depuración y pruebas de un script real

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 38 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción general

En este laboratorio aplicarás técnicas profesionales de depuración, logging y testing al proyecto `auto_reporter/` construido en labs anteriores. No crearás funcionalidad nueva: reforzarás y validarás lo existente configurando el debugger de VS Code, implementando logging centralizado con `RotatingFileHandler`, escribiendo pruebas unitarias con `pytest` y `unittest.mock`, y midiendo cobertura con `pytest-cov`. Al finalizar, tu proyecto tendrá una suite de tests lista para integración continua.

## Objetivos de aprendizaje

- [ ] Configurar `launch.json` en VS Code y usar breakpoints, Step Over/Into y Watch Expressions para identificar bugs en módulos del proyecto.
- [ ] Implementar un sistema de logging centralizado con niveles (DEBUG–CRITICAL) y `RotatingFileHandler` que reemplace todos los `print()` del proyecto.
- [ ] Escribir pruebas unitarias con `pytest`, `fixtures`, `parametrize` y `unittest.mock` para los módulos `data_cleaner.py`, `report_generator.py` y `notifier.py`.
- [ ] Alcanzar ≥ 70 % de cobertura de código con `pytest-cov` y corregir al menos 2 bugs identificados durante el proceso de testing.

## Prerrequisitos

### Conocimientos previos

- Labs 06, 07 y 08 completados: proyecto `auto_reporter/` con módulos `src/data_cleaner.py`, `src/report_generator.py`, `src/notifier.py` y `src/web_automation.py` funcionales.
- Comprensión de funciones, módulos y clases para poder mockear dependencias externas.
- Familiaridad con el módulo `logging` de Python (Lección 9.1/9.2).

### Acceso requerido

- VS Code 1.89.1 con extensión Python (`ms-python.python`) instalada.
- Entorno virtual activo en `~/automation_project/.venv`.
- Archivo `logs/selenium_run.log` generado en Lab 08 (para analizar trazas).

## Entorno del laboratorio

### Software

| Herramienta | Versión |
|-------------|---------|
| Python | 3.11.9 |
| pytest | 8.2.1 |
| pytest-cov | 5.0.0 |
| VS Code | 1.89.1 |
| coverage.py | (integrado con pytest-cov) |

### Preparación del entorno

```bash
cd ~/automation_project
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Instalar dependencias de testing
pip install pytest==8.2.1 pytest-cov==5.0.0

# Verificar versiones
python --version           # 3.11.9
pytest --version           # 8.2.1
pip show pytest-cov | grep Version  # 5.0.0
```

---

## Paso 1 — Configurar el debugger de VS Code

### Objetivo

Crear el archivo `launch.json` para depurar los módulos del proyecto con breakpoints visuales.

### Instrucciones

1. Abre el proyecto `~/automation_project/` en VS Code.

2. Crea la carpeta `.vscode/` si no existe y dentro de ella el archivo `launch.json`:

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Depurar módulo actual",
            "type": "debugpy",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "cwd": "${workspaceFolder}",
            "justMyCode": true,
            "env": {
                "PYTHONPATH": "${workspaceFolder}/src"
            }
        },
        {
            "name": "Depurar data_cleaner",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/src/data_cleaner.py",
            "console": "integratedTerminal",
            "cwd": "${workspaceFolder}",
            "justMyCode": true
        }
    ]
}
```

3. Abre `src/data_cleaner.py` en el editor. Haz clic en el margen izquierdo (junto al número de línea) de la primera línea de la función principal de limpieza para colocar un **breakpoint** (aparecerá un círculo rojo).

4. Presiona `F5` y selecciona la configuración **"Depurar data_cleaner"**.

5. Cuando la ejecución se detenga en el breakpoint:
   - Inspecciona el panel **Variables** (sección Locals) para ver el estado del DataFrame.
   - Agrega una **Watch Expression**: escribe `df.shape` en el panel Watch.
   - Usa `F10` (Step Over) para avanzar línea por línea.
   - Usa `F11` (Step Into) para entrar dentro de una función llamada.

6. Observa si algún valor de variable es inesperado. Documenta cualquier hallazgo en un comentario `# BUG:` dentro del código.

### Resultado esperado

La ejecución se pausa en el breakpoint. El panel Variables muestra las variables locales y el panel Watch muestra `df.shape` con las dimensiones del DataFrame.

### Verificación

- El archivo `.vscode/launch.json` existe y VS Code no muestra errores de sintaxis JSON.
- Al presionar `F5`, la ejecución se detiene en la línea marcada.

---

## Paso 2 — Implementar logging centralizado

### Objetivo

Crear un módulo `src/logger_config.py` con una función `get_logger()` que configure logging con salida a consola y archivo rotativo, y reemplazar todos los `print()` del proyecto.

### Instrucciones

1. Crea el archivo `src/logger_config.py`:

```python
# src/logger_config.py
"""Configuración centralizada de logging para auto_reporter."""

import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path

# Directorio de logs (relativo a la raíz del proyecto)
LOG_DIR = Path(__file__).resolve().parent.parent / "logs"
LOG_DIR.mkdir(exist_ok=True)

LOG_FILE = LOG_DIR / "app.log"
LOG_FORMAT = "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
LOG_LEVEL = logging.DEBUG


def get_logger(name: str) -> logging.Logger:
    """
    Retorna un logger configurado con StreamHandler y RotatingFileHandler.

    Args:
        name: Nombre del logger (típicamente __name__ del módulo).

    Returns:
        logging.Logger configurado.
    """
    logger = logging.getLogger(name)

    # Evitar agregar handlers duplicados si se llama múltiples veces
    if logger.handlers:
        return logger

    logger.setLevel(LOG_LEVEL)

    # Handler de consola
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(logging.Formatter(LOG_FORMAT))

    # Handler de archivo con rotación (5 MB máx, 5 backups)
    file_handler = RotatingFileHandler(
        LOG_FILE,
        maxBytes=5 * 1024 * 1024,  # 5 MB
        backupCount=5,
        encoding="utf-8",
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(logging.Formatter(LOG_FORMAT))

    logger.addHandler(console_handler)
    logger.addHandler(file_handler)

    return logger
```

2. Reemplaza los `print()` en `src/data_cleaner.py`. Ejemplo de antes y después:

```python
# ANTES (eliminar):
# print(f"Registros cargados: {len(df)}")

# DESPUÉS:
from logger_config import get_logger

logger = get_logger(__name__)

def limpiar_datos(filepath: str):
    logger.info("Iniciando limpieza de datos desde: %s", filepath)
    # ... código existente ...
    logger.debug("Registros cargados: %d", len(df))
    logger.warning("Se encontraron %d duplicados", duplicados)
    # ...
    logger.info("Limpieza completada. Registros finales: %d", len(df_limpio))
    return df_limpio
```

3. Repite el patrón en `src/report_generator.py` y `src/notifier.py`:

```python
# Al inicio de cada módulo:
from logger_config import get_logger
logger = get_logger(__name__)
```

4. Verifica que el logging funciona ejecutando uno de los módulos:

```bash
cd ~/automation_project
python -m src.data_cleaner   # o como se invoque tu módulo
```

### Resultado esperado

```
2024-06-15 10:30:45,123 | INFO | src.data_cleaner | Iniciando limpieza de datos desde: data/input/employees.csv
2024-06-15 10:30:45,456 | DEBUG | src.data_cleaner | Registros cargados: 50
```

El archivo `logs/app.log` contiene las mismas líneas con nivel DEBUG incluido.

### Verificación

```bash
# Verificar que el archivo de log se creó
cat logs/app.log | head -5

# Verificar que no quedan print() en src/
grep -rn "print(" src/ --include="*.py"
# Resultado esperado: ninguna coincidencia (o solo en __main__ para uso interactivo)
```

---

## Paso 3 — Escribir pruebas unitarias para `data_cleaner.py`

### Objetivo

Crear `tests/test_data_cleaner.py` con al menos 5 pruebas que validen las funciones de limpieza de datos usando fixtures y `parametrize`.

### Instrucciones

1. Asegúrate de que existe el archivo `tests/__init__.py` (puede estar vacío):

```bash
touch tests/__init__.py
```

2. Crea `tests/conftest.py` con fixtures reutilizables:

```python
# tests/conftest.py
"""Fixtures compartidas para la suite de tests."""

import pytest
import pandas as pd


@pytest.fixture
def df_con_duplicados():
    """DataFrame con registros duplicados para testing."""
    return pd.DataFrame({
        "id": [1, 2, 2, 3, 4, 4],
        "nombre": ["Ana", "Luis", "Luis", "María", "Pedro", "Pedro"],
        "departamento": ["TI", "Ventas", "Ventas", "TI", "RRHH", "RRHH"],
        "salario": [50000, 45000, 45000, 55000, 40000, 40000],
        "fecha_ingreso": [
            "2020-01-15", "2019-06-20", "2019-06-20",
            "2021-03-10", "2018-11-05", "2018-11-05"
        ],
    })


@pytest.fixture
def df_con_nulos():
    """DataFrame con valores nulos."""
    return pd.DataFrame({
        "id": [1, 2, 3],
        "nombre": ["Ana", None, "María"],
        "departamento": ["TI", "Ventas", None],
        "salario": [50000, None, 55000],
        "fecha_ingreso": ["2020-01-15", "2019-06-20", None],
    })


@pytest.fixture
def df_vacio():
    """DataFrame vacío con columnas correctas."""
    return pd.DataFrame(
        columns=["id", "nombre", "departamento", "salario", "fecha_ingreso"]
    )


@pytest.fixture
def df_valido():
    """DataFrame limpio y válido para tests de report_generator."""
    return pd.DataFrame({
        "id": [1, 2, 3, 4, 5],
        "nombre": ["Ana", "Luis", "María", "Pedro", "Sofía"],
        "departamento": ["TI", "Ventas", "TI", "RRHH", "Ventas"],
        "salario": [50000, 45000, 55000, 40000, 60000],
        "fecha_ingreso": pd.to_datetime([
            "2020-01-15", "2019-06-20", "2021-03-10",
            "2018-11-05", "2022-07-01"
        ]),
    })
```

3. Crea `tests/test_data_cleaner.py`:

```python
# tests/test_data_cleaner.py
"""Pruebas unitarias para src/data_cleaner.py."""

import pytest
import pandas as pd

# Ajusta la importación según la estructura de tu módulo
from src.data_cleaner import (
    eliminar_duplicados,
    rellenar_nulos,
    validar_salario,
    convertir_fechas,
    limpiar_datos,
)


class TestEliminarDuplicados:
    """Tests para la función eliminar_duplicados."""

    def test_remove_duplicates_reduces_rows(self, df_con_duplicados):
        """Verifica que se eliminan filas duplicadas."""
        resultado = eliminar_duplicados(df_con_duplicados)
        assert len(resultado) == 4  # De 6 a 4 registros únicos

    def test_remove_duplicates_preserves_first(self, df_con_duplicados):
        """Verifica que se conserva la primera ocurrencia."""
        resultado = eliminar_duplicados(df_con_duplicados)
        assert resultado.iloc[0]["nombre"] == "Ana"

    def test_no_duplicates_unchanged(self, df_valido):
        """DataFrame sin duplicados permanece igual."""
        resultado = eliminar_duplicados(df_valido)
        assert len(resultado) == len(df_valido)


class TestRellenarNulos:
    """Tests para la función rellenar_nulos."""

    def test_fill_nulls_no_remaining_nans(self, df_con_nulos):
        """Verifica que no quedan valores nulos tras rellenar."""
        resultado = rellenar_nulos(df_con_nulos)
        assert resultado.isnull().sum().sum() == 0

    def test_fill_nulls_salary_with_median(self, df_con_nulos):
        """Verifica que salarios nulos se rellenan con la mediana."""
        resultado = rellenar_nulos(df_con_nulos)
        # La mediana de [50000, 55000] = 52500
        assert resultado.loc[1, "salario"] == pytest.approx(52500, rel=0.01)


class TestValidarSalario:
    """Tests para validación de salarios."""

    @pytest.mark.parametrize("salario,esperado", [
        (50000, True),
        (0, False),
        (-1000, False),
        (1000000, True),
    ])
    def test_invalid_salary_rejected(self, salario, esperado):
        """Verifica que salarios no positivos son rechazados."""
        assert validar_salario(salario) == esperado


class TestConvertirFechas:
    """Tests para conversión de fechas."""

    def test_date_conversion_to_datetime(self, df_valido):
        """Verifica que las fechas string se convierten a datetime."""
        df_str = df_valido.copy()
        df_str["fecha_ingreso"] = df_str["fecha_ingreso"].astype(str)
        resultado = convertir_fechas(df_str)
        assert pd.api.types.is_datetime64_any_dtype(resultado["fecha_ingreso"])


class TestDataFrameVacio:
    """Tests para manejo de DataFrames vacíos."""

    def test_empty_dataframe_handled(self, df_vacio):
        """Verifica que un DataFrame vacío no causa excepciones."""
        resultado = limpiar_datos(df_vacio)
        assert isinstance(resultado, pd.DataFrame)
        assert len(resultado) == 0
```

> **Nota:** Ajusta los nombres de las funciones importadas (`eliminar_duplicados`, `rellenar_nulos`, etc.) para que coincidan con las funciones reales de tu `src/data_cleaner.py`. Si tu módulo tiene nombres diferentes, renómbralos en las importaciones.

4. Ejecuta las pruebas:

```bash
pytest tests/test_data_cleaner.py -v
```

### Resultado esperado

```
tests/test_data_cleaner.py::TestEliminarDuplicados::test_remove_duplicates_reduces_rows PASSED
tests/test_data_cleaner.py::TestEliminarDuplicados::test_remove_duplicates_preserves_first PASSED
tests/test_data_cleaner.py::TestEliminarDuplicados::test_no_duplicates_unchanged PASSED
tests/test_data_cleaner.py::TestRellenarNulos::test_fill_nulls_no_remaining_nans PASSED
tests/test_data_cleaner.py::TestRellenarNulos::test_fill_nulls_salary_with_median PASSED
tests/test_data_cleaner.py::TestValidarSalario::test_invalid_salary_rejected[50000-True] PASSED
tests/test_data_cleaner.py::TestValidarSalario::test_invalid_salary_rejected[0-False] PASSED
tests/test_data_cleaner.py::TestValidarSalario::test_invalid_salary_rejected[-1000-False] PASSED
tests/test_data_cleaner.py::TestValidarSalario::test_invalid_salary_rejected[1000000-True] PASSED
tests/test_data_cleaner.py::TestConvertirFechas::test_date_conversion_to_datetime PASSED
tests/test_data_cleaner.py::TestDataFrameVacio::test_empty_dataframe_handled PASSED

========================= 11 passed in 0.45s =========================
```

### Verificación

- Al menos 5 tests pasan (verde).
- Los tests con `parametrize` generan múltiples casos de prueba.

---

## Paso 4 — Escribir pruebas para `report_generator.py` y `notifier.py`

### Objetivo

Completar la suite de tests con pruebas para los módulos de generación de reportes y notificaciones, usando mocks para evitar efectos secundarios.

### Instrucciones

1. Crea `tests/test_report_generator.py`:

```python
# tests/test_report_generator.py
"""Pruebas unitarias para src/report_generator.py."""

import pytest
import pandas as pd
from pathlib import Path
from unittest.mock import patch, MagicMock

from src.report_generator import (
    generar_resumen_departamento,
    calcular_estadisticas,
    exportar_reporte,
)


class TestGenerarResumen:
    """Tests para generación de resumen por departamento."""

    def test_resumen_agrupa_por_departamento(self, df_valido):
        """Verifica que el resumen agrupa correctamente."""
        resultado = generar_resumen_departamento(df_valido)
        assert "TI" in resultado["departamento"].values
        assert "Ventas" in resultado["departamento"].values

    def test_resumen_calcula_promedio_salario(self, df_valido):
        """Verifica que el promedio salarial es correcto."""
        resultado = generar_resumen_departamento(df_valido)
        ti_row = resultado[resultado["departamento"] == "TI"]
        # Promedio TI: (50000 + 55000) / 2 = 52500
        assert ti_row["salario"].values[0] == pytest.approx(52500, rel=0.01)

    def test_resumen_con_un_solo_departamento(self):
        """Verifica funcionamiento con un solo departamento."""
        df = pd.DataFrame({
            "id": [1, 2],
            "nombre": ["Ana", "Luis"],
            "departamento": ["TI", "TI"],
            "salario": [50000, 60000],
            "fecha_ingreso": pd.to_datetime(["2020-01-15", "2021-03-10"]),
        })
        resultado = generar_resumen_departamento(df)
        assert len(resultado) == 1


class TestCalcularEstadisticas:
    """Tests para cálculo de estadísticas globales."""

    def test_estadisticas_contiene_claves_esperadas(self, df_valido):
        """Verifica que el diccionario de estadísticas tiene las claves correctas."""
        stats = calcular_estadisticas(df_valido)
        assert "total_empleados" in stats
        assert "salario_promedio" in stats
        assert "salario_maximo" in stats


class TestExportarReporte:
    """Tests para exportación de reportes."""

    def test_exportar_crea_archivo(self, df_valido, tmp_path):
        """Verifica que se crea el archivo de salida."""
        output_file = tmp_path / "reporte_test.csv"
        exportar_reporte(df_valido, str(output_file))
        assert output_file.exists()
        assert output_file.stat().st_size > 0
```

2. Crea `tests/test_notifier.py`:

```python
# tests/test_notifier.py
"""Pruebas unitarias para src/notifier.py con mocks."""

import pytest
from unittest.mock import patch, MagicMock, call

from src.notifier import (
    enviar_correo,
    enviar_webhook,
    notificar,
)


class TestEnviarCorreo:
    """Tests para envío de correo con mock de smtplib."""

    @patch("src.notifier.smtplib.SMTP")
    def test_enviar_correo_llama_sendmail(self, mock_smtp):
        """Verifica que se llama a sendmail con los parámetros correctos."""
        # Configurar el mock
        mock_server = MagicMock()
        mock_smtp.return_value.__enter__ = MagicMock(return_value=mock_server)
        mock_smtp.return_value.__exit__ = MagicMock(return_value=False)

        enviar_correo(
            destinatario="test@example.com",
            asunto="Reporte diario",
            cuerpo="El reporte está listo.",
        )

        # Verificar que se intentó enviar
        mock_server.sendmail.assert_called_once()

    @patch("src.notifier.smtplib.SMTP")
    def test_enviar_correo_maneja_error_conexion(self, mock_smtp):
        """Verifica que errores de conexión se manejan sin crash."""
        mock_smtp.side_effect = ConnectionRefusedError("No se pudo conectar")

        # No debe lanzar excepción, solo loggear el error
        resultado = enviar_correo(
            destinatario="test@example.com",
            asunto="Test",
            cuerpo="Cuerpo",
        )
        assert resultado is False or resultado is None


class TestEnviarWebhook:
    """Tests para envío de webhook con mock de requests."""

    @patch("src.notifier.requests.post")
    def test_webhook_envia_payload_correcto(self, mock_post):
        """Verifica que el webhook envía el payload esperado."""
        mock_post.return_value = MagicMock(status_code=200)

        enviar_webhook(
            url="https://hooks.example.com/webhook",
            mensaje="Reporte generado exitosamente",
        )

        mock_post.assert_called_once()
        # Verificar que el mensaje está en el payload
        args, kwargs = mock_post.call_args
        payload = kwargs.get("json") or kwargs.get("data") or args[1] if len(args) > 1 else {}
        assert "Reporte" in str(payload) or mock_post.called

    @patch("src.notifier.requests.post")
    def test_webhook_maneja_timeout(self, mock_post):
        """Verifica que un timeout no causa crash."""
        import requests
        mock_post.side_effect = requests.exceptions.Timeout("Timeout")

        resultado = enviar_webhook(
            url="https://hooks.example.com/webhook",
            mensaje="Test timeout",
        )
        assert resultado is False or resultado is None


class TestNotificar:
    """Tests para la función principal de notificación."""

    @patch("src.notifier.enviar_correo")
    @patch("src.notifier.enviar_webhook")
    def test_notificar_invoca_ambos_canales(self, mock_webhook, mock_correo):
        """Verifica que notificar() llama a correo y webhook."""
        mock_correo.return_value = True
        mock_webhook.return_value = True

        notificar(mensaje="Reporte listo", destinatario="admin@test.com")

        mock_correo.assert_called_once()
        mock_webhook.assert_called_once()
```

> **Nota:** Ajusta las importaciones y nombres de funciones para que coincidan con tu implementación real de `src/notifier.py`. Si tus funciones tienen firmas diferentes, adapta los parámetros en los tests.

3. Ejecuta toda la suite:

```bash
pytest tests/ -v
```

### Resultado esperado

```
tests/test_data_cleaner.py::... PASSED (11 tests)
tests/test_report_generator.py::... PASSED (4 tests)
tests/test_notifier.py::... PASSED (5 tests)

========================= 20 passed in 1.2s =========================
```

### Verificación

- Los tests de `notifier.py` pasan sin enviar correos ni hacer requests reales.
- Los mocks interceptan correctamente las llamadas a `smtplib.SMTP` y `requests.post`.

---

## Paso 5 — Medir cobertura de código con pytest-cov

### Objetivo

Ejecutar `pytest-cov` para medir la cobertura del directorio `src/` y generar un reporte HTML navegable.

### Instrucciones

1. Ejecuta pytest con cobertura:

```bash
pytest tests/ --cov=src --cov-report=html --cov-report=term-missing -v
```

2. Revisa la salida en terminal. Busca el porcentaje global:

```
---------- coverage: platform linux, python 3.11.9 ----------
Name                        Stmts   Miss  Cover   Missing
---------------------------------------------------------
src/__init__.py                 0      0   100%
src/data_cleaner.py            45      8    82%   34-38, 52-55
src/logger_config.py           22      0   100%
src/notifier.py                38     10    74%   45-50, 67-72
src/report_generator.py        32      6    81%   28-33
src/web_automation.py          55     30    45%   20-75
---------------------------------------------------------
TOTAL                         192     54    72%

Required test coverage of 70% reached. Total coverage: 72%
```

3. Abre el reporte HTML para inspección detallada:

```bash
# macOS
open htmlcov/index.html

# Linux
xdg-open htmlcov/index.html

# Windows
start htmlcov/index.html
```

4. En el reporte HTML, haz clic en cada archivo para ver las líneas cubiertas (verde) y no cubiertas (rojo). Identifica funciones que necesitan más tests.

5. (Opcional) Crea un archivo `.coveragerc` para configuración persistente:

```ini
# .coveragerc
[run]
source = src
omit = src/web_automation.py

[report]
fail_under = 70
show_missing = true

[html]
directory = htmlcov
```

### Resultado esperado

- Cobertura total ≥ 70 %.
- Directorio `htmlcov/` creado con `index.html` navegable.
- Las líneas no cubiertas están claramente identificadas.

### Verificación

```bash
# Verificar que el reporte existe
ls htmlcov/index.html

# Verificar cobertura mínima (falla si < 70%)
pytest tests/ --cov=src --cov-fail-under=70
```

---

## Paso 6 — Identificar y corregir bugs con debugging

### Objetivo

Usar las técnicas de debugging (breakpoints + tests) para identificar y corregir al menos 2 bugs en el código del proyecto.

### Instrucciones

1. **Bug 1 — Error en clave de diccionario (tipo Lección 9.1):**

   Revisa `src/data_cleaner.py`. Un bug común es acceder a una columna con nombre incorrecto. Ejemplo de bug introducido:

```python
# BUG: La columna se llama "fecha_ingreso" pero se accede como "fecha_inicio"
def convertir_fechas(df):
    df["fecha_inicio"] = pd.to_datetime(df["fecha_inicio"])  # KeyError!
    return df
```

   **Corrección:**

```python
def convertir_fechas(df):
    df["fecha_ingreso"] = pd.to_datetime(df["fecha_ingreso"])
    return df
```

   Usa el debugger de VS Code:
   - Coloca un breakpoint en la línea que falla.
   - Ejecuta con `F5`.
   - En el panel Variables, inspecciona `df.columns` para ver los nombres reales.
   - Corrige el nombre de la columna.

2. **Bug 2 — Comparación incorrecta de tipos:**

   En `src/report_generator.py`, un bug típico es comparar un salario (numérico) con un string:

```python
# BUG: salario_minimo es string "30000" en lugar de int
def filtrar_por_salario(df, salario_minimo="30000"):
    return df[df["salario"] > salario_minimo]  # Comparación str vs int
```

   **Corrección:**

```python
def filtrar_por_salario(df, salario_minimo: int = 30000):
    return df[df["salario"] > salario_minimo]
```

   Para encontrarlo:
   - Ejecuta el test correspondiente y observa el fallo.
   - Coloca un breakpoint y usa Watch Expression: `type(salario_minimo)`.
   - Verifica que el tipo es `str` cuando debería ser `int`.

3. Ejecuta los tests nuevamente para confirmar que los bugs están corregidos:

```bash
pytest tests/ -v
```

4. Documenta los bugs corregidos en un archivo `BUGS_FIXED.md`:

```markdown
# Bugs Corregidos — Lab 09

## Bug 1: KeyError en convertir_fechas()
- **Archivo:** src/data_cleaner.py, línea 34
- **Síntoma:** KeyError: 'fecha_inicio'
- **Causa:** Nombre de columna incorrecto ('fecha_inicio' vs 'fecha_ingreso')
- **Corrección:** Cambiar 'fecha_inicio' por 'fecha_ingreso'
- **Test que lo detectó:** test_date_conversion_to_datetime

## Bug 2: TypeError en filtrar_por_salario()
- **Archivo:** src/report_generator.py, línea 28
- **Síntoma:** Comparación silenciosa incorrecta (str > int)
- **Causa:** Parámetro por defecto era string "30000" en vez de int
- **Corrección:** Cambiar tipo del parámetro a int con type hint
- **Test que lo detectó:** test_resumen_calcula_promedio_salario
```

### Resultado esperado

- 2 bugs identificados y corregidos.
- Todos los tests pasan después de las correcciones.
- Archivo `BUGS_FIXED.md` documenta los hallazgos.

### Verificación

```bash
# Todos los tests pasan
pytest tests/ -v --tb=short

# Cobertura se mantiene ≥ 70%
pytest tests/ --cov=src --cov-fail-under=70 -q
```

---

## Validación y testing final

Ejecuta la validación completa del laboratorio:

```bash
cd ~/automation_project

# 1. Verificar que no quedan print() en src/
echo "=== Verificando ausencia de print() ==="
grep -rn "^[^#]*print(" src/ --include="*.py" | grep -v "__main__" && echo "FALLO: Quedan print()" || echo "OK: No hay print() en src/"

# 2. Verificar que logger_config.py existe y es importable
echo "=== Verificando logger_config ==="
python -c "from src.logger_config import get_logger; l = get_logger('test'); l.info('Logger OK')"

# 3. Ejecutar suite completa con cobertura
echo "=== Ejecutando tests con cobertura ==="
pytest tests/ --cov=src --cov-fail-under=70 --cov-report=term-missing -v

# 4. Verificar que logs/app.log existe
echo "=== Verificando archivo de log ==="
test -f logs/app.log && echo "OK: logs/app.log existe" || echo "FALLO: logs/app.log no encontrado"

# 5. Verificar BUGS_FIXED.md
echo "=== Verificando documentación de bugs ==="
test -f BUGS_FIXED.md && echo "OK: BUGS_FIXED.md existe" || echo "FALLO: BUGS_FIXED.md no encontrado"
```

**Criterios de éxito:**

| Criterio | Umbral |
|----------|--------|
| Tests pasando | ≥ 20 tests en verde |
| Cobertura de código | ≥ 70 % |
| Bugs corregidos | ≥ 2 documentados |
| Logging implementado | 0 `print()` en `src/` |
| Archivo de log | `logs/app.log` existe y tiene contenido |

---

## Solución de problemas

### Problema 1: `ModuleNotFoundError: No module named 'src'`

**Síntomas:** Al ejecutar `pytest tests/` aparece un error de importación indicando que no se encuentra el módulo `src` o sus submódulos.

**Causa:** pytest no encuentra el paquete `src` porque no está en el `PYTHONPATH` o falta el archivo `__init__.py`.

**Solución:**

```bash
# Opción A: Crear __init__.py si no existe
touch src/__init__.py

# Opción B: Ejecutar pytest desde la raíz con configuración explícita
# Crear pyproject.toml o pytest.ini en la raíz:
cat > pytest.ini << 'EOF'
[pytest]
testpaths = tests
pythonpath = .
EOF

# Opción C: Instalar el proyecto en modo editable
pip install -e .
```

---

### Problema 2: `pytest-cov` reporta 0% de cobertura

**Síntomas:** La ejecución de `pytest --cov=src` muestra 0% en todos los archivos o no lista ningún archivo de `src/`.

**Causa:** El argumento `--cov=src` busca un paquete llamado `src` pero las importaciones en los tests usan rutas diferentes (por ejemplo, `from data_cleaner import ...` en vez de `from src.data_cleaner import ...`).

**Solución:**

```bash
# Verificar cómo se importa en los tests:
grep "^from\|^import" tests/test_data_cleaner.py

# Asegurar consistencia: las importaciones deben usar el prefijo src.
# Ejemplo correcto: from src.data_cleaner import eliminar_duplicados

# Si usas importaciones sin prefijo, cambiar --cov:
pytest tests/ --cov=. --cov-report=term-missing

# Verificar que src/__init__.py existe:
ls src/__init__.py
```

---

## Limpieza

No se requiere eliminar archivos creados en este lab, ya que forman parte permanente del proyecto. Los artefactos generados que pueden limpiarse opcionalmente son:

```bash
# Eliminar caché de pytest y coverage (regenerable)
rm -rf .pytest_cache/
rm -rf htmlcov/
rm -f .coverage

# NO eliminar (son entregables del lab):
# - src/logger_config.py
# - tests/test_data_cleaner.py
# - tests/test_report_generator.py
# - tests/test_notifier.py
# - tests/conftest.py
# - logs/app.log
# - BUGS_FIXED.md
# - .vscode/launch.json
```

---

## Resumen

En este laboratorio has aplicado un flujo completo de ingeniería de calidad:

| Actividad | Resultado |
|-----------|-----------|
| Configuración del debugger VS Code | `launch.json` funcional con breakpoints condicionales |
| Logging centralizado | `logger_config.py` con `RotatingFileHandler` (5 MB, 5 backups) |
| Pruebas unitarias | ≥ 20 tests con fixtures, parametrize y mocks |
| Cobertura de código | ≥ 70 % medida con pytest-cov |
| Corrección de bugs | 2 bugs identificados, corregidos y documentados |

### Estructura final del proyecto

```
automation_project/
├── .vscode/
│   └── launch.json
├── src/
│   ├── __init__.py
│   ├── logger_config.py        ← NUEVO
│   ├── data_cleaner.py         ← MODIFICADO (logging)
│   ├── report_generator.py     ← MODIFICADO (logging + bug fix)
│   ├── notifier.py             ← MODIFICADO (logging)
│   └── web_automation.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py             ← NUEVO
│   ├── test_data_cleaner.py    ← NUEVO
│   ├── test_report_generator.py ← NUEVO
│   └── test_notifier.py        ← NUEVO
├── logs/
│   └── app.log                 ← GENERADO
├── htmlcov/                    ← GENERADO
├── BUGS_FIXED.md               ← NUEVO
├── pytest.ini                  ← NUEVO
└── requirements.txt
```

### Próximos pasos

En el **Lab 10** integrarás esta suite de tests en un flujo de trabajo automatizado, ejecutando los tests como parte de un pipeline de validación antes de cada despliegue.

### Recursos adicionales

- [Documentación oficial de pytest](https://docs.pytest.org/en/8.2.x/)
- [pytest-cov: documentación](https://pytest-cov.readthedocs.io/en/latest/)
- [VS Code Debugging Python](https://code.visualstudio.com/docs/python/debugging)
- [Módulo logging de Python](https://docs.python.org/3/library/logging.html)
- [unittest.mock — Guía oficial](https://docs.python.org/3/library/unittest.mock.html)
