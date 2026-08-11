# Práctica 7 — Automatización completa con alertas

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 38 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción general

En esta práctica integrarás las técnicas de funciones reutilizables aprendidas en la Lección 7.1 para construir un pipeline de automatización completo con notificaciones. Refactorizarás el script monolítico de generación de reportes del Lab 06 en módulos independientes, implementarás envío de correo electrónico con adjuntos vía SMTP/TLS, notificaciones webhook a Discord/Slack, lógica de reintentos con backoff exponencial, y programación periódica con `schedule`. Al finalizar, tendrás un sistema modular ejecutable de forma desatendida.

## Objetivos de aprendizaje

- [ ] Refactorizar un script monolítico en módulos reutilizables siguiendo principios DRY y SRP
- [ ] Configurar envío automático de correos con adjuntos usando `smtplib` con autenticación STARTTLS (puerto 587)
- [ ] Implementar notificaciones webhook HTTP con mensajes JSON de resumen de KPIs
- [ ] Diseñar un decorador de reintentos con backoff exponencial (máximo 3 intentos)
- [ ] Programar la ejecución periódica del pipeline completo usando `schedule 1.2.2`

## Prerrequisitos

### Conocimientos previos

- Lab 06 completado con archivos `ventas_limpias.csv`, `reporte_ventas.xlsx` y `reporte_ventas.html` generados
- Comprensión de funciones, módulos, decoradores y manejo de excepciones en Python
- Familiaridad con el protocolo SMTP y conceptos básicos de webhooks HTTP

### Accesos requeridos

- Cuenta Gmail con **Contraseña de Aplicación** habilitada (Seguridad → Verificación en 2 pasos → Contraseñas de aplicación) o cuenta SMTP alternativa
- URL de Webhook de Discord (Configuración del servidor → Integraciones → Webhooks → Nuevo webhook) o Slack (api.slack.com/messaging/webhooks)
- Conexión a Internet activa

## Entorno del laboratorio

### Software requerido

| Herramienta | Versión |
|-------------|---------|
| Python | 3.11.9 |
| pip | 24.0+ |
| requests | 2.32.3 |
| schedule | 1.2.2 |
| python-dotenv | 1.0.1 |
| pandas | 2.2.2 |
| openpyxl | 3.1.2 |

### Configuración inicial del entorno

```bash
# Navegar al directorio del proyecto
cd ~/automation_project

# Activar entorno virtual
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate

# Instalar dependencias adicionales para este lab
pip install requests==2.32.3 schedule==1.2.2 python-dotenv==1.0.1

# Verificar instalación
python -c "import requests, schedule, dotenv; print('OK: requests', requests.__version__, '| schedule', schedule.__version__, '| dotenv', dotenv.__version__)"
```

**Salida esperada:**
```
OK: requests 2.32.3 | schedule 1.2.2 | dotenv 1.0.1
```

### Estructura de archivos objetivo

Al finalizar este lab, la estructura dentro de `~/automation_project/` será:

```
automation_project/
├── src/
│   ├── __init__.py
│   ├── data_loader.py
│   ├── data_cleaner.py
│   ├── report_generator.py
│   ├── notifier.py
│   └── scheduler_runner.py
├── data/
│   ├── input/
│   │   └── ventas_limpias.csv
│   └── output/
│       ├── reporte_ventas.xlsx
│       └── reporte_ventas.html
├── logs/
├── .env
├── .gitignore
└── requirements.txt
```

---

## Paso 1: Crear el archivo de configuración de entorno (.env)

**Objetivo:** Configurar las credenciales sensibles de forma segura usando variables de entorno, separándolas del código fuente.

### Instrucciones

1. En la raíz del proyecto `~/automation_project/`, crea el archivo `.env`:

```bash
cd ~/automation_project
touch .env
```

2. Abre `.env` en tu editor y añade las siguientes variables (reemplaza con tus credenciales reales):

```ini
# Configuración SMTP (Gmail con App Password)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=tu_correo@gmail.com
SMTP_PASSWORD=tu_contraseña_de_aplicacion
EMAIL_DESTINATARIO=destinatario@ejemplo.com

# Webhook (Discord o Slack)
WEBHOOK_URL=https://discord.com/api/webhooks/TU_ID/TU_TOKEN

# Configuración del pipeline
REPORTE_HORA=08:00
```

3. Crea o actualiza el archivo `.gitignore` para proteger las credenciales:

```bash
cat >> .gitignore << 'EOF'
# Variables de entorno sensibles
.env

# Entorno virtual
.venv/

# Archivos de caché
__pycache__/
*.pyc

# Logs temporales
logs/*.log
EOF
```

4. Verifica que `.env` está excluido:

```bash
cat .gitignore | grep ".env"
```

**Salida esperada:**
```
.env
```

### Verificación

```bash
# Confirmar que .env existe y tiene contenido
test -f .env && echo "✓ .env creado correctamente" || echo "✗ .env no encontrado"
wc -l .env
```

El archivo debe tener al menos 7 líneas (incluyendo comentarios).

---

## Paso 2: Crear el módulo data_loader.py

**Objetivo:** Extraer la lógica de carga de datos en una función reutilizable con validaciones, anotaciones de tipo y docstring profesional.

