# ADR-002: Bounded Contexts y Modelo de Dominio (DDD)

## Metadata

| Campo | Valor |
|-------|-------|
| **ID** | ADR-002 |
| **Titulo** | Bounded Contexts y Modelo de Dominio |
| **Estado** | Aceptado |
| **Fecha** | 2026-01-31 |
| **Autores** | Equipo de Arquitectura |
| **Revisores** | Tech Lead, Domain Experts |
| **Tags** | DDD, bounded-context, domain-model, microservices |

---

## Contexto

"Entrevistador Inteligente" es una plataforma compleja que abarca multiples subdominios: gestion de usuarios, procesamiento de CVs, matching inteligente, simulacion de entrevistas con IA, facturacion y analiticas.

Para mantener la coherencia del modelo de dominio, reducir el acoplamiento entre equipos y permitir escalabilidad independiente, necesitamos definir claramente los Bounded Contexts siguiendo los principios de Domain-Driven Design (DDD).

### Problemas a Resolver

1. **Complejidad del dominio**: Multiples subdominios con vocabulario y reglas distintas
2. **Escalabilidad de equipos**: Permitir desarrollo paralelo sin conflictos
3. **Evolucion independiente**: Cada contexto debe poder evolucionar a su propio ritmo
4. **Integridad conceptual**: Mantener modelos coherentes dentro de cada contexto

---

## Decision

Adoptamos una arquitectura basada en **6 Bounded Contexts** claramente delimitados, con relaciones explicitas definidas en un Context Map.

---

## Bounded Contexts

### Vista General

```
+------------------------------------------------------------------+
|                     ENTREVISTADOR INTELIGENTE                     |
+------------------------------------------------------------------+
|                                                                   |
|  +---------------+    +------------------+    +----------------+  |
|  |   IDENTITY    |    |  CV MANAGEMENT   |    |  JOB MATCHING  |  |
|  |    CONTEXT    |    |     CONTEXT      |    |    CONTEXT     |  |
|  +-------+-------+    +--------+---------+    +-------+--------+  |
|          |                     |                      |           |
|          +----------+----------+----------+-----------+           |
|                     |                     |                       |
|  +------------------v---+    +------------v-----------+           |
|  |     INTERVIEW        |    |       BILLING          |           |
|  |      CONTEXT         |    |       CONTEXT          |           |
|  +----------+-----------+    +------------+-----------+           |
|             |                             |                       |
|             +-------------+---------------+                       |
|                           |                                       |
|              +------------v------------+                          |
|              |      ANALYTICS          |                          |
|              |       CONTEXT           |                          |
|              +-------------------------+                          |
|                                                                   |
+------------------------------------------------------------------+
```

---

## 1. Identity Context

### Proposito
Gestionar la identidad, autenticacion, autorizacion y perfiles de usuarios en la plataforma.

### Lenguaje Ubicuo
- **User**: Persona registrada en el sistema
- **Candidate**: Usuario que busca empleo y practica entrevistas
- **Company**: Organizacion que contrata y evalua candidatos
- **Recruiter**: Usuario de empresa que gestiona procesos
- **Credentials**: Datos de autenticacion del usuario
- **Role**: Conjunto de permisos asignados
- **Session**: Sesion activa de usuario autenticado

### Modelo de Dominio

```
+------------------------------------------------------------------+
|                       IDENTITY CONTEXT                            |
+------------------------------------------------------------------+
|                                                                   |
|  AGGREGATES:                                                      |
|  +-----------+     +-------------+     +------------------+       |
|  |   User    |     |   Company   |     |   Subscription   |       |
|  | (root)    |     |   (root)    |     |     (root)       |       |
|  +-----------+     +-------------+     +------------------+       |
|       |                  |                     |                  |
|  +----v----+       +-----v-----+         +-----v------+           |
|  | Profile |       | Recruiter |         |    Plan    |           |
|  +---------+       +-----------+         +------------+           |
|  | Email   |       | Candidate |                                  |
|  +---------+       | Invitation|                                  |
|                    +-----------+                                  |
|                                                                   |
|  VALUE OBJECTS:                                                   |
|  +----------+ +------------+ +--------+ +---------------+         |
|  | UserId   | | Email      | | Role   | | PasswordHash  |         |
|  +----------+ +------------+ +--------+ +---------------+         |
|  | CompanyId| | Phone      | | Token  | | RefreshToken  |         |
|  +----------+ +------------+ +--------+ +---------------+         |
|                                                                   |
+------------------------------------------------------------------+
```

### Agregados

#### User (Aggregate Root)
```typescript
User {
  id: UserId
  email: Email
  passwordHash: PasswordHash
  profile: Profile
  role: Role
  status: UserStatus
  createdAt: DateTime
  lastLoginAt: DateTime

  // Invariantes
  - Email debe ser unico en el sistema
  - Password debe cumplir politica de seguridad
  - Usuario inactivo no puede autenticarse
}

Profile {
  firstName: string
  lastName: string
  phone: Phone (optional)
  avatarUrl: URL (optional)
  timezone: Timezone
  language: Language
}
```

