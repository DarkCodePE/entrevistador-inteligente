# ADR-013: Arquitectura de Privacidad y Compliance (LPDP Peru)

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura, Legal
Revisores: CTO, DPO designado, Asesoria Legal

---

## Contexto

### Marco Regulatorio Aplicable

Entrevistador Inteligente Peru maneja datos personales sensibles sujetos a multiples regulaciones:

| Regulacion | Ambito | Fecha Limite | Multas |
|------------|--------|--------------|--------|
| **LPDP (Ley 29733)** | Peru - Obligatorio | Vigente | 0.5 - 100 UIT (USD 750 - 150,500) |
| **Reglamento LPDP (DS 003-2013-JUS)** | Peru - Obligatorio | Vigente | Incluido en LPDP |
| **DPO Obligatorio (2025)** | Peru - Empresas grandes | Nov 2025 | Hasta 100 UIT |
| **GDPR** | UE - Para expansion | Futuro | Hasta 20M EUR o 4% facturacion |

### Datos Sensibles Manejados

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CLASIFICACION DE DATOS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────┐   ┌─────────────────────┐                       │
│  │   DATOS PERSONALES  │   │   DATOS SENSIBLES   │                       │
│  │   (Art. 2 LPDP)     │   │   (Art. 2.5 LPDP)   │                       │
│  ├─────────────────────┤   ├─────────────────────┤                       │
│  │ - Nombre completo   │   │ - Historial laboral │                       │
│  │ - Email             │   │ - Datos educativos  │                       │
│  │ - Telefono          │   │ - Salario esperado  │                       │
│  │ - Direccion         │   │ - Evaluaciones IA   │                       │
│  │ - DNI               │   │ - Grabaciones       │                       │
│  └─────────────────────┘   │ - Biometricos (fut) │                       │
│                            └─────────────────────┘                       │
│                                                                          │
│  ┌─────────────────────┐   ┌─────────────────────┐                       │
│  │   DATOS FINANCIEROS │   │   DATOS CONDUCTUALES│                       │
│  ├─────────────────────┤   ├─────────────────────┤                       │
│  │ - Info de pago      │   │ - Comportamiento    │                       │
│  │ - Historial compras │   │ - Preferencias      │                       │
│  │ - Facturacion       │   │ - Patrones de uso   │                       │
│  └─────────────────────┘   │ - Logs de sesion    │                       │
│                            └─────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Derechos ARCO (Art. 18-22 LPDP)

Los titulares de datos tienen derecho a:
- **A**cceso: Conocer que datos tenemos
- **R**ectificacion: Corregir datos inexactos
- **C**ancelacion: Eliminar datos (derecho al olvido)
- **O**posicion: Oponerse al tratamiento

### Problema a Resolver

1. Cumplir con LPDP y reglamento desde el MVP
2. Preparar arquitectura para GDPR (expansion EU)
3. Designar DPO antes de Nov 2025
4. Implementar mecanismos tecnicos para derechos ARCO
5. Proteger datos sensibles de candidatos y empresas
6. Establecer politicas de retencion y eliminacion

---

## Decision

**Adoptamos una arquitectura Privacy-by-Design con las siguientes decisiones clave:**

### 1. Data Residency

**Decision: Almacenamiento primario en Peru con opcion multi-region**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DATA RESIDENCY STRATEGY                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  FASE 1 (MVP):                    FASE 2 (LATAM):                        │
│  ┌─────────────────────┐          ┌─────────────────────┐                │
│  │   AWS sa-east-1     │          │   AWS sa-east-1     │ (Brasil)       │
│  │   (Sao Paulo)       │          │   AWS sa-east-1     │ (Peru backup)  │
│  │   ───────────────   │          │   AWS us-east-1     │ (Mexico)       │
│  │   Region primaria   │          └─────────────────────┘                │
│  │   Datos Peru        │                                                 │
│  └─────────────────────┘          FASE 3 (GDPR):                         │
│                                   ┌─────────────────────┐                │
│  Latencia Peru: ~30-50ms          │   AWS eu-west-1     │ (Irlanda)      │
│  Compliance: LPDP OK              │   Datos EU aislados │                │
│  Costo: Optimizado                └─────────────────────┘                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Justificacion:**
- AWS sa-east-1 (Sao Paulo) es la region mas cercana con servicios completos
- LPDP no exige almacenamiento en Peru, pero si proteccion adecuada
- Latencia aceptable (~30-50ms) para usuarios peruanos
- Preparado para expansion LATAM sin migracion