### Instrucciones

1. Asegúrate de que existe el archivo `__init__.py` en `src/`:

```bash
touch ~/automation_project/src/__init__.py
```

2. Crea el archivo `src/data_loader.py`:

```python
# src/data_loader.py
"""
Módulo de carga de datos para el pipeline de automatización.
Funciones puras para lectura y validación de archivos CSV.
"""

import pandas as pd
from pathlib import Path


def cargar_csv(ruta: str | Path, encoding: str = "utf-8") -> pd.DataFrame:
    """
    Carga un archivo CSV y retorna un DataFrame validado.

    Args:
        ruta: Ruta absoluta o relativa al archivo CSV.
        encoding: Codificación del archivo (por defecto utf-8).

    Returns:
        DataFrame con los datos cargados.

    Raises:
        FileNotFoundError: Si el archivo no existe.
        ValueError: Si el archivo está vacío o no tiene columnas válidas.
    """
    ruta = Path(ruta).resolve()

    if not ruta.exists():
        raise FileNotFoundError(f"Archivo no encontrado: {ruta}")

    df = pd.read_csv(ruta, encoding=encoding)

    if df.empty:
        raise ValueError(f"El archivo está vacío: {ruta}")

    print(f"[data_loader] ✓ Cargados {len(df)} registros desde {ruta.name}")
    return df


def validar_columnas(df: pd.DataFrame, columnas_requeridas: list[str]) -> bool:
    """
    Verifica que un DataFrame contenga las columnas esperadas.

    Args:
        df: DataFrame a validar.
        columnas_requeridas: Lista de nombres de columnas obligatorias.

    Returns:
        True si todas las columnas están presentes.

    Raises:
        KeyError: Si faltan columnas requeridas.
    """
    faltantes = set(columnas_requeridas) - set(df.columns)
    if faltantes:
        raise KeyError(f"Columnas faltantes en el DataFrame: {faltantes}")

    print(f"[data_loader] ✓ Validación de columnas exitosa: {columnas_requeridas}")
    return True
```

### Verificación

```bash
cd ~/automation_project
python -c "
from src.data_loader import cargar_csv, validar_columnas
print('✓ Módulo data_loader importado correctamente')
print(f'  Funciones: cargar_csv, validar_columnas')
"
```

**Salida esperada:**
```
✓ Módulo data_loader importado correctamente
  Funciones: cargar_csv, validar_columnas
```

---

## Paso 3: Crear los módulos data_cleaner.py y report_generator.py

**Objetivo:** Separar la lógica de limpieza de datos y generación de reportes en módulos independientes con funciones reutilizables.

### Instrucciones

1. Crea `src/data_cleaner.py`:

```python
# src/data_cleaner.py
"""
Módulo de limpieza y transformación de datos.
Aplica reglas de negocio para normalizar el dataset de ventas.
"""

import pandas as pd


def eliminar_duplicados(df: pd.DataFrame, subset: list[str] | None = None) -> pd.DataFrame:
    """
    Elimina filas duplicadas del DataFrame.

    Args:
        df: DataFrame de entrada.
        subset: Columnas a considerar para detectar duplicados.

    Returns:
        DataFrame sin duplicados.
    """
    antes = len(df)
    df_limpio = df.drop_duplicates(subset=subset).reset_index(drop=True)
    eliminados = antes - len(df_limpio)
    print(f"[data_cleaner] ✓ Eliminados {eliminados} duplicados")
    return df_limpio


def rellenar_nulos(df: pd.DataFrame, estrategia: dict[str, any] | None = None) -> pd.DataFrame:
    """
    Rellena valores nulos según la estrategia indicada.

    Args:
        df: DataFrame con posibles valores nulos.
        estrategia: Diccionario {columna: valor_relleno}. Si None, usa 0 para
                    numéricas y 'N/A' para texto.

    Returns:
        DataFrame sin valores nulos en las columnas especificadas.
    """
    df_resultado = df.copy()

    if estrategia:
        df_resultado.fillna(estrategia, inplace=True)
    else:
        for col in df_resultado.select_dtypes(include="number").columns:
            df_resultado[col].fillna(0, inplace=True)
        for col in df_resultado.select_dtypes(include="object").columns:
            df_resultado[col].fillna("N/A", inplace=True)

    nulos_restantes = df_resultado.isnull().sum().sum()
    print(f"[data_cleaner] ✓ Nulos restantes: {nulos_restantes}")
    return df_resultado
```

2. Crea `src/report_generator.py`:

```python
# src/report_generator.py
"""
Módulo de generación de reportes en múltiples formatos.
Genera archivos Excel y HTML a partir de DataFrames procesados.
"""

import pandas as pd
from pathlib import Path
from datetime import datetime


def generar_reporte_excel(
    df: pd.DataFrame,
    ruta_salida: str | Path,
    nombre_hoja: str = "Reporte"
) -> Path:
    """
    Genera un archivo Excel (.xlsx) a partir de un DataFrame.

    Args:
        df: DataFrame con los datos del reporte.
        ruta_salida: Ruta donde guardar el archivo Excel.
        nombre_hoja: Nombre de la hoja de cálculo.

    Returns:
        Path al archivo generado.
    """
    ruta_salida = Path(ruta_salida).resolve()
    ruta_salida.parent.mkdir(parents=True, exist_ok=True)

    df.to_excel(ruta_salida, index=False, sheet_name=nombre_hoja)
    print(f"[report_generator] ✓ Excel generado: {ruta_salida.name} ({len(df)} filas)")
    return ruta_salida


def calcular_kpis(df: pd.DataFrame, columna_monto: str = "monto") -> dict:
    """
    Calcula KPIs de resumen a partir de una columna numérica.

    Args:
        df: DataFrame con datos de ventas.
        columna_monto: Nombre de la columna con valores monetarios.

    Returns:
        Diccionario con KPIs: total, promedio, maximo, registros, fecha_reporte.
    """
    kpis = {
        "total": round(float(df[columna_monto].sum()), 2),
        "promedio": round(float(df[columna_monto].mean()), 2),
        "maximo": round(float(df[columna_monto].max()), 2),
        "registros": len(df),
        "fecha_reporte": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    }
    print(f"[report_generator] ✓ KPIs calculados: total=${kpis['total']:,.2f}")
    return kpis
```

### Verificación

```bash
python -c "
from src.data_cleaner import eliminar_duplicados, rellenar_nulos
from src.report_generator import generar_reporte_excel, calcular_kpis
print('✓ Módulos data_cleaner y report_generator importados correctamente')
"
```

**Salida esperada:**
```
✓ Módulos data_cleaner y report_generator importados correctamente
```

---

## Paso 4: Crear el módulo notifier.py con envío de correo y webhook

**Objetivo:** Implementar las funciones `send_email()` y `send_webhook()` con reintentos automáticos, siguiendo los principios de funciones reutilizables de la Lección 7.1.

### Instrucciones

1. Crea el archivo `src/notifier.py`:

```python
# src/notifier.py
"""
Módulo de notificaciones para el pipeline de automatización.
Implementa envío de correo SMTP con adjuntos y notificaciones webhook.
Incluye decorador de reintentos con backoff exponencial.
"""

import smtplib
import time
import functools
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders
from pathlib import Path

import requests
from dotenv import load_dotenv
import os

# Cargar variables de entorno
load_dotenv()


# ─── DECORADOR DE REINTENTOS ───────────────────────────────────────────────

def retry(max_intentos: int = 3, delay_base: float = 2.0):
    """
    Decorador que reintenta una función con backoff exponencial.

    Args:
        max_intentos: Número máximo de intentos (default 3).
        delay_base: Tiempo base en segundos para el backoff exponencial.

    El delay entre intentos es: delay_base * (2 ** intento)
    Ejemplo con delay_base=2: intento 1 → 2s, intento 2 → 4s, intento 3 → 8s
    """
    def decorador(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            ultimo_error = None
            for intento in range(1, max_intentos + 1):
                try:
                    resultado = func(*args, **kwargs)
                    if intento > 1:
                        print(f"[retry] ✓ {func.__name__} exitoso en intento {intento}")
                    return resultado
                except Exception as e:
                    ultimo_error = e
                    if intento < max_intentos:
                        espera = delay_base * (2 ** (intento - 1))
                        print(
                            f"[retry] ⚠ {func.__name__} falló (intento {intento}/{max_intentos}): {e}"
                        )
                        print(f"[retry]   Reintentando en {espera:.1f}s...")
                        time.sleep(espera)
                    else:
                        print(
                            f"[retry] ✗ {func.__name__} falló después de {max_intentos} intentos"
                        )
            raise ultimo_error
        return wrapper
    return decorador


# ─── ENVÍO DE CORREO ELECTRÓNICO ──────────────────────────────────────────

@retry(max_intentos=3, delay_base=2.0)
def send_email(
    destinatario: str,
    asunto: str,
    cuerpo: str,
    adjunto_path: str | Path | None = None,
    smtp_host: str | None = None,
    smtp_port: int | None = None,
    smtp_user: str | None = None,
    smtp_password: str | None = None,
) -> bool:
    """
    Envía un correo electrónico con autenticación SMTP sobre TLS.

    Args:
        destinatario: Dirección de correo del receptor.
        asunto: Asunto del correo.
        cuerpo: Cuerpo del mensaje en texto plano.
        adjunto_path: Ruta opcional a un archivo para adjuntar.
        smtp_host: Servidor SMTP (default desde .env: SMTP_HOST).
        smtp_port: Puerto SMTP (default desde .env: SMTP_PORT).
        smtp_user: Usuario SMTP (default desde .env: SMTP_USER).
        smtp_password: Contraseña SMTP (default desde .env: SMTP_PASSWORD).

    Returns:
        True si el correo se envió exitosamente.

    Raises:
        smtplib.SMTPException: Si falla la conexión o el envío.
        FileNotFoundError: Si el archivo adjunto no existe.
    """
    # Cargar configuración desde .env si no se proporciona
    host = smtp_host or os.getenv("SMTP_HOST", "smtp.gmail.com")
    port = smtp_port or int(os.getenv("SMTP_PORT", "587"))
    user = smtp_user or os.getenv("SMTP_USER")
    password = smtp_password or os.getenv("SMTP_PASSWORD")

    if not user or not password:
        raise ValueError("SMTP_USER y SMTP_PASSWORD son requeridos (configurar en .env)")

    # Construir mensaje
    msg = MIMEMultipart()
    msg["From"] = user
    msg["To"] = destinatario
    msg["Subject"] = asunto
    msg.attach(MIMEText(cuerpo, "plain", "utf-8"))

    # Adjuntar archivo si se proporcionó
    if adjunto_path:
        adjunto_path = Path(adjunto_path).resolve()
        if not adjunto_path.exists():
            raise FileNotFoundError(f"Archivo adjunto no encontrado: {adjunto_path}")

        with open(adjunto_path, "rb") as f:
            parte = MIMEBase("application", "octet-stream")
            parte.set_payload(f.read())
            encoders.encode_base64(parte)
            parte.add_header(
                "Content-Disposition",
                f"attachment; filename={adjunto_path.name}"
            )
            msg.attach(parte)

    # Enviar correo
    with smtplib.SMTP(host, port) as servidor:
        servidor.ehlo()
        servidor.starttls()
        servidor.ehlo()
        servidor.login(user, password)
        servidor.sendmail(user, destinatario, msg.as_string())

    print(f"[notifier] ✓ Correo enviado a {destinatario}: '{asunto}'")
    return True


# ─── NOTIFICACIÓN WEBHOOK ─────────────────────────────────────────────────

@retry(max_intentos=3, delay_base=2.0)
def send_webhook(
    mensaje: str,
    kpis: dict | None = None,
    webhook_url: str | None = None,
) -> bool:
    """
    Envía una notificación a un canal de Discord o Slack vía webhook.

    Args:
        mensaje: Texto principal de la notificación.
        kpis: Diccionario opcional con KPIs para incluir en el mensaje.
        webhook_url: URL del webhook (default desde .env: WEBHOOK_URL).

    Returns:
        True si la notificación se envió exitosamente.

    Raises:
        ValueError: Si no se proporciona webhook_url.
        requests.HTTPError: Si el servidor responde con error HTTP.
    """
    url = webhook_url or os.getenv("WEBHOOK_URL")

    if not url:
        raise ValueError("WEBHOOK_URL es requerida (configurar en .env)")

    # Construir el cuerpo del mensaje
    contenido = mensaje
    if kpis:
        contenido += "\n```"
        contenido += f"\n📊 Resumen de KPIs:"
        contenido += f"\n   Total ventas:   ${kpis.get('total', 0):,.2f}"
        contenido += f"\n   Promedio:       ${kpis.get('promedio', 0):,.2f}"
        contenido += f"\n   Máximo:         ${kpis.get('maximo', 0):,.2f}"
        contenido += f"\n   Registros:      {kpis.get('registros', 0)}"
        contenido += f"\n   Fecha reporte:  {kpis.get('fecha_reporte', 'N/A')}"
        contenido += "\n```"

    # Determinar formato según la URL (Discord vs Slack)
    if "discord" in url.lower():
        payload = {"content": contenido}
    else:
        # Formato Slack
        payload = {"text": contenido}

    response = requests.post(url, json=payload, timeout=10)
    response.raise_for_status()

    print(f"[notifier] ✓ Webhook enviado exitosamente (status: {response.status_code})")
    return True
