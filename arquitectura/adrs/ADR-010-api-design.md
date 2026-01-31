# ADR-010: Estrategia de Diseno de API

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead

---

## Contexto

### Consumidores de API

La plataforma "Entrevistador Inteligente" debe servir a multiples consumidores con diferentes necesidades:

| Consumidor | Caracteristicas | Prioridad |
|------------|-----------------|-----------|
| **Web App** (React/Next.js) | Alto volumen, SSR, rich interactions | MVP |
| **Mobile App** (React Native) | Offline support, payloads pequenos, bateria | MVP |
| **WhatsApp Bot** | Mensajes cortos, webhooks, estado conversacional | Fase 2 |
| **Integraciones B2B** (ATS externos) | Bulk operations, webhooks, alta fiabilidad | Fase 2 |
| **Public API** (Partners) | Rate limiting estricto, documentacion clara, versionado | Fase 3 |

### Requisitos Criticos

1. **Mobile-first**: Payloads optimizados, respuestas rapidas
2. **Offline support**: Caching agresivo, sync incremental
3. **Real-time**: WebSockets para simulaciones de entrevista
4. **Escalabilidad B2B**: Bulk operations, webhooks confiables
5. **Developer Experience**: SDKs, documentacion, sandbox

### Endpoints por Dominio

```
Identity Context:
  POST /auth/register
  POST /auth/login
  POST /auth/verify-otp
  POST /auth/refresh
  POST /auth/logout
  GET  /users/me
  PUT  /users/me

CV Context:
  POST /cvs/upload
  GET  /cvs
  GET  /cvs/{id}
  PUT  /cvs/{id}
  DELETE /cvs/{id}
  GET  /cvs/{id}/analysis
  POST /cvs/{id}/optimize
  GET  /cvs/{id}/versions

Matching Context:
  GET  /jobs
  GET  /jobs/{id}
  GET  /jobs/recommendations
  POST /jobs/{id}/apply
  GET  /applications
  GET  /applications/{id}
  GET  /matches
  GET  /matches/{id}/score-breakdown

Interview Context:
  POST /interviews
  GET  /interviews
  GET  /interviews/{id}
  POST /interviews/{id}/start
  POST /interviews/{id}/respond
  POST /interviews/{id}/end
  GET  /interviews/{id}/feedback
  GET  /interviews/{id}/transcript

Billing Context:
  GET  /subscriptions/plans
  GET  /subscriptions/current
  POST /subscriptions
  PUT  /subscriptions/{id}
  DELETE /subscriptions/{id}
  POST /payments/checkout
  GET  /payments/history
  GET  /invoices/{id}
```

---

## Decision

### 1. Estilo de API: REST con JSON:API + GraphQL para BFF