### 2. Encryption Strategy

**Decision: Encryption at-rest y in-transit con KMS gestionado**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ENCRYPTION ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  IN-TRANSIT:                                                             │
│  ┌──────────┐  TLS 1.3  ┌──────────┐  TLS 1.3  ┌──────────┐             │
│  │  Client  │──────────►│   ALB    │──────────►│  Backend │             │
│  └──────────┘           └──────────┘           └──────────┘             │
│                                                      │                   │
│  AT-REST:                                           │                   │
│  ┌─────────────────────────────────────────────────┴─────┐              │
│  │                                                        │              │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │              │
│  │  │  PostgreSQL  │  │      S3      │  │   ElastiCache│ │              │
│  │  │  AES-256     │  │  SSE-KMS     │  │  TLS + Enc   │ │              │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │              │
│  │                           │                            │              │
│  │                    ┌──────┴──────┐                    │              │
│  │                    │   AWS KMS   │                    │              │
│  │                    │   CMK Peru  │                    │              │
│  │                    └─────────────┘                    │              │
│  └───────────────────────────────────────────────────────┘              │
│                                                                          │
│  APPLICATION-LEVEL ENCRYPTION (Datos Sensibles):                         │
│  ┌─────────────────────────────────────────────────────────┐            │
│  │  CVs, Grabaciones, Evaluaciones IA                      │            │
│  │  ───────────────────────────────────────────            │            │
│  │  Envelope Encryption:                                   │            │
│  │  - DEK (Data Encryption Key) por documento              │            │
│  │  - KEK (Key Encryption Key) en KMS                      │            │
│  │  - Rotacion automatica cada 90 dias                     │            │
│  └─────────────────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Implementacion tecnica:**

| Capa | Mecanismo | Algoritmo | Rotacion |
|------|-----------|-----------|----------|
| Transporte | TLS 1.3 | ECDHE + AES-256-GCM | Certificados cada 90 dias |
| Base de datos | AWS RDS Encryption | AES-256 | Automatica via KMS |
| Almacenamiento | S3 SSE-KMS | AES-256 | Cada 365 dias |
| Aplicacion | Envelope Encryption | AES-256-GCM | DEK cada 90 dias |
| Backups | Encrypted snapshots | AES-256 | Heredado de fuente |

### 3. Consent Management

**Decision: Sistema de consentimiento granular con audit trail**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CONSENT MANAGEMENT SYSTEM                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  TIPOS DE CONSENTIMIENTO:                                                │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  [x] Terminos de servicio (obligatorio)                         │    │
│  │  [x] Politica de privacidad (obligatorio)                       │    │
│  │  [ ] Procesamiento de CV con IA (opcional pero necesario)       │    │
│  │  [ ] Grabacion de entrevistas simuladas (opcional)              │    │
│  │  [ ] Compartir CV con empresas (opcional)                       │    │
│  │  [ ] Comunicaciones de marketing (opcional)                     │    │
│  │  [ ] Analytics y mejora del servicio (opcional)                 │    │
│  │  [ ] Transferencia internacional de datos (si aplica)           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  MODELO DE DATOS:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  consent_records                                                 │    │
│  │  ─────────────────                                               │    │
│  │  id: UUID                                                        │    │
│  │  user_id: UUID (FK)                                              │    │
│  │  consent_type: ENUM                                              │    │
│  │  granted: BOOLEAN                                                │    │
│  │  granted_at: TIMESTAMP                                           │    │
│  │  revoked_at: TIMESTAMP (nullable)                                │    │
│  │  ip_address: VARCHAR (hasheado)                                  │    │
│  │  user_agent: VARCHAR                                             │    │
│  │  version: VARCHAR (version de politica)                          │    │
│  │  proof_hash: VARCHAR (SHA-256 del consentimiento)                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Requisitos LPDP implementados:**
- Consentimiento libre, expreso, inequivoco e informado (Art. 13.5)
- Registro de consentimientos con timestamp y prueba
- Facilidad para revocar consentimiento
- Re-consentimiento al cambiar politicas

### 4. Data Retention Policies

