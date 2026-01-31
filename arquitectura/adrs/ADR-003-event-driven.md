# ADR-003: Arquitectura Orientada a Eventos

**Estado:** Aceptado
**Fecha:** 2026-01-31
**Autores:** System Architecture Team
**Revisores:** Tech Lead, CTO

---

## Contexto

El sistema Entrevistador Inteligente esta compuesto por multiples Bounded Contexts que necesitan coordinarse de manera eficiente:

- **Identity Context**: Gestion de usuarios, autenticacion, suscripciones
- **CV Context**: Upload, parsing, optimizacion y scoring de CVs
- **Matching Context**: Algoritmos de matching candidato-oferta
- **Interview Context**: Simulacion de entrevistas con IA, feedback
- **Billing Context**: Suscripciones, pagos, facturacion

### Caracteristicas del Sistema

| Aspecto | Descripcion |
|---------|-------------|
| Operaciones asincronas | CV parsing (5-30s), AI interviews (5-30min), matching batch |
| Volumen esperado Y1 | 50K usuarios, 200K entrevistas, 100K CVs |
| Latencia requerida | Real-time para UX, batch para procesamiento pesado |
| Equipo | 3-5 developers, experiencia moderada en sistemas distribuidos |
| Presupuesto infra | $3,000-8,000/mes inicial |

### Problemas a Resolver

1. **Acoplamiento temporal**: Servicios bloqueados esperando respuestas sincronas
2. **Escalabilidad independiente**: CV parsing necesita escalar diferente que matching
3. **Resiliencia**: Falla en billing no debe afectar entrevistas
4. **Auditoria**: Necesidad de tracking completo del journey del usuario
5. **Integracion futura**: Webhooks para empresas, integraciones con ATS

---

## Decision

### 1. Arquitectura Event-Driven (sin Event Sourcing completo)

Adoptamos una arquitectura orientada a eventos utilizando el patron **Event Notification** para la comunicacion entre Bounded Contexts, sin implementar Event Sourcing completo en esta fase.

**Justificacion:**

| Aspecto | Event Sourcing | Event-Driven Simple | Decision |
|---------|----------------|---------------------|----------|
| Complejidad | Alta | Media | Simple |
| Curva aprendizaje | 6-12 meses | 1-3 meses | Simple |
| Reconstruccion estado | Completa | No disponible | No critico para MVP |
| Debugging | Complejo (replay) | Tradicional + logs | Tradicional |
| Storage | Alto (todos eventos) | Bajo (estado actual) | Eficiente |
| Equipo requerido | Senior DDD | Mid-level | Disponible |

**Event Sourcing se evaluara en Fase 2** cuando:
- Volumen > 500K usuarios
- Requisitos de auditoria regulatoria
- Necesidad de replay/debugging complejo

### 2. Event Broker: Redis Streams

Seleccionamos **Redis Streams** como message broker principal.

**Evaluacion de Alternativas:**

| Criterio | Redis Streams | RabbitMQ | Apache Kafka | Amazon SQS |
|----------|---------------|----------|--------------|------------|
| **Complejidad operacional** | Baja | Media | Alta | Muy Baja |
| **Costo mensual (estimado)** | $50-150 | $100-300 | $500-2000 | $50-200 |
| **Latencia** | Sub-ms | 1-5ms | 5-20ms | 10-50ms |
| **Throughput** | 100K+ msg/s | 50K+ msg/s | 1M+ msg/s | 3K msg/s |
| **Persistencia** | Configurable | Si | Si | Si |
| **Consumer Groups** | Si | Si | Si | No nativo |
| **Exactamente una vez** | No nativo | Si | Si | No |
| **Experiencia equipo** | Alta | Media | Baja | Alta |
| **Ya en stack (cache)** | Si | No | No | No |

**Decision: Redis Streams** porque:

1. **Ya esta en el stack**: Usamos Redis para cache y sessions
2. **Simplicidad operacional**: Un solo sistema que mantener
3. **Costo**: Reutilizamos infraestructura existente
4. **Rendimiento**: Suficiente para volumen MVP (100K+ msg/s)
5. **Consumer Groups**: Soporte nativo para multiples consumidores

