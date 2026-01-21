# 🌐 Test RAG Frontend

Interfaz web para el sistema **Test RAG API** construida con Vue 3 y Vite.

---

## 🚀 Quick Start - How to Run

### Run Frontend Locally

**Prerequisites:** Node.js 22+ installed, Backend API running on `http://localhost:8000`.

```bash
# Navigate to frontend directory
cd /mnt/c/dev/workspace/frontend/test-rag-frontend

# Install dependencies
npm install

# Run development server
npm run dev
```

**Access:** http://localhost:5173

**Note:** The frontend expects the backend API to be available at `http://localhost:8000`. Make sure the backend is running before starting the frontend.

---

## 📑 Tabla de Contenidos

- [Quick Start - How to Run](#🚀-quick-start---how-to-run)
- [Proyectos](#📦-proyectos)
- [Arquitectura](#️🏗️-arquitectura)
- [Tecnologías](#🚀-tecnologías)
- [Prerequisitos](#📋-prerequisitos)
- [Instalación](#️🛠️-instalación)
  - [Backend (Desarrollo local)](#1-backend-desarrollo-local)
  - [Backend (Docker)](#2-backend-docker)
  - [Frontend](#3-frontend)
- [API Endpoints](#📡-api-endpoints)
  - [Salud del sistema](#salud-del-sistema)
  - [Agregar documento](#agregar-documento)
  - [Hacer pregunta (RAG)](#hacer-pregunta-rag)
  - [Historial de queries](#historial-de-queries)
  - [Detalle de query específica](#detalle-de-query-específica)
- [Base de Datos](#🗄️-base-de-datos)
- [Docker](#🐳-docker)
- [Workflow RAG](#🎯-workflow-rag)
- [Testing](#🧪-testing)
- [Variables de Entorno](#📝-variables-de-entorno)
- [Roadmap](#🚧-roadmap)
- [Licencia](#📄-licencia)
- [Autor](#👤-autor)

---

## 📦 Proyectos

| Proyecto | Ubicación | Descripción |
|----------|-----------|-------------|
| **test-rag-api** | `/mnt/c/dev/workspace/ia/test-rag-api` | Backend FastAPI con RAG |
| **test-rag-frontend** | `/mnt/c/dev/workspace/frontend/test-rag-frontend` | Frontend Vue 3 |

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🏗️ Arquitectura

```
┌─────────────────────┐
│  test-rag-frontend  │  Vue 3 + Vite
│    (Port 5173)      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   test-rag-api      │  FastAPI REST API
│    (Port 8000)      │
└──────────┬──────────┘
           │
           ├──────────────┬──────────────┬──────────────┐
           ▼              ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
    │PostgreSQL│   │  Qdrant  │   │  Ollama  │   │   GPU    │
    │ (5433)   │   │ (6335)   │   │ (11434)  │   │ RTX 5070 │
    └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🚀 Tecnologías

### Backend (test-rag-api)
- **FastAPI** - Framework web moderno para Python
- **PostgreSQL** - Base de datos relacional (historial de queries)
- **Qdrant** - Base de datos vectorial (embeddings)
- **Ollama** - Ejecución local de LLMs con GPU
- **SQLAlchemy** - ORM para PostgreSQL
- **UV** - Gestor de paquetes Python (10-100x más rápido que pip)

### Frontend (test-rag-frontend)
- **Vue 3** - Framework JavaScript progresivo
- **Vite** - Build tool ultrarrápido
- **Axios** - Cliente HTTP

### Modelos IA
- **qwen2.5:7b** - Modelo general (embeddings + generación)
- **qwen2.5-coder:14b** - Modelo especializado en código

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 📋 Prerequisitos

- Docker Desktop
- WSL2 (Windows) o Linux
- NVIDIA GPU con drivers instalados
- Ollama instalado en host
- Node.js 22+ (para frontend)
- Python 3.12+ (para desarrollo local)
- UV (gestor de paquetes Python)

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🛠️ Instalación

### 1. Backend (Desarrollo local)

```bash
# Navegar al proyecto backend
cd /mnt/c/dev/workspace/ia/test-rag-api

# Instalar dependencias con UV
uv sync

# Configurar variables de entorno
cp .env.example .env
# Editar .env con tus configuraciones

# Ejecutar servidor de desarrollo
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Acceso:** http://localhost:8000

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

### 2. Backend (Docker)

```bash
# Navegar al proyecto backend
cd /mnt/c/dev/workspace/ia/test-rag-api

# Levantar stack completo (API + PostgreSQL + Qdrant)
docker compose -f docker-compose.full.yml up -d

# Ver logs en tiempo real
docker compose -f docker-compose.full.yml logs -f

# Ver estado de servicios
docker compose -f docker-compose.full.yml ps

# Parar servicios
docker compose -f docker-compose.full.yml down

# Parar y eliminar volúmenes (¡cuidado, borra datos!)
docker compose -f docker-compose.full.yml down -v
```

**Acceso:** http://localhost:8000

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

### 3. Frontend

```bash
# Navegar al proyecto frontend
cd /mnt/c/dev/workspace/frontend/test-rag-frontend

# Instalar dependencias
npm install

# Ejecutar servidor de desarrollo
npm run dev

# Build para producción
npm run build
```

**Acceso:** http://localhost:5173

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 📡 API Endpoints

### Salud del sistema

```bash
GET /health
```

**Respuesta:**
```json
{
  "api": "ok",
  "postgres": "ok",
  "qdrant": "ok",
  "ollama": "ok"
}
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

### Agregar documento

```bash
POST /documents
Content-Type: application/json

{
  "text": "FastAPI es un framework moderno de Python para construir APIs."
}
```

**Respuesta:**
```json
{
  "status": "ok",
  "id": 76473992,
  "text": "FastAPI es un framework moderno de Python para construir APIs.",
  "embedding_size": 3584
}
```

**Ejemplo curl:**
```bash
curl -X POST http://localhost:8000/documents \
  -H "Content-Type: application/json" \
  -d '{"text": "FastAPI es un framework moderno de Python."}'
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

### Hacer pregunta (RAG)

```bash
POST /query
Content-Type: application/json

{
  "question": "¿Qué es FastAPI?"
}
```

**Respuesta:**
```json
{
  "id": 1,
  "question": "¿Qué es FastAPI?",
  "answer": "FastAPI es un framework moderno de Python para construir APIs rápidas y fáciles de usar.",
  "sources": [
    "FastAPI es un framework moderno de Python para construir APIs."
  ],
  "saved_to_db": true
}
```

**Ejemplo curl:**
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "¿Qué es FastAPI?"}'
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

### Historial de queries

```bash
GET /history?limit=10
```

**Respuesta:**
```json
{
  "total": 1,
  "queries": [
    {
      "id": 1,
      "question": "¿Qué es FastAPI?",
      "answer": "FastAPI es un framework moderno...",
      "sources_count": 1,
      "model": "qwen2.5:7b",
      "created_at": "2026-01-11T16:20:02.662676"
    }
  ]
}
```

**Ejemplo curl:**
```bash
curl http://localhost:8000/history?limit=20
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

### Detalle de query específica

```bash
GET /history/{query_id}
```

**Respuesta:**
```json
{
  "id": 1,
  "question": "¿Qué es FastAPI?",
  "answer": "FastAPI es un framework moderno de Python...",
  "sources": [
    "FastAPI es un framework moderno de Python."
  ],
  "model": "qwen2.5:7b",
  "created_at": "2026-01-11T16:20:02.662676"
}
```

**Ejemplo curl:**
```bash
curl http://localhost:8000/history/1
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🗄️ Base de datos

### PostgreSQL

**Tabla: query_history**

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | INTEGER | Primary key (autoincremental) |
| question | TEXT | Pregunta del usuario |
| answer | TEXT | Respuesta generada por el LLM |
| sources | JSON | Array de documentos fuente utilizados |
| model_used | VARCHAR(100) | Nombre del modelo LLM usado |
| created_at | DATETIME | Timestamp de creación (UTC) |

**Ejemplo de registro:**
```sql
SELECT * FROM query_history WHERE id = 1;
```

### Qdrant

**Colección: test_docs**

- **Vector size:** 3584 dimensiones (qwen2.5:7b embeddings)
- **Distance:** Cosine similarity
- **Payload schema:** 
  ```json
  {
    "text": "Contenido del documento original"
  }
  ```

**Verificar colección:**
```bash
curl http://localhost:6335/collections/test_docs
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🐳 Docker

### Servicios

| Servicio | Imagen | Puerto Host | Puerto Interno | Descripción |
|----------|--------|-------------|----------------|-------------|
| api | test-rag-api | 8000 | 8000 | FastAPI backend |
| postgres | postgres:16-alpine | 5433 | 5432 | PostgreSQL database |
| qdrant | qdrant/qdrant:latest | 6335 | 6333 | Vector database |

### Volúmenes persistentes

| Volumen | Descripción | Ubicación |
|---------|-------------|-----------|
| `postgres-data` | Datos PostgreSQL | Managed by Docker |
| `qdrant-data` | Vectores e índices Qdrant | Managed by Docker |

### Network

- **Nombre:** `rag-network`
- **Driver:** bridge
- **Comunicación interna:** Todos los servicios pueden comunicarse usando nombres de servicio

### Comandos útiles

```bash
# Ver logs de servicio específico
docker compose -f docker-compose.full.yml logs api
docker compose -f docker-compose.full.yml logs postgres
docker compose -f docker-compose.full.yml logs qdrant

# Reiniciar servicio específico
docker compose -f docker-compose.full.yml restart api

# Entrar a contenedor
docker exec -it test-rag-api bash
docker exec -it test-rag-postgres psql -U dev -d test_rag

# Ver uso de recursos
docker stats

# Backup PostgreSQL
docker exec test-rag-postgres pg_dump -U dev test_rag > backup.sql
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🎯 Workflow RAG

```
┌─────────────────────────────────────────────────────────────────┐
│                      FLUJO COMPLETO RAG                         │
└─────────────────────────────────────────────────────────────────┘

1. 📝 Usuario pregunta
   └─> Frontend (test-rag-frontend) envía POST /query

2. 🔢 Generar embedding de la pregunta
   └─> test-rag-api → Ollama: convierte pregunta a vector (3584 dim)

3. 🔍 Búsqueda semántica
   └─> test-rag-api → Qdrant: encuentra top 3 documentos similares
   └─> Usa cosine similarity entre vectores

4. 📋 Construcción de prompt
   └─> Si hay documentos relevantes (score > 0.5):
       ├─> Contexto = documentos encontrados
       └─> Prompt = "Contexto: {docs}\n\nPregunta: {question}"
   └─> Si NO hay documentos relevantes:
       └─> Prompt = "Pregunta: {question}" (respuesta directa)

5. 🤖 Generación de respuesta
   └─> test-rag-api → Ollama: genera respuesta basada en prompt
   └─> Modelo: qwen2.5:7b (local, GPU acelerada)

6. 💾 Persistencia
   └─> test-rag-api → PostgreSQL: guarda query + answer + sources

7. ✅ Respuesta al usuario
   └─> test-rag-api → Frontend: JSON con answer + sources + id

8. 📊 Mostrar resultado
   └─> Frontend muestra respuesta + fuentes utilizadas
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🧪 Testing

### Test completo del flujo

```bash
# 1. Verificar salud del sistema
curl http://localhost:8000/health

# 2. Agregar documentos de prueba
curl -X POST http://localhost:8000/documents \
  -H "Content-Type: application/json" \
  -d '{"text": "Python es un lenguaje de programación interpretado y de alto nivel."}'

curl -X POST http://localhost:8000/documents \
  -H "Content-Type: application/json" \
  -d '{"text": "FastAPI es un framework web moderno para construir APIs con Python."}'

curl -X POST http://localhost:8000/documents \
  -H "Content-Type: application/json" \
  -d '{"text": "Qdrant es una base de datos vectorial open source escrita en Rust."}'

# 3. Hacer pregunta relacionada (debe usar RAG)
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "¿Qué es FastAPI?"}'

# 4. Hacer pregunta general (respuesta directa)
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "¿Cómo te llamas?"}'

# 5. Ver historial completo
curl http://localhost:8000/history

# 6. Ver detalle de una query específica
curl http://localhost:8000/history/1
```

### Verificar base de datos

```bash
# PostgreSQL
docker exec -it test-rag-postgres psql -U dev -d test_rag -c "SELECT * FROM query_history;"

# Qdrant
curl http://localhost:6335/collections/test_docs
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 📝 Variables de Entorno

### Archivo `.env` (desarrollo local)

```env
# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=dev
POSTGRES_PASSWORD=dev123
POSTGRES_DB=test_rag

# Qdrant
QDRANT_HOST=localhost
QDRANT_PORT=6333

# Ollama
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5:7b
```

### Archivo `.env.docker` (contenedores)

```env
# Database
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=dev
POSTGRES_PASSWORD=dev123
POSTGRES_DB=test_rag

# Qdrant
QDRANT_HOST=qdrant
QDRANT_PORT=6333

# Ollama (host)
OLLAMA_HOST=http://host.docker.internal:11434
OLLAMA_MODEL=qwen2.5:7b
```

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 🚧 Roadmap

### Features planeados

- [ ] **Autenticación JWT** - Login de usuarios
- [ ] **Rate limiting** - Prevenir abuso de API
- [ ] **Cache con Redis** - Respuestas frecuentes
- [ ] **Tests unitarios** - Pytest + coverage
- [ ] **Tests de integración** - End-to-end testing
- [ ] **CI/CD pipeline** - GitHub Actions
- [ ] **Deploy AWS/Railway** - Producción
- [ ] **Streaming responses** - Respuestas en tiempo real
- [ ] **Multi-tenancy** - Múltiples usuarios/organizaciones
- [ ] **Vector search filters** - Filtros por metadata
- [ ] **Document upload** - Subir PDFs/documentos
- [ ] **Export/Import** - Backup de vectores
- [ ] **Analytics dashboard** - Métricas de uso
- [ ] **Webhooks** - Notificaciones de eventos

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 📄 Licencia

MIT License

Copyright (c) 2026 Toni Benitez

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)

---

## 👤 Autor

**Toni Benitez**

📍 Barcelona, Catalonia, España  
🏢 Fullstack Developer + IA  
💻 Stack: JavaScript/TypeScript (Vue, React, NestJS) + Python (FastAPI, RAG)

### Stack del proyecto

**Backend:** FastAPI + PostgreSQL + Qdrant + Ollama  
**Frontend:** Vue 3 + Vite  
**Infrastructure:** Docker + WSL2  

---

**Proyecto creado como validación del entorno de desarrollo completo para desarrollo Fullstack + IA.**

[⬆️ Volver al inicio](#📑-tabla-de-contenidos)