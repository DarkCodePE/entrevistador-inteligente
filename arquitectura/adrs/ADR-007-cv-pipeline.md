# ADR-007: Pipeline de Procesamiento de CV

| Metadata | Value |
|----------|-------|
| Estado | Propuesto |
| Fecha | 2026-01-31 |
| Autores | Equipo de Arquitectura |
| Revisores | - |
| Relacionado con | ADR-005 (Almacenamiento), ADR-006 (AI/ML) |

## Contexto

"Entrevistador Inteligente" requiere un pipeline robusto para procesar CVs en multiples formatos y extraer informacion estructurada. El sistema debe soportar:

- Formatos diversos: PDF, DOCX, DOC, imagenes (fotos de CV)
- Documentos escaneados que requieren OCR
- Extraccion de entidades clave (nombre, contacto, experiencia, educacion, skills)
- Normalizacion de terminologia tecnica
- Generacion de embeddings para matching semantico
- Scoring de compatibilidad con sistemas ATS
- Sugerencias de optimizacion para el candidato

### Flujo del Pipeline

```
+------------------+     +---------------+     +------------------+
|    Upload        |     |    OCR        |     |    Parsing       |
| PDF/Word/Image   | --> | (si imagen)   | --> | Extraer secciones|
+------------------+     +---------------+     +------------------+
                                                       |
                                                       v
+------------------+     +---------------+     +------------------+
|   Sugerencias    |     |   ATS Score   |     |    Embedding     |
|   de Mejora      | <-- |   Calculo     | <-- |  Vectorizacion   |
+------------------+     +---------------+     +------------------+
                                ^
                                |
                         +---------------+
                         |     NLP       |
                         | Entidades +   |
                         | Skills        |
                         +---------------+
```

### Requisitos de Performance

| Metrica | Target |
|---------|--------|
| Tiempo de procesamiento completo | < 30 segundos |
| Accuracy en extraccion de datos | > 95% |
| Tamano maximo de documento | 10 paginas |
| Formatos soportados | PDF, DOCX, DOC, PNG, JPG, JPEG |

## Decisiones

### Decision 1: Parser de CV

**Opciones evaluadas:**

| Opcion | Pros | Contras | Costo |
|--------|------|---------|-------|
| **Affinda** | Alta precision (98%+), API lista, soporta 100+ idiomas | Costo por documento ($0.10-0.50/CV), vendor lock-in | Alto |
| **AWS Textract** | Integracion AWS, buena para formularios | Requiere post-procesamiento, menos especializado en CVs | Medio |
| **resume-parser (OSS)** | Gratuito, control total, personalizable | Menor precision inicial, requiere mantenimiento | Bajo |
| **Hybrid (OSS + LLM)** | Flexibilidad, precision mejorable, control total | Complejidad de implementacion | Medio |

**Decision: Hybrid (resume-parser + LLM fallback)**

**Justificacion:**
- Control total sobre el pipeline sin dependencias de terceros
- Costo predecible (solo llamadas a LLM cuando sea necesario)
- Posibilidad de mejorar iterativamente el modelo
- El LLM (Claude/GPT) actua como fallback para casos complejos

**Arquitectura del parser:**

```
                    +------------------+
                    |  Documento Raw   |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | resume-parser    |
                    | (extraccion      |
                    |  inicial)        |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
              v                             v
     +----------------+            +----------------+
     | Confidence     |            | Confidence     |
     | >= 85%         |            | < 85%          |
     +-------+--------+            +-------+--------+
             |                             |
             v                             v
     +----------------+            +----------------+
     | Usar resultado |            | LLM Fallback   |
     | directo        |            | (Claude/GPT)   |
     +----------------+            +----------------+
```

---

### Decision 2: Motor de OCR

**Opciones evaluadas:**

| Opcion | Pros | Contras | Costo |
|--------|------|---------|-------|
| **Tesseract** | Open source, sin costo, amplio soporte | Precision variable, requiere pre-procesamiento | Gratuito |
| **Google Vision** | Alta precision (99%+), handwriting support | Costo por imagen, latencia de red | $1.50/1000 imgs |
| **Azure OCR** | Buena precision, integracion Azure | Costo similar a Google, menos features | $1.00/1000 imgs |
| **Tesseract + Pre-processing** | Gratuito con precision mejorada | Complejidad adicional | Gratuito |

**Decision: Tesseract con pipeline de pre-procesamiento**

**Justificacion:**
- Costo cero para OCR (importante en fase inicial)
- Pipeline de pre-procesamiento mejora la precision significativamente
- Fallback a Google Vision para documentos problematicos
- Control total sobre el procesamiento

**Pipeline de pre-procesamiento:**