#### Company (Aggregate Root)
```typescript
Company {
  id: CompanyId
  name: string
  domain: Domain
  recruiters: Recruiter[]
  invitations: CandidateInvitation[]
  settings: CompanySettings

  // Invariantes
  - Debe tener al menos un recruiter admin
  - Dominio de email debe ser verificado
  - Limite de recruiters segun plan
}
```

### Eventos de Dominio

| Evento | Descripcion | Consumidores |
|--------|-------------|--------------|
| `UserRegistered` | Usuario completo registro | CV Management, Analytics |
| `UserAuthenticated` | Login exitoso | Analytics |
| `CompanyCreated` | Nueva empresa registrada | Billing, Analytics |
| `RecruiterInvited` | Recruiter invitado a empresa | Notifications |
| `CandidateInvitedByCompany` | Candidato invitado | CV Management, Interview |
| `UserRoleChanged` | Cambio de permisos | All contexts |
| `SubscriptionChanged` | Cambio de plan | Billing, All contexts |

---

## 2. CV Management Context

### Proposito
Gestionar el ciclo de vida completo de CVs: parsing, almacenamiento, optimizacion ATS y versionado.

### Lenguaje Ubicuo
- **Resume**: Documento CV del candidato
- **ParsedResume**: CV procesado y estructurado
- **ATSScore**: Puntuacion de compatibilidad ATS
- **Skill**: Habilidad extraida del CV
- **Experience**: Experiencia laboral estructurada
- **Education**: Formacion academica
- **Optimization**: Sugerencia de mejora del CV

### Modelo de Dominio

```
+------------------------------------------------------------------+
|                    CV MANAGEMENT CONTEXT                          |
+------------------------------------------------------------------+
|                                                                   |
|  AGGREGATES:                                                      |
|  +----------------+     +------------------+                      |
|  |    Resume      |     |  OptimizationJob |                      |
|  |    (root)      |     |     (root)       |                      |
|  +----------------+     +------------------+                      |
|         |                       |                                 |
|  +------v-------+        +------v--------+                        |
|  | ParsedData   |        | Suggestion    |                        |
|  +--------------+        +---------------+                        |
|  | Experience[] |        | ATSAnalysis   |                        |
|  +--------------+        +---------------+                        |
|  | Education[]  |                                                 |
|  +--------------+                                                 |
|  | Skill[]      |                                                 |
|  +--------------+                                                 |
|                                                                   |
|  VALUE OBJECTS:                                                   |
|  +----------+ +------------+ +----------+ +--------------+        |
|  | ResumeId | | SkillLevel | | ATSScore | | FileMetadata |        |
|  +----------+ +------------+ +----------+ +--------------+        |
|  | Version  | | DateRange  | | Keywords | | ParseStatus  |        |
|  +----------+ +------------+ +----------+ +--------------+        |
|                                                                   |
+------------------------------------------------------------------+
```

### Agregados

#### Resume (Aggregate Root)
```typescript
Resume {
  id: ResumeId
  candidateId: UserId  // Reference to Identity Context
  versions: ResumeVersion[]
  currentVersion: ResumeVersion
  parsedData: ParsedData
  atsScore: ATSScore
  status: ResumeStatus

  // Invariantes
  - Maximo 5 versiones activas por candidato
  - Archivo debe ser PDF, DOC o DOCX
  - Tamano maximo 5MB
}

ParsedData {
  personalInfo: PersonalInfo
  experiences: Experience[]
  education: Education[]
  skills: Skill[]
  languages: Language[]
  certifications: Certification[]
  rawText: string
  confidence: number
}

Experience {
  company: string
  position: string
  period: DateRange
  description: string
  skills: Skill[]
  isCurrentJob: boolean
}
```

#### OptimizationJob (Aggregate Root)
```typescript
OptimizationJob {
  id: OptimizationJobId
  resumeId: ResumeId
  targetJobId: JobId (optional)
  suggestions: Suggestion[]
  atsAnalysis: ATSAnalysis
  status: OptimizationStatus

  // Invariantes
  - Solo un job activo por resume
  - Requiere resume parseado exitosamente
}
```

### Eventos de Dominio

| Evento | Descripcion | Consumidores |
|--------|-------------|--------------|
| `ResumeUploaded` | CV subido al sistema | Parser Service |
| `ResumeParsed` | CV procesado exitosamente | Job Matching, Analytics |
| `ResumeParsingFailed` | Error en parsing | Notifications |
| `ATSScoreCalculated` | Score ATS generado | Analytics, Job Matching |
| `OptimizationCompleted` | Sugerencias generadas | Notifications |
| `ResumeVersionCreated` | Nueva version de CV | Job Matching |

---

## 3. Job Matching Context

### Proposito
Realizar matching semantico entre candidatos y ofertas de empleo, generando recomendaciones personalizadas.

### Lenguaje Ubicuo
- **Job**: Oferta de empleo
- **JobRequirement**: Requisito de la posicion
- **MatchScore**: Puntuacion de compatibilidad
- **Recommendation**: Sugerencia de trabajo
- **SkillGap**: Brecha de habilidades
- **CandidateProfile**: Perfil consolidado para matching

