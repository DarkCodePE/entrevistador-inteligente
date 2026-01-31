# ADR-014: Arquitectura Multi-Tenant para B2B

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead, Security Lead

---

## Contexto

### Descripcion del Problema

"Entrevistador Inteligente" expande su modelo de negocio para incluir clientes empresariales (B2B). Las empresas contratan licencias para que sus candidatos y empleados accedan a la plataforma, requiriendo:

- Aislamiento de datos entre empresas
- Administracion delegada por empresa
- Branding personalizado (white-label futuro)
- Integracion con sistemas corporativos (ATS, SSO)
- Facturacion y quotas por tenant

### Modelo de Negocio B2B

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TIERS DE LICENCIAMIENTO B2B                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │     STARTER     │  │     GROWTH      │  │   ENTERPRISE    │      │
│  │   S/499/mes     │  │   S/999/mes     │  │     Custom      │      │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤      │
│  │ - 20 usuarios   │  │ - 100 usuarios  │  │ - Ilimitados    │      │
│  │ - Reportes      │  │ - Custom Q's    │  │ - White-label   │      │
│  │   basicos       │  │ - Priority      │  │ - API access    │      │
│  │ - Email support │  │   support       │  │ - Dedicated CSM │      │
│  │                 │  │ - SSO           │  │ - SLA 99.9%     │      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Requisitos Criticos

| Requisito | Descripcion | Prioridad |
|-----------|-------------|-----------|
| **Data Isolation** | Datos de una empresa nunca visibles a otra | Critico |
| **Performance Isolation** | Un tenant no afecta performance de otros | Alto |
| **Customization** | Branding, preguntas, flujos personalizados | Medio |
| **Compliance** | Auditoria, GDPR, ley de proteccion de datos Peru | Critico |
| **Scalability** | Soportar 100+ tenants con 1000+ users c/u | Alto |
| **Cost Efficiency** | Mantener costos operativos razonables | Alto |

---

## Decision

### 1. Estrategia de Aislamiento: Row-Level Security (RLS) con Schema Compartido

**Adoptamos Row-Level Security (RLS) en PostgreSQL con un schema compartido como estrategia principal de aislamiento de datos.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MODELO DE DATOS MULTI-TENANT                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    PostgreSQL Database                        │    │
│  │                                                               │    │
│  │  ┌─────────────────────────────────────────────────────────┐ │    │
│  │  │                  Schema: public                          │ │    │
│  │  │                                                          │ │    │
│  │  │  ┌─────────────────┐  ┌─────────────────┐               │ │    │
│  │  │  │    tenants      │  │     users       │               │ │    │
│  │  │  │ ────────────────│  │ ────────────────│               │ │    │
│  │  │  │ id (PK)         │  │ id (PK)         │               │ │    │
│  │  │  │ name            │◄─│ tenant_id (FK)  │               │ │    │
│  │  │  │ slug            │  │ email           │               │ │    │
│  │  │  │ tier            │  │ role            │               │ │    │
│  │  │  │ settings (JSONB)│  │ ...             │               │ │    │
│  │  │  └─────────────────┘  └─────────────────┘               │ │    │
│  │  │                                                          │ │    │
│  │  │  ┌─────────────────┐  ┌─────────────────┐               │ │    │
│  │  │  │   interviews    │  │   questions     │               │ │    │
│  │  │  │ ────────────────│  │ ────────────────│               │ │    │
│  │  │  │ id (PK)         │  │ id (PK)         │               │ │    │
│  │  │  │ tenant_id (FK)  │  │ tenant_id (FK)  │               │ │    │
│  │  │  │ user_id (FK)    │  │ category        │               │ │    │
│  │  │  │ ...             │  │ content         │               │ │    │
│  │  │  └─────────────────┘  └─────────────────┘               │ │    │
│  │  │                                                          │ │    │
│  │  │         RLS Policies enforce tenant_id filtering         │ │    │
│  │  └─────────────────────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Implementacion RLS en PostgreSQL

