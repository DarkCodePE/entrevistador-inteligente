# ADR-004: Infraestructura de IA/ML

**Estado:** Aprobado
**Fecha:** 2026-01-31
**Autores:** System Architecture Team
**Revisores:** CTO, ML Lead, Product Owner

---

## Contexto

"Entrevistador Inteligente" requiere capacidades avanzadas de IA/ML para cumplir su propuesta de valor core:

1. **CV Parsing**: Extraer datos estructurados de CVs en multiples formatos
2. **Semantic Matching**: Matching inteligente candidato-oferta laboral
3. **Interview Simulation**: Simulacion conversacional de entrevistas en espanol peruano
4. **Feedback Generation**: Analisis de respuestas y generacion de scoring/recomendaciones
5. **Speech Recognition**: (Fase 2) Transcripcion de audio con Whisper
6. **Video Analysis**: (Fase 3) Analisis de expresion facial y lenguaje corporal

### Restricciones del Proyecto

| Restriccion | Valor | Justificacion |
|-------------|-------|---------------|
| Presupuesto AI/ML Y1 | $30,000-50,000 | Startup en etapa pre-seed |
| Latencia maxima LLM | < 3 segundos | UX conversacional fluida |
| Costo por entrevista | < $0.15 | Margen viable en plan basico (S/29/mes) |
| Idioma principal | Espanol peruano | Mercado objetivo |
| Usuarios concurrentes Y1 | 500-1,000 | Proyeccion de crecimiento |

### Metricas de Exito AI/ML

| Metrica | Target MVP | Target Y1 |
|---------|------------|-----------|
| Precision CV parsing | > 85% | > 95% |
| Relevancia matching (user feedback) | > 70% | > 85% |
| Satisfaction entrevista simulada | > 4.0/5.0 | > 4.5/5.0 |
| Tiempo respuesta LLM (p95) | < 5s | < 3s |
| Uptime servicios AI | > 99% | > 99.5% |

---

## Decision

### 1. LLM Provider: Estrategia Hibrida OpenAI + Anthropic

**Decision:** Usar OpenAI GPT-4o-mini como modelo primario con GPT-4o para evaluacion, y Anthropic Claude como fallback.

#### Opciones Evaluadas

| Opcion | Pros | Contras | Costo/1M tokens |
|--------|------|---------|-----------------|
| **OpenAI GPT-4o** | Mejor razonamiento, herramientas maduras | Mas caro, rate limits | $5 input / $15 output |
| **OpenAI GPT-4o-mini** | Balance costo/calidad, rapido | Menor capacidad | $0.15 input / $0.60 output |
| **Anthropic Claude 3.5 Sonnet** | Excelente en espanol, contexto largo | Menor ecosistema | $3 input / $15 output |
| **Anthropic Claude 3.5 Haiku** | Muy rapido, economico | Menor razonamiento | $0.25 input / $1.25 output |
| **Google Gemini 1.5 Pro** | Contexto 1M tokens, multimodal | Menos maduro, menos consistente | $3.50 input / $10.50 output |
| **Open-source (Llama 3.1 70B)** | Sin costos API, control total | Infra costosa, latencia | $0 API + $2,000-5,000/mes GPU |
| **Mistral Large** | Buen espanol, europeo | Menor ecosistema | $4 input / $12 output |

#### Arquitectura LLM Seleccionada

```
+-----------------------------------------------------------------------+
|                    ARQUITECTURA LLM TIERED                             |
+-----------------------------------------------------------------------+

    Caso de Uso                 Modelo                  Costo Estimado
    ---------------            --------                ----------------

    [Preguntas Entrevista]  -> GPT-4o-mini             $0.008/entrevista
    [Follow-ups dinamicos]  -> GPT-4o-mini             $0.005/entrevista
    [Evaluacion respuestas] -> GPT-4o                  $0.025/entrevista
    [Feedback detallado]    -> GPT-4o                  $0.020/entrevista
                                                       ----------------
    TOTAL por entrevista (30 min, 15 preguntas):       ~$0.06

    [Fallback/redundancia]  -> Claude 3.5 Sonnet      (activar si OpenAI falla)

+-----------------------------------------------------------------------+
|                    FLUJO DE REDUNDANCIA                                |
+-----------------------------------------------------------------------+

    Request -> [Circuit Breaker] -> OpenAI (primary)
                     |
                     +-> (si falla/timeout) -> Anthropic (secondary)
                     |
                     +-> (si ambos fallan) -> Respuesta cached/generica
```

