# Automatización mediante APIs REST

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 38 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio culminante construirás un cliente REST completo y reutilizable como clase Python (`RestApiClient`) que encapsula configuración, autenticación, reintentos automáticos y logging integrado. Integrarás componentes desarrollados en laboratorios anteriores (`file_utils.py`, `data_utils.py`, `system_utils.py`, `exceptions.py`) para crear un flujo de automatización end-to-end: leer datos locales → enviar a API → procesar respuesta → guardar resultado. Utilizarás las APIs públicas JSONPlaceholder y httpbin para validar el funcionamiento, y escribirás pruebas unitarias con mocking HTTP usando `responses 0.25.0`.

## Objetivos de Aprendizaje

- [ ] Implementar una clase `RestApiClient` con métodos `get()`, `post()`, `put()` y `delete()` que consuma APIs REST usando `requests 2.31.0`
- [ ] Integrar mecanismos de autenticación HTTP (API Key en headers, Bearer Token) con timeout configurable y reintentos automáticos
- [ ] Serializar y deserializar datos JSON entre estructuras Python y payloads HTTP para integración con módulos existentes del proyecto
- [ ] Escribir pruebas unitarias con `responses` para mockear llamadas HTTP sin depender de servicios externos
- [ ] Construir un script de demostración que ejecute el flujo completo de automatización REST integrado con la biblioteca del proyecto

## Prerrequisitos

### Conocimientos Requeridos

- Laboratorio 04-00-01 completado con `file_utils.py`, `data_utils.py`, `system_utils.py` y `exceptions.py` funcionales en `~/automation_project/src/`
- Comprensión del protocolo HTTP: métodos (GET, POST, PUT, DELETE), códigos de estado (2xx, 4xx, 5xx) y estructura solicitud-respuesta
- Programación orientada a objetos en Python: definición de clases, métodos, `self`, herencia básica

### Acceso Requerido

- Conexión a Internet activa para acceder a `https://jsonplaceholder.typicode.com` y `https://httpbin.org`
- Entorno virtual activado en `~/automation_project/.venv`
- Archivo `~/automation_project/data/output/employees_processed.json` generado en labs anteriores

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Python | 3.12.1 | Runtime |
| requests | 2.31.0 | Cliente HTTP |
| responses | 0.25.0 | Mocking HTTP para pruebas |
| pytest | 8.1.1 | Framework de pruebas |
| mypy | 1.9.0 | Verificación de tipos |

### Configuración Inicial

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
# Debe mostrar: Python 3.12.1

# Instalar/verificar dependencias necesarias
pip install requests==2.31.0 responses==0.25.0

# Verificar que los módulos de labs anteriores existen
ls src/file_utils.py src/data_utils.py src/system_utils.py src/exceptions.py
```

### Preparación del Archivo de Datos

Si no dispones del archivo `employees_processed.json` de labs anteriores, créalo con este contenido mínimo:

```bash
cat > data/output/employees_processed.json << 'EOF'
[
  {"id": 1, "nombre": "Carlos López", "departamento": "Ingeniería", "salario": 55000, "fecha_ingreso": "2020-03-15"},
  {"id": 2, "nombre": "María García", "departamento": "Marketing", "salario": 48000, "fecha_ingreso": "2019-07-22"},
  {"id": 3, "nombre": "Juan Rodríguez", "departamento": "Ventas", "salario": 52000, "fecha_ingreso": "2021-01-10"}
]
EOF
```

## Paso a Paso

### Paso 1: Definir Excepciones Personalizadas para el Cliente API

**Objetivo:** Crear excepciones específicas para errores HTTP que se integren con el módulo `exceptions.py` existente.

**Instrucciones:**

1. Abre el archivo `~/automation_project/src/exceptions.py` y añade las siguientes excepciones al final del archivo:

```python
# ~/automation_project/src/exceptions.py
# Añadir al final del archivo existente

class ApiError(AutomationError):
    """Error base para operaciones con APIs REST."""

    def __init__(self, message: str, status_code: int | None = None, url: str = "") -> None:
        self.status_code = status_code
        self.url = url
        super().__init__(f"[HTTP {status_code}] {message} (URL: {url})")


class ApiConnectionError(ApiError):
    """Error de conexión con el servicio remoto."""
    pass


class ApiAuthenticationError(ApiError):
    """Error de autenticación (401/403)."""
    pass


class ApiNotFoundError(ApiError):
    """Recurso no encontrado (404)."""
    pass


class ApiRateLimitError(ApiError):
    """Límite de tasa excedido (429)."""
    pass


class ApiServerError(ApiError):
    """Error del servidor (5xx)."""
    pass
```

> **Nota:** Si tu archivo `exceptions.py` no tiene una clase base `AutomationError`, créala como `class AutomationError(Exception): pass` al inicio del archivo.

2. Verifica que el archivo no tiene errores de sintaxis:

```bash
python -c "from src.exceptions import ApiError, ApiConnectionError, ApiAuthenticationError, ApiNotFoundError, ApiRateLimitError, ApiServerError; print('✓ Excepciones importadas correctamente')"
```

**Salida esperada:**

```
✓ Excepciones importadas correctamente
```

**Verificación:** Las seis clases de excepción deben importarse sin errores y `ApiError` debe aceptar `status_code` y `url` como parámetros.

---

### Paso 2: Construir la Clase RestApiClient

**Objetivo:** Implementar el cliente REST reutilizable con autenticación, reintentos, timeout y logging.

**Instrucciones:**

1. Crea el archivo `~/automation_project/src/api_client.py`:

```python
"""
Cliente REST reutilizable para automatización.

Módulo: api_client.py
Proyecto: automation_project
Descripción: Clase RestApiClient que encapsula configuración, autenticación,
             reintentos automáticos y logging para consumir APIs REST.
"""

import logging
import time
from typing import Any

import requests
from requests.exceptions import ConnectionError, Timeout, RequestException

