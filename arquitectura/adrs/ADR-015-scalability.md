# ADR-015: Estrategia de Escalabilidad

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead, DevOps Lead

---

## Contexto

### Proyecciones de Carga

Basado en el modelo de negocio y proyecciones de mercado para Peru y expansion LATAM:

| Fase | Periodo | Usuarios | Entrevistas/Mes | Usuarios Concurrentes |
|------|---------|----------|-----------------|----------------------|
| **MVP** | Mes 1-3 | 500 | 1,000 | 50 max |
| **Growth** | Mes 6-12 | 5,000 | 15,000 | 200 |
| **Scale** | Ano 2+ | 50,000 | 200,000 | 1,000 |

### Metricas Derivadas

```
MVP (Mes 1-3):
- Requests/segundo (promedio): ~0.5 RPS
- Requests/segundo (pico): ~5 RPS
- Storage estimado: 10 GB (CVs + datos)
- Procesamiento CV: ~33/dia
- Sesiones de entrevista IA: ~33/dia

Growth (Mes 6-12):
- Requests/segundo (promedio): ~5 RPS
- Requests/segundo (pico): ~50 RPS
- Storage estimado: 100 GB
- Procesamiento CV: ~500/dia
- Sesiones de entrevista IA: ~500/dia

Scale (Ano 2+):
- Requests/segundo (promedio): ~50 RPS
- Requests/segundo (pico): ~500 RPS
- Storage estimado: 1 TB+
- Procesamiento CV: ~6,600/dia
- Sesiones de entrevista IA: ~6,600/dia
```

### Problema a Resolver

Definir una estrategia de escalabilidad que:
1. Mantenga costos bajos en fase MVP (<$500/mes)
2. Escale de forma predecible con el crecimiento
3. Maneje los cuellos de botella especificos de IA/ML
4. Permita expansion geografica (multi-region)
5. Optimice la relacion costo/rendimiento en cada fase

---

## Decision

### 1. Cloud Provider: Vercel + Supabase (MVP) -> AWS (Scale)

**Fase MVP-Growth: Vercel + Supabase**

```
┌─────────────────────────────────────────────────────────────┐
│                         Vercel                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Next.js   │  │  API Routes │  │   Edge      │          │
│  │   Frontend  │  │  (Serverless)│  │  Functions  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
      │  Supabase   │ │  Supabase   │ │  Supabase   │
      │  PostgreSQL │ │   Storage   │ │    Auth     │
      │  + pgvector │ │   (S3)      │ │             │
      └─────────────┘ └─────────────┘ └─────────────┘
                              │
                      ┌───────────────┐
                      │   Upstash     │
                      │   Redis       │
                      │   (Cache/Queue)│
                      └───────────────┘
```

**Fase Scale: Migracion a AWS**

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ CloudFront  │  │     ECS     │  │   Lambda    │          │
│  │    CDN      │  │   Fargate   │  │  (Workers)  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                              │                               │
│          ┌───────────────────┼───────────────────┐          │
│          ▼                   ▼                   ▼          │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐    │
│  │    RDS      │     │     S3      │     │ElastiCache  │    │
│  │  PostgreSQL │     │   Storage   │     │   Redis     │    │
│  │  + pgvector │     │   + CDN     │     │   Cluster   │    │
│  └─────────────┘     └─────────────┘     └─────────────┘    │
│          │                                                   │
│          ▼                                                   │
│  ┌─────────────┐     ┌─────────────┐                        │
│  │    RDS      │     │  Pinecone   │                        │
│  │   Replica   │     │  (Vectors)  │                        │
│  └─────────────┘     └─────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

**Justificacion:**

| Criterio | Vercel+Supabase | AWS | GCP | Azure |
|----------|-----------------|-----|-----|-------|
| Costo MVP | $50-200/mes | $300-600/mes | $250-500/mes | $300-600/mes |
| Time-to-market | 1-2 semanas | 4-6 semanas | 3-5 semanas | 4-6 semanas |
| Complejidad ops | Minima | Alta | Media | Alta |
| Escalabilidad max | Media | Muy Alta | Muy Alta | Muy Alta |
| Vendor lock-in | Bajo | Medio | Medio | Medio |
| Edge performance | Excelente | Buena | Buena | Buena |