```

### Verificación

```bash
python -c "
from src.notifier import retry, send_email, send_webhook
print('✓ Módulo notifier importado correctamente')
print('  Funciones disponibles: retry, send_email, send_webhook')

# Verificar que el decorador funciona
@retry(max_intentos=2, delay_base=0.1)
def funcion_test():
    return 'ok'

assert funcion_test() == 'ok'
print('✓ Decorador retry funciona correctamente')
"
```

**Salida esperada:**
```
✓ Módulo notifier importado correctamente
  Funciones disponibles: retry, send_email, send_webhook
✓ Decorador retry funciona correctamente
```

---

## Paso 5: Crear el módulo scheduler_runner.py

**Objetivo:** Implementar la programación periódica del pipeline completo usando `schedule 1.2.2`, integrando todos los módulos anteriores en un flujo cohesivo.

### Instrucciones

1. Crea el archivo `src/scheduler_runner.py`:

```python
# src/scheduler_runner.py
"""
Orquestador del pipeline de automatización con programación periódica.
Integra todos los módulos src/ en un flujo de ejecución programable.
"""

import schedule
import time
import sys
from pathlib import Path
from datetime import datetime

# Asegurar que el directorio raíz está en el path
sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from src.data_loader import cargar_csv, validar_columnas
from src.data_cleaner import eliminar_duplicados, rellenar_nulos
from src.report_generator import generar_reporte_excel, calcular_kpis
from src.notifier import send_email, send_webhook

# Rutas del proyecto
PROYECTO_ROOT = Path(__file__).resolve().parent.parent
RUTA_CSV = PROYECTO_ROOT / "data" / "input" / "ventas_limpias.csv"
RUTA_EXCEL = PROYECTO_ROOT / "data" / "output" / "reporte_ventas.xlsx"