**Decision: Retencion basada en proposito con eliminacion automatizada**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DATA RETENTION MATRIX                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  TIPO DE DATO              │ RETENCION    │ BASE LEGAL     │ ACCION     │
│  ──────────────────────────┼──────────────┼────────────────┼─────────── │
│  Cuenta usuario activa     │ Indefinida   │ Contrato       │ Mantener   │
│  Cuenta inactiva           │ 2 anos       │ LPDP Art. 9    │ Anonimizar │
│  CV y datos laborales      │ 3 anos       │ Plazo laboral  │ Eliminar   │
│  Grabaciones entrevistas   │ 1 ano        │ Consentimiento │ Eliminar   │
│  Evaluaciones IA           │ 2 anos       │ Servicio       │ Anonimizar │
│  Logs de acceso            │ 1 ano        │ Seguridad      │ Eliminar   │
│  Datos de pago             │ 10 anos      │ SUNAT/Tributario│ Archivar  │
│  Consentimientos           │ 10 anos      │ Prueba legal   │ Archivar   │
│  Analytics agregados       │ 5 anos       │ Negocio        │ Mantener   │
│  Backups                   │ 90 dias      │ Operacional    │ Rotar      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Proceso automatizado:**

```
┌───────────────────────────────────────────────────────────────────────┐
│                    RETENTION AUTOMATION FLOW                           │
│                                                                        │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │
│   │  Scheduled  │───►│  Identify   │───►│  Classify   │               │
│   │  Job (Daily)│    │  Expired    │    │  Action     │               │
│   └─────────────┘    └─────────────┘    └─────────────┘               │
│                                                │                       │
│                      ┌────────────────────────┬┴───────────────────┐   │
│                      ▼                        ▼                    ▼   │
│              ┌─────────────┐          ┌─────────────┐      ┌─────────┐│
│              │  Anonimizar │          │  Eliminar   │      │ Archivar││
│              │  (GDPR ok)  │          │  (Secure)   │      │ (Cold)  ││
│              └─────────────┘          └─────────────┘      └─────────┘│
│                      │                        │                    │   │
│                      └────────────────────────┴────────────────────┘   │
│                                               │                        │
│                                        ┌──────┴──────┐                 │
│                                        │  Audit Log  │                 │
│                                        │  (Inmutable)│                 │
│                                        └─────────────┘                 │
└───────────────────────────────────────────────────────────────────────┘
```

### 5. Derechos ARCO - Implementacion Tecnica

**Decision: API dedicada para ejercicio de derechos con SLA de 10 dias**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ARCO RIGHTS IMPLEMENTATION                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ENDPOINT: POST /api/v1/privacy/arco-request                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  {                                                               │    │
│  │    "request_type": "ACCESS|RECTIFY|DELETE|PORTABILITY|OPPOSE",  │    │
│  │    "scope": ["cv", "interviews", "evaluations", "all"],         │    │
│  │    "reason": "string (requerido para oposicion)",               │    │
│  │    "verification": {                                             │    │
│  │      "method": "email|sms|document",                            │    │
│  │      "document_type": "DNI|CE|Pasaporte",                       │    │
│  │      "document_number": "string"                                 │    │
│  │    }                                                             │    │
│  │  }                                                               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  FLUJO DE PROCESAMIENTO:                                                 │
│                                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │ Solicitud│──►│Verificar │──►│ Aprobar/ │──►│ Ejecutar │             │
│  │  Usuario │   │Identidad │   │ Rechazar │   │  Accion  │             │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘             │
│       │              │              │               │                   │
│       │         Max 48h        Max 72h         Max 120h                 │
│       │              │              │               │                   │
│       └──────────────┴──────────────┴───────────────┘                   │
│                            │                                            │
│                    Total: Max 10 dias habiles                           │
│                    (LPDP Art. 63 Reglamento)                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Implementacion por tipo de derecho:**

| Derecho | Accion Tecnica | SLA | Complejidad |
|---------|----------------|-----|-------------|
| **Acceso** | Export JSON/PDF de todos los datos | 5 dias | Media |
| **Rectificacion** | Update con validacion | 3 dias | Baja |
| **Cancelacion** | Soft delete + scheduled hard delete | 10 dias | Alta |
| **Oposicion** | Flag en perfil + stop processing | 3 dias | Baja |
| **Portabilidad** | Export en formato estructurado (JSON) | 10 dias | Media |

### 6. Data Portability