**Decision:** Vercel + Supabase para MVP/Growth por simplicidad y costo. Migracion planificada a AWS cuando alcancemos >$2,000/mes o necesitemos capacidades avanzadas.

---

### 2. Container Orchestration: Serverless (MVP) -> ECS Fargate (Scale)

**Fase MVP-Growth: Serverless Functions**

```typescript
// vercel.json - Configuracion serverless
{
  "functions": {
    "api/**/*.ts": {
      "maxDuration": 30,
      "memory": 1024
    },
    "api/cv/process.ts": {
      "maxDuration": 60,
      "memory": 3008
    },
    "api/interview/session.ts": {
      "maxDuration": 300,
      "memory": 1024
    }
  }
}
```

**Fase Scale: ECS Fargate con Auto-scaling**

```yaml
# ecs-task-definition.yaml
version: '3'
services:
  api:
    image: entrevistador/api:latest
    cpu: 512
    memory: 1024
    auto_scaling:
      min_capacity: 2
      max_capacity: 20
      target_cpu_utilization: 70
      target_memory_utilization: 80
      scale_in_cooldown: 300
      scale_out_cooldown: 60

  cv-worker:
    image: entrevistador/cv-worker:latest
    cpu: 1024
    memory: 2048
    auto_scaling:
      min_capacity: 1
      max_capacity: 10
      target_queue_depth: 100

  interview-worker:
    image: entrevistador/interview-worker:latest
    cpu: 512
    memory: 1024
    auto_scaling:
      min_capacity: 2
      max_capacity: 15
      target_concurrent_sessions: 50
```

**Comparativa de Orquestacion:**

| Criterio | Serverless | ECS Fargate | Kubernetes | Cloud Run |
|----------|------------|-------------|------------|-----------|
| Costo MVP | Muy bajo | Medio | Alto | Bajo |
| Complejidad | Minima | Media | Alta | Baja |
| Cold starts | Si | No | No | Si |
| Control | Bajo | Alto | Muy Alto | Medio |
| Portabilidad | Baja | Media | Alta | Media |

**Decision:** Serverless para MVP (simplicidad), migracion a ECS Fargate para Scale (control y eficiencia).

---

### 3. Database Scaling Strategy

#### Fase MVP: Supabase PostgreSQL

```
┌─────────────────────────────────────┐
│         Supabase PostgreSQL          │
│         (Pro Plan - $25/mes)         │
│  ┌─────────────────────────────┐    │
│  │     8 GB RAM, 2 vCPU        │    │
│  │     50 GB Storage           │    │
│  │     pgvector extension      │    │
│  │     Connection pooling      │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

#### Fase Growth: Supabase + Optimizaciones

```
┌─────────────────────────────────────────────────────────────┐
│              Supabase PostgreSQL (Team Plan)                 │
│  ┌─────────────────────────────┐                            │
│  │    Primary Database         │                            │
│  │    16 GB RAM, 4 vCPU        │◄──── Writes                │
│  │    100 GB Storage           │                            │
│  └─────────────────────────────┘                            │
│                  │                                           │
│                  ▼                                           │
│  ┌─────────────────────────────┐                            │
│  │    Read Replica             │◄──── Analytics/Reports     │
│  │    (Point-in-time recovery) │                            │
│  └─────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│         Upstash Redis               │
│    (Query cache, session cache)     │
└─────────────────────────────────────┘
```

#### Fase Scale: RDS + Sharding Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS RDS PostgreSQL                        │
│                                                              │
│  ┌─────────────────────────────┐                            │
│  │       Primary Writer        │                            │
│  │    db.r6g.xlarge (32GB)     │◄──── Writes                │
│  │    Multi-AZ deployment      │                            │
│  └─────────────────────────────┘                            │
│           │                │                                 │
│           ▼                ▼                                 │
│  ┌─────────────┐  ┌─────────────┐                           │
│  │Read Replica │  │Read Replica │◄──── Reads (70% traffic)  │
│  │  (Same AZ)  │  │ (Cross AZ)  │                           │
│  └─────────────┘  └─────────────┘                           │
│                                                              │
│  Sharding Strategy (Ano 2+):                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Shard by tenant_id (empresas B2B)                  │    │
│  │  - Shard 1: tenant_id % 4 == 0 (PE-Lima)            │    │
│  │  - Shard 2: tenant_id % 4 == 1 (PE-Provincias)      │    │
│  │  - Shard 3: tenant_id % 4 == 2 (Colombia)           │    │
│  │  - Shard 4: tenant_id % 4 == 3 (Mexico+Chile)       │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Schema Partitioning (Growth Phase):**

```sql
-- Particionamiento por fecha para tablas grandes
CREATE TABLE interview_sessions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    -- ... otros campos
) PARTITION BY RANGE (created_at);