### Modelo de Dominio

```
+------------------------------------------------------------------+
|                     JOB MATCHING CONTEXT                          |
+------------------------------------------------------------------+
|                                                                   |
|  AGGREGATES:                                                      |
|  +-------------+    +------------------+    +----------------+    |
|  |    Job      |    | CandidateProfile |    |   MatchResult  |    |
|  |   (root)    |    |     (root)       |    |    (root)      |    |
|  +-------------+    +------------------+    +----------------+    |
|        |                   |                       |              |
|  +-----v------+     +------v-------+        +------v-------+      |
|  | Requirement|     | SkillVector  |        | SkillGap[]   |      |
|  +------------+     +--------------+        +--------------+      |
|  | Benefit[]  |     | Preferences  |        | Recommendation|     |
|  +------------+     +--------------+        +--------------+      |
|                                                                   |
|  VALUE OBJECTS:                                                   |
|  +---------+ +------------+ +------------+ +----------------+     |
|  | JobId   | | MatchScore | | Salary     | | SkillVector    |     |
|  +---------+ +------------+ +------------+ +----------------+     |
|  | Location| | Experience | | Industry   | | ConfidenceScore|     |
|  +---------+ | Level      | +------------+ +----------------+     |
|             +------------+                                        |
|                                                                   |
+------------------------------------------------------------------+
```

### Agregados

#### Job (Aggregate Root)
```typescript
Job {
  id: JobId
  companyId: CompanyId  // Reference to Identity Context
  title: string
  description: string
  requirements: JobRequirement[]
  benefits: Benefit[]
  salary: SalaryRange
  location: Location
  workMode: WorkMode  // Remote, Hybrid, OnSite
  status: JobStatus
  expiresAt: DateTime

  // Invariantes
  - Debe tener al menos 1 requirement
  - Fecha expiracion debe ser futura
  - Empresa debe tener plan activo
}

JobRequirement {
  skill: Skill
  level: SkillLevel
  isRequired: boolean
  yearsExperience: number (optional)
}
```

#### CandidateProfile (Aggregate Root)
```typescript
CandidateProfile {
  id: CandidateProfileId
  candidateId: UserId
  skillVector: SkillVector  // Embedding del candidato
  preferences: MatchPreferences
  lastUpdated: DateTime

  // Invariantes
  - Se regenera cuando cambia el resume
  - Vector debe tener dimension fija (ej: 768)
}

MatchPreferences {
  desiredRoles: string[]
  salaryExpectation: SalaryRange
  preferredLocations: Location[]
  workModePreference: WorkMode[]
  industries: Industry[]
}
```

#### MatchResult (Aggregate Root)
```typescript
MatchResult {
  id: MatchResultId
  candidateProfileId: CandidateProfileId
  jobId: JobId
  overallScore: MatchScore
  skillGaps: SkillGap[]
  recommendations: Recommendation[]
  calculatedAt: DateTime

  // Invariantes
  - Score entre 0 y 100
  - Debe incluir explicacion de gaps
}
```

### Eventos de Dominio

| Evento | Descripcion | Consumidores |
|--------|-------------|--------------|
| `JobCreated` | Nueva oferta publicada | Matching Engine, Analytics |
| `JobExpired` | Oferta vencida | Notifications |
| `MatchCalculated` | Match candidato-job calculado | Notifications, Analytics |
| `RecommendationsGenerated` | Nuevas recomendaciones | Notifications |
| `CandidateProfileUpdated` | Perfil matching actualizado | Matching Engine |
| `SkillGapsIdentified` | Brechas identificadas | Interview Context |

---

## 4. Interview Context

### Proposito
Gestionar simulaciones de entrevistas con IA, generar feedback y analizar desempeno del candidato.

### Lenguaje Ubicuo
- **InterviewSession**: Sesion de entrevista simulada
- **Question**: Pregunta de entrevista
- **Answer**: Respuesta del candidato
- **Feedback**: Retroalimentacion detallada
- **PerformanceScore**: Puntuacion de desempeno
- **InterviewType**: Tipo de entrevista (tecnica, behavioral, etc.)

### Modelo de Dominio

```
+------------------------------------------------------------------+
|                      INTERVIEW CONTEXT                            |
+------------------------------------------------------------------+
|                                                                   |
|  AGGREGATES:                                                      |
|  +------------------+    +-------------------+                    |
|  | InterviewSession |    | InterviewTemplate |                    |
|  |     (root)       |    |      (root)       |                    |
|  +------------------+    +-------------------+                    |
|          |                        |                               |
|  +-------v--------+       +-------v--------+                      |
|  | Question[]     |       | QuestionBank   |                      |
|  +----------------+       +----------------+                      |
|  | Answer[]       |       | EvaluationCriteria                    |
|  +----------------+       +----------------+                      |
|  | Feedback       |                                               |
|  +----------------+                                               |
|  | Recording      |                                               |
|  +----------------+                                               |
|                                                                   |
|  VALUE OBJECTS:                                                   |
|  +-----------+ +---------------+ +-------------+ +------------+   |
|  | SessionId | | PerformanceScore | | Duration  | | Difficulty |  |
|  +-----------+ +---------------+ +-------------+ +------------+   |
|  | Transcript| | SentimentScore| | BodyLanguage| | VoiceTone  |   |
|  +-----------+ +---------------+ +-------------+ +------------+   |
|                                                                   |
+------------------------------------------------------------------+
```