**Limitaciones aceptadas:**
- Sin garantia "exactly-once" nativa (mitigado con idempotencia)
- Persistencia limitada vs Kafka (suficiente para MVP)
- Escalamiento horizontal mas complejo que Kafka

**Plan de migracion a Kafka:** Cuando throughput > 50K eventos/hora sostenido

### 3. Patrones de Comunicacion

#### 3.1 Outbox Pattern (Transactional Outbox)

Para garantizar consistencia entre la base de datos y los eventos publicados:

```
┌─────────────────────────────────────────────────────────────────┐
│                      OUTBOX PATTERN                              │
└─────────────────────────────────────────────────────────────────┘

  Service                          PostgreSQL                Redis Streams
     │                                 │                          │
     │  1. Begin Transaction           │                          │
     │ ───────────────────────────────►│                          │
     │                                 │                          │
     │  2. INSERT/UPDATE entity        │                          │
     │ ───────────────────────────────►│                          │
     │                                 │                          │
     │  3. INSERT event to outbox      │                          │
     │ ───────────────────────────────►│                          │
     │                                 │                          │
     │  4. Commit Transaction          │                          │
     │ ───────────────────────────────►│                          │
     │                                 │                          │
     │                                 │   5. Outbox Processor    │
     │                                 │      (polling/CDC)       │
     │                                 │ ────────────────────────►│
     │                                 │                          │
     │                                 │   6. Mark as published   │
     │                                 │◄─────────────────────────│
```

**Implementacion:**

```sql
-- Tabla outbox en cada servicio
CREATE TABLE event_outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP NULL,
    retry_count INT DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished ON event_outbox(created_at)
    WHERE published_at IS NULL;
```

#### 3.2 Saga Pattern (Coreografiado)

Para operaciones que cruzan multiples Bounded Contexts, usamos Sagas coreografiadas:

```
┌─────────────────────────────────────────────────────────────────┐
│              SAGA: Usuario Premium - Subscripcion                │
└─────────────────────────────────────────────────────────────────┘

 Identity          Billing           CV              Interview
    │                 │               │                  │
    │ UserUpgradeToPremiumRequested   │                  │
    │────────────────►│               │                  │
    │                 │               │                  │
    │                 │ PaymentProcessed                 │
    │                 │──────────────►│                  │
    │                 │               │                  │
    │                 │               │ CVQuotaIncreased │
    │                 │               │─────────────────►│
    │                 │               │                  │
    │                 │               │    InterviewQuotaIncreased
    │◄────────────────┼───────────────┼──────────────────│
    │                 │               │                  │
    │ UserUpgradedToPremium (final)   │                  │
    │────────────────────────────────────────────────────►
```

**Compensacion en caso de falla:**

```
┌─────────────────────────────────────────────────────────────────┐
│           SAGA COMPENSACION: Payment Failed                      │
└─────────────────────────────────────────────────────────────────┘

 Identity          Billing           CV              Interview
    │                 │               │                  │
    │ UserUpgradeToPremiumRequested   │                  │
    │────────────────►│               │                  │
    │                 │               │                  │
    │                 │ PaymentFailed │                  │
    │◄────────────────│               │                  │
    │                 │               │                  │
    │ UserUpgradeRolledBack           │                  │
    │──────────────────────────────────────────────────────►
    │                 │               │                  │
    │ [Notificar usuario]             │                  │
```

#### 3.3 Event Notification (Fire and Forget)

Para eventos informativos que no requieren compensacion:

```
CVParsed ──► Matching (actualiza vectores)
         ──► Analytics (registra metricas)
         ──► Notifications (opcional email)
```

---

## Eventos de Dominio

### Catalogo de Eventos por Bounded Context

#### Identity Context