-- Particiones mensuales
CREATE TABLE interview_sessions_2026_01
    PARTITION OF interview_sessions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE interview_sessions_2026_02
    PARTITION OF interview_sessions
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Indices optimizados
CREATE INDEX idx_sessions_user_date ON interview_sessions(user_id, created_at DESC);
CREATE INDEX idx_sessions_status ON interview_sessions(status) WHERE status = 'active';
```

**Connection Pooling:**

```typescript
// MVP: Supabase built-in pooling (pgbouncer)
// Growth: Configuracion explicita

// supabase-config.ts
export const dbConfig = {
  // Transaction mode para queries cortas
  connectionString: process.env.DATABASE_URL,
  max: 20,  // Pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
};

// Scale: PgBouncer standalone
// pgbouncer.ini
/*
[databases]
entrevistador = host=primary.rds.amazonaws.com port=5432 dbname=entrevistador

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
reserve_pool_timeout = 3
*/
```

---

### 4. Caching Strategy

#### Arquitectura de Cache Multi-Capa

```
┌─────────────────────────────────────────────────────────────┐
│                      Caching Layers                          │
│                                                              │
│  Layer 1: Browser/CDN Cache                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Static assets: 1 year (immutable)                │    │
│  │  - API responses: 1-60 seconds (stale-while-revalidate) │
│  │  - HTML pages: 0 seconds (dynamic)                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Layer 2: Edge Cache (Vercel/CloudFront)                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Job listings: 5 min cache                        │    │
│  │  - Company profiles: 10 min cache                   │    │
│  │  - Static content: 24h cache                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Layer 3: Application Cache (Redis)                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Session data: 24h TTL                            │    │
│  │  - User preferences: 1h TTL                         │    │
│  │  - LLM response cache: 7 days TTL                   │    │
│  │  - Matching scores: 1h TTL                          │    │
│  │  - Rate limiting counters: 1 min TTL                │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Layer 4: Database Query Cache                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Prepared statements                              │    │
│  │  - Materialized views (analytics)                   │    │
│  │  - pg_stat_statements monitoring                    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### Implementacion Redis

