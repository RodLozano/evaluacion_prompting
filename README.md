# 🎮 Análisis de Reseñas de Videojuegos con LLMs
### Entregable Final — Módulo de Prompt Engineering
**Master in Business Analytics · UAM + Accenture**

---

## 📋 Descripción del proyecto

Este proyecto implementa un **pipeline de dos LLMs en cadena** para extraer información estructurada a partir de reseñas de videojuegos en texto libre.

El enfoque divide una tarea compleja en dos subtareas controlables e independientes:

1. **Filtrado semántico** — Un primer LLM clasifica cada reseña como `RELEVANTE` o `IRRELEVANTE`, eliminando spam, texto vacío y contenido sin valor informativo.
2. **Extracción estructurada** — Un segundo LLM procesa únicamente las reseñas relevantes y genera un objeto JSON con atributos clave de cada reseña.

---

## 🗂️ Estructura del repositorio

```
📦 evaluacion_prompting/
├── 📓 entregable_prompt_engineering.ipynb     # Notebook principal ejecutable
├── 📁 input/
│   └── videogames_reviews.csv                 # Dataset de entrada
├── 📁 output/
│   └── analisis_videojuegos_resultados_final.csv  # Output final estructurado
├── 📄 README.md                               # Este archivo
└── 📄 .env                                    # API keys (NO subir a GitHub)
```

---

## ⚙️ Requisitos e instalación

### Dependencias

```bash
pip install groq pandas tqdm python-dotenv ipykernel
```

### Configuración de la API Key