**Decision: Formato JSON estructurado compatible con estandares**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DATA PORTABILITY FORMAT                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  {                                                                       │
│    "export_metadata": {                                                  │
│      "format_version": "1.0",                                           │
│      "generated_at": "2026-01-31T10:00:00Z",                            │
│      "platform": "Entrevistador Inteligente",                           │
│      "user_id_hash": "sha256:...",                                      │
│      "request_id": "uuid"                                                │
│    },                                                                    │
│    "personal_data": {                                                    │
│      "profile": { ... },                                                 │
│      "contact": { ... }                                                  │
│    },                                                                    │
│    "professional_data": {                                                │
│      "cvs": [ { ... } ],                                                 │
│      "work_history": [ { ... } ],                                        │
│      "education": [ { ... } ],                                           │
│      "skills": [ { ... } ]                                               │
│    },                                                                    │
│    "platform_data": {                                                    │
│      "interviews": [ { ... } ],                                          │
│      "evaluations": [ { ... } ],                                         │
│      "applications": [ { ... } ]                                         │
│    },                                                                    │
│    "consents": [ { ... } ],                                              │
│    "activity_log": [ { ... } ]                                           │
│  }                                                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7. Breach Notification Process

**Decision: Proceso automatizado con notificacion en 72 horas**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      BREACH NOTIFICATION PROCESS                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  DETECCION (T+0)                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Alertas de seguridad (WAF, IDS, SIEM)                        │    │
│  │  - Anomalias de acceso                                           │    │
│  │  - Reportes de usuarios                                          │    │
│  │  - Auditorias internas                                           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  EVALUACION (T+0 a T+24h)                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Clasificar severidad (Alta/Media/Baja)                       │    │
│  │  - Identificar datos afectados                                   │    │
│  │  - Estimar usuarios impactados                                   │    │
│  │  - Determinar causa raiz                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│         ┌────────────────────┼────────────────────┐                     │
│         ▼                    ▼                    ▼                      │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│  │ Severidad   │     │ Severidad   │     │ Severidad   │                │
│  │    ALTA     │     │   MEDIA     │     │    BAJA     │                │
│  │             │     │             │     │             │                │
│  │ Notificar:  │     │ Notificar:  │     │ Documentar  │                │
│  │ - ANPD 24h  │     │ - ANPD 72h  │     │ internamente│                │
│  │ - Usuarios  │     │ - Usuarios  │     │             │                │
│  │ - Prensa    │     │   afectados │     │             │                │
│  └─────────────┘     └─────────────┘     └─────────────┘                │
│                              │                                           │
│                              ▼                                           │
│  NOTIFICACION A ANPD (Art. 28.5 Reglamento LPDP)                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Contenido del reporte:                                          │    │
│  │  - Naturaleza de la brecha                                       │    │
│  │  - Categorias de datos afectados                                 │    │
│  │  - Numero aproximado de afectados                                │    │
│  │  - Medidas adoptadas                                             │    │
│  │  - Datos de contacto del DPO                                     │    │
│  │  - Consecuencias probables                                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8. Pseudonymization vs Anonymization

**Decision: Estrategia hibrida segun caso de uso**

