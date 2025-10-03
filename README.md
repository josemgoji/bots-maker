# Bot Maker — Plataforma para crear y ejecutar bots con/sin RAG

Plataforma para **definir un BotSpec**, **construir** un bot (ingesta → normalización → chunking → embeddings → indexado) y **ejecutarlo** vía endpoint `/bots/{id}/query` con caché de handles y modo RAG opcional.

## ✨ Características (MVP)

- **Agnóstico de LLM** mediante providers (OpenAI/Azure/HF local).
- **RAG opcional** con VectorDB (Chroma por defecto).
- **BotSpec** declarativo (YAML/JSON) con validación DTO.
- **Pipeline de construcción** (ingesta → limpieza → chunking → embeddings → upsert).
- **Runtime** con caché por `bot_id` y TTL; modo chat puro o RAG.
- **Artefactos** versionados (manifest + prompts).
- **FastAPI** con OpenAPI y CORS.

---

## 🏗️ Arquitectura de flujos

### 1) Creación del bot (`POST /bots`)
```mermaid
flowchart TD
  %% ========= ENDPOINT 1: POST /bots =========
  subgraph A [POST bots]
    A0[Cliente] --> A1[bots_controller POST bots]
    A1 --> A2[Validar BotSpec - dto request]
    A2 --> A3{Spec valida?}
    A3 -- No --> Aerr[400 Bad Request]
    A3 -- Si --> A4[bot_factory - build]
    A4 --> A5[reader - leer fuentes]
    A5 --> A6[normalize - limpiar]
    A6 --> A7[chunking - partir docs]
    A7 --> A8[embeddings - vectorizar]
    A8 --> A9[chroma - crear indice y upsert]
    A9 --> A10[bot_repository - guardar metadatos]
    A10 --> A11[artifact_store - escribir manifest y prompts]
    A11 --> A12[Responder 200: bot_id, version, status ready]
  end
  %% ========= ERRORES COMUNES =========
  classDef err fill:#fee,border:#f66,color:#600
  Aerr:::err
```

### 2) Consulta del bot (`POST /bots/:id/query`)
```mermaid
flowchart TD
  %% ========= ENDPOINT 2: POST /bots/:id/query =========
  subgraph B [POST bots :id query]
    B0[Cliente] --> B1[bots_controller POST bots :id query]
    B1 --> B2[Validar Query - dto request]
    B2 --> B3{BotHandle en cache? bot_registry}
    B3 -- Si --> B4[Usar BotHandle cacheado]
    B3 -- No --> B5[bot_handle_loader - cargar perfil y version]
    B5 --> B6[bot_repository - leer perfil y version activa]
    B6 --> B7{Tiene vectordb index?}
    B7 -- Si --> B8[chroma - preparar retriever]
    B7 -- No --> B8a[Modo chat sin RAG]
    B8 --> B9[agent base - construir con llm y retriever]
    B8a --> B9
    B9 --> B10[Guardar en cache con TTL]
    B4 --> B11{Modo RAG?}
    B10 --> B11
    B11 -- Si --> B12[retriever query k - obtener contexto]
    B12 --> B13[Componer prompt con contexto]
    B11 -- No --> B13a[Componer prompt de chat]
    B13 --> B14[azure_openia - LLM chat]
    B13a --> B14
    B14 --> B15[Formatear respuesta text y sources]
    B15 --> B16[Responder 200 JSON]
  end
```