```typescript
// cache-service.ts
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL,
  token: process.env.UPSTASH_REDIS_TOKEN,
});

// Cache patterns
export const CachePatterns = {
  // LLM Response caching (costoso, cache agresivo)
  async cacheLLMResponse(
    prompt: string,
    response: string,
    ttl: number = 7 * 24 * 60 * 60 // 7 dias
  ) {
    const key = `llm:${hashPrompt(prompt)}`;
    await redis.setex(key, ttl, JSON.stringify({
      response,
      timestamp: Date.now(),
    }));
  },

  // Matching scores (cambia con nuevos CVs/vacantes)
  async cacheMatchingScore(
    cvId: string,
    jobId: string,
    score: number,
    ttl: number = 3600 // 1 hora
  ) {
    const key = `match:${cvId}:${jobId}`;
    await redis.setex(key, ttl, score.toString());
  },

  // Session cache (acceso frecuente)
  async cacheSession(
    sessionId: string,
    data: SessionData,
    ttl: number = 86400 // 24 horas
  ) {
    const key = `session:${sessionId}`;
    await redis.setex(key, ttl, JSON.stringify(data));
  },

  // Rate limiting
  async checkRateLimit(
    userId: string,
    action: string,
    limit: number,
    windowSeconds: number
  ): Promise<boolean> {
    const key = `ratelimit:${action}:${userId}`;
    const current = await redis.incr(key);
    if (current === 1) {
      await redis.expire(key, windowSeconds);
    }
    return current <= limit;
  },
};
```

#### CDN Configuration

```typescript
// next.config.js - Vercel Edge caching
module.exports = {
  async headers() {
    return [
      {
        source: '/api/jobs/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 's-maxage=300, stale-while-revalidate=600',
          },
        ],
      },
      {
        source: '/api/companies/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 's-maxage=600, stale-while-revalidate=1200',
          },
        ],
      },
      {
        source: '/_next/static/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
    ];
  },
};
```

---

### 5. Auto-scaling Policies

#### Metricas de Escalamiento

```yaml
# auto-scaling-policies.yaml
scaling_policies:
  api_service:
    metric: cpu_utilization
    target: 70%
    scale_out:
      threshold: 75%
      cooldown: 60s
      increment: 2
    scale_in:
      threshold: 40%
      cooldown: 300s
      decrement: 1
    limits:
      min: 2
      max: 20

  cv_processing_worker:
    metric: queue_depth
    target: 100 messages
    scale_out:
      threshold: 150 messages
      cooldown: 30s
      increment: 2
    scale_in:
      threshold: 20 messages
      cooldown: 600s
      decrement: 1
    limits:
      min: 1
      max: 10

  interview_service:
    metric: concurrent_sessions
    target: 50 per instance
    scale_out:
      threshold: 60 sessions
      cooldown: 30s
      increment: 1
    scale_in:
      threshold: 20 sessions
      cooldown: 300s
      decrement: 1
    limits:
      min: 2
      max: 15

  database_read_replicas:
    metric: read_latency_p95
    target: 50ms
    scale_out:
      threshold: 100ms
      action: add_read_replica
    limits:
      min: 1
      max: 3
```

#### Schedule-based Scaling (Peru timezone)

```yaml
# scheduled-scaling.yaml
scheduled_actions:
  - name: morning_peak_scaleup
    schedule: "cron(0 7 * * ? *)"  # 7 AM PET
    timezone: America/Lima
    action:
      min_capacity: 4
      desired_capacity: 6

  - name: business_hours
    schedule: "cron(0 9 * * ? *)"  # 9 AM PET
    timezone: America/Lima
    action:
      min_capacity: 6
      desired_capacity: 8

  - name: evening_scaledown
    schedule: "cron(0 22 * * ? *)"  # 10 PM PET
    timezone: America/Lima
    action:
      min_capacity: 2
      desired_capacity: 2

  - name: weekend_minimal
    schedule: "cron(0 0 ? * SAT *)"  # Saturday midnight
    timezone: America/Lima
    action:
      min_capacity: 1
      desired_capacity: 2
```

---

### 6. Bottleneck Mitigation Strategies

#### Bottleneck 1: LLM API Rate Limits y Costos