### Agregados

#### InterviewSession (Aggregate Root)
```typescript
InterviewSession {
  id: SessionId
  candidateId: UserId
  templateId: InterviewTemplateId (optional)
  targetJobId: JobId (optional)
  type: InterviewType
  questions: Question[]
  answers: Answer[]
  feedback: SessionFeedback
  recording: Recording (optional)
  status: SessionStatus
  duration: Duration
  scheduledAt: DateTime
  completedAt: DateTime (optional)

  // Invariantes
  - Minimo 5, maximo 20 preguntas por sesion
  - Duracion maxima 90 minutos
  - Feedback se genera solo al completar
}

Question {
  id: QuestionId
  text: string
  category: QuestionCategory
  difficulty: Difficulty
  expectedTopics: string[]
  timeLimit: Duration (optional)
  order: number
}

Answer {
  questionId: QuestionId
  transcript: string
  audioUrl: URL (optional)
  videoUrl: URL (optional)
  duration: Duration
  sentiment: SentimentScore
  keywords: string[]
  evaluation: AnswerEvaluation
}

SessionFeedback {
  overallScore: PerformanceScore
  categoryScores: Map<Category, Score>
  strengths: string[]
  areasForImprovement: string[]
  detailedFeedback: string
  recommendations: string[]
  comparisonToAverage: number
}
```

#### InterviewTemplate (Aggregate Root)
```typescript
InterviewTemplate {
  id: InterviewTemplateId
  name: string
  type: InterviewType
  industry: Industry (optional)
  role: string (optional)
  questionBank: Question[]
  evaluationCriteria: EvaluationCriteria
  isPublic: boolean
  createdBy: UserId (optional)

  // Invariantes
  - Templates publicos son inmutables
  - Minimo 10 preguntas en el banco
}
```

### Eventos de Dominio

| Evento | Descripcion | Consumidores |
|--------|-------------|--------------|
| `InterviewSessionStarted` | Sesion iniciada | Analytics |
| `InterviewSessionCompleted` | Sesion completada | Analytics, Billing |
| `InterviewSessionAbandoned` | Sesion abandonada | Analytics |
| `FeedbackGenerated` | Feedback listo | Notifications |
| `AnswerRecorded` | Respuesta grabada | AI Processing |
| `PerformanceScoreCalculated` | Score generado | Analytics |

---

## 5. Billing Context

### Proposito
Gestionar suscripciones, planes, pagos y facturacion de la plataforma.

### Lenguaje Ubicuo
- **Subscription**: Suscripcion activa de usuario/empresa
- **Plan**: Tipo de plan con features y limites
- **Invoice**: Factura generada
- **Payment**: Pago realizado
- **UsageQuota**: Cuota de uso consumida
- **PricePoint**: Punto de precio

### Modelo de Dominio

```
+------------------------------------------------------------------+
|                       BILLING CONTEXT                             |
+------------------------------------------------------------------+
|                                                                   |
|  AGGREGATES:                                                      |
|  +---------------+    +-----------+    +---------------+          |
|  | Subscription  |    |   Plan    |    |    Invoice    |          |
|  |    (root)     |    |  (root)   |    |    (root)     |          |
|  +---------------+    +-----------+    +---------------+          |
|         |                   |                  |                  |
|  +------v------+     +------v------+    +------v------+           |
|  | UsageQuota  |     | PlanFeature |    | LineItem    |           |
|  +-------------+     +-------------+    +-------------+           |
|  | BillingCycle|     | PricePoint  |    | Payment     |           |
|  +-------------+     +-------------+    +-------------+           |
|                                                                   |
|  VALUE OBJECTS:                                                   |
|  +----------+ +------------+ +----------+ +----------------+      |
|  | PlanId   | | Currency   | | TaxRate  | | PaymentMethod  |      |
|  +----------+ +------------+ +----------+ +----------------+      |
|  | Amount   | | InvoiceId  | | Period   | | TransactionId  |      |
|  +----------+ +------------+ +----------+ +----------------+      |
|                                                                   |
+------------------------------------------------------------------+
```

### Agregados

#### Subscription (Aggregate Root)
```typescript
Subscription {
  id: SubscriptionId
  customerId: UserId | CompanyId
  customerType: CustomerType
  planId: PlanId
  status: SubscriptionStatus
  currentPeriod: BillingPeriod
  usageQuota: UsageQuota
  paymentMethodId: PaymentMethodId
  trialEndsAt: DateTime (optional)
  cancelledAt: DateTime (optional)

  // Invariantes
  - Solo una suscripcion activa por customer
  - Trial solo para planes pagos
  - Cancelacion efectiva al final del periodo
}

UsageQuota {
  interviewsUsed: number
  interviewsLimit: number
  cvUploadsUsed: number
  cvUploadsLimit: number
  matchingQueriesUsed: number
  matchingQueriesLimit: number
  resetAt: DateTime
}

BillingPeriod {
  startDate: DateTime
  endDate: DateTime
  billingCycle: BillingCycle  // Monthly, Yearly
}
```