| Evento | Payload | Consumidores | Criticidad |
|--------|---------|--------------|------------|
| `UserRegistered` | userId, email, registrationSource | CV, Analytics, Notifications | Alta |
| `UserVerified` | userId, verificationMethod | Billing, CV | Alta |
| `UserUpgradedToPremium` | userId, planId, validUntil | CV, Interview, Notifications | Alta |
| `UserDowngraded` | userId, previousPlan, reason | CV, Interview | Media |
| `UserDeactivated` | userId, reason | All contexts | Alta |

**Esquema UserRegistered:**

```json
{
  "eventId": "uuid-v4",
  "eventType": "identity.user.registered",
  "eventVersion": "1.0",
  "timestamp": "2026-01-31T10:30:00Z",
  "aggregateId": "user-123",
  "aggregateType": "User",
  "payload": {
    "userId": "user-123",
    "email": "juan@example.com",
    "registrationSource": "google_oauth",
    "locale": "es-PE",
    "referralCode": "REF-456"
  },
  "metadata": {
    "correlationId": "req-789",
    "causationId": "cmd-012",
    "userAgent": "Mozilla/5.0...",
    "ipCountry": "PE"
  }
}
```

#### CV Context

| Evento | Payload | Consumidores | Criticidad |
|--------|---------|--------------|------------|
| `CVUploaded` | cvId, userId, format, size | CV Processor | Alta |
| `CVParsed` | cvId, extractedData, confidence | Matching, Analytics | Alta |
| `CVParsingFailed` | cvId, errorCode, errorMessage | Notifications | Media |
| `CVOptimized` | cvId, suggestions, score | Notifications | Media |
| `CVScoreCalculated` | cvId, overallScore, dimensions | Matching, Analytics | Alta |

**Esquema CVParsed:**

```json
{
  "eventId": "uuid-v4",
  "eventType": "cv.document.parsed",
  "eventVersion": "1.0",
  "timestamp": "2026-01-31T10:31:00Z",
  "aggregateId": "cv-456",
  "aggregateType": "CV",
  "payload": {
    "cvId": "cv-456",
    "userId": "user-123",
    "extractedData": {
      "personalInfo": {
        "name": "Juan Perez",
        "email": "juan@example.com",
        "phone": "+51987654321",
        "location": "Lima, Peru"
      },
      "experience": [
        {
          "company": "Tech Corp",
          "role": "Software Engineer",
          "startDate": "2022-01",
          "endDate": "2025-12",
          "description": "..."
        }
      ],
      "education": [...],
      "skills": ["Python", "AWS", "PostgreSQL"],
      "languages": [{"language": "Spanish", "level": "native"}]
    },
    "parsingConfidence": 0.92,
    "processingTimeMs": 3500
  },
  "metadata": {
    "correlationId": "req-789",
    "parserVersion": "2.1.0",
    "ocrProvider": "azure"
  }
}
```

#### Matching Context

| Evento | Payload | Consumidores | Criticidad |
|--------|---------|--------------|------------|
| `JobMatchesGenerated` | userId, matches[], algorithm | Notifications | Alta |
| `MatchAccepted` | matchId, userId, jobId | Analytics, Billing | Alta |
| `MatchRejected` | matchId, userId, jobId, reason | Analytics | Baja |
| `ApplicationSubmitted` | applicationId, userId, jobId | Notifications, Analytics | Alta |
| `ApplicationStatusChanged` | applicationId, newStatus | Notifications | Media |

#### Interview Context

| Evento | Payload | Consumidores | Criticidad |
|--------|---------|--------------|------------|
| `InterviewStarted` | interviewId, userId, config | Analytics | Media |
| `InterviewCompleted` | interviewId, duration, questionsCount | Billing, Analytics | Alta |
| `InterviewAbandoned` | interviewId, atQuestion, reason | Analytics | Baja |
| `FeedbackGenerated` | interviewId, scores, recommendations | CV, Notifications | Alta |
| `SkillScoreUpdated` | userId, skill, newScore, delta | Matching, Analytics | Media |

**Esquema InterviewCompleted:**