def ejecutar_pipeline(tipo: str = "diario") -> bool:
    """
    Ejecuta el pipeline completo de generación y notificación de reportes.

    Args:
        tipo: Tipo de ejecución ('diario' o 'semanal').

    Returns:
        True si el pipeline se completó sin errores.
    """
    print(f"\n{'='*60}")
    print(f"[pipeline] Iniciando ejecución {tipo}: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'='*60}\n")

    try:
        # Paso 1: Cargar datos
        print("── Paso 1: Carga de datos ──")
        df = cargar_csv(RUTA_CSV)
        validar_columnas(df, ["producto", "monto", "fecha"])

        # Paso 2: Limpiar datos
        print("\n── Paso 2: Limpieza de datos ──")
        df = eliminar_duplicados(df)
        df = rellenar_nulos(df)

        # Paso 3: Generar reporte
        print("\n── Paso 3: Generación de reporte ──")
        ruta_reporte = generar_reporte_excel(df, RUTA_EXCEL)
        kpis = calcular_kpis(df, columna_monto="monto")

        # Paso 4: Enviar notificaciones
        print("\n── Paso 4: Notificaciones ──")

        # Enviar correo con reporte adjunto
        asunto = f"[Reporte {tipo.upper()}] Ventas - {kpis['fecha_reporte']}"
        cuerpo = (
            f"Reporte de ventas {tipo} generado automáticamente.\n\n"
            f"Resumen:\n"
            f"  - Total ventas: ${kpis['total']:,.2f}\n"
            f"  - Promedio: ${kpis['promedio']:,.2f}\n"
            f"  - Registros procesados: {kpis['registros']}\n\n"
            f"Se adjunta el archivo Excel con el detalle completo."
        )

        import os
        destinatario = os.getenv("EMAIL_DESTINATARIO", os.getenv("SMTP_USER"))
        send_email(
            destinatario=destinatario,
            asunto=asunto,
            cuerpo=cuerpo,
            adjunto_path=ruta_reporte,
        )

        # Enviar notificación webhook
        send_webhook(
            mensaje=f"✅ Pipeline {tipo} completado exitosamente",
            kpis=kpis,
        )

        print(f"\n{'='*60}")
        print(f"[pipeline] ✓ Pipeline {tipo} completado exitosamente")
        print(f"{'='*60}\n")
        return True

    except Exception as e:
        print(f"\n[pipeline] ✗ ERROR en pipeline {tipo}: {e}")

        # Intentar notificar el error por webhook
        try:
            send_webhook(mensaje=f"❌ ERROR en pipeline {tipo}: {str(e)[:200]}")
        except Exception:
            print("[pipeline] ⚠ No se pudo enviar notificación de error")

        return False


def ejecutar_pipeline_diario():
    """Wrapper para ejecución diaria."""
    ejecutar_pipeline(tipo="diario")


def ejecutar_pipeline_semanal():
    """Wrapper para ejecución semanal."""
    ejecutar_pipeline(tipo="semanal")


def iniciar_scheduler():
    """
    Configura y ejecuta el scheduler con las tareas programadas.

    Tareas:
        - Diaria: Todos los días a las 08:00
        - Semanal: Cada lunes a las 08:00
    """
    import os
    from dotenv import load_dotenv
    load_dotenv()

    hora = os.getenv("REPORTE_HORA", "08:00")

    # Programar tarea diaria
    schedule.every().day.at(hora).do(ejecutar_pipeline_diario)
    print(f"[scheduler] ✓ Tarea diaria programada a las {hora}")

    # Programar tarea semanal (lunes)
    schedule.every().monday.at(hora).do(ejecutar_pipeline_semanal)
    print(f"[scheduler] ✓ Tarea semanal programada: lunes a las {hora}")

    print(f"[scheduler] Scheduler activo. Presiona Ctrl+C para detener.\n")
    print(f"[scheduler] Próximas ejecuciones:")
    for job in schedule.get_jobs():
        print(f"  → {job}")

    # Bucle principal
    try:
        while True:
            schedule.run_pending()
            time.sleep(30)  # Verificar cada 30 segundos
    except KeyboardInterrupt:
        print("\n[scheduler] Detenido por el usuario.")


if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(description="Pipeline de automatización con alertas")
    parser.add_argument(
        "--modo",
        choices=["manual", "scheduler"],
        default="manual",
        help="Modo de ejecución: 'manual' ejecuta una vez, 'scheduler' programa ejecución periódica",
    )
    parser.add_argument(
        "--tipo",
        choices=["diario", "semanal"],
        default="diario",
        help="Tipo de reporte (solo para modo manual)",
    )
    args = parser.parse_args()

    if args.modo == "manual":
        exito = ejecutar_pipeline(tipo=args.tipo)
        sys.exit(0 if exito else 1)
    else:
        iniciar_scheduler()
```

### Verificación

```bash
python -c "
from src.scheduler_runner import ejecutar_pipeline, iniciar_scheduler
print('✓ Módulo scheduler_runner importado correctamente')
print('  Funciones: ejecutar_pipeline, iniciar_scheduler')
"
```

**Salida esperada:**
```
✓ Módulo scheduler_runner importado correctamente
  Funciones: ejecutar_pipeline, iniciar_scheduler
