# ADR-006: Estrategia de Autenticacion y Autorizacion

**Estado:** Aceptado
**Fecha:** 2026-01-31
**Autores:** Equipo de Arquitectura
**Revisores:** Security Lead, Product Owner

## Contexto

"Entrevistador Inteligente" requiere un sistema de autenticacion robusto que soporte multiples metodos de registro, autenticacion multi-factor via WhatsApp (preferido en el mercado peruano), y un modelo de autorizacion que permita multi-tenancy para clientes B2B. El sistema debe cumplir con la Ley de Proteccion de Datos Personales (LPDP) de Peru.

### Requisitos Funcionales

1. **Metodos de Registro:**
   - Email tradicional con verificacion
   - OAuth: Google, Facebook, Apple
   - Verificacion preferida via WhatsApp OTP (alto uso en Peru)

2. **Roles del Sistema:**
   - Candidato: Usuario final que realiza entrevistas
   - Empresa Admin: Administrador de cuenta empresarial
   - Empresa User: Usuario de empresa con permisos limitados
   - Super Admin: Administrador de la plataforma

3. **Multi-tenancy:**
   - Cada empresa es un tenant aislado
   - Admin puede invitar usuarios a su organizacion
   - Datos segregados por tenant

4. **Compliance LPDP:**
   - Consentimiento explicito para tratamiento de datos
   - Derecho de acceso, rectificacion y supresion
   - Registro de consentimientos

## Decision

### 1. Proveedor de Autenticacion: Supabase Auth

**Seleccionado:** Supabase Auth
**Alternativas evaluadas:** Auth0, Clerk, Custom

#### Matriz de Evaluacion

| Criterio | Peso | Supabase Auth | Auth0 | Clerk | Custom |
|----------|------|---------------|-------|-------|--------|
| Costo mensual (10K MAU) | 25% | $25 | $240 | $50 | $0* |
| OAuth Social | 15% | 9/10 | 10/10 | 10/10 | 7/10 |
| Integracion Supabase DB | 20% | 10/10 | 6/10 | 6/10 | 8/10 |
| WhatsApp OTP nativo | 15% | 3/10** | 5/10** | 2/10 | 10/10 |
| Time-to-market | 15% | 9/10 | 9/10 | 9/10 | 4/10 |
| Compliance (GDPR/LPDP) | 10% | 8/10 | 10/10 | 8/10 | 6/10 |
| **Score Ponderado** | | **7.65** | 7.35 | 6.80 | 6.25 |

*Custom requiere desarrollo y mantenimiento significativo
**Requiere integracion externa con WhatsApp Business API

#### Justificacion

1. **Integracion nativa con Supabase:** Row Level Security (RLS) integrado con auth.uid()
2. **Costo competitivo:** Tier gratuito generoso, pricing predecible
3. **OAuth pre-configurado:** Google, Facebook, Apple listos para usar
4. **JWT nativo:** Tokens compatibles con nuestra arquitectura serverless
5. **Extensibilidad:** Hooks para logica custom (WhatsApp OTP)

### 2. Estrategia de Tokens: JWT con Refresh Tokens

**Seleccionado:** JWT (stateless) + Refresh Token rotation
**Alternativa:** Session-based

```
+------------------+     +------------------+     +------------------+
|   Access Token   |     |  Refresh Token   |     |   Session Store  |
|------------------|     |------------------|     |------------------|
| Duracion: 15min  |     | Duracion: 7 dias |     | Redis (opcional) |
| Stateless        |     | One-time use     |     | Blacklist tokens |
| En memoria       |     | httpOnly cookie  |     | Revocacion       |
+------------------+     +------------------+     +------------------+
```

#### Estructura del JWT

```json
{
  "sub": "user_uuid",
  "email": "usuario@ejemplo.com",
  "role": "empresa_admin",
  "tenant_id": "empresa_uuid",
  "permissions": ["interviews:create", "interviews:read", "users:invite"],
  "iat": 1706745600,
  "exp": 1706746500,
  "iss": "entrevistador-inteligente"
}
```

#### Justificacion

- **Escalabilidad:** Sin estado en servidor, ideal para Edge Functions
- **Performance:** Validacion local sin round-trip a DB
- **Seguridad:** Refresh token rotation previene token reuse attacks
- **Compatibilidad:** Standard abierto, facil integracion con terceros

### 3. Modelo de Autorizacion: RBAC con Permisos Granulares

