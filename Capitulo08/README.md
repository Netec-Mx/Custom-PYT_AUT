# Práctica 8 — Automatización de una aplicación web

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 38 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En esta práctica construirás un módulo completo de automatización web usando Selenium 4.21.0 con Google Chrome en modo headless. Automatizarás el flujo de login y llenado de formularios en el sitio público de pruebas `https://the-internet.herokuapp.com`, implementando esperas explícitas, capturas de pantalla con timestamp, extracción de texto mediante OCR con pytesseract, y logging estructurado con rotación de archivos. El resultado se integrará al pipeline existente del proyecto `auto_reporter/`.

## Objetivos de Aprendizaje

- [ ] Configurar Selenium WebDriver con ChromeOptions para ejecución headless y automatizar navegación, login y llenado de formularios
- [ ] Implementar estrategias de localización robustas (By.ID, By.CSS_SELECTOR, By.XPATH) combinadas con esperas explícitas WebDriverWait
- [ ] Capturar pantallas automáticas en puntos clave y extraer texto de imágenes mediante pytesseract
- [ ] Registrar cada acción de Selenium en un archivo de log con RotatingFileHandler (1 MB, 3 backups)
- [ ] Integrar la función `run_web_automation()` al scheduler del pipeline existente

## Prerrequisitos

### Conocimientos previos

- Lab 07 completado con módulos `src/` refactorizados y pipeline funcional
- Comprensión de la arquitectura Selenium (Lección 8.1): capas, protocolo W3C, ciclo de vida de sesión
- Familiaridad con el módulo `logging` de Python y manejo de archivos CSV con `csv` estándar

### Acceso y software requerido

| Software | Versión | Verificación |
|----------|---------|--------------|
| Python | 3.12.1 | `python --version` |
| selenium | 4.21.0 | `pip show selenium` |
| Google Chrome | 125.0.6422.112 | `google-chrome --version` o menú Chrome |
| Pillow | 10.3.0 | `pip show Pillow` |
| pytesseract | 0.3.10 | `pip show pytesseract` |
| Tesseract-OCR | 5.x | `tesseract --version` |
| Conexión a Internet | — | Acceso a `the-internet.herokuapp.com` |

## Entorno del Laboratorio

### Estructura de directorios objetivo

```
~/automation_project/
├── src/
│   └── web_automation.py          ← Módulo principal (se crea en este lab)
├── tests/
│   └── test_web_automation.py     ← Pruebas básicas (se crea en este lab)
├── data/
│   └── input/
│       └── employees.csv          ← Existente de labs anteriores
├── logs/
│   └── selenium_run.log           ← Generado por el módulo
├── screenshots/                   ← Directorio nuevo para capturas
├── scheduler_runner.py            ← Existente, se modifica
└── requirements.txt
```

### Comandos de preparación del entorno

```bash
# Navegar al directorio del proyecto
cd ~/automation_project

# Activar entorno virtual
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate

# Verificar versión de Python
python --version
# Esperado: Python 3.12.1

# Instalar dependencias adicionales para este lab
pip install selenium==4.21.0 Pillow==10.3.0 pytesseract==0.3.10

# Crear directorio de screenshots
mkdir -p screenshots

# Verificar instalaciones
python -c "import selenium; print(selenium.__version__)"
python -c "import pytesseract; print(pytesseract.get_tesseract_version())"
```

> **Nota sobre Tesseract-OCR:** En Ubuntu: `sudo apt install tesseract-ocr`. En macOS: `brew install tesseract`. En Windows: descargar el instalador desde https://github.com/UB-Mannheim/tesseract/wiki y agregar al PATH.

---

## Paso a Paso

### Paso 1: Configurar el módulo de logging con rotación

**Objetivo:** Crear la infraestructura de logging que registrará cada acción de Selenium con timestamps, niveles y rotación automática de archivos.

**Instrucciones:**

1. Crea el archivo `src/web_automation.py`:

```bash
touch src/web_automation.py
```

2. Escribe el siguiente código base con la configuración de logging:

```python
"""
Módulo de automatización web con Selenium.
Automatiza login y llenado de formularios en the-internet.herokuapp.com.
"""

import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path
from datetime import datetime

# Rutas del proyecto
PROJECT_ROOT = Path(__file__).resolve().parent.parent
LOGS_DIR = PROJECT_ROOT / "logs"
SCREENSHOTS_DIR = PROJECT_ROOT / "screenshots"

# Crear directorios si no existen
LOGS_DIR.mkdir(exist_ok=True)
SCREENSHOTS_DIR.mkdir(exist_ok=True)


def configurar_logger() -> logging.Logger:
    """Configura logger con RotatingFileHandler (1MB, 3 backups)."""
    logger = logging.getLogger("web_automation")
    logger.setLevel(logging.INFO)

    # Evitar duplicación de handlers si se llama múltiples veces
    if logger.handlers:
        return logger

    # Handler con rotación: 1 MB máximo, 3 archivos de respaldo
    file_handler = RotatingFileHandler(
        filename=LOGS_DIR / "selenium_run.log",
        maxBytes=1_048_576,  # 1 MB
        backupCount=3,
        encoding="utf-8",
    )
    file_handler.setLevel(logging.INFO)

    # Formato estructurado con timestamp, nivel y mensaje
    formatter = logging.Formatter(
        fmt="%(asctime)s | %(levelname)-8s | %(funcName)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )
    file_handler.setFormatter(formatter)

    # Handler de consola para feedback inmediato
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)

    logger.addHandler(file_handler)
    logger.addHandler(console_handler)

    return logger


# Instancia global del logger
logger = configurar_logger()
```

3. Verifica que el módulo se importa correctamente:

```bash
python -c "from src.web_automation import logger; logger.info('Logger configurado correctamente')"
```

**Resultado esperado:**

```
2024-XX-XX HH:MM:SS | INFO     | <module> | Logger configurado correctamente
```

**Verificación:** Confirma que se creó el archivo `logs/selenium_run.log`:

```bash
cat logs/selenium_run.log
```

Debe contener la línea de log con el timestamp.

---

### Paso 2: Configurar WebDriver con ChromeOptions

**Objetivo:** Implementar una función factory que cree el WebDriver con opciones optimizadas para automatización (headless, tamaño de ventana fijo, flags de estabilidad).

**Instrucciones:**

1. Agrega las siguientes importaciones y función al archivo `src/web_automation.py`:

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import (
    TimeoutException,
    NoSuchElementException,
    WebDriverException,
)


def crear_driver(headless: bool = True) -> webdriver.Chrome:
    """
    Crea y retorna una instancia de Chrome WebDriver configurada.

    Args:
        headless: Si True, ejecuta Chrome sin interfaz gráfica.

    Returns:
        Instancia configurada de webdriver.Chrome.
    """
    opciones = Options()

    if headless:
        opciones.add_argument("--headless=new")

    # Configuración de ventana y estabilidad
    opciones.add_argument("--window-size=1920,1080")
    opciones.add_argument("--no-sandbox")
    opciones.add_argument("--disable-dev-shm-usage")
    opciones.add_argument("--disable-gpu")
    opciones.add_argument("--disable-extensions")

    logger.info(f"Creando driver Chrome (headless={headless})")

    driver = webdriver.Chrome(options=opciones)
    driver.implicitly_wait(0)  # Desactivar esperas implícitas; usaremos explícitas

    logger.info(f"Driver creado exitosamente. Sesión: {driver.session_id}")
    return driver
```

2. Prueba la creación del driver:

```bash
python -c "
from src.web_automation import crear_driver, logger
driver = crear_driver(headless=True)
driver.get('https://the-internet.herokuapp.com')
logger.info(f'Título: {driver.title}')
driver.quit()
logger.info('Driver cerrado correctamente')
"
```

**Resultado esperado:**

```
2024-XX-XX HH:MM:SS | INFO     | crear_driver | Creando driver Chrome (headless=True)
2024-XX-XX HH:MM:SS | INFO     | crear_driver | Driver creado exitosamente. Sesión: abc123...
2024-XX-XX HH:MM:SS | INFO     | <module> | Título: The Internet
2024-XX-XX HH:MM:SS | INFO     | <module> | Driver cerrado correctamente
```

**Verificación:** Si el título impreso es `The Internet`, la conexión y el driver funcionan correctamente.

---

### Paso 3: Implementar captura de pantalla con timestamp

**Objetivo:** Crear una función reutilizable que guarde screenshots con nombres únicos basados en timestamp, facilitando la trazabilidad de cada ejecución.

**Instrucciones:**

1. Agrega la siguiente función a `src/web_automation.py`:

```python
def capturar_pantalla(driver: webdriver.Chrome, nombre_paso: str) -> Path:
    """
    Captura screenshot del estado actual del navegador.

    Args:
        driver: Instancia activa de WebDriver.
        nombre_paso: Identificador descriptivo del paso actual.

    Returns:
        Path al archivo de screenshot guardado.
    """
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    nombre_archivo = f"{timestamp}_{nombre_paso}.png"
    ruta_screenshot = SCREENSHOTS_DIR / nombre_archivo

    driver.save_screenshot(str(ruta_screenshot))
    logger.info(f"Screenshot guardado: {ruta_screenshot.name}")

    return ruta_screenshot