### 3) Diagrama de secuencia (build)
```mermaid
sequenceDiagram
    autonumber
    participant C as application/controllers/bots_controller.py
    participant DI as application/dependencies/container.py
    participant BF as application/use_cases/build/bot_factory.py
    participant RE as services/ingestion/reader.py
    participant NO as services/ingestion/normalize.py
    participant CH as services/ingestion/chunking.py
    participant EM as services/embeddings.py
    participant VX as services/vectordb/chroma.py
    participant REPO as persistence/bot_repository.py
    participant ART as persistence/artifact_store.py
    participant CFG as config/settings.py

    Note over C: POST /bots (recibe BotSpec)
    C->>DI: get_bot_factory()
    DI-->>C: BotFactory (singleton)

    C->>BF: create(spec)
    activate BF
    BF->>CFG: leer settings (dirs, llaves, etc.)

    BF->>RE: leer fuentes (files/URIs)
    RE-->>BF: documentos crudos

    BF->>NO: normalizar/limpiar
    NO-->>BF: docs normalizados

    BF->>CH: chunking (size/overlap)
    CH-->>BF: chunks

    BF->>EM: embed(chunks)
    EM-->>BF: embeddings

    BF->>VX: create_index(index_name, dim)
    BF->>VX: upsert(index_name, items)
    VX-->>BF: stats (opcional)

    BF->>REPO: create_bot + create_version(status="ready", vectordb_index)
    REPO-->>BF: {bot_id, version}

    BF->>REPO: save_profile(BotProfile)
    BF->>ART: write_manifest(bot_id, version, manifest.json)
    ART-->>BF: manifest_uri

    BF-->>C: {bot_id, version, status="ready"}
    deactivate BF
```

---

## 📁 Estructura (sugerida)

```
.
├── README.md
├── bot_creador_de_bots_blueprint_v_1.md
├── requirements.txt
└── src/
    ├── application/
    │   ├── main.py                   # FastAPI app (incluye routers/controllers)
    │   ├── dependencies/
    │   │   └── container.py          # wiring DI / singletons / settings
    │   └── controllers/
    │       └── bots_controller.py    # POST /bots, POST /bots/{id}/query
    ├── config/
    │   └── settings.py               # Pydantic BaseSettings
    ├── domain/
    │   ├── entities.py               # Bot, BotVersion, BotProfile
    │   └── specs/schema.py           # BotSpec DTO/validación
    ├── persistence/
    │   ├── bot_repository.py         # metadatos/versions/estado
    │   └── artifact_store.py         # manifest + prompts
    ├── services/
    │   ├── embeddings.py             # wrapper embeddings (OpenAI/HF)
    │   ├── llm/                      # providers LLM (openai/azure/hf)
    │   ├── vectordb/chroma.py        # create_index, upsert, retriever
    │   ├── ingestion/
    │   │   ├── reader.py             # files/URIs
    │   │   ├── normalize.py          # limpieza/metadata
    │   │   └── chunking.py           # chunk size/overlap
    │   └── runtime/
    │       ├── bot_handle_loader.py  # carga perfil+version
    │       └── bot_registry.py       # caché TTL por bot_id
    └── utils/
        └── logger.py
```

---

## ⚙️ Configuración

### Requisitos
- Python 3.12+
- (Opcional) Chroma corriendo localmente (o `chromadb` in-memory).

### Instalación
```bash
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### Variables de entorno (`.env`)
```env
# FastAPI
APP_ENV=dev
API_HOST=0.0.0.0
API_PORT=8000
CORS_ORIGINS=*

# LLM (elige solo uno)
OPENAI_API_KEY=
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=
HF_HOME_MODEL=meta-llama/Llama-3.2-3B-Instruct   # si usas transformers local

# Embeddings
EMBEDDINGS_PROVIDER=openai         # openai|hf
EMBEDDINGS_MODEL=text-embedding-3-large

# VectorDB
VECTORDB_PROVIDER=chroma
VECTORDB_DIR=./data/chroma

# Artefactos / Storage local
ARTIFACTS_DIR=./data/artifacts
```

> `config/settings.py` debe cargar estas variables con **Pydantic BaseSettings** y exponer un singleton en `dependencies/container.py`.

---

## ▶️ Ejecución

```bash
uvicorn src.application.main:app --host 0.0.0.0 --port 8000 --reload
```
- Docs: `http://localhost:8000/docs`
- Healthcheck recomendado en `main.py` (`GET /health`).

---

## 📜 Esquema de BotSpec (mínimo)