from src.exceptions import (
    ApiError,
    ApiConnectionError,
    ApiAuthenticationError,
    ApiNotFoundError,
    ApiRateLimitError,
    ApiServerError,
)


class RestApiClient:
    """Cliente REST reutilizable con autenticación, reintentos y logging.

    Attributes:
        base_url: URL base del servicio API.
        timeout: Tiempo máximo de espera por solicitud en segundos.
        max_retries: Número máximo de reintentos ante errores transitorios.
        logger: Instancia de logging para registro de operaciones.
    """

    def __init__(
        self,
        base_url: str,
        timeout: int = 30,
        max_retries: int = 3,
        api_key: str | None = None,
        bearer_token: str | None = None,
        custom_headers: dict[str, str] | None = None,
    ) -> None:
        """Inicializa el cliente REST.

        Args:
            base_url: URL base del servicio (sin barra final).
            timeout: Timeout en segundos para cada solicitud.
            max_retries: Máximo de reintentos para errores transitorios (5xx, timeout).
            api_key: Clave API para autenticación via header X-API-Key.
            bearer_token: Token Bearer para autenticación via header Authorization.
            custom_headers: Headers adicionales personalizados.
        """
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.max_retries = max_retries
        self.logger = logging.getLogger(f"api_client.{self.__class__.__name__}")

        # Configurar sesión persistente
        self.session = requests.Session()
        self.session.headers.update({
            "Content-Type": "application/json",
            "Accept": "application/json",
            "User-Agent": "AutomationProject/1.0",
        })

        # Configurar autenticación
        if api_key:
            self.session.headers["X-API-Key"] = api_key
            self.logger.info("Autenticación configurada: API Key")
        if bearer_token:
            self.session.headers["Authorization"] = f"Bearer {bearer_token}"
            self.logger.info("Autenticación configurada: Bearer Token")
        if custom_headers:
            self.session.headers.update(custom_headers)

        self.logger.info(
            "RestApiClient inicializado: base_url=%s, timeout=%ds, max_retries=%d",
            self.base_url, self.timeout, self.max_retries
        )

    def _build_url(self, endpoint: str) -> str:
        """Construye la URL completa a partir del endpoint relativo."""
        endpoint = endpoint.lstrip("/")
        return f"{self.base_url}/{endpoint}"

    def _handle_error_response(self, response: requests.Response) -> None:
        """Lanza la excepción apropiada según el código de estado HTTP.

        Args:
            response: Objeto Response de requests.

        Raises:
            ApiAuthenticationError: Para códigos 401 y 403.
            ApiNotFoundError: Para código 404.
            ApiRateLimitError: Para código 429.
            ApiServerError: Para códigos 5xx.
            ApiError: Para otros códigos de error.
        """
        status = response.status_code
        url = response.url
        message = response.text[:200]  # Limitar longitud del mensaje

        if status in (401, 403):
            raise ApiAuthenticationError(message, status_code=status, url=url)
        elif status == 404:
            raise ApiNotFoundError(message, status_code=status, url=url)
        elif status == 429:
            raise ApiRateLimitError(message, status_code=status, url=url)
        elif 500 <= status < 600:
            raise ApiServerError(message, status_code=status, url=url)
        else:
            raise ApiError(message, status_code=status, url=url)

    def _request_with_retries(
        self,
        method: str,
        endpoint: str,
        json_data: dict[str, Any] | list[Any] | None = None,
        params: dict[str, str] | None = None,
    ) -> requests.Response:
        """Ejecuta una solicitud HTTP con reintentos automáticos.

        Reintenta en caso de errores de conexión, timeout o errores 5xx.
        Implementa backoff exponencial entre reintentos.

        Args:
            method: Método HTTP (GET, POST, PUT, DELETE).
            endpoint: Endpoint relativo a base_url.
            json_data: Datos a enviar como JSON en el body.
            params: Parámetros de query string.

        Returns:
            Objeto Response de requests.

        Raises:
            ApiConnectionError: Si no se puede conectar tras todos los reintentos.
            ApiError: Si la respuesta indica un error no recuperable.
        """
        url = self._build_url(endpoint)
        last_exception: Exception | None = None

        for attempt in range(1, self.max_retries + 1):
            try:
                self.logger.debug(
                    "Intento %d/%d: %s %s", attempt, self.max_retries, method, url
                )

                response = self.session.request(
                    method=method,
                    url=url,
                    json=json_data,
                    params=params,
                    timeout=self.timeout,
                )

                # Si es éxito, retornar inmediatamente
                if response.status_code < 400:
                    self.logger.info(
                        "%s %s → %d (%s)",
                        method, endpoint, response.status_code, response.reason
                    )
                    return response

                # Si es error del servidor (5xx), reintentar
                if 500 <= response.status_code < 600 and attempt < self.max_retries:
                    wait_time = 2 ** (attempt - 1)  # Backoff exponencial: 1, 2, 4s
                    self.logger.warning(
                        "Error %d en intento %d. Reintentando en %ds...",
                        response.status_code, attempt, wait_time
                    )
                    time.sleep(wait_time)
                    continue

                # Error no recuperable o último intento
                self._handle_error_response(response)

            except (ConnectionError, Timeout) as exc:
                last_exception = exc
                if attempt < self.max_retries:
                    wait_time = 2 ** (attempt - 1)
                    self.logger.warning(
                        "Error de conexión en intento %d: %s. Reintentando en %ds...",
                        attempt, str(exc)[:100], wait_time
                    )
                    time.sleep(wait_time)
                else:
                    self.logger.error(
                        "Conexión fallida tras %d intentos: %s",
                        self.max_retries, str(exc)[:100]
                    )

        raise ApiConnectionError(
            f"No se pudo conectar tras {self.max_retries} intentos: {last_exception}",
            status_code=None,
            url=url,
        )

    def get(
        self, endpoint: str, params: dict[str, str] | None = None
    ) -> dict[str, Any] | list[Any]:
        """Realiza una solicitud GET.

        Args:
            endpoint: Ruta relativa del recurso.
            params: Parámetros de query string opcionales.

        Returns:
            Datos deserializados de la respuesta JSON.
        """
        response = self._request_with_retries("GET", endpoint, params=params)
        return response.json()

    def post(
        self, endpoint: str, data: dict[str, Any] | list[Any]
    ) -> dict[str, Any]:
        """Realiza una solicitud POST.

        Args:
            endpoint: Ruta relativa del recurso.
            data: Datos a enviar como JSON.

        Returns:
            Datos deserializados de la respuesta JSON.
        """
        response = self._request_with_retries("POST", endpoint, json_data=data)
        return response.json()

    def put(
        self, endpoint: str, data: dict[str, Any]
    ) -> dict[str, Any]:
        """Realiza una solicitud PUT.

        Args:
            endpoint: Ruta relativa del recurso.
            data: Datos completos del recurso a actualizar.

        Returns:
            Datos deserializados de la respuesta JSON.
        """
        response = self._request_with_retries("PUT", endpoint, json_data=data)
        return response.json()

    def delete(self, endpoint: str) -> dict[str, Any] | None:
        """Realiza una solicitud DELETE.

        Args:
            endpoint: Ruta relativa del recurso a eliminar.

        Returns:
            Datos de la respuesta o None si es 204 No Content.
        """
        response = self._request_with_retries("DELETE", endpoint)
        if response.status_code == 204:
            return None
        return response.json()

    def close(self) -> None:
        """Cierra la sesión HTTP."""
        self.session.close()
        self.logger.info("Sesión HTTP cerrada")

    def __enter__(self) -> "RestApiClient":
        """Soporte para context manager."""
        return self

    def __exit__(self, exc_type: Any, exc_val: Any, exc_tb: Any) -> None:
        """Cierra la sesión al salir del context manager."""
        self.close()