#### Estimacion de Costos LLM (Y1)

| Escenario | Entrevistas/mes | Costo LLM/mes | Costo LLM/ano |
|-----------|-----------------|---------------|---------------|
| Conservador | 5,000 | $300 | $3,600 |
| Base | 15,000 | $900 | $10,800 |
| Optimista | 40,000 | $2,400 | $28,800 |

**Justificacion:**
- GPT-4o-mini ofrece el mejor balance costo/calidad para flujo conversacional
- GPT-4o para evaluacion asegura feedback de alta calidad
- Dual-provider strategy evita vendor lock-in y mejora uptime
- Costo por entrevista ($0.06) permite margen saludable en plan basico

---

### 2. Embedding Model: OpenAI text-embedding-3-small

**Decision:** Usar OpenAI text-embedding-3-small (1536 dimensiones) para embeddings semanticos.

#### Opciones Evaluadas

| Modelo | Dimensiones | Costo/1M tokens | Performance (MTEB) | Latencia |
|--------|-------------|-----------------|-------------------|----------|
| **OpenAI text-embedding-3-small** | 1536 | $0.02 | 62.3% | 50ms |
| OpenAI text-embedding-3-large | 3072 | $0.13 | 64.6% | 80ms |
| Cohere embed-multilingual-v3.0 | 1024 | $0.10 | 64.0% | 60ms |
| Voyage-2 | 1024 | $0.10 | 63.5% | 55ms |
| sentence-transformers (local) | 768 | $0 + infra | 58.0% | 20ms local |

#### Arquitectura de Embeddings

```
+-----------------------------------------------------------------------+
|                    PIPELINE DE EMBEDDINGS                              |
+-----------------------------------------------------------------------+

    [CV Upload]                          [Job Description]
         |                                      |
         v                                      v
    +-----------+                         +-----------+
    | NLP Parser|                         | JD Parser |
    | (spaCy)   |                         | (spaCy)   |
    +-----------+                         +-----------+
         |                                      |
         v                                      v
    +-------------------+               +-------------------+
    | Chunking          |               | Chunking          |
    | - Skills          |               | - Requirements    |
    | - Experience      |               | - Skills needed   |
    | - Education       |               | - Company culture |
    +-------------------+               +-------------------+
         |                                      |
         v                                      v
    +-----------------------------------------------+
    |        OpenAI text-embedding-3-small          |
    |              (Batch Processing)               |
    +-----------------------------------------------+
         |                                      |
         v                                      v
    +-----------------------------------------------+
    |              Vector Database                  |
    |              (pgvector)                       |
    +-----------------------------------------------+
                          |
                          v
                 [Similarity Search]
                 [Matching Score 0-100]

+-----------------------------------------------------------------------+
|                    ESTRATEGIA DE CACHING                               |
+-----------------------------------------------------------------------+

    1. CV embeddings: Cache indefinido (hasta update de CV)
    2. Job embeddings: Cache 24h (refresh nocturno)
    3. Query embeddings: Cache 1h por usuario

    Ahorro estimado: 70-80% de llamadas a API
```

#### Estimacion de Costos Embeddings (Y1)

| Item | Volumen/mes | Tokens/item | Costo/mes |
|------|-------------|-------------|-----------|
| CVs nuevos | 5,000 | 2,000 | $0.20 |
| Jobs nuevos | 10,000 | 1,500 | $0.30 |
| Queries matching | 50,000 | 500 | $0.50 |
| **TOTAL** | - | - | **$1.00/mes** |

**Justificacion:**
- Costo practicamente negligible ($12/ano)
- Performance suficiente para matching semantico
- Misma API de OpenAI, menos complejidad operacional
- Caching agresivo reduce costos 70%+