```yaml
bot:
  name: "Mi primer bot"
  language: es
  tone: friendly
  provider:
    llm: openai         # openai|azure|hf
    model: gpt-4o-mini  # o tu modelo local HF
    temperature: 0.2
  rag:
    enabled: true
    k: 6
  datasets:
    - type: files
      files:
        - ./samples/guia.pdf
      chunking:
        size: 800
        overlap: 120
```

> El controlador valida el DTO (Pydantic) y responde `400` si el spec es inválido.

---

## 🧪 Endpoints (ejemplos)

### 1) Crear bot
`POST /bots`

```json
{
  "bot": {
    "name": "FAQ Compañía",
    "language": "es",
    "tone": "neutral",
    "provider": { "llm": "openai", "model": "gpt-4o-mini", "temperature": 0.2 },
    "rag": { "enabled": true, "k": 6 },
    "datasets": [
      {
        "type": "files",
        "files": ["./samples/faq.pdf"],
        "chunking": { "size": 800, "overlap": 120 }
      }
    ]
  }
}
```

**Respuesta 200**
```json
{
  "bot_id": "bot_123",
  "version": "v1",
  "status": "ready"
}
```

### 2) Consultar bot (RAG o chat puro)
`POST /bots/{bot_id}/query`

```json
{
  "query": "¿Cómo solicito soporte técnico?",
  "mode": "rag",
  "max_tokens": 300
}
```

**Respuesta 200**
```json
{
  "text": "Puedes abrir un ticket en ...",
  "sources": [
    {"id":"faq.pdf#p2","score":0.83,"snippet":"Para soporte técnico..."}
  ],
  "usage": {"tokens_in": 122, "tokens_out": 91, "latency_ms": 740}
}
```

---

## 🧩 Componentes clave (MVP)

- **`bot_factory.py`**: orquesta el build (ingesta → normalizar → chunking → embeddings → index → persistencia → artefactos).
- **`bot_repository.py`**: crea/lee metadatos de `bot_id`, `version`, estado y referencias a índices.
- **`artifact_store.py`**: guarda `manifest.json`, prompts y configuración de runtime por versión.
- **`bot_registry.py`**: caché en memoria (TTL) de `BotHandle` para acelerar `/query`.
- **`chroma.py`**: crea índice y expone `as_retriever(k)`; upsert de embeddings.

---

## ✅ Checklist de calidad (útil durante el dev)

- [ ] Validación estricta de BotSpec (Pydantic + JSONSchema opcional).
- [ ] Hash de chunk para **cache de embeddings** (no re-embed en rebuild).
- [ ] Manejo de errores: 400 (spec/query inválida), 404 (bot no existe), 500 (fallo interno).
- [ ] Logs con `logger.py` (niveles + contexto de `bot_id`/`version`/`req_id`).
- [ ] Métricas básicas: latencia, tokens, hits de caché.
- [ ] Tests de smoke para `/bots` y `/bots/{id}/query`.

---

## 🧭 Roadmap corto

- Streaming de respuestas (Server-Sent Events).
- Reranking barato (RapidFuzz) previo al LLM.
- Evaluación automática post-build y promoción de versión.
- Adapters extra: FAISS / Elastic, Bedrock / Azure OpenAI.
- Widget embebible (iframe/script) y API keys por workspace.

---

## 🤝 Contribuir

1. Crea un branch `feat/xxxx` o `fix/xxxx`.
2. Añade tests y docstrings.
3. PR con descripción breve (ES/EN), incluye ejemplos y screenshots si aplica.

---

## 🧰 Comandos útiles (sugeridos)

```bash
# Lint/format (si usas ruff/black)
ruff check src && black src

# Tests
pytest -q

# Correr API (dev)
uvicorn src.application.main:app --reload
```

---

## 🆘 Notas de implementación

- Si `rag.enabled=false`, el runtime construye un **prompt de chat** sin contexto.
- El **index name** puede ser `{bot_id}:{version}`; guarda su referencia en `bot_repository`.
- El **perfil de ejecución** (`BotProfile`) incluye tono, límites y provider LLM; se serializa en artefactos.
- La **caché** de `BotHandle` invalídala cuando cambie la `version` activa.