**Adoptamos un enfoque hibrido:**

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
│                    (Rate Limiting, Auth)                         │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  REST API     │   │  GraphQL BFF  │   │  WebSocket    │
│  (Public/B2B) │   │  (Web/Mobile) │   │  (Real-time)  │
│  /api/v1/*    │   │  /graphql     │   │  /ws          │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
                              ▼
                ┌───────────────────────┐
                │   Modular Monolith    │
                │   (Domain Services)   │
                └───────────────────────┘
```

| Capa | Tecnologia | Uso Principal |
|------|------------|---------------|
| **REST API** | Express + OpenAPI 3.1 | Public API, B2B, Webhooks |
| **GraphQL BFF** | Apollo Server | Web App, Mobile App |
| **WebSocket** | Socket.io | Entrevistas real-time |
| **tRPC** | No adoptado | Descartado (ver alternativas) |

#### Justificacion REST para API Publica/B2B

1. **Universalidad**: Cualquier lenguaje/plataforma puede consumirla
2. **Caching HTTP nativo**: CDN, proxies, navegadores
3. **Herramientas maduras**: Postman, curl, OpenAPI ecosystem
4. **ATS Integration**: La mayoria espera REST
5. **Webhooks naturales**: Patron bien establecido

#### Justificacion GraphQL para BFF (Web/Mobile)

1. **Fetch exacto**: Solo los campos necesarios (mobile-first)
2. **Batching**: Multiples queries en una request
3. **Strong typing**: Generacion automatica de tipos para React/RN
4. **Real-time**: Subscriptions integradas
5. **Evolucion sin versionado**: Agregar campos sin breaking changes

### 2. Versionado de API

**Estrategia: URL Path Versioning para REST, Schema Evolution para GraphQL**

```
REST (versionado explicito):
  /api/v1/users/me
  /api/v2/users/me  (cuando sea necesario)

GraphQL (evolucion continua):
  /graphql  (siempre mismo endpoint)
  Deprecation via @deprecated directive
```

#### Politica de Versionado REST

| Aspecto | Politica |
|---------|----------|
| **Formato** | `/api/v{major}/resource` |
| **Breaking changes** | Nueva version major |
| **Soporte paralelo** | Minimo 12 meses para version anterior |
| **Sunset header** | Obligatorio 6 meses antes de deprecation |
| **Changelog** | Publicado en cada release |

```http
# Ejemplo de Sunset Header
HTTP/1.1 200 OK
Sunset: Sat, 31 Jan 2027 23:59:59 GMT
Deprecation: true
Link: </api/v2/users>; rel="successor-version"
```

#### Politica de Evolucion GraphQL

```graphql
type User {
  id: ID!
  email: String!
  fullName: String!
  # Campo deprecado - usar fullName
  name: String @deprecated(reason: "Use fullName instead. Will be removed 2027-06-01")
}
```

### 3. Autenticacion y Autorizacion

**Estrategia Hibrida: JWT + API Keys**

```
┌─────────────────────────────────────────────────────────────┐
│                     Authentication Flow                      │
└─────────────────────────────────────────────────────────────┘

Usuarios (Web/Mobile):
┌──────┐    ┌──────────┐    ┌─────────────┐    ┌──────────┐
│Client│───►│  Login   │───►│ JWT Access  │───►│  API     │
│      │    │ (OAuth2) │    │ + Refresh   │    │ Requests │
└──────┘    └──────────┘    └─────────────┘    └──────────┘

B2B / Partners:
┌──────┐    ┌──────────┐    ┌─────────────┐    ┌──────────┐
│Client│───►│ API Key  │───►│ Key in      │───►│  API     │
│      │    │ Portal   │    │ Header      │    │ Requests │
└──────┘    └──────────┘    └─────────────┘    └──────────┘
```

#### JWT para Usuarios (B2C/B2B Users)

```typescript
// Access Token (corta duracion)
interface AccessToken {
  sub: string;           // User ID
  email: string;
  roles: string[];       // ['candidate', 'recruiter', 'admin']
  permissions: string[]; // ['cv:read', 'cv:write', 'interview:create']
  org_id?: string;       // Para usuarios B2B
  iat: number;
  exp: number;           // 15 minutos
}

// Refresh Token (larga duracion, rotacion)
interface RefreshToken {
  sub: string;
  jti: string;           // Token ID para revocacion
  family: string;        // Familia para detectar reutilizacion
  iat: number;
  exp: number;           // 7 dias
}
```

| Aspecto | Configuracion |
|---------|---------------|
| **Access Token TTL** | 15 minutos |
| **Refresh Token TTL** | 7 dias (30 dias con "remember me") |
| **Algoritmo** | RS256 (asimetrico) |
| **Refresh Rotation** | Si, con deteccion de reutilizacion |
| **Revocacion** | Blacklist en Redis |

#### API Keys para Integraciones B2B

```typescript
interface ApiKey {
  key_id: string;        // Prefijo publico: ei_live_xxxx
  key_hash: string;      // Argon2 hash del secret
  name: string;          // "ATS Integration - BuenTrabajo"
  org_id: string;
  scopes: string[];      // ['jobs:read', 'applications:write']
  rate_limit: number;    // Requests por minuto
  ip_whitelist?: string[];
  created_at: Date;
  last_used_at: Date;
  expires_at?: Date;
}
```

```http
# Uso de API Key
GET /api/v1/jobs HTTP/1.1
Authorization: Bearer ei_live_xxxxxxxxxxxxxxxxxxxx
X-API-Key-ID: ei_live_abc123
```

#### OAuth 2.0 para Partners (Fase 3)

```
┌──────────┐                              ┌──────────────┐
│  Partner │                              │ Entrevistador│
│   App    │                              │ Inteligente  │
└────┬─────┘                              └──────┬───────┘
     │                                           │
     │  1. Authorization Request                 │
     │  GET /oauth/authorize?                    │
     │    client_id=xxx&                         │
     │    redirect_uri=xxx&                      │
     │    scope=cv:read jobs:read&               │
     │    state=xyz                              │
     │──────────────────────────────────────────►│
     │                                           │
     │  2. User Grants Permission                │
     │◄──────────────────────────────────────────│
     │                                           │
     │  3. Authorization Code                    │
     │  302 Redirect to redirect_uri?code=xxx   │
     │◄──────────────────────────────────────────│
     │                                           │
     │  4. Exchange Code for Tokens              │
     │  POST /oauth/token                        │
     │──────────────────────────────────────────►│
     │                                           │
     │  5. Access + Refresh Tokens               │
     │◄──────────────────────────────────────────│
```

### 4. Rate Limiting

**Estrategia: Token Bucket con limites diferenciados**

```
┌─────────────────────────────────────────────────────────────┐
│                    Rate Limiting Tiers                       │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────┐
                    │   Request   │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Identify    │
                    │ Client Tier │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   Free Tier   │  │  Premium B2C  │  │   B2B/API     │
│  60 req/min   │  │  300 req/min  │  │ 1000 req/min  │
│  10 AI/hour   │  │  100 AI/hour  │  │ Custom/Plan   │
└───────────────┘  └───────────────┘  └───────────────┘
```

| Tier | Requests/min | AI Operations/hour | Burst | Identificador |
|------|--------------|-------------------|-------|---------------|
| **Anonymous** | 20 | 0 | 40 | IP |
| **Free** | 60 | 10 | 120 | User ID |
| **Premium** | 300 | 100 | 600 | User ID |
| **Business** | 600 | 300 | 1200 | Org ID |
| **Enterprise** | 1000+ | Custom | Custom | API Key |

#### Headers de Rate Limit

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 299
X-RateLimit-Reset: 1706745600
X-RateLimit-Policy: "300;w=60"
Retry-After: 30  # Solo cuando se excede
```

#### Rate Limit por Endpoint (AI-Heavy)

```typescript
const rateLimitConfig = {
  // Endpoints standard
  'GET /api/v1/*': { points: 1 },
  'POST /api/v1/*': { points: 2 },

  // Endpoints AI-intensive (consumen mas cuota)
  'POST /api/v1/cvs/upload': { points: 10 },
  'POST /api/v1/cvs/{id}/optimize': { points: 20 },
  'POST /api/v1/interviews': { points: 30 },
  'POST /api/v1/interviews/{id}/respond': { points: 5 },

  // Bulk operations B2B
  'POST /api/v1/jobs/bulk': { points: 50 },
};
```

### 5. Documentacion de API

**Stack de Documentacion:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Documentation Stack                       │
└─────────────────────────────────────────────────────────────┘

┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   OpenAPI     │    │   GraphQL     │    │   Redoc /     │
│   3.1 Spec    │    │   Schema +    │    │   Swagger UI  │
│   (REST)      │    │   Apollo      │    │   (Portal)    │
└───────┬───────┘    └───────┬───────┘    └───────────────┘
        │                    │
        ▼                    ▼
┌───────────────────────────────────────┐
│           SDK Generation              │
│  TypeScript | Python | Go | Java      │
└───────────────────────────────────────┘
```

#### OpenAPI 3.1 Specification

```yaml
# openapi.yaml (extracto)
openapi: 3.1.0
info:
  title: Entrevistador Inteligente API
  version: 1.0.0
  description: |
    API para la plataforma de busqueda de empleo con IA.

    ## Autenticacion
    - **JWT Bearer**: Para usuarios autenticados
    - **API Key**: Para integraciones B2B

    ## Rate Limits
    Ver seccion de rate limiting para detalles por tier.
  contact:
    email: api@entrevistadorinteligente.pe
  license:
    name: Proprietary

servers:
  - url: https://api.entrevistadorinteligente.pe/v1
    description: Production
  - url: https://sandbox.entrevistadorinteligente.pe/v1
    description: Sandbox

tags:
  - name: Authentication
    description: User authentication and authorization
  - name: CVs
    description: CV management and AI optimization
  - name: Jobs
    description: Job listings and recommendations
  - name: Interviews
    description: AI interview simulations
  - name: Billing
    description: Subscriptions and payments

paths:
  /auth/login:
    post:
      tags: [Authentication]
      summary: User login
      operationId: login
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LoginRequest'
      responses:
        '200':
          description: Successful login
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthTokens'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /cvs/{id}/optimize:
    post:
      tags: [CVs]
      summary: Optimize CV with AI
      operationId: optimizeCV
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OptimizeRequest'
      responses:
        '202':
          description: Optimization started
          headers:
            Location:
              schema:
                type: string
              description: URL to check optimization status
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AsyncJobResponse'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

  schemas:
    LoginRequest:
      type: object
      required: [email, password]
      properties:
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
        remember_me:
          type: boolean
          default: false

    AuthTokens:
      type: object
      properties:
        access_token:
          type: string
        refresh_token:
          type: string
        expires_in:
          type: integer
        token_type:
          type: string
          enum: [Bearer]

  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

#### GraphQL Schema (Extracto)

```graphql
# schema.graphql
type Query {
  # User
  me: User!

  # CVs
  cvs(first: Int, after: String): CVConnection!
  cv(id: ID!): CV

  # Jobs
  jobs(
    filter: JobFilterInput
    first: Int
    after: String
  ): JobConnection!

  jobRecommendations(
    cvId: ID!
    limit: Int = 10
  ): [JobRecommendation!]!

  # Interviews
  interviews(
    status: InterviewStatus
    first: Int
    after: String
  ): InterviewConnection!

  interview(id: ID!): Interview
}

type Mutation {
  # Auth
  login(input: LoginInput!): AuthPayload!
  refreshToken(token: String!): AuthPayload!

  # CV
  uploadCV(input: UploadCVInput!): CV!
  optimizeCV(id: ID!, input: OptimizeCVInput!): AsyncJob!

  # Interview
  createInterview(input: CreateInterviewInput!): Interview!
  respondToInterview(
    interviewId: ID!
    input: InterviewResponseInput!
  ): InterviewResponse!
}

type Subscription {
  # Real-time interview
  interviewMessage(interviewId: ID!): InterviewMessage!

  # CV optimization progress
  cvOptimizationProgress(jobId: ID!): OptimizationProgress!
}

type User {
  id: ID!
  email: String!
  fullName: String!
  phone: String
  subscription: Subscription
  cvs: [CV!]!
  interviews: [Interview!]!
  createdAt: DateTime!
}

type CV {
  id: ID!
  filename: String!
  status: CVStatus!
  analysis: CVAnalysis
  versions: [CVVersion!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type CVAnalysis {
  overallScore: Float!
  sections: [SectionAnalysis!]!
  keywords: [Keyword!]!
  suggestions: [Suggestion!]!
  atsCompatibility: Float!
}

type Interview {
  id: ID!
  type: InterviewType!
  status: InterviewStatus!
  job: Job
  messages: [InterviewMessage!]!
  feedback: InterviewFeedback
  startedAt: DateTime
  completedAt: DateTime
}

enum InterviewType {
  BEHAVIORAL
  TECHNICAL
  CASE_STUDY
  COMPETENCY
}

enum InterviewStatus {
  PENDING
  IN_PROGRESS
  COMPLETED
  ABANDONED
}
```

### 6. Generacion de SDKs

**Estrategia: Generacion automatica desde especificaciones**

```
┌─────────────────────────────────────────────────────────────┐
│                    SDK Generation Pipeline                   │
└─────────────────────────────────────────────────────────────┘

┌───────────────┐         ┌───────────────┐
│   OpenAPI     │────────►│   openapi-    │
│   3.1 Spec    │         │   generator   │
└───────────────┘         └───────┬───────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│  TypeScript   │       │    Python     │       │     Java      │
│     SDK       │       │     SDK       │       │     SDK       │
└───────────────┘       └───────────────┘       └───────────────┘

┌───────────────┐         ┌───────────────┐
│   GraphQL     │────────►│   graphql-    │
│   Schema      │         │   codegen     │
└───────────────┘         └───────┬───────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│  React Hooks  │       │  React Native │       │   Apollo      │
│  + Types      │       │   + Types     │       │   Client      │
└───────────────┘       └───────────────┘       └───────────────┘
```

#### SDKs Generados

| SDK | Uso | Generador |
|-----|-----|-----------|
| **@ei/api-client** | TypeScript/JS | openapi-typescript-codegen |
| **@ei/react-hooks** | React Web | graphql-codegen |
| **@ei/react-native** | Mobile | graphql-codegen + custom |
| **ei-python** | Python integrations | openapi-python-client |
| **ei-java** | Java/Android | openapi-generator |

```typescript
// Ejemplo de uso: @ei/api-client
import { EntrevistadorClient } from '@ei/api-client';

const client = new EntrevistadorClient({
  apiKey: 'ei_live_xxxxx',
  environment: 'production',
});

// Typed request/response
const cv = await client.cvs.upload({
  file: cvFile,
  optimizeOnUpload: true,
});

// With pagination
const jobs = await client.jobs.list({
  location: 'Lima',
  remote: true,
  page: 1,
  limit: 20,
});
```

```typescript
// Ejemplo de uso: @ei/react-hooks (GraphQL)
import { useMyProfile, useOptimizeCvMutation } from '@ei/react-hooks';

function ProfilePage() {
  const { data, loading } = useMyProfile();
  const [optimizeCv, { loading: optimizing }] = useOptimizeCvMutation();

  const handleOptimize = async (cvId: string) => {
    const result = await optimizeCv({
      variables: { id: cvId, input: { targetJob: 'Software Engineer' } },
    });
    // Typed result
    console.log(result.data?.optimizeCV.jobId);
  };

  return (/* ... */);
}
```

### 7. WebSocket para Real-time (Entrevistas)

```
┌─────────────────────────────────────────────────────────────┐
│                  WebSocket Architecture                      │
└─────────────────────────────────────────────────────────────┘

