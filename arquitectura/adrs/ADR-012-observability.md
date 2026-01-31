# ADR-012: Estrategia de Observabilidad y Monitoreo

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead, SRE Lead

---

## Contexto

### Necesidad de Observabilidad

"Entrevistador Inteligente Peru" es una plataforma SaaS critica que integra multiples componentes: procesamiento de CVs con IA, matching inteligente, simulaciones de entrevistas en tiempo real, y operaciones de pago. La observabilidad es fundamental para:

1. **Detectar problemas proactivamente** antes de que impacten usuarios
2. **Diagnosticar issues rapidamente** cuando ocurren incidentes
3. **Optimizar performance** de componentes criticos (especialmente IA)
4. **Controlar costos** de infraestructura y APIs de LLM
5. **Medir KPIs de negocio** para decision-making basado en datos

### Los Tres Pilares de Observabilidad

```
┌─────────────────────────────────────────────────────────────────────┐
│                      OBSERVABILIDAD                                  │
├─────────────────┬─────────────────────┬────────────────────────────┤
│      LOGS       │       METRICS       │          TRACES            │
│   (Eventos)     │    (Agregaciones)   │    (Flujo de requests)     │
├─────────────────┼─────────────────────┼────────────────────────────┤
│ - Structured    │ - Business KPIs     │ - Request propagation      │
│ - Centralized   │ - Technical health  │ - Latency breakdown        │
│ - Searchable    │ - Alertas           │ - Error correlation        │
│ - Contextual    │ - Dashboards        │ - Dependency mapping       │
└─────────────────┴─────────────────────┴────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │     ALERTS      │
                    │  (Proactive)    │
                    └─────────────────┘
```

### Restricciones del Proyecto

| Factor | Detalle |
|--------|---------|
| Presupuesto mensual observabilidad | $100-300 USD (startup stage) |
| Equipo DevOps | 0.5 FTE inicial (shared responsibility) |
| Expertise requerido | Bajo (equipo full-stack, no SRE dedicado) |
| Vendor lock-in tolerance | Bajo (preferir open source) |
| Retention de datos | 30 dias logs, 1 ano metricas |

---

## Decision

**Adoptamos un stack de observabilidad Open Source "Best-of-Breed" con OpenTelemetry como estandar de instrumentacion:**

| Pilar | Solucion | Justificacion |
|-------|----------|---------------|
| **Logging** | Pino + Grafana Loki | Structured, bajo costo, nativo Cloud |
| **Metrics** | Prometheus + Grafana | Estandar de facto, excelente alerting |
| **Tracing** | OpenTelemetry + Grafana Tempo | Vendor-neutral, bajo overhead |
| **Dashboards** | Grafana (unified) | Un solo pane of glass |
| **Alerting** | Grafana Alerting + PagerDuty (on-call) | Alertas inteligentes, escalation |

### Arquitectura de Observabilidad

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION LAYER                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   API        │  │   Workers    │  │   Frontend   │  │   Cron Jobs  │     │
│  │   (NestJS)   │  │   (Bull)     │  │   (Next.js)  │  │              │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                 │                 │                 │              │
│         └─────────────────┴─────────────────┴─────────────────┘              │
│                                     │                                        │
│                        ┌────────────┴────────────┐                          │
│                        │   OpenTelemetry SDK     │                          │
│                        │   (Auto-instrumentation)│                          │
│                        └────────────┬────────────┘                          │
└─────────────────────────────────────┼───────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │    Logs      │  │   Metrics    │  │   Traces     │
           │    (Pino)    │  │  (Prometheus │  │   (OTLP)     │
           │              │  │   client)    │  │              │
           └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                  │                 │                 │
                  ▼                 ▼                 ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │ Grafana Loki │  │  Prometheus  │  │ Grafana Tempo│
           │              │  │              │  │              │
           └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                  │                 │                 │
                  └─────────────────┼─────────────────┘
                                    │
                                    ▼
                           ┌──────────────────┐
                           │     GRAFANA      │
                           │   (Dashboards)   │
                           │   (Alerting)     │
                           └────────┬─────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
             ┌──────────┐    ┌──────────┐    ┌──────────┐
             │  Slack   │    │PagerDuty │    │  Email   │
             │(warnings)│    │(critical)│    │(reports) │
             └──────────┘    └──────────┘    └──────────┘