```sql
-- Funcion para obtener tenant_id del contexto de sesion
CREATE OR REPLACE FUNCTION current_tenant_id()
RETURNS UUID AS $$
BEGIN
  RETURN current_setting('app.current_tenant_id', true)::UUID;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Tabla base con tenant_id
CREATE TABLE interviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID NOT NULL REFERENCES users(id),
    score INTEGER,
    feedback JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    -- Indice compuesto para queries eficientes
    CONSTRAINT fk_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

-- Indice para filtrado por tenant
CREATE INDEX idx_interviews_tenant ON interviews(tenant_id);

-- Habilitar RLS
ALTER TABLE interviews ENABLE ROW LEVEL SECURITY;

-- Politica de aislamiento
CREATE POLICY tenant_isolation ON interviews
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- Politica para superadmins (cross-tenant analytics)
CREATE POLICY superadmin_access ON interviews
    FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM users
            WHERE id = current_setting('app.current_user_id', true)::UUID
            AND role = 'superadmin'
        )
    );
```

### 2. Identificacion de Tenant: Subdominio + Header Fallback

**Estrategia hibrida**: Subdominio como metodo primario, header como fallback para APIs.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FLUJO DE IDENTIFICACION DE TENANT                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Request entrante                                                    │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Tenant Resolution Middleware                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ├──► 1. Verificar subdominio                                  │
│       │    acme.entrevistador.ai → tenant: acme                     │
│       │                                                              │
│       ├──► 2. Si no hay subdominio, verificar header               │
│       │    X-Tenant-ID: acme-corp-uuid                              │
│       │                                                              │
│       ├──► 3. Si API key, extraer tenant del token                 │
│       │    Bearer: api_acme_xxx → tenant: acme                      │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Set PostgreSQL session variable                  │    │
│  │         SET app.current_tenant_id = 'uuid-here'               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Todas las queries automaticamente filtradas por RLS                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Middleware de Resolucion

```typescript
// src/modules/shared/infrastructure/middleware/tenant-resolver.ts
import { Injectable, NestMiddleware, UnauthorizedException } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { TenantService } from '../services/tenant.service';

@Injectable()
export class TenantResolverMiddleware implements NestMiddleware {
  constructor(private readonly tenantService: TenantService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    let tenantIdentifier: string | null = null;

    // 1. Subdominio
    const host = req.hostname;
    const subdomain = this.extractSubdomain(host);
    if (subdomain && subdomain !== 'www' && subdomain !== 'app') {
      tenantIdentifier = subdomain;
    }

    // 2. Header fallback
    if (!tenantIdentifier) {
      tenantIdentifier = req.headers['x-tenant-id'] as string;
    }

    // 3. API Key extraction
    if (!tenantIdentifier && req.headers.authorization) {
      tenantIdentifier = await this.extractTenantFromApiKey(
        req.headers.authorization
      );
    }

    if (!tenantIdentifier) {
      throw new UnauthorizedException('Tenant not identified');
    }

    // Resolver tenant y validar
    const tenant = await this.tenantService.resolveByIdentifier(tenantIdentifier);
    if (!tenant || !tenant.isActive) {
      throw new UnauthorizedException('Invalid or inactive tenant');
    }

    // Adjuntar al request para uso posterior
    req['tenant'] = tenant;
    req['tenantId'] = tenant.id;

    next();
  }

  private extractSubdomain(host: string): string | null {
    const parts = host.split('.');
    if (parts.length >= 3) {
      return parts[0];
    }
    return null;
  }

  private async extractTenantFromApiKey(authHeader: string): Promise<string | null> {
    // Implementar extraccion de tenant desde API key
    return null;
  }
}
```

### 3. Quotas y Limites por Tenant

**Sistema de quotas basado en tiers con enforcement en tiempo real.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SISTEMA DE QUOTAS POR TIER                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Quota Configuration                        │    │
│  │                                                               │    │
│  │  STARTER:                                                     │    │
│  │    max_users: 20                                              │    │
│  │    max_interviews_per_month: 100                              │    │
│  │    max_custom_questions: 0                                    │    │
│  │    api_access: false                                          │    │
│  │    sso_enabled: false                                         │    │
│  │    storage_gb: 5                                              │    │
│  │                                                               │    │
│  │  GROWTH:                                                      │    │
│  │    max_users: 100                                             │    │
│  │    max_interviews_per_month: 500                              │    │
│  │    max_custom_questions: 100                                  │    │
│  │    api_access: false                                          │    │
│  │    sso_enabled: true                                          │    │
│  │    storage_gb: 25                                             │    │
│  │                                                               │    │
│  │  ENTERPRISE:                                                  │    │
│  │    max_users: unlimited                                       │    │
│  │    max_interviews_per_month: unlimited                        │    │
│  │    max_custom_questions: unlimited                            │    │
│  │    api_access: true                                           │    │
│  │    sso_enabled: true                                          │    │
│  │    storage_gb: custom                                         │    │
│  │    white_label: true                                          │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Quota Enforcement Flow                     │    │
│  │                                                               │    │
│  │   Request ──► Check Quota ──► Allow/Deny ──► Update Usage    │    │
│  │                    │                              │           │    │
│  │                    ▼                              ▼           │    │
│  │              Redis Cache                    PostgreSQL        │    │
│  │           (real-time counts)              (authoritative)     │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Modelo de Datos para Quotas

