# ADR-005: Estrategia de Almacenamiento de Datos

## Metadata

| Campo | Valor |
|-------|-------|
| **Estado** | Propuesto |
| **Fecha** | 2026-01-31 |
| **Autores** | Equipo de Arquitectura |
| **Revisores** | CTO, Security Lead, DBA Lead |
| **Contexto DDD** | Multi-Bounded Context |

---

## Contexto

Entrevistador Inteligente requiere una estrategia de almacenamiento de datos que soporte:

1. **Diversidad de datos**: Desde perfiles de usuario hasta embeddings vectoriales para matching semantico
2. **Escalabilidad**: De 500 usuarios (MVP) a 75,000+ usuarios (Ano 3)
3. **Cumplimiento regulatorio**: LPDP (Ley 29733 - Proteccion de Datos Personales Peru)
4. **Multi-tenancy**: Soporte B2B con aislamiento de datos por empresa
5. **Performance**: Busquedas semanticas en <100ms, queries transaccionales en <50ms

### Tipos de Datos a Almacenar

| Tipo | Volumen Estimado (Ano 3) | Caracteristicas |
|------|-------------------------|-----------------|
| Usuarios | 75,000 registros | Transaccional, PII sensible |
| CVs (PDFs) | 150,000 archivos (~2TB) | Archivos binarios, inmutables |
| CVs (parseados) | 150,000 documentos | Semi-estructurado, JSON |
| Embeddings CV | 150,000 vectores (1536 dim) | Alta dimensionalidad |
| Ofertas empleo | 500,000 registros | Semi-estructurado, frecuente update |
| Embeddings Jobs | 500,000 vectores | Alta dimensionalidad |
| Entrevistas | 1,000,000 sesiones | Transcripts largos, scoring |
| Audio (futuro) | 500,000 archivos (~5TB) | Archivos grandes, inmutables |
| Billing | 100,000 transacciones | ACID requerido, auditoria |
| Analytics/Logs | ~50GB/mes | Time-series, alta ingesta |

---

## Decision

### 1. Base de Datos Primaria: PostgreSQL

**Seleccion: PostgreSQL 16+**

#### Evaluacion de Alternativas

| Criterio | PostgreSQL | MySQL 8.0 | CockroachDB |
|----------|------------|-----------|-------------|
| ACID Compliance | Excelente | Bueno | Excelente |
| JSON Support (JSONB) | Nativo, indexable | JSON nativo | Bueno |
| Extension pgvector | Si | No | No |
| Full-text Search | Nativo | Nativo | Limitado |
| Partitioning | Declarativo | Hash/Range | Automatico |
| Multi-tenancy (RLS) | Row Level Security | Manual | Excelente |
| Costo Cloud | Medio | Bajo | Alto |
| Expertise Peru | Alto | Muy Alto | Bajo |
| **Score Total** | **9/10** | **7/10** | **7/10** |

#### Justificacion

1. **pgvector nativo**: Evita necesidad de vector store separado para volumen MVP
2. **JSONB**: Esquema flexible para CV parsing sin migraciones frecuentes
3. **Row Level Security (RLS)**: Multi-tenancy B2B sin cambios de codigo
4. **Madurez**: 35+ anos de desarrollo, comunidad activa en LATAM
5. **Costo**: Instancias RDS/Cloud SQL accesibles para startup

### 2. Vector Store: pgvector (MVP) -> Qdrant (Scale)

**Seleccion: Estrategia Hibrida por Fases**

#### Fase 1-2 (0-12 meses): pgvector

| Criterio | pgvector | Pinecone | Qdrant | Weaviate |
|----------|----------|----------|--------|----------|
| Integracion PostgreSQL | Nativa | API externa | API externa | API externa |
| Costo (10K vectores) | $0 (incluido) | $70/mes | $0 (self-host) | $0 (self-host) |
| Costo (1M vectores) | ~$50/mes (recursos) | $400/mes | $100/mes | $150/mes |
| Latencia busqueda | 10-50ms | 20-50ms | 5-20ms | 10-30ms |
| Filtrado hibrido | SQL nativo | Metadata | Avanzado | GraphQL |
| Operaciones | Una sola DB | Servicio adicional | Container | Container |
| **Score MVP** | **9/10** | **6/10** | **7/10** | **6/10** |

#### Fase 3+ (12+ meses): Migracion a Qdrant

Cuando supere 500K vectores, migrar a Qdrant por:
- HNSW optimizado con 150x-12,500x mejor performance que IVFFlat
- Filtrado avanzado con payloads
- Snapshots y replicacion

#### Justificacion

1. **Simplicidad operativa**: Una sola base de datos para MVP reduce complejidad
2. **Performance suficiente**: pgvector con IVFFlat soporta <100K vectores eficientemente
3. **Migracion limpia**: Qdrant tiene importadores de PostgreSQL
4. **Costo optimizado**: Sin servicio adicional hasta que sea necesario

### 3. Cache: Redis

**Seleccion: Redis 7.x (Managed)**