---

### 3. Vector Database: PostgreSQL + pgvector

**Decision:** Usar PostgreSQL con extension pgvector para almacenamiento y busqueda de vectores.

#### Opciones Evaluadas

| Opcion | Costo/mes | Escalabilidad | Complejidad | Latencia (1M vectors) |
|--------|-----------|---------------|-------------|----------------------|
| **pgvector (self-hosted)** | $50-100 | Media-Alta | Baja | 20-50ms |
| Pinecone | $70-700 | Muy Alta | Muy Baja | 10-20ms |
| Qdrant (cloud) | $25-250 | Alta | Baja | 15-30ms |
| Weaviate (cloud) | $25-200 | Alta | Media | 15-30ms |
| Milvus (self-hosted) | $100-300 | Muy Alta | Alta | 10-20ms |
| ChromaDB (embedded) | $0 | Baja | Muy Baja | 5-10ms |

#### Arquitectura pgvector

```
+-----------------------------------------------------------------------+
|                    ARQUITECTURA PGVECTOR                               |
+-----------------------------------------------------------------------+

    PostgreSQL 16 + pgvector 0.6+
    |
    +-- Database: entrevistador_ai
        |
        +-- Table: cv_embeddings
        |   +-- id (UUID, PK)
        |   +-- user_id (UUID, FK)
        |   +-- embedding (vector(1536))
        |   +-- metadata (JSONB)
        |   +-- created_at (timestamp)
        |   +-- INDEX: ivfflat (embedding vector_cosine_ops) LISTS=100
        |
        +-- Table: job_embeddings
        |   +-- id (UUID, PK)
        |   +-- job_id (UUID, FK)
        |   +-- embedding (vector(1536))
        |   +-- metadata (JSONB)
        |   +-- created_at (timestamp)
        |   +-- INDEX: ivfflat (embedding vector_cosine_ops) LISTS=100
        |
        +-- Table: user_preference_embeddings
            +-- (similar structure)

+-----------------------------------------------------------------------+
|                    QUERY DE MATCHING                                   |
+-----------------------------------------------------------------------+

    -- Encontrar top 20 jobs para un CV
    SELECT
        j.id,
        j.title,
        j.company,
        1 - (ce.embedding <=> je.embedding) AS similarity_score
    FROM cv_embeddings ce
    CROSS JOIN LATERAL (
        SELECT je.*, j.*
        FROM job_embeddings je
        JOIN jobs j ON je.job_id = j.id
        WHERE j.status = 'active'
        ORDER BY ce.embedding <=> je.embedding
        LIMIT 20
    ) je
    WHERE ce.user_id = $1;

    -- Performance: ~20-50ms para 100K vectores con IVFFLAT index
```

#### Roadmap de Escalabilidad

| Fase | Vectores | Solucion | Costo/mes |
|------|----------|----------|-----------|
| MVP | < 50K | pgvector single node | $50 |
| Growth | 50K-500K | pgvector + read replicas | $150 |
| Scale | 500K-2M | pgvector + HNSW index | $300 |
| Enterprise | > 2M | Migrar a Pinecone/Qdrant | $500+ |

**Justificacion:**
- Consolidacion: misma infra que DB principal (PostgreSQL)
- Costo inicial minimo ($50/mes incluido en DB principal)
- Performance suficiente para Y1 (hasta 500K vectores)
- Migracion a servicio dedicado si es necesario
- Sin vendor lock-in (estandar SQL + extension)

---

### 4. CV Parsing Pipeline: Azure Form Recognizer + spaCy

**Decision:** Usar Azure Form Recognizer para OCR y spaCy para NLP/NER.

#### Opciones Evaluadas (OCR)

| Servicio | Precision | Costo/1K docs | Idiomas | Latencia |
|----------|-----------|---------------|---------|----------|
| **Azure Form Recognizer** | 95%+ | $1.50 | 164 | 2-5s |
| Google Document AI | 94%+ | $1.50 | 100+ | 2-4s |
| Amazon Textract | 93%+ | $1.50 | 25 | 3-6s |
| Tesseract (OSS) | 85%+ | $0 + infra | 100+ | 1-3s |
| Affinda | 90%+ | $0.10-0.50 | 50+ | 2-5s |