**Seleccionado:** Role-Based Access Control (RBAC) con permisos
**Alternativa:** Attribute-Based Access Control (ABAC)

#### Definicion de Roles y Permisos

```
+------------------+----------------------------------------+
|      Rol         |              Permisos                  |
+------------------+----------------------------------------+
| Candidato        | interviews:own:*, profile:own:*        |
|                  | results:own:read                       |
+------------------+----------------------------------------+
| Empresa User     | interviews:tenant:read,                |
|                  | candidates:tenant:read,                |
|                  | reports:tenant:read                    |
+------------------+----------------------------------------+
| Empresa Admin    | interviews:tenant:*,                   |
|                  | candidates:tenant:*,                   |
|                  | users:tenant:*,                        |
|                  | billing:tenant:*,                      |
|                  | settings:tenant:*                      |
+------------------+----------------------------------------+
| Super Admin      | *:*:* (acceso total)                   |
+------------------+----------------------------------------+
```

#### Implementacion con Supabase RLS

```sql
-- Politica para candidatos: solo sus propias entrevistas
CREATE POLICY "candidates_own_interviews" ON interviews
  FOR ALL
  USING (
    auth.uid() = candidate_id
    AND auth.jwt()->>'role' = 'candidato'
  );

-- Politica para empresa: entrevistas de su tenant
CREATE POLICY "empresa_tenant_interviews" ON interviews
  FOR SELECT
  USING (
    tenant_id = (auth.jwt()->>'tenant_id')::uuid
    AND auth.jwt()->>'role' IN ('empresa_admin', 'empresa_user')
  );
```

#### Justificacion de RBAC sobre ABAC

| Aspecto | RBAC | ABAC |
|---------|------|------|
| Complejidad | Baja | Alta |
| Performance | O(1) lookup | Evaluacion runtime |
| Auditoria | Simple | Compleja |
| Casos de uso | 95% cubiertos | Overkill para MVP |

ABAC se reserva para futuras reglas complejas (ej: "puede ver entrevistas creadas en ultimos 30 dias").

### 4. Multi-Factor Authentication: WhatsApp OTP

**Seleccionado:** WhatsApp Business API via Twilio
**Alternativa:** SMS tradicional, TOTP (Google Authenticator)

#### Flujo de WhatsApp OTP

```
+--------+      +------------+      +--------+      +-----------+
| Cliente| ---> | Supabase   | ---> | Edge   | ---> | Twilio    |
|        |      | Auth Hook  |      | Func   |      | WhatsApp  |
+--------+      +------------+      +--------+      +-----------+
    |                                                     |
    |              +------------------+                   |
    |<-------------|  WhatsApp msg    |<------------------|
    |              |  "Tu codigo: 123"|                   |
    |              +------------------+                   |
    |                                                     |
    +--> Ingresa OTP --> Valida --> Session creada
```

#### Configuracion Twilio WhatsApp

```typescript
// /supabase/functions/send-whatsapp-otp/index.ts
import { Twilio } from 'twilio';

const client = new Twilio(
  Deno.env.get('TWILIO_ACCOUNT_SID'),
  Deno.env.get('TWILIO_AUTH_TOKEN')
);

export async function sendWhatsAppOTP(phone: string, code: string) {
  await client.messages.create({
    from: 'whatsapp:+14155238886', // Twilio Sandbox o numero propio
    to: `whatsapp:${phone}`,
    body: `Tu codigo de verificacion para Entrevistador Inteligente es: ${code}. Valido por 5 minutos.`
  });
}
```

#### Costos WhatsApp Business API

| Volumen mensual | Costo por mensaje | Total estimado |
|-----------------|-------------------|----------------|
| 1,000 OTPs | $0.0042 (Peru) | $4.20 |
| 10,000 OTPs | $0.0042 | $42.00 |
| 50,000 OTPs | $0.0035 (volumen) | $175.00 |

#### Justificacion WhatsApp sobre SMS

1. **Penetracion en Peru:** 95% de usuarios moviles usan WhatsApp
2. **Costo:** 60% mas barato que SMS tradicional
3. **Confiabilidad:** Mayor tasa de entrega que SMS
4. **UX:** Usuarios familiarizados, no requiere app adicional

### 5. Politica de Contrasenas

```typescript
const passwordPolicy = {
  minLength: 8,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: false, // Reduce friccion
  preventCommonPasswords: true,
  preventUserInfoInPassword: true,
  historyCount: 5, // No reusar ultimas 5
  maxAge: null, // Sin expiracion forzada (NIST 800-63B)
};
```