| Criterio | Redis | Memcached | KeyDB |
|----------|-------|-----------|-------|
| Estructuras de datos | Rico (Sets, Sorted Sets, Streams) | Key-Value simple | Compatible Redis |
| Persistencia | RDB + AOF | No | RDB + AOF |
| Pub/Sub | Nativo | No | Nativo |
| Session storage | Excelente | Bueno | Excelente |
| Rate limiting | Nativo (sliding window) | Manual | Nativo |
| Costo AWS ElastiCache | ~$25/mes (t3.micro) | ~$20/mes | Self-hosted |
| **Score Total** | **9/10** | **6/10** | **7/10** |

#### Casos de Uso

```
+------------------+----------------------------+----------------+
| Caso de Uso      | Patron                     | TTL            |
+------------------+----------------------------+----------------+
| Sessions         | Hash con user data         | 24h            |
| Rate Limiting    | Sliding window counter     | 1min-1h        |
| Job Embeddings   | Vector cache hot           | 1h             |
| API Responses    | JSON cached                | 5-15min        |
| Feature Flags    | Hash                       | Sin expiracion |
| Real-time queues | Streams/Lists              | N/A            |
+------------------+----------------------------+----------------+
```

### 4. File Storage: Cloudflare R2

**Seleccion: Cloudflare R2**

| Criterio | AWS S3 | GCS | Cloudflare R2 | MinIO |
|----------|--------|-----|---------------|-------|
| Costo almacenamiento | $0.023/GB | $0.020/GB | $0.015/GB | Self-hosted |
| Costo egress | $0.09/GB | $0.12/GB | **$0 (gratis)** | N/A |
| S3 Compatible | Nativo | Interop | 100% | 100% |
| CDN integrado | CloudFront ($) | Cloud CDN ($) | Workers (incluido) | No |
| Regiones LATAM | Sao Paulo | Santiago | Edge global | N/A |
| **Score Total** | **7/10** | **7/10** | **9/10** | **6/10** |

#### Justificacion

1. **Zero egress fees**: Critico para descargas frecuentes de CVs/audio
2. **S3 compatible**: Migracion trivial si cambiamos de proveedor
3. **Workers integration**: Procesamiento edge para thumbnails/transcoding
4. **Costo proyectado Ano 3**: ~$105/mes (7TB) vs S3 ~$160/mes + egress

#### Estructura de Buckets

```
entrevistador-prod/
  |-- cvs/
  |     |-- originals/          # PDFs originales (encriptados)
  |     |-- parsed/             # JSON parseados
  |     |-- thumbnails/         # Previews
  |
  |-- audio/
  |     |-- interviews/         # Grabaciones (futuro)
  |     |-- transcripts/        # Whisper output
  |
  |-- exports/
  |     |-- reports/            # Reportes generados
  |     |-- invoices/           # Facturas PDF
  |
  |-- temp/                     # Uploads temporales (TTL: 24h)
```

### 5. Search Engine: Typesense

**Seleccion: Typesense**

| Criterio | Elasticsearch | Algolia | Typesense | Meilisearch |
|----------|---------------|---------|-----------|-------------|
| Latencia busqueda | 10-50ms | 5-20ms | 5-15ms | 10-30ms |
| Costo (100K docs) | $75/mes | $35/mes | $30/mes | Gratis |
| Costo (1M docs) | $300/mes | $250/mes | $100/mes | $50/mes |
| Typo tolerance | Configurable | Excelente | Excelente | Excelente |
| Geo search | Si | Si | Si | Si |
| Faceting | Avanzado | Avanzado | Bueno | Bueno |
| Self-hosted | Si | No | Si | Si |
| Integracion vector | Plugin | No | Nativo | Experimental |
| **Score Total** | **7/10** | **7/10** | **9/10** | **7/10** |

#### Justificacion

1. **Hybrid search**: Combina keyword + semantic en una query
2. **Typo tolerance nativo**: Critico para busquedas en espanol con tildes
3. **Costo predecible**: Sin sorpresas de pricing como Algolia
4. **Self-hosted option**: Control total cuando escalemos

#### Indices Planeados

```yaml
collections:
  - name: job_postings
    fields:
      - name: title
        type: string
        facet: false
      - name: company
        type: string
        facet: true
      - name: location
        type: string
        facet: true
      - name: industry
        type: string
        facet: true
      - name: salary_range
        type: int32[]
        facet: true
      - name: embedding
        type: float[]
        num_dim: 1536
      - name: geo_location
        type: geopoint
```

---

## Schema DDD-Aligned

### Diagrama de Bounded Contexts

