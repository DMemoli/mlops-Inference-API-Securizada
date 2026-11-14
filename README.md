# API de Inferencia Segura — Seguridad y Gestión de Riesgos en MLOps (Unidades 1, 2 y 3)

Proyecto académico de una API REST desarrollada con **FastAPI** e integrada con **Google Gemini**. El trabajo comenzó con una API de inferencia, incorporó controles de seguridad y *grounding* en la Unidad 2 y sumó en la Unidad 3 un proceso de **evaluación continua y CI/CD** con **GitHub Actions, MLflow, Ragas y Render**.

El objetivo es impedir que un cambio en el modelo o en sus controles llegue a despliegue si no supera una batería fija de pruebas de seguridad y calidad.

> **Alcance:** laboratorio educativo. El archivo de alineación del Mundial 2026 describe un escenario ficticio de la práctica; no representa resultados deportivos reales. El sistema no constituye una solución de producción endurecida ni implementa un RAG con búsqueda vectorial.

## 1. Funcionalidades

- **`GET /`**: comprobación de disponibilidad de la API.
- **`POST /inferencia`**: genera una respuesta con Gemini a partir de un `prompt`; permite indicar el `model` para comparar versiones.
- **Validación con Pydantic**: texto de 5 a 100 caracteres y expresión regular de caracteres permitidos; errores de validación con respuesta genérica.
- **LLM-as-a-Judge**: filtro semántico previo a la inferencia para detectar solicitudes maliciosas o intentos de manipular las instrucciones.
- **Guardrails y separación de roles**: instrucciones de sistema, contenido del usuario delimitado y filtros de seguridad del proveedor.
- **Grounding con JSON local**: respuestas basadas en `alineacion_espana_final_2026.json`; la base se carga en memoria al iniciar la API de la Unidad 3.
- **Docker sin usuario root**: ejecución con el usuario `appuser`.
- **Evaluación automatizada**: 10 casos de seguridad y 10 consultas funcionales estáticas mediante `eval.py` y `TestClient`.
- **MLflow y Ragas**: registro de parámetros y métricas y medición semántica de relevancia.
- **GitHub Actions y Render**: ejecución de las pruebas tras cambios en `main` y solicitud de despliegue solo cuando se aprueban los umbrales.

## 2. Estructura del repositorio

```text
mlops-Inference-API-Securizada/
├── .github/
│   └── workflows/
│       └── cicd.yml                       # Pipeline CI/CD
├── eval_data/
│   ├── eval_security.json                 # 10 pruebas de seguridad
│   └── eval_functional.json               # 10 consultas funcionales
├── main_2.py                              # Versión de la API (Unidad 2)
├── main_3.py                              # API con selección dinámica de modelo (Unidad 3)
├── eval.py                                # Evaluaciones, métricas y quality gates
├── alineacion_espana_final_2026.json     # Base de conocimiento del ejercicio
├── requirements.txt                      # Dependencias de la API
├── requirements-test.txt                 # Dependencias adicionales de evaluación
├── dockerfile                             # Imagen Docker de ejecución sin root
├── .gitignore
└── README.md
```

**Importante:** `eval.py` debe importar la versión de API que se pretende evaluar (`main_3.py`). Para que el despliegue sea coherente con las pruebas, la imagen de producción también debe ejecutar esa misma versión. En el `dockerfile` consultado, el comando de inicio todavía apuntaba a `main_2:app`; revisar este punto antes de desplegar.

## 3. Requisitos previos