Client                    Server                    AI Service
  │                         │                           │
  │  1. WS Connect          │                           │
  │  /ws?token=xxx          │                           │
  │────────────────────────►│                           │
  │                         │                           │
  │  2. Subscribe           │                           │
  │  {type:"subscribe",     │                           │
  │   interview_id:"xxx"}   │                           │
  │────────────────────────►│                           │
  │                         │                           │
  │  3. User Message        │                           │
  │  {type:"message",       │  4. Process              │
  │   content:"..."}        │────────────────────────►  │
  │────────────────────────►│                           │
  │                         │  5. AI Response           │
  │  6. AI Message          │◄────────────────────────  │
  │  {type:"message",       │                           │
  │   content:"...",        │                           │
  │   is_ai:true}           │                           │
  │◄────────────────────────│                           │
  │                         │                           │
  │  7. Typing Indicator    │                           │
  │  {type:"typing",        │                           │
  │   is_typing:true}       │                           │
  │◄────────────────────────│                           │
```

#### Protocolo WebSocket

```typescript
// Mensajes del cliente
type ClientMessage =
  | { type: 'subscribe'; interview_id: string }
  | { type: 'unsubscribe'; interview_id: string }
  | { type: 'message'; interview_id: string; content: string }
  | { type: 'ping' };