```

---

## Implementacion Detallada

### 1. Logging con Pino + Grafana Loki

#### Configuracion de Pino

```typescript
// src/shared/infrastructure/logging/logger.config.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
    bindings: (bindings) => ({
      pid: bindings.pid,
      host: bindings.hostname,
      service: 'entrevistador-api',
      version: process.env.APP_VERSION,
      environment: process.env.NODE_ENV,
    }),
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie',
      'password',
      'token',
      'apiKey',
      'creditCard',
      '*.password',
      '*.token',
    ],
    censor: '[REDACTED]',
  },
});
```

#### Estructura de Log Estandarizada

```typescript
// Formato de log estructurado
interface LogEntry {
  timestamp: string;
  level: 'debug' | 'info' | 'warn' | 'error' | 'fatal';
  message: string;

  // Contexto
  service: string;
  version: string;
  environment: string;

  // Request context (cuando aplica)
  requestId?: string;
  userId?: string;
  tenantId?: string;
  sessionId?: string;

  // Error context (cuando aplica)
  error?: {
    name: string;
    message: string;
    stack: string;
    code?: string;
  };

  // Business context
  module?: string;
  action?: string;
  duration?: number;

  // Custom data
  data?: Record<string, unknown>;
}
```

#### Middleware de Request Logging

```typescript
// src/shared/infrastructure/logging/request-logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { logger } from './logger.config';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestLoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const requestId = req.headers['x-request-id'] || uuidv4();
    const startTime = Date.now();

    // Attach to request for downstream use
    req['requestId'] = requestId;
    req['logger'] = logger.child({ requestId });

    // Log request start
    req['logger'].info({
      message: 'Request started',
      method: req.method,
      path: req.path,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
    });

    // Log response
    res.on('finish', () => {
      const duration = Date.now() - startTime;
      const logLevel = res.statusCode >= 500 ? 'error' :
                       res.statusCode >= 400 ? 'warn' : 'info';

      req['logger'][logLevel]({
        message: 'Request completed',
        method: req.method,
        path: req.path,
        statusCode: res.statusCode,
        duration,
        contentLength: res.get('content-length'),
      });
    });

    next();
  }
}
```

### 2. Metricas con Prometheus + Grafana

#### Metricas de Negocio a Trackear

```typescript
// src/shared/infrastructure/metrics/business-metrics.ts
import { Counter, Histogram, Gauge } from 'prom-client';

// === REGISTROS Y USUARIOS ===
export const userRegistrations = new Counter({
  name: 'ei_user_registrations_total',
  help: 'Total de usuarios registrados',
  labelNames: ['plan', 'source', 'country'],
});

export const dailyActiveUsers = new Gauge({
  name: 'ei_daily_active_users',
  help: 'Usuarios activos en las ultimas 24 horas',
  labelNames: ['plan'],
});

// === CVs ===
export const cvsUploaded = new Counter({
  name: 'ei_cvs_uploaded_total',
  help: 'CVs subidos por usuarios',
  labelNames: ['format', 'country'],
});

export const cvParsingDuration = new Histogram({
  name: 'ei_cv_parsing_duration_seconds',
  help: 'Tiempo de parsing de CVs',
  labelNames: ['parser', 'success'],
  buckets: [0.5, 1, 2, 5, 10, 30],
});

export const cvOptimizations = new Counter({
  name: 'ei_cv_optimizations_total',
  help: 'Optimizaciones de CV realizadas',
  labelNames: ['type', 'success'],
});

// === ENTREVISTAS ===
export const interviewsStarted = new Counter({
  name: 'ei_interviews_started_total',
  help: 'Entrevistas iniciadas',
  labelNames: ['type', 'industry', 'country'],
});

export const interviewsCompleted = new Counter({
  name: 'ei_interviews_completed_total',
  help: 'Entrevistas completadas',
  labelNames: ['type', 'score_range'],
});

export const interviewDuration = new Histogram({
  name: 'ei_interview_duration_minutes',
  help: 'Duracion de entrevistas',
  labelNames: ['type'],
  buckets: [5, 10, 15, 20, 30, 45, 60],
});

export const interviewCompletionRate = new Gauge({
  name: 'ei_interview_completion_rate',
  help: 'Tasa de completion de entrevistas',
});