#### Justificacion (alineado con NIST 800-63B)

- **Sin expiracion forzada:** Estudios muestran que reduce seguridad
- **Sin caracteres especiales obligatorios:** Aumenta friccion sin mejora significativa
- **Verificacion contra listas de contrasenas comunes:** Mas efectivo

## Flujos de Autenticacion

### Flujo 1: Registro de Candidato

```
+----------+     +----------+     +----------+     +----------+
|  1.Form  | --> | 2.Email  | --> |3.WhatsApp| --> |4.Perfil  |
|  Registro|     | Verify   |     |   OTP    |     | Completo |
+----------+     +----------+     +----------+     +----------+
     |                |                |                |
     v                v                v                v
  Email,pass      Click link       6 digitos       Onboarding
  +telefono       en email         via WA          completado
```

```typescript
// Paso 1: Registro inicial
const { user, error } = await supabase.auth.signUp({
  email: 'candidato@email.com',
  password: 'securePassword123',
  options: {
    data: {
      phone: '+51999888777',
      role: 'candidato',
      consent_lpdp: true,
      consent_date: new Date().toISOString()
    }
  }
});

// Paso 2: Despues de verificar email, enviar OTP WhatsApp
await supabase.functions.invoke('send-whatsapp-otp', {
  body: { phone: '+51999888777' }
});

// Paso 3: Verificar OTP
await supabase.functions.invoke('verify-whatsapp-otp', {
  body: { phone: '+51999888777', code: '123456' }
});
```

### Flujo 2: Login Social (Google/Facebook)

```
+----------+     +----------+     +----------+     +----------+
| 1.Click  | --> | 2.OAuth  | --> | 3.Return | --> |4.Session |
| "Google" |     | Consent  |     | Callback |     | Created  |
+----------+     +----------+     +----------+     +----------+
```

```typescript
// Iniciar OAuth
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: 'https://app.entrevistadorinteligente.com/auth/callback',
    scopes: 'email profile'
  }
});

// Callback handler
const { data: { session } } = await supabase.auth.getSession();
```

### Flujo 3: Registro de Empresa

```
+----------+     +----------+     +----------+     +----------+
| 1.Admin  | --> | 2.Create | --> | 3.Invite | --> | 4.Users  |
| Registro |     | Tenant   |     | Link     |     | Join     |
+----------+     +----------+     +----------+     +----------+
     |                |                |                |
     v                v                v                v
  Datos empresa   tenant_id        Email magic      Aceptan
  + admin info    generado         link enviado     invitacion
```

```typescript
// Admin crea empresa
const { data: tenant } = await supabase
  .from('tenants')
  .insert({
    name: 'Empresa SAC',
    ruc: '20123456789',
    plan: 'business',
    admin_id: user.id
  })
  .select()
  .single();

// Invitar usuario
await supabase.functions.invoke('invite-user', {
  body: {
    email: 'usuario@empresa.com',
    tenant_id: tenant.id,
    role: 'empresa_user'
  }
});
```

### Flujo 4: Password Reset

```
+----------+     +----------+     +----------+     +----------+
| 1.Request| --> | 2.Email  | --> | 3.New    | --> |4.Success |
| Reset    |     | Link     |     | Password |     | Login    |
+----------+     +----------+     +----------+     +----------+
```

```typescript
// Solicitar reset
await supabase.auth.resetPasswordForEmail('user@email.com', {
  redirectTo: 'https://app.entrevistadorinteligente.com/reset-password'
});

// Actualizar password
await supabase.auth.updateUser({
  password: 'newSecurePassword456'
});
```

### Flujo 5: Session Management

```typescript
// Configuracion de sesion
const sessionConfig = {
  accessTokenLifetime: 900,      // 15 minutos
  refreshTokenLifetime: 604800,  // 7 dias
  refreshTokenRotation: true,
  refreshTokenReuseInterval: 10, // 10 segundos grace period
};

// Refresh automatico en cliente
supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'TOKEN_REFRESHED') {
    // Token actualizado automaticamente
  }
  if (event === 'SIGNED_OUT') {
    // Limpiar estado local
    router.push('/login');
  }
});

// Logout (invalida refresh token)
await supabase.auth.signOut();
```

## Esquema de Base de Datos