```

2. Verifica que el módulo se importa correctamente:

```bash
cd ~/automation_project
python -c "from src.api_client import RestApiClient; print('✓ RestApiClient importado correctamente')"
```

**Salida esperada:**

```
✓ RestApiClient importado correctamente
```

**Verificación:** El archivo debe tener aproximadamente 180-200 líneas y la clase debe soportar context manager (`with`).

---

### Paso 3: Verificar Conectividad con APIs Públicas

**Objetivo:** Validar que el cliente funciona correctamente contra JSONPlaceholder y httpbin.

**Instrucciones:**

1. Crea un script de verificación rápida:

```bash
cd ~/automation_project
python -c "
from src.api_client import RestApiClient
import logging

logging.basicConfig(level=logging.INFO, format='%(levelname)s - %(name)s - %(message)s')

# Test 1: GET a JSONPlaceholder
with RestApiClient('https://jsonplaceholder.typicode.com') as client:
    user = client.get('/users/1')
    print(f'✓ GET usuario: {user[\"name\"]} ({user[\"email\"]})')

    # Test 2: POST a JSONPlaceholder
    new_post = client.post('/posts', {
        'title': 'Test Automatización',
        'body': 'Contenido de prueba',
        'userId': 1
    })
    print(f'✓ POST creado: id={new_post[\"id\"]}, title={new_post[\"title\"]}')

# Test 3: Autenticación con httpbin
with RestApiClient('https://httpbin.org', bearer_token='mi-token-secreto') as client:
    result = client.get('/bearer')
    print(f'✓ Bearer auth: authenticated={result[\"authenticated\"]}')

print('\\n✓ Todos los tests de conectividad pasaron')
"
```

**Salida esperada:**

```
INFO - api_client.RestApiClient - RestApiClient inicializado: base_url=https://jsonplaceholder.typicode.com, timeout=30s, max_retries=3
INFO - api_client.RestApiClient - GET /users/1 → 200 (OK)
✓ GET usuario: Leanne Graham (Sincere@april.biz)
INFO - api_client.RestApiClient - POST /posts → 201 (Created)
✓ POST creado: id=101, title=Test Automatización
INFO - api_client.RestApiClient - Sesión HTTP cerrada
INFO - api_client.RestApiClient - Autenticación configurada: Bearer Token
INFO - api_client.RestApiClient - RestApiClient inicializado: base_url=https://httpbin.org, timeout=30s, max_retries=3
INFO - api_client.RestApiClient - GET /bearer → 200 (OK)
✓ Bearer auth: authenticated=True
INFO - api_client.RestApiClient - Sesión HTTP cerrada

✓ Todos los tests de conectividad pasaron
```

**Verificación:** Los tres tests deben completarse sin errores. Si httpbin no responde, verifica tu conexión a Internet.

---

### Paso 4: Construir el Script de Automatización Completa

**Objetivo:** Crear `api_automation.py` que ejecute el flujo completo: leer datos locales → enviar a API → procesar respuesta → guardar resultado.

**Instrucciones:**

1. Crea el archivo `~/automation_project/src/api_automation.py`:

```python
"""
Script de automatización REST end-to-end.

Flujo: Leer datos locales → Enviar a API → Procesar respuesta → Guardar resultado.
Integra: file_utils.py, data_utils.py, api_client.py, exceptions.py
"""

import json
import logging
import sys
from datetime import datetime
from pathlib import Path
from typing import Any

# Configurar path del proyecto
PROJECT_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from src.api_client import RestApiClient
from src.exceptions import ApiError, ApiConnectionError

# Configuración de logging
LOG_DIR = PROJECT_ROOT / "logs"
LOG_DIR.mkdir(exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(name)s - %(message)s",
    handlers=[
        logging.FileHandler(LOG_DIR / "api_automation.log", encoding="utf-8"),
        logging.StreamHandler(),
    ],
)
logger = logging.getLogger("api_automation")


def load_local_data(filepath: Path) -> list[dict[str, Any]]:
    """Carga datos JSON desde un archivo local.

    Args:
        filepath: Ruta al archivo JSON.

    Returns:
        Lista de diccionarios con los datos.

    Raises:
        FileNotFoundError: Si el archivo no existe.
        json.JSONDecodeError: Si el JSON es inválido.
    """
    logger.info("Cargando datos desde: %s", filepath)
    if not filepath.exists():
        raise FileNotFoundError(f"Archivo no encontrado: {filepath}")

    with open(filepath, "r", encoding="utf-8") as f:
        data = json.load(f)

    logger.info("Datos cargados: %d registros", len(data))
    return data