// === CONVERSION Y MONETIZACION ===
export const planConversions = new Counter({
  name: 'ei_plan_conversions_total',
  help: 'Conversiones entre planes',
  labelNames: ['from_plan', 'to_plan'],
});

export const revenueGenerated = new Counter({
  name: 'ei_revenue_generated_soles',
  help: 'Revenue generado en soles',
  labelNames: ['plan', 'payment_method'],
});

export const churnEvents = new Counter({
  name: 'ei_churn_events_total',
  help: 'Eventos de churn (cancelaciones)',
  labelNames: ['plan', 'reason'],
});

export const monthlyRecurringRevenue = new Gauge({
  name: 'ei_mrr_soles',
  help: 'Monthly Recurring Revenue en soles',
});

// === NPS Y SATISFACCION ===
export const npsResponses = new Counter({
  name: 'ei_nps_responses_total',
  help: 'Respuestas NPS recibidas',
  labelNames: ['score_category'], // promoter, passive, detractor
});

export const npsScore = new Gauge({
  name: 'ei_nps_score',
  help: 'NPS Score actual (-100 a 100)',
});

// === B2B ===
export const jobPostings = new Counter({
  name: 'ei_job_postings_total',
  help: 'Vacantes publicadas por empresas',
  labelNames: ['company_size', 'industry'],
});

export const candidateMatches = new Counter({
  name: 'ei_candidate_matches_total',
  help: 'Matches candidato-vacante generados',
  labelNames: ['match_score_range'],
});
```

#### Metricas Tecnicas

```typescript
// src/shared/infrastructure/metrics/technical-metrics.ts
import { Counter, Histogram, Gauge, Summary } from 'prom-client';

// === API LATENCY ===
export const httpRequestDuration = new Histogram({
  name: 'ei_http_request_duration_seconds',
  help: 'Duracion de requests HTTP',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

export const httpRequestDurationSummary = new Summary({
  name: 'ei_http_request_duration_summary',
  help: 'Summary de duracion para percentiles precisos',
  labelNames: ['method', 'route'],
  percentiles: [0.5, 0.9, 0.95, 0.99],
  maxAgeSeconds: 600,
  ageBuckets: 5,
});

// === ERRORES ===
export const httpErrors = new Counter({
  name: 'ei_http_errors_total',
  help: 'Errores HTTP por endpoint',
  labelNames: ['method', 'route', 'status_code', 'error_type'],
});

export const unhandledExceptions = new Counter({
  name: 'ei_unhandled_exceptions_total',
  help: 'Excepciones no manejadas',
  labelNames: ['error_name', 'module'],
});

// === LLM / AI ===
export const llmApiCalls = new Counter({
  name: 'ei_llm_api_calls_total',
  help: 'Llamadas a APIs de LLM',
  labelNames: ['provider', 'model', 'operation', 'success'],
});

export const llmApiLatency = new Histogram({
  name: 'ei_llm_api_latency_seconds',
  help: 'Latencia de APIs de LLM',
  labelNames: ['provider', 'model', 'operation'],
  buckets: [0.5, 1, 2, 5, 10, 30, 60],
});

export const llmTokensUsed = new Counter({
  name: 'ei_llm_tokens_used_total',
  help: 'Tokens consumidos en LLM APIs',
  labelNames: ['provider', 'model', 'type'], // input, output
});

export const llmCostUsd = new Counter({
  name: 'ei_llm_cost_usd_total',
  help: 'Costo de LLM APIs en USD',
  labelNames: ['provider', 'model', 'user_plan'],
});

export const llmCostPerUser = new Gauge({
  name: 'ei_llm_cost_per_user_usd',
  help: 'Costo promedio de LLM por usuario',
  labelNames: ['plan'],
});

// === QUEUES ===
export const queueDepth = new Gauge({
  name: 'ei_queue_depth',
  help: 'Profundidad de colas de trabajo',
  labelNames: ['queue_name'],
});

export const queueJobDuration = new Histogram({
  name: 'ei_queue_job_duration_seconds',
  help: 'Duracion de jobs en cola',
  labelNames: ['queue_name', 'job_type', 'success'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 120],
});

export const queueJobsProcessed = new Counter({
  name: 'ei_queue_jobs_processed_total',
  help: 'Jobs procesados',
  labelNames: ['queue_name', 'job_type', 'status'],
});

// === DATABASE ===
export const dbConnectionsActive = new Gauge({
  name: 'ei_db_connections_active',
  help: 'Conexiones activas a la base de datos',
  labelNames: ['pool'],
});

export const dbConnectionsIdle = new Gauge({
  name: 'ei_db_connections_idle',
  help: 'Conexiones idle en el pool',
  labelNames: ['pool'],
});

export const dbQueryDuration = new Histogram({
  name: 'ei_db_query_duration_seconds',
  help: 'Duracion de queries a la base de datos',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
});

// === CACHE ===
export const cacheHits = new Counter({
  name: 'ei_cache_hits_total',
  help: 'Cache hits',
  labelNames: ['cache_name'],
});

export const cacheMisses = new Counter({
  name: 'ei_cache_misses_total',
  help: 'Cache misses',
  labelNames: ['cache_name'],
});

// === EXTERNAL SERVICES ===
export const externalApiLatency = new Histogram({
  name: 'ei_external_api_latency_seconds',
  help: 'Latencia de APIs externas',
  labelNames: ['service', 'operation', 'success'],
  buckets: [0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

export const paymentEvents = new Counter({
  name: 'ei_payment_events_total',
  help: 'Eventos de pago',
  labelNames: ['provider', 'event_type', 'success'],
});
```

### 3. Distributed Tracing con OpenTelemetry

#### Configuracion de OpenTelemetry

```typescript
// src/shared/infrastructure/tracing/otel.config.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';

const traceExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://tempo:4318/v1/traces',
});

export const otelSdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'entrevistador-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
  }),
  spanProcessor: new BatchSpanProcessor(traceExporter, {
    maxQueueSize: 2048,
    maxExportBatchSize: 512,
    scheduledDelayMillis: 5000,
    exportTimeoutMillis: 30000,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingPaths: ['/health', '/metrics', '/ready'],
      },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-nestjs-core': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
      '@opentelemetry/instrumentation-ioredis': { enabled: true },
    }),
  ],
});