```

---

## Paso 6: Preparar datos de prueba y ejecutar el pipeline manualmente

**Objetivo:** Crear un dataset de prueba (si no existe del Lab 06) y ejecutar el pipeline completo en modo manual para verificar la integración de todos los módulos.

### Instrucciones

1. Verifica si existe el archivo de datos del Lab 06. Si no existe, crea uno de prueba:

```bash
# Verificar existencia
ls ~/automation_project/data/input/ventas_limpias.csv 2>/dev/null && echo "✓ Archivo existe" || echo "⚠ Creando archivo de prueba..."
```

2. Si el archivo no existe, crea uno con datos de ejemplo:

```python
# Ejecutar solo si ventas_limpias.csv no existe
# Guardar como: crear_datos_prueba.py (temporal)

import pandas as pd
from pathlib import Path
import random
from datetime import datetime, timedelta

# Crear directorio si no existe
output_dir = Path.home() / "automation_project" / "data" / "input"
output_dir.mkdir(parents=True, exist_ok=True)

ruta_csv = output_dir / "ventas_limpias.csv"

if not ruta_csv.exists():
    random.seed(42)
    productos = ["Laptop", "Monitor", "Teclado", "Mouse", "Auriculares", "Webcam", "SSD", "RAM"]
    
    datos = []
    fecha_base = datetime(2024, 1, 1)
    
    for i in range(50):
        datos.append({
            "id": i + 1,
            "producto": random.choice(productos),
            "monto": round(random.uniform(25.0, 2500.0), 2),
            "cantidad": random.randint(1, 10),
            "fecha": (fecha_base + timedelta(days=random.randint(0, 180))).strftime("%Y-%m-%d"),
            "vendedor": f"Vendedor_{random.randint(1, 8):02d}",
        })
    
    df = pd.DataFrame(datos)
    df.to_csv(ruta_csv, index=False)
    print(f"✓ Archivo creado: {ruta_csv} ({len(df)} registros)")
else:
    print(f"✓ Archivo ya existe: {ruta_csv}")
```

```bash
python crear_datos_prueba.py
```

3. Crea el directorio de salida:

```bash
mkdir -p ~/automation_project/data/output
```

4. **Ejecuta el pipeline completo en modo manual:**

```bash
cd ~/automation_project
python -m src.scheduler_runner --modo manual --tipo diario
```

### Salida esperada (ejemplo)

```
============================================================
[pipeline] Iniciando ejecución diario: 2024-06-15 10:30:45
============================================================

── Paso 1: Carga de datos ──
[data_loader] ✓ Cargados 50 registros desde ventas_limpias.csv
[data_loader] ✓ Validación de columnas exitosa: ['producto', 'monto', 'fecha']

── Paso 2: Limpieza de datos ──
[data_cleaner] ✓ Eliminados 0 duplicados
[data_cleaner] ✓ Nulos restantes: 0

── Paso 3: Generación de reporte ──
[report_generator] ✓ Excel generado: reporte_ventas.xlsx (50 filas)
[report_generator] ✓ KPIs calculados: total=$58,432.15

── Paso 4: Notificaciones ──
[notifier] ✓ Correo enviado a destinatario@ejemplo.com: '[Reporte DIARIO] Ventas...'
[notifier] ✓ Webhook enviado exitosamente (status: 204)

============================================================
[pipeline] ✓ Pipeline diario completado exitosamente
============================================================
```

> **Nota:** Si las credenciales SMTP o la URL del webhook no son válidas, verás mensajes de reintento del decorador `@retry`. Esto es comportamiento esperado — confirma que el mecanismo de reintentos funciona.

### Verificación

```bash
# Verificar que el reporte Excel fue generado
ls -la ~/automation_project/data/output/reporte_ventas.xlsx

# Verificar contenido
python -c "
import pandas as pd
df = pd.read_excel('data/output/reporte_ventas.xlsx')
print(f'✓ Reporte Excel: {len(df)} filas, {len(df.columns)} columnas')
print(f'  Columnas: {list(df.columns)}')
"
```

---

## Paso 7: Verificar el scheduler (ejecución breve)

**Objetivo:** Confirmar que el scheduler se configura correctamente y detecta las tareas programadas sin dejarlo ejecutando indefinidamente.

### Instrucciones

1. Ejecuta el scheduler con un timeout de 5 segundos para verificar la configuración:

```python
# Guardar como: test_scheduler.py (temporal)

import schedule
import sys
sys.path.insert(0, ".")

from src.scheduler_runner import ejecutar_pipeline_diario, ejecutar_pipeline_semanal
from dotenv import load_dotenv
import os

load_dotenv()
hora = os.getenv("REPORTE_HORA", "08:00")

# Configurar tareas
schedule.every().day.at(hora).do(ejecutar_pipeline_diario)
schedule.every().monday.at(hora).do(ejecutar_pipeline_semanal)

print(f"✓ Scheduler configurado correctamente")
print(f"  Hora programada: {hora}")
print(f"  Tareas registradas: {len(schedule.get_jobs())}")
print(f"\n  Detalle de tareas:")
for i, job in enumerate(schedule.get_jobs(), 1):
    print(f"    {i}. {job}")

print(f"\n✓ El scheduler está listo para ejecutarse con:")
print(f"  python -m src.scheduler_runner --modo scheduler")