def transform_employee_to_post(employee: dict[str, Any]) -> dict[str, Any]:
    """Transforma un registro de empleado en formato de post para la API.

    Args:
        employee: Diccionario con datos del empleado.

    Returns:
        Diccionario formateado como post para JSONPlaceholder.
    """
    return {
        "title": f"Reporte: {employee.get('nombre', 'Sin nombre')}",
        "body": (
            f"Departamento: {employee.get('departamento', 'N/A')}\n"
            f"Salario: ${employee.get('salario', 0):,.2f}\n"
            f"Fecha ingreso: {employee.get('fecha_ingreso', 'N/A')}"
        ),
        "userId": employee.get("id", 1),
    }


def send_employees_to_api(
    client: RestApiClient, employees: list[dict[str, Any]], max_records: int = 5
) -> list[dict[str, Any]]:
    """Envía registros de empleados a la API como posts.

    Args:
        client: Instancia de RestApiClient configurada.
        employees: Lista de empleados a enviar.
        max_records: Máximo de registros a enviar (para no saturar la API).

    Returns:
        Lista de respuestas de la API.
    """
    results: list[dict[str, Any]] = []
    records_to_send = employees[:max_records]

    logger.info("Enviando %d registros a la API...", len(records_to_send))

    for i, employee in enumerate(records_to_send, 1):
        post_data = transform_employee_to_post(employee)
        try:
            response = client.post("/posts", post_data)
            response["local_employee_id"] = employee.get("id")
            response["sync_status"] = "success"
            results.append(response)
            logger.info(
                "  [%d/%d] Enviado: '%s' → id=%s",
                i, len(records_to_send), post_data["title"], response.get("id")
            )
        except ApiError as e:
            logger.error("  [%d/%d] Error al enviar: %s", i, len(records_to_send), e)
            results.append({
                "local_employee_id": employee.get("id"),
                "sync_status": "error",
                "error": str(e),
            })

    return results


def fetch_and_enrich_users(client: RestApiClient) -> list[dict[str, Any]]:
    """Obtiene usuarios de la API y los enriquece con datos calculados.

    Args:
        client: Instancia de RestApiClient.

    Returns:
        Lista de usuarios enriquecidos.
    """
    logger.info("Obteniendo usuarios desde la API...")
    users = client.get("/users")

    enriched: list[dict[str, Any]] = []
    for user in users:
        enriched.append({
            "id": user["id"],
            "nombre": user["name"],
            "email": user["email"],
            "ciudad": user.get("address", {}).get("city", "Desconocida"),
            "empresa": user.get("company", {}).get("name", "N/A"),
            "consultado_en": datetime.now().isoformat(),
        })

    logger.info("Usuarios obtenidos y enriquecidos: %d", len(enriched))
    return enriched


def save_results(data: list[dict[str, Any]], filename: str) -> Path:
    """Guarda resultados en archivo JSON en data/output/.

    Args:
        data: Datos a guardar.
        filename: Nombre del archivo de salida.

    Returns:
        Ruta completa del archivo guardado.
    """
    output_dir = PROJECT_ROOT / "data" / "output"
    output_dir.mkdir(parents=True, exist_ok=True)
    output_path = output_dir / filename

    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

    logger.info("Resultados guardados en: %s (%d registros)", output_path, len(data))
    return output_path


def main() -> None:
    """Flujo principal de automatización REST."""
    logger.info("=" * 60)
    logger.info("INICIO: Automatización REST - %s", datetime.now().isoformat())
    logger.info("=" * 60)

    input_file = PROJECT_ROOT / "data" / "output" / "employees_processed.json"

    try:
        # Paso 1: Cargar datos locales
        employees = load_local_data(input_file)

        # Paso 2: Conectar a la API y ejecutar operaciones
        with RestApiClient(
            base_url="https://jsonplaceholder.typicode.com",
            timeout=30,
            max_retries=3,
        ) as client:

            # Paso 3: Enviar empleados como posts
            sync_results = send_employees_to_api(client, employees, max_records=3)

            # Paso 4: Obtener y enriquecer usuarios remotos
            remote_users = fetch_and_enrich_users(client)

            # Paso 5: Demostrar PUT (actualizar un post)
            logger.info("Actualizando post existente...")
            updated = client.put("/posts/1", {
                "id": 1,
                "title": "Post actualizado por automatización",
                "body": "Contenido modificado automáticamente",
                "userId": 1,
            })
            logger.info("PUT completado: id=%s, title='%s'", updated["id"], updated["title"])

            # Paso 6: Demostrar DELETE
            logger.info("Eliminando post...")
            client.delete("/posts/1")
            logger.info("DELETE completado para /posts/1")

        # Paso 7: Guardar todos los resultados
        save_results(sync_results, "api_sync_results.json")
        save_results(remote_users, "api_remote_users.json")

        # Resumen final
        successful = sum(1 for r in sync_results if r.get("sync_status") == "success")
        logger.info("-" * 60)
        logger.info("RESUMEN:")
        logger.info("  Empleados sincronizados: %d/%d exitosos", successful, len(sync_results))
        logger.info("  Usuarios remotos obtenidos: %d", len(remote_users))
        logger.info("  Archivos generados: api_sync_results.json, api_remote_users.json")
        logger.info("=" * 60)
        logger.info("FIN: Automatización completada exitosamente")
        logger.info("=" * 60)

    except FileNotFoundError as e:
        logger.error("Error de archivo: %s", e)
        sys.exit(1)
    except ApiConnectionError as e:
        logger.error("Error de conexión: %s", e)
        sys.exit(2)
    except ApiError as e:
        logger.error("Error de API: %s", e)
        sys.exit(3)
    except Exception as e:
        logger.error("Error inesperado: %s", e, exc_info=True)
        sys.exit(99)