#### Plan (Aggregate Root)
```typescript
Plan {
  id: PlanId
  name: string
  tier: PlanTier  // Free, Pro, Enterprise
  targetAudience: TargetAudience  // Candidate, Company
  features: PlanFeature[]
  pricePoints: PricePoint[]
  limits: PlanLimits
  isActive: boolean

  // Invariantes
  - Precio en USD y EUR minimo
  - Free tier sin limites de tiempo
}

PlanFeature {
  code: string
  name: string
  description: string
  isIncluded: boolean
}

PlanLimits {
  maxInterviewsPerMonth: number
  maxCVUploads: number
  maxMatchingQueries: number
  maxRecruiters: number (for Company)
  advancedAnalytics: boolean
  prioritySupport: boolean
}
```

#### Invoice (Aggregate Root)
```typescript
Invoice {
  id: InvoiceId
  subscriptionId: SubscriptionId
  customerId: UserId | CompanyId
  lineItems: LineItem[]
  subtotal: Money
  tax: Money
  total: Money
  status: InvoiceStatus
  dueDate: DateTime
  paidAt: DateTime (optional)
  payments: Payment[]

  // Invariantes
  - Total = Subtotal + Tax
  - Paid cuando payments >= total
}
```

### Eventos de Dominio

| Evento | Descripcion | Consumidores |
|--------|-------------|--------------|
| `SubscriptionCreated` | Nueva suscripcion | Identity, All contexts |
| `SubscriptionUpgraded` | Plan mejorado | All contexts |
| `SubscriptionDowngraded` | Plan reducido | All contexts |
| `SubscriptionCancelled` | Suscripcion cancelada | All contexts |
| `PaymentSucceeded` | Pago exitoso | Notifications |
| `PaymentFailed` | Pago fallido | Notifications |
| `InvoiceGenerated` | Factura creada | Notifications |
| `UsageQuotaExceeded` | Cuota superada | Notifications |
| `TrialEnding` | Trial por vencer | Notifications |

---

## 6. Analytics Context

### Proposito
Recopilar, procesar y presentar metricas, tracking y reportes de la plataforma.

### Lenguaje Ubicuo
- **Event**: Evento de tracking
- **Metric**: Metrica calculada
- **Report**: Reporte generado
- **Dashboard**: Panel de metricas
- **Cohort**: Grupo de usuarios para analisis
- **Funnel**: Embudo de conversion

### Modelo de Dominio

```
+------------------------------------------------------------------+
|                      ANALYTICS CONTEXT                            |
+------------------------------------------------------------------+
|                                                                   |
|  AGGREGATES:                                                      |
|  +-------------+    +-----------+    +-----------------+          |
|  | EventStream |    |  Report   |    | UserAnalytics   |          |
|  |   (root)    |    |  (root)   |    |    (root)       |          |
|  +-------------+    +-----------+    +-----------------+          |
|        |                  |                   |                   |
|  +-----v------+    +------v------+     +------v-------+           |
|  | Event[]    |    | Metric[]    |     | ProgressTrack|           |
|  +------------+    +-------------+     +--------------+           |
|  | Dimension[]|    | Visualization|    | SkillProgress|           |
|  +------------+    +-------------+     +--------------+           |
|                                                                   |
|  VALUE OBJECTS:                                                   |
|  +----------+ +------------+ +-----------+ +---------------+      |
|  | EventId  | | MetricValue| | TimeRange | | Aggregation   |      |
|  +----------+ +------------+ +-----------+ +---------------+      |
|  | Timestamp| | Dimension  | | Granularity| | ChartType    |      |
|  +----------+ +------------+ +-----------+ +---------------+      |
|                                                                   |
+------------------------------------------------------------------+
```

### Agregados

#### EventStream (Aggregate Root)
```typescript
EventStream {
  id: EventStreamId
  entityType: EntityType  // User, Company, System
  entityId: string
  events: AnalyticsEvent[]

  // Invariantes
  - Eventos inmutables una vez creados
  - Retention policy aplicada
}

AnalyticsEvent {
  id: EventId
  type: EventType
  timestamp: DateTime
  properties: Map<string, any>
  dimensions: Dimension[]
  sessionId: string (optional)
  source: EventSource
}
```

#### Report (Aggregate Root)
```typescript
Report {
  id: ReportId
  name: string
  type: ReportType
  owner: UserId | CompanyId
  metrics: Metric[]
  filters: ReportFilter[]
  timeRange: TimeRange
  schedule: ReportSchedule (optional)
  lastGeneratedAt: DateTime

  // Invariantes
  - Minimo 1 metrica
  - TimeRange maximo 1 anio
}

Metric {
  name: string
  aggregation: Aggregation
  dimensions: Dimension[]
  value: MetricValue
  comparison: MetricComparison (optional)
}
```