```
+------------------------------------------------------------------+
|                        ENTREVISTADOR INTELIGENTE                  |
+------------------------------------------------------------------+
|                                                                   |
|  +-----------------+     +-----------------+     +---------------+|
|  | IDENTITY        |     | CV CONTEXT      |     | MATCHING      ||
|  | CONTEXT         |     |                 |     | CONTEXT       ||
|  |                 |     |                 |     |               ||
|  | - users         |<--->| - cvs           |<--->| - job_postings||
|  | - sessions      |     | - cv_sections   |     | - job_embeds  ||
|  | - roles         |     | - cv_embeddings |     | - matches     ||
|  | - permissions   |     | - skills        |     | - scores      ||
|  |                 |     |                 |     |               ||
|  +-----------------+     +-----------------+     +---------------+|
|          |                       |                      |         |
|          |                       v                      |         |
|          |               +-----------------+            |         |
|          |               | INTERVIEW       |<-----------+         |
|          +-------------->| CONTEXT         |                      |
|                          |                 |                      |
|                          | - interviews    |                      |
|                          | - questions     |                      |
|                          | - responses     |                      |
|                          | - feedback      |                      |
|                          | - transcripts   |                      |
|                          +-----------------+                      |
|                                  |                                |
|  +-----------------+             |             +-----------------+|
|  | BILLING         |<------------+------------>| ANALYTICS       ||
|  | CONTEXT         |                           | CONTEXT         ||
|  |                 |                           |                 ||
|  | - subscriptions |                           | - events        ||
|  | - payments      |                           | - metrics       ||
|  | - invoices      |                           | - audit_logs    ||
|  | - plans         |                           | - user_activity ||
|  |                 |                           |                 ||
|  +-----------------+                           +-----------------+|
|                                                                   |
+------------------------------------------------------------------+
```

### Esquema PostgreSQL por Contexto

#### Identity Context

```sql
-- Schema: identity
CREATE SCHEMA IF NOT EXISTS identity;

CREATE TABLE identity.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    email_verified_at TIMESTAMPTZ,
    password_hash VARCHAR(255) NOT NULL,

    -- Profile
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    avatar_url TEXT,
    locale VARCHAR(10) DEFAULT 'es-PE',
    timezone VARCHAR(50) DEFAULT 'America/Lima',

    -- Multi-tenancy
    tenant_id UUID REFERENCES identity.tenants(id),

    -- Compliance
    terms_accepted_at TIMESTAMPTZ,
    privacy_accepted_at TIMESTAMPTZ,
    marketing_consent BOOLEAN DEFAULT FALSE,
    data_retention_consent BOOLEAN DEFAULT TRUE,

    -- Status
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'deleted')),
    deleted_at TIMESTAMPTZ,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,
    login_count INTEGER DEFAULT 0
);

CREATE TABLE identity.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan_id UUID REFERENCES billing.plans(id),
    settings JSONB DEFAULT '{}',

    -- B2B metadata
    industry VARCHAR(100),
    employee_count INTEGER,
    contact_email VARCHAR(255),

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE identity.sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES identity.users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    ip_address INET,
    user_agent TEXT,
    device_fingerprint VARCHAR(255),

    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE identity.roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    permissions JSONB DEFAULT '[]',
    is_system BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE identity.user_roles (
    user_id UUID REFERENCES identity.users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES identity.roles(id) ON DELETE CASCADE,
    tenant_id UUID REFERENCES identity.tenants(id),
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    granted_by UUID REFERENCES identity.users(id),

    PRIMARY KEY (user_id, role_id, tenant_id)
);

-- Row Level Security para multi-tenancy
ALTER TABLE identity.users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON identity.users
    USING (tenant_id = current_setting('app.current_tenant')::UUID
           OR current_setting('app.is_superadmin')::BOOLEAN = TRUE);

-- Indices
CREATE INDEX idx_users_email ON identity.users(email);
CREATE INDEX idx_users_tenant ON identity.users(tenant_id);
CREATE INDEX idx_sessions_user ON identity.sessions(user_id);
CREATE INDEX idx_sessions_token ON identity.sessions(token_hash);
```

#### CV Context

```sql
-- Schema: cv
CREATE SCHEMA IF NOT EXISTS cv;

CREATE TABLE cv.cvs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES identity.users(id) ON DELETE CASCADE,

    -- File reference
    original_file_key TEXT NOT NULL,  -- R2 key
    original_filename VARCHAR(255),
    file_size_bytes INTEGER,
    mime_type VARCHAR(100),

    -- Parsed data
    parsed_data JSONB,  -- Estructura completa del CV
    raw_text TEXT,      -- Texto extraido para indexing
    language VARCHAR(10) DEFAULT 'es',

    -- Status
    parsing_status VARCHAR(20) DEFAULT 'pending'
        CHECK (parsing_status IN ('pending', 'processing', 'completed', 'failed')),
    parsing_error TEXT,
    parsed_at TIMESTAMPTZ,

    -- Versioning
    version INTEGER DEFAULT 1,
    is_primary BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE cv.cv_sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cv_id UUID NOT NULL REFERENCES cv.cvs(id) ON DELETE CASCADE,

    section_type VARCHAR(50) NOT NULL
        CHECK (section_type IN ('education', 'experience', 'skills', 'certifications',
                                 'languages', 'projects', 'summary', 'other')),

    content JSONB NOT NULL,
    display_order INTEGER DEFAULT 0,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE cv.cv_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cv_id UUID NOT NULL REFERENCES cv.cvs(id) ON DELETE CASCADE,

    -- Vector (pgvector)
    embedding vector(1536) NOT NULL,  -- OpenAI ada-002 dimension

    -- Metadata
    model_version VARCHAR(50) DEFAULT 'text-embedding-ada-002',
    section_type VARCHAR(50),  -- NULL = full CV embedding

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE cv.skills (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cv_id UUID NOT NULL REFERENCES cv.cvs(id) ON DELETE CASCADE,

    skill_name VARCHAR(255) NOT NULL,
    skill_category VARCHAR(100),
    proficiency_level VARCHAR(20) CHECK (proficiency_level IN ('beginner', 'intermediate', 'advanced', 'expert')),
    years_experience DECIMAL(4,1),

    -- Normalizacion
    canonical_skill_id UUID,  -- Link a skills normalizados
    confidence_score DECIMAL(3,2),  -- Confidence del parser

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indices
CREATE INDEX idx_cvs_user ON cv.cvs(user_id);
CREATE INDEX idx_cvs_primary ON cv.cvs(user_id, is_primary) WHERE is_primary = TRUE;
CREATE INDEX idx_cv_embeddings_vector ON cv.cv_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_cv_sections_cv ON cv.cv_sections(cv_id);
CREATE INDEX idx_skills_cv ON cv.skills(cv_id);
CREATE INDEX idx_skills_name ON cv.skills(skill_name);
```