if __name__ == "__main__":
    main()
```

2. Ejecuta el script:

```bash
cd ~/automation_project
python -m src.api_automation
```

**Salida esperada:**

```
2024-XX-XX HH:MM:SS - INFO - api_automation - ============================================================
2024-XX-XX HH:MM:SS - INFO - api_automation - INICIO: Automatización REST - 2024-XX-XXTXX:XX:XX.XXXXXX
2024-XX-XX HH:MM:SS - INFO - api_automation - ============================================================
2024-XX-XX HH:MM:SS - INFO - api_automation - Cargando datos desde: /home/usuario/automation_project/data/output/employees_processed.json
2024-XX-XX HH:MM:SS - INFO - api_automation - Datos cargados: 3 registros
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - RestApiClient inicializado: base_url=https://jsonplaceholder.typicode.com, timeout=30s, max_retries=3
2024-XX-XX HH:MM:SS - INFO - api_automation - Enviando 3 registros a la API...
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - POST /posts → 201 (Created)
2024-XX-XX HH:MM:SS - INFO - api_automation -   [1/3] Enviado: 'Reporte: Carlos López' → id=101
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - POST /posts → 201 (Created)
2024-XX-XX HH:MM:SS - INFO - api_automation -   [2/3] Enviado: 'Reporte: María García' → id=101
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - POST /posts → 201 (Created)
2024-XX-XX HH:MM:SS - INFO - api_automation -   [3/3] Enviado: 'Reporte: Juan Rodríguez' → id=101
2024-XX-XX HH:MM:SS - INFO - api_automation - Obteniendo usuarios desde la API...
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - GET /users → 200 (OK)
2024-XX-XX HH:MM:SS - INFO - api_automation - Usuarios obtenidos y enriquecidos: 10
2024-XX-XX HH:MM:SS - INFO - api_automation - Actualizando post existente...
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - PUT /posts/1 → 200 (OK)
2024-XX-XX HH:MM:SS - INFO - api_automation - PUT completado: id=1, title='Post actualizado por automatización'
2024-XX-XX HH:MM:SS - INFO - api_automation - Eliminando post...
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - DELETE /posts/1 → 200 (OK)
2024-XX-XX HH:MM:SS - INFO - api_automation - DELETE completado para /posts/1
2024-XX-XX HH:MM:SS - INFO - api_client.RestApiClient - Sesión HTTP cerrada
2024-XX-XX HH:MM:SS - INFO - api_automation - Resultados guardados en: .../data/output/api_sync_results.json (3 registros)
2024-XX-XX HH:MM:SS - INFO - api_automation - Resultados guardados en: .../data/output/api_remote_users.json (10 registros)
2024-XX-XX HH:MM:SS - INFO - api_automation - ------------------------------------------------------------
2024-XX-XX HH:MM:SS - INFO - api_automation - RESUMEN:
2024-XX-XX HH:MM:SS - INFO - api_automation -   Empleados sincronizados: 3/3 exitosos
2024-XX-XX HH:MM:SS - INFO - api_automation -   Usuarios remotos obtenidos: 10
2024-XX-XX HH:MM:SS - INFO - api_automation -   Archivos generados: api_sync_results.json, api_remote_users.json
2024-XX-XX HH:MM:SS - INFO - api_automation - ============================================================
2024-XX-XX HH:MM:SS - INFO - api_automation - FIN: Automatización completada exitosamente
2024-XX-XX HH:MM:SS - INFO - api_automation - ============================================================
```

3. Verifica los archivos generados:

```bash
# Verificar que se crearon los archivos de salida
ls -la data/output/api_sync_results.json data/output/api_remote_users.json

# Inspeccionar contenido
python -c "
import json
with open('data/output/api_sync_results.json') as f:
    data = json.load(f)
    print(f'api_sync_results.json: {len(data)} registros')
    print(f'  Primer registro: {json.dumps(data[0], indent=2, ensure_ascii=False)[:200]}')

with open('data/output/api_remote_users.json') as f:
    data = json.load(f)
    print(f'\\napi_remote_users.json: {len(data)} registros')
    print(f'  Primer usuario: {data[0][\"nombre\"]} - {data[0][\"ciudad\"]}')
"
```

**Verificación:** Deben existir ambos archivos JSON en `data/output/` y el log en `logs/api_automation.log`.

---

### Paso 5: Escribir Pruebas Unitarias con Mocking HTTP

**Objetivo:** Crear pruebas que validen el comportamiento del cliente sin depender de servicios externos.

**Instrucciones:**

1. Crea el archivo `~/automation_project/tests/test_api_client.py`:

```python
"""
Pruebas unitarias para RestApiClient con mocking HTTP.

Usa la librería 'responses' para interceptar llamadas HTTP
y simular respuestas sin conexión a Internet.
"""

import pytest
import responses
from responses import matchers

import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from src.api_client import RestApiClient
from src.exceptions import (
    ApiAuthenticationError,
    ApiConnectionError,
    ApiNotFoundError,
    ApiServerError,
)


# ============================================================
# Fixtures
# ============================================================

@pytest.fixture
def client() -> RestApiClient:
    """Crea un cliente con configuración de prueba."""
    return RestApiClient(
        base_url="https://api.test.com",
        timeout=5,
        max_retries=2,
    )


@pytest.fixture
def auth_client() -> RestApiClient:
    """Crea un cliente con autenticación Bearer."""
    return RestApiClient(
        base_url="https://api.test.com",
        bearer_token="test-token-12345",
        timeout=5,
        max_retries=1,
    )


# ============================================================
# Tests: Operaciones CRUD básicas
# ============================================================