// Mensajes del servidor
type ServerMessage =
  | { type: 'subscribed'; interview_id: string }
  | { type: 'message'; interview_id: string; message: InterviewMessage }
  | { type: 'typing'; interview_id: string; is_typing: boolean }
  | { type: 'interview_ended'; interview_id: string; feedback_ready: boolean }
  | { type: 'error'; code: string; message: string }
  | { type: 'pong' };

interface InterviewMessage {
  id: string;
  content: string;
  is_ai: boolean;
  timestamp: string;
  metadata?: {
    sentiment?: 'positive' | 'neutral' | 'negative';
    topics?: string[];
  };
}
```

#### Fallback para Offline/Degraded

```typescript
// Cliente con fallback
class InterviewClient {
  private ws: WebSocket | null = null;
  private messageQueue: Message[] = [];

  async sendMessage(content: string) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      // Enviar via WebSocket
      this.ws.send(JSON.stringify({ type: 'message', content }));
    } else {
      // Fallback a REST
      await this.httpClient.post(`/interviews/${this.interviewId}/respond`, {
        content,
      });
      // Queue para sync cuando reconecte
      this.messageQueue.push({ content, timestamp: Date.now() });
    }
  }
}
```

### 8. Estructura de Respuestas

#### REST API (JSON:API inspirado)

```json
// Respuesta exitosa singular
{
  "data": {
    "id": "cv_123",
    "type": "cv",
    "attributes": {
      "filename": "mi_cv.pdf",
      "status": "analyzed",
      "created_at": "2026-01-31T10:00:00Z"
    },
    "relationships": {
      "analysis": {
        "data": { "id": "analysis_456", "type": "cv_analysis" }
      }
    }
  },
  "included": [
    {
      "id": "analysis_456",
      "type": "cv_analysis",
      "attributes": {
        "overall_score": 78.5,
        "ats_compatibility": 85
      }
    }
  ],
  "meta": {
    "request_id": "req_abc123"
  }
}