```sql
-- Tabla de configuracion de quotas por tier
CREATE TABLE tier_quotas (
    tier VARCHAR(50) PRIMARY KEY,
    max_users INTEGER,
    max_interviews_per_month INTEGER,
    max_custom_questions INTEGER,
    api_access BOOLEAN DEFAULT FALSE,
    sso_enabled BOOLEAN DEFAULT FALSE,
    storage_gb INTEGER,
    white_label BOOLEAN DEFAULT FALSE,
    features JSONB DEFAULT '{}'
);

-- Tabla de uso actual por tenant
CREATE TABLE tenant_usage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    current_users INTEGER DEFAULT 0,
    interviews_count INTEGER DEFAULT 0,
    storage_used_mb INTEGER DEFAULT 0,
    api_calls INTEGER DEFAULT 0,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, period_start)
);

-- Funcion para verificar quota antes de accion
CREATE OR REPLACE FUNCTION check_quota(
    p_tenant_id UUID,
    p_resource VARCHAR(50),
    p_increment INTEGER DEFAULT 1
) RETURNS BOOLEAN AS $$
DECLARE
    v_tier VARCHAR(50);
    v_limit INTEGER;
    v_current INTEGER;
BEGIN
    -- Obtener tier del tenant
    SELECT tier INTO v_tier FROM tenants WHERE id = p_tenant_id;

    -- Obtener limite para el recurso
    EXECUTE format('SELECT %I FROM tier_quotas WHERE tier = $1', 'max_' || p_resource)
    INTO v_limit USING v_tier;

    -- Si es unlimited (-1), permitir
    IF v_limit = -1 THEN
        RETURN TRUE;
    END IF;

    -- Obtener uso actual
    EXECUTE format('SELECT %I FROM tenant_usage WHERE tenant_id = $1 AND period_start <= CURRENT_DATE AND period_end >= CURRENT_DATE', p_resource || '_count')
    INTO v_current USING p_tenant_id;

    RETURN COALESCE(v_current, 0) + p_increment <= v_limit;
END;
$$ LANGUAGE plpgsql;
```

### 4. Jerarquia de Roles B2B

**Modelo de roles con permisos granulares por tenant.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    JERARQUIA DE ROLES B2B                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                    ┌─────────────────┐                               │
│                    │   SUPERADMIN    │ (Plataforma)                  │
│                    │   - Cross-tenant│                               │
│                    │   - Analytics   │                               │
│                    │   - Config      │                               │
│                    └────────┬────────┘                               │
│                             │                                        │
│              ┌──────────────┼──────────────┐                        │
│              ▼              ▼              ▼                        │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐       │
│  │   TENANT ADMIN  │ │   TENANT ADMIN  │ │   TENANT ADMIN  │       │
│  │   (Empresa A)   │ │   (Empresa B)   │ │   (Empresa C)   │       │
│  └────────┬────────┘ └─────────────────┘ └─────────────────┘       │
│           │                                                          │
│           ├──────────────────────────────┐                          │
│           ▼                              ▼                          │
│  ┌─────────────────┐           ┌─────────────────┐                  │
│  │     MANAGER     │           │     MANAGER     │                  │
│  │  - Reportes     │           │  - Area RRHH    │                  │
│  │  - Invitar      │           │                 │                  │
│  │  - Config basic │           │                 │                  │
│  └────────┬────────┘           └────────┬────────┘                  │
│           │                              │                          │
│           ▼                              ▼                          │
│  ┌─────────────────┐           ┌─────────────────┐                  │
│  │      USER       │           │      USER       │                  │
│  │  - Entrevistas  │           │  - Entrevistas  │                  │
│  │  - Ver propio   │           │  - Ver propio   │                  │
│  │    progreso     │           │    progreso     │                  │
│  └─────────────────┘           └─────────────────┘                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Modelo de Permisos