// Custom span para operaciones de negocio
import { trace, SpanKind, SpanStatusCode } from '@opentelemetry/api';

export const tracer = trace.getTracer('entrevistador-api');

export function traceOperation<T>(
  name: string,
  attributes: Record<string, string | number | boolean>,
  operation: () => Promise<T>
): Promise<T> {
  return tracer.startActiveSpan(
    name,
    { kind: SpanKind.INTERNAL, attributes },
    async (span) => {
      try {
        const result = await operation();
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error.message,
        });
        span.recordException(error);
        throw error;
      } finally {
        span.end();
      }
    }
  );
}
```

#### Ejemplo de Tracing en Servicio de Entrevistas

```typescript
// src/modules/interview-sim/application/interview.service.ts
import { traceOperation, tracer } from '@shared/infrastructure/tracing';

@Injectable()
export class InterviewService {
  async conductInterview(userId: string, interviewId: string): Promise<InterviewResult> {
    return traceOperation(
      'interview.conduct',
      {
        'user.id': userId,
        'interview.id': interviewId,
        'interview.type': 'behavioral',
      },
      async () => {
        // Span hijo para LLM call
        const question = await tracer.startActiveSpan(
          'llm.generate_question',
          { attributes: { 'llm.provider': 'anthropic', 'llm.model': 'claude-3-haiku' } },
          async (span) => {
            try {
              const result = await this.llmService.generateQuestion(context);
              span.setAttribute('llm.tokens.input', result.usage.input);
              span.setAttribute('llm.tokens.output', result.usage.output);
              return result;
            } finally {
              span.end();
            }
          }
        );

        // Span hijo para scoring
        const score = await tracer.startActiveSpan(
          'interview.score_response',
          async (span) => {
            const result = await this.scoringService.evaluate(response);
            span.setAttribute('score.value', result.score);
            span.end();
            return result;
          }
        );

        return { question, score };
      }
    );
  }
}
```

### 4. Sistema de Alertas

#### Alertas Criticas (PagerDuty - On-Call)

```yaml
# grafana/alerts/critical.yaml
groups:
  - name: critical-alerts
    interval: 30s
    rules:
      # Error Rate > 1%
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(ei_http_errors_total{status_code=~"5.."}[5m]))
            /
            sum(rate(ei_http_request_duration_seconds_count[5m]))
          ) > 0.01
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Error rate above 1%"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"
          runbook_url: "https://wiki.internal/runbooks/high-error-rate"

      # Latency P95 > 3s
      - alert: HighLatencyP95
        expr: |
          histogram_quantile(0.95,
            sum(rate(ei_http_request_duration_seconds_bucket[5m])) by (le)
          ) > 3
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "P95 latency above 3 seconds"
          description: "P95 latency is {{ $value | humanizeDuration }}"

      # LLM API Failures
      - alert: LLMApiFailures
        expr: |
          (
            sum(rate(ei_llm_api_calls_total{success="false"}[5m]))
            /
            sum(rate(ei_llm_api_calls_total[5m]))
          ) > 0.05
        for: 3m
        labels:
          severity: critical
          team: ai
        annotations:
          summary: "LLM API failure rate above 5%"
          description: "{{ $labels.provider }} {{ $labels.model }} failing at {{ $value | humanizePercentage }}"

      # Payment Failures
      - alert: PaymentFailures
        expr: |
          sum(rate(ei_payment_events_total{event_type="charge", success="false"}[10m])) > 3
        for: 5m
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "Multiple payment failures detected"
          description: "{{ $value }} payment failures in the last 10 minutes"

      # Queue Backup
      - alert: QueueBackup
        expr: ei_queue_depth > 1000
        for: 10m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Queue {{ $labels.queue_name }} is backing up"
          description: "Queue depth is {{ $value }} items"

      # Database Connections Exhausted
      - alert: DatabaseConnectionsExhausted
        expr: |
          ei_db_connections_active / ei_db_connections_active + ei_db_connections_idle > 0.9
        for: 5m
        labels:
          severity: critical
          team: dba
        annotations:
          summary: "Database connection pool near exhaustion"
          description: "{{ $value | humanizePercentage }} of connections in use"