```
┌─────────────────────────────────────────────────────────────┐
│              LLM Request Flow                                │
│                                                              │
│  Request ──► Cache Check ──► Queue ──► Rate Limiter ──► LLM │
│     │            │            │            │            │    │
│     │            ▼            ▼            ▼            │    │
│     │       Hit? Return   Priority    Token Bucket     │    │
│     │                     Sorting      Control         │    │
│     │                                                  │    │
│     └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// llm-queue-manager.ts
import { Queue } from 'bullmq';

const llmQueue = new Queue('llm-requests', {
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
  },
});

// Tiered model routing
const ModelTiers = {
  FAST: 'claude-3-haiku',      // $0.25/1M tokens - quick responses
  BALANCED: 'claude-3-sonnet', // $3/1M tokens - most tasks
  POWERFUL: 'claude-3-opus',   // $15/1M tokens - complex analysis
};

export async function routeLLMRequest(
  task: string,
  complexity: 'low' | 'medium' | 'high',
  priority: number = 5
): Promise<string> {
  // 1. Check semantic cache
  const cached = await checkSemanticCache(task);
  if (cached) return cached;

  // 2. Select model tier
  const model = complexity === 'low' ? ModelTiers.FAST
    : complexity === 'medium' ? ModelTiers.BALANCED
    : ModelTiers.POWERFUL;

  // 3. Queue with priority
  const job = await llmQueue.add('llm-task', {
    task,
    model,
    timestamp: Date.now(),
  }, {
    priority,
    rateLimiterKey: model,
  });

  return job.waitUntilFinished();
}

// Rate limits by model
const rateLimits = {
  [ModelTiers.FAST]: { requests: 1000, window: 60 },
  [ModelTiers.BALANCED]: { requests: 100, window: 60 },
  [ModelTiers.POWERFUL]: { requests: 20, window: 60 },
};
```

**Cost Optimization:**

| Estrategia | Ahorro Estimado |
|------------|-----------------|
| Response caching | 40-60% |
| Tiered models | 30-50% |
| Prompt compression | 10-20% |
| Batch processing | 15-25% |

#### Bottleneck 2: CV Processing Queue

```
┌─────────────────────────────────────────────────────────────┐
│              CV Processing Pipeline                          │
│                                                              │
│  Upload ──► Validation ──► Parse ──► Extract ──► Vectorize  │
│    │           │            │          │           │         │
│    ▼           ▼            ▼          ▼           ▼         │
│  S3/R2     Format       PDF/DOCX    NER/LLM    pgvector     │
│            Check        Parser      Extraction  Embedding    │
│                                                              │
│  Async Queue (BullMQ/SQS):                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Priority Levels:                                   │    │
│  │  1. Premium users (immediate)                       │    │
│  │  2. Active job seekers (< 5 min)                   │    │
│  │  3. Bulk uploads (< 30 min)                        │    │
│  │  4. Re-processing (< 2 hours)                      │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// cv-processor.ts
export const cvProcessingConfig = {
  // Concurrent processing limits
  concurrency: {
    mvp: 5,
    growth: 20,
    scale: 100,
  },

  // Timeouts per stage
  timeouts: {
    upload: 30000,      // 30s
    validation: 5000,    // 5s
    parsing: 60000,      // 60s
    extraction: 120000,  // 2 min
    vectorization: 30000, // 30s
  },

  // Retry policies
  retries: {
    parsing: 3,
    extraction: 2,
    vectorization: 3,
  },

  // Worker auto-scaling
  scaling: {
    queueDepthTrigger: 100,
    scaleUpIncrement: 2,
    scaleDownDelay: 600000, // 10 min
  },
};
```

#### Bottleneck 3: Vector Search at Scale

