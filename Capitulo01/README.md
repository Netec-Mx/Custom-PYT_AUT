# Configuración del entorno y primer script de automatización

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 24 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |

## Descripción general

En este laboratorio construirás desde cero la infraestructura completa del proyecto `automation_project` que utilizarás durante todo el curso. Verificarás la instalación de Python 3.12.1, crearás un entorno virtual aislado, instalarás las herramientas de desarrollo iniciales y establecerás la estructura de directorios estándar. Finalmente, escribirás y ejecutarás un script que inspecciona el entorno configurado y reporta información del sistema utilizando módulos estándar de Python.

## Objetivos de aprendizaje

- [ ] Verificar que Python 3.12.1 y pip 24.0 están correctamente instalados y accesibles desde la terminal
- [ ] Crear y activar un entorno virtual con `venv` que aísle las dependencias del proyecto
- [ ] Instalar dependencias iniciales (black, pytest, mypy) y generar un archivo `requirements.txt` con versiones exactas
- [ ] Establecer la estructura de directorios estándar del proyecto (`src/`, `tests/`, `data/`, `logs/`, `backups/`)
- [ ] Escribir y ejecutar un script `setup_check.py` que reporte versiones, rutas y estado del entorno usando módulos estándar

## Prerrequisitos

### Conocimientos previos

- Sintaxis básica de Python: variables, funciones, imports, f-strings, bucles y condicionales
- Manejo básico de terminal/línea de comandos: navegación de directorios (`cd`), creación de carpetas (`mkdir`), listado de archivos (`ls`/`dir`)
- Concepto general de qué es un entorno virtual (no se requiere experiencia previa creándolos)

### Acceso requerido