```python
# Pseudocodigo del pipeline de pre-procesamiento
def preprocess_for_ocr(image):
    # 1. Deskew - corregir inclinacion
    image = deskew(image)

    # 2. Binarizacion adaptativa
    image = adaptive_threshold(image)

    # 3. Eliminacion de ruido
    image = denoise(image)

    # 4. Mejora de contraste
    image = enhance_contrast(image)

    # 5. Escalado optimo (300 DPI)
    image = resize_to_optimal_dpi(image)

    return image
```

---

### Decision 3: Motor de NLP

**Opciones evaluadas:**

| Opcion | Pros | Contras | Costo |
|--------|------|---------|-------|
| **spaCy** | Rapido, eficiente, bueno para NER | Modelos pre-entrenados limitados para CVs | Gratuito |
| **Hugging Face** | Modelos especializados, comunidad activa | Requiere GPU para mejor performance | Gratuito/GPU |
| **OpenAI API** | Alta precision, zero-shot, multilingue | Costo por token, latencia | $0.002/1K tokens |
| **spaCy + Custom NER** | Control total, optimizado para dominio | Requiere datos de entrenamiento | Gratuito |

**Decision: spaCy con modelos custom + OpenAI fallback**

**Justificacion:**
- spaCy es extremadamente rapido para NER local
- Entrenamiento de modelos custom para skills tecnicos
- OpenAI como fallback para contextos ambiguos
- Balance optimo entre costo y precision

**Entidades a extraer:**

```yaml
entities:
  - PERSON_NAME      # Nombre del candidato
  - EMAIL            # Correo electronico
  - PHONE            # Telefono
  - LOCATION         # Ubicacion
  - COMPANY          # Empresas donde trabajo
  - JOB_TITLE        # Titulos de puesto
  - EDUCATION        # Instituciones educativas
  - DEGREE           # Titulos academicos
  - SKILL_TECHNICAL  # Habilidades tecnicas
  - SKILL_SOFT       # Habilidades blandas
  - LANGUAGE         # Idiomas
  - CERTIFICATION    # Certificaciones
  - DATE_RANGE       # Periodos de tiempo
```

---

### Decision 4: Modelo de Embeddings

**Opciones evaluadas:**

| Opcion | Dimensiones | Costo | Latencia | Precision |
|--------|-------------|-------|----------|-----------|
| **OpenAI text-embedding-3-small** | 1536 | $0.02/1M tokens | ~100ms | Alta |
| **OpenAI text-embedding-3-large** | 3072 | $0.13/1M tokens | ~150ms | Muy Alta |
| **sentence-transformers (local)** | 384-768 | Gratuito | ~50ms | Media-Alta |
| **Cohere embed-v3** | 1024 | $0.10/1M tokens | ~100ms | Alta |

**Decision: OpenAI text-embedding-3-small**

**Justificacion:**
- Excelente balance precision/costo
- 1536 dimensiones suficientes para matching semantico
- Consistencia con otros componentes del sistema
- Facil integracion y mantenimiento

**Estrategia de embedding:**

```yaml
embedding_strategy:
  # Embeddings separados por seccion para matching granular
  sections:
    - skills_embedding      # Solo skills
    - experience_embedding  # Experiencia laboral
    - education_embedding   # Educacion
    - full_cv_embedding     # CV completo

  # Normalizacion antes de embedding
  preprocessing:
    - lowercase
    - remove_stopwords
    - skill_normalization
    - acronym_expansion
```

---

### Decision 5: Modelo de Procesamiento

**Opciones evaluadas:**

| Opcion | Pros | Contras | Complejidad |
|--------|------|---------|-------------|
| **Sincrono** | Simple, facil debugging | Bloquea UI, timeout en archivos grandes | Baja |
| **Async (Queue)** | No bloquea, escalable, retry | Complejidad, necesita workers | Media |
| **Hybrid** | Rapido para archivos simples, robusto para complejos | Logica condicional | Media |

**Decision: Async con Queue (Bull/BullMQ)**

**Justificacion:**
- Permite procesar CVs pesados sin bloquear
- Escalabilidad horizontal con workers
- Reintentos automaticos en fallos
- Visibilidad del estado del procesamiento

**Arquitectura de procesamiento:**

```
+-------------+     +------------------+     +------------------+
|   Cliente   |     |   API Gateway    |     |   Redis Queue    |
|   Upload CV | --> |   /api/cv/upload | --> |   cv-processing  |
+-------------+     +------------------+     +--------+---------+
                                                      |
                           +--------------------------|
                           |                          |
                           v                          v
                    +-------------+            +-------------+
                    |  Worker 1   |            |  Worker 2   |
                    |  (OCR+Parse)|            |  (NLP+Embed)|
                    +------+------+            +------+------+
                           |                          |
                           v                          v
                    +------------------------------------------+
                    |              PostgreSQL                   |
                    |  (Resultados + pgvector para embeddings)  |
                    +------------------------------------------+
                                      |
                                      v
                    +------------------------------------------+
                    |           WebSocket Notification          |
                    |         (Notifica al cliente)             |
                    +------------------------------------------+
```