```typescript
// src/modules/identity/domain/permissions.ts
export enum Permission {
  // Tenant Management
  TENANT_VIEW = 'tenant:view',
  TENANT_UPDATE = 'tenant:update',
  TENANT_BILLING = 'tenant:billing',

  // User Management
  USER_INVITE = 'user:invite',
  USER_REMOVE = 'user:remove',
  USER_VIEW_ALL = 'user:view_all',
  USER_UPDATE_ROLE = 'user:update_role',

  // Interviews
  INTERVIEW_CREATE = 'interview:create',
  INTERVIEW_VIEW_OWN = 'interview:view_own',
  INTERVIEW_VIEW_ALL = 'interview:view_all',
  INTERVIEW_EXPORT = 'interview:export',

  // Questions
  QUESTION_CREATE = 'question:create',
  QUESTION_UPDATE = 'question:update',
  QUESTION_DELETE = 'question:delete',

  // Reports
  REPORT_VIEW_BASIC = 'report:view_basic',
  REPORT_VIEW_ADVANCED = 'report:view_advanced',
  REPORT_EXPORT = 'report:export',

  // API
  API_ACCESS = 'api:access',
  API_MANAGE_KEYS = 'api:manage_keys',

  // SSO
  SSO_CONFIGURE = 'sso:configure',

  // Branding
  BRANDING_UPDATE = 'branding:update',
}

export const ROLE_PERMISSIONS: Record<string, Permission[]> = {
  superadmin: Object.values(Permission),

  tenant_admin: [
    Permission.TENANT_VIEW,
    Permission.TENANT_UPDATE,
    Permission.TENANT_BILLING,
    Permission.USER_INVITE,
    Permission.USER_REMOVE,
    Permission.USER_VIEW_ALL,
    Permission.USER_UPDATE_ROLE,
    Permission.INTERVIEW_VIEW_ALL,
    Permission.INTERVIEW_EXPORT,
    Permission.QUESTION_CREATE,
    Permission.QUESTION_UPDATE,
    Permission.QUESTION_DELETE,
    Permission.REPORT_VIEW_BASIC,
    Permission.REPORT_VIEW_ADVANCED,
    Permission.REPORT_EXPORT,
    Permission.API_ACCESS,
    Permission.API_MANAGE_KEYS,
    Permission.SSO_CONFIGURE,
    Permission.BRANDING_UPDATE,
  ],

  manager: [
    Permission.TENANT_VIEW,
    Permission.USER_INVITE,
    Permission.USER_VIEW_ALL,
    Permission.INTERVIEW_VIEW_ALL,
    Permission.QUESTION_CREATE,
    Permission.REPORT_VIEW_BASIC,
    Permission.REPORT_VIEW_ADVANCED,
  ],

  user: [
    Permission.INTERVIEW_CREATE,
    Permission.INTERVIEW_VIEW_OWN,
    Permission.REPORT_VIEW_BASIC,
  ],
};
```

### 5. Single Sign-On (SSO)