```
┌─────────────────────────────────────────────────────────────┐
│           Vector Search Evolution                            │
│                                                              │
│  MVP (< 10K vectors):                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  pgvector with HNSW index                           │    │
│  │  - IVFFlat: lists = sqrt(n_vectors)                 │    │
│  │  - HNSW: m = 16, ef_construction = 64               │    │
│  │  - Query time: < 50ms                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Growth (10K - 1M vectors):                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  pgvector optimizado + caching                      │    │
│  │  - HNSW: m = 32, ef_construction = 128              │    │
│  │  - Partitioned tables by category                   │    │
│  │  - Result caching en Redis                          │    │
│  │  - Query time: < 100ms                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Scale (1M+ vectors):                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Hybrid: pgvector + Pinecone                        │    │
│  │  - Hot data: Pinecone (P1 pod)                      │    │
│  │  - Cold data: pgvector                              │    │
│  │  - Metadata filtering en PostgreSQL                 │    │
│  │  - Query time: < 50ms                               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- pgvector optimization for Growth phase
-- Create optimized HNSW index
CREATE INDEX CONCURRENTLY idx_cv_embeddings_hnsw
ON cv_embeddings
USING hnsw (embedding vector_cosine_ops)
WITH (m = 32, ef_construction = 128);

-- Partition by job category for faster filtering
CREATE TABLE cv_embeddings_tech PARTITION OF cv_embeddings
FOR VALUES IN ('software', 'data', 'devops', 'qa');

CREATE TABLE cv_embeddings_business PARTITION OF cv_embeddings
FOR VALUES IN ('sales', 'marketing', 'finance', 'hr');

-- Materialized view for common searches
CREATE MATERIALIZED VIEW top_matches_by_job AS
SELECT
  j.id as job_id,
  c.id as cv_id,
  1 - (c.embedding <=> j.embedding) as similarity
FROM jobs j
CROSS JOIN LATERAL (
  SELECT id, embedding
  FROM cv_embeddings
  ORDER BY embedding <=> j.embedding
  LIMIT 100
) c
WHERE j.status = 'active';

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY top_matches_by_job;
```

#### Bottleneck 4: Database Connections

```typescript
// connection-management.ts
export const connectionStrategies = {
  mvp: {
    // Supabase connection pooling (built-in)
    poolMode: 'transaction',
    maxConnections: 20,
    idleTimeout: 30000,
  },

  growth: {
    // Application-level pooling
    primary: {
      max: 30,
      min: 5,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    },
    replica: {
      max: 50,
      min: 10,
      idleTimeoutMillis: 60000,
    },
    // Route reads to replica
    readWriteSplit: true,
  },

  scale: {
    // PgBouncer with aggressive pooling
    pgbouncer: {
      poolMode: 'transaction',
      maxClientConn: 2000,
      defaultPoolSize: 25,
      reservePoolSize: 5,
      reservePoolTimeout: 3,
    },
    // Connection per service limits
    serviceLimits: {
      api: 40,
      workers: 20,
      analytics: 10,
      cron: 5,
    },
  },
};
```

#### Bottleneck 5: File Storage (CVs, Recordings)

```
┌─────────────────────────────────────────────────────────────┐
│              File Storage Architecture                       │
│                                                              │
│  MVP: Supabase Storage                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Direct upload to Supabase Storage                │    │
│  │  - 50MB max file size                               │    │
│  │  - Basic CDN via Supabase                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Growth: Cloudflare R2 + CDN                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - Presigned URLs for direct upload                 │    │
│  │  - Cloudflare CDN (global edge)                     │    │
│  │  - Lifecycle policies (archive after 90 days)       │    │
│  │  - Image/PDF optimization on-the-fly                │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  Scale: S3 + CloudFront + Glacier                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  - S3 Standard: active files                        │    │
│  │  - S3 Intelligent-Tiering: automatic optimization   │    │
│  │  - S3 Glacier: archived CVs (> 1 year inactive)     │    │
│  │  - CloudFront with signed URLs                      │    │
│  │  - Lambda@Edge for on-demand processing             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// storage-service.ts
export const storageConfig = {
  mvp: {
    provider: 'supabase',
    bucket: 'cvs',
    maxFileSize: 50 * 1024 * 1024, // 50MB
    allowedTypes: ['application/pdf', 'application/msword',
                   'application/vnd.openxmlformats-officedocument.wordprocessingml.document'],
  },

  growth: {
    provider: 'cloudflare-r2',
    buckets: {
      cvs: {
        publicAccess: false,
        lifecycleRules: [
          { action: 'transition', days: 90, storage: 'infrequent' },
        ],
      },
      recordings: {
        publicAccess: false,
        lifecycleRules: [
          { action: 'transition', days: 30, storage: 'infrequent' },
          { action: 'delete', days: 365 },
        ],
      },
    },
    cdn: {
      provider: 'cloudflare',
      cacheControl: 'private, max-age=3600',
    },
  },

  scale: {
    provider: 'aws-s3',
    buckets: {
      cvs: {
        storageClass: 'INTELLIGENT_TIERING',
        versioning: true,
        encryption: 'AES256',
      },
      recordings: {
        storageClass: 'STANDARD',
        lifecycleRules: [
          { transition: { days: 30, storageClass: 'GLACIER_IR' } },
          { expiration: { days: 730 } }, // 2 years
        ],
      },
    },
    cdn: {
      provider: 'cloudfront',
      signedUrls: true,
      originAccessControl: true,
    },
  },
};
```

