# Empaquetado y despliegue de un proyecto

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 42 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |
| **Tecnologías** | Python 3.11.9, Git 2.45.1, GitHub, setuptools 70.0.0, build 1.2.1, twine 5.1.0, python-dotenv 1.0.1 |

## Descripción General

En este laboratorio final transformarás el proyecto `auto_reporter` en un paquete Python profesional, instalable y desplegable. Aplicarás documentación con docstrings estilo Google, control de versiones con Git y commits semánticos, empaquetado con `pyproject.toml` siguiendo PEP 517/518, y medidas de seguridad para proteger credenciales. Al finalizar, instalarás el paquete generado en un entorno virtual limpio y verificarás su funcionamiento completo.

## Objetivos de Aprendizaje

- [ ] Documentar todos los módulos públicos de `src/` con docstrings estilo Google y generar un `README.md` completo con instrucciones de instalación y uso
- [ ] Inicializar un repositorio Git local con historial de commits semánticos y publicar el proyecto en GitHub con `.gitignore` y gestión correcta de secretos
- [ ] Empaquetar el proyecto como distribución Python instalable usando `pyproject.toml` con setuptools 70.0.0 y build 1.2.1, generando archivos `.whl` y `.tar.gz`
- [ ] Aplicar medidas de seguridad: escaneo de secretos hardcodeados, manejo seguro de credenciales con `.env` y validación de entradas
- [ ] Verificar la instalación del paquete en un entorno virtual limpio y confirmar la ejecución del CLI entry point

## Prerrequisitos

### Conocimientos Requeridos

- Labs 06–09 completados: proyecto `auto_reporter/` con pipeline funcional y tests pasando con cobertura ≥ 70%
- Familiaridad con docstrings y PEP 257 (Lección 10.1)
- Conceptos básicos de Git (init, add, commit, push, branch, merge)
- Cuenta GitHub gratuita creada y configurada con SSH o token personal

### Software Instalado

| Herramienta | Versión | Verificación |
|-------------|---------|--------------|
| Python | 3.11.9 | `python --version` |
| Git | 2.45.1 | `git --version` |
| pip | ≥ 24.0 | `pip --version` |
| setuptools | 70.0.0 | `pip show setuptools` |
| build | 1.2.1 | `pip show build` |
| twine | 5.1.0 | `pip show twine` |
| python-dotenv | 1.0.1 | `pip show python-dotenv` |

## Entorno del Laboratorio

### Estructura del Proyecto Esperada (pre-lab)

```
~/proyectos/auto_reporter/
├── src/
│   ├── __init__.py
│   ├── config.py
│   ├── data_loader.py
│   ├── reporter.py
│   └── notifier.py
├── tests/
│   ├── __init__.py
│   ├── test_data_loader.py
│   ├── test_reporter.py
│   └── test_notifier.py
├── data/
│   └── input/
├── logs/
├── .env
├── requirements.txt
└── .venv/
```

### Comandos de Preparación

```bash
# Navegar al directorio del proyecto
cd ~/proyectos/auto_reporter

# Activar entorno virtual
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate

# Instalar herramientas de empaquetado
pip install setuptools==70.0.0 build==1.2.1 twine==5.1.0

# Verificar que los tests existentes pasan
pytest tests/ -v
```

---

## Paso 1: Documentar Módulos con Docstrings Estilo Google

### Objetivo

Agregar docstrings profesionales estilo Google a todos los módulos y funciones públicas de `src/`, incluyendo el docstring de módulo, de clase y de cada función.

### Instrucciones

1. Abre el archivo `src/__init__.py` y agrega el docstring de paquete:

```python
"""
auto_reporter - Pipeline de automatización para generación de reportes.

Este paquete proporciona herramientas para cargar datos de empleados,
generar reportes en múltiples formatos y enviar notificaciones automáticas.

Modules:
    config: Gestión de configuración y variables de entorno.
    data_loader: Carga y validación de datos desde archivos CSV.
    reporter: Generación de reportes en HTML y PDF.
    notifier: Envío de notificaciones por correo electrónico.

Example:
    >>> from auto_reporter.data_loader import cargar_empleados
    >>> datos = cargar_empleados("data/input/employees.csv")
    >>> print(len(datos))
    50
"""

__version__ = "1.0.0"
```

2. Documenta el módulo `src/config.py`:

```python
"""
Módulo de configuración del proyecto auto_reporter.

Gestiona la carga de variables de entorno desde archivos .env
y proporciona acceso centralizado a la configuración del sistema.

Typical usage::

    from auto_reporter.config import obtener_config
    config = obtener_config()
    print(config.smtp_host)
"""

import os
from dataclasses import dataclass
from dotenv import load_dotenv


@dataclass
class Config:
    """
    Contenedor de configuración del sistema.

    Attributes:
        smtp_host (str): Servidor SMTP para envío de correos.
        smtp_port (int): Puerto del servidor SMTP.
        smtp_user (str): Usuario de autenticación SMTP.
        smtp_password (str): Contraseña SMTP (cargada desde .env).
        data_path (str): Ruta al directorio de datos de entrada.
        output_path (str): Ruta al directorio de salida de reportes.
    """

    smtp_host: str
    smtp_port: int
    smtp_user: str
    smtp_password: str
    data_path: str
    output_path: str


def obtener_config() -> Config:
    """
    Carga y retorna la configuración del sistema desde variables de entorno.

    Lee el archivo .env del directorio raíz del proyecto y construye
    un objeto Config con todos los parámetros necesarios.

    Returns:
        Config: Objeto con la configuración completa del sistema.

    Raises:
        EnvironmentError: Si alguna variable requerida no está definida.

    Example:
        >>> config = obtener_config()
        >>> isinstance(config.smtp_port, int)
        True
    """
    load_dotenv()

    required_vars = ["SMTP_HOST", "SMTP_PORT", "SMTP_USER", "SMTP_PASSWORD"]
    for var in required_vars:
        if not os.getenv(var):
            raise EnvironmentError(f"Variable de entorno requerida no definida: {var}")

    return Config(
        smtp_host=os.getenv("SMTP_HOST", ""),
        smtp_port=int(os.getenv("SMTP_PORT", "587")),
        smtp_user=os.getenv("SMTP_USER", ""),
        smtp_password=os.getenv("SMTP_PASSWORD", ""),
        data_path=os.getenv("DATA_PATH", "data/input"),
        output_path=os.getenv("OUTPUT_PATH", "data/output"),
    )
```

3. Documenta el módulo `src/data_loader.py`:

```python
"""
Módulo de carga y validación de datos para auto_reporter.

Proporciona funciones para leer archivos CSV de empleados,
validar su estructura y transformar los datos para el pipeline de reportes.
"""

import csv
from pathlib import Path
from typing import Any


def cargar_empleados(ruta_csv: str) -> list[dict[str, Any]]:
    """
    Carga datos de empleados desde un archivo CSV.

    Lee el archivo CSV especificado y retorna una lista de diccionarios
    donde cada diccionario representa un registro de empleado.

    Args:
        ruta_csv (str): Ruta al archivo CSV de empleados.
            El archivo debe contener las columnas: id, nombre,
            departamento, salario, fecha_ingreso.

    Returns:
        list[dict[str, Any]]: Lista de diccionarios con los datos
            de cada empleado.

    Raises:
        FileNotFoundError: Si el archivo CSV no existe en la ruta indicada.
        ValueError: Si el archivo no contiene las columnas requeridas.

    Example:
        >>> empleados = cargar_empleados("data/input/employees.csv")
        >>> empleados[0]["nombre"]
        'Juan Pérez'
    """
    ruta = Path(ruta_csv)
    if not ruta.exists():
        raise FileNotFoundError(f"Archivo no encontrado: {ruta_csv}")

    with open(ruta, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        columnas_requeridas = {"id", "nombre", "departamento", "salario", "fecha_ingreso"}
        if not columnas_requeridas.issubset(set(reader.fieldnames or [])):
            raise ValueError(
                f"Columnas faltantes. Requeridas: {columnas_requeridas}"
            )
        return list(reader)


def validar_registro(registro: dict[str, Any]) -> bool:
    """
    Valida que un registro de empleado contenga datos válidos.

    Args:
        registro (dict[str, Any]): Diccionario con datos del empleado.

    Returns:
        bool: True si el registro es válido, False en caso contrario.
    """
    campos_obligatorios = ["id", "nombre", "departamento", "salario"]
    return all(registro.get(campo) for campo in campos_obligatorios)
```