---

## Normalizacion de Skills

### Diccionario de Normalizacion

```json
{
  "skill_aliases": {
    "javascript": ["js", "ecmascript", "es6", "es2015", "es2020"],
    "typescript": ["ts"],
    "python": ["py", "python3", "python2"],
    "react": ["reactjs", "react.js", "react js"],
    "node.js": ["node", "nodejs"],
    "postgresql": ["postgres", "psql", "pg"],
    "mongodb": ["mongo"],
    "amazon web services": ["aws", "amazon aws"],
    "google cloud platform": ["gcp", "google cloud"],
    "kubernetes": ["k8s"],
    "docker": ["docker-compose", "dockerfile"],
    "machine learning": ["ml"],
    "artificial intelligence": ["ai"],
    "continuous integration": ["ci", "ci/cd"],
    "continuous deployment": ["cd"]
  },
  "skill_categories": {
    "programming_languages": ["javascript", "python", "java", "go", "rust"],
    "frameworks": ["react", "angular", "vue", "django", "fastapi"],
    "databases": ["postgresql", "mongodb", "mysql", "redis"],
    "cloud": ["aws", "gcp", "azure"],
    "devops": ["docker", "kubernetes", "terraform", "jenkins"]
  }
}
```

---

## ATS Score Algorithm

### Factores de Puntuacion

```yaml
ats_scoring:
  # Peso de cada factor (total = 100)
  factors:
    keyword_match: 35        # Match con job description
    formatting: 20           # Formato limpio, parseable
    section_completeness: 15 # Secciones requeridas presentes
    experience_relevance: 15 # Experiencia relevante al puesto
    education_match: 10      # Educacion requerida
    certifications: 5        # Certificaciones relevantes

  # Penalizaciones
  penalties:
    - tables_detected: -10   # Tablas dificultan parsing
    - images_in_cv: -5       # Imagenes no parseables
    - missing_contact: -15   # Sin info de contacto
    - no_dates: -10          # Experiencia sin fechas
    - inconsistent_format: -5

  # Bonus
  bonuses:
    - quantified_achievements: +5  # Logros con numeros
    - linkedin_url: +2
    - github_url: +3
    - portfolio_url: +2
```

### Sugerencias de Mejora

```yaml
improvement_suggestions:
  categories:
    - formatting:
        - "Evitar uso de tablas para mejor compatibilidad ATS"
        - "Usar formato cronologico inverso"
        - "Incluir fechas en todas las experiencias"

    - content:
        - "Agregar logros cuantificables (%, $, numeros)"
        - "Incluir palabras clave del puesto objetivo"
        - "Expandir seccion de skills tecnicos"

    - missing_sections:
        - "Agregar resumen profesional"
        - "Incluir informacion de contacto completa"
        - "Agregar seccion de certificaciones"

    - optimization:
        - "Reducir longitud a 2 paginas maximo"
        - "Usar verbos de accion al inicio de bullets"
        - "Remover experiencia no relevante (>10 anos)"
```

---

## Modelo de Datos

### Schema de CV Procesado

```typescript
interface ProcessedCV {
  id: string;
  userId: string;
  originalFile: {
    name: string;
    type: 'pdf' | 'docx' | 'doc' | 'image';
    size: number;
    url: string;
  };

  extractedData: {
    personalInfo: {
      fullName: string;
      email: string;
      phone: string;
      location: string;
      linkedinUrl?: string;
      githubUrl?: string;
      portfolioUrl?: string;
    };

    summary?: string;

    experience: Array<{
      company: string;
      title: string;
      location?: string;
      startDate: string;
      endDate?: string;
      current: boolean;
      description: string;
      achievements: string[];
    }>;

    education: Array<{
      institution: string;
      degree: string;
      field: string;
      startDate?: string;
      endDate?: string;
      gpa?: number;
    }>;

    skills: {
      technical: Array<{
        name: string;
        normalizedName: string;
        category: string;
        confidence: number;
      }>;
      soft: string[];
      languages: Array<{
        language: string;
        proficiency: string;
      }>;
    };

    certifications: Array<{
      name: string;
      issuer: string;
      date?: string;
      expirationDate?: string;
      credentialId?: string;
    }>;
  };

  embeddings: {
    fullCv: number[];      // 1536 dims
    skills: number[];      // 1536 dims
    experience: number[];  // 1536 dims
  };

  atsScore: {
    overall: number;       // 0-100
    breakdown: {
      keywordMatch: number;
      formatting: number;
      completeness: number;
      relevance: number;
    };
    penalties: string[];
    bonuses: string[];
  };

  suggestions: Array<{
    category: string;
    priority: 'high' | 'medium' | 'low';
    message: string;
    affectedSection?: string;
  }>;

  metadata: {
    processingTime: number;  // ms
    ocrApplied: boolean;
    llmFallbackUsed: boolean;
    confidence: number;      // 0-1
    version: string;
  };

  createdAt: Date;
  updatedAt: Date;
}
```