#### Matching Context

```sql
-- Schema: matching
CREATE SCHEMA IF NOT EXISTS matching;

CREATE TABLE matching.job_postings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Source
    source VARCHAR(50) NOT NULL CHECK (source IN ('scraped', 'api', 'manual', 'partner')),
    external_id VARCHAR(255),
    source_url TEXT,

    -- Job details
    title VARCHAR(255) NOT NULL,
    company_name VARCHAR(255),
    company_logo_url TEXT,

    description TEXT NOT NULL,
    requirements JSONB,
    benefits JSONB,

    -- Classification
    industry VARCHAR(100),
    job_function VARCHAR(100),
    seniority_level VARCHAR(50),
    employment_type VARCHAR(50) CHECK (employment_type IN ('full-time', 'part-time', 'contract', 'internship', 'freelance')),

    -- Location
    location_city VARCHAR(100),
    location_country VARCHAR(100) DEFAULT 'Peru',
    is_remote BOOLEAN DEFAULT FALSE,

    -- Compensation
    salary_min DECIMAL(12,2),
    salary_max DECIMAL(12,2),
    salary_currency VARCHAR(3) DEFAULT 'PEN',
    salary_period VARCHAR(20) DEFAULT 'monthly',

    -- Status
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'expired', 'filled', 'removed')),
    posted_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,

    -- Tenant (para ofertas B2B)
    tenant_id UUID REFERENCES identity.tenants(id),

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE matching.job_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES matching.job_postings(id) ON DELETE CASCADE,

    embedding vector(1536) NOT NULL,
    model_version VARCHAR(50) DEFAULT 'text-embedding-ada-002',

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE matching.matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    cv_id UUID NOT NULL REFERENCES cv.cvs(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES matching.job_postings(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES identity.users(id) ON DELETE CASCADE,

    -- Scores
    overall_score DECIMAL(5,4) NOT NULL,  -- 0.0000 - 1.0000
    semantic_score DECIMAL(5,4),
    skill_match_score DECIMAL(5,4),
    experience_score DECIMAL(5,4),

    -- Breakdown
    matching_skills JSONB,     -- Skills que matchean
    missing_skills JSONB,      -- Skills faltantes
    explanation TEXT,          -- Explicacion para el usuario

    -- User interaction
    user_rating INTEGER CHECK (user_rating BETWEEN 1 AND 5),
    user_feedback TEXT,
    applied_at TIMESTAMPTZ,
    saved_at TIMESTAMPTZ,
    dismissed_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(cv_id, job_id)
);

-- Indices
CREATE INDEX idx_jobs_status ON matching.job_postings(status) WHERE status = 'active';
CREATE INDEX idx_jobs_location ON matching.job_postings(location_city, location_country);
CREATE INDEX idx_jobs_industry ON matching.job_postings(industry);
CREATE INDEX idx_jobs_tenant ON matching.job_postings(tenant_id);
CREATE INDEX idx_job_embeddings_vector ON matching.job_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_matches_user ON matching.matches(user_id);
CREATE INDEX idx_matches_score ON matching.matches(overall_score DESC);
```

#### Interview Context