```
┌─────────────────────────────────────────────────────────────────────────┐
│               PSEUDONYMIZATION vs ANONYMIZATION STRATEGY                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PSEUDONYMIZATION (Reversible - Datos siguen siendo personales)         │
│  ───────────────────────────────────────────────────────────────        │
│  Usar para:                                                              │
│  - Datos operacionales que necesitan re-identificacion                  │
│  - CVs en proceso de matching                                           │
│  - Entrevistas activas                                                  │
│                                                                          │
│  Tecnica: Tokenizacion con vault seguro                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  "Juan Perez" ──► "USR_a1b2c3d4" (token en vault separado)      │    │
│  │  "email@x.com" ──► "EMAIL_e5f6g7h8" (lookup table cifrada)      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ANONYMIZATION (Irreversible - Ya no son datos personales)              │
│  ───────────────────────────────────────────────────────────────        │
│  Usar para:                                                              │
│  - Analytics agregados                                                   │
│  - Training de modelos ML                                                │
│  - Reportes estadisticos                                                 │
│  - Datos post-retencion                                                  │
│                                                                          │
│  Tecnicas:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. Generalizacion: Edad 25 ──► Rango "20-30"                   │    │
│  │  2. Supresion: Eliminar campos identificadores                   │    │
│  │  3. Perturbacion: Agregar ruido a salarios                       │    │
│  │  4. K-anonimato: Asegurar k>=5 registros identicos              │    │
│  │  5. Differential Privacy: Para queries estadisticos              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9. Encryption Keys Management

**Decision: AWS KMS con separacion de ambientes y rotacion automatica**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      KEY MANAGEMENT ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                         AWS KMS                                  │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │                    Master Key (CMK)                        │  │    │
│  │  │         Region: sa-east-1 | Multi-region: future          │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  │                              │                                   │    │
│  │         ┌────────────────────┼────────────────────┐             │    │
│  │         ▼                    ▼                    ▼              │    │
│  │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐        │    │
│  │  │ CMK-PROD    │     │ CMK-STAGING │     │ CMK-DEV     │        │    │
│  │  │             │     │             │     │             │        │    │
│  │  │ Rotation:   │     │ Rotation:   │     │ Rotation:   │        │    │
│  │  │ 90 dias     │     │ 180 dias    │     │ Manual      │        │    │
│  │  └─────────────┘     └─────────────┘     └─────────────┘        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    DATA ENCRYPTION KEYS (DEK)                    │    │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│    │
│  │  │  DEK-RDS    │ │  DEK-S3     │ │ DEK-CVs     │ │DEK-Recordings│    │
│  │  │  (DB)       │ │  (Files)    │ │ (Sensible)  │ │  (Media)    ││    │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘│    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  KEY POLICIES:                                                           │
│  - Separacion por ambiente (prod/staging/dev)                           │
│  - Principio de minimo privilegio                                        │
│  - Auditoria de uso via CloudTrail                                       │
│  - Backup de claves en region secundaria                                 │
│  - Procedimiento de recovery documentado                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10. Audit Logging

**Decision: Logging inmutable con retencion de 2 anos**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AUDIT LOGGING ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  EVENTOS AUDITADOS:                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Acceso a datos personales (lectura/escritura)                │    │
│  │  - Cambios de consentimiento                                     │    │
│  │  - Solicitudes ARCO                                              │    │
│  │  - Accesos administrativos                                       │    │
│  │  - Exportacion de datos                                          │    │
│  │  - Intentos de acceso fallidos                                   │    │
│  │  - Cambios de configuracion de seguridad                         │    │
│  │  - Eliminacion de datos                                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  FORMATO DE LOG:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  {                                                               │    │
│  │    "timestamp": "2026-01-31T10:00:00.000Z",                      │    │
│  │    "event_id": "uuid",                                           │    │
│  │    "event_type": "DATA_ACCESS|CONSENT_CHANGE|ARCO_REQUEST|...", │    │
│  │    "actor": {                                                    │    │
│  │      "user_id": "uuid (o hasheado)",                             │    │
│  │      "role": "user|admin|system",                                │    │
│  │      "ip_address": "hasheado",                                   │    │
│  │      "session_id": "uuid"                                        │    │
│  │    },                                                            │    │
│  │    "resource": {                                                 │    │
│  │      "type": "cv|profile|interview|...",                         │    │
│  │      "id": "uuid",                                               │    │
│  │      "owner_id": "uuid (hasheado)"                               │    │
│  │    },                                                            │    │
│  │    "action": "read|write|delete|export",                         │    │
│  │    "result": "success|failure",                                  │    │
│  │    "metadata": { ... }                                           │    │
│  │  }                                                               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ARQUITECTURA:                                                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │   App    │───►│ Kinesis  │───►│  Lambda  │───►│    S3    │          │
│  │  Logs    │    │ Firehose │    │ Transform│    │ Glacier  │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│                                                        │                │
│                                                  WORM (Write Once       │
│                                                  Read Many) enabled     │
│                                                  Retencion: 2 anos      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11. Access Controls

**Decision: RBAC + ABAC con segregacion de datos por tenant**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ACCESS CONTROL MODEL                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  RBAC (Role-Based Access Control):                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  CANDIDATO          EMPRESA           ADMIN           DPO       │    │
│  │  ──────────         ───────           ─────           ───       │    │
│  │  - Ver su perfil    - Ver candidatos  - Todo          - Auditar │    │
│  │  - Editar CV          que aplicaron   - Soporte       - ARCO    │    │
│  │  - Aplicar jobs     - Publicar jobs   - Config        - Reportes│    │
│  │  - Ver historial    - Ver analytics   - Usuarios      - Brechas │    │
│  │  - ARCO propio      - Mensajes        - Logs                    │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ABAC (Attribute-Based Access Control):                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Politica ejemplo:                                               │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │  ALLOW action:read ON resource:cv                          │  │    │
│  │  │  WHERE                                                     │  │    │
│  │  │    subject.role = "recruiter" AND                          │  │    │
│  │  │    subject.company_id = resource.shared_with AND           │  │    │
│  │  │    resource.consent.share_with_companies = true AND        │  │    │
│  │  │    subject.subscription.status = "active"                  │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  TENANT ISOLATION:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Empresa A         Empresa B         Candidatos                 │    │
│  │  ┌──────────┐     ┌──────────┐      ┌──────────┐                │    │
│  │  │ Schema A │     │ Schema B │      │Schema PUB│                │    │
│  │  │          │     │          │      │          │                │    │
│  │  │ RLS      │     │ RLS      │      │ RLS      │                │    │
│  │  │ Enabled  │     │ Enabled  │      │ Enabled  │                │    │
│  │  └──────────┘     └──────────┘      └──────────┘                │    │
│  │                                                                  │    │
│  │  Row Level Security (RLS) en PostgreSQL:                        │    │
│  │  - Cada query filtrado por tenant_id automaticamente            │    │
│  │  - Imposible acceder a datos de otro tenant                     │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12. Data Masking

**Decision: Masking dinamico para ambientes no productivos**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DATA MASKING STRATEGY                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  TIPOS DE MASKING:                                                       │
│                                                                          │
│  1. MASKING ESTATICO (Para copias a staging/dev)                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Campo           │ Original           │ Masked                  │    │
│  │  ─────────────────┼────────────────────┼───────────────────────  │    │
│  │  nombre          │ "Juan Perez"       │ "Usuario_12345"         │    │
│  │  email           │ "juan@email.com"   │ "user_12345@masked.local"│   │
│  │  telefono        │ "+51987654321"     │ "+51900000000"          │    │
│  │  dni             │ "12345678"         │ "00000000"              │    │
│  │  direccion       │ "Av. Lima 123"     │ "Direccion Masked"      │    │
│  │  cv_texto        │ [contenido real]   │ [lorem ipsum generado]  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  2. MASKING DINAMICO (Para visualizacion segun rol)                     │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Vista ADMIN:      "Juan Perez | +51987654321 | 12345678"       │    │
│  │  Vista RECRUITER:  "Juan P*** | +51987****** | ********"        │    │
│  │  Vista ANALYTICS:  "Usuario | ****** | ********"                │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  IMPLEMENTACION:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  // Middleware de masking                                        │    │
│  │  @MaskingPolicy({                                                │    │
│  │    field: 'phone',                                               │    │
│  │    strategy: 'partial',                                          │    │
│  │    visibleChars: 4,                                              │    │
│  │    roles: ['admin', 'owner']  // Roles que ven sin masking      │    │
│  │  })                                                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Compliance LPDP** | Cumplimiento total con Ley 29733 y reglamento |
| **Preparacion GDPR** | Arquitectura lista para expansion EU |
| **Confianza del usuario** | Transparencia en manejo de datos |
| **Reduccion de riesgos** | Menor exposicion a multas y demandas |
| **Ventaja competitiva** | Diferenciador vs competencia menos rigurosa |
| **Auditabilidad** | Evidencia completa para autoridades |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Complejidad adicional** | Automatizacion de procesos ARCO |
| **Costo de infraestructura** | KMS y logging tienen costo, pero menor que multas |
| **Latencia en queries** | RLS y encryption optimizados, caching |
| **Overhead de desarrollo** | Templates y librerias reutilizables |
| **Friccion en UX** | Consentimientos claros y minimos |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Brecha de datos | Baja | Critico | Encryption + monitoring + incident response |
| Multa por incumplimiento | Baja | Alto | Auditoria periodica + DPO dedicado |
| Solicitud ARCO no atendida | Baja | Medio | Sistema automatizado + SLA tracking |
| Perdida de claves | Muy Baja | Critico | Backup multi-region + recovery drills |
| Cambio regulatorio | Media | Medio | Arquitectura flexible + monitoreo legal |

---

## DPO Requirements

### Timeline de Cumplimiento

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DPO IMPLEMENTATION TIMELINE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  2025                                                                    │
│  ────                                                                    │
│  Q1: Evaluar si calificamos como "empresa grande"                       │
│  Q2: Iniciar busqueda/capacitacion de DPO                               │
│  Q3: DPO designado y registrado ante ANPD                               │
│  Q4: DPO operativo antes de Nov 2025                                    │
│                                                                          │
│  CRITERIOS "EMPRESA GRANDE" (Aun pendiente definicion oficial):         │
│  - Ingresos anuales > X UIT                                             │
│  - Cantidad de datos procesados                                         │
│  - Tipo de datos (sensibles = mas probable requerir DPO)                │
│                                                                          │
│  FUNCIONES DEL DPO:                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Supervisar cumplimiento LPDP                                  │    │
│  │  - Punto de contacto con ANPD                                    │    │
│  │  - Atender solicitudes ARCO complejas                           │    │
│  │  - Realizar evaluaciones de impacto (EIPD)                       │    │
│  │  - Capacitar al personal                                         │    │
│  │  - Gestionar incidentes de seguridad                            │    │
│  │  - Auditorias internas periodicas                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  PERFIL DPO:                                                             │
│  - Conocimiento profundo de LPDP y reglamento                           │
│  - Experiencia en seguridad de la informacion                           │
│  - Independencia en su funcion                                          │
│  - Puede ser interno o externo (outsourced)                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Alternativas Consideradas

### Opcion A: Cumplimiento Minimo

**Descripcion:** Implementar solo lo estrictamente requerido por LPDP.

**Pros:**
- Menor costo inicial
- Implementacion mas rapida

**Contras:**
- No preparado para GDPR
- Mayor riesgo de brechas
- Refactoring costoso al escalar

**Veredicto:** Rechazado - Deuda de compliance inaceptable

### Opcion B: Solucion SaaS de Privacy (OneTrust, TrustArc)

**Descripcion:** Usar plataforma third-party para gestion de privacidad.

**Pros:**
- Implementacion rapida
- Expertise incluido
- Actualizaciones automaticas

**Contras:**
- Costo elevado ($15-50K/ano)
- Dependencia de vendor
- Puede no cubrir especificidades LPDP Peru

**Veredicto:** Considerar en Fase 2 cuando haya presupuesto

### Opcion C: Privacy-by-Design Completo (Seleccionada)

**Descripcion:** Arquitectura nativa con privacy integrado desde el inicio.

**Pros:**
- Control total
- Optimizado para LPDP + GDPR
- Costo predecible
- Sin vendor lock-in

**Contras:**
- Mayor esfuerzo inicial
- Requiere expertise interno

**Veredicto:** Seleccionada - Balance optimo para startup en crecimiento

---

## Implementacion Tecnica

### Estructura de Modulos Privacy

```
src/modules/privacy/
├── domain/
│   ├── entities/
│   │   ├── Consent.ts
│   │   ├── ArcoRequest.ts
│   │   ├── DataSubject.ts
│   │   └── AuditLog.ts
│   ├── value-objects/
│   │   ├── ConsentType.ts
│   │   ├── RetentionPolicy.ts
│   │   └── DataCategory.ts
│   └── events/
│       ├── ConsentGranted.ts
│       ├── ConsentRevoked.ts
│       ├── ArcoRequestCreated.ts
│       └── DataDeleted.ts
├── application/
│   ├── commands/
│   │   ├── GrantConsentCommand.ts
│   │   ├── RevokeConsentCommand.ts
│   │   ├── ProcessArcoRequestCommand.ts
│   │   └── DeleteUserDataCommand.ts
│   ├── queries/
│   │   ├── GetUserConsentsQuery.ts
│   │   ├── ExportUserDataQuery.ts
│   │   └── GetAuditLogsQuery.ts
│   └── services/
│       ├── ConsentService.ts
│       ├── ArcoService.ts
│       ├── DataRetentionService.ts
│       ├── AnonymizationService.ts
│       └── BreachNotificationService.ts
├── infrastructure/
│   ├── encryption/
│   │   ├── KmsClient.ts
│   │   ├── EnvelopeEncryption.ts
│   │   └── FieldLevelEncryption.ts
│   ├── masking/
│   │   ├── DynamicMasking.ts
│   │   └── StaticMasking.ts
│   ├── audit/
│   │   ├── AuditLogger.ts
│   │   └── ImmutableAuditStore.ts
│   └── repositories/
│       ├── ConsentRepository.ts
│       └── ArcoRequestRepository.ts
└── api/
    ├── controllers/
    │   ├── ConsentController.ts
    │   ├── ArcoController.ts
    │   └── PrivacyDashboardController.ts
    └── middleware/
        ├── ConsentVerificationMiddleware.ts
        ├── AuditMiddleware.ts
        └── MaskingMiddleware.ts