# Limpiar tareas de prueba
schedule.clear()
```

```bash
cd ~/automation_project
python test_scheduler.py
```

### Salida esperada

```
✓ Scheduler configurado correctamente
  Hora programada: 08:00
  Tareas registradas: 2

  Detalle de tareas:
    1. Every 1 day at 08:00:00 do ejecutar_pipeline_diario() (last run: [never], next run: 2024-06-16 08:00:00)
    2. Every 1 week at 08:00:00 do ejecutar_pipeline_semanal() (last run: [never], next run: 2024-06-17 08:00:00)

✓ El scheduler está listo para ejecutarse con:
  python -m src.scheduler_runner --modo scheduler
```

---

## Validación y pruebas

### Prueba integral de todos los módulos

Ejecuta el siguiente script de validación que verifica la estructura completa del proyecto:

```bash
cd ~/automation_project
python -c "
import sys
from pathlib import Path

print('═══ VALIDACIÓN FINAL DEL LAB 07 ═══\n')
errores = []

# 1. Verificar módulos
modulos = [
    'src/__init__.py',
    'src/data_loader.py',
    'src/data_cleaner.py', 
    'src/report_generator.py',
    'src/notifier.py',
    'src/scheduler_runner.py',
]
print('1. Verificando módulos src/:')
for mod in modulos:
    existe = Path(mod).exists()
    estado = '✓' if existe else '✗'
    print(f'   {estado} {mod}')
    if not existe:
        errores.append(f'Falta: {mod}')

# 2. Verificar archivos de configuración
print('\n2. Verificando configuración:')
configs = ['.env', '.gitignore']
for cfg in configs:
    existe = Path(cfg).exists()
    estado = '✓' if existe else '✗'
    print(f'   {estado} {cfg}')
    if not existe:
        errores.append(f'Falta: {cfg}')

# 3. Verificar .env excluido en .gitignore
if Path('.gitignore').exists():
    contenido = Path('.gitignore').read_text()
    excluido = '.env' in contenido
    estado = '✓' if excluido else '✗'
    print(f'   {estado} .env está en .gitignore')
    if not excluido:
        errores.append('.env no está excluido en .gitignore')

# 4. Verificar imports
print('\n3. Verificando imports de módulos:')
try:
    from src.data_loader import cargar_csv, validar_columnas
    print('   ✓ data_loader: cargar_csv, validar_columnas')
except ImportError as e:
    print(f'   ✗ data_loader: {e}')
    errores.append(str(e))

try:
    from src.data_cleaner import eliminar_duplicados, rellenar_nulos
    print('   ✓ data_cleaner: eliminar_duplicados, rellenar_nulos')
except ImportError as e:
    print(f'   ✗ data_cleaner: {e}')
    errores.append(str(e))

try:
    from src.report_generator import generar_reporte_excel, calcular_kpis
    print('   ✓ report_generator: generar_reporte_excel, calcular_kpis')
except ImportError as e:
    print(f'   ✗ report_generator: {e}')
    errores.append(str(e))

try:
    from src.notifier import retry, send_email, send_webhook
    print('   ✓ notifier: retry, send_email, send_webhook')
except ImportError as e:
    print(f'   ✗ notifier: {e}')
    errores.append(str(e))

try:
    from src.scheduler_runner import ejecutar_pipeline, iniciar_scheduler
    print('   ✓ scheduler_runner: ejecutar_pipeline, iniciar_scheduler')
except ImportError as e:
    print(f'   ✗ scheduler_runner: {e}')
    errores.append(str(e))

# 5. Verificar decorador retry
print('\n4. Verificando decorador retry:')
try:
    from src.notifier import retry
    contador = {'n': 0}
    
    @retry(max_intentos=3, delay_base=0.01)
    def funcion_falla_dos_veces():
        contador['n'] += 1
        if contador['n'] < 3:
            raise ConnectionError('Simulando fallo')
        return 'éxito'
    
    resultado = funcion_falla_dos_veces()
    assert resultado == 'éxito'
    assert contador['n'] == 3
    print('   ✓ Retry con backoff exponencial funciona correctamente')
except Exception as e:
    print(f'   ✗ Error en retry: {e}')
    errores.append(f'Retry falló: {e}')

# 6. Verificar datos
print('\n5. Verificando datos:')
ruta_csv = Path('data/input/ventas_limpias.csv')
if ruta_csv.exists():
    import pandas as pd
    df = pd.read_csv(ruta_csv)
    print(f'   ✓ ventas_limpias.csv: {len(df)} registros')
else:
    print('   ✗ ventas_limpias.csv no encontrado')
    errores.append('Falta ventas_limpias.csv')

ruta_excel = Path('data/output/reporte_ventas.xlsx')
if ruta_excel.exists():
    print(f'   ✓ reporte_ventas.xlsx generado')
else:
    print('   ⚠ reporte_ventas.xlsx no generado (ejecutar pipeline manualmente)')

# Resumen
print(f'\n{'═'*40}')
if not errores:
    print('✅ VALIDACIÓN EXITOSA: Todos los componentes están correctos')
else:
    print(f'⚠ VALIDACIÓN CON {len(errores)} ERROR(ES):')
    for e in errores:
        print(f'   - {e}')