```json
{
  "eventId": "uuid-v4",
  "eventType": "interview.session.completed",
  "eventVersion": "1.0",
  "timestamp": "2026-01-31T11:00:00Z",
  "aggregateId": "interview-789",
  "aggregateType": "Interview",
  "payload": {
    "interviewId": "interview-789",
    "userId": "user-123",
    "jobType": "software_engineer",
    "difficulty": "mid",
    "duration": {
      "totalSeconds": 1800,
      "activeSeconds": 1650
    },
    "questionsAnswered": 12,
    "questionsTotal": 15,
    "scores": {
      "overall": 78,
      "content": 82,
      "communication": 75,
      "technical": 80
    },
    "creditsUsed": 1,
    "interviewType": "ai_simulation"
  },
  "metadata": {
    "correlationId": "session-456",
    "llmModel": "gpt-4",
    "llmTokensUsed": 12500
  }
}
```

#### Billing Context

| Evento | Payload | Consumidores | Criticidad |
|--------|---------|--------------|------------|
| `SubscriptionCreated` | subscriptionId, userId, planId | Identity, CV, Interview | Alta |
| `PaymentProcessed` | paymentId, subscriptionId, amount | Identity, Notifications | Alta |
| `PaymentFailed` | paymentId, subscriptionId, errorCode | Identity, Notifications | Alta |
| `SubscriptionRenewed` | subscriptionId, newValidUntil | Identity | Alta |
| `SubscriptionCancelled` | subscriptionId, reason, effectiveDate | Identity, CV, Interview | Alta |

---

## Topologia de Streams

```
┌─────────────────────────────────────────────────────────────────┐
│                     REDIS STREAMS TOPOLOGY                       │
└─────────────────────────────────────────────────────────────────┘

                        ┌─────────────────┐
                        │  identity.events │
                        │  (stream)        │
                        └────────┬────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ cv-service      │    │ billing-service │    │ analytics-svc   │
│ (consumer group)│    │ (consumer group)│    │ (consumer group)│
└─────────────────┘    └─────────────────┘    └─────────────────┘


                        ┌─────────────────┐
                        │   cv.events     │
                        │   (stream)      │
                        └────────┬────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ matching-svc    │    │ notification-svc│    │ analytics-svc   │
│ (consumer group)│    │ (consumer group)│    │ (consumer group)│
└─────────────────┘    └─────────────────┘    └─────────────────┘


                        ┌─────────────────┐
                        │ interview.events│
                        │ (stream)        │
                        └────────┬────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ billing-service │    │ cv-service      │    │ analytics-svc   │
│ (consumer group)│    │ (consumer group)│    │ (consumer group)│
└─────────────────┘    └─────────────────┘    └─────────────────┘


                        ┌─────────────────┐
                        │  billing.events │
                        │  (stream)       │
                        └────────┬────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ identity-service│    │ cv-service      │    │ interview-svc   │
│ (consumer group)│    │ (consumer group)│    │ (consumer group)│
└─────────────────┘    └─────────────────┘    └─────────────────┘


                        ┌─────────────────┐
                        │ matching.events │
                        │ (stream)        │
                        └────────┬────────┘
                                 │
         ┌───────────────────────┤
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│ notification-svc│    │ analytics-svc   │
│ (consumer group)│    │ (consumer group)│
└─────────────────┘    └─────────────────┘
```

### Configuracion de Streams

```bash
# Crear streams con retention de 7 dias
XGROUP CREATE identity.events identity-consumers $ MKSTREAM
XGROUP CREATE cv.events cv-consumers $ MKSTREAM
XGROUP CREATE interview.events interview-consumers $ MKSTREAM
XGROUP CREATE billing.events billing-consumers $ MKSTREAM
XGROUP CREATE matching.events matching-consumers $ MKSTREAM

# Configurar maxlen (limitar eventos en memoria)
# ~100000 permite aproximacion para eficiencia
CONFIG SET stream-node-max-bytes 4096
```

---

## Implementacion

### Event Publisher (TypeScript/Node.js)