```

### Esquema de Base de Datos Privacy

```sql
-- Schema: privacy
CREATE SCHEMA IF NOT EXISTS privacy;

-- Tabla de consentimientos
CREATE TABLE privacy.consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES identity.users(id),
    consent_type VARCHAR(50) NOT NULL,
    granted BOOLEAN NOT NULL,
    granted_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    ip_address_hash VARCHAR(64),
    user_agent TEXT,
    policy_version VARCHAR(20) NOT NULL,
    proof_hash VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tabla de solicitudes ARCO
CREATE TABLE privacy.arco_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES identity.users(id),
    request_type VARCHAR(20) NOT NULL, -- ACCESS, RECTIFY, DELETE, PORTABILITY, OPPOSE
    scope JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    reason TEXT,
    verification_method VARCHAR(20),
    verified_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    result JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    sla_deadline TIMESTAMPTZ NOT NULL
);

-- Tabla de politicas de retencion
CREATE TABLE privacy.retention_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_category VARCHAR(50) NOT NULL UNIQUE,
    retention_days INT NOT NULL,
    action VARCHAR(20) NOT NULL, -- DELETE, ANONYMIZE, ARCHIVE
    legal_basis TEXT NOT NULL,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tabla de audit logs (append-only)
CREATE TABLE privacy.audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(50) NOT NULL,
    actor_id UUID,
    actor_role VARCHAR(20),
    actor_ip_hash VARCHAR(64),
    resource_type VARCHAR(50),
    resource_id UUID,
    resource_owner_hash VARCHAR(64),
    action VARCHAR(20) NOT NULL,
    result VARCHAR(20) NOT NULL,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indices