// Respuesta de coleccion
{
  "data": [
    { "id": "job_1", "type": "job", "attributes": {...} },
    { "id": "job_2", "type": "job", "attributes": {...} }
  ],
  "meta": {
    "total": 150,
    "page": 1,
    "per_page": 20,
    "request_id": "req_xyz789"
  },
  "links": {
    "self": "/api/v1/jobs?page=1",
    "next": "/api/v1/jobs?page=2",
    "last": "/api/v1/jobs?page=8"
  }
}

// Respuesta de error
{
  "errors": [
    {
      "code": "VALIDATION_ERROR",
      "title": "Invalid input",
      "detail": "Email format is invalid",
      "source": { "pointer": "/data/attributes/email" },
      "meta": {
        "expected_format": "email@domain.com"
      }
    }
  ],
  "meta": {
    "request_id": "req_err456"
  }
}
```

#### GraphQL Errors

```json
{
  "data": null,
  "errors": [
    {
      "message": "CV not found",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["cv"],
      "extensions": {
        "code": "NOT_FOUND",
        "http": { "status": 404 }
      }
    }
  ]
}
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Flexibilidad de consumo** | REST para B2B, GraphQL para apps propias |
| **Mobile optimizado** | GraphQL permite fetch exacto de campos |
| **Developer experience** | SDKs auto-generados, documentacion interactiva |
| **Escalabilidad B2B** | Rate limiting granular, API keys con scopes |
| **Real-time nativo** | WebSockets para entrevistas, GraphQL subscriptions |
| **Evolucion controlada** | Versionado URL para REST, deprecation para GraphQL |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Complejidad dual** | GraphQL solo para BFF interno, REST para externo |
| **Overhead de GraphQL** | Caching con Apollo, persisted queries |
| **Mantenimiento SDKs** | Generacion automatica en CI/CD |
| **Learning curve** | Documentacion interna, workshops |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Over-fetching REST para mobile | Media | Medio | Campos sparse via query params |
| N+1 en GraphQL | Alta | Alto | DataLoader obligatorio |
| API abuse | Media | Alto | Rate limiting + WAF |
| Breaking changes accidentales | Baja | Alto | Contract testing, versiones |