```sql
-- Tabla de tenants (empresas)
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  ruc VARCHAR(11) UNIQUE,
  plan TEXT DEFAULT 'free',
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Extension de perfil de usuario
CREATE TABLE user_profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  tenant_id UUID REFERENCES tenants(id),
  role TEXT NOT NULL CHECK (role IN ('candidato', 'empresa_admin', 'empresa_user', 'super_admin')),
  phone VARCHAR(20),
  phone_verified BOOLEAN DEFAULT FALSE,
  lpdp_consent BOOLEAN DEFAULT FALSE,
  lpdp_consent_date TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Invitaciones pendientes
CREATE TABLE invitations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL,
  tenant_id UUID REFERENCES tenants(id) NOT NULL,
  role TEXT NOT NULL,
  token TEXT UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  accepted_at TIMESTAMPTZ,
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Audit log para compliance
CREATE TABLE auth_audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  action TEXT NOT NULL,
  ip_address INET,
  user_agent TEXT,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indices
CREATE INDEX idx_user_profiles_tenant ON user_profiles(tenant_id);
CREATE INDEX idx_invitations_token ON invitations(token);
CREATE INDEX idx_auth_audit_user ON auth_audit_log(user_id, created_at);
```

## Compliance LPDP

### Requisitos Implementados

| Requisito LPDP | Implementacion |
|----------------|----------------|
| Consentimiento informado | Checkbox obligatorio + registro en DB |
| Finalidad especifica | Texto explicativo en registro |
| Derecho de acceso | Endpoint /api/user/data-export |
| Derecho de rectificacion | UI para editar perfil |
| Derecho de supresion | Endpoint /api/user/delete-account |
| Seguridad | Cifrado, RLS, audit logs |

### Registro de Consentimiento

```typescript
const consentRecord = {
  user_id: user.id,
  consent_type: 'lpdp_data_processing',
  consent_text: 'Acepto el tratamiento de mis datos personales...',
  consent_version: '1.0',
  ip_address: request.headers.get('x-forwarded-for'),
  user_agent: request.headers.get('user-agent'),
  granted_at: new Date().toISOString()
};
```

## Seguridad

### Medidas Implementadas

1. **Rate Limiting:**
   - Login: 5 intentos / 15 minutos
   - OTP: 3 intentos / 5 minutos
   - Password reset: 3 / hora

2. **Proteccion contra ataques:**
   - CSRF tokens en forms
   - Secure + HttpOnly + SameSite cookies
   - Content Security Policy headers

3. **Monitoreo:**
   - Alertas en logins desde nueva ubicacion
   - Notificacion de cambio de password
   - Audit log de acciones sensibles

### Headers de Seguridad

```typescript
const securityHeaders = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'unsafe-inline'",
  'Referrer-Policy': 'strict-origin-when-cross-origin'
};
```

## Consecuencias

### Positivas

1. **Integracion nativa:** Supabase Auth + RLS simplifican autorizacion
2. **Costo optimizado:** ~$70/mes para 10K usuarios (auth + WhatsApp)
3. **UX local:** WhatsApp OTP familiar para usuarios peruanos
4. **Escalabilidad:** JWT stateless escala horizontalmente
5. **Compliance:** Arquitectura preparada para LPDP

### Negativas

1. **Vendor lock-in:** Dependencia de Supabase (mitigado: standard JWT)
2. **WhatsApp dependency:** Requiere cuenta Business verificada
3. **Complejidad OTP:** Logica custom para WhatsApp vs SMS nativo

### Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Supabase downtime | Baja | Alto | Fallback a magic link email |
| WhatsApp API changes | Media | Medio | Abstraccion de proveedor OTP |
| Token theft | Baja | Alto | Refresh rotation + short-lived access |
| Brute force | Media | Medio | Rate limiting + account lockout |

## Metricas de Exito

| Metrica | Target | Medicion |
|---------|--------|----------|
| Tasa de registro exitoso | > 85% | Completados / Iniciados |
| Tiempo de verificacion OTP | < 30s | P95 latencia |
| Tasa de login exitoso | > 95% | Logins / Intentos |
| Incidentes de seguridad | 0 | Monitoreo continuo |

## Referencias

- [Supabase Auth Documentation](https://supabase.com/docs/guides/auth)
- [NIST 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio WhatsApp Business API](https://www.twilio.com/whatsapp)
- [Ley 29733 - LPDP Peru](https://www.gob.pe/institucion/congreso/normas-legales/243470-29733)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

## Historial de Cambios

| Version | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Version inicial |