#### UserAnalytics (Aggregate Root)
```typescript
UserAnalytics {
  id: UserAnalyticsId
  userId: UserId
  progressTrack: ProgressTrack
  skillProgress: SkillProgress[]
  interviewHistory: InterviewAnalytics[]
  engagementScore: number
  predictedChurn: number

  // Invariantes
  - Actualizado en near-real-time
  - Skill progress limitado a skills activos
}

ProgressTrack {
  interviewsCompleted: number
  averageScore: number
  improvementRate: number
  weakAreas: string[]
  strongAreas: string[]
  recommendedFocus: string[]
}
```

### Eventos de Dominio

| Evento | Descripcion | Consumidores |
|--------|-------------|--------------|
| `EventTracked` | Evento registrado | Internal processing |
| `ReportGenerated` | Reporte listo | Notifications |
| `AlertTriggered` | Alerta activada | Notifications |
| `CohortUpdated` | Cohorte recalculado | Internal |
| `ProgressMilestoneReached` | Hito alcanzado | Notifications, Gamification |

---

## Context Map

### Diagrama de Relaciones

```
+------------------------------------------------------------------+
|                         CONTEXT MAP                               |
+------------------------------------------------------------------+
|                                                                   |
|                      +---------------+                            |
|           +--------->|   IDENTITY    |<---------+                 |
|           |          |    CONTEXT    |          |                 |
|           |          +-------+-------+          |                 |
|           |                  |                  |                 |
|           | U/D              | U/D              | U/D             |
|           |                  |                  |                 |
|  +--------+-------+  +-------v-------+  +-------+--------+        |
|  |  CV MANAGEMENT |  |   BILLING     |  |  ANALYTICS     |        |
|  |    CONTEXT     |  |   CONTEXT     |  |   CONTEXT      |        |
|  +--------+-------+  +-------+-------+  +-------+--------+        |
|           |                  |                  |                 |
|           | CS               | U/D              | D               |
|           |                  |                  |                 |
|  +--------v-------+          |          +-------+                 |
|  |  JOB MATCHING  |          |          |                         |
|  |    CONTEXT     |----------+----------+                         |
|  +--------+-------+     U/D  |                                    |
|           |                  |                                    |
|           | CS               |                                    |
|           |                  |                                    |
|  +--------v-------+          |                                    |
|  |   INTERVIEW    |<---------+                                    |
|  |    CONTEXT     |                                               |
|  +----------------+                                               |
|                                                                   |
|  LEGEND:                                                          |
|  U/D = Upstream/Downstream (arrow points to downstream)           |
|  CS  = Conformist/Shared Kernel                                   |
|  D   = Downstream only (consumer)                                 |
|                                                                   |
+------------------------------------------------------------------+
```

### Tipos de Relacion

| Upstream | Downstream | Tipo | Descripcion |
|----------|------------|------|-------------|
| Identity | CV Management | Customer-Supplier | Identity provee UserId |
| Identity | Billing | Customer-Supplier | Identity provee customer info |
| Identity | Job Matching | Customer-Supplier | Identity provee user context |
| Identity | Interview | Customer-Supplier | Identity provee candidate info |
| Identity | Analytics | Published Language | Identity publica eventos |
| CV Management | Job Matching | Conformist | Job Matching consume parsed CVs |
| CV Management | Analytics | Published Language | CV events para tracking |
| Job Matching | Interview | Shared Kernel | Comparten SkillGap concept |
| Billing | All Contexts | Published Language | Plan limits broadcast |
| Interview | Analytics | Published Language | Interview metrics |
| All Contexts | Analytics | Published Language | Eventos de dominio |

---

## Anti-Corruption Layers (ACL)

### ACL: CV Management <- External Parsers

```
+------------------------------------------------------------------+
|                   EXTERNAL PARSER ACL                             |
+------------------------------------------------------------------+
|                                                                   |
|  EXTERNAL PARSER APIs           INTERNAL MODEL                    |
|  (Affinda, Sovren, etc.)        (CV Management)                   |
|                                                                   |
|  +------------------+           +------------------+              |
|  | ExternalResume   |    ACL    |    ParsedData    |              |
|  | {                |  ------>  | {                |              |
|  |   raw_text,      |    ||     |   experiences[], |              |
|  |   extracted_data,|    ||     |   education[],   |              |
|  |   confidence_pct |    ||     |   skills[],      |              |
|  | }                |    ||     |   confidence     |              |
|  +------------------+    ||     | }                |              |
|                          ||     +------------------+              |
|                          ||                                       |
|                    +-----------+                                  |
|                    | Translator|                                  |
|                    | Adapter   |                                  |
|                    +-----------+                                  |
|                                                                   |
+------------------------------------------------------------------+
```