```

2. Verifica la función capturando una pantalla de prueba:

```bash
python -c "
from src.web_automation import crear_driver, capturar_pantalla
driver = crear_driver(headless=True)
driver.get('https://the-internet.herokuapp.com')
ruta = capturar_pantalla(driver, 'pagina_principal')
print(f'Archivo creado: {ruta}')
driver.quit()
"
```

**Resultado esperado:**

```
2024-XX-XX HH:MM:SS | INFO     | capturar_pantalla | Screenshot guardado: 20240601_143022_pagina_principal.png
Archivo creado: /home/usuario/automation_project/screenshots/20240601_143022_pagina_principal.png
```

**Verificación:**

```bash
ls screenshots/*.png
```

Debe mostrar al menos un archivo `.png` con el patrón de nombre esperado.

---

### Paso 4: Automatizar el flujo de login

**Objetivo:** Automatizar el proceso completo de login en `/login` usando esperas explícitas, localización de elementos por ID y CSS_SELECTOR, y verificación del resultado con assertions.

**Instrucciones:**

1. Agrega la función de login a `src/web_automation.py`:

```python
BASE_URL = "https://the-internet.herokuapp.com"


def automatizar_login(driver: webdriver.Chrome) -> bool:
    """
    Automatiza el login en the-internet.herokuapp.com/login.

    Credenciales: tomsmith / SuperSecretPassword!

    Args:
        driver: Instancia activa de WebDriver.

    Returns:
        True si el login fue exitoso, False en caso contrario.
    """
    url_login = f"{BASE_URL}/login"
    logger.info(f"Navegando a: {url_login}")
    driver.get(url_login)

    capturar_pantalla(driver, "01_login_pagina_cargada")

    try:
        # Esperar a que el campo username esté presente (máximo 10 segundos)
        wait = WebDriverWait(driver, 10)

        # Localizar elementos por ID
        campo_usuario = wait.until(
            EC.presence_of_element_located((By.ID, "username"))
        )
        campo_password = wait.until(
            EC.presence_of_element_located((By.ID, "password"))
        )

        # Localizar botón de submit por CSS Selector
        boton_login = wait.until(
            EC.element_to_be_clickable(
                (By.CSS_SELECTOR, "button[type='submit']")
            )
        )

        # Ingresar credenciales
        campo_usuario.clear()
        campo_usuario.send_keys("tomsmith")
        logger.info("Usuario ingresado: tomsmith")

        campo_password.clear()
        campo_password.send_keys("SuperSecretPassword!")
        logger.info("Password ingresado: ********")

        capturar_pantalla(driver, "02_login_credenciales_ingresadas")

        # Hacer clic en el botón de login
        boton_login.click()
        logger.info("Botón de login presionado")

        # Verificar login exitoso: esperar mensaje de éxito
        mensaje_exito = wait.until(
            EC.presence_of_element_located(
                (By.CSS_SELECTOR, "div.flash.success")
            )
        )

        texto_mensaje = mensaje_exito.text
        logger.info(f"Mensaje recibido: {texto_mensaje}")

        capturar_pantalla(driver, "03_login_exitoso")

        # Assertion: verificar que el mensaje contiene texto esperado
        assert "You logged into a secure area!" in texto_mensaje, (
            f"Mensaje inesperado: {texto_mensaje}"
        )

        logger.info("✓ Login completado exitosamente")
        return True

    except TimeoutException:
        logger.error("Timeout esperando elementos de login")
        capturar_pantalla(driver, "error_login_timeout")
        return False
    except AssertionError as e:
        logger.error(f"Assertion fallida en login: {e}")
        capturar_pantalla(driver, "error_login_assertion")
        return False
    except WebDriverException as e:
        logger.error(f"Error de WebDriver en login: {e}")
        capturar_pantalla(driver, "error_login_webdriver")
        return False
```

2. Ejecuta la función de login:

```bash
python -c "
from src.web_automation import crear_driver, automatizar_login
driver = crear_driver(headless=True)
resultado = automatizar_login(driver)
print(f'Login exitoso: {resultado}')
driver.quit()
"
```

**Resultado esperado:**

```
2024-XX-XX HH:MM:SS | INFO     | automatizar_login | Navegando a: https://the-internet.herokuapp.com/login
2024-XX-XX HH:MM:SS | INFO     | capturar_pantalla | Screenshot guardado: 20240601_143100_01_login_pagina_cargada.png
2024-XX-XX HH:MM:SS | INFO     | automatizar_login | Usuario ingresado: tomsmith
2024-XX-XX HH:MM:SS | INFO     | automatizar_login | Password ingresado: ********
2024-XX-XX HH:MM:SS | INFO     | capturar_pantalla | Screenshot guardado: 20240601_143101_02_login_credenciales_ingresadas.png
2024-XX-XX HH:MM:SS | INFO     | automatizar_login | Botón de login presionado
2024-XX-XX HH:MM:SS | INFO     | automatizar_login | Mensaje recibido: You logged into a secure area! ...
2024-XX-XX HH:MM:SS | INFO     | capturar_pantalla | Screenshot guardado: 20240601_143102_03_login_exitoso.png
2024-XX-XX HH:MM:SS | INFO     | automatizar_login | ✓ Login completado exitosamente
Login exitoso: True
```

**Verificación:** El valor impreso debe ser `Login exitoso: True` y deben existir 3 screenshots nuevos en `screenshots/`.

---

### Paso 5: Automatizar llenado de formulario con datos del CSV

**Objetivo:** Leer datos del archivo `employees.csv` y usarlos para llenar dinámicamente el formulario de `/forgot_password`, demostrando automatización basada en datos externos.

**Instrucciones:**

1. Agrega la función de llenado de formulario a `src/web_automation.py`:

```python
import csv


def leer_emails_csv(ruta_csv: Path) -> list[str]:
    """
    Lee el CSV de empleados y genera emails basados en el nombre.

    Args:
        ruta_csv: Ruta al archivo CSV de empleados.

    Returns:
        Lista de emails generados (nombre.lower()@empresa.com).
    """
    emails = []
    with open(ruta_csv, "r", encoding="utf-8") as archivo:
        reader = csv.DictReader(archivo)
        for fila in reader:
            nombre = fila["nombre"].strip().lower().replace(" ", ".")
            email = f"{nombre}@empresa.com"
            emails.append(email)

    logger.info(f"Leídos {len(emails)} emails del CSV: {ruta_csv.name}")
    return emails


def automatizar_formulario(driver: webdriver.Chrome, emails: list[str], max_envios: int = 3) -> int:
    """
    Automatiza el llenado del formulario forgot_password con emails del CSV.

    Args:
        driver: Instancia activa de WebDriver.
        emails: Lista de emails a usar en el formulario.
        max_envios: Número máximo de formularios a enviar.

    Returns:
        Cantidad de formularios enviados exitosamente.
    """
    url_formulario = f"{BASE_URL}/forgot_password"
    envios_exitosos = 0

    for i, email in enumerate(emails[:max_envios]):
        logger.info(f"Formulario {i+1}/{max_envios}: navegando a {url_formulario}")
        driver.get(url_formulario)

        try:
            wait = WebDriverWait(driver, 10)

            # Localizar campo de email por ID
            campo_email = wait.until(
                EC.presence_of_element_located((By.ID, "email"))
            )

            # Limpiar y escribir el email
            campo_email.clear()
            campo_email.send_keys(email)
            logger.info(f"Email ingresado: {email}")

            # Localizar botón submit por XPATH
            boton_submit = wait.until(
                EC.element_to_be_clickable(
                    (By.XPATH, "//button[@type='submit'] | //i[contains(@class,'icon-2x')]/..")
                )
            )

            capturar_pantalla(driver, f"04_formulario_{i+1}_llenado")

            # Enviar formulario
            boton_submit.click()
            logger.info(f"Formulario {i+1} enviado para: {email}")

            # Esperar respuesta (la página cambia tras envío)
            wait.until(
                EC.url_changes(url_formulario)
            )

            capturar_pantalla(driver, f"05_formulario_{i+1}_enviado")
            envios_exitosos += 1

        except TimeoutException:
            logger.warning(f"Timeout en formulario {i+1} para {email}")
            capturar_pantalla(driver, f"error_formulario_{i+1}_timeout")
        except WebDriverException as e:
            logger.error(f"Error WebDriver en formulario {i+1}: {e}")
            capturar_pantalla(driver, f"error_formulario_{i+1}")

    logger.info(f"Formularios enviados exitosamente: {envios_exitosos}/{max_envios}")
    return envios_exitosos
```

2. Ejecuta el llenado de formularios:

```bash
python -c "
from pathlib import Path
from src.web_automation import crear_driver, leer_emails_csv, automatizar_formulario

csv_path = Path('data/input/employees.csv')
emails = leer_emails_csv(csv_path)
print(f'Emails generados: {len(emails)}')
print(f'Primeros 3: {emails[:3]}')

driver = crear_driver(headless=True)
enviados = automatizar_formulario(driver, emails, max_envios=3)
print(f'Formularios enviados: {enviados}')
driver.quit()
"
```

**Resultado esperado:**

```
Emails generados: 50
Primeros 3: ['juan.perez@empresa.com', 'maria.garcia@empresa.com', 'carlos.lopez@empresa.com']
...
Formularios enviados: 3
```

**Verificación:** Deben existir screenshots con prefijo `04_` y `05_` en la carpeta `screenshots/`.

---

### Paso 6: Implementar OCR con pytesseract sobre capturas

**Objetivo:** Recortar una región de interés de un screenshot y extraer texto visible usando pytesseract, demostrando la integración de Selenium con procesamiento de imágenes.

**Instrucciones:**

1. Agrega la función de OCR a `src/web_automation.py`:

```python
from PIL import Image
import pytesseract


def extraer_texto_screenshot(ruta_imagen: Path, region: tuple[int, int, int, int] | None = None) -> str:
    """
    Extrae texto de un screenshot usando pytesseract OCR.

    Args:
        ruta_imagen: Ruta al archivo PNG del screenshot.
        region: Tupla (left, top, right, bottom) para recortar.
                Si None, procesa la imagen completa.

    Returns:
        Texto extraído de la imagen.
    """
    logger.info(f"Procesando OCR en: {ruta_imagen.name}")

    imagen = Image.open(ruta_imagen)

    if region:
        imagen = imagen.crop(region)
        logger.info(f"Imagen recortada a región: {region}")

    # Extraer texto con pytesseract
    texto = pytesseract.image_to_string(imagen, lang="eng")
    texto_limpio = texto.strip()

    logger.info(f"Texto extraído ({len(texto_limpio)} caracteres): {texto_limpio[:100]}...")

    return texto_limpio
```

2. Prueba la extracción OCR sobre el screenshot del login exitoso:

```bash
python -c "
from pathlib import Path
from src.web_automation import extraer_texto_screenshot, SCREENSHOTS_DIR

# Buscar el screenshot del login exitoso
screenshots = sorted(SCREENSHOTS_DIR.glob('*03_login_exitoso*.png'))
if screenshots:
    ruta = screenshots[-1]  # El más reciente
    # Región superior donde aparece el mensaje flash (aproximada)
    texto = extraer_texto_screenshot(ruta, region=(0, 100, 1920, 300))
    print(f'Texto extraído: {texto}')
else:
    print('No se encontró screenshot de login exitoso. Ejecuta el Paso 4 primero.')
"
```

**Resultado esperado:**

```
2024-XX-XX HH:MM:SS | INFO     | extraer_texto_screenshot | Procesando OCR en: 20240601_143102_03_login_exitoso.png
2024-XX-XX HH:MM:SS | INFO     | extraer_texto_screenshot | Imagen recortada a región: (0, 100, 1920, 300)
2024-XX-XX HH:MM:SS | INFO     | extraer_texto_screenshot | Texto extraído (42 caracteres): You logged into a secure area!...
Texto extraído: You logged into a secure area!
```

**Verificación:** El texto extraído debe contener al menos parcialmente el mensaje "You logged into a secure area" o fragmentos reconocibles. La precisión del OCR puede variar según la resolución y el renderizado headless.

> **Nota:** Si pytesseract no reconoce texto correctamente, ajusta la región de recorte. En modo headless con `--window-size=1920,1080`, el mensaje flash suele estar en la parte superior de la página.

---

### Paso 7: Crear la función principal de orquestación

**Objetivo:** Unificar todas las funciones en una función `run_web_automation()` que ejecute el flujo completo y retorne un resumen de resultados para integración con el pipeline.

**Instrucciones:**

1. Agrega la función orquestadora al final de `src/web_automation.py`:

```python
def run_web_automation() -> dict:
    """
    Ejecuta el flujo completo de automatización web.

    Pasos:
        1. Login en the-internet.herokuapp.com
        2. Llenado de formularios con datos del CSV
        3. OCR sobre captura del login exitoso

    Returns:
        Diccionario con resultados de la ejecución.
    """
    logger.info("=" * 60)
    logger.info("INICIO: Automatización web")
    logger.info("=" * 60)

    resultados = {
        "timestamp": datetime.now().isoformat(),
        "login_exitoso": False,
        "formularios_enviados": 0,
        "texto_ocr": "",
        "screenshots_generados": 0,
        "errores": [],
    }

    # Contar screenshots antes de iniciar
    screenshots_antes = len(list(SCREENSHOTS_DIR.glob("*.png")))

    driver = None
    try:
        # Crear driver
        driver = crear_driver(headless=True)

        # Paso 1: Login
        resultados["login_exitoso"] = automatizar_login(driver)

        # Paso 2: Llenado de formularios
        csv_path = PROJECT_ROOT / "data" / "input" / "employees.csv"
        if csv_path.exists():
            emails = leer_emails_csv(csv_path)
            resultados["formularios_enviados"] = automatizar_formulario(
                driver, emails, max_envios=3
            )
        else:
            logger.warning(f"CSV no encontrado: {csv_path}")
            resultados["errores"].append(f"CSV no encontrado: {csv_path}")

        # Paso 3: OCR sobre screenshot de login
        screenshots_login = sorted(
            SCREENSHOTS_DIR.glob("*03_login_exitoso*.png")
        )
        if screenshots_login:
            texto = extraer_texto_screenshot(
                screenshots_login[-1], region=(0, 100, 1920, 300)
            )
            resultados["texto_ocr"] = texto

    except WebDriverException as e:
        error_msg = f"Error crítico de WebDriver: {e}"
        logger.error(error_msg)
        resultados["errores"].append(error_msg)

    except Exception as e:
        error_msg = f"Error inesperado: {e}"
        logger.error(error_msg)
        resultados["errores"].append(error_msg)

    finally:
        if driver:
            driver.quit()
            logger.info("Driver cerrado correctamente")

    # Contar screenshots generados
    screenshots_despues = len(list(SCREENSHOTS_DIR.glob("*.png")))
    resultados["screenshots_generados"] = screenshots_despues - screenshots_antes

    logger.info("=" * 60)
    logger.info(f"FIN: Automatización web | Login: {resultados['login_exitoso']} | "
                f"Formularios: {resultados['formularios_enviados']} | "
                f"Screenshots: {resultados['screenshots_generados']}")
    logger.info("=" * 60)

    return resultados


# Punto de entrada para ejecución directa
if __name__ == "__main__":
    resultado = run_web_automation()
    print("\n--- RESUMEN ---")
    for clave, valor in resultado.items():
        print(f"  {clave}: {valor}")
```

2. Ejecuta el flujo completo:

```bash
python -m src.web_automation
```

**Resultado esperado:**

```
============================================================
INICIO: Automatización web
============================================================
... (logs de login, formularios, OCR) ...
============================================================
FIN: Automatización web | Login: True | Formularios: 3 | Screenshots: 8
============================================================

--- RESUMEN ---
  timestamp: 2024-06-01T14:31:22.123456
  login_exitoso: True
  formularios_enviados: 3
  texto_ocr: You logged into a secure area!
  screenshots_generados: 8
  errores: []
```

**Verificación:** Todos los campos del resumen deben mostrar valores positivos y la lista de errores debe estar vacía.

---

### Paso 8: Integrar con scheduler_runner.py

**Objetivo:** Agregar la función `run_web_automation()` al pipeline existente del scheduler para que pueda ejecutarse de forma programada junto con las demás tareas.

**Instrucciones:**

1. Edita el archivo `scheduler_runner.py` en la raíz del proyecto y agrega la integración:

```python
# Agregar al inicio del archivo (junto a los otros imports)
from src.web_automation import run_web_automation

# Agregar esta función al flujo del pipeline
def ejecutar_pipeline_completo():
    """Ejecuta todas las tareas del pipeline incluyendo automatización web."""
    resultados_pipeline = {}

    # ... (tareas existentes del Lab 07) ...

    # Tarea: Automatización web
    print("\n[PIPELINE] Ejecutando automatización web...")
    try:
        resultado_web = run_web_automation()
        resultados_pipeline["web_automation"] = resultado_web
        if resultado_web["login_exitoso"]:
            print("[PIPELINE] ✓ Automatización web completada")
        else:
            print("[PIPELINE] ⚠ Automatización web con advertencias")
    except Exception as e:
        print(f"[PIPELINE] ✗ Error en automatización web: {e}")
        resultados_pipeline["web_automation"] = {"error": str(e)}

    return resultados_pipeline
```

2. Si tu `scheduler_runner.py` no tiene la estructura anterior, crea una versión mínima funcional:

```python
#!/usr/bin/env python
"""Scheduler runner - Pipeline de automatización."""

from src.web_automation import run_web_automation


def ejecutar_pipeline_completo() -> dict:
    """Ejecuta el pipeline completo de automatización."""
    resultados = {}

    print("=" * 50)
    print("PIPELINE DE AUTOMATIZACIÓN")
    print("=" * 50)

    # Tarea: Automatización web
    print("\n[TAREA] Automatización web...")
    resultados["web_automation"] = run_web_automation()

    print("\n" + "=" * 50)
    print("PIPELINE FINALIZADO")
    print("=" * 50)

    return resultados


if __name__ == "__main__":
    resultado = ejecutar_pipeline_completo()
    print(f"\nResultados: {resultado.get('web_automation', {}).get('login_exitoso')}")
```

3. Ejecuta el pipeline:

```bash
python scheduler_runner.py
```

**Resultado esperado:**

```
==================================================
PIPELINE DE AUTOMATIZACIÓN
==================================================

[TAREA] Automatización web...
... (logs de la automatización) ...

==================================================
PIPELINE FINALIZADO
==================================================

Resultados: True
```

**Verificación:** El pipeline debe completarse sin excepciones no manejadas y el resultado de login debe ser `True`.

---

## Validación y Pruebas

### Prueba automatizada

Crea el archivo `tests/test_web_automation.py`:

```python
"""Tests para el módulo de automatización web."""

import pytest
from pathlib import Path
from unittest.mock import patch, MagicMock
from src.web_automation import (
    configurar_logger,
    capturar_pantalla,
    leer_emails_csv,
    SCREENSHOTS_DIR,
    LOGS_DIR,
    PROJECT_ROOT,
)


class TestLogger:
    """Tests para la configuración de logging."""

    def test_logger_crea_archivo(self):
        """Verifica que el logger crea el archivo de log."""
        log_file = LOGS_DIR / "selenium_run.log"
        assert log_file.exists(), "El archivo selenium_run.log no fue creado"

    def test_logger_tiene_handlers(self):
        """Verifica que el logger tiene handlers configurados."""
        logger = configurar_logger()
        assert len(logger.handlers) >= 2, "Logger debe tener al menos 2 handlers"


class TestLeerCSV:
    """Tests para lectura de emails desde CSV."""

    def test_leer_emails_retorna_lista(self):
        """Verifica que se generan emails desde el CSV."""
        csv_path = PROJECT_ROOT / "data" / "input" / "employees.csv"
        if csv_path.exists():
            emails = leer_emails_csv(csv_path)
            assert len(emails) > 0, "Debe retornar al menos un email"
            assert "@empresa.com" in emails[0], "Emails deben tener dominio correcto"

    def test_formato_email(self):
        """Verifica el formato de los emails generados."""
        csv_path = PROJECT_ROOT / "data" / "input" / "employees.csv"
        if csv_path.exists():
            emails = leer_emails_csv(csv_path)
            for email in emails:
                assert "@" in email, f"Email inválido: {email}"
                assert " " not in email, f"Email con espacios: {email}"


class TestScreenshots:
    """Tests para verificar screenshots generados."""

    def test_directorio_screenshots_existe(self):
        """Verifica que el directorio de screenshots existe."""
        assert SCREENSHOTS_DIR.exists(), "Directorio screenshots/ no existe"

    def test_screenshots_generados(self):
        """Verifica que se generaron screenshots durante la ejecución."""
        screenshots = list(SCREENSHOTS_DIR.glob("*.png"))
        assert len(screenshots) > 0, (
            "No se encontraron screenshots. Ejecuta run_web_automation() primero."
        )

    def test_screenshot_tiene_contenido(self):
        """Verifica que los screenshots no están vacíos."""
        screenshots = list(SCREENSHOTS_DIR.glob("*.png"))
        if screenshots:
            for screenshot in screenshots[:3]:
                assert screenshot.stat().st_size > 1000, (
                    f"Screenshot vacío o corrupto: {screenshot.name}"
                )
```

Ejecuta las pruebas:

```bash
pytest tests/test_web_automation.py -v
```

**Resultado esperado:**

```
tests/test_web_automation.py::TestLogger::test_logger_crea_archivo PASSED
tests/test_web_automation.py::TestLogger::test_logger_tiene_handlers PASSED
tests/test_web_automation.py::TestLeerCSV::test_leer_emails_retorna_lista PASSED
tests/test_web_automation.py::TestLeerCSV::test_formato_email PASSED
tests/test_web_automation.py::TestScreenshots::test_directorio_screenshots_existe PASSED
tests/test_web_automation.py::TestScreenshots::test_screenshots_generados PASSED
tests/test_web_automation.py::TestScreenshots::test_screenshot_tiene_contenido PASSED
```

### Verificación integral final

```bash
# Verificar que el log tiene entradas
echo "--- Últimas 10 líneas del log ---"
tail -10 logs/selenium_run.log

# Verificar cantidad de screenshots
echo "--- Screenshots generados ---"
ls -la screenshots/*.png | wc -l

# Verificar tamaño del log (debe ser < 1MB para no rotar aún)
echo "--- Tamaño del log ---"
du -h logs/selenium_run.log
```

---

## Solución de Problemas

### Problema 1: WebDriverException — Chrome not reachable

**Síntomas:**
```
selenium.common.exceptions.WebDriverException: Message: unknown error: Chrome failed to start: exited abnormally.
```

**Causa:** Chrome no está instalado, la versión es incompatible con chromedriver, o faltan dependencias del sistema en Linux (librerías gráficas necesarias incluso en modo headless).

**Solución:**

```bash
# Verificar que Chrome está instalado
google-chrome --version  # Linux
# o
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --version  # macOS

# En Ubuntu, instalar dependencias faltantes
sudo apt update
sudo apt install -y libglib2.0-0 libnss3 libgconf-2-4 libfontconfig1 libxss1

# Verificar que Selenium Manager puede resolver el driver
python -c "from selenium import webdriver; print(webdriver.Chrome(options=webdriver.ChromeOptions()))"

# Si persiste, forzar la descarga del driver
pip install --force-reinstall selenium==4.21.0
```

---

### Problema 2: pytesseract.TesseractNotFoundError

**Síntomas:**
```
pytesseract.pytesseract.TesseractNotFoundError: tesseract is not installed or it's not in your PATH.
```

**Causa:** El binario de Tesseract-OCR no está instalado en el sistema operativo o no está en el PATH del sistema. El paquete Python `pytesseract` es solo un wrapper; requiere que el motor OCR esté instalado de forma independiente.

**Solución:**

```bash
# Ubuntu/Debian
sudo apt install tesseract-ocr

# macOS (con Homebrew)
brew install tesseract

# Windows: descargar instalador desde
# https://github.com/UB-Mannheim/tesseract/wiki
# Después, configurar la ruta en Python si no está en PATH:
```

Si en Windows Tesseract no está en PATH, agrega esta línea al inicio de `web_automation.py`:

```python
# Solo necesario en Windows si Tesseract no está en PATH
import pytesseract
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'
```

Verificar instalación:

```bash
tesseract --version
python -c "import pytesseract; print(pytesseract.get_tesseract_version())"
```

---

## Limpieza

Después de completar y verificar el lab, ejecuta los siguientes comandos para limpiar artefactos temporales (conservando los archivos de código):

```bash
# Eliminar screenshots de prueba (opcional, se verifican en Lab 09)
# rm screenshots/*.png

# Verificar que el log no excede el tamaño esperado
du -h logs/selenium_run.log

# Confirmar que no quedaron procesos Chrome huérfanos
# Linux/macOS:
ps aux | grep -i chrome | grep -v grep
# Si hay procesos huérfanos:
# pkill -f chromedriver

# Windows (PowerShell):
# Get-Process | Where-Object {$_.Name -like "*chrome*"}
```

> **Importante:** NO elimines los screenshots ni el archivo `selenium_run.log`. Serán verificados en el Lab 09.

---

## Resumen

En esta práctica has construido un módulo completo de automatización web que:

1. **Configura logging profesional** con `RotatingFileHandler` (1 MB, 3 backups) para trazabilidad de cada ejecución
2. **Crea un WebDriver robusto** con ChromeOptions optimizadas para modo headless y CI/CD
3. **Automatiza login** con esperas explícitas (`WebDriverWait`) y verificación por assertion
4. **Llena formularios dinámicamente** con datos leídos de un CSV usando múltiples estrategias de localización (By.ID, By.CSS_SELECTOR, By.XPATH)
5. **Captura screenshots** con timestamp en cada paso crítico para auditoría visual
6. **Extrae texto de imágenes** mediante pytesseract OCR
7. **Integra todo en el pipeline** existente a través de `run_web_automation()`

### Conceptos clave aplicados

| Concepto | Implementación |
|----------|---------------|
| Arquitectura Selenium | Driver → Browser Driver → Chrome (protocolo W3C) |
| Esperas explícitas | `WebDriverWait(driver, 10)` + `EC.presence_of_element_located()` |
| Localización de elementos | `By.ID`, `By.CSS_SELECTOR`, `By.XPATH` |
| Modo headless | `--headless=new` + `--window-size=1920,1080` |
| Logging con rotación | `RotatingFileHandler(maxBytes=1_048_576, backupCount=3)` |
| OCR | `pytesseract.image_to_string()` + `Pillow` para recorte |

### Recursos adicionales

- [Selenium Python Documentation](https://www.selenium.dev/selenium/docs/api/py/)
- [The Internet - Herokuapp (sitio de pruebas)](https://the-internet.herokuapp.com)
- [pytesseract Documentation](https://pypi.org/project/pytesseract/)
- [Python logging — RotatingFileHandler](https://docs.python.org/3/library/logging.handlers.html#rotatingfilehandler)