```sql
-- Schema: interview
CREATE SCHEMA IF NOT EXISTS interview;

CREATE TABLE interview.interviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES identity.users(id) ON DELETE CASCADE,

    -- Context
    job_id UUID REFERENCES matching.job_postings(id),
    cv_id UUID REFERENCES cv.cvs(id),

    -- Interview type
    interview_type VARCHAR(50) NOT NULL CHECK (interview_type IN (
        'behavioral', 'technical', 'case_study', 'hr_screening', 'competency', 'mixed'
    )),
    industry VARCHAR(100),
    role_level VARCHAR(50),

    -- Configuration
    language VARCHAR(10) DEFAULT 'es',
    difficulty_level VARCHAR(20) DEFAULT 'medium' CHECK (difficulty_level IN ('easy', 'medium', 'hard', 'expert')),
    target_duration_minutes INTEGER DEFAULT 30,

    -- Status
    status VARCHAR(20) DEFAULT 'in_progress' CHECK (status IN ('in_progress', 'completed', 'abandoned', 'expired')),
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    actual_duration_seconds INTEGER,

    -- Scoring
    overall_score DECIMAL(5,2),  -- 0-100
    communication_score DECIMAL(5,2),
    content_score DECIMAL(5,2),
    structure_score DECIMAL(5,2),
    confidence_score DECIMAL(5,2),

    -- AI Feedback
    ai_feedback JSONB,
    improvement_areas JSONB,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE interview.questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id UUID NOT NULL REFERENCES interview.interviews(id) ON DELETE CASCADE,

    -- Question content
    question_text TEXT NOT NULL,
    question_type VARCHAR(50) NOT NULL CHECK (question_type IN (
        'behavioral', 'technical', 'situational', 'competency', 'follow_up'
    )),
    category VARCHAR(100),

    -- Metadata
    difficulty_level VARCHAR(20),
    expected_answer_points JSONB,
    follow_up_to UUID REFERENCES interview.questions(id),

    -- Order
    sequence_number INTEGER NOT NULL,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE interview.responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id UUID NOT NULL REFERENCES interview.questions(id) ON DELETE CASCADE,
    interview_id UUID NOT NULL REFERENCES interview.interviews(id) ON DELETE CASCADE,

    -- Response content
    response_text TEXT,
    response_audio_key TEXT,  -- R2 key para audio (futuro)

    -- Metrics
    response_duration_seconds INTEGER,
    word_count INTEGER,

    -- Scoring
    score DECIMAL(5,2),
    content_score DECIMAL(5,2),
    delivery_score DECIMAL(5,2),

    -- AI Analysis
    ai_analysis JSONB,
    key_points_covered JSONB,
    missed_points JSONB,
    suggested_improvement TEXT,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE interview.feedback (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id UUID NOT NULL REFERENCES interview.interviews(id) ON DELETE CASCADE,

    -- Feedback type
    feedback_type VARCHAR(50) NOT NULL CHECK (feedback_type IN (
        'summary', 'strength', 'improvement', 'action_item', 'resource'
    )),

    content TEXT NOT NULL,
    priority INTEGER DEFAULT 0,
    category VARCHAR(100),

    -- Resources
    resource_url TEXT,
    resource_type VARCHAR(50),

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indices
CREATE INDEX idx_interviews_user ON interview.interviews(user_id);
CREATE INDEX idx_interviews_status ON interview.interviews(status);
CREATE INDEX idx_interviews_date ON interview.interviews(created_at DESC);
CREATE INDEX idx_questions_interview ON interview.questions(interview_id);
CREATE INDEX idx_responses_interview ON interview.responses(interview_id);
CREATE INDEX idx_feedback_interview ON interview.feedback(interview_id);
```

#### Billing Context

```sql
-- Schema: billing
CREATE SCHEMA IF NOT EXISTS billing;

CREATE TABLE billing.plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    name VARCHAR(100) NOT NULL,
    slug VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,

    -- Pricing
    price_monthly DECIMAL(10,2) NOT NULL,
    price_yearly DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'PEN',

    -- Limits
    interviews_per_month INTEGER,
    cv_uploads INTEGER,
    ai_feedback_detail VARCHAR(20) CHECK (ai_feedback_detail IN ('basic', 'detailed', 'comprehensive')),
    priority_support BOOLEAN DEFAULT FALSE,

    -- Metadata
    is_active BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    display_order INTEGER DEFAULT 0,

    features JSONB,  -- Lista de features para UI

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE billing.subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    user_id UUID REFERENCES identity.users(id) ON DELETE SET NULL,
    tenant_id UUID REFERENCES identity.tenants(id) ON DELETE SET NULL,
    plan_id UUID NOT NULL REFERENCES billing.plans(id),

    -- Status
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN (
        'active', 'canceled', 'past_due', 'paused', 'trialing', 'expired'
    )),

    -- Billing
    billing_cycle VARCHAR(20) CHECK (billing_cycle IN ('monthly', 'yearly')),
    current_period_start TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN DEFAULT FALSE,
    canceled_at TIMESTAMPTZ,

    -- Payment provider
    provider VARCHAR(50),  -- 'stripe', 'mercadopago', etc.
    provider_subscription_id VARCHAR(255),
    provider_customer_id VARCHAR(255),

    -- Trial
    trial_start TIMESTAMPTZ,
    trial_end TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    CHECK (user_id IS NOT NULL OR tenant_id IS NOT NULL)
);

CREATE TABLE billing.payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID REFERENCES billing.subscriptions(id),

    -- Amount
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'PEN',

    -- Status
    status VARCHAR(20) NOT NULL CHECK (status IN (
        'pending', 'processing', 'succeeded', 'failed', 'refunded', 'disputed'
    )),

    -- Payment method
    payment_method_type VARCHAR(50),  -- 'card', 'bank_transfer', 'yape', etc.
    payment_method_last4 VARCHAR(4),

    -- Provider
    provider VARCHAR(50) NOT NULL,
    provider_payment_id VARCHAR(255),
    provider_fee DECIMAL(10,2),

    -- Dates
    paid_at TIMESTAMPTZ,
    failed_at TIMESTAMPTZ,
    refunded_at TIMESTAMPTZ,

    -- Metadata
    failure_reason TEXT,
    metadata JSONB,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE billing.invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID REFERENCES billing.subscriptions(id),
    payment_id UUID REFERENCES billing.payments(id),

    -- Invoice details
    invoice_number VARCHAR(50) NOT NULL UNIQUE,

    -- Amounts
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    total DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'PEN',

    -- Status
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'open', 'paid', 'void', 'uncollectible')),

    -- Billing info
    billing_name VARCHAR(255),
    billing_email VARCHAR(255),
    billing_address JSONB,
    tax_id VARCHAR(50),  -- RUC para Peru

    -- PDF
    pdf_url TEXT,

    -- Dates
    issue_date DATE NOT NULL,
    due_date DATE,
    paid_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indices
CREATE INDEX idx_subscriptions_user ON billing.subscriptions(user_id);
CREATE INDEX idx_subscriptions_tenant ON billing.subscriptions(tenant_id);
CREATE INDEX idx_subscriptions_status ON billing.subscriptions(status);
CREATE INDEX idx_payments_subscription ON billing.payments(subscription_id);
CREATE INDEX idx_payments_status ON billing.payments(status);
CREATE INDEX idx_invoices_subscription ON billing.invoices(subscription_id);
```