**Soporte para SAML 2.0 y OIDC para tiers Growth y Enterprise.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FLUJO SSO ENTERPRISE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                         SAML 2.0 Flow                                │
│  ┌──────────┐         ┌──────────────┐         ┌──────────────┐    │
│  │  Usuario │         │ Entrevistador│         │ IdP Empresa  │    │
│  │ Empresa  │         │  (SP)        │         │ (Okta/Azure) │    │
│  └────┬─────┘         └──────┬───────┘         └──────┬───────┘    │
│       │                      │                        │             │
│       │  1. Accede a         │                        │             │
│       │  acme.entrevistador.ai                        │             │
│       │─────────────────────►│                        │             │
│       │                      │                        │             │
│       │  2. Redirect a IdP   │                        │             │
│       │◄─────────────────────│                        │             │
│       │                      │                        │             │
│       │  3. Login en IdP     │                        │             │
│       │──────────────────────┼───────────────────────►│             │
│       │                      │                        │             │
│       │  4. SAML Assertion   │                        │             │
│       │◄─────────────────────┼────────────────────────│             │
│       │                      │                        │             │
│       │  5. POST Assertion   │                        │             │
│       │─────────────────────►│                        │             │
│       │                      │                        │             │
│       │  6. Validar + Sesion │                        │             │
│       │◄─────────────────────│                        │             │
│       │                      │                        │             │
│                                                                      │
│                         OIDC Flow                                    │
│  ┌──────────┐         ┌──────────────┐         ┌──────────────┐    │
│  │  Usuario │         │ Entrevistador│         │ IdP Empresa  │    │
│  │ Empresa  │         │  (Client)    │         │ (Google/MS)  │    │
│  └────┬─────┘         └──────┬───────┘         └──────┬───────┘    │
│       │                      │                        │             │
│       │  1. Login request    │                        │             │
│       │─────────────────────►│                        │             │
│       │                      │                        │             │
│       │  2. Redirect to IdP  │ 3. Auth request        │             │
│       │◄─────────────────────│───────────────────────►│             │
│       │                      │                        │             │
│       │  4. User authenticates                        │             │
│       │──────────────────────┼───────────────────────►│             │
│       │                      │                        │             │
│       │  5. Redirect + Code  │                        │             │
│       │◄─────────────────────┼────────────────────────│             │
│       │                      │                        │             │
│       │  6. Code to App      │ 7. Exchange code       │             │
│       │─────────────────────►│───────────────────────►│             │
│       │                      │                        │             │
│       │                      │ 8. ID Token + Access   │             │
│       │                      │◄───────────────────────│             │
│       │                      │                        │             │
│       │  9. Session created  │                        │             │
│       │◄─────────────────────│                        │             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Configuracion SSO por Tenant