```typescript
// src/shared/events/event-publisher.ts

import { Redis } from 'ioredis';
import { v4 as uuidv4 } from 'uuid';

interface DomainEvent<T = unknown> {
  eventId: string;
  eventType: string;
  eventVersion: string;
  timestamp: string;
  aggregateId: string;
  aggregateType: string;
  payload: T;
  metadata: EventMetadata;
}

interface EventMetadata {
  correlationId: string;
  causationId?: string;
  userId?: string;
  [key: string]: unknown;
}

class EventPublisher {
  private redis: Redis;
  private serviceName: string;

  constructor(redis: Redis, serviceName: string) {
    this.redis = redis;
    this.serviceName = serviceName;
  }

  async publish<T>(
    streamName: string,
    eventType: string,
    aggregateId: string,
    aggregateType: string,
    payload: T,
    metadata: Partial<EventMetadata> = {}
  ): Promise<string> {
    const event: DomainEvent<T> = {
      eventId: uuidv4(),
      eventType,
      eventVersion: '1.0',
      timestamp: new Date().toISOString(),
      aggregateId,
      aggregateType,
      payload,
      metadata: {
        correlationId: metadata.correlationId || uuidv4(),
        ...metadata,
        publishedBy: this.serviceName,
      },
    };

    const messageId = await this.redis.xadd(
      streamName,
      'MAXLEN', '~', '100000',
      '*',
      'event', JSON.stringify(event)
    );

    return messageId;
  }

  // Publish with outbox pattern (transactional)
  async publishWithOutbox<T>(
    db: Database, // Knex or Prisma instance
    streamName: string,
    eventType: string,
    aggregateId: string,
    aggregateType: string,
    payload: T,
    metadata: Partial<EventMetadata> = {}
  ): Promise<void> {
    const event: DomainEvent<T> = {
      eventId: uuidv4(),
      eventType,
      eventVersion: '1.0',
      timestamp: new Date().toISOString(),
      aggregateId,
      aggregateType,
      payload,
      metadata: {
        correlationId: metadata.correlationId || uuidv4(),
        ...metadata,
        publishedBy: this.serviceName,
      },
    };

    // Insert into outbox table (same transaction as business logic)
    await db('event_outbox').insert({
      id: event.eventId,
      aggregate_type: aggregateType,
      aggregate_id: aggregateId,
      event_type: eventType,
      stream_name: streamName,
      payload: JSON.stringify(event),
    });
  }
}

export { EventPublisher, DomainEvent, EventMetadata };
```

### Event Consumer (TypeScript/Node.js)

```typescript
// src/shared/events/event-consumer.ts

import { Redis } from 'ioredis';
import { DomainEvent } from './event-publisher';

type EventHandler<T = unknown> = (event: DomainEvent<T>) => Promise<void>;

interface ConsumerConfig {
  streamName: string;
  groupName: string;
  consumerName: string;
  handlers: Map<string, EventHandler>;
  batchSize?: number;
  blockMs?: number;
}

class EventConsumer {
  private redis: Redis;
  private config: ConsumerConfig;
  private running: boolean = false;

  constructor(redis: Redis, config: ConsumerConfig) {
    this.redis = redis;
    this.config = {
      batchSize: 10,
      blockMs: 5000,
      ...config,
    };
  }

  async start(): Promise<void> {
    this.running = true;

    // Ensure consumer group exists
    try {
      await this.redis.xgroup(
        'CREATE',
        this.config.streamName,
        this.config.groupName,
        '$',
        'MKSTREAM'
      );
    } catch (err: any) {
      if (!err.message.includes('BUSYGROUP')) {
        throw err;
      }
    }

    await this.consumeLoop();
  }

  async stop(): Promise<void> {
    this.running = false;
  }

  private async consumeLoop(): Promise<void> {
    while (this.running) {
      try {
        const messages = await this.redis.xreadgroup(
          'GROUP',
          this.config.groupName,
          this.config.consumerName,
          'COUNT',
          this.config.batchSize!,
          'BLOCK',
          this.config.blockMs!,
          'STREAMS',
          this.config.streamName,
          '>'
        );

        if (messages) {
          for (const [, streamMessages] of messages) {
            for (const [messageId, fields] of streamMessages) {
              await this.processMessage(messageId, fields);
            }
          }
        }
      } catch (error) {
        console.error('Error consuming events:', error);
        await this.sleep(1000);
      }
    }
  }

  private async processMessage(
    messageId: string,
    fields: string[]
  ): Promise<void> {
    const eventData = fields[1]; // ['event', '{...}']
    const event: DomainEvent = JSON.parse(eventData);

    const handler = this.config.handlers.get(event.eventType);

    if (handler) {
      try {
        await handler(event);
        // Acknowledge successful processing
        await this.redis.xack(
          this.config.streamName,
          this.config.groupName,
          messageId
        );
      } catch (error) {
        console.error(`Error processing event ${event.eventId}:`, error);
        // Event will be reprocessed (not acknowledged)
        // Implement dead letter queue for repeated failures
      }
    } else {
      // No handler for this event type, acknowledge to skip
      await this.redis.xack(
        this.config.streamName,
        this.config.groupName,
        messageId
      );
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}

export { EventConsumer, ConsumerConfig, EventHandler };
```