---

## Diagrama de Secuencia

```
Usuario          Frontend         API           Queue          Worker         DB
   |                |              |              |               |            |
   |-- Upload CV -->|              |              |               |            |
   |                |-- POST /cv ->|              |               |            |
   |                |              |-- Validate ->|               |            |
   |                |              |-- Store raw -|---------------|----------->|
   |                |              |-- Enqueue -->|               |            |
   |                |<- 202 + jobId|              |               |            |
   |<- "Processing" |              |              |               |            |
   |                |              |              |-- Pick job -->|            |
   |                |              |              |               |-- OCR? --->|
   |                |              |              |               |-- Parse -->|
   |                |              |              |               |-- NLP ---->|
   |                |              |              |               |-- Embed -->|
   |                |              |              |               |-- Score -->|
   |                |              |              |               |-- Save --->|
   |                |              |<-- WebSocket "complete" ------|            |
   |<-- WS notify --|              |              |               |            |
   |-- GET result ->|              |              |               |            |
   |                |-- GET /cv/id |--------------|---------------|----------->|
   |<-- CV data ----|<-------------|              |               |            |
```

---

## Metricas y Monitoreo

### KPIs del Pipeline

```yaml
metrics:
  # Latencia
  - cv_processing_duration_seconds:
      type: histogram
      buckets: [1, 5, 10, 20, 30, 60]
      labels: [file_type, ocr_applied]

  # Throughput
  - cv_processed_total:
      type: counter
      labels: [status, file_type]

  # Accuracy
  - extraction_confidence_score:
      type: histogram
      buckets: [0.5, 0.7, 0.8, 0.9, 0.95, 1.0]

  # Errores
  - cv_processing_errors_total:
      type: counter
      labels: [error_type, stage]

  # Queue
  - cv_queue_depth:
      type: gauge
  - cv_queue_wait_time_seconds:
      type: histogram

alerts:
  - name: HighProcessingLatency
    condition: cv_processing_duration_seconds_p95 > 30
    severity: warning

  - name: LowExtractionConfidence
    condition: avg(extraction_confidence_score) < 0.8
    severity: warning

  - name: HighErrorRate
    condition: rate(cv_processing_errors_total[5m]) > 0.1
    severity: critical
```

---

## Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| OCR falla en documentos de baja calidad | Media | Alto | Fallback a Google Vision, notificar usuario |
| Extraccion incorrecta de datos | Media | Alto | Confidence score + revision manual |
| Timeout en archivos grandes | Baja | Medio | Queue async, limites de tamano |
| Costos de API escalan | Media | Medio | Caching agresivo, modelos locales |
| Ataques de inyeccion via CV | Baja | Alto | Sanitizacion, sandbox de parsing |

---

## Plan de Implementacion

### Fase 1: MVP (Semanas 1-2)
- [ ] Setup de queue (BullMQ + Redis)
- [ ] Parser basico (resume-parser)
- [ ] OCR con Tesseract
- [ ] Extraccion de campos basicos
- [ ] Almacenamiento en PostgreSQL

### Fase 2: NLP + Embeddings (Semanas 3-4)
- [ ] Integracion spaCy con NER custom
- [ ] Normalizacion de skills
- [ ] Generacion de embeddings (OpenAI)
- [ ] Almacenamiento en pgvector

### Fase 3: Scoring + Sugerencias (Semana 5)
- [ ] Algoritmo de ATS scoring
- [ ] Generacion de sugerencias
- [ ] UI para visualizacion de resultados

### Fase 4: Optimizacion (Semana 6)
- [ ] LLM fallback para casos edge
- [ ] Mejora de accuracy
- [ ] Performance tuning
- [ ] Monitoreo y alertas

---

## Referencias

- [resume-parser npm](https://www.npmjs.com/package/resume-parser)
- [spaCy NER](https://spacy.io/usage/linguistic-features#named-entities)
- [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [BullMQ Documentation](https://docs.bullmq.io/)
- [Tesseract.js](https://tesseract.projectnaptha.com/)
- [pgvector](https://github.com/pgvector/pgvector)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-31 | Equipo de Arquitectura | Version inicial |