---

### 7. Cost Analysis by Phase

#### MVP Phase (Mes 1-3): Target < $500/mes

| Servicio | Provider | Plan | Costo/mes |
|----------|----------|------|-----------|
| Frontend + API | Vercel | Pro | $20 |
| Database | Supabase | Pro | $25 |
| Auth | Supabase | Included | $0 |
| Storage | Supabase | Pro (8GB) | Included |
| Cache/Queue | Upstash Redis | Free/Pay-as-go | $10 |
| LLM API | Anthropic | Pay-as-go | $200-300 |
| Monitoring | Vercel Analytics | Included | $0 |
| Domain + SSL | Cloudflare | Free | $0 |
| Email | Resend | Free tier | $0 |
| **Total** | | | **$255-355** |

#### Growth Phase (Mes 6-12): Target < $2,000/mes

| Servicio | Provider | Plan | Costo/mes |
|----------|----------|------|-----------|
| Frontend + API | Vercel | Pro | $20 |
| Database | Supabase | Team | $599 |
| Read Replica | Supabase | Add-on | $100 |
| Storage | Cloudflare R2 | Pay-as-go | $50 |
| Cache | Upstash Redis | Pro | $50 |
| Queue | Upstash QStash | Pro | $30 |
| LLM API | Anthropic | Pay-as-go | $800 |
| Monitoring | Datadog | Pro | $100 |
| CDN | Cloudflare | Pro | $20 |
| Email | Resend | Pro | $20 |
| **Total** | | | **$1,789** |

#### Scale Phase (Ano 2+): Target < $10,000/mes

| Servicio | Provider | Plan | Costo/mes |
|----------|----------|------|-----------|
| Compute | AWS ECS Fargate | On-demand | $1,500 |
| Database | AWS RDS | db.r6g.xlarge | $800 |
| Read Replicas (2) | AWS RDS | db.r6g.large x2 | $600 |
| Cache | AWS ElastiCache | r6g.large | $300 |
| Storage | AWS S3 | Standard + IA | $200 |
| CDN | AWS CloudFront | Standard | $300 |
| Queue | AWS SQS | Standard | $50 |
| Vector DB | Pinecone | S1 | $70 |
| LLM API | Anthropic | Enterprise | $4,000 |
| Monitoring | Datadog | Enterprise | $500 |
| WAF + Shield | AWS | Standard | $200 |
| Secrets | AWS Secrets Manager | Standard | $50 |
| **Total** | | | **$8,570** |

#### Cost Optimization Strategies