### Outbox Processor

```typescript
// src/shared/events/outbox-processor.ts

import { Redis } from 'ioredis';
import { Knex } from 'knex';

interface OutboxEntry {
  id: string;
  aggregate_type: string;
  aggregate_id: string;
  event_type: string;
  stream_name: string;
  payload: string;
  created_at: Date;
  retry_count: number;
}

class OutboxProcessor {
  private db: Knex;
  private redis: Redis;
  private running: boolean = false;
  private pollIntervalMs: number;
  private maxRetries: number;

  constructor(
    db: Knex,
    redis: Redis,
    pollIntervalMs: number = 1000,
    maxRetries: number = 5
  ) {
    this.db = db;
    this.redis = redis;
    this.pollIntervalMs = pollIntervalMs;
    this.maxRetries = maxRetries;
  }

  async start(): Promise<void> {
    this.running = true;
    await this.processLoop();
  }

  async stop(): Promise<void> {
    this.running = false;
  }

  private async processLoop(): Promise<void> {
    while (this.running) {
      try {
        await this.processOutboxBatch();
      } catch (error) {
        console.error('Outbox processor error:', error);
      }
      await this.sleep(this.pollIntervalMs);
    }
  }

  private async processOutboxBatch(): Promise<void> {
    const entries: OutboxEntry[] = await this.db('event_outbox')
      .whereNull('published_at')
      .where('retry_count', '<', this.maxRetries)
      .orderBy('created_at', 'asc')
      .limit(100);

    for (const entry of entries) {
      try {
        // Publish to Redis Stream
        await this.redis.xadd(
          entry.stream_name,
          'MAXLEN', '~', '100000',
          '*',
          'event', entry.payload
        );

        // Mark as published
        await this.db('event_outbox')
          .where('id', entry.id)
          .update({ published_at: new Date() });

      } catch (error) {
        console.error(`Failed to publish outbox entry ${entry.id}:`, error);

        // Increment retry count
        await this.db('event_outbox')
          .where('id', entry.id)
          .increment('retry_count', 1);
      }
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}

export { OutboxProcessor };
```

### Ejemplo: CV Service Event Handlers