---

## Alternativas Consideradas

### Opcion A: Solo REST

**Pros:**
- Simplicidad, un solo paradigma
- Caching HTTP nativo
- Tooling universal

**Contras:**
- Over-fetching para mobile
- Multiples requests para vistas complejas
- Versionado rigido

**Veredicto:** Rechazado - No optimiza para mobile-first

### Opcion B: Solo GraphQL

**Pros:**
- Fetch exacto siempre
- Un solo endpoint
- Strong typing

**Contras:**
- B2B/ATS esperan REST
- Caching mas complejo
- Learning curve para partners

**Veredicto:** Rechazado - No viable para integraciones B2B

### Opcion C: tRPC

**Pros:**
- Type safety end-to-end
- Zero boilerplate
- Excelente DX para TypeScript

**Contras:**
- Solo TypeScript (no Java, Python SDKs)
- No standard para B2B
- Acoplamiento cliente-servidor

**Veredicto:** Rechazado - Limita ecosistema de integraciones

### Opcion D: gRPC

**Pros:**
- Alto rendimiento
- Contratos estrictos (protobuf)
- Streaming bidireccional

**Contras:**
- No nativo en navegadores (requiere proxy)
- Tooling menos maduro para web
- Complejidad para equipo pequeno

**Veredicto:** Rechazado - Overhead innecesario, no browser-native