1. Regístrate en [console.groq.com](https://console.groq.com) y genera una API key gratuita
2. Crea un archivo `.env` en la raíz del proyecto:

```
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ Añade `.env` a tu `.gitignore`. Nunca subas la API key a GitHub.

### Entorno virtual

```bash
python -m venv venv
source venv/bin/activate          # Mac/Linux
venv\Scripts\activate             # Windows

pip install groq pandas tqdm python-dotenv ipykernel
python -m ipykernel install --user --name=venv
```

---

## 🚀 Cómo ejecutar

1. Coloca el archivo `videogames_reviews.csv` en la carpeta `input/`
2. Abre `entregable_prompt_engineering.ipynb` en VS Code o Jupyter
3. Selecciona el kernel `venv`
4. Ejecuta todas las celdas en orden

El resultado final se guardará automáticamente en `output/analisis_videojuegos_resultados_final.csv`.

---

## 🧠 Arquitectura del pipeline

```
videogames_reviews.csv
        │
        ▼
┌─────────────────────────┐
│  PASO 1 — Selección     │
│  Top 100 reseñas        │  Ordenadas por longitud de contenido (mayor → menor)
│  más largas             │  Garantiza suficiente texto para extraer información
└──────────┬──────────────┘
           │ 100 reseñas
           ▼
┌─────────────────────────┐
│  PASO 2 — Filtrado      │  Modelo: llama-3.1-8b-instant
│  LLM #1                 │  Batch: 2 reseñas por llamada
│  RELEVANTE/IRRELEVANTE  │  Temperature: 0.0 | Sleep: 2.5s entre batches
└──────────┬──────────────┘
           │ N reseñas relevantes
           ▼
┌─────────────────────────┐
│  PASO 3 — Extracción    │  Modelo: llama-3.3-70b-versatile
│  LLM #2                 │  Batch: 1 reseña por llamada
│  JSON estructurado      │  Temperature: 0.0 | Sleep: 4.5s entre llamadas
└──────────┬──────────────┘
           │
           ▼
analisis_videojuegos_resultados_final.csv
```

---

## 🤖 Modelos y justificación

| Rol | Modelo | Justificación |
|-----|--------|---------------|
| LLM #1 — Filtrado | `llama-3.1-8b-instant` | Tarea binaria simple que no requiere razonamiento complejo. Modelo ligero, muy rápido (560 t/s) y con rate limits más generosos en el tier gratuito. |
| LLM #2 — Extracción | `llama-3.3-70b-versatile` | La extracción de JSON estructurado requiere mayor comprensión del texto. El modelo grande produce outputs más coherentes y con menos errores de formato. |

**Proveedor elegido: [Groq](https://groq.com)**  
Groq utiliza LPU (Language Processing Unit), lo que permite inferencia ultra-rápida. Su tier gratuito es suficiente para este pipeline y su latencia baja lo hace ideal para bucles con múltiples llamadas secuenciales.

---

## 📐 Decisiones de Prompt Engineering

### LLM #1 — Prompt de filtrado

- **Tarea binaria** (`RELEVANTE` / `IRRELEVANTE`): salida simple y parseable sin ambigüedad
- **Criterios explícitos** de irrelevancia: spam, emojis, texto vacío, frases sin contenido semántico
- **Campo `razon`** en el output: permite auditar y depurar el pipeline fácilmente
- **Output en JSON array**: parseo robusto con `find("[")` / `rfind("]")` aunque el modelo añada texto alrededor
- **Temperature = 0.0**: clasificación determinista

```json
[
  {"id": 5, "decision": "RELEVANTE", "razon": "expresa opinión clara sobre gameplay y gráficos"},
  {"id": 6, "decision": "IRRELEVANTE", "razon": "texto vacío, solo contiene emojis"}
]
```

### LLM #2 — Prompt de extracción

- **System prompt separado**: ancla el rol del modelo como analista experto antes del user prompt
- **Schema JSON explícito**: el modelo recibe exactamente la estructura que debe devolver, reduciendo alucinaciones de formato
- **Valores cerrados** en campos categóricos (Dificultad, GeneroPercibido): facilita el análisis posterior y evita valores libres inconsistentes
- **Triple comilla** para delimitar el texto de la reseña: previene inyección de prompt desde el contenido de la reseña
- **1 reseña por llamada**: evita que el modelo mezcle entidades entre reseñas distintas
- **Temperature = 0.0**: output determinista y parseable

---

## 📊 Campos extraídos (schema de salida)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `Id` | int | Identificador de la reseña |
| `SentimientoGeneral` | string | `Positivo` / `Negativo` / `Neutral` |
| `AspectosPositivos` | list | Aspectos del juego valorados positivamente |
| `AspectosNegativos` | list | Aspectos del juego valorados negativamente |
| `Dificultad` | string | `Demasiado Fácil` / `Fácil` / `Equilibrado` / `Difícil` / `Demasiado Difícil` / `No Mencionado` |
| `GeneroPercibido` | string | `Acción` / `RPG` / `Aventura` / `Estrategia` / `Terror` / ... / `No Mencionado` |
| `HorasJugadas` | string/null | Horas mencionadas en la reseña, o `null` si no se indica |
| `Recomendado` | int | Ground truth del CSV original (1 = Recomendado, 0 = No recomendado) |

---

## ⚡ Gestión de rate limits

Las 100 reseñas seleccionadas son las más largas del dataset, con una longitud media de ~6.700 caracteres (~1.700 tokens). Esto obligó a ajustar el batching para no superar el límite de **6.000 TPM** del tier gratuito de Groq:

| Parámetro | Valor | Razonamiento |
|-----------|-------|--------------|
| `BATCH_SIZE_FILTER` | 2 | 2 × ~1.700 tokens ≈ 3.400 tokens/llamada → por debajo del límite de 6.000 TPM |
| `SLEEP_FILTER` | 2.5s | Espaciado entre batches para no acumular tokens en la ventana de 1 minuto |
| `BATCH_SIZE_EXTRACT` | 1 | Máxima precisión; 1 reseña por llamada evita mezcla de entidades |
| `SLEEP_EXTRACT` | 4.5s | Respeta el límite de 1K RPM de `llama-3.3-70b-versatile` |

---

## 🔧 Robustez y manejo de errores

- **3 reintentos automáticos** por llamada ante errores de red, timeout o JSON malformado
- **Parseo defensivo**: `find("{")` / `rfind("}")` extrae el JSON aunque el modelo añada texto introductorio
- **Fallback conservador en filtrado**: si una llamada falla tras 3 intentos, la reseña se marca como `RELEVANTE` para no perder datos
- **Fallback en extracción**: si falla la extracción, se devuelve un objeto vacío con el ID y el campo `Recomendado` del CSV original
- **Logs con `tqdm`**: barra de progreso en tiempo real para monitorizar el pipeline

---

## 👤 Autor

Entregable desarrollado para el **Módulo de Prompt Engineering**  
Master in Business Analytics — UAM + Accenture