#### Opciones Evaluadas (NLP/NER)

| Libreria | Performance | Espanol | Costo | Customizacion |
|----------|-------------|---------|-------|---------------|
| **spaCy** | Muy Bueno | es_core_news_lg | OSS | Alta |
| Stanza | Bueno | Si | OSS | Media |
| Flair | Muy Bueno | Si | OSS | Alta |
| OpenAI GPT | Excelente | Si | $0.03/CV | Baja |
| AWS Comprehend | Bueno | Si | $0.01/unit | Baja |

#### Pipeline de CV Parsing

```
+-----------------------------------------------------------------------+
|                    CV PARSING PIPELINE                                 |
+-----------------------------------------------------------------------+

    [CV Upload]
         |
         v
    +-------------------+
    | Format Detection  |
    | - PDF             |
    | - Word (.docx)    |
    | - Image (jpg/png) |
    | - LinkedIn export |
    +-------------------+
         |
         +-- PDF/Image -----> [Azure Form Recognizer]
         |                           |
         +-- Word ---------> [python-docx]
         |                           |
         +-- LinkedIn -----> [LinkedIn Parser]
         |                           |
         v                           v
    +-----------------------------------------------+
    |              Raw Text Extraction               |
    +-----------------------------------------------+
         |
         v
    +-----------------------------------------------+
    |              spaCy NLP Pipeline                |
    |                                               |
    | 1. Tokenization (es_core_news_lg)             |
    | 2. POS Tagging                                |
    | 3. Named Entity Recognition                   |
    |    - PERSON, ORG, DATE, LOC, GPE             |
    | 4. Custom NER (fine-tuned for CVs)           |
    |    - SKILL, EDUCATION, TITLE, COMPANY        |
    +-----------------------------------------------+
         |
         v
    +-----------------------------------------------+
    |              Section Detection                 |
    |                                               |
    | - Datos Personales                            |
    | - Experiencia Laboral                         |
    | - Educacion                                   |
    | - Habilidades/Skills                          |
    | - Idiomas                                     |
    | - Certificaciones                             |
    | - Referencias                                 |
    +-----------------------------------------------+
         |
         v
    +-----------------------------------------------+
    |              Structured Output (JSON)          |
    +-----------------------------------------------+
    {
      "personal_info": {
        "name": "Juan Perez",
        "email": "juan@email.com",
        "phone": "+51 999 888 777",
        "location": "Lima, Peru",
        "linkedin": "linkedin.com/in/juanperez"
      },
      "experience": [
        {
          "title": "Software Engineer",
          "company": "Tech Company",
          "start_date": "2022-01",
          "end_date": "present",
          "description": "...",
          "skills_used": ["Python", "AWS", "Docker"]
        }
      ],
      "education": [...],
      "skills": {
        "technical": ["Python", "JavaScript", "SQL"],
        "soft": ["Liderazgo", "Comunicacion"],
        "languages": [{"lang": "Ingles", "level": "Avanzado"}]
      },
      "certifications": [...],
      "parsed_at": "2026-01-31T10:00:00Z",
      "confidence_score": 0.92
    }
```

#### Estimacion de Costos CV Parsing (Y1)

| Componente | Volumen/mes | Costo/unidad | Costo/mes |
|------------|-------------|--------------|-----------|
| Azure Form Recognizer | 5,000 CVs | $0.0015 | $7.50 |
| spaCy (self-hosted) | 5,000 CVs | $0 | $0 |
| Compute (CV processor) | - | - | $50 |
| **TOTAL** | - | - | **$57.50/mes** |

**Justificacion:**
- Azure Form Recognizer: Mejor precision en documentos multi-formato
- spaCy: OSS, excelente para espanol, customizable
- Costo total ~$0.01/CV (muy bajo)
- Pipeline modular permite optimizaciones futuras

---

### 5. Speech Recognition (Fase 2): OpenAI Whisper

**Decision:** Usar OpenAI Whisper API para transcripcion de audio en entrevistas.