- Python **3.11**, `pip` y Git.
- Una API key habilitada para Google Gemini, obtenida en [Google AI Studio](https://aistudio.google.com/).
- Docker Desktop (opcional para uso local; necesario si se prueba la imagen).
- Repositorio de GitHub; una cuenta de [Render](https://render.com/) para la etapa CD.

Los modelos y sus límites pueden cambiar. Comprobar disponibilidad y cuotas en [Google AI Studio — Rate Limits](https://aistudio.google.com/rate-limit).

## 4. Instalación y ejecución local

### 4.1. Clonar el repositorio

```bash
git clone https://github.com/DMemoli/mlops-Inference-API-Securizada.git
cd mlops-Inference-API-Securizada
```

### 4.2. Crear el entorno virtual

**Windows (PowerShell):**

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

**Linux / macOS:**

```bash
python3 -m venv .venv
./.venv/bin/python -m pip install -r requirements.txt
```

Para ejecutar también las pruebas de la Unidad 3, instalar en ese entorno las dependencias adicionales:

```bash
python -m pip install -r requirements-test.txt
```

En Windows se puede sustituir `python` por `\.venv\Scripts\python.exe` si el entorno no está activado.

### 4.3. Configurar la clave

Crear un archivo `.env` en la raíz (no debe subirse al repositorio):

```dotenv
GEMINI_API_KEY=TU_CLAVE_REAL
```

`main_3.py` carga el archivo mediante `python-dotenv`. Evitar definir simultáneamente `GEMINI_API_KEY` y `GOOGLE_API_KEY` con claves diferentes, porque las librerías pueden priorizarlas de distinta manera. No publicar claves en código, registros, imágenes Docker ni commits.

### 4.4. Iniciar FastAPI

**Windows:**

```powershell
.\.venv\Scripts\python.exe -m uvicorn main_3:app --host 127.0.0.1 --port 8000 --reload
```

**Linux / macOS:**

```bash
./.venv/bin/python -m uvicorn main_3:app --host 127.0.0.1 --port 8000 --reload
```

Abrir `http://localhost:8000/` para comprobar el estado o `http://localhost:8000/docs` para utilizar Swagger UI.

### 4.5. Consultar el endpoint

**POST** `http://localhost:8000/inferencia` con `Content-Type: application/json`:

```json
{
  "prompt": "¿Quién es el portero titular de España en el mundial?",
  "model": "gemini-3.5-flash-lite"
}
```

El campo `model` es opcional; el valor predeterminado en la versión utilizada para la Unidad 3 es `gemini-3.5-flash-lite`. Para la prueba de flexibilidad se utilizó `gemini-3.1-flash-lite`.

Una solicitud legítima completada correctamente devuelve **HTTP 200** con una propiedad `resultado`. Una solicitud maliciosa detectada por el juez puede recibir **HTTP 403**; un error de formato, **HTTP 400**. Las respuestas de error del proveedor deben tratarse separadamente de los rechazos de seguridad.

### 4.6. Ejecutar con Docker

El archivo del repositorio se llama `dockerfile` (en minúsculas):

```bash
docker build -f dockerfile -t mlops-inferencia-segura .
```

Para **forzar la misma versión (`main_3.py`) que evalúa CI** sin depender del `CMD` actual del Dockerfile:

```bash
docker run --rm -p 8000:8000 --env-file .env mlops-inferencia-segura \
  uvicorn main_3:app --host 0.0.0.0 --port 8000
```

Comprobar que la imagen no ejecuta procesos como root:

```bash
docker run --rm --entrypoint id mlops-inferencia-segura
```

La salida debe identificar un UID distinto de `0`. Antes de generar imágenes, agregar o verificar un `.dockerignore` que excluya, como mínimo, `.env`, `.venv`, `.git` y directorios locales de MLflow para no incorporar secretos ni archivos innecesarios.

## 5. Evaluación automatizada — Unidad 3

### 5.1. Datasets reproducibles

Los casos se mantienen en archivos JSON estáticos para comparar versiones con las mismas preguntas:

| Archivo | Contenido | Finalidad |
|---|---|---|
| `eval_data/eval_security.json` | 10 entradas: 5 ataques y 5 solicitudes legítimas | Evaluar eficacia y falsos positivos del LLM Juez |
| `eval_data/eval_functional.json` | 10 preguntas relacionadas con el JSON de alineación | Medir relevancia de las respuestas mediante Ragas |

Los textos de seguridad se diseñaron para pasar la validación de caracteres de Pydantic: el bloqueo de una inyección debe atribuirse al control semántico y no a un error de sintaxis.

### 5.2. Métricas y condiciones de aprobación

| Métrica | Condición del pipeline |
|---|---:|
| `security_recall` | **1,00** (100 % de ataques detectados) |
| `false_positive_rate` | **0,00** (sin bloqueo de casos legítimos) |
| `answer_relevancy` | **≥ 0,75** (promedio de Ragas) |

Se utiliza `TestClient` de FastAPI para probar `/inferencia` sin levantar un servidor HTTP externo. Los parámetros y resultados se registran con MLflow en el experimento `Evaluacion_Produccion_SOC`.

**Precisión conceptual:** *Answer Relevancy* estima la relación entre pregunta y respuesta; **no equivale por sí sola a una prueba de fidelidad factual o ausencia de alucinaciones**. Una ejecución que no puede completarse por un error del proveedor tampoco debe computarse como relevancia `0.0`.

### 5.3. Ejecutar las evaluaciones localmente

Con la clave configurada y las dependencias de `requirements-test.txt` instaladas, desde la raíz:

```bash
python -u eval.py
```

Ejemplo para probar la versión alternativa desde **PowerShell**:

```powershell
$env:GEMINI_MODEL="gemini-3.1-flash-lite"
$env:EVAL_PAUSA_SEGUNDOS="30"
python -u eval.py
```

En **Linux / macOS**:

```bash
GEMINI_MODEL=gemini-3.1-flash-lite EVAL_PAUSA_SEGUNDOS=30 python -u eval.py
```

Estos parámetros corresponden a la versión de `eval.py` utilizada en las ejecuciones finales de la práctica. Una vez terminada la prueba, restablecer o eliminar `GEMINI_MODEL` para no comparar accidentalmente un modelo diferente.

### 5.4. Manejo de cuotas y errores transitorios

En las primeras ejecuciones aparecieron errores **HTTP 429** (límites de uso) y **HTTP 503** (indisponibilidad temporal), especialmente con el servicio gratuito de Gemini. La prueba directa de la API key devolvió `Gemini funciona: OK`, pero eso no garantizaba capacidad para toda la batería.

Para poder completar la evaluación se introdujeron medidas de contingencia:

- **Solicitudes secuenciales** con una pausa preventiva de **30 segundos** entre consultas a la API.
- Función común de solicitud y **reintentos ante errores transitorios** del proveedor (por ejemplo, respuestas 502/503 de la API), con **hasta tres reintentos adicionales** y esperas de **al menos 60 segundos**.
- **`max_workers=1` en Ragas**, para evitar evaluaciones concurrentes, y pausa previa a la fase semántica.
- Si se agotan los reintentos, **la evaluación se declara inconclusa y se bloquea el despliegue**, en vez de atribuir al modelo una calificación artificial de cero.
- Adaptación de la métrica `AnswerRelevancy` a **`strictness=1`** para evitar el error `400 INVALID_ARGUMENT: Multiple candidates is not enabled for this model`.

**Costo de la solución:** estas pausas y reintentos **incrementan significativamente la duración del pipeline**. Son un *workaround* aceptable para el laboratorio con una cuota gratuita limitada; no deben interpretarse como una optimización de rendimiento. En un entorno productivo convendría dimensionar cuota, presupuesto, límites de concurrencia y observabilidad.

## 6. CI/CD con GitHub Actions y Render

El archivo `.github/workflows/cicd.yml` ejecuta el trabajo `evaluacion_y_despliegue` ante cambios en la rama `main`. Sus etapas principales son:

1. Obtener el código (`actions/checkout@v5`).
2. Configurar Python 3.11 (`actions/setup-python@v6`) en **Ubuntu 24.04**.
3. Instalar `requirements-test.txt`.
4. Ejecutar `python eval.py` con la clave inyectada desde GitHub Secrets.
5. Si el proceso finaliza con código `0`, invocar el **Deploy Hook** de Render. Un fallo de evaluación (`exit code 1`) impide avanzar al despliegue.

En **GitHub → Settings → Secrets and variables → Actions**, configurar:

- `GEMINI_API_KEY`: clave del proyecto de Google AI Studio.
- `RENDER_DEPLOY_HOOK`: URL privada proporcionada por Render.

En **Render → Settings**, desactivar el despliegue automático por cada push (**Auto-Deploy: No**) si el objetivo es que la liberación dependa exclusivamente de la aprobación del pipeline.

**Comprobación recomendada del webhook:** utilizar `curl --fail-with-body --show-error --silent --request POST "$RENDER_DEPLOY_HOOK"` con el secreto como variable de entorno. La opción `--fail-with-body` hace fallar el paso ante errores HTTP 4xx/5xx; `curl -X POST` sin esa opción puede terminar con código `0` aunque Render responda `Not Found`.

## 7. Evidencias del trabajo práctico (14/11/2026)

Los siguientes valores proceden de las ejecuciones exportadas de GitHub Actions del ejercicio:

| Escenario | Modelo | Recall | Falsos positivos | Relevancia | Resultado |
|---|---|---:|---:|---:|---|
| **Prueba verde** | `gemini-3.5-flash-lite` | 1,000 | 0,000 | 0,822 | Evaluación aprobada |
| **Prueba roja** (degradación deliberada del juez) | `gemini-3.5-flash-lite` | 0,000 | 0,000 | 0,798 | **Despliegue bloqueado**, como se esperaba |
| **Prueba de flexibilidad** | `gemini-3.1-flash-lite` | 1,000 | 0,000 | 0,824 | Evaluación aprobada; Render devolvió un identificador de despliegue |

En la **prueba roja**, la relevancia fue suficiente, pero el *recall* cayó a cero: el pipeline emitió `exit code 1` y no ejecutó la etapa de despliegue. Esto demuestra que la puerta de seguridad funciona independientemente de la calidad funcional.

Durante la **prueba de flexibilidad** se registraron errores transitorios **503** y **502**; los reintentos permitieron completar los 20 casos y la evaluación de Ragas. El proceso duró aproximadamente **19 minutos**, frente a unos **12 minutos** en la ejecución verde. Una parte importante de esa latencia respondió a las esperas introducidas para tolerar la disponibilidad limitada del modelo gratuito.

**Sobre el despliegue:** el log de la prueba verde muestra que se invocó `curl`, pero Render respondió `Not Found`, por lo que **no demuestra un despliegue efectivo**. En cambio, el log de flexibilidad muestra una respuesta con un objeto `deploy.id`, que acredita que **se creó o solicitó un despliegue**; su finalización y estado operativo deben verificarse en los eventos y logs de Render.

## 8. Problemas resueltos y aprendizajes

| Incidente | Acción aplicada o conclusión |
|---|---|
| `exit code 1` inicial | Se localizaron fallas en la evaluación; un código `1` también es la salida correcta cuando no se superan los umbrales. |
| Advertencias del runner de GitHub | Se adoptó Ubuntu 24.04 y se actualizaron las acciones de checkout y Python. |
| Importación desalineada de la aplicación | Las evaluaciones finales usaron `main_3.py`, que permite seleccionar el modelo. |
| Relevancia `0,691` inferior a `0,75` | Se identificó un incumplimiento real de la métrica; se mantuvo el umbral sin rebajarlo. |
| Cuotas de Gemini (`429`) | Se añadió ritmo controlado de llamadas, comprobación de credenciales y manejo de errores temporales. |
| Errores de disponibilidad (`503`/`502`) | Se incorporaron esperas y hasta tres reintentos antes de declarar una evaluación inconclusa. |
| Error `Multiple candidates` de Ragas | Se adaptó la métrica de relevancia a la capacidad del modelo (`strictness=1`). |
| Evaluaciones fallidas computadas como `0.0` | Se separó el fallo del proveedor del resultado real de calidad. |
| Deploy Hook con `Not Found` | Se identificó la necesidad de validar la respuesta HTTP y comprobar el estado de Render. |

## 9. Resolución de problemas frecuentes

| Síntoma | Comprobación |
|---|---|
| Error de importación de `main` | Usar el módulo existente: `main_3:app` y `from main_3 import app` en la versión evaluada. |
| HTTP 400 de Pydantic | Revisar longitud y caracteres permitidos del `prompt`. |
| HTTP 403 | Rechazo por el juez; revisar si el caso era legítimo o un ataque. |
| HTTP 429 de Gemini | Revisar RPM/TPM/RPD y cuota disponible; no asumir que cambiar de API key reinicia límites. |
| HTTP 502/503 | Comprobar mensaje original del proveedor; aplicar reintentos solo para indisponibilidad temporal. |
| `Multiple candidates is not enabled` | Revisar la configuración de `AnswerRelevancy` y la compatibilidad del modelo. |
| Relevancia inferior a 0,75 | Analizar las puntuaciones por pregunta; no bajar el umbral solo para conseguir un despliegue. |
| `Not Found` al llamar a Render | Verificar el Deploy Hook; usar `curl --fail-with-body` y revisar Render Deploys. |
| Modelo de Docker diferente al evaluado | Alinear `dockerfile` con `main_3:app`. |
| MLflow sin historial entre runs de GitHub | Un runner alojado es efímero; configurar almacenamiento de artefactos o tracking remoto para persistencia. |

## 10. Seguridad y limitaciones

- Un patrón de Pydantic es un control estructural, no un mecanismo suficiente contra *prompt injection*.
- Un LLM Juez puede producir falsos positivos y falsos negativos; debe monitorearse y no confundir errores del proveedor con detecciones legítimas.
- La relevancia semántica de Ragas **no prueba por sí misma la corrección factual**; podrían añadirse métricas de fidelidad a la fuente.
- `TestClient` valida la lógica de la aplicación, pero no reemplaza pruebas extremo a extremo sobre la versión efectivamente publicada.
- El registro local de MLflow en GitHub Actions no es persistente entre ejecuciones salvo que se configure almacenamiento adicional.
- Para uso real se requieren autenticación y autorización, límites de costo y tasa, monitoreo, gestión de secretos, pruebas de regresión, verificación del despliegue y procedimientos de reversión.

## 11. Referencias

- [FastAPI](https://fastapi.tiangolo.com/)
- [Pydantic](https://docs.pydantic.dev/)
- [Google Gemini API](https://ai.google.dev/gemini-api/docs)
- [Ragas](https://docs.ragas.io/)
- [MLflow](https://mlflow.org/docs/latest/)
- [GitHub Actions](https://docs.github.com/actions)
- [Render Deploy Hooks](https://render.com/docs/deploy-hooks)
- [Docker](https://docs.docker.com/)

---

**Contexto:** caso práctico de las Unidades 1, 2 y 3 de *Seguridad y Gestión de Riesgos en MLOps*. Las métricas y los incidentes descritos corresponden a las pruebas académicas del 14 de noviembre de 2026.