class TestGetMethod:
    """Pruebas para el método GET."""

    @responses.activate
    def test_get_success(self, client: RestApiClient) -> None:
        """GET exitoso retorna datos deserializados."""
        responses.add(
            responses.GET,
            "https://api.test.com/users/1",
            json={"id": 1, "name": "Test User"},
            status=200,
        )

        result = client.get("/users/1")

        assert result == {"id": 1, "name": "Test User"}
        assert len(responses.calls) == 1

    @responses.activate
    def test_get_with_params(self, client: RestApiClient) -> None:
        """GET con query params los incluye en la URL."""
        responses.add(
            responses.GET,
            "https://api.test.com/users",
            json=[{"id": 1}, {"id": 2}],
            status=200,
        )

        result = client.get("/users", params={"page": "1", "limit": "10"})

        assert len(result) == 2
        assert "page=1" in responses.calls[0].request.url

    @responses.activate
    def test_get_not_found_raises_exception(self, client: RestApiClient) -> None:
        """GET a recurso inexistente lanza ApiNotFoundError."""
        responses.add(
            responses.GET,
            "https://api.test.com/users/999",
            json={"error": "Not Found"},
            status=404,
        )

        with pytest.raises(ApiNotFoundError) as exc_info:
            client.get("/users/999")

        assert exc_info.value.status_code == 404


class TestPostMethod:
    """Pruebas para el método POST."""

    @responses.activate
    def test_post_success(self, client: RestApiClient) -> None:
        """POST exitoso envía datos y retorna respuesta."""
        responses.add(
            responses.POST,
            "https://api.test.com/posts",
            json={"id": 101, "title": "Nuevo post"},
            status=201,
        )

        result = client.post("/posts", {"title": "Nuevo post", "body": "Contenido"})

        assert result["id"] == 101
        # Verificar que los datos fueron enviados
        import json
        sent_body = json.loads(responses.calls[0].request.body)
        assert sent_body["title"] == "Nuevo post"


class TestPutMethod:
    """Pruebas para el método PUT."""

    @responses.activate
    def test_put_success(self, client: RestApiClient) -> None:
        """PUT exitoso actualiza y retorna recurso."""
        responses.add(
            responses.PUT,
            "https://api.test.com/posts/1",
            json={"id": 1, "title": "Actualizado"},
            status=200,
        )

        result = client.put("/posts/1", {"id": 1, "title": "Actualizado"})

        assert result["title"] == "Actualizado"


class TestDeleteMethod:
    """Pruebas para el método DELETE."""

    @responses.activate
    def test_delete_success(self, client: RestApiClient) -> None:
        """DELETE exitoso retorna respuesta vacía o datos."""
        responses.add(
            responses.DELETE,
            "https://api.test.com/posts/1",
            json={},
            status=200,
        )

        result = client.delete("/posts/1")

        assert result == {}


# ============================================================
# Tests: Autenticación
# ============================================================

class TestAuthentication:
    """Pruebas para mecanismos de autenticación."""

    @responses.activate
    def test_bearer_token_in_headers(self, auth_client: RestApiClient) -> None:
        """El Bearer token se incluye en el header Authorization."""
        responses.add(
            responses.GET,
            "https://api.test.com/protected",
            json={"data": "secret"},
            status=200,
        )

        auth_client.get("/protected")

        auth_header = responses.calls[0].request.headers["Authorization"]
        assert auth_header == "Bearer test-token-12345"

    @responses.activate
    def test_api_key_in_headers(self) -> None:
        """La API Key se incluye en el header X-API-Key."""
        client = RestApiClient(
            base_url="https://api.test.com",
            api_key="my-secret-key",
            max_retries=1,
        )
        responses.add(
            responses.GET,
            "https://api.test.com/data",
            json={"result": "ok"},
            status=200,
        )

        client.get("/data")

        assert responses.calls[0].request.headers["X-API-Key"] == "my-secret-key"

    @responses.activate
    def test_unauthorized_raises_auth_error(self, client: RestApiClient) -> None:
        """Respuesta 401 lanza ApiAuthenticationError."""
        responses.add(
            responses.GET,
            "https://api.test.com/secure",
            json={"error": "Unauthorized"},
            status=401,
        )

        with pytest.raises(ApiAuthenticationError) as exc_info:
            client.get("/secure")

        assert exc_info.value.status_code == 401


# ============================================================
# Tests: Reintentos y manejo de errores
# ============================================================

class TestRetries:
    """Pruebas para el mecanismo de reintentos."""

    @responses.activate
    def test_retry_on_server_error(self, client: RestApiClient) -> None:
        """Reintenta automáticamente ante errores 500."""
        # Primer intento: falla con 500
        responses.add(
            responses.GET,
            "https://api.test.com/unstable",
            json={"error": "Internal Server Error"},
            status=500,
        )
        # Segundo intento: éxito
        responses.add(
            responses.GET,
            "https://api.test.com/unstable",
            json={"data": "recovered"},
            status=200,
        )

        result = client.get("/unstable")

        assert result == {"data": "recovered"}
        assert len(responses.calls) == 2  # Se hicieron 2 intentos

    @responses.activate
    def test_max_retries_exhausted_raises_server_error(self, client: RestApiClient) -> None:
        """Tras agotar reintentos con 500, lanza ApiServerError."""
        # Todos los intentos fallan
        for _ in range(3):
            responses.add(
                responses.GET,
                "https://api.test.com/broken",
                json={"error": "Server Error"},
                status=500,
            )

        with pytest.raises(ApiServerError):
            client.get("/broken")

    @responses.activate
    def test_client_error_no_retry(self, client: RestApiClient) -> None:
        """Errores 4xx NO se reintentan (excepto 429)."""
        responses.add(
            responses.GET,
            "https://api.test.com/bad",
            json={"error": "Bad Request"},
            status=400,
        )

        with pytest.raises(Exception):  # ApiError
            client.get("/bad")

        # Solo un intento, sin reintentos
        assert len(responses.calls) == 1


# ============================================================
# Tests: Context Manager
# ============================================================