4. Aplica el mismo patrón de documentación a `src/reporter.py` y `src/notifier.py`, asegurando que cada función pública tenga docstring con secciones `Args`, `Returns`, `Raises` y `Example` donde corresponda.

5. Verifica que los docstrings son accesibles:

```bash
python -c "from src.data_loader import cargar_empleados; help(cargar_empleados)"
```

### Salida Esperada

```
Help on function cargar_empleados in module src.data_loader:

cargar_empleados(ruta_csv: str) -> list[dict[str, Any]]
    Carga datos de empleados desde un archivo CSV.

    Lee el archivo CSV especificado y retorna una lista de diccionarios
    donde cada diccionario representa un registro de empleado.

    Args:
        ruta_csv (str): Ruta al archivo CSV de empleados.
    ...
```

### Verificación

```bash
# Verificar que __doc__ está definido en todos los módulos
python -c "
import src.config, src.data_loader
assert src.config.__doc__ is not None, 'config sin docstring'
assert src.data_loader.__doc__ is not None, 'data_loader sin docstring'
print('✓ Todos los módulos tienen docstring de módulo')
"
```

---

## Paso 2: Crear README.md Completo

### Objetivo

Generar un archivo `README.md` profesional con todas las secciones necesarias para que un nuevo desarrollador pueda instalar, configurar y usar el proyecto.

### Instrucciones

1. Crea el archivo `README.md` en la raíz del proyecto:

```bash
touch README.md
```

2. Escribe el contenido completo del README:

```markdown
# auto_reporter

Pipeline de automatización para generación de reportes de empleados con
notificaciones por correo electrónico.

## Descripción

`auto_reporter` es un paquete Python que automatiza el proceso de:
- Carga y validación de datos de empleados desde archivos CSV
- Generación de reportes en formato HTML
- Envío de notificaciones automáticas por correo electrónico

## Arquitectura

```
auto_reporter/
├── src/                    # Código fuente principal
│   ├── __init__.py         # Inicializador del paquete
│   ├── config.py           # Gestión de configuración (.env)
│   ├── data_loader.py      # Carga y validación de datos CSV
│   ├── reporter.py         # Generación de reportes HTML
│   └── notifier.py         # Envío de correos SMTP
├── tests/                  # Suite de tests unitarios
├── data/
│   ├── input/              # Archivos CSV de entrada
│   └── output/             # Reportes generados
├── logs/                   # Archivos de log
├── pyproject.toml          # Configuración de empaquetado
├── requirements.txt        # Dependencias de desarrollo
└── .env.example            # Plantilla de variables de entorno
```

## Requisitos

- Python >= 3.11
- pip >= 24.0
- Conexión a Internet (para envío SMTP)

## Instalación

### Desde el paquete distribuible (recomendado)

```bash
pip install dist/auto_reporter-1.0.0-py3-none-any.whl
```

### Desde el código fuente (desarrollo)

```bash
git clone https://github.com/<tu-usuario>/auto_reporter.git
cd auto_reporter
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -e .
```

## Configuración (.env)

Crea un archivo `.env` en la raíz del proyecto basándote en `.env.example`:

```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=tu_correo@gmail.com
SMTP_PASSWORD=tu_app_password
DATA_PATH=data/input
OUTPUT_PATH=data/output
```

> ⚠️ **NUNCA** subas el archivo `.env` al repositorio. Está excluido en `.gitignore`.

## Uso

### Como CLI

```bash
auto-reporter --input data/input/employees.csv --output data/output/
```

### Como librería

```python
from auto_reporter.data_loader import cargar_empleados
from auto_reporter.reporter import generar_reporte

empleados = cargar_empleados("data/input/employees.csv")
generar_reporte(empleados, "data/output/reporte.html")
```

## Ejecución de Tests

```bash
# Ejecutar todos los tests
pytest tests/ -v

# Con cobertura
pytest tests/ --cov=src --cov-report=html
```

## Extensibilidad

Para agregar nuevas fuentes de datos:
1. Crear un nuevo módulo en `src/` (ej: `src/api_loader.py`)
2. Implementar una función que retorne `list[dict[str, Any]]`
3. Registrar el loader en `src/__init__.py`
4. Agregar tests en `tests/`
```

3. Crea el archivo `.env.example` como plantilla segura:

```bash
cat > .env.example << 'EOF'
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=user@example.com
SMTP_PASSWORD=CHANGE_ME
DATA_PATH=data/input
OUTPUT_PATH=data/output
EOF
```

### Salida Esperada

El archivo `README.md` debe existir con al menos 80 líneas de contenido estructurado.

### Verificación

```bash
# Verificar que el README tiene las secciones requeridas
grep -c "## " README.md
# Esperado: al menos 8 secciones (encabezados nivel 2)

wc -l README.md
# Esperado: >= 80 líneas
```

---

## Paso 3: Inicializar Repositorio Git con Commits Semánticos

### Objetivo

Crear un repositorio Git local con historial organizado mediante commits semánticos y un `.gitignore` que proteja archivos sensibles.

### Instrucciones

1. Crea el archivo `.gitignore`:

```bash
cat > .gitignore << 'EOF'
# Entorno virtual
.venv/
venv/

# Variables de entorno (secretos)
.env

# Cache de Python
__pycache__/
*.pyc
*.pyo
*.egg-info/

# Distribución
dist/
build/

# Cobertura y reportes
htmlcov/
.coverage

# Logs y salidas generadas
logs/
screenshots/
data/output/

# IDE
.vscode/
.idea/
EOF
```

2. Inicializa el repositorio Git:

```bash
git init
git branch -m main
```

3. Realiza commits semánticos organizados por funcionalidad:

```bash
# Commit 1: Estructura base
git add src/__init__.py src/config.py
git commit -m "feat: agregar módulo de configuración con carga desde .env"

# Commit 2: Data loader
git add src/data_loader.py
git commit -m "feat: implementar carga y validación de datos CSV"

# Commit 3: Reporter
git add src/reporter.py
git commit -m "feat: agregar generación de reportes HTML"

# Commit 4: Notifier
git add src/notifier.py
git commit -m "feat: implementar envío de notificaciones SMTP"

# Commit 5: Tests
git add tests/
git commit -m "test: agregar suite de tests unitarios con pytest"

# Commit 6: Documentación
git add README.md .env.example
git commit -m "docs: crear README.md con instrucciones completas"

# Commit 7: Configuración de proyecto
git add .gitignore requirements.txt
git commit -m "chore: agregar .gitignore y requirements.txt"
```

4. Crea la rama `develop` y practica merge:

```bash
# Crear rama develop
git checkout -b develop

# Simular un cambio en develop
echo "# Changelog" > CHANGELOG.md
echo "" >> CHANGELOG.md
echo "## [1.0.0] - $(date +%Y-%m-%d)" >> CHANGELOG.md
echo "- Release inicial del proyecto auto_reporter" >> CHANGELOG.md

git add CHANGELOG.md
git commit -m "docs: agregar CHANGELOG.md con release inicial"

# Volver a main y hacer merge
git checkout main
git merge develop --no-ff -m "Merge branch 'develop' into main"
```

5. Publica en GitHub:

```bash
# Crear repositorio en GitHub (via web o gh CLI)
# Luego conectar el remoto:
git remote add origin https://github.com/<tu-usuario>/auto_reporter.git
git push -u origin main
git push origin develop
```

### Salida Esperada

```
$ git log --oneline
a1b2c3d (HEAD -> main) Merge branch 'develop' into main
d4e5f6g (develop) docs: agregar CHANGELOG.md con release inicial
h7i8j9k chore: agregar .gitignore y requirements.txt
l0m1n2o docs: crear README.md con instrucciones completas
p3q4r5s test: agregar suite de tests unitarios con pytest
t6u7v8w feat: implementar envío de notificaciones SMTP
x9y0z1a feat: agregar generación de reportes HTML
b2c3d4e feat: implementar carga y validación de datos CSV
f5g6h7i feat: agregar módulo de configuración con carga desde .env
```

### Verificación

```bash
# Verificar que .env NO está rastreado
git status --short | grep -v ".env" && echo "✓ .env no está en el repositorio"

# Verificar historial de commits
git log --oneline | wc -l
# Esperado: >= 8 commits

# Verificar que existen ambas ramas
git branch -a
# Esperado: * main y develop
```

---

## Paso 4: Crear pyproject.toml y Empaquetar el Proyecto

### Objetivo

Configurar el empaquetado del proyecto con `pyproject.toml` siguiendo PEP 517/518 y generar los archivos de distribución `.whl` y `.tar.gz`.