```sql
-- Tabla de configuracion SSO
CREATE TABLE tenant_sso_config (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) UNIQUE,
    provider VARCHAR(20) NOT NULL, -- 'saml', 'oidc'
    enabled BOOLEAN DEFAULT FALSE,

    -- SAML config
    saml_entry_point VARCHAR(500),
    saml_issuer VARCHAR(500),
    saml_certificate TEXT,
    saml_callback_url VARCHAR(500),

    -- OIDC config
    oidc_issuer VARCHAR(500),
    oidc_client_id VARCHAR(255),
    oidc_client_secret_encrypted TEXT,
    oidc_authorization_url VARCHAR(500),
    oidc_token_url VARCHAR(500),
    oidc_userinfo_url VARCHAR(500),

    -- Mapeo de atributos
    attribute_mapping JSONB DEFAULT '{
        "email": "email",
        "firstName": "given_name",
        "lastName": "family_name",
        "groups": "groups"
    }',

    -- Auto-provisioning
    auto_provision_users BOOLEAN DEFAULT TRUE,
    default_role VARCHAR(50) DEFAULT 'user',

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 6. API Access para Integracion ATS

**API RESTful con autenticacion por API Key para tier Enterprise.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    API ACCESS PARA ATS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    API Key Management                         │    │
│  │                                                               │    │
│  │  Format: ei_live_[tenant_short]_[random_32_chars]            │    │
│  │  Example: ei_live_acme_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6     │    │
│  │                                                               │    │
│  │  Scopes:                                                      │    │
│  │    - interviews:read    - Leer entrevistas                   │    │
│  │    - interviews:write   - Crear/actualizar entrevistas       │    │
│  │    - users:read         - Leer usuarios                      │    │
│  │    - users:write        - Crear/invitar usuarios             │    │
│  │    - reports:read       - Acceder a reportes                 │    │
│  │    - webhooks:manage    - Configurar webhooks                │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Endpoints Principales                      │    │
│  │                                                               │    │
│  │  POST   /api/v1/interviews           Crear entrevista        │    │
│  │  GET    /api/v1/interviews/:id       Obtener entrevista      │    │
│  │  GET    /api/v1/interviews           Listar entrevistas      │    │
│  │  POST   /api/v1/users/invite         Invitar usuario         │    │
│  │  GET    /api/v1/users                Listar usuarios         │    │
│  │  GET    /api/v1/reports/summary      Resumen de metricas     │    │
│  │  POST   /api/v1/webhooks             Registrar webhook       │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Webhook Events                             │    │
│  │                                                               │    │
│  │  interview.started      - Entrevista iniciada                │    │
│  │  interview.completed    - Entrevista completada              │    │
│  │  interview.scored       - Score calculado                    │    │
│  │  user.invited           - Usuario invitado                   │    │
│  │  user.activated         - Usuario activo                     │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7. Cross-Tenant Analytics (Agregado)

**Analytics agregados para insights de plataforma sin exponer datos individuales.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CROSS-TENANT ANALYTICS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Datos Agregados (Sin PII)                        │    │
│  │                                                               │    │
│  │  - Promedio de score por industria                           │    │
│  │  - Distribucion de duracion de entrevistas                   │    │
│  │  - Preguntas mas dificiles (agregado)                        │    │
│  │  - Tendencias de uso por region                              │    │
│  │  - Benchmarks por rol/posicion                               │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Materialized Views para Performance             │    │
│  │                                                               │    │
│  │  CREATE MATERIALIZED VIEW aggregated_benchmarks AS           │    │
│  │  SELECT                                                       │    │
│  │      t.industry,                                              │    │
│  │      i.position_type,                                         │    │
│  │      DATE_TRUNC('month', i.created_at) as period,            │    │
│  │      COUNT(*) as interview_count,                             │    │
│  │      AVG(i.score) as avg_score,                               │    │
│  │      PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY i.score)    │    │
│  │        as median_score                                        │    │
│  │  FROM interviews i                                            │    │
│  │  JOIN tenants t ON i.tenant_id = t.id                        │    │
│  │  WHERE i.created_at > NOW() - INTERVAL '12 months'           │    │
│  │  GROUP BY t.industry, i.position_type,                       │    │
│  │           DATE_TRUNC('month', i.created_at)                   │    │
│  │  HAVING COUNT(*) >= 10; -- Minimo para anonimato             │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Aislamiento garantizado** | RLS a nivel de BD asegura que datos nunca se filtren |
| **Costo-eficiente** | Schema compartido reduce costos de infraestructura |
| **Escalabilidad** | Puede soportar miles de tenants sin cambios arquitectonicos |
| **Flexibilidad de identificacion** | Subdominio para UX, header para APIs, compatible con ambos |
| **SSO enterprise-grade** | SAML + OIDC cubren 95% de IdPs corporativos |
| **API-first** | Integracion ATS simple para clientes Enterprise |
| **Compliance-ready** | Auditoria y aislamiento facilitan cumplimiento regulatorio |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Complejidad de RLS** | Testing exhaustivo, politicas bien documentadas |
| **Performance overhead RLS** | Indices optimizados, caching agresivo |
| **Administracion de schemas SSL** | Automatizacion con scripts de provision |
| **Noisy neighbor potential** | Rate limiting por tenant, quotas estrictas |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Data leak entre tenants | Muy Baja | Critico | RLS + tests automatizados + auditorias |
| Performance degradation | Media | Alto | Monitoring por tenant, alertas tempranas |
| SSO misconfiguration | Media | Medio | Wizard de configuracion, validacion |
| Quota gaming | Baja | Bajo | Monitoreo de patrones anomalos |

---

## Alternativas Consideradas

### Opcion A: Database-per-Tenant

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│   DB    │  │   DB    │  │   DB    │
│ Tenant A│  │ Tenant B│  │ Tenant C│
└─────────┘  └─────────┘  └─────────┘
```

**Pros:**
- Maximo aislamiento
- Backup/restore por tenant simple
- Sin riesgo de noisy neighbor

**Contras:**
- Costo de infraestructura 5-10x mayor
- Complejidad operacional alta
- Migraciones de schema complejas
- No escala mas alla de 50-100 tenants

**Veredicto:** Rechazado - Costo prohibitivo para modelo SaaS con muchos tenants pequenos

### Opcion B: Schema-per-Tenant

```
┌────────────────────────────────────────┐
│              PostgreSQL                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐│
│  │ schema_a │ │ schema_b │ │ schema_c ││
│  └──────────┘ └──────────┘ └──────────┘│
└────────────────────────────────────────┘
```

**Pros:**
- Buen aislamiento logico
- Restore por schema posible
- Menor costo que DB-per-tenant

**Contras:**
- Limite practico de ~500 schemas
- Migraciones requieren script por schema
- Connection pooling complejo

**Veredicto:** Rechazado - Limitaciones de escalabilidad y complejidad operacional

### Opcion C: RLS (Seleccionada)

**Pros:**
- Escalabilidad ilimitada de tenants
- Migraciones simples (un schema)
- Costo optimo de infraestructura
- Connection pooling estandar