```typescript
// cost-optimization.ts
export const costOptimizations = {
  // 1. Reserved capacity (30-40% savings)
  reservations: {
    compute: '1-year reserved instances for base load',
    database: '1-year reserved RDS for primary',
    cache: 'Reserved ElastiCache nodes',
  },

  // 2. Spot instances for workers (70% savings)
  spotInstances: {
    cvProcessing: true,
    batchAnalytics: true,
    reindexing: true,
  },

  // 3. Intelligent tiering
  storageTiering: {
    hot: 'S3 Standard (0-30 days)',
    warm: 'S3 Intelligent-Tiering (30-90 days)',
    cold: 'S3 Glacier IR (90+ days)',
    archive: 'S3 Glacier Deep Archive (1+ year)',
  },

  // 4. Right-sizing
  rightSizing: {
    frequency: 'monthly',
    metrics: ['cpu_utilization', 'memory_utilization', 'network_throughput'],
    target: '70% average utilization',
  },

  // 5. LLM cost control
  llmCostControl: {
    caching: 'Semantic caching for similar prompts',
    tiering: 'Use Haiku for simple tasks, Sonnet for complex',
    batching: 'Batch similar requests',
    compression: 'Prompt optimization',
  },
};
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Costo controlado** | Arquitectura serverless minimiza costos en baja carga |
| **Escalabilidad predecible** | Path claro de MVP -> Growth -> Scale |
| **Operaciones simplificadas** | Managed services reducen overhead operacional |
| **Performance global** | Edge computing y CDN desde el inicio |
| **Vendor flexibility** | Migracion planificada permite cambiar providers |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Cold starts en MVP** | Acceptable para carga baja, migracion a containers en Growth |
| **Vendor lock-in parcial** | Abstracciones en codigo, interfaces estandar |
| **Complejidad de migracion** | Plan detallado, migracion gradual con feature flags |
| **Costos LLM impredecibles** | Caching agresivo, tiered models, budgets y alertas |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Costos LLM exceden budget | Media | Alto | Caching, tiering, rate limits, alertas |
| Latencia en picos | Baja | Medio | Auto-scaling, queue management |
| Migracion compleja | Media | Medio | Planificacion detallada, rollback strategy |
| Database bottleneck | Baja | Alto | Monitoring proactivo, read replicas |

---

## Migration Triggers

### De MVP a Growth

Activar cuando:
- Usuarios activos > 2,000
- Entrevistas/mes > 5,000
- Costos Supabase > $100/mes
- Latencia P95 > 500ms
- Queue depth promedio > 50

### De Growth a Scale

Activar cuando:
- Usuarios activos > 20,000
- Entrevistas/mes > 100,000
- Costos totales > $3,000/mes
- Necesidad de compliance (SOC2, GDPR)
- Expansion multi-region requerida

---

## Plan de Implementacion

### Fase 1: MVP Setup (Semana 1-2)

```bash
# Setup inicial
1. Vercel project creation
2. Supabase project + schema
3. Upstash Redis setup
4. Anthropic API integration
5. Basic monitoring setup
```

### Fase 2: Growth Optimization (Mes 6)

```bash
# Optimizaciones
1. Cloudflare R2 migration
2. Read replica activation
3. Redis caching implementation
4. Queue system (BullMQ/QStash)
5. Auto-scaling policies
```

### Fase 3: Scale Migration (Ano 2)

```bash
# AWS Migration
1. Infrastructure as Code (Terraform)
2. ECS Fargate deployment
3. RDS migration with replication
4. CloudFront + S3 setup
5. Pinecone integration
6. Full observability stack
```

---

## Metricas de Exito

| Metrica | MVP | Growth | Scale |
|---------|-----|--------|-------|
| Latencia P50 | < 100ms | < 80ms | < 50ms |
| Latencia P95 | < 500ms | < 300ms | < 200ms |
| Disponibilidad | 99% | 99.5% | 99.9% |
| Costo por usuario | < $1 | < $0.50 | < $0.25 |
| Costo por entrevista | < $0.50 | < $0.15 | < $0.05 |
| Time to scale | N/A | < 2 min | < 30s |

---

## Referencias

- [Vercel Scaling Documentation](https://vercel.com/docs/concepts/limits/overview)
- [Supabase Architecture](https://supabase.com/docs/guides/platform/performance)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [pgvector Performance Tuning](https://github.com/pgvector/pgvector)
- [Pinecone Scaling Guide](https://docs.pinecone.io/docs/scaling)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