#### Analytics Context

```sql
-- Schema: analytics
CREATE SCHEMA IF NOT EXISTS analytics;

-- TimescaleDB hypertable (si usamos TimescaleDB extension)
CREATE TABLE analytics.events (
    id UUID DEFAULT gen_random_uuid(),

    -- Event identity
    event_name VARCHAR(100) NOT NULL,
    event_category VARCHAR(50),

    -- Actor
    user_id UUID REFERENCES identity.users(id) ON DELETE SET NULL,
    tenant_id UUID REFERENCES identity.tenants(id) ON DELETE SET NULL,
    session_id UUID,

    -- Event data
    properties JSONB,

    -- Context
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    page_url TEXT,

    -- Device
    device_type VARCHAR(20),
    browser VARCHAR(50),
    os VARCHAR(50),

    -- Geo
    country VARCHAR(2),
    city VARCHAR(100),

    -- Time
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (id, timestamp)
);

-- Si usamos TimescaleDB
-- SELECT create_hypertable('analytics.events', 'timestamp', chunk_time_interval => INTERVAL '1 day');

CREATE TABLE analytics.metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    metric_name VARCHAR(100) NOT NULL,
    metric_value DECIMAL(15,4) NOT NULL,

    -- Dimensions
    dimensions JSONB,

    -- Aggregation
    aggregation_type VARCHAR(20) DEFAULT 'sum' CHECK (aggregation_type IN ('sum', 'avg', 'count', 'min', 'max')),
    aggregation_period VARCHAR(20) CHECK (aggregation_period IN ('minute', 'hour', 'day', 'week', 'month')),

    -- Time
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE analytics.audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Actor
    user_id UUID REFERENCES identity.users(id) ON DELETE SET NULL,
    actor_type VARCHAR(50) DEFAULT 'user' CHECK (actor_type IN ('user', 'system', 'api', 'admin')),

    -- Action
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID,

    -- Changes
    old_values JSONB,
    new_values JSONB,

    -- Context
    ip_address INET,
    user_agent TEXT,

    -- Compliance
    reason TEXT,
    compliance_flag BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indices con partitioning para eventos
CREATE INDEX idx_events_user_time ON analytics.events(user_id, timestamp DESC);
CREATE INDEX idx_events_name_time ON analytics.events(event_name, timestamp DESC);
CREATE INDEX idx_events_category ON analytics.events(event_category, timestamp DESC);
CREATE INDEX idx_metrics_name_period ON analytics.metrics(metric_name, period_start);
CREATE INDEX idx_audit_user ON analytics.audit_logs(user_id, created_at DESC);
CREATE INDEX idx_audit_resource ON analytics.audit_logs(resource_type, resource_id);
```

---

## Consideraciones de Seguridad y Compliance

### 1. LPDP (Ley 29733) - Proteccion de Datos Personales Peru