```typescript
// ACL Implementation
class ExternalParserACL {
  translate(externalResult: ExternalParseResult): ParsedData {
    return {
      experiences: this.mapExperiences(externalResult.work_history),
      education: this.mapEducation(externalResult.education_entries),
      skills: this.normalizeSkills(externalResult.extracted_skills),
      confidence: externalResult.confidence_pct / 100
    };
  }

  private normalizeSkills(external: ExternalSkill[]): Skill[] {
    // Mapeo a taxonomia interna de skills
  }
}
```

### ACL: Billing <- Payment Gateways

```
+------------------------------------------------------------------+
|                    PAYMENT GATEWAY ACL                            |
+------------------------------------------------------------------+
|                                                                   |
|  STRIPE/PAYPAL APIs             INTERNAL MODEL                    |
|                                 (Billing)                         |
|                                                                   |
|  +------------------+           +------------------+              |
|  | StripeCharge     |    ACL    |    Payment       |              |
|  | {                |  ------>  | {                |              |
|  |   id,            |    ||     |   id,            |              |
|  |   amount_cents,  |    ||     |   amount: Money, |              |
|  |   currency,      |    ||     |   status,        |              |
|  |   status,        |    ||     |   paidAt         |              |
|  |   metadata       |    ||     | }                |              |
|  | }                |    ||     +------------------+              |
|  +------------------+    ||                                       |
|                    +-----------+                                  |
|                    | Payment   |                                  |
|                    | Adapter   |                                  |
|                    +-----------+                                  |
|                                                                   |
+------------------------------------------------------------------+
```

### ACL: Interview <- AI Providers

```
+------------------------------------------------------------------+
|                      AI PROVIDER ACL                              |
+------------------------------------------------------------------+
|                                                                   |
|  OPENAI/ANTHROPIC APIs          INTERNAL MODEL                    |
|                                 (Interview)                       |
|                                                                   |
|  +------------------+           +------------------+              |
|  | ChatCompletion   |    ACL    |  SessionFeedback |              |
|  | {                |  ------>  | {                |              |
|  |   choices[],     |    ||     |   overallScore,  |              |
|  |   usage,         |    ||     |   strengths[],   |              |
|  |   model          |    ||     |   improvements[],|              |
|  | }                |    ||     |   detailed       |              |
|  +------------------+    ||     | }                |              |
|                    +-----------+ +------------------+             |
|                    | Feedback  |                                  |
|                    | Generator |                                  |
|                    +-----------+                                  |
|                                                                   |
+------------------------------------------------------------------+
```

---

## User Journey Mapping

### Candidate Journey

```
+------------------------------------------------------------------+
|                     CANDIDATE USER JOURNEY                        |
+------------------------------------------------------------------+
|                                                                   |
|  1. REGISTRO          2. UPLOAD CV         3. MATCHING            |
|  +-----------+        +------------+       +------------+         |
|  | Identity  |  --->  |    CV      |  ---> |    Job     |         |
|  | Context   |        | Management |       |  Matching  |         |
|  +-----------+        +------------+       +------------+         |
|       |                     |                    |                |
|  UserRegistered        ResumeParsed        MatchCalculated        |
|       |                     |                    |                |
|       v                     v                    v                |
|  +----+----+          +-----+-----+        +-----+-----+          |
|  |Analytics|          | Analytics |        | Analytics |          |
|  +---------+          +-----------+        +-----------+          |
|                                                                   |
|                                                                   |
|  4. SIMULACION        5. FEEDBACK          6. PAGO                |
|  +------------+       +------------+       +------------+         |
|  | Interview  |  ---> | Interview  |  ---> |  Billing   |         |
|  |  Context   |       |  Context   |       |  Context   |         |
|  +------------+       +------------+       +------------+         |
|       |                     |                    |                |
|  SessionStarted       FeedbackGenerated    PaymentSucceeded       |
|       |                     |                    |                |
|       v                     v                    v                |
|  +----+----+          +-----+-----+        +-----+-----+          |
|  |Analytics|          | Analytics |        | Analytics |          |
|  +---------+          +-----------+        +-----------+          |
|                                                                   |
+------------------------------------------------------------------+
```

### Company Journey

```
+------------------------------------------------------------------+
|                      COMPANY USER JOURNEY                         |
+------------------------------------------------------------------+
|                                                                   |
|  1. REGISTRO          2. SETUP             3. INVITAR             |
|  +-----------+        +------------+       +------------+         |
|  | Identity  |  --->  |  Identity  |  ---> | Identity   |         |
|  | Context   |        |  Context   |       | Context    |         |
|  +-----------+        +------------+       +------------+         |
|       |                     |                    |                |
|  CompanyCreated       CompanyConfigured    CandidateInvited       |
|       |                     |                    |                |
|       v                     v                    v                |
|  +----+----+          +-----+-----+        +-----+-----+          |
|  | Billing  |         | Billing   |        |    CV     |          |
|  | (Trial)  |         | (Plan)    |        | Management|          |
|  +---------+          +-----------+        +-----------+          |
|                                                                   |
|                                                                   |
|  4. VER REPORTES      5. PAGAR                                    |
|  +------------+       +------------+                              |
|  | Analytics  |  ---> |  Billing   |                              |
|  |  Context   |       |  Context   |                              |
|  +------------+       +------------+                              |
|       |                     |                                     |
|  ReportGenerated      PaymentSucceeded                            |
|       |                     |                                     |
|       v                     v                                     |
|  +----+----+          +-----+-----+                               |
|  | Interview|         | Analytics |                               |
|  | (Results)|         +-----------+                               |
|  +---------+                                                      |
|                                                                   |
+------------------------------------------------------------------+
```