CREATE INDEX idx_consents_user ON privacy.consents(user_id);
CREATE INDEX idx_consents_type ON privacy.consents(consent_type);
CREATE INDEX idx_arco_user ON privacy.arco_requests(user_id);
CREATE INDEX idx_arco_status ON privacy.arco_requests(status);
CREATE INDEX idx_audit_actor ON privacy.audit_logs(actor_id);
CREATE INDEX idx_audit_resource ON privacy.audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_created ON privacy.audit_logs(created_at);

-- RLS para audit_logs (solo lectura, nunca delete/update)
ALTER TABLE privacy.audit_logs ENABLE ROW LEVEL SECURITY;
CREATE POLICY audit_read_only ON privacy.audit_logs
    FOR SELECT USING (true);
-- No INSERT policy needed, handled by service account
```

---

## Checklist de Compliance

### LPDP (Ley 29733)

| Articulo | Requisito | Estado | Implementacion |
|----------|-----------|--------|----------------|
| Art. 5 | Principio de consentimiento | Pendiente | ConsentService |
| Art. 6 | Principio de finalidad | Pendiente | Politicas documentadas |
| Art. 7 | Principio de proporcionalidad | Pendiente | Minimizacion de datos |
| Art. 8 | Principio de calidad | Pendiente | Validacion + rectificacion |
| Art. 9 | Principio de seguridad | Pendiente | Encryption + access control |
| Art. 10 | Principio de disposicion de recurso | Pendiente | ArcoService |
| Art. 13 | Consentimiento | Pendiente | ConsentService |
| Art. 18-22 | Derechos ARCO | Pendiente | ArcoService + API |
| Art. 28 | Medidas de seguridad | Pendiente | Security architecture |
| Art. 32 | Transferencias internacionales | Pendiente | Clausulas contractuales |

### Reglamento LPDP (DS 003-2013-JUS)

| Articulo | Requisito | Estado | Implementacion |
|----------|-----------|--------|----------------|
| Art. 28.5 | Notificacion de incidentes | Pendiente | BreachNotificationService |
| Art. 63 | Plazo derechos ARCO (10 dias) | Pendiente | SLA tracking |
| Art. 39-44 | Medidas de seguridad | Pendiente | Security controls |

---

## Metricas de Compliance

| Metrica | Objetivo | Frecuencia |
|---------|----------|------------|
| Tiempo respuesta ARCO | < 10 dias habiles | Por solicitud |
| Tasa de consentimiento valido | > 99% | Mensual |
| Cobertura de encryption | 100% datos sensibles | Continuo |
| Audit logs completitud | 100% eventos criticos | Continuo |
| Tiempo deteccion brecha | < 24 horas | Por incidente |
| Tiempo notificacion ANPD | < 72 horas | Por incidente |
| Retencion compliance | 100% politicas aplicadas | Semanal |

---

## Referencias

- [Ley 29733 - LPDP Peru](https://www.gob.pe/institucion/congreso/normas-legales/118374-29733)
- [DS 003-2013-JUS - Reglamento LPDP](https://www.gob.pe/institucion/minjus/normas-legales/1070958-003-2013-jus)
- [ANPD Peru - Autoridad Nacional de Proteccion de Datos](https://www.gob.pe/anpd)
- [GDPR - Reglamento General de Proteccion de Datos](https://gdpr.eu/)
- [OWASP Privacy Risks](https://owasp.org/www-project-top-10-privacy-risks/)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