- Python 3.12.1 instalado en el sistema (descargable desde https://www.python.org/downloads/release/python-3121/)
- Git 2.44.0 instalado (descargable desde https://git-scm.com/downloads)
- Visual Studio Code 1.88.1 instalado con la extensión de Python
- Acceso a terminal con permisos de escritura en el directorio home del usuario

## Entorno del laboratorio

### Hardware mínimo

| Recurso | Requisito |
|---------|-----------|
| Procesador | x86_64, mínimo 2 núcleos |
| RAM | 4 GB mínimo |
| Almacenamiento | 10 GB de espacio libre |
| Conectividad | Internet requerida para `pip install` |

### Software requerido

| Herramienta | Versión exacta |
|-------------|----------------|
| Python | 3.12.1 |
| pip | 24.0 |
| Git | 2.44.0 |
| VS Code | 1.88.1 |
| black | 24.3.0 |
| pytest | 8.1.1 |
| mypy | 1.9.0 |

### Convenciones de rutas

| Sistema operativo | Ruta del proyecto |
|-------------------|-------------------|
| macOS/Linux | `~/automation_project/` |
| Windows | `C:\Users\<usuario>\automation_project\` |

---

## Paso 1: Verificar la instalación de Python y pip

### Objetivo

Confirmar que Python 3.12.1 y pip 24.0 están disponibles en el PATH del sistema y responden correctamente desde la terminal.

### Instrucciones

1. Abre una terminal (Terminal en macOS/Linux, PowerShell en Windows).

2. Verifica la versión de Python:

```bash
python --version
```

> **Nota para macOS/Linux:** Si `python` no se reconoce, intenta con `python3 --version`. En ese caso, usa `python3` en todos los comandos posteriores de este laboratorio.

3. Verifica la versión de pip:

```bash
pip --version
```

> **Nota:** En algunos sistemas puede ser necesario usar `pip3` en lugar de `pip`.

4. Verifica que Git está instalado:

```bash
git --version
```

### Salida esperada

```
Python 3.12.1
pip 24.0 from /path/to/pip (python 3.12)
git version 2.44.0
```

### Verificación

- La versión de Python debe ser exactamente `3.12.1` (no 3.11.x ni 3.13.x)
- pip debe reportar versión `24.0` y estar asociado a Python 3.12
- Git debe reportar versión `2.44.0`

Si pip no está en versión 24.0, actualízalo:

```bash
python -m pip install --upgrade pip==24.0
```

---

## Paso 2: Crear la estructura de directorios del proyecto

### Objetivo

Establecer la estructura de directorios estándar de `automation_project` que será utilizada en todos los laboratorios del curso.

### Instrucciones

1. Navega al directorio home del usuario:

```bash
# macOS/Linux
cd ~

# Windows PowerShell
cd $HOME
```

2. Crea el directorio raíz del proyecto y toda la estructura de subdirectorios:

**macOS/Linux:**

```bash
mkdir -p ~/automation_project/src
mkdir -p ~/automation_project/tests
mkdir -p ~/automation_project/data/input
mkdir -p ~/automation_project/data/output
mkdir -p ~/automation_project/logs
mkdir -p ~/automation_project/backups
```

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\automation_project\src"
New-Item -ItemType Directory -Force -Path "$HOME\automation_project\tests"
New-Item -ItemType Directory -Force -Path "$HOME\automation_project\data\input"
New-Item -ItemType Directory -Force -Path "$HOME\automation_project\data\output"
New-Item -ItemType Directory -Force -Path "$HOME\automation_project\logs"
New-Item -ItemType Directory -Force -Path "$HOME\automation_project\backups"
```

3. Navega al directorio del proyecto:

```bash
cd ~/automation_project
```

4. Verifica la estructura creada:

**macOS/Linux:**

```bash
find . -type d | sort
```

**Windows PowerShell:**

```powershell
Get-ChildItem -Recurse -Directory | Select-Object FullName
```

### Salida esperada

```
.
./backups
./data
./data/input
./data/output
./logs
./src
./tests
```

### Verificación

Confirma que existen exactamente 7 directorios (incluyendo la raíz): `src/`, `tests/`, `data/input/`, `data/output/`, `logs/`, `backups/`. Esta estructura será respetada en todos los laboratorios posteriores.

---

## Paso 3: Crear y activar el entorno virtual

### Objetivo

Crear un entorno virtual aislado con `venv` dentro del proyecto para que las dependencias no interfieran con el Python del sistema ni con otros proyectos.

### Instrucciones

1. Asegúrate de estar en el directorio raíz del proyecto:

```bash
cd ~/automation_project
```

2. Crea el entorno virtual en la carpeta `.venv`:

```bash
python -m venv .venv
```

3. Activa el entorno virtual:

**macOS/Linux:**

```bash
source .venv/bin/activate
```

**Windows PowerShell:**

```powershell
.venv\Scripts\activate
```

> **Nota Windows:** Si recibes un error de política de ejecución, ejecuta primero: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`

4. Verifica que el entorno virtual está activo comprobando la versión de Python dentro del entorno:

```bash
python --version
```

5. Verifica que `pip` apunta al entorno virtual:

```bash
pip --version
```

### Salida esperada

Después de activar el entorno, tu prompt de terminal mostrará el prefijo `(.venv)`:

```
(.venv) usuario@maquina:~/automation_project$
```

La salida de `python --version` debe ser:

```
Python 3.12.1
```

La salida de `pip --version` debe mostrar una ruta que contenga `.venv`:

```
pip 24.0 from /home/usuario/automation_project/.venv/lib/python3.12/site-packages/pip (python 3.12)
```

### Verificación

- El prompt muestra `(.venv)` al inicio
- `which python` (macOS/Linux) o `Get-Command python` (Windows) apunta a `.venv/bin/python` o `.venv\Scripts\python.exe`
- La versión de Python es exactamente 3.12.1

---

## Paso 4: Instalar dependencias iniciales y generar requirements.txt

### Objetivo

Instalar las herramientas de desarrollo del curso (black, pytest, mypy) con versiones exactas y generar un archivo `requirements.txt` reproducible.

### Instrucciones

1. Con el entorno virtual activo, instala las dependencias con versiones exactas:

```bash
pip install black==24.3.0 pytest==8.1.1 mypy==1.9.0
```

2. Verifica que las tres herramientas se instalaron correctamente:

```bash
black --version
pytest --version
mypy --version
```

3. Crea el archivo `requirements.txt` con las versiones exactas. Escribe manualmente el archivo para tener control preciso sobre su contenido:

```bash
cat > requirements.txt << 'EOF'
black==24.3.0
pytest==8.1.1
mypy==1.9.0
requests==2.31.0
responses==0.25.0
EOF
```

**Windows PowerShell:**

```powershell
@"
black==24.3.0
pytest==8.1.1
mypy==1.9.0
requests==2.31.0
responses==0.25.0
"@ | Out-File -FilePath requirements.txt -Encoding utf8
```

> **Nota:** Incluimos `requests` y `responses` en el archivo porque serán necesarios en laboratorios posteriores, aunque no los instalamos todavía en este lab.

4. Verifica el contenido del archivo:

```bash
cat requirements.txt
```

### Salida esperada

Salida de `black --version`:

```
black, 24.3.0 (compiled: yes)
Python (CPython) 3.12.1
```

Salida de `pytest --version`:

```
pytest 8.1.1
```

Salida de `mypy --version`:

```
mypy 1.9.0 (compiled: yes)
```

Contenido de `requirements.txt`:

```
black==24.3.0
pytest==8.1.1
mypy==1.9.0
requests==2.31.0
responses==0.25.0
```

### Verificación

- Los tres paquetes están instalados con las versiones exactas especificadas
- El archivo `requirements.txt` existe en `~/automation_project/requirements.txt`
- Cada línea del archivo usa el formato `paquete==versión` (doble signo igual para bloquear versiones)

---

## Paso 5: Inicializar el repositorio Git

### Objetivo

Inicializar un repositorio Git en el proyecto y crear un archivo `.gitignore` apropiado para proyectos Python.

### Instrucciones

1. Desde el directorio raíz del proyecto, inicializa el repositorio:

```bash
cd ~/automation_project
git init
```

2. Crea un archivo `.gitignore` para excluir archivos que no deben versionarse:

```bash
cat > .gitignore << 'EOF'
# Entorno virtual
.venv/

# Bytecode de Python
__pycache__/
*.pyc
*.pyo

# Directorios de salida y logs
data/output/
logs/

# Archivos del IDE
.vscode/
.idea/

# Archivos del sistema operativo
.DS_Store
Thumbs.db
EOF
```

**Windows PowerShell:**

```powershell
@"
# Entorno virtual
.venv/

# Bytecode de Python
__pycache__/
*.pyc
*.pyo

# Directorios de salida y logs
data/output/
logs/

# Archivos del IDE
.vscode/
.idea/

# Archivos del sistema operativo
.DS_Store
Thumbs.db
"@ | Out-File -FilePath .gitignore -Encoding utf8
```

3. Verifica el estado del repositorio:

```bash
git status
```

### Salida esperada

```
Initialized empty Git repository in /home/usuario/automation_project/.git/
```

Salida de `git status`:

```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	.gitignore
	requirements.txt

nothing added to commit but untracked files present (use "git add" to track)
```

### Verificación

- El directorio `.git/` existe dentro de `~/automation_project/`
- El archivo `.gitignore` excluye `.venv/`, `__pycache__/` y `data/output/`
- `git status` no muestra el directorio `.venv` como archivo sin seguimiento (confirmando que `.gitignore` funciona)

---

## Paso 6: Escribir el script de verificación del entorno

### Objetivo

Crear el script `setup_check.py` en el directorio `src/` que inspeccione el entorno configurado y reporte información del sistema utilizando los módulos estándar `sys`, `os` y `platform`.

### Instrucciones

1. Abre VS Code desde el directorio del proyecto:

```bash
code .
```

2. Crea el archivo `src/setup_check.py` con el siguiente contenido:

```python
"""
setup_check.py — Script de verificación del entorno de desarrollo.

Inspecciona el entorno configurado y reporta:
- Versión de Python e intérprete
- Sistema operativo y arquitectura
- Ruta del entorno virtual
- Estructura de directorios del proyecto
- Paquetes instalados en el entorno

Módulos utilizados: sys, os, platform
"""

import sys
import os
import platform


def obtener_info_python() -> dict:
    """Recopila información sobre la instalación de Python."""
    return {
        "version": platform.python_version(),
        "implementacion": platform.python_implementation(),
        "ejecutable": sys.executable,
        "ruta_prefijo": sys.prefix,
    }


def obtener_info_sistema() -> dict:
    """Recopila información sobre el sistema operativo."""
    return {
        "sistema": platform.system(),
        "version_so": platform.version(),
        "arquitectura": platform.machine(),
        "nombre_nodo": platform.node(),
    }


def verificar_entorno_virtual() -> dict:
    """Verifica si el script se ejecuta dentro de un entorno virtual."""
    en_venv = sys.prefix != sys.base_prefix
    return {
        "activo": en_venv,
        "ruta_venv": sys.prefix if en_venv else "No detectado",
        "ruta_base": sys.base_prefix,
    }


def verificar_estructura_directorios() -> dict:
    """Verifica la existencia de los directorios del proyecto."""
    # Obtener la raíz del proyecto (un nivel arriba de src/)
    directorio_script = os.path.dirname(os.path.abspath(__file__))
    raiz_proyecto = os.path.dirname(directorio_script)

    directorios_requeridos = [
        "src",
        "tests",
        "data/input",
        "data/output",
        "logs",
        "backups",
    ]

    resultados = {}
    for directorio in directorios_requeridos:
        ruta_completa = os.path.join(raiz_proyecto, directorio)
        resultados[directorio] = os.path.isdir(ruta_completa)

    return resultados


def imprimir_reporte(info_python: dict, info_sistema: dict,
                     info_venv: dict, estructura: dict) -> None:
    """Imprime un reporte formateado del estado del entorno."""
    separador = "=" * 60

    print(separador)
    print("  REPORTE DE VERIFICACIÓN DEL ENTORNO")
    print(separador)

    # Sección: Python
    print("\n📐 INFORMACIÓN DE PYTHON")
    print("-" * 40)
    for clave, valor in info_python.items():
        print(f"  {clave:<20} → {valor}")

    # Sección: Sistema operativo
    print("\n💻 INFORMACIÓN DEL SISTEMA")
    print("-" * 40)
    for clave, valor in info_sistema.items():
        print(f"  {clave:<20} → {valor}")

    # Sección: Entorno virtual
    print("\n🔒 ENTORNO VIRTUAL")
    print("-" * 40)
    for clave, valor in info_venv.items():
        print(f"  {clave:<20} → {valor}")

    # Sección: Estructura de directorios
    print("\n📁 ESTRUCTURA DE DIRECTORIOS")
    print("-" * 40)
    todos_existen = True
    for directorio, existe in estructura.items():
        icono = "✓" if existe else "✗"
        estado = "OK" if existe else "FALTA"
        print(f"  [{icono}] {directorio:<20} → {estado}")
        if not existe:
            todos_existen = False

    # Resumen final
    print(f"\n{separador}")
    if todos_existen and info_venv["activo"]:
        print("  ✓ RESULTADO: Entorno configurado correctamente")
    else:
        print("  ✗ RESULTADO: Se detectaron problemas en la configuración")
        if not info_venv["activo"]:
            print("    → El entorno virtual NO está activo")
        if not todos_existen:
            print("    → Faltan directorios en la estructura")
    print(separador)


def main() -> None:
    """Función principal que orquesta la verificación del entorno."""
    info_python = obtener_info_python()
    info_sistema = obtener_info_sistema()
    info_venv = verificar_entorno_virtual()
    estructura = verificar_estructura_directorios()

    imprimir_reporte(info_python, info_sistema, info_venv, estructura)


if __name__ == "__main__":
    main()
```

3. Guarda el archivo (`Ctrl+S` o `Cmd+S`).

### Salida esperada

El archivo `src/setup_check.py` debe existir con aproximadamente 110 líneas de código. No hay salida en terminal en este paso (la ejecución es en el paso siguiente).

### Verificación

- El archivo existe en la ruta `~/automation_project/src/setup_check.py`
- El archivo contiene las funciones: `obtener_info_python`, `obtener_info_sistema`, `verificar_entorno_virtual`, `verificar_estructura_directorios`, `imprimir_reporte` y `main`
- El bloque `if __name__ == "__main__":` está presente al final del archivo

---

## Paso 7: Ejecutar y validar el script de verificación

### Objetivo

Ejecutar el script `setup_check.py` desde la terminal con el entorno virtual activo y confirmar que todas las verificaciones pasan correctamente.

### Instrucciones

1. Asegúrate de que el entorno virtual está activo (el prompt debe mostrar `(.venv)`):

```bash
# Si no está activo, actívalo:
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate
```

2. Ejecuta el script desde el directorio raíz del proyecto:

```bash
python src/setup_check.py
```

3. Verifica que el resultado muestra "Entorno configurado correctamente".

4. Ejecuta el formateador `black` sobre el script para verificar que cumple con el estilo de código:

```bash
black --check src/setup_check.py
```

5. Ejecuta `mypy` para verificar el tipado estático:

```bash
mypy src/setup_check.py
```

### Salida esperada

Salida de `python src/setup_check.py`:

```
============================================================
  REPORTE DE VERIFICACIÓN DEL ENTORNO
============================================================

📐 INFORMACIÓN DE PYTHON
----------------------------------------
  version              → 3.12.1
  implementacion       → CPython
  ejecutable           → /home/usuario/automation_project/.venv/bin/python
  ruta_prefijo         → /home/usuario/automation_project/.venv

💻 INFORMACIÓN DEL SISTEMA
----------------------------------------
  sistema              → Linux
  version_so           → #1 SMP PREEMPT_DYNAMIC ...
  arquitectura         → x86_64
  nombre_nodo          → mi-maquina

🔒 ENTORNO VIRTUAL
----------------------------------------
  activo               → True
  ruta_venv            → /home/usuario/automation_project/.venv
  ruta_base            → /usr/local

📁 ESTRUCTURA DE DIRECTORIOS
----------------------------------------
  [✓] src                  → OK
  [✓] tests                → OK
  [✓] data/input           → OK
  [✓] data/output          → OK
  [✓] logs                 → OK
  [✓] backups              → OK

============================================================
  ✓ RESULTADO: Entorno configurado correctamente
============================================================
```

Salida de `black --check`:

```
All done! ✨ 🍰 ✨
1 file would be left unchanged.
```

Salida de `mypy`:

```
Success: no issues found in 1 source file
```

### Verificación

- El script reporta Python 3.12.1
- El entorno virtual aparece como "activo: True"
- Los 6 directorios muestran estado "OK"
- El resultado final es "Entorno configurado correctamente"
- `black` no reporta cambios necesarios
- `mypy` no reporta errores de tipo

---

## Paso 8: Realizar el primer commit en Git

### Objetivo

Registrar el estado inicial del proyecto en el historial de Git con un commit descriptivo.

### Instrucciones

1. Configura tu identidad en Git (si no lo has hecho previamente):

```bash
git config user.name "Tu Nombre"
git config user.email "tu.email@ejemplo.com"
```

2. Añade todos los archivos al staging area:

```bash
git add .
```

3. Verifica qué archivos se incluirán en el commit:

```bash
git status
```

4. Realiza el primer commit:

```bash
git commit -m "feat: configuración inicial del entorno de desarrollo

- Estructura de directorios: src/, tests/, data/, logs/, backups/
- Entorno virtual .venv con Python 3.12.1
- Dependencias: black 24.3.0, pytest 8.1.1, mypy 1.9.0
- Script setup_check.py para verificación del entorno
- Archivo .gitignore para proyecto Python"
```

5. Verifica el historial de commits:

```bash
git log --oneline
```

### Salida esperada

Salida de `git status` (antes del commit):

```
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   .gitignore
	new file:   requirements.txt
	new file:   src/setup_check.py
```

Salida de `git log --oneline`:

```
abc1234 feat: configuración inicial del entorno de desarrollo
```

### Verificación

- El commit se creó exitosamente (sin errores)
- `git log` muestra exactamente un commit
- Los archivos `.venv/`, `__pycache__/` y `data/output/` **no** aparecen en el commit (están excluidos por `.gitignore`)

---

## Validación y pruebas

Ejecuta la siguiente secuencia de comandos para validar que todo el laboratorio se completó correctamente:

```bash
# 1. Verificar que estamos en el directorio correcto
pwd
# Esperado: /home/usuario/automation_project (o equivalente)

# 2. Verificar entorno virtual activo
python -c "import sys; assert sys.prefix != sys.base_prefix, 'venv no activo'"
echo "✓ Entorno virtual activo"

# 3. Verificar versión de Python
python -c "import platform; v = platform.python_version(); assert v == '3.12.1', f'Versión incorrecta: {v}'"
echo "✓ Python 3.12.1 verificado"

# 4. Verificar paquetes instalados
python -c "import black; assert black.__version__ == '24.3.0'"
echo "✓ black 24.3.0 instalado"

python -c "import pytest; assert pytest.__version__ == '8.1.1'"
echo "✓ pytest 8.1.1 instalado"

python -c "import mypy; assert mypy.version.__version__ == '1.9.0'"
echo "✓ mypy 1.9.0 instalado"

# 5. Verificar estructura de directorios
python -c "
import os
dirs = ['src', 'tests', 'data/input', 'data/output', 'logs', 'backups']
for d in dirs:
    assert os.path.isdir(d), f'Falta directorio: {d}'
print('✓ Estructura de directorios completa')
"

# 6. Verificar archivos clave
python -c "
import os
archivos = ['requirements.txt', '.gitignore', 'src/setup_check.py']
for a in archivos:
    assert os.path.isfile(a), f'Falta archivo: {a}'
print('✓ Archivos del proyecto presentes')
"

# 7. Ejecutar el script de verificación
python src/setup_check.py

# 8. Verificar repositorio Git
git log --oneline
echo "✓ Repositorio Git inicializado con commit"
```

Si todos los comandos se ejecutan sin errores y muestran las marcas `✓`, el laboratorio está completado exitosamente.

---

## Solución de problemas

### Problema 1: Error "python: command not found" o versión incorrecta

**Síntomas:**

```
bash: python: command not found
```

O bien:

```
Python 3.11.5
```

**Causa:** Python 3.12.1 no está en el PATH del sistema, o existe otra versión de Python que tiene prioridad. En macOS/Linux es común que el comando `python` apunte a Python 2.x o no exista, mientras que `python3` apunta a una versión diferente a 3.12.1.

**Solución:**

1. Verifica qué versiones de Python están disponibles:

```bash
# macOS/Linux
which python3
python3 --version
ls /usr/local/bin/python3*

# Windows PowerShell
Get-Command python | Select-Object Source
py -0  # Lista versiones instaladas por el launcher
```

2. Si tienes múltiples versiones, usa la ruta completa para crear el entorno virtual:

```bash
# macOS/Linux (ejemplo con ruta específica)
/usr/local/bin/python3.12 -m venv .venv

# Windows (usando el launcher py)
py -3.12 -m venv .venv
```

3. Una vez creado el entorno virtual con la versión correcta, `python` dentro del entorno siempre apuntará a 3.12.1.

---

### Problema 2: Error de permisos al activar el entorno virtual en Windows PowerShell

**Síntomas:**

```
.venv\Scripts\activate : File .venv\Scripts\Activate.ps1 cannot be loaded because
running scripts is disabled on this system.
```

**Causa:** La política de ejecución de PowerShell está configurada como `Restricted` por defecto en Windows, lo que impide ejecutar scripts `.ps1` incluyendo el script de activación del entorno virtual.

**Solución:**

1. Cambia la política de ejecución para el usuario actual:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

2. Confirma el cambio cuando se solicite (escribe `Y` y presiona Enter).

3. Intenta activar el entorno virtual nuevamente:

```powershell
.venv\Scripts\activate
```

4. Verifica que la política se aplicó:

```powershell
Get-ExecutionPolicy -Scope CurrentUser
# Esperado: RemoteSigned
```

> **Alternativa:** Si no puedes cambiar la política, usa `cmd.exe` en lugar de PowerShell y ejecuta `.venv\Scripts\activate.bat`.

---

## Limpieza

Este laboratorio es **fundacional** — no se debe limpiar ningún recurso creado. La estructura de directorios, el entorno virtual, el archivo `requirements.txt` y el script `setup_check.py` serán utilizados en todos los laboratorios posteriores del curso.

Si por alguna razón necesitas reiniciar el laboratorio desde cero:

```bash
# ⚠️ SOLO si necesitas empezar de nuevo — esto elimina TODO el proyecto
cd ~
rm -rf automation_project   # macOS/Linux
# Remove-Item -Recurse -Force automation_project  # Windows
```

---

## Resumen

En este laboratorio completaste la configuración completa del entorno de desarrollo para el curso:

| Logro | Detalle |
|-------|---------|
| Python verificado | Versión 3.12.1 confirmada |
| Entorno virtual | `.venv` creado y funcional |
| Dependencias | black 24.3.0, pytest 8.1.1, mypy 1.9.0 instalados |
| Estructura | 6 directorios estándar creados |
| Script funcional | `setup_check.py` ejecutándose con módulos `sys`, `os`, `platform` |
| Control de versiones | Repositorio Git inicializado con primer commit |

### Conceptos clave aplicados

- **Tipado dinámico**: Las variables en el script (`info_python`, `estructura`) reciben su tipo según el valor asignado (diccionarios), sin declaración explícita
- **Indentación como estructura**: Los bloques `if/else` y `for` en el script dependen de la indentación correcta de 4 espacios
- **Ejecución desde terminal**: El script se invoca con `python src/setup_check.py`, siguiendo el ciclo lectura → análisis → bytecode → ejecución
- **f-strings**: Utilizados extensivamente en `imprimir_reporte()` para formatear la salida de forma legible
- **Módulos estándar**: `sys`, `os` y `platform` proporcionan acceso a información del sistema sin dependencias externas

### Recursos adicionales

- [Documentación oficial de venv](https://docs.python.org/3/library/venv.html)
- [PEP 405 — Python Virtual Environments](https://peps.python.org/pep-0405/)
- [Documentación del módulo platform](https://docs.python.org/3/library/platform.html)
- [Git — Configuración inicial](https://git-scm.com/book/es/v2/Inicio---Sobre-el-Control-de-Versiones-Configurando-Git-por-primera-vez)