#### Opciones Evaluadas

| Servicio | WER Espanol | Costo/hora | Latencia | Acentos LATAM |
|----------|-------------|------------|----------|---------------|
| **OpenAI Whisper API** | 3-5% | $0.36 | Near RT | Muy Bueno |
| Whisper (self-hosted) | 3-5% | $0 + GPU | RT posible | Muy Bueno |
| Google Speech-to-Text | 4-6% | $0.48 | RT | Bueno |
| AWS Transcribe | 5-7% | $0.48 | RT | Medio |
| Azure Speech | 4-6% | $0.40 | RT | Bueno |
| AssemblyAI | 4-6% | $0.37 | RT | Bueno |

#### Arquitectura Speech Recognition

```
+-----------------------------------------------------------------------+
|                    SPEECH RECOGNITION PIPELINE                         |
+-----------------------------------------------------------------------+

    [Usuario habla]
         |
         v
    +-------------------+
    | Audio Capture     |
    | - WebRTC          |
    | - MediaRecorder   |
    | - 16kHz, mono     |
    +-------------------+
         |
         v
    +-------------------+
    | Audio Preprocessing|
    | - Noise reduction |
    | - Normalization   |
    | - VAD (silences)  |
    +-------------------+
         |
         v
    +-----------------------------------------------+
    |              Whisper API                       |
    |                                               |
    | Model: whisper-1                              |
    | Language: es (Spanish)                        |
    | Response format: verbose_json                 |
    | Temperature: 0                                |
    |                                               |
    | Output:                                       |
    | - Transcription text                          |
    | - Word-level timestamps                       |
    | - Confidence scores                           |
    | - Language detection                          |
    +-----------------------------------------------+
         |
         v
    +-------------------+
    | Post-processing   |
    | - Punctuation     |
    | - Capitalization  |
    | - Peru vocab fix  |
    +-------------------+
         |
         v
    [Transcription -> LLM for evaluation]

+-----------------------------------------------------------------------+
|                    CONSIDERACIONES PERU                                |
+-----------------------------------------------------------------------+

    Retos especificos del espanol peruano:

    1. Vocabulario local (peruanismos):
       - "chamba" -> trabajo
       - "chela" -> cerveza (contexto informal)
       - "pata" -> amigo

    2. Acentos regionales:
       - Costeno (Lima)
       - Andino (Sierra)
       - Amazonico (Selva)

    3. Velocidad de habla: 120-150 palabras/min (mas rapido que Espana)

    4. Code-switching: Mezcla espanol-ingles en tech

    Mitigacion:
    - Fine-tuning con dataset peruano (1000+ horas) en Fase 3
    - Vocabulary expansion para terminos locales
    - Post-processing rules para correccion
```

#### Estimacion de Costos Speech (Fase 2)

| Escenario | Entrevistas audio/mes | Duracion promedio | Costo/mes |
|-----------|----------------------|-------------------|-----------|
| Conservador | 2,000 | 20 min | $240 |
| Base | 5,000 | 20 min | $600 |
| Optimista | 10,000 | 20 min | $1,200 |

**Justificacion:**
- Whisper: Mejor performance en espanol LATAM
- Costo razonable ($0.006/min)
- API simple, sin necesidad de infra GPU inicial
- Opcion de self-hosting en el futuro para reducir costos

---

### 6. Video Analysis (Fase 3): MediaPipe + Custom Models

**Decision:** Usar Google MediaPipe para deteccion facial y modelos custom para analisis de expresiones.

#### Capacidades Planificadas

| Feature | Tecnologia | Complejidad | Valor Usuario |
|---------|------------|-------------|---------------|
| Eye contact tracking | MediaPipe Face Mesh | Media | Alto |
| Head pose estimation | MediaPipe | Media | Medio |
| Facial expressions | FER + custom | Alta | Alto |
| Body posture | MediaPipe Pose | Media | Medio |
| Gesture detection | MediaPipe Hands | Alta | Bajo |
| Background analysis | Custom CV | Baja | Bajo |

#### Arquitectura Video Analysis