```typescript
// src/cv-service/events/handlers.ts

import { EventConsumer, EventHandler } from '../../shared/events/event-consumer';
import { DomainEvent } from '../../shared/events/event-publisher';
import { CVService } from '../services/cv-service';

interface UserRegisteredPayload {
  userId: string;
  email: string;
  locale: string;
}

interface SubscriptionCreatedPayload {
  subscriptionId: string;
  userId: string;
  planId: string;
  quotas: {
    cvAnalysis: number;
    interviews: number;
  };
}

export function createCVEventHandlers(cvService: CVService): Map<string, EventHandler> {
  const handlers = new Map<string, EventHandler>();

  // Handle new user registration - create CV profile
  handlers.set(
    'identity.user.registered',
    async (event: DomainEvent<UserRegisteredPayload>) => {
      const { userId, locale } = event.payload;
      await cvService.createUserCVProfile(userId, { locale });
      console.log(`CV profile created for user ${userId}`);
    }
  );

  // Handle subscription creation - update CV quotas
  handlers.set(
    'billing.subscription.created',
    async (event: DomainEvent<SubscriptionCreatedPayload>) => {
      const { userId, quotas } = event.payload;
      await cvService.updateUserQuotas(userId, quotas.cvAnalysis);
      console.log(`CV quotas updated for user ${userId}: ${quotas.cvAnalysis}`);
    }
  );

  // Handle subscription cancelled - reset to free tier
  handlers.set(
    'billing.subscription.cancelled',
    async (event: DomainEvent<{ userId: string }>) => {
      const { userId } = event.payload;
      await cvService.resetToFreeTier(userId);
      console.log(`User ${userId} reset to free tier`);
    }
  );

  return handlers;
}

// Bootstrap consumer
export async function startCVEventConsumer(
  redis: Redis,
  cvService: CVService
): Promise<EventConsumer> {
  const handlers = createCVEventHandlers(cvService);

  const consumer = new EventConsumer(redis, {
    streamName: 'identity.events',
    groupName: 'cv-service',
    consumerName: `cv-service-${process.env.HOSTNAME || 'local'}`,
    handlers,
    batchSize: 10,
    blockMs: 5000,
  });

  await consumer.start();
  return consumer;
}
```

---

## Idempotencia

Para garantizar procesamiento exactamente una vez (at-least-once delivery + idempotencia):

```typescript
// src/shared/events/idempotency.ts

import { Redis } from 'ioredis';

class IdempotencyGuard {
  private redis: Redis;
  private keyPrefix: string;
  private ttlSeconds: number;

  constructor(redis: Redis, serviceName: string, ttlSeconds: number = 86400) {
    this.redis = redis;
    this.keyPrefix = `idempotency:${serviceName}`;
    this.ttlSeconds = ttlSeconds;
  }

  async isProcessed(eventId: string): Promise<boolean> {
    const key = `${this.keyPrefix}:${eventId}`;
    const exists = await this.redis.exists(key);
    return exists === 1;
  }

  async markAsProcessed(eventId: string): Promise<void> {
    const key = `${this.keyPrefix}:${eventId}`;
    await this.redis.setex(key, this.ttlSeconds, '1');
  }

  async processOnce<T>(
    eventId: string,
    handler: () => Promise<T>
  ): Promise<T | null> {
    if (await this.isProcessed(eventId)) {
      console.log(`Event ${eventId} already processed, skipping`);
      return null;
    }

    const result = await handler();
    await this.markAsProcessed(eventId);
    return result;
  }
}

export { IdempotencyGuard };
```

---

## Monitoreo y Observabilidad

### Metricas a Capturar

| Metrica | Tipo | Descripcion |
|---------|------|-------------|
| `events_published_total` | Counter | Total eventos publicados por stream |
| `events_consumed_total` | Counter | Total eventos consumidos por consumer group |
| `events_failed_total` | Counter | Eventos que fallaron procesamiento |
| `event_processing_duration_ms` | Histogram | Latencia de procesamiento |
| `consumer_lag` | Gauge | Diferencia entre ultimo evento y ultimo procesado |
| `outbox_pending_count` | Gauge | Eventos pendientes en outbox |

### Health Checks

```typescript
// src/shared/events/health-check.ts

interface StreamHealth {
  streamName: string;
  length: number;
  consumerGroups: {
    name: string;
    pending: number;
    consumers: number;
    lastDelivered: string;
  }[];
}

async function checkStreamHealth(
  redis: Redis,
  streamName: string
): Promise<StreamHealth> {
  const info = await redis.xinfo('STREAM', streamName);
  const groups = await redis.xinfo('GROUPS', streamName);

  return {
    streamName,
    length: info.length,
    consumerGroups: groups.map((g: any) => ({
      name: g.name,
      pending: g.pending,
      consumers: g.consumers,
      lastDelivered: g['last-delivered-id'],
    })),
  };
}
```

### Alertas Recomendadas