---

## Implementacion

### Fase 1: MVP (Semanas 1-12)

```
Semanas 1-4:
- [ ] Setup API Gateway con rate limiting basico
- [ ] Implementar endpoints REST core (auth, cvs, jobs)
- [ ] OpenAPI spec inicial
- [ ] JWT authentication

Semanas 5-8:
- [ ] GraphQL BFF para web/mobile
- [ ] WebSocket para entrevistas
- [ ] SDK TypeScript auto-generado
- [ ] Documentacion Swagger UI

Semanas 9-12:
- [ ] Rate limiting avanzado
- [ ] API Keys para B2B
- [ ] Monitoring y alertas
- [ ] Sandbox environment
```

### Fase 2: Crecimiento (Meses 4-8)

```
- [ ] OAuth 2.0 para partners
- [ ] SDKs Python y Java
- [ ] Webhooks para B2B
- [ ] API analytics dashboard
- [ ] Versionado v2 si necesario
```

### Fase 3: Escala (Meses 9-12)

```
- [ ] Public API portal
- [ ] Developer program
- [ ] Rate limiting por endpoint
- [ ] SLA monitoring
- [ ] Multi-region API
```

---

## Criterios de Exito

| Metrica | Objetivo MVP | Objetivo Ano 1 |
|---------|--------------|----------------|
| Latencia P50 REST | < 100ms | < 50ms |
| Latencia P95 REST | < 300ms | < 150ms |
| Latencia P50 GraphQL | < 150ms | < 100ms |
| WebSocket latency | < 200ms | < 100ms |
| Uptime API | 99.5% | 99.9% |
| SDK adoption | 80% partners | 95% partners |
| Documentacion score | > 80% | > 95% |

---

## Decision Final

**API Hibrida (REST + GraphQL BFF + WebSocket)** es la estrategia optima porque:

1. **Mobile-first**: GraphQL optimiza payloads para React Native
2. **B2B compatible**: REST standard para ATS e integraciones
3. **Real-time nativo**: WebSockets para simulaciones de entrevista
4. **Evolucion controlada**: Versionado explicito REST, deprecation suave GraphQL
5. **Developer experience**: SDKs auto-generados, documentacion interactiva
6. **Escalable**: Rate limiting granular, scopes por API key

---

## Referencias

- [REST API Design - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
- [JSON:API Specification](https://jsonapi.org/)
- [OpenAPI 3.1 Specification](https://spec.openapis.org/oas/v3.1.0)
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
- [Apollo Server Documentation](https://www.apollographql.com/docs/apollo-server/)
- [Socket.io Documentation](https://socket.io/docs/v4/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
