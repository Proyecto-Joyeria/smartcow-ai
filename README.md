# 🧠 SmartCow AI

Servicio de Inteligencia Artificial de **SmartCow Tracker** — Predicción de salud animal mediante análisis de series de tiempo de signos vitales con Machine Learning.

---

## 🧱 Stack Tecnológico

| Categoría | Tecnología | Versión |
|-----------|------------|---------|
| Lenguaje | Python | 3.11 |
| Framework API | FastAPI | 0.115.x |
| Servidor ASGI | Uvicorn | 0.30.x |
| Validación | Pydantic | 2.x |
| ML | scikit-learn | 1.5.x |
| Manipulación de datos | pandas | 2.2.x |
| Cómputo numérico | NumPy | 1.26.x |
| Serialización de modelos | joblib | 1.4.x |
| HTTP Client | httpx | 0.27.x |
| Tests | pytest | latest |

---

## 📁 Estructura del Proyecto

```
smartcow-ai/
├── app/
│   ├── routers/
│   │   └── predict.py        # Endpoint POST /predict/health
│   ├── services/
│   │   └── prediction.py     # Lógica de predicción e inferencia
│   ├── models/
│   │   └── health_model.py   # Carga y gestión del modelo ML
│   ├── schemas/
│   │   └── prediction.py     # Schemas Pydantic (request/response)
│   ├── core/
│   │   ├── config.py         # Variables de entorno (pydantic-settings)
│   │   └── logging.py        # Configuración de logs
│   └── main.py               # Entry point FastAPI
├── ml/
│   ├── training/
│   │   └── train.py          # Script de entrenamiento del modelo
│   ├── models/               # Modelos serializados (.joblib)
│   └── data/                 # Datasets de entrenamiento (ignorados en git)
├── tests/
│   ├── test_predict.py
│   └── test_model.py
├── requirements.txt
├── Dockerfile
└── .env.example
```

---

## 🚀 Inicio Rápido

### Prerrequisitos

- Python >= 3.11
- pip

### 1. Clonar e instalar

```bash
git clone https://github.com/tu-usuario/smartcow-ai.git
cd smartcow-ai

# Crear y activar entorno virtual
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# Instalar dependencias
pip install -r requirements.txt
```

### 2. Variables de entorno

```bash
cp .env.example .env
# Editar .env según necesidad
```

### 3. Levantar en desarrollo

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

El servicio corre en `http://localhost:8000`.

La documentación interactiva (Swagger UI) está en `http://localhost:8000/docs`.

### 4. Con Docker

```bash
docker build -t smartcow-ai .
docker run -p 8000:8000 --env-file .env smartcow-ai
```

---

## 🔑 Variables de Entorno

```env
# App
APP_ENV=development
APP_PORT=8000

# Modelo
MODEL_PATH=ml/models/health_model.joblib
MODEL_VERSION=1.0.0

# Logging
LOG_LEVEL=INFO
```

---

## 📡 API

Base URL: `http://localhost:8000`

### `POST /predict/health`

Recibe las últimas 72 horas de signos vitales de un animal y retorna la predicción de riesgo de salud.

**Request:**
```json
{
  "animalId": "clx1a2b3c4d5e6f7",
  "vitals": [
    {
      "timestamp": 1748390400000,
      "tempC": 38.4,
      "heartBpm": 62,
      "activity": 75
    }
  ]
}
```

**Response:**
```json
{
  "riskScore": 0.72,
  "riskLevel": "HIGH",
  "confidence": 0.88,
  "topFeatures": [
    { "feature": "temp_trend_6h", "importance": 0.45 },
    { "feature": "activity_mean_24h", "importance": 0.31 },
    { "feature": "heart_std_1h", "importance": 0.24 }
  ],
  "recommendation": "Temperatura en tendencia ascendente durante las últimas 6 horas. Se recomienda revisión veterinaria en las próximas 12 horas."
}
```

### `GET /health`

Health check del servicio.

```json
{
  "status": "ok",
  "model_version": "1.0.0",
  "model_loaded": true
}
```

---

## 🤖 Modelo de Machine Learning

### Arquitectura

El modelo es un **Random Forest Classifier** con 200 árboles entrenado sobre series de tiempo de signos vitales bovinos.

### Pipeline de preprocesamiento

1. **Interpolación** de valores faltantes en la serie de tiempo
2. **Normalización Z-score** de temperatura, ritmo cardíaco y actividad
3. **Extracción de features** en ventanas de 1h, 6h y 24h:
   - Media, desviación estándar, tendencia (pendiente de regresión lineal)
   - Cruces por umbral (temperatura > 39°C, actividad < 10)
   - Variabilidad del ritmo cardíaco (HRV aproximado)

### Salida

- `riskScore`: probabilidad de evento de salud en las próximas 24-48h (0.0 - 1.0)
- `riskLevel`: umbral LOW (< 0.35) / MEDIUM (0.35-0.65) / HIGH (> 0.65)
- `topFeatures`: importancia de las 3 features más influyentes (explicabilidad)
- `recommendation`: texto generado con la acción sugerida

### Reentrenamiento

```bash
# Ejecutar script de entrenamiento con nuevos datos
python ml/training/train.py --data ml/data/vitals_labeled.csv

# El modelo se serializa automáticamente con versión y fecha
# Ej: ml/models/health_model_v20260528.joblib
```

Los modelos se versionan por fecha y se almacenan en S3 en producción.

---

## 🧪 Tests

```bash
# Activar entorno virtual
source venv/bin/activate

# Ejecutar todos los tests
pytest

# Con cobertura
pytest --cov=app tests/

# Verbose
pytest -v
```

---

## 🐳 Docker

```bash
# Build
docker build -t smartcow-ai .

# Ejecutar
docker run -p 8000:8000 --env-file .env smartcow-ai

# Ver logs
docker logs -f smartcow-ai
```

---

## 🔗 Integración con el Backend

Este servicio es consumido exclusivamente por `smartcow-api` a través de la red interna del Docker Compose. El backend actúa como intermediario:

1. El cliente web solicita una predicción para el animal `X`
2. `smartcow-api` recopila las últimas 72 horas de vitales desde PostgreSQL
3. `smartcow-api` llama a `POST http://smartcow-ai:8000/predict/health`
4. El resultado se devuelve al cliente y se almacena en la tabla `ai_predictions`

> **Circuit Breaker:** Si el servicio falla 3 veces consecutivas, `smartcow-api` abre el circuit breaker y devuelve `"Predicción temporalmente no disponible"` sin propagar el error al usuario.

---

## 📋 Scripts Disponibles

| Comando | Descripción |
|---------|-------------|
| `uvicorn app.main:app --reload` | Servidor de desarrollo con hot reload |
| `pytest` | Ejecutar tests |
| `pytest --cov=app` | Tests con cobertura |
| `python ml/training/train.py` | Entrenar el modelo con nuevos datos |

---

## 📄 Documentación Técnica

La documentación completa del proyecto se encuentra en el repositorio [`smartcow-infra`](https://github.com/tu-usuario/smartcow-infra):

- `SDD v1.0.0` — Sección 3.6: Diseño del módulo de IA
- `ADR-006` — Decisión: Servicio de IA como contenedor Python independiente
- `IDD v1.0.0` — Contrato del endpoint `/predict/health`

---

## 🤝 Contribución

1. Crea una rama desde `develop`: `git checkout -b feature/nombre-feature`
2. Activa el entorno virtual y haz tus cambios
3. Asegúrate de pasar los tests: `pytest`
4. Abre un Pull Request hacia `develop`

---

## 📝 Licencia

Proyecto — SmartCow Tracker © 2026