sys.exit(0 if not errores else 1)
"
```

### Prueba del decorador retry con fallo real

```bash
python -c "
from src.notifier import retry
import time

@retry(max_intentos=3, delay_base=0.5)
def operacion_inestable():
    import random
    if random.random() < 0.7:
        raise TimeoutError('Conexión agotada')
    return 'Operación exitosa'

# Ejecutar varias veces para ver reintentos
for i in range(3):
    try:
        resultado = operacion_inestable()
        print(f'  Intento global {i+1}: {resultado}')
    except TimeoutError as e:
        print(f'  Intento global {i+1}: Falló definitivamente - {e}')
"
```

---

## Solución de problemas

### Problema 1: Error de autenticación SMTP

**Síntomas:**
```
smtplib.SMTPAuthenticationError: (535, b'5.7.8 Username and Password not accepted')
```

**Causa:** Gmail requiere una "Contraseña de Aplicación" específica cuando la verificación en 2 pasos está habilitada. Una contraseña normal de la cuenta no funcionará.

**Solución:**

1. Ve a https://myaccount.google.com/security
2. Activa la **Verificación en 2 pasos** si no está activa
3. Ve a https://myaccount.google.com/apppasswords
4. Selecciona "Otro (nombre personalizado)" → escribe "automation_project"
5. Copia la contraseña de 16 caracteres generada (sin espacios)
6. Actualiza `.env`:

```ini
SMTP_PASSWORD=abcdefghijklmnop
```

7. Verifica la conexión:

```python
import smtplib, os
from dotenv import load_dotenv
load_dotenv()

with smtplib.SMTP("smtp.gmail.com", 587) as s:
    s.ehlo()
    s.starttls()
    s.login(os.getenv("SMTP_USER"), os.getenv("SMTP_PASSWORD"))
    print("✓ Autenticación SMTP exitosa")
```

---

### Problema 2: Webhook retorna error 400/401

**Síntomas:**
```
requests.exceptions.HTTPError: 400 Client Error: Bad Request
```
o
```
requests.exceptions.HTTPError: 401 Unauthorized
```

**Causa:** La URL del webhook es incorrecta, ha expirado, o el formato del payload JSON no coincide con lo esperado por la plataforma (Discord espera `{"content": "..."}`, Slack espera `{"text": "..."}`).

**Solución:**

1. Verifica que la URL del webhook esté completa en `.env`:

```bash
# Discord (ejemplo):
WEBHOOK_URL=https://discord.com/api/webhooks/1234567890/ABCdef_token_completo

# Slack (ejemplo):
WEBHOOK_URL=https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX
```

2. Prueba el webhook manualmente con `curl`:

```bash
# Para Discord:
curl -X POST -H "Content-Type: application/json" \
  -d '{"content": "Test desde terminal"}' \
  "$(grep WEBHOOK_URL .env | cut -d= -f2)"

# Para Slack:
curl -X POST -H "Content-Type: application/json" \
  -d '{"text": "Test desde terminal"}' \
  "$(grep WEBHOOK_URL .env | cut -d= -f2)"
```

3. Si el webhook expiró, genera uno nuevo desde la configuración del servidor de Discord (Configuración → Integraciones → Webhooks) o desde la app de Slack.

4. Verifica que el módulo `notifier.py` detecta correctamente la plataforma:

```python
from src.notifier import send_webhook
# Test directo
send_webhook(mensaje="🧪 Test de conectividad webhook")
```

---

## Limpieza

Elimina los archivos temporales creados durante el lab:

```bash
cd ~/automation_project

# Eliminar scripts temporales de prueba
rm -f crear_datos_prueba.py test_scheduler.py

# Opcional: limpiar caché de Python
find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null

# Verificar estado final del proyecto
echo "Estructura final:"
find src/ -name "*.py" | sort
echo ""
ls -la .env .gitignore
```

> **Importante:** NO elimines los módulos en `src/` ni el archivo `.env`. Estos serán la entrada directa del Lab 08.

---

## Resumen

En esta práctica has construido un pipeline de automatización completo y modular:

| Módulo | Responsabilidad |
|--------|----------------|
| `data_loader.py` | Carga y validación de archivos CSV |
| `data_cleaner.py` | Limpieza: duplicados y valores nulos |
| `report_generator.py` | Generación de Excel y cálculo de KPIs |
| `notifier.py` | Correo SMTP + webhook + decorador retry |
| `scheduler_runner.py` | Orquestación y programación periódica |

**Principios aplicados:**
- **DRY (Don't Repeat Yourself):** Cada función existe en un solo lugar y se importa donde se necesita
- **SRP (Single Responsibility):** Cada módulo tiene una responsabilidad claramente delimitada
- **Funciones reutilizables:** Parámetros con defaults, type hints, docstrings, manejo explícito de excepciones
- **Seguridad:** Credenciales en `.env`, excluidas de control de versiones

### Recursos adicionales

- [Documentación oficial de smtplib](https://docs.python.org/3/library/smtplib.html)
- [schedule — Documentación](https://schedule.readthedocs.io/en/stable/)
- [python-dotenv — Documentación](https://saurabh-kumar.com/python-dotenv/)
- [Discord Webhooks Guide](https://discord.com/developers/docs/resources/webhook)
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)