```

#### Alertas de Warning (Slack)

```yaml
# grafana/alerts/warning.yaml
groups:
  - name: warning-alerts
    interval: 1m
    rules:
      # LLM Cost Spike
      - alert: LLMCostSpike
        expr: |
          sum(increase(ei_llm_cost_usd_total[1h])) > 50
        for: 0m
        labels:
          severity: warning
          team: ai
        annotations:
          summary: "LLM costs spiking"
          description: "${{ $value }} spent on LLM in the last hour"

      # Interview Completion Rate Drop
      - alert: LowInterviewCompletionRate
        expr: ei_interview_completion_rate < 0.6
        for: 30m
        labels:
          severity: warning
          team: product
        annotations:
          summary: "Interview completion rate below 60%"
          description: "Completion rate is {{ $value | humanizePercentage }}"

      # High CV Parsing Failures
      - alert: CVParsingFailures
        expr: |
          sum(rate(ei_cv_parsing_duration_seconds_count{success="false"}[15m]))
          /
          sum(rate(ei_cv_parsing_duration_seconds_count[15m])) > 0.1
        for: 15m
        labels:
          severity: warning
          team: ai
        annotations:
          summary: "CV parsing failure rate above 10%"

      # Cache Hit Rate Low
      - alert: LowCacheHitRate
        expr: |
          sum(rate(ei_cache_hits_total[5m]))
          /
          (sum(rate(ei_cache_hits_total[5m])) + sum(rate(ei_cache_misses_total[5m]))) < 0.7
        for: 15m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Cache hit rate below 70%"

      # Slow Database Queries
      - alert: SlowDatabaseQueries
        expr: |
          histogram_quantile(0.95,
            sum(rate(ei_db_query_duration_seconds_bucket[5m])) by (le, table)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
          team: dba
        annotations:
          summary: "Slow queries detected on {{ $labels.table }}"
```

#### Alertas de Negocio (Email Daily Digest)

```yaml
# grafana/alerts/business.yaml
groups:
  - name: business-alerts
    interval: 1h
    rules:
      # Registrations Drop
      - alert: RegistrationsDropped
        expr: |
          sum(increase(ei_user_registrations_total[24h]))
          <
          sum(increase(ei_user_registrations_total[24h] offset 7d)) * 0.5
        for: 0m
        labels:
          severity: info
          team: growth
        annotations:
          summary: "Daily registrations dropped 50% vs last week"

      # Conversion Rate Change
      - alert: ConversionRateChange
        expr: |
          (
            sum(increase(ei_plan_conversions_total{to_plan="premium"}[7d]))
            /
            sum(increase(ei_user_registrations_total[7d]))
          ) < 0.02
        for: 0m
        labels:
          severity: info
          team: product
        annotations:
          summary: "Free to Premium conversion rate below 2%"

      # Churn Spike
      - alert: ChurnSpike
        expr: |
          sum(increase(ei_churn_events_total[7d]))
          >
          sum(increase(ei_churn_events_total[7d] offset 7d)) * 1.5
        for: 0m
        labels:
          severity: warning
          team: customer-success
        annotations:
          summary: "Churn increased 50% vs previous week"

      # NPS Drop
      - alert: NPSDrop
        expr: ei_nps_score < 30
        for: 24h
        labels:
          severity: warning
          team: product
        annotations:
          summary: "NPS score dropped below 30"
```

### 5. Dashboards de Grafana

#### Dashboard de Negocio (Executive)

```json
{
  "title": "Entrevistador Inteligente - Business Dashboard",
  "panels": [
    {
      "title": "Daily Active Users",
      "type": "stat",
      "targets": [{ "expr": "ei_daily_active_users" }]
    },
    {
      "title": "Registrations Today",
      "type": "stat",
      "targets": [{ "expr": "sum(increase(ei_user_registrations_total[24h]))" }]
    },
    {
      "title": "CVs Uploaded Today",
      "type": "stat",
      "targets": [{ "expr": "sum(increase(ei_cvs_uploaded_total[24h]))" }]
    },
    {
      "title": "Interviews Completed Today",
      "type": "stat",
      "targets": [{ "expr": "sum(increase(ei_interviews_completed_total[24h]))" }]
    },
    {
      "title": "MRR (Soles)",
      "type": "stat",
      "targets": [{ "expr": "ei_mrr_soles" }]
    },
    {
      "title": "NPS Score",
      "type": "gauge",
      "targets": [{ "expr": "ei_nps_score" }],
      "thresholds": [
        { "value": 0, "color": "red" },
        { "value": 30, "color": "yellow" },
        { "value": 50, "color": "green" }
      ]
    },
    {
      "title": "Registration Trend (7d)",
      "type": "timeseries",
      "targets": [{ "expr": "sum(increase(ei_user_registrations_total[1d]))" }]
    },
    {
      "title": "Free vs Premium Users",
      "type": "piechart",
      "targets": [
        { "expr": "ei_daily_active_users{plan='free'}", "legendFormat": "Free" },
        { "expr": "ei_daily_active_users{plan='premium'}", "legendFormat": "Premium" }
      ]
    },
    {
      "title": "Conversion Funnel",
      "type": "bar-gauge",
      "targets": [
        { "expr": "sum(increase(ei_user_registrations_total[30d]))", "legendFormat": "Registered" },
        { "expr": "sum(increase(ei_cvs_uploaded_total[30d]))", "legendFormat": "CV Uploaded" },
        { "expr": "sum(increase(ei_interviews_started_total[30d]))", "legendFormat": "Interview Started" },
        { "expr": "sum(increase(ei_plan_conversions_total{to_plan='premium'}[30d]))", "legendFormat": "Converted" }
      ]
    }
  ]
}
```

#### Dashboard Tecnico (SRE/Engineering)

```json
{
  "title": "Entrevistador Inteligente - Technical Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [{ "expr": "sum(rate(ei_http_request_duration_seconds_count[5m]))" }]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(ei_http_errors_total[5m])) / sum(rate(ei_http_request_duration_seconds_count[5m]))"
      }],
      "thresholds": [
        { "value": 0, "color": "green" },
        { "value": 0.01, "color": "yellow" },
        { "value": 0.05, "color": "red" }
      ]
    },
    {
      "title": "Latency Percentiles",
      "type": "timeseries",
      "targets": [
        { "expr": "histogram_quantile(0.50, sum(rate(ei_http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p50" },
        { "expr": "histogram_quantile(0.95, sum(rate(ei_http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p95" },
        { "expr": "histogram_quantile(0.99, sum(rate(ei_http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p99" }
      ]
    },
    {
      "title": "LLM API Latency",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.95, sum(rate(ei_llm_api_latency_seconds_bucket[5m])) by (le, provider))"
      }]
    },
    {
      "title": "LLM Costs (USD/hour)",
      "type": "timeseries",
      "targets": [{ "expr": "sum(rate(ei_llm_cost_usd_total[1h]))" }]
    },
    {
      "title": "Queue Depths",
      "type": "timeseries",
      "targets": [{ "expr": "ei_queue_depth", "legendFormat": "{{ queue_name }}" }]
    },
    {
      "title": "Database Connections",
      "type": "timeseries",
      "targets": [
        { "expr": "ei_db_connections_active", "legendFormat": "Active" },
        { "expr": "ei_db_connections_idle", "legendFormat": "Idle" }
      ]
    },
    {
      "title": "Cache Hit Rate",
      "type": "gauge",
      "targets": [{
        "expr": "sum(rate(ei_cache_hits_total[5m])) / (sum(rate(ei_cache_hits_total[5m])) + sum(rate(ei_cache_misses_total[5m])))"
      }]
    }
  ]
}
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Costo controlado** | Stack open source, ~$150/mes en infra |
| **Vendor neutral** | OpenTelemetry permite cambiar backends |
| **Correlacion completa** | Logs-Metrics-Traces conectados por trace_id |
| **Alerting proactivo** | Detectar problemas antes que usuarios |
| **Insights de negocio** | Dashboards ejecutivos automatizados |
| **Debug eficiente** | Distributed tracing reduce MTTR |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Setup inicial complejo** | Docker Compose pre-configurado, IaC templates |
| **Expertise requerido** | Runbooks detallados, alertas con links |
| **Overhead de instrumentacion** | Sampling en produccion (10-20%) |
| **Storage de logs** | Retention policies, log levels por ambiente |