### Instrucciones

1. Crea el archivo `pyproject.toml` en la raíz del proyecto:

```toml
[build-system]
requires = ["setuptools==70.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "auto-reporter"
version = "1.0.0"
description = "Pipeline de automatización para generación de reportes de empleados"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.11"
authors = [
    {name = "Tu Nombre", email = "tu@email.com"},
]
keywords = ["automatización", "reportes", "csv", "smtp"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.11",
    "Topic :: Office/Business",
]
dependencies = [
    "python-dotenv==1.0.1",
    "pandas==2.2.2",
    "openpyxl==3.1.2",
    "Jinja2==3.1.4",
]

[project.optional-dependencies]
dev = [
    "pytest==8.1.1",
    "pytest-cov>=5.0.0",
    "black==24.3.0",
    "mypy==1.9.0",
]

[project.scripts]
auto-reporter = "src.reporter:main"

[tool.setuptools.packages.find]
where = ["."]
include = ["src*"]
```

2. Asegúrate de que `src/reporter.py` tenga una función `main()` como entry point:

```python
# Agregar al final de src/reporter.py

def main() -> None:
    """
    Entry point principal del CLI auto-reporter.

    Ejecuta el pipeline completo: carga datos, genera reporte
    y envía notificación.

    Example:
        Desde la línea de comandos::

            $ auto-reporter
    """
    import argparse

    parser = argparse.ArgumentParser(
        description="Genera reportes de empleados automáticamente"
    )
    parser.add_argument(
        "--input",
        default="data/input/employees.csv",
        help="Ruta al archivo CSV de entrada",
    )
    parser.add_argument(
        "--output",
        default="data/output",
        help="Directorio de salida para reportes",
    )
    args = parser.parse_args()

    print(f"[auto-reporter] Cargando datos desde: {args.input}")
    print(f"[auto-reporter] Generando reporte en: {args.output}")
    print("[auto-reporter] Pipeline completado exitosamente.")


if __name__ == "__main__":
    main()
```

3. Construye el paquete:

```bash
# Limpiar distribuciones anteriores si existen
rm -rf dist/ build/ *.egg-info

# Construir el paquete
python -m build
```

4. Verifica los archivos generados:

```bash
ls dist/
```

5. Valida el paquete con twine:

```bash
twine check dist/*
```

### Salida Esperada

```
$ python -m build
* Creating venv isolated environment...
* Installing packages in isolated environment... (setuptools==70.0.0, wheel)
* Getting build dependencies for sdist...
* Building sdist...
* Building wheel...
Successfully built auto_reporter-1.0.0.tar.gz and auto_reporter-1.0.0-py3-none-any.whl

$ ls dist/
auto_reporter-1.0.0-py3-none-any.whl
auto_reporter-1.0.0.tar.gz

$ twine check dist/*
Checking dist/auto_reporter-1.0.0-py3-none-any.whl: PASSED
Checking dist/auto_reporter-1.0.0.tar.gz: PASSED
```

### Verificación

```bash
# Verificar que ambos archivos existen
test -f dist/auto_reporter-1.0.0-py3-none-any.whl && echo "✓ Wheel generado"
test -f dist/auto_reporter-1.0.0.tar.gz && echo "✓ Sdist generado"

# Verificar contenido del wheel
python -m zipfile -l dist/auto_reporter-1.0.0-py3-none-any.whl | head -20
```

---

## Paso 5: Aplicar Medidas de Seguridad

### Objetivo

Escanear el código en busca de credenciales hardcodeadas, documentar las prácticas de seguridad implementadas y crear un archivo `SECURITY.md`.

### Instrucciones

1. Escanea el código fuente en busca de secretos hardcodeados:

```bash
# Buscar patrones comunes de credenciales
echo "=== Escaneando secretos hardcodeados ==="
grep -rn "password\s*=" src/ --include="*.py" | grep -v "os.getenv" | grep -v "\.env"
grep -rn "token\s*=" src/ --include="*.py" | grep -v "os.getenv"
grep -rn "secret\s*=" src/ --include="*.py" | grep -v "os.getenv"
grep -rn "api_key\s*=" src/ --include="*.py" | grep -v "os.getenv"
echo "=== Escaneo completado ==="
```

2. Si encuentras credenciales hardcodeadas, reemplázalas con lectura desde `.env`:

```python
# INCORRECTO (secreto hardcodeado):
# smtp_password = "mi_clave_secreta_123"

# CORRECTO (lectura desde variable de entorno):
import os
from dotenv import load_dotenv

load_dotenv()
smtp_password = os.getenv("SMTP_PASSWORD")
if not smtp_password:
    raise EnvironmentError("SMTP_PASSWORD no configurada en .env")
```

3. Crea el archivo `SECURITY.md`:

```bash
cat > SECURITY.md << 'EOF'
# Política de Seguridad - auto_reporter

## Prácticas Implementadas

### 1. Gestión de Credenciales
- **Todas** las credenciales se cargan desde variables de entorno (archivo `.env`)
- El archivo `.env` está excluido del repositorio mediante `.gitignore`
- Se proporciona `.env.example` como plantilla sin valores reales

### 2. Protección del Repositorio
- `.gitignore` configurado para excluir:
  - `.env` (credenciales)
  - `__pycache__/` (bytecode compilado)
  - `dist/` y `*.egg-info/` (artefactos de build)
  - `logs/` (posible información sensible en logs)
  - `htmlcov/` (reportes de cobertura)

### 3. Validación de Entradas
- Los archivos CSV se validan antes de procesarse (columnas requeridas)
- Las rutas de archivo se sanitizan usando `pathlib.Path`
- Los parámetros de configuración se validan al inicio de la ejecución

### 4. Dependencias
- Versiones de dependencias bloqueadas en `pyproject.toml`
- Actualizaciones de seguridad revisadas periódicamente

## Reporte de Vulnerabilidades

Si encuentras una vulnerabilidad de seguridad, por favor:
1. **NO** abras un issue público
2. Envía un correo a: security@tu-dominio.com
3. Incluye: descripción, pasos para reproducir y posible impacto

## Checklist de Seguridad Pre-Deploy

- [ ] No hay credenciales hardcodeadas en el código
- [ ] `.env` no está en el historial de Git
- [ ] Todas las dependencias están en versiones estables
- [ ] Las entradas del usuario están validadas
- [ ] Los logs no contienen información sensible
EOF
```

4. Verifica que `.env` no está en el historial de Git:

```bash
git log --all --full-history -- .env
# No debe mostrar ningún resultado
```

5. Haz commit de los archivos de seguridad:

```bash
git add SECURITY.md
git commit -m "docs: agregar SECURITY.md con políticas de seguridad"

git add pyproject.toml
git commit -m "chore: agregar pyproject.toml para empaquetado PEP 517"
```

### Salida Esperada

```
$ grep -rn "password\s*=" src/ --include="*.py" | grep -v "os.getenv" | grep -v "\.env"
(sin resultados — no hay secretos hardcodeados)

$ git log --all --full-history -- .env
(sin resultados — .env nunca fue committed)
```

### Verificación

```bash
# Confirmar que SECURITY.md existe y tiene contenido
test -f SECURITY.md && echo "✓ SECURITY.md creado"
wc -l SECURITY.md
# Esperado: >= 30 líneas

# Confirmar que .env.example existe pero .env no está rastreado
test -f .env.example && echo "✓ .env.example existe"
git ls-files .env | wc -l
# Esperado: 0 (no rastreado)
```

---

## Paso 6: Instalar y Verificar en Entorno Limpio

### Objetivo

Crear un entorno virtual nuevo, instalar el paquete generado y verificar que el CLI y las importaciones funcionan correctamente.

### Instrucciones

1. Crea un entorno virtual limpio para pruebas de instalación:

```bash
# Salir del entorno virtual actual
deactivate

# Crear entorno de prueba temporal
python -m venv /tmp/test_auto_reporter
source /tmp/test_auto_reporter/bin/activate  # Windows: /tmp/test_auto_reporter/Scripts/activate
```

2. Instala el paquete desde el wheel generado:

```bash
pip install ~/proyectos/auto_reporter/dist/auto_reporter-1.0.0-py3-none-any.whl
```

3. Verifica que el paquete se instaló correctamente:

```bash
pip show auto-reporter
```

4. Prueba el CLI entry point:

```bash
auto-reporter --help
```

5. Prueba la importación como librería:

```bash
python -c "
from src.data_loader import cargar_empleados
from src.config import Config
from src import __version__
print(f'auto_reporter versión: {__version__}')
print(f'Config class: {Config.__doc__[:50]}...')
print('✓ Todas las importaciones exitosas')
"
```

6. Limpia el entorno de prueba:

```bash
deactivate
rm -rf /tmp/test_auto_reporter
```

7. Reactiva el entorno original y haz el commit final:

```bash
cd ~/proyectos/auto_reporter
source .venv/bin/activate

git add dist/
git commit -m "chore: agregar distribuciones wheel y sdist v1.0.0"
git push origin main
```

### Salida Esperada

```
$ pip show auto-reporter
Name: auto-reporter
Version: 1.0.0
Summary: Pipeline de automatización para generación de reportes de empleados
Home-page:
Author: Tu Nombre
Author-email: tu@email.com
License: MIT
Requires: Jinja2, openpyxl, pandas, python-dotenv
Required-by:

$ auto-reporter --help
usage: auto-reporter [-h] [--input INPUT] [--output OUTPUT]

Genera reportes de empleados automáticamente

options:
  -h, --help       show this help message and exit
  --input INPUT    Ruta al archivo CSV de entrada
  --output OUTPUT  Directorio de salida para reportes
```

### Verificación

```bash
# Verificación final completa (ejecutar desde entorno original)
echo "=== VERIFICACIÓN FINAL DEL LAB ==="
echo ""

# 1. Docstrings
python -c "from src import __version__; assert __version__ == '1.0.0'" && echo "✓ 1. Versión correcta en __init__.py"

# 2. README
test -f README.md && echo "✓ 2. README.md existe"

# 3. Git
test -d .git && echo "✓ 3. Repositorio Git inicializado"
git log --oneline | wc -l | xargs -I{} echo "  → {} commits en historial"

# 4. Empaquetado
test -f dist/auto_reporter-1.0.0-py3-none-any.whl && echo "✓ 4. Wheel generado"
test -f dist/auto_reporter-1.0.0.tar.gz && echo "✓ 5. Sdist generado"

# 5. Seguridad
test -f SECURITY.md && echo "✓ 6. SECURITY.md creado"
test -f .gitignore && echo "✓ 7. .gitignore configurado"

# 6. pyproject.toml
test -f pyproject.toml && echo "✓ 8. pyproject.toml configurado"

echo ""
echo "=== LAB COMPLETADO ==="
```

---

## Validación y Testing

Ejecuta la siguiente secuencia completa para validar que todos los objetivos del laboratorio se cumplieron:

```bash
#!/bin/bash
# Script de validación final: validar_lab10.sh

PASS=0
FAIL=0

check() {
    if eval "$1"; then
        echo "✓ PASS: $2"
        ((PASS++))
    else
        echo "✗ FAIL: $2"
        ((FAIL++))
    fi
}

echo "╔══════════════════════════════════════════════╗"
echo "║  VALIDACIÓN LAB 10 - Empaquetado y Deploy   ║"
echo "╚══════════════════════════════════════════════╝"
echo ""

# Documentación
check "python -c 'import src; assert src.__doc__ is not None'" "Docstring en src/__init__.py"
check "python -c 'from src.config import obtener_config; assert obtener_config.__doc__ is not None'" "Docstring en obtener_config()"
check "python -c 'from src.data_loader import cargar_empleados; assert cargar_empleados.__doc__ is not None'" "Docstring en cargar_empleados()"
check "test -f README.md" "README.md existe"
check "[ $(grep -c '## ' README.md) -ge 7 ]" "README tiene >= 7 secciones"

# Git
check "test -d .git" "Repositorio Git inicializado"
check "[ $(git log --oneline | wc -l) -ge 8 ]" "Al menos 8 commits"
check "git branch | grep -q develop" "Rama develop existe"
check "git log --oneline | grep -q 'feat:'" "Commits con prefijo feat:"
check "git log --oneline | grep -q 'docs:'" "Commits con prefijo docs:"

# Empaquetado
check "test -f pyproject.toml" "pyproject.toml existe"
check "test -f dist/auto_reporter-1.0.0-py3-none-any.whl" "Wheel .whl generado"
check "test -f dist/auto_reporter-1.0.0.tar.gz" "Sdist .tar.gz generado"
check "twine check dist/* 2>&1 | grep -q PASSED" "twine check pasa"

# Seguridad
check "test -f .gitignore" ".gitignore existe"
check "grep -q '.env' .gitignore" ".env en .gitignore"
check "test -f SECURITY.md" "SECURITY.md existe"
check "test -f .env.example" ".env.example existe"
check "[ $(git ls-files .env | wc -l) -eq 0 ]" ".env no rastreado en Git"

echo ""
echo "════════════════════════════════════════"
echo "  Resultados: $PASS pasaron, $FAIL fallaron"
echo "════════════════════════════════════════"
```

