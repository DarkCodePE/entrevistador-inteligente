# Arquitectura - Entrevistador Inteligente
## Architecture Decision Records (ADRs) con Domain-Driven Design

**Generado:** 31 Enero 2026
**Metodología:** DDD (Domain-Driven Design) + USACF Framework
**Total ADRs:** 15 documentos (~600KB)

---

## Resumen de Decisiones Clave

| Decisión | Selección | Justificación |
|----------|-----------|---------------|
| **Arquitectura** | Modular Monolith | Equipo pequeño, time-to-market |
| **Backend** | Node.js + TypeScript | Ecosystem, team experience |
| **Frontend** | Next.js 14 + PWA | Mobile-first Peru |
| **Database** | PostgreSQL + pgvector | DDD schemas, vector search |
| **Cache** | Redis | Events, sessions, rate limiting |
| **LLM** | Claude 3.5 Sonnet + GPT-4o-mini | Spanish quality, cost optimization |
| **Auth** | Supabase Auth | Cost-effective, RLS |
| **Payments** | Culqi + Nubefact | Yape/Plin, SUNAT compliance |
| **Cloud** | Vercel + Supabase (MVP) → AWS | Simplicity then scale |
| **Observability** | Grafana Stack | Open source, $70-100/month |

---

## Índice de ADRs

### Arquitectura Core

| ADR | Título | Descripción |
|-----|--------|-------------|
| [ADR-001](adrs/ADR-001-architecture-style.md) | Estilo de Arquitectura | Modular Monolith vs Microservices |
| [ADR-002](adrs/ADR-002-bounded-contexts.md) | Bounded Contexts (DDD) | 6 contextos, Context Map, agregados |
| [ADR-003](adrs/ADR-003-event-driven.md) | Event-Driven | Redis Streams, eventos de dominio |

### Infraestructura de Datos

| ADR | Título | Descripción |
|-----|--------|-------------|
| [ADR-004](adrs/ADR-004-ai-ml-infrastructure.md) | AI/ML Infrastructure | LLMs, embeddings, vector DB |
| [ADR-005](adrs/ADR-005-data-storage.md) | Data Storage | PostgreSQL, Redis, R2, schemas |

### Seguridad y Auth

| ADR | Título | Descripción |
|-----|--------|-------------|
| [ADR-006](adrs/ADR-006-authentication.md) | Authentication | Supabase Auth, JWT, WhatsApp OTP |
| [ADR-013](adrs/ADR-013-privacy-compliance.md) | Privacy Compliance | LPDP Perú, encryption, ARCO |
| [ADR-014](adrs/ADR-014-multi-tenancy.md) | Multi-tenancy B2B | RLS, SSO, tiers empresariales |

### Módulos de Negocio

| ADR | Título | Descripción |
|-----|--------|-------------|
| [ADR-007](adrs/ADR-007-cv-pipeline.md) | CV Pipeline | Parsing, OCR, embeddings, ATS score |
| [ADR-008](adrs/ADR-008-interview-engine.md) | Interview Engine | Motor IA, scoring, feedback |
| [ADR-009](adrs/ADR-009-payment-integration.md) | Payment Integration | Culqi, Yape/Plin, facturación |

### Frontend y APIs

| ADR | Título | Descripción |
|-----|--------|-------------|
| [ADR-010](adrs/ADR-010-api-design.md) | API Design | REST + GraphQL + WebSocket |
| [ADR-011](adrs/ADR-011-frontend-strategy.md) | Frontend Strategy | Next.js, PWA, mobile-first |

### Operaciones

| ADR | Título | Descripción |
|-----|--------|-------------|
| [ADR-012](adrs/ADR-012-observability.md) | Observability | Grafana stack, alertas |
| [ADR-015](adrs/ADR-015-scalability.md) | Scalability | Auto-scaling, caching, migration path |

---

## Bounded Contexts (DDD)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ENTREVISTADOR INTELIGENTE                        │
├─────────────┬─────────────┬─────────────┬─────────────┬─────────────┤
│  IDENTITY   │     CV      │   MATCHING  │  INTERVIEW  │   BILLING   │
│   Context   │  MANAGEMENT │   Context   │   Context   │   Context   │
│             │   Context   │             │             │             │
│ ─────────── │ ─────────── │ ─────────── │ ─────────── │ ─────────── │
│ User        │ CV          │ CandidateP  │ Interview   │ Subscription│
│ Company     │ CVVersion   │ JobPosting  │ Question    │ Payment     │
│ Role        │ Skill       │ Match       │ Response    │ Invoice     │
│ Session     │ CVEmbedding │ Application │ Feedback    │ Plan        │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
                                    │
                            ┌───────┴───────┐
                            │   ANALYTICS   │
                            │    Context    │
                            │ ───────────── │
                            │ Event         │
                            │ Metric        │
                            │ Report        │
                            └───────────────┘
```

---

## Stack Tecnológico

### Backend
```
Node.js 20+ / TypeScript 5.x
├── Framework: NestJS (modular)
├── ORM: Prisma
├── Queue: BullMQ + Redis
├── Events: Redis Streams
└── API: REST + GraphQL + WebSocket
```

### Frontend
```
Next.js 14+ / React 18
├── State: Zustand + React Query
├── UI: Tailwind + shadcn/ui
├── PWA: next-pwa
└── Real-time: Socket.io
```

### Data
```
PostgreSQL 16+
├── pgvector (embeddings)
├── RLS (multi-tenancy)
├── Schemas por contexto DDD
│
Redis 7.x
├── Cache
├── Sessions
├── Events
├── Rate limiting
│
Cloudflare R2
└── Files (CVs, recordings)
```

### AI/ML
```
LLMs
├── Claude 3.5 Sonnet (primary)
├── GPT-4o-mini (fallback)
└── Whisper (speech, v2)

Embeddings
└── OpenAI text-embedding-3-small

Vector Search
└── pgvector → Qdrant (scale)
```

---

## Costos Proyectados

| Fase | Infra | AI/ML | Total/mes |
|------|-------|-------|-----------|
| **MVP** (M1-6) | $255-355 | $441 | ~$700-800 |
| **Growth** (M7-12) | $1,789 | $1,505 | ~$3,300 |
| **Scale** (Y2+) | $8,570 | $4,870 | ~$13,500 |

---

## Timeline de Implementación

```
Semana 1-2:  Setup proyecto, auth, database schemas
Semana 3-4:  CV pipeline (upload, parsing, embeddings)
Semana 5-6:  Matching engine, job recommendations
Semana 7-8:  Interview simulator (texto MVP)
Semana 9-10: Billing, subscriptions, payments
Semana 11-12: Testing, polish, launch beta
```

---

## Documentos Relacionados

- [Investigación de Mercado](../market-research.md)
- [Análisis de Gaps](../gap-analysis.md)
- [Modelo de Negocio](../business-model.md)
- [Análisis de Riesgos](../risk-analysis.md)
- [Resumen Ejecutivo](../EXECUTIVE-SUMMARY.md)

---

*Arquitectura diseñada con Claude Flow V3 + USACF Framework*