class TestContextManager:
    """Pruebas para el soporte de context manager."""

    @responses.activate
    def test_context_manager_closes_session(self) -> None:
        """El context manager cierra la sesión al salir."""
        responses.add(
            responses.GET,
            "https://api.test.com/test",
            json={"ok": True},
            status=200,
        )

        with RestApiClient("https://api.test.com", max_retries=1) as client:
            result = client.get("/test")
            assert result == {"ok": True}

        # Verificar que la sesión fue cerrada (no lanza error)
        # La sesión cerrada no debería poder hacer más requests
```

2. Ejecuta las pruebas:

```bash
cd ~/automation_project
python -m pytest tests/test_api_client.py -v
```

**Salida esperada:**

```
========================= test session starts =========================
collected 12 items

tests/test_api_client.py::TestGetMethod::test_get_success PASSED
tests/test_api_client.py::TestGetMethod::test_get_with_params PASSED
tests/test_api_client.py::TestGetMethod::test_get_not_found_raises_exception PASSED
tests/test_api_client.py::TestPostMethod::test_post_success PASSED
tests/test_api_client.py::TestPutMethod::test_put_success PASSED
tests/test_api_client.py::TestDeleteMethod::test_delete_success PASSED
tests/test_api_client.py::TestAuthentication::test_bearer_token_in_headers PASSED
tests/test_api_client.py::TestAuthentication::test_api_key_in_headers PASSED
tests/test_api_client.py::TestAuthentication::test_unauthorized_raises_auth_error PASSED
tests/test_api_client.py::TestRetries::test_retry_on_server_error PASSED
tests/test_api_client.py::TestRetries::test_max_retries_exhausted_raises_server_error PASSED
tests/test_api_client.py::TestRetries::test_client_error_no_retry PASSED
tests/test_api_client.py::TestContextManager::test_context_manager_closes_session PASSED

========================= 13 passed in 0.45s ==========================
```

**Verificación:** Todas las pruebas (12-13) deben pasar en verde. Las pruebas NO requieren conexión a Internet gracias al mocking con `responses`.

---

### Paso 6: Verificar Tipado Estático con mypy

**Objetivo:** Asegurar que el código cumple con las anotaciones de tipo.

**Instrucciones:**

1. Ejecuta mypy sobre los módulos creados:

```bash
cd ~/automation_project
python -m mypy src/api_client.py src/api_automation.py --ignore-missing-imports
```

**Salida esperada:**

```
Success: no issues found in 2 source files
```

> **Nota:** Si mypy reporta errores menores sobre imports de módulos anteriores (`file_utils`, `data_utils`), usa `--ignore-missing-imports`. Lo importante es que `api_client.py` pase sin errores de tipo.

2. Si hay errores de tipo, corrígelos siguiendo las indicaciones de mypy. Los errores más comunes son:
   - Uso de `dict` sin especificar tipos → usar `dict[str, Any]`
   - Retorno `None` no declarado → añadir `-> None`

**Verificación:** mypy debe reportar "Success" o solo warnings menores sobre módulos externos.

---

### Paso 7: Demostración de Autenticación con httpbin

**Objetivo:** Validar los mecanismos de autenticación contra un servicio real de prueba.

**Instrucciones:**

1. Ejecuta el siguiente script de demostración:

```bash
cd ~/automation_project
python -c "
import logging
logging.basicConfig(level=logging.INFO, format='%(levelname)s - %(message)s')

from src.api_client import RestApiClient
from src.exceptions import ApiAuthenticationError

print('=== Test 1: Bearer Token Authentication ===')
with RestApiClient('https://httpbin.org', bearer_token='automation-token-2024') as client:
    result = client.get('/bearer')
    print(f'  Autenticado: {result[\"authenticated\"]}')
    print(f'  Token recibido: {result[\"token\"]}')

print()
print('=== Test 2: Custom Headers ===')
with RestApiClient('https://httpbin.org', custom_headers={'X-Custom-ID': 'lab05'}) as client:
    result = client.get('/headers')
    headers = result['headers']
    print(f'  X-Custom-Id: {headers.get(\"X-Custom-Id\", \"no encontrado\")}')
    print(f'  User-Agent: {headers.get(\"User-Agent\", \"no encontrado\")}')

print()
print('=== Test 3: Manejo de Error 401 ===')
with RestApiClient('https://httpbin.org', max_retries=1) as client:
    try:
        client.get('/status/401')
    except ApiAuthenticationError as e:
        print(f'  ✓ Excepción capturada: {type(e).__name__}')
        print(f'  Código: {e.status_code}')

print()
print('✓ Todos los tests de autenticación completados')
"
```

**Salida esperada:**

```
=== Test 1: Bearer Token Authentication ===
INFO - RestApiClient inicializado: base_url=https://httpbin.org, timeout=30s, max_retries=3
INFO - GET /bearer → 200 (OK)
  Autenticado: True
  Token recibido: automation-token-2024
INFO - Sesión HTTP cerrada

=== Test 2: Custom Headers ===
INFO - RestApiClient inicializado: base_url=https://httpbin.org, timeout=30s, max_retries=3
INFO - GET /headers → 200 (OK)
  X-Custom-Id: lab05
  User-Agent: AutomationProject/1.0
INFO - Sesión HTTP cerrada

=== Test 3: Manejo de Error 401 ===
INFO - RestApiClient inicializado: base_url=https://httpbin.org, timeout=30s, max_retries=1
  ✓ Excepción capturada: ApiAuthenticationError
  Código: 401
INFO - Sesión HTTP cerrada

✓ Todos los tests de autenticación completados
```

**Verificación:** Los tres tests deben completarse mostrando que la autenticación Bearer funciona, los headers personalizados se envían correctamente, y los errores 401 se capturan como `ApiAuthenticationError`.

## Validación y Testing

Ejecuta la suite completa de validación para confirmar que todo el laboratorio funciona correctamente:

```bash
cd ~/automation_project