Ejecuta el script:

```bash
chmod +x validar_lab10.sh
bash validar_lab10.sh
```

**Criterio de éxito**: Todos los checks deben pasar (0 FAIL).

---

## Solución de Problemas

### Problema 1: `python -m build` falla con "No module named 'setuptools'"

**Síntomas:**

```
ERROR: Backend subprocess exited when trying to invoke build_wheel
ModuleNotFoundError: No module named 'setuptools'
```

**Causa:** El paquete `build` crea un entorno aislado para construir el paquete, pero no puede resolver la versión exacta de setuptools especificada en `[build-system].requires`.

**Solución:**

```bash
# Verificar que setuptools está instalado en el entorno actual
pip install setuptools==70.0.0 wheel

# Si persiste, construir sin aislamiento
python -m build --no-isolation

# Alternativa: verificar que pyproject.toml tiene la sintaxis correcta
python -c "import tomllib; tomllib.load(open('pyproject.toml', 'rb'))"
```

---

### Problema 2: `twine check` reporta "warning: description_content_type missing"

**Síntomas:**

```
Checking dist/auto_reporter-1.0.0.tar.gz: PASSED, with warnings
warning: `long_description_content_type` missing.
```

**Causa:** El campo `readme` en `pyproject.toml` no está correctamente configurado o el archivo `README.md` no se incluye en la distribución.

**Solución:**

```toml
# Verificar que pyproject.toml tiene la línea:
[project]
readme = "README.md"
```

```bash
# Asegurar que README.md está incluido en el sdist
tar -tzf dist/auto_reporter-1.0.0.tar.gz | grep README
# Debe mostrar: auto_reporter-1.0.0/README.md

# Si no aparece, agregar MANIFEST.in:
echo "include README.md" > MANIFEST.in

# Reconstruir
rm -rf dist/ build/
python -m build
twine check dist/*
```

---

## Limpieza

```bash
# Eliminar entorno de prueba temporal (si no se hizo en el Paso 6)
rm -rf /tmp/test_auto_reporter

# Opcional: limpiar artefactos de build intermedios (conservar dist/)
rm -rf build/
rm -rf *.egg-info

# Los archivos dist/ se conservan como entregable del laboratorio
echo "Artefactos finales conservados en dist/"
ls -la dist/
```

---

## Resumen

En este laboratorio final consolidaste todas las habilidades del curso transformando un proyecto de automatización en un paquete Python profesional:

| Actividad | Resultado |
|-----------|-----------|
| Documentación | Docstrings Google-style en todos los módulos + README.md completo |
| Control de versiones | Repositorio Git con ≥ 8 commits semánticos, ramas main/develop |
| Empaquetado | `pyproject.toml` PEP 517 → `.whl` y `.tar.gz` validados con twine |
| Seguridad | Escaneo de secretos, `.gitignore`, `SECURITY.md`, credenciales en `.env` |
| Despliegue | Instalación verificada en entorno limpio con CLI funcional |

### Conceptos Clave Aplicados

- **PEP 257**: Convenciones de docstrings
- **PEP 517/518**: Sistema de build declarativo con `pyproject.toml`
- **Commits semánticos**: `feat:`, `fix:`, `docs:`, `test:`, `chore:`
- **Principio de mínimo privilegio**: credenciales solo en `.env`, nunca en código
- **Distribución reproducible**: versiones exactas de dependencias bloqueadas

### Recursos Adicionales

- [PEP 517 – Build system interface](https://peps.python.org/pep-0517/)
- [setuptools – Guía del usuario](https://setuptools.pypa.io/en/latest/userguide/)
- [Conventional Commits](https://www.conventionalcommits.org/es/)
- [Python Packaging User Guide](https://packaging.python.org/)
- [GitHub – Creación de repositorios](https://docs.github.com/es/repositories/creating-and-managing-repositories)