**Contras:**
- Requiere disciplina en queries
- Overhead minimo de RLS

**Veredicto:** Aceptado - Balance optimo entre aislamiento, costo y escalabilidad

---

## Modelo de Datos Completo

```sql
-- Tabla principal de tenants
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tier VARCHAR(50) NOT NULL DEFAULT 'starter',
    industry VARCHAR(100),
    country VARCHAR(2) DEFAULT 'PE',

    -- Estado
    is_active BOOLEAN DEFAULT TRUE,
    trial_ends_at TIMESTAMPTZ,

    -- Configuracion
    settings JSONB DEFAULT '{
        "timezone": "America/Lima",
        "language": "es",
        "interview_duration_minutes": 30
    }',

    -- Branding (white-label)
    branding JSONB DEFAULT '{
        "logo_url": null,
        "primary_color": "#3B82F6",
        "company_name": null
    }',

    -- Billing
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255),

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Usuarios con tenant_id
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255), -- null si usa SSO

    -- Perfil
    first_name VARCHAR(100),
    last_name VARCHAR(100),

    -- Rol dentro del tenant
    role VARCHAR(50) NOT NULL DEFAULT 'user',

    -- SSO
    sso_provider VARCHAR(50),
    sso_subject_id VARCHAR(255),

    -- Estado
    is_active BOOLEAN DEFAULT TRUE,
    email_verified_at TIMESTAMPTZ,
    last_login_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(tenant_id, email)
);

-- Indice para RLS performance
CREATE INDEX idx_users_tenant ON users(tenant_id);

-- API Keys
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    key_hash VARCHAR(255) NOT NULL, -- bcrypt hash of key
    key_prefix VARCHAR(20) NOT NULL, -- para identificacion (ei_live_acme_...)

    scopes TEXT[] NOT NULL DEFAULT '{}',

    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    is_active BOOLEAN DEFAULT TRUE,

    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Webhooks
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    url VARCHAR(500) NOT NULL,
    events TEXT[] NOT NULL,
    secret VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,

    last_triggered_at TIMESTAMPTZ,
    failure_count INTEGER DEFAULT 0,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Habilitar RLS en todas las tablas tenant-scoped
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhooks ENABLE ROW LEVEL SECURITY;

-- Politicas RLS
CREATE POLICY tenant_isolation_users ON users
    USING (tenant_id = current_tenant_id());

CREATE POLICY tenant_isolation_api_keys ON api_keys
    USING (tenant_id = current_tenant_id());

CREATE POLICY tenant_isolation_webhooks ON webhooks
    USING (tenant_id = current_tenant_id());
```

---

## Plan de Implementacion

### Fase 1: Fundacion (Semanas 1-3)
- Implementar modelo de datos multi-tenant
- Configurar RLS en PostgreSQL
- Crear middleware de resolucion de tenant
- Setup de subdominios wildcard

### Fase 2: Administracion (Semanas 4-5)
- Panel de administracion de tenant
- Gestion de usuarios y roles
- Sistema de quotas basico

### Fase 3: SSO (Semanas 6-7)
- Integracion SAML 2.0
- Integracion OIDC
- Wizard de configuracion SSO

### Fase 4: API & Webhooks (Semanas 8-9)
- API Keys management
- Endpoints REST para ATS
- Sistema de webhooks

### Fase 5: Analytics & Polish (Semanas 10-12)
- Cross-tenant analytics
- Reportes por tenant
- Branding white-label basico

---

## Decision Final

**Row-Level Security con Schema Compartido** es la estrategia optima para multi-tenancy en "Entrevistador Inteligente" porque:

1. **Aislamiento robusto**: RLS a nivel de base de datos previene leaks
2. **Costo-eficiente**: Un solo schema reduce costos de infraestructura
3. **Escalable**: Soporta desde 10 hasta 10,000+ tenants
4. **Operacionalmente simple**: Migraciones y backups estandar
5. **Enterprise-ready**: SSO, API, y quotas cubren requisitos B2B

---

## Referencias

- [PostgreSQL Row Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Multi-tenant SaaS Architecture - AWS](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/multi-tenancy-models.html)
- [SAML 2.0 Specification](https://docs.oasis-open.org/security/saml/v2.0/)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [Designing Multi-Tenant Applications - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