---

## Shared Kernel

### Skills Domain

Compartido entre CV Management, Job Matching e Interview:

```typescript
// Shared Kernel: Skills
namespace SharedKernel.Skills {

  interface Skill {
    id: SkillId
    name: string
    category: SkillCategory
    aliases: string[]
  }

  enum SkillCategory {
    Technical = 'TECHNICAL',
    Soft = 'SOFT',
    Language = 'LANGUAGE',
    Tool = 'TOOL',
    Framework = 'FRAMEWORK',
    Methodology = 'METHODOLOGY'
  }

  interface SkillLevel {
    skill: Skill
    proficiency: Proficiency  // Beginner, Intermediate, Advanced, Expert
    yearsExperience: number
    lastUsed: Date
  }

  interface SkillGap {
    required: SkillLevel
    current: SkillLevel | null
    gapSeverity: GapSeverity  // Critical, Important, Nice-to-have
    recommendations: string[]
  }
}
```

---

## Consecuencias

### Positivas

1. **Autonomia de equipos**: Cada contexto puede ser desarrollado independientemente
2. **Escalabilidad**: Contextos pueden escalar de forma independiente
3. **Claridad conceptual**: Vocabulario y reglas claras por contexto
4. **Evolucion controlada**: Cambios encapsulados, ACLs protegen de cambios externos
5. **Testing aislado**: Cada contexto testeable de forma independiente

### Negativas

1. **Complejidad inicial**: Mayor esfuerzo de setup y coordinacion
2. **Duplicacion controlada**: Algunos conceptos duplicados entre contextos
3. **Latencia**: Comunicacion entre contextos anade latencia
4. **Consistencia eventual**: No hay transacciones ACID cross-context

### Mitigaciones

| Riesgo | Mitigacion |
|--------|------------|
| Complejidad | Documentacion clara, ADRs, diagramas actualizados |
| Duplicacion | Shared Kernel para conceptos criticos |
| Latencia | Event-driven async, caching, CQRS |
| Consistencia | Sagas, eventos de compensacion |

---

## Referencias

- Evans, Eric. "Domain-Driven Design: Tackling Complexity in the Heart of Software"
- Vernon, Vaughn. "Implementing Domain-Driven Design"
- Context Mapping Pattern: https://www.infoq.com/articles/ddd-contextmapping/
- ADR-001: Arquitectura Base (referencia interna)

---

## Apendice A: Event Storming Results

```
+------------------------------------------------------------------+
|                    EVENT STORMING SUMMARY                         |
+------------------------------------------------------------------+
|                                                                   |
|  IDENTITY CONTEXT:                                                |
|  [UserRegistered] [UserAuthenticated] [CompanyCreated]            |
|  [RecruiterInvited] [CandidateInvited] [SubscriptionChanged]      |
|                                                                   |
|  CV MANAGEMENT CONTEXT:                                           |
|  [ResumeUploaded] [ResumeParsed] [ResumeParsingFailed]            |
|  [ATSScoreCalculated] [OptimizationCompleted]                     |
|                                                                   |
|  JOB MATCHING CONTEXT:                                            |
|  [JobCreated] [JobExpired] [MatchCalculated]                      |
|  [RecommendationsGenerated] [SkillGapsIdentified]                 |
|                                                                   |
|  INTERVIEW CONTEXT:                                               |
|  [SessionStarted] [SessionCompleted] [SessionAbandoned]           |
|  [AnswerRecorded] [FeedbackGenerated]                             |
|                                                                   |
|  BILLING CONTEXT:                                                 |
|  [SubscriptionCreated] [PaymentSucceeded] [PaymentFailed]         |
|  [InvoiceGenerated] [UsageQuotaExceeded] [TrialEnding]            |
|                                                                   |
|  ANALYTICS CONTEXT:                                               |
|  [EventTracked] [ReportGenerated] [AlertTriggered]                |
|  [ProgressMilestoneReached]                                       |
|                                                                   |
+------------------------------------------------------------------+
```

---

## Apendice B: Glossary

| Termino | Contexto | Definicion |
|---------|----------|------------|
| User | Identity | Persona registrada en el sistema |
| Candidate | Identity | Usuario que busca empleo |
| Company | Identity | Organizacion que contrata |
| Resume | CV Management | Documento CV del candidato |
| ParsedData | CV Management | CV procesado y estructurado |
| Job | Job Matching | Oferta de empleo |
| MatchScore | Job Matching | Puntuacion de compatibilidad |
| InterviewSession | Interview | Sesion de entrevista simulada |
| Subscription | Billing | Suscripcion activa |
| Plan | Billing | Tipo de plan con features |
| Event | Analytics | Evento de tracking |
| Report | Analytics | Reporte generado |