```
+----------------------------------------------------------------------+
|                    REQUERIMIENTOS LPDP                                |
+----------------------------------------------------------------------+
| Requisito                    | Implementacion                        |
+----------------------------------------------------------------------+
| Consentimiento informado     | Checkbox explicitio en registro       |
|                              | + terms_accepted_at, privacy_accepted |
+----------------------------------------------------------------------+
| Finalidad especifica         | Documentar en privacy policy          |
|                              | + marketing_consent separado          |
+----------------------------------------------------------------------+
| Acceso a datos propios       | API /me/data-export                   |
|                              | + Dashboard de datos personales       |
+----------------------------------------------------------------------+
| Rectificacion                | Edicion de perfil + CV                |
|                              | + Solicitud via soporte               |
+----------------------------------------------------------------------+
| Cancelacion (eliminacion)    | Soft delete + purge a 30 dias         |
|                              | + data_retention_consent flag         |
+----------------------------------------------------------------------+
| Transferencia internacional  | Standard contractual clauses          |
|                              | + Disclosure en privacy policy        |
+----------------------------------------------------------------------+
| Medidas de seguridad         | Encriptacion, access controls         |
|                              | + Audit logging                       |
+----------------------------------------------------------------------+
```

### 2. Encriptacion

```yaml
# Estrategia de encriptacion por capa

Transport Layer:
  - TLS 1.3 para todas las conexiones
  - Certificate pinning para apps moviles (futuro)
  - HSTS enabled

Application Layer:
  - Campos PII encriptados con AES-256-GCM:
      - password_hash (bcrypt, no reversible)
      - phone
      - billing_address
      - tax_id
  - API keys encriptadas en database
  - Tokens JWT con firma RS256

Storage Layer:
  - PostgreSQL: Encryption at rest (AWS RDS / Cloud SQL default)
  - Redis: Encryption at rest + in transit
  - R2: Server-side encryption (SSE)
  - Backups: Encrypted con customer-managed keys

Key Management:
  - AWS KMS / GCP Cloud KMS para key rotation
  - Envelope encryption para datos sensibles
  - Key rotation cada 90 dias
```

### 3. Backup y Disaster Recovery

```
+----------------------------------------------------------------------+
|                    ESTRATEGIA DE BACKUP                               |
+----------------------------------------------------------------------+
| Componente      | RPO        | RTO       | Estrategia                |
+----------------------------------------------------------------------+
| PostgreSQL      | 5 min      | 1 hora    | Point-in-time recovery    |
|                 |            |           | + Daily snapshots         |
|                 |            |           | + Cross-region replica    |
+----------------------------------------------------------------------+
| Redis           | 1 hora     | 15 min    | RDB snapshots cada hora   |
|                 |            |           | + AOF para datos criticos |
+----------------------------------------------------------------------+
| R2 (files)      | 0 (sync)   | Inmediato | Versionamiento habilitado |
|                 |            |           | + Cross-region replication|
+----------------------------------------------------------------------+
| Typesense       | 1 hora     | 30 min    | Snapshots programados     |
|                 |            |           | + Rebuild desde PostgreSQL|
+----------------------------------------------------------------------+

Retention Policy:
  - Daily backups: 7 dias
  - Weekly backups: 4 semanas
  - Monthly backups: 12 meses
  - Yearly backups: 7 anos (compliance)

Disaster Recovery:
  - Primary region: us-east-1 (Virginia) - cercano a Peru
  - DR region: sa-east-1 (Sao Paulo)
  - Failover automatico para database
  - DNS failover con health checks
```

### 4. Multi-tenancy B2B

```sql
-- Estrategia: Row Level Security (RLS) con shared database

-- 1. Contexto de tenant via session variable
CREATE OR REPLACE FUNCTION set_tenant_context(tenant_uuid UUID)
RETURNS VOID AS $$
BEGIN
    PERFORM set_config('app.current_tenant', tenant_uuid::TEXT, FALSE);
END;
$$ LANGUAGE plpgsql;

-- 2. Politicas RLS en tablas sensibles
CREATE POLICY tenant_isolation_users ON identity.users
    FOR ALL
    USING (
        tenant_id = current_setting('app.current_tenant', TRUE)::UUID
        OR current_setting('app.is_superadmin', TRUE)::BOOLEAN = TRUE
        OR tenant_id IS NULL  -- Usuarios B2C sin tenant
    );

CREATE POLICY tenant_isolation_jobs ON matching.job_postings
    FOR ALL
    USING (
        tenant_id = current_setting('app.current_tenant', TRUE)::UUID
        OR current_setting('app.is_superadmin', TRUE)::BOOLEAN = TRUE
        OR tenant_id IS NULL  -- Jobs publicos
    );

-- 3. Indices para performance con RLS
CREATE INDEX idx_users_tenant_email ON identity.users(tenant_id, email);
CREATE INDEX idx_jobs_tenant_status ON matching.job_postings(tenant_id, status);

-- 4. Vista para metricas por tenant
CREATE VIEW analytics.tenant_metrics AS
SELECT
    t.id as tenant_id,
    t.name as tenant_name,
    COUNT(DISTINCT u.id) as user_count,
    COUNT(DISTINCT i.id) as interview_count,
    AVG(i.overall_score) as avg_interview_score
FROM identity.tenants t
LEFT JOIN identity.users u ON u.tenant_id = t.id
LEFT JOIN interview.interviews i ON i.user_id = u.id
GROUP BY t.id, t.name;
```

---

## Diagrama de Arquitectura de Datos