```
+-----------------------------------------------------------------------+
|                    VIDEO ANALYSIS PIPELINE (FASE 3)                    |
+-----------------------------------------------------------------------+

    [Video Stream]
         |
         v
    +-------------------+
    | Frame Extraction  |
    | - 10 FPS          |
    | - 720p max        |
    | - Face crop       |
    +-------------------+
         |
         +---> [MediaPipe Face Mesh] ---> Eye contact score
         |                          ---> Head pose (pitch, yaw, roll)
         |
         +---> [FER Model] -----------> Emotion classification
         |                          ---> Confidence detection
         |
         +---> [MediaPipe Pose] -----> Posture analysis
         |                          ---> Hand gestures
         |
         v
    +-----------------------------------------------+
    |              Aggregation Service               |
    |                                               |
    | Metrics per minute:                           |
    | - Eye contact %: tiempo mirando a camara      |
    | - Posture score: 0-100                        |
    | - Confidence level: nervioso/neutral/confiado |
    | - Engagement: expresiones positivas           |
    +-----------------------------------------------+
         |
         v
    +-----------------------------------------------+
    |              Feedback Generation               |
    |                                               |
    | "Tu contacto visual fue del 65%, intenta      |
    |  mirar mas a la camara. Tu postura fue        |
    |  buena. Detectamos algo de nerviosismo al     |
    |  inicio que mejoro con el tiempo."            |
    +-----------------------------------------------+

+-----------------------------------------------------------------------+
|                    CONSIDERACIONES TECNICAS                            |
+-----------------------------------------------------------------------+

    1. Procesamiento:
       - Edge (browser): MediaPipe en WebAssembly
       - Backend: Solo para modelos custom

    2. Privacidad:
       - Video NO se almacena por default
       - Solo metricas agregadas
       - Opt-in para review manual

    3. Recursos:
       - MediaPipe: ~30ms/frame en browser moderno
       - GPU recomendada para FER (pero funciona sin)
```

#### Estimacion de Costos Video Analysis (Fase 3)

| Componente | Costo Desarrollo | Costo Operacion/mes |
|------------|------------------|---------------------|
| MediaPipe integration | $5,000 | $0 (client-side) |
| FER model training | $10,000 | $200 (inference) |
| Backend processing | $5,000 | $300 (compute) |
| **TOTAL** | **$20,000** | **$500/mes** |

**Justificacion:**
- Procesamiento en edge reduce costos de servidor
- MediaPipe es OSS y muy optimizado
- Feature premium justifica inversion
- Privacidad by design (no almacenar video)

---

### 7. Caching Strategy: Multi-Layer Cache

**Decision:** Implementar cache de 3 capas para reducir costos LLM 50-70%.

#### Arquitectura de Cache