| Alerta | Condicion | Severidad |
|--------|-----------|-----------|
| Consumer Lag Alto | lag > 1000 eventos por 5min | Warning |
| Consumer Lag Critico | lag > 10000 eventos por 5min | Critical |
| Outbox Acumulado | pending > 100 por 1min | Warning |
| Procesamiento Lento | p95 > 1000ms | Warning |
| Fallos Repetidos | failures > 10/min por consumer | Critical |
| Stream Lleno | length > 90% maxlen | Warning |

---

## Estrategia de Migracion

### Fase 1: MVP (Mes 1-3)

1. Implementar event publisher/consumer base
2. Outbox pattern para Identity y Billing
3. Eventos criticos: UserRegistered, CVParsed, InterviewCompleted
4. Monitoreo basico con logs estructurados

### Fase 2: Estabilizacion (Mes 4-6)

1. Agregar todos los eventos del catalogo
2. Implementar Sagas para flujos complejos
3. Dead Letter Queue para eventos fallidos
4. Dashboard de monitoreo con Grafana

### Fase 3: Optimizacion (Mes 7-12)

1. Evaluar migracion a Kafka si throughput > 50K/hora
2. Event Sourcing para Interview Context (auditoria)
3. Event replay para debugging
4. CDC (Change Data Capture) para outbox

---

## Alternativas Consideradas

### Alternativa 1: Comunicacion Sincrona (HTTP/gRPC)

**Pros:**
- Mas simple de implementar inicialmente
- Feedback inmediato de errores
- Debugging mas facil

**Contras:**
- Acoplamiento temporal entre servicios
- Cascading failures
- No escala para operaciones largas (CV parsing)
- Dificil agregar nuevos consumidores

**Decision:** Rechazado por acoplamiento y problemas de resiliencia

### Alternativa 2: Apache Kafka desde el inicio

**Pros:**
- Escalabilidad probada a millones de eventos
- Garantias de ordenamiento
- Retention ilimitada
- Ecosistema maduro (Kafka Streams, Connect)

**Contras:**
- Complejidad operacional alta (ZooKeeper/KRaft)
- Costo inicial $500-2000/mes
- Curva de aprendizaje para el equipo
- Over-engineering para MVP

**Decision:** Rechazado para MVP, plan de migracion para Fase 3

### Alternativa 3: Amazon SQS + SNS

**Pros:**
- Serverless, sin operaciones
- Pay-per-use
- Integracion nativa con AWS

**Contras:**
- Sin ordenamiento garantizado
- Latencia mayor (10-50ms)
- Fan-out complejo con SNS
- Vendor lock-in

**Decision:** Rechazado por latencia y complejidad de fan-out

---

## Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Perdida de eventos en Redis | Baja | Alto | Persistencia AOF, replicas, outbox pattern |
| Consumer lag excesivo | Media | Medio | Auto-scaling, alertas, particionamiento |
| Eventos duplicados | Media | Bajo | Idempotencia obligatoria en handlers |
| Eventos fuera de orden | Baja | Medio | Ordenamiento por aggregate, version check |
| Schema evolution breaking | Media | Alto | Versionado de eventos, backward compatibility |

---

## Metricas de Exito

| Metrica | Target MVP | Target Y1 |
|---------|-----------|-----------|
| Event latency p95 | < 100ms | < 50ms |
| Event processing success rate | > 99% | > 99.9% |
| Consumer lag promedio | < 100 eventos | < 50 eventos |
| Outbox processing delay | < 5s | < 1s |
| System availability | 99.5% | 99.9% |

---

## Referencias

1. [Redis Streams Documentation](https://redis.io/docs/data-types/streams/)
2. [Transactional Outbox Pattern - Microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
3. [Saga Pattern - Microservices.io](https://microservices.io/patterns/data/saga.html)
4. [Event-Driven Architecture - Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)
5. [Designing Event-Driven Systems - Confluent](https://www.confluent.io/designing-event-driven-systems/)

---

## Historial de Decisiones

| Version | Fecha | Cambio |
|---------|-------|--------|
| 1.0 | 2026-01-31 | Documento inicial |

---

*Documento generado como parte de la arquitectura de Entrevistador Inteligente*