```
                                    +-------------------+
                                    |   Application     |
                                    |   (Node.js)       |
                                    +--------+----------+
                                             |
                    +------------------------+------------------------+
                    |                        |                        |
                    v                        v                        v
           +----------------+      +------------------+      +----------------+
           |    Redis       |      |   PostgreSQL 16  |      | Cloudflare R2  |
           |    (Cache)     |      |    (Primary DB)  |      |   (Files)      |
           +----------------+      +------------------+      +----------------+
           |                |      |                  |      |                |
           | - Sessions     |      | Schemas:         |      | Buckets:       |
           | - Rate limits  |      | - identity       |      | - cvs/         |
           | - Hot vectors  |      | - cv             |      | - audio/       |
           | - API cache    |      | - matching       |      | - exports/     |
           |                |      | - interview      |      |                |
           +----------------+      | - billing        |      +----------------+
                                   | - analytics      |
                                   |                  |
                                   | Extensions:      |
                                   | - pgvector       |
                                   | - pg_trgm        |
                                   +--------+---------+
                                            |
                    +-----------------------+-----------------------+
                    |                                               |
                    v                                               v
           +------------------+                           +------------------+
           |   Typesense      |                           |   Analytics      |
           |   (Search)       |                           |   Pipeline       |
           +------------------+                           +------------------+
           |                  |                           |                  |
           | Collections:     |                           | - Event stream   |
           | - job_postings   |                           | - TimescaleDB    |
           | - candidates     |                           |   (opcional)     |
           | (with vectors)   |                           | - Metabase       |
           |                  |                           |                  |
           +------------------+                           +------------------+

                              FUTURE: Qdrant Cluster
                              +------------------+
                              |     Qdrant       |
                              |  (Scale Vector)  |
                              +------------------+
                              | - CV embeddings  |
                              | - Job embeddings |
                              | - HNSW index     |
                              +------------------+
```

---

## Plan de Migracion e Implementacion

### Fase 1: MVP (Meses 1-6)

| Semana | Componente | Tarea |
|--------|------------|-------|
| 1-2 | PostgreSQL | Setup RDS/Cloud SQL, schemas identity + cv |
| 2-3 | pgvector | Instalar extension, crear indices IVFFlat |
| 3-4 | Redis | Setup ElastiCache, configurar session store |
| 4-5 | R2 | Crear buckets, configurar lifecycle rules |
| 5-6 | Typesense | Deploy container, sync inicial desde PostgreSQL |

### Fase 2: Growth (Meses 6-12)

| Actividad | Timeline |
|-----------|----------|
| Read replicas PostgreSQL | Mes 7 |
| Redis cluster mode | Mes 8 |
| Typesense HA (3 nodos) | Mes 9 |
| Analytics pipeline completo | Mes 10-11 |
| Backup cross-region | Mes 12 |

### Fase 3: Scale (Meses 12-24)

| Actividad | Timeline |
|-----------|----------|
| Migracion a Qdrant | Mes 14-15 |
| Sharding PostgreSQL (si necesario) | Mes 18+ |
| Multi-region (Colombia) | Mes 20+ |

---

## Consecuencias

### Positivas

1. **Stack cohesivo**: PostgreSQL como fuente de verdad con extensiones reduce fragmentacion
2. **Costo optimizado**: pgvector elimina servicio vector separado en MVP
3. **Compliance built-in**: RLS para multi-tenancy, audit logs nativos
4. **Migracion gradual**: Path claro de pgvector a Qdrant cuando sea necesario
5. **Developer experience**: Stack familiar para devs LATAM

### Negativas

1. **Complejidad schemas**: 6 schemas requieren disciplina de equipo
2. **pgvector limitaciones**: Performance degrada >500K vectores
3. **Vendor lock-in parcial**: R2 API compatible pero features especificos Cloudflare
4. **Overhead RLS**: Impacto menor en queries complejas

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| pgvector no escala | Media | Alto | Plan de migracion a Qdrant documentado |
| Costos R2 ocultos | Baja | Medio | Monitoreo con alertas, lifecycle policies |
| Typesense downtime | Baja | Medio | Fallback a PostgreSQL full-text |
| Breach de datos | Baja | Muy Alto | Encriptacion, audits, penetration testing |

---

## Referencias

- [PostgreSQL 16 Documentation](https://www.postgresql.org/docs/16/)
- [pgvector Performance Tuning](https://github.com/pgvector/pgvector)
- [Qdrant vs pgvector Benchmark](https://qdrant.tech/benchmarks/)
- [Cloudflare R2 Pricing](https://developers.cloudflare.com/r2/pricing/)
- [Typesense Documentation](https://typesense.org/docs/)
- [Ley 29733 - LPDP Peru](https://www.gob.pe/institucion/pcm/informes-publicaciones/3621665-ley-n-29733)
- [OWASP Data Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

---

## Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| CTO | | | |
| Security Lead | | | |
| DBA Lead | | | |
| Product Owner | | | |

---

*Documento generado como parte del proceso de Architecture Decision Records*
*Version: 1.0 | Fecha: 2026-01-31*