```
+-----------------------------------------------------------------------+
|                    MULTI-LAYER CACHE ARCHITECTURE                      |
+-----------------------------------------------------------------------+

    [Request]
         |
         v
    +-------------------+     HIT
    | L1: In-Memory     |-------------> [Response]
    | (Redis)           |
    | TTL: 1 hora       |
    +-------------------+
         | MISS
         v
    +-------------------+     HIT
    | L2: Semantic Cache|-------------> [Response]
    | (pgvector)        |
    | TTL: 24 horas     |
    +-------------------+
         | MISS
         v
    +-------------------+     HIT
    | L3: Response DB   |-------------> [Response]
    | (PostgreSQL)      |
    | TTL: 7 dias       |
    +-------------------+
         | MISS
         v
    +-------------------+
    | LLM API Call      |
    | (OpenAI/Anthropic)|
    +-------------------+
         |
         v
    [Cache Response in L1, L2, L3]

+-----------------------------------------------------------------------+
|                    CACHE STRATEGIES BY USE CASE                        |
+-----------------------------------------------------------------------+

    1. PREGUNTAS DE ENTREVISTA
       Cache Key: hash(job_type + difficulty + category)

       Ejemplo:
       - "Pregunta tecnica para Backend Developer, nivel senior"
       - Si existe pregunta similar (cosine > 0.95), reusar
       - Si no, generar nueva y cachear

       Hit Rate Esperado: 60-70%

    2. FEEDBACK DE RESPUESTAS
       Cache Key: hash(question + response_embedding)

       Buscar respuestas similares (cosine > 0.90)
       Si existe, adaptar feedback existente
       Si no, generar nuevo

       Hit Rate Esperado: 40-50%

    3. CV ANALYSIS
       Cache Key: CV embedding

       Si CV muy similar existe (cosine > 0.98), reusar estructura
       Solo regenerar secciones con diferencias

       Hit Rate Esperado: 30-40%

+-----------------------------------------------------------------------+
|                    SEMANTIC CACHE IMPLEMENTATION                       |
+-----------------------------------------------------------------------+

    -- Tabla de cache semantico
    CREATE TABLE semantic_cache (
        id UUID PRIMARY KEY,
        cache_type VARCHAR(50),
        query_embedding vector(1536),
        query_text TEXT,
        response_text TEXT,
        metadata JSONB,
        hit_count INTEGER DEFAULT 0,
        created_at TIMESTAMP,
        expires_at TIMESTAMP,
        INDEX idx_cache_embedding (query_embedding vector_cosine_ops)
    );

    -- Buscar en cache
    SELECT response_text,
           1 - (query_embedding <=> $1) AS similarity
    FROM semantic_cache
    WHERE cache_type = $2
      AND expires_at > NOW()
      AND 1 - (query_embedding <=> $1) > 0.95
    ORDER BY similarity DESC
    LIMIT 1;
```

#### Ahorro Estimado con Cache

| Capa | Hit Rate | Ahorro LLM |
|------|----------|------------|
| L1 (Redis) | 20% | $200/mes |
| L2 (Semantic) | 35% | $350/mes |
| L3 (DB) | 15% | $150/mes |
| **TOTAL** | **70%** | **$700/mes** |

---

### 8. ML Pipeline: Vertex AI (Google Cloud)

**Decision:** Usar Google Cloud Vertex AI para entrenamiento y despliegue de modelos custom.

#### Justificacion vs Alternativas

| Plataforma | Pros | Contras | Costo Base |
|------------|------|---------|------------|
| **Vertex AI** | Integrado con GCP, AutoML, buen pricing | Vendor lock-in | $50/mes |
| SageMaker | Completo, enterprise | Complejo, caro | $100/mes |
| Azure ML | Buenas herramientas | Menos flexible | $75/mes |
| Self-hosted (MLflow) | Control total, OSS | Mas trabajo ops | $200/mes (infra) |

#### Modelos Custom Planificados

| Modelo | Proposito | Timeline | Costo Training |
|--------|-----------|----------|----------------|
| CV Skill Extractor | NER para skills Peru | Q2 2026 | $500 |
| Interview Scorer | Scoring respuestas | Q3 2026 | $1,500 |
| Peru ASR Fine-tune | Whisper peruano | Q4 2026 | $3,000 |
| Emotion Classifier | FER personalizado | Q1 2027 | $2,000 |

---

## Resumen de Costos AI/ML

### Costos Mensuales Proyectados

| Componente | MVP (M1-M6) | Growth (M7-M12) | Scale (Y2) |
|------------|-------------|-----------------|------------|
| LLM (OpenAI/Anthropic) | $300 | $900 | $2,500 |
| Embeddings | $1 | $5 | $20 |
| CV Parsing (Azure) | $10 | $50 | $150 |
| Vector DB (pgvector) | $50 | $100 | $300 |
| Speech (Whisper) | $0 | $300 | $1,000 |
| Video Analysis | $0 | $0 | $500 |
| ML Platform (Vertex) | $50 | $100 | $300 |
| Cache (Redis) | $30 | $50 | $100 |
| **TOTAL** | **$441/mes** | **$1,505/mes** | **$4,870/mes** |

### Costo por Usuario Activo

| Fase | MAU | Costo AI/ML | Costo/MAU |
|------|-----|-------------|-----------|
| MVP | 2,000 | $441 | $0.22 |
| Growth | 12,000 | $1,505 | $0.13 |
| Scale | 40,000 | $4,870 | $0.12 |

---

## Consecuencias