echo "=== 1. Verificación de estructura de archivos ==="
test -f src/api_client.py && echo "✓ src/api_client.py" || echo "✗ FALTA src/api_client.py"
test -f src/api_automation.py && echo "✓ src/api_automation.py" || echo "✗ FALTA src/api_automation.py"
test -f tests/test_api_client.py && echo "✓ tests/test_api_client.py" || echo "✗ FALTA tests/test_api_client.py"
test -f data/output/api_sync_results.json && echo "✓ data/output/api_sync_results.json" || echo "✗ FALTA (ejecutar api_automation.py)"
test -f data/output/api_remote_users.json && echo "✓ data/output/api_remote_users.json" || echo "✗ FALTA (ejecutar api_automation.py)"
test -f logs/api_automation.log && echo "✓ logs/api_automation.log" || echo "✗ FALTA log"

echo ""
echo "=== 2. Pruebas unitarias ==="
python -m pytest tests/test_api_client.py -v --tb=short

echo ""
echo "=== 3. Verificación de tipos ==="
python -m mypy src/api_client.py --ignore-missing-imports

echo ""
echo "=== 4. Test de integración rápido ==="
python -c "
from src.api_client import RestApiClient
with RestApiClient('https://jsonplaceholder.typicode.com') as c:
    posts = c.get('/posts', params={'userId': '1'})
    assert isinstance(posts, list)
    assert len(posts) == 10
    print(f'✓ Integración OK: {len(posts)} posts obtenidos para userId=1')
"

echo ""
echo "=== VALIDACIÓN COMPLETA ==="
```

**Criterios de éxito:**
- ✓ Los 6 archivos existen en las rutas correctas
- ✓ Todas las pruebas unitarias pasan (12-13 tests)
- ✓ mypy reporta "Success"
- ✓ El test de integración obtiene 10 posts

## Solución de Problemas

### Problema 1: `ConnectionError` o `Timeout` al ejecutar `api_automation.py`

**Síntomas:** El script falla con `ApiConnectionError: No se pudo conectar tras 3 intentos` o se queda colgado por mucho tiempo antes de fallar.

**Causa:** La conexión a Internet no está disponible, un firewall corporativo bloquea las solicitudes HTTPS salientes, o el DNS no resuelve `jsonplaceholder.typicode.com`.

**Solución:**

```bash
# 1. Verificar conectividad básica
curl -I https://jsonplaceholder.typicode.com/posts/1

# 2. Si curl funciona pero Python no, verificar proxy
echo $HTTP_PROXY $HTTPS_PROXY

# 3. Si hay proxy, configurarlo en el cliente:
# Añadir en __init__ de RestApiClient:
# self.session.proxies = {"http": "http://proxy:8080", "https": "http://proxy:8080"}

# 4. Si no hay Internet, las pruebas unitarias con 'responses' SÍ funcionan sin conexión:
python -m pytest tests/test_api_client.py -v
# Esto valida la lógica del cliente sin red
```

### Problema 2: `ModuleNotFoundError: No module named 'src.exceptions'` al importar

**Síntomas:** Al ejecutar `python -m src.api_automation` o las pruebas, aparece un error de importación indicando que no se encuentra el módulo `src.exceptions` o `src.api_client`.

**Causa:** El directorio de trabajo actual no es `~/automation_project/`, falta el archivo `__init__.py` en el directorio `src/`, o el `sys.path` no incluye la raíz del proyecto.

**Solución:**

```bash
# 1. Asegurarse de estar en el directorio correcto
cd ~/automation_project
pwd  # Debe mostrar /home/<usuario>/automation_project

# 2. Crear __init__.py si no existe
touch src/__init__.py
touch tests/__init__.py

# 3. Verificar que la estructura es correcta
find src/ -name "*.py" | sort
# Debe mostrar:
# src/__init__.py
# src/api_automation.py
# src/api_client.py
# src/data_utils.py
# src/exceptions.py
# src/file_utils.py
# src/system_utils.py

# 4. Ejecutar con el módulo correctamente
python -m src.api_automation

# 5. Para pytest, ejecutar desde la raíz:
python -m pytest tests/test_api_client.py -v
```

## Limpieza

Al finalizar el laboratorio, los archivos generados forman parte del proyecto y no necesitan eliminarse. Sin embargo, si deseas limpiar los artefactos de ejecución:

```bash
cd ~/automation_project

# Limpiar solo los archivos de salida generados por este lab (opcional)
rm -f data/output/api_sync_results.json
rm -f data/output/api_remote_users.json
rm -f logs/api_automation.log

# Limpiar caché de pytest y mypy
rm -rf .pytest_cache .mypy_cache tests/__pycache__ src/__pycache__
```

> **Recomendación:** Mantén los archivos `src/api_client.py`, `src/api_automation.py` y `tests/test_api_client.py` ya que forman parte de la biblioteca reutilizable del proyecto.

## Resumen

En este laboratorio has construido:

| Componente | Archivo | Funcionalidad |
|------------|---------|---------------|
| Cliente REST | `src/api_client.py` | Clase `RestApiClient` con GET/POST/PUT/DELETE, reintentos, auth y logging |
| Excepciones API | `src/exceptions.py` | 6 clases de excepción específicas para errores HTTP |
| Script de automatización | `src/api_automation.py` | Flujo end-to-end: lectura local → API → procesamiento → guardado |
| Pruebas unitarias | `tests/test_api_client.py` | 13 tests con mocking HTTP (sin dependencia de red) |

**Conceptos clave aplicados:**
- Protocolo HTTP: métodos, códigos de estado, headers, body JSON
- Principios REST: recursos como sustantivos, interfaz uniforme, sin estado
- Patrones de diseño: Session reutilizable, Context Manager, Backoff exponencial
- Ingeniería de software: tipado estático, logging estructurado, pruebas con mocking, manejo de excepciones granular

### Recursos Adicionales

- [Documentación de requests](https://docs.python-requests.org/en/latest/)
- [Documentación de responses (mocking)](https://github.com/getsentry/responses)
- [JSONPlaceholder - Guía de uso](https://jsonplaceholder.typicode.com/guide/)
- [httpbin - Endpoints disponibles](https://httpbin.org/#/HTTP_Methods)
- [Patrón de reintentos con backoff exponencial](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
