# GCP Agents Training & ADK Workspace

Este repositorio contiene el código, las habilidades de desarrollo (**ADK Skills Suite**) y los agentes desarrollados siguiendo el curso oficial de Google Cloud: **Developing Agents with Google Agents CLI and ADK**.

---

## 📁 Estructura del Repositorio

```text
.
├── .agents/                    # ADK Skills Suite (workflow, adk-code, eval, deploy, etc.)
├── skills-lock.json            # Lockfile con versiones de las habilidades de agents-cli
├── ambient-expense-agent/      # Agente de Aprobación de Gastos (Lección 4)
│   ├── app/
│   │   ├── agent.py            # Lógica principal del agente, herramientas y reglas de negocio
│   │   ├── __init__.py
│   │   └── fast_api_app.py     # Servidor FastAPI para endpoints y triggers Pub/Sub
│   ├── tests/
│   │   └── eval/
│   │       ├── datasets/
│   │       │   └── basic-dataset.json  # Dataset con los 5 casos de prueba de evaluación
│   │       ├── eval_config.yaml        # Configuración de métricas de evaluación
│   │       └── response_quality.py     # LLM-as-a-judge evaluador (Gemini 2.5 Flash)
│   ├── artifacts/
│   │   ├── traces/             # Trazas de inferencia generadas por `eval generate`
│   │   └── grade_results/      # Reportes JSON y HTML generados por `eval grade` (Score: 5.0/5.0)
│   ├── .env.example            # Plantilla para variables de entorno (no trackeado en git)
│   ├── pyproject.toml          # Dependencias gestionadas por uv / pip
│   ├── uv.lock                 # Lockfile de dependencias exactas
│   └── agents-cli-manifest.yaml# Manifiesto del agente para agents-cli
└── README.md                   # Esta guía
```

---

## 🚀 Cómo configurar y recrear el entorno en una nueva máquina

Sigue estos pasos cuando clones el repositorio en otra computadora para recrear los archivos y dependencias no trackeados en git (`.env`, `.venv`, credenciales ADC).

### 1. Prerrequisitos
Asegúrate de contar con:
* **Python 3.11+**
* **Google Cloud SDK (`gcloud`)** instalado y en el `PATH`.
* **uv** (gestor de paquetes y herramientas Python).

Si no tienes `uv` instalado:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Instala `agents-cli`:
```bash
uv tool install google-agents-cli
```

Asegúrate de que `~/.local/bin` esté en tu `PATH`:
```bash
export PATH="$HOME/.local/bin:$PATH"
```

---

### 2. Autenticación con Google Cloud (Application Default Credentials)
El agente utiliza Vertex AI y requiere credenciales locales (ADC) asociadas al proyecto de GCP:

```bash
# 1. Login interactivo
gcloud auth login

# 2. Configurar Application Default Credentials (ADC)
gcloud auth application-default login

# 3. Establecer el proyecto activo
gcloud config set project agent-with-billing
```

---

### 3. Recrear el archivo de entorno (`.env`)
Por seguridad, los archivos `.env` con configuraciones locales no se suben al repositorio. Recrea el archivo a partir de la plantilla `.env.example`:

```bash
cd ambient-expense-agent
cp .env.example .env
```

Verifica que el contenido de `.env` sea:
```env
GOOGLE_GENAI_USE_ENTERPRISE=true
GOOGLE_CLOUD_PROJECT=agent-with-billing
GOOGLE_CLOUD_LOCATION=us-central1
```

---

### 4. Recrear el Entorno Virtual (`.venv`) e instalar dependencias
Ejecuta el comando oficial de `agents-cli`:

```bash
# Estando dentro de ambient-expense-agent:
agents-cli install
```
*(Alternativamente puedes usar: `uv sync --dev --extra lint`)*

---

### 5. Verificación de Código y Linting
Ejecuta los validadores de tipos y formateo (`ruff`, `codespell`, `ty`):

```bash
agents-cli lint
```
*Debe indicar: `All checks passed!`*

---

### 6. Ejecución y Pruebas Locales

#### Opción A: Pruebas rápidas por CLI (`agents-cli run`)
Prueba de gasto bajo (< \$100 $\rightarrow$ auto-aprobación):
```bash
agents-cli run 'Approve expense for $45 from bob@company.com, category meals, description Team lunch'
```

Prueba de gasto alto (>= \$100 $\rightarrow$ enrutado a aprobación humana):
```bash
agents-cli run 'Approve expense for $250 from alice@company.com, category equipment, description Monitor'
```

Prueba enviando un payload JSON:
```bash
agents-cli run '{"amount": 45, "submitter": "bob@company.com", "category": "meals", "description": "Team lunch", "date": "2026-04-12"}'
```

#### Opción B: Playground Interactivo Web
Para probarlo con interfaz gráfica y hot-reload:
```bash
agents-cli dev
```
Abre tu navegador en: [http://localhost:8080/dev-ui](http://localhost:8080/dev-ui).

---

### 7. Ciclo de Evaluación Automatizada (Quality Flywheel)
El proyecto incluye un dataset en `tests/eval/datasets/basic-dataset.json` con los 5 escenarios del curso de Google Cloud:
1. Auto-aprobación (< \$100).
2. Aprobación humana (>= \$100).
3. Redacción y contención de PII (SSN en la descripción).
4. Detección y rechazo de Prompt Injection.
5. Gasto limpio de alto valor para rechazo/revisión de equipamiento.

Para correr la evaluación completa:
```bash
# 1. Generar trazas de ejecución
agents-cli eval generate

# 2. Calificar trazas con LLM-as-a-judge (Gemini 2.5 Flash)
agents-cli eval grade --qps 1
```

Los resultados y el reporte visual en HTML se guardarán automáticamente en `artifacts/grade_results/`.
*(Último resultado registrado: **5.0000 / 5.0000** sin fallas).*