### Positivas

1. **Costo optimizado**: $0.06/entrevista permite margenes saludables
2. **Latencia controlada**: < 3s para respuestas LLM
3. **Escalabilidad**: Arquitectura soporta 40K+ MAU sin cambios mayores
4. **Sin vendor lock-in**: Multi-provider strategy para LLM y vector DB
5. **Localizacion**: Pipeline preparado para espanol peruano

### Negativas

1. **Complejidad operacional**: Multiples servicios que mantener
2. **Dependencia de terceros**: OpenAI, Anthropic, Azure
3. **Costos variables**: LLM puede escalar con uso
4. **Latencia de cache miss**: Primera vez puede ser lento

### Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| OpenAI rate limits | Media | Alto | Anthropic fallback, request queuing |
| Costos LLM disparan | Media | Alto | Caching agresivo, tiered models |
| Calidad espanol peruano | Alta | Medio | Fine-tuning, post-processing rules |
| pgvector performance | Baja | Alto | Migracion a Pinecone si necesario |
| Data breach | Baja | Critico | Encryption, compliance, audits |

---

## Metricas de Monitoreo

```
+-----------------------------------------------------------------------+
|                    DASHBOARD AI/ML METRICS                             |
+-----------------------------------------------------------------------+

    COST METRICS (Daily)
    +---------------------------+
    | LLM Cost Today:    $32.50 |
    | Cost/Interview:    $0.058 |
    | Cache Hit Rate:    68%    |
    | Budget Remaining:  85%    |
    +---------------------------+

    PERFORMANCE METRICS (Real-time)
    +---------------------------+
    | LLM Latency p50:   1.2s   |
    | LLM Latency p95:   2.8s   |
    | Embedding Latency: 45ms   |
    | Error Rate:        0.1%   |
    +---------------------------+

    QUALITY METRICS (Weekly)
    +---------------------------+
    | CV Parse Accuracy: 91%    |
    | Matching Relevance: 78%   |
    | Interview NPS:     4.2    |
    | Feedback Quality:  4.1    |
    +---------------------------+

    USAGE METRICS (Daily)
    +---------------------------+
    | Interviews Today:  450    |
    | CVs Parsed:        120    |
    | Matching Queries:  2,300  |
    | Active Sessions:   85     |
    +---------------------------+
```

---

## Implementacion

### Fase 1: MVP (M1-M3)

- [x] Setup OpenAI API integration
- [ ] Implement basic interview flow with GPT-4o-mini
- [ ] CV parsing with Azure Form Recognizer + spaCy
- [ ] Basic embeddings with text-embedding-3-small
- [ ] pgvector setup in PostgreSQL
- [ ] L1 cache (Redis) implementation

### Fase 2: Optimization (M4-M6)

- [ ] Semantic cache (L2) implementation
- [ ] Anthropic fallback integration
- [ ] Cache warming for common queries
- [ ] Performance monitoring dashboard
- [ ] Cost alerts and budgets

### Fase 3: Speech (M7-M9)

- [ ] Whisper API integration
- [ ] Audio preprocessing pipeline
- [ ] Real-time transcription UI
- [ ] Peru vocabulary enhancements

### Fase 4: Video (M10-M12)

- [ ] MediaPipe integration
- [ ] Eye contact tracking MVP
- [ ] Basic emotion detection
- [ ] Video feedback generation

---

## Referencias

1. OpenAI API Documentation: https://platform.openai.com/docs
2. Anthropic Claude API: https://docs.anthropic.com
3. pgvector Documentation: https://github.com/pgvector/pgvector
4. Azure Form Recognizer: https://docs.microsoft.com/azure/form-recognizer
5. spaCy Spanish Models: https://spacy.io/models/es
6. OpenAI Whisper: https://platform.openai.com/docs/guides/speech-to-text
7. MediaPipe: https://developers.google.com/mediapipe
8. Google Vertex AI: https://cloud.google.com/vertex-ai

---

## Historial de Cambios

| Version | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-31 | System Architecture Team | Documento inicial |

---

*ADR-004: Aprobado por CTO y ML Lead - 2026-01-31*