### Costos Estimados

| Componente | Self-Hosted (Startup) | Managed (Scale) |
|------------|----------------------|-----------------|
| Grafana Cloud Free | $0 | - |
| Loki (storage) | ~$30/mes | $50-100/mes |
| Prometheus | ~$20/mes | Included in GC |
| Tempo | ~$20/mes | Included in GC |
| Alerting (PagerDuty) | $0 (free tier) | $25/user/mes |
| **Total** | **~$70-100/mes** | **$200-400/mes** |

---

## Alternativas Consideradas

### Opcion A: All-in-One SaaS (Datadog)

**Pros:**
- Setup inmediato
- UI excelente
- Correlacion automatica
- Soporte 24/7

**Contras:**
- Costo: $500-2000+/mes para startup
- Vendor lock-in alto
- Precios impredecibles con escala

**Veredicto:** Rechazado - Costo prohibitivo para etapa actual

### Opcion B: AWS Native (CloudWatch + X-Ray)

**Pros:**
- Integracion nativa AWS
- Sin infra adicional
- Buen para serverless

**Contras:**
- Dashboards limitados
- Query language complejo
- Lock-in a AWS
- Costo escala con volumen

**Veredicto:** Rechazado - Lock-in y limitaciones de visualizacion

### Opcion C: Sentry + New Relic APM

**Pros:**
- Sentry excelente para errors
- New Relic APM completo

**Contras:**
- Dos vendors diferentes
- New Relic costoso ($99/user/mes)
- Overlap de funcionalidades

**Veredicto:** Rechazado - Fragmentacion y costo

---

## Plan de Implementacion

### Fase 1: Foundation (Semana 1-2)
- [ ] Setup Docker Compose con Grafana stack
- [ ] Configurar Pino con formato estructurado
- [ ] Instrumentar endpoints principales con metricas basicas
- [ ] Dashboard tecnico inicial

### Fase 2: Business Metrics (Semana 3-4)
- [ ] Implementar metricas de negocio
- [ ] Dashboard ejecutivo
- [ ] Alertas criticas (error rate, latency)

### Fase 3: Tracing (Semana 5-6)
- [ ] Integrar OpenTelemetry SDK
- [ ] Instrumentar servicios de IA
- [ ] Correlacion logs-traces

### Fase 4: Alerting & Runbooks (Semana 7-8)
- [ ] Configurar PagerDuty (on-call rotation)
- [ ] Crear runbooks para cada alerta
- [ ] Slack integration para warnings
- [ ] Email digests para business alerts

---

## Referencias

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana Loki Best Practices](https://grafana.com/docs/loki/latest/best-practices/)
- [Prometheus Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)
- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [The Three Pillars of Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/)
- [Pino Logger Documentation](https://getpino.io/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
