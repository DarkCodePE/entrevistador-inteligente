# ADR-009: Integracion de Pagos para Peru

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead, CFO

---

## Contexto

### Descripcion del Problema
"Entrevistador Inteligente Peru" requiere una estrategia integral de pagos que:
- Soporte los metodos de pago mas utilizados en Peru
- Maneje suscripciones B2C y contratos B2B
- Cumpla con regulaciones tributarias peruanas (SUNAT)
- Escale a otros mercados LATAM en el futuro

### Panorama de Pagos en Peru

| Metodo | Penetracion | Segmento | Prioridad |
|--------|-------------|----------|-----------|
| **Yape** | 15M+ usuarios | B2C masivo | Alta |
| **Plin** | 8M+ usuarios | B2C masivo | Alta |
| **Tarjetas Visa/MC** | 40% bancarizados | B2C premium, B2B | Alta |
| **PayPal** | Usuarios tech | B2C premium | Media |
| **PagoEfectivo** | No bancarizados | B2C | Media |
| **Transferencia Bancaria** | Empresas | B2B | Alta |

### Modelos de Pricing Definidos

#### B2C - Freemium

| Plan | Precio | Caracteristicas |
|------|--------|-----------------|
| **Free** | S/0 | 3 simulaciones/mes, CV basico |
| **Premium** | S/29.90/mes | Simulaciones ilimitadas, CV optimizado |
| **Pro** | S/79.90/mes | Todo Premium + Tech Interviews especializadas |

#### B2B - Enterprise

| Plan | Precio | Caracteristicas |
|------|--------|-----------------|
| **Starter** | S/499/mes | Hasta 20 usuarios, reportes basicos |
| **Growth** | S/999/mes | Hasta 100 usuarios, analytics avanzado |
| **Enterprise** | Custom | Usuarios ilimitados, SLA, soporte dedicado |

### Requisitos Regulatorios Peru

| Requisito | Descripcion | Obligatoriedad |
|-----------|-------------|----------------|
| **IGV** | 18% sobre servicios digitales | Obligatorio |
| **Facturacion Electronica** | Integracion con SUNAT | Obligatorio desde 2023 |
| **Retenciones B2B** | 3% agentes de retencion | Aplica a clientes B2B designados |
| **Comprobantes** | Boleta (B2C) / Factura (B2B) | Obligatorio |
| **Bancarizacion** | Pagos >S/2,000 por transferencia | Obligatorio |

---

## Decision

### Decision 1: Payment Gateway Principal

**Adoptamos Culqi como Payment Gateway principal, con Mercado Pago como secundario.**

#### Comparativa de Gateways

| Criterio | Culqi | Mercado Pago | Stripe |
|----------|-------|--------------|--------|
| **Tarjetas** | Si | Si | Si |
| **Yape** | Si (nativo) | No | No |
| **Plin** | Si | No | No |
| **PagoEfectivo** | Si | Si | No |
| **Comision tarjetas** | 3.99% + S/1 | 4.49% + S/1.40 | 2.9% + $0.30 |
| **Comision Yape/Plin** | 1.5% | N/A | N/A |
| **Facturacion SUNAT** | Integrado | Parcial | No |
| **Soporte local** | Lima | Regional | Remoto |
| **SDK calidad** | Bueno | Bueno | Excelente |
| **Suscripciones** | Basico | Si | Excelente |
| **Expansion LATAM** | Peru | Regional | Global |

#### Justificacion

1. **Yape/Plin nativo**: Culqi es el unico gateway que soporta billeteras digitales peruanas directamente
2. **Comisiones competitivas**: 1.5% en Yape vs 3.99% en tarjetas reduce CAC
3. **Integracion SUNAT**: Facilita cumplimiento tributario
4. **Soporte local**: Resolucion de problemas en horario Peru

```
┌─────────────────────────────────────────────────────────────┐
│                    Payment Module                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐│
│  │    Culqi      │    │ Mercado Pago  │    │    PayPal     ││
│  │   (Primary)   │    │  (Secondary)  │    │   (Premium)   ││
│  │               │    │               │    │               ││
│  │ - Yape/Plin   │    │ - Backup      │    │ - USD         ││
│  │ - Tarjetas    │    │ - PagoEfectivo│    │ - Enterprise  ││
│  │ - SUNAT       │    │ - Expansion   │    │               ││
│  └───────────────┘    └───────────────┘    └───────────────┘│
│           │                   │                   │          │
│           └───────────────────┼───────────────────┘          │
│                               ▼                              │
│                    ┌───────────────────┐                     │
│                    │  Payment Router   │                     │
│                    │  (Seleccion auto) │                     │
│                    └───────────────────┘                     │
│                               │                              │
│           ┌───────────────────┼───────────────────┐          │
│           ▼                   ▼                   ▼          │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐  │
│  │  Subscription │   │   Invoice     │   │   Webhook     │  │
│  │   Manager     │   │   Generator   │   │   Handler     │  │
│  └───────────────┘   └───────────────┘   └───────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Decision 2: Gestion de Suscripciones

**Adoptamos solucion hibrida: Culqi Subscriptions para B2C + Sistema propio para B2B.**

#### Arquitectura de Suscripciones

```
┌─────────────────────────────────────────────────────────────┐
│                 Subscription Management                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  B2C Subscriptions                       ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  ││
│  │  │   Free      │  │  Premium    │  │      Pro        │  ││
│  │  │   Tier      │  │  S/29.90    │  │    S/79.90      │  ││
│  │  │             │  │             │  │                 │  ││
│  │  │ - 3 sims    │  │ - Unlimited │  │ - All Premium   │  ││
│  │  │ - Basic CV  │  │ - CV Opt    │  │ - Tech Int.     │  ││
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  ││
│  │                                                          ││
│  │  Gateway: Culqi Subscriptions (auto-billing)             ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  B2B Subscriptions                       ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  ││
│  │  │  Starter    │  │   Growth    │  │   Enterprise    │  ││
│  │  │  S/499/mes  │  │  S/999/mes  │  │    Custom       │  ││
│  │  │             │  │             │  │                 │  ││
│  │  │ - 20 users  │  │ - 100 users │  │ - Unlimited     │  ││
│  │  │ - Basic     │  │ - Advanced  │  │ - SLA           │  ││
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  ││
│  │                                                          ││
│  │  Sistema: Interno (facturacion mensual, transferencia)   ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### Flujo de Suscripcion B2C

```
Usuario                    App                     Culqi                   SUNAT
   │                        │                        │                        │
   │  1. Selecciona Plan    │                        │                        │
   │───────────────────────>│                        │                        │
   │                        │                        │                        │
   │  2. Elige Metodo Pago  │                        │                        │
   │<───────────────────────│                        │                        │
   │───────────────────────>│                        │                        │
   │                        │                        │                        │
   │                        │  3. Crear Suscripcion  │                        │
   │                        │───────────────────────>│                        │
   │                        │                        │                        │
   │                        │  4. Procesar Pago      │                        │
   │                        │<───────────────────────│                        │
   │                        │                        │                        │
   │                        │  5. Generar Boleta     │                        │
   │                        │───────────────────────────────────────────────>│
   │                        │                        │                        │
   │  6. Confirmar Plan     │                        │                        │
   │<───────────────────────│                        │                        │
   │                        │                        │                        │
   │                        │  [Mensual - Auto]      │                        │
   │                        │  7. Cobro Recurrente   │                        │
   │                        │<───────────────────────│                        │
   │                        │                        │                        │
```

### Decision 3: Facturacion Electronica SUNAT

**Adoptamos Nubefact como proveedor de facturacion electronica.**

#### Comparativa PSE (Proveedor de Servicios Electronicos)

| Criterio | Nubefact | Efact | Bizlinks |
|----------|----------|-------|----------|
| **API REST** | Si | Si | Si |
| **Costo/documento** | S/0.15 | S/0.18 | S/0.20 |
| **Integracion Culqi** | Nativa | Manual | Manual |
| **Uptime SLA** | 99.9% | 99.5% | 99.5% |
| **Soporte** | 24/7 | Horario oficina | Horario oficina |

#### Tipos de Comprobantes

| Tipo | Uso | Serie |
|------|-----|-------|
| **Boleta (B)** | Ventas B2C | B001-XXXXX |
| **Factura (F)** | Ventas B2B | F001-XXXXX |
| **Nota de Credito** | Devoluciones | BC01/FC01 |
| **Nota de Debito** | Ajustes | BD01/FD01 |

#### Flujo de Emision

```
┌─────────────────────────────────────────────────────────────┐
│                  Invoice Generation Flow                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────┐                                          │
│  │    Payment    │                                          │
│  │   Confirmed   │                                          │
│  └───────┬───────┘                                          │
│          │                                                   │
│          ▼                                                   │
│  ┌───────────────┐                                          │
│  │  Determinar   │                                          │
│  │ Tipo Cliente  │                                          │
│  └───────┬───────┘                                          │
│          │                                                   │
│    ┌─────┴─────┐                                            │
│    ▼           ▼                                            │
│ ┌─────┐    ┌─────┐                                          │
│ │ B2C │    │ B2B │                                          │
│ │ DNI │    │ RUC │                                          │
│ └──┬──┘    └──┬──┘                                          │
│    │          │                                              │
│    ▼          ▼                                              │
│ ┌───────┐ ┌───────┐                                         │
│ │Boleta │ │Factura│                                         │
│ └───┬───┘ └───┬───┘                                         │
│     │         │                                              │
│     └────┬────┘                                              │
│          │                                                   │
│          ▼                                                   │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐│
│  │   Nubefact    │───>│     SUNAT     │───>│   Usuario     ││
│  │   API Call    │    │  Validacion   │    │  Email/PDF    ││
│  └───────────────┘    └───────────────┘    └───────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Decision 4: Manejo de IGV y Retenciones

**Adoptamos modelo de precios con IGV incluido para B2C, IGV desglosado para B2B.**

#### Estructura de Precios con IGV

| Plan | Precio Usuario | Base Imponible | IGV (18%) | Total |
|------|---------------|----------------|-----------|-------|
| Premium B2C | S/29.90 | S/25.34 | S/4.56 | S/29.90 |
| Pro B2C | S/79.90 | S/67.71 | S/12.19 | S/79.90 |
| Starter B2B | S/499 + IGV | S/499.00 | S/89.82 | S/588.82 |
| Growth B2B | S/999 + IGV | S/999.00 | S/179.82 | S/1,178.82 |

#### Manejo de Retenciones B2B

```
┌─────────────────────────────────────────────────────────────┐
│              B2B Payment with Retention                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Factura Total: S/1,178.82 (S/999 + IGV)                    │
│                                                              │
│  Cliente es Agente de Retencion?                            │
│       │                                                      │
│  ┌────┴────┐                                                │
│  │         │                                                │
│  ▼         ▼                                                │
│ [SI]      [NO]                                              │
│  │         │                                                │
│  ▼         ▼                                                │
│ Retencion  Pago                                             │
│ 3%         Total                                            │
│  │         │                                                │
│  ▼         │                                                │
│ S/35.36    │                                                │
│ retenido   │                                                │
│  │         │                                                │
│  ▼         ▼                                                │
│ Pago:      Pago:                                            │
│ S/1,143.46 S/1,178.82                                       │
│                                                              │
│ Comprobante de Retencion emitido por cliente                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Decision 5: Politica de Reembolsos

**Adoptamos politica de reembolso proporcional con periodo de gracia de 7 dias.**

| Escenario | Politica | Accion |
|-----------|----------|--------|
| **Cancelacion < 7 dias** | Reembolso 100% | Nota de credito |
| **Cancelacion 8-15 dias** | Reembolso 50% | Nota de credito parcial |
| **Cancelacion > 15 dias** | Sin reembolso | Acceso hasta fin de periodo |
| **Falla tecnica nuestra** | Reembolso 100% + 1 mes gratis | Nota de credito + credito |
| **Fraude detectado** | Sin reembolso | Suspension cuenta |

#### Flujo de Reembolso

```
Solicitud         Evaluacion        Aprobacion         Ejecucion
    │                 │                  │                  │
    ▼                 ▼                  ▼                  ▼
┌─────────┐    ┌───────────┐     ┌───────────┐     ┌───────────┐
│ Usuario │───>│  Sistema  │────>│  Soporte  │────>│  Culqi    │
│ Solicita│    │ Verifica  │     │  Aprueba  │     │  Refund   │
└─────────┘    │ Politica  │     └───────────┘     └───────────┘
               └───────────┘            │                  │
                    │                   │                  ▼
                    │                   │          ┌───────────┐
                    │                   │          │ Nubefact  │
                    │                   └─────────>│ N. Credito│
                    │                              └───────────┘
               < 48 horas SLA
```

### Decision 6: PCI DSS Compliance

**Adoptamos modelo SAQ-A (minima exposicion) delegando tokenizacion a Culqi.**

#### Modelo de Seguridad

```
┌─────────────────────────────────────────────────────────────┐
│                    PCI DSS Compliance                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  NUNCA en nuestros servidores:                              │
│  ✗ Numero de tarjeta (PAN)                                  │
│  ✗ CVV/CVC                                                  │
│  ✗ Fecha de expiracion                                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Frontend                              ││
│  │  ┌───────────────┐                                      ││
│  │  │  Culqi.js     │  Tokenizacion en cliente             ││
│  │  │  (iframe)     │  Datos sensibles -> Culqi directo    ││
│  │  └───────┬───────┘                                      ││
│  │          │                                               ││
│  │          ▼                                               ││
│  │  ┌───────────────┐                                      ││
│  │  │    Token      │  Solo token viaja a backend          ││
│  │  │  (tkn_xxxxx)  │                                      ││
│  │  └───────┬───────┘                                      ││
│  └──────────┼──────────────────────────────────────────────┘│
│             │                                                │
│             ▼                                                │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Backend                               ││
│  │  ┌───────────────┐    ┌───────────────┐                 ││
│  │  │ Token + Monto │───>│  Culqi API    │                 ││
│  │  │ (sin datos    │    │  (procesa)    │                 ││
│  │  │  sensibles)   │    │               │                 ││
│  │  └───────────────┘    └───────────────┘                 ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Cumplimiento: SAQ-A (22 controles vs 300+ en SAQ-D)        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Decision 7: Moneda y Expansion Regional

**Adoptamos PEN como moneda primaria, USD para planes Enterprise internacionales.**

| Segmento | Moneda | Gateway | Facturacion |
|----------|--------|---------|-------------|
| B2C Peru | PEN | Culqi | SUNAT |
| B2B Peru | PEN | Culqi/Transferencia | SUNAT |
| B2B Internacional | USD | PayPal/Stripe | Invoice internacional |

#### Plan de Expansion Regional

```
Fase 1: Peru             Fase 2: Colombia          Fase 3: Mexico
(2026)                   (2027)                    (2028)

┌──────────────┐        ┌──────────────┐         ┌──────────────┐
│    Culqi     │        │   PayU       │         │   Conekta    │
│    PEN       │        │   COP        │         │   MXN        │
│    SUNAT     │        │   DIAN       │         │   SAT        │
│              │        │              │         │              │
│  Yape/Plin   │        │  Nequi/      │         │  CoDi/SPEI   │
│              │        │  Daviplata   │         │              │
└──────────────┘        └──────────────┘         └──────────────┘
```

---

## Arquitectura Tecnica

### Modelo de Datos

```sql
-- Esquema: payments

CREATE TABLE payment_methods (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),
    type VARCHAR(20) NOT NULL, -- 'card', 'yape', 'plin', 'bank_transfer'
    provider VARCHAR(20) NOT NULL, -- 'culqi', 'mercadopago', 'paypal'
    token VARCHAR(255), -- Token del gateway
    last_four VARCHAR(4), -- Ultimos 4 digitos (tarjetas)
    brand VARCHAR(20), -- 'visa', 'mastercard', 'yape', 'plin'
    is_default BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    company_id UUID REFERENCES companies(id),
    plan_code VARCHAR(20) NOT NULL, -- 'free', 'premium', 'pro', 'starter', 'growth', 'enterprise'
    status VARCHAR(20) NOT NULL, -- 'active', 'cancelled', 'past_due', 'trialing'
    billing_cycle VARCHAR(10) NOT NULL, -- 'monthly', 'yearly'
    current_period_start TIMESTAMP NOT NULL,
    current_period_end TIMESTAMP NOT NULL,
    cancel_at_period_end BOOLEAN DEFAULT false,
    gateway_subscription_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE invoices (
    id UUID PRIMARY KEY,
    subscription_id UUID REFERENCES subscriptions(id),
    user_id UUID REFERENCES users(id),
    company_id UUID REFERENCES companies(id),

    -- Montos
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL, -- IGV
    retention_amount DECIMAL(10,2) DEFAULT 0, -- Retencion B2B
    total DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'PEN',

    -- SUNAT
    document_type VARCHAR(10) NOT NULL, -- 'boleta', 'factura'
    document_series VARCHAR(10) NOT NULL, -- 'B001', 'F001'
    document_number VARCHAR(20) NOT NULL,
    sunat_response_code VARCHAR(10),
    sunat_hash VARCHAR(255),

    -- Cliente
    customer_doc_type VARCHAR(3), -- 'DNI', 'RUC'
    customer_doc_number VARCHAR(11),
    customer_name VARCHAR(255),
    customer_address TEXT,

    status VARCHAR(20) NOT NULL, -- 'draft', 'pending', 'paid', 'void', 'refunded'
    paid_at TIMESTAMP,
    voided_at TIMESTAMP,

    pdf_url TEXT,
    xml_url TEXT,

    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE payments (
    id UUID PRIMARY KEY,
    invoice_id UUID REFERENCES invoices(id),
    payment_method_id UUID REFERENCES payment_methods(id),

    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'PEN',

    gateway VARCHAR(20) NOT NULL,
    gateway_payment_id VARCHAR(255),
    gateway_response JSONB,

    status VARCHAR(20) NOT NULL, -- 'pending', 'completed', 'failed', 'refunded'
    failure_reason TEXT,

    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE refunds (
    id UUID PRIMARY KEY,
    payment_id UUID REFERENCES payments(id),
    invoice_id UUID REFERENCES invoices(id),

    amount DECIMAL(10,2) NOT NULL,
    reason VARCHAR(50) NOT NULL, -- 'customer_request', 'technical_issue', 'fraud'

    credit_note_series VARCHAR(10),
    credit_note_number VARCHAR(20),

    gateway_refund_id VARCHAR(255),
    status VARCHAR(20) NOT NULL, -- 'pending', 'completed', 'failed'

    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indices
CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_company ON subscriptions(company_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
CREATE INDEX idx_invoices_subscription ON invoices(subscription_id);
CREATE INDEX idx_invoices_document ON invoices(document_series, document_number);
CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

### Estructura de Modulo

```
src/modules/payments/
├── domain/
│   ├── entities/
│   │   ├── PaymentMethod.ts
│   │   ├── Subscription.ts
│   │   ├── Invoice.ts
│   │   ├── Payment.ts
│   │   └── Refund.ts
│   ├── value-objects/
│   │   ├── Money.ts
│   │   ├── TaxInfo.ts
│   │   ├── DocumentNumber.ts  # RUC/DNI
│   │   └── InvoiceNumber.ts
│   ├── events/
│   │   ├── PaymentCompleted.ts
│   │   ├── SubscriptionCreated.ts
│   │   ├── SubscriptionCancelled.ts
│   │   └── InvoiceGenerated.ts
│   └── repositories/
│       ├── IPaymentMethodRepository.ts
│       ├── ISubscriptionRepository.ts
│       └── IInvoiceRepository.ts
├── application/
│   ├── commands/
│   │   ├── CreateSubscription.ts
│   │   ├── CancelSubscription.ts
│   │   ├── ProcessPayment.ts
│   │   ├── ProcessRefund.ts
│   │   └── AddPaymentMethod.ts
│   ├── queries/
│   │   ├── GetSubscriptionStatus.ts
│   │   ├── GetInvoiceHistory.ts
│   │   └── GetPaymentMethods.ts
│   └── services/
│       ├── SubscriptionService.ts
│       ├── PaymentProcessingService.ts
│       └── InvoiceGenerationService.ts
├── infrastructure/
│   ├── gateways/
│   │   ├── CulqiGateway.ts
│   │   ├── MercadoPagoGateway.ts
│   │   ├── PayPalGateway.ts
│   │   └── PaymentGatewayFactory.ts
│   ├── billing/
│   │   ├── NubefactClient.ts
│   │   └── SunatValidator.ts
│   ├── repositories/
│   │   ├── PostgresPaymentMethodRepository.ts
│   │   ├── PostgresSubscriptionRepository.ts
│   │   └── PostgresInvoiceRepository.ts
│   └── webhooks/
│       ├── CulqiWebhookHandler.ts
│       └── MercadoPagoWebhookHandler.ts
└── api/
    ├── PaymentsController.ts
    ├── SubscriptionsController.ts
    ├── InvoicesController.ts
    └── WebhooksController.ts
```

### Integracion con Otros Modulos

```
┌─────────────────────────────────────────────────────────────┐
│                    Module Integration                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────┐                                          │
│  │   Identity    │◄───── Usuario autenticado                │
│  │    Module     │       Empresa verificada                 │
│  └───────┬───────┘                                          │
│          │                                                   │
│          │  UserCreated                                      │
│          │  CompanyVerified                                  │
│          ▼                                                   │
│  ┌───────────────┐                                          │
│  │   Payments    │                                          │
│  │    Module     │                                          │
│  └───────┬───────┘                                          │
│          │                                                   │
│          │  SubscriptionActivated                           │
│          │  SubscriptionCancelled                           │
│          │  PaymentCompleted                                │
│          ▼                                                   │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐│
│  │ CV Management │    │ Interview Sim │    │Company Portal ││
│  │    Module     │    │    Module     │    │    Module     ││
│  │               │    │               │    │               ││
│  │ Desbloquea    │    │ Desbloquea    │    │ Activa plan   ││
│  │ features      │    │ simulaciones  │    │ enterprise    ││
│  │ premium       │    │ ilimitadas    │    │               ││
│  └───────────────┘    └───────────────┘    └───────────────┘│
│                                                              │
│  Event Bus: RabbitMQ / Internal                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Cobertura de mercado** | 95%+ de metodos de pago peruanos cubiertos |
| **Comisiones optimizadas** | 1.5% Yape vs 4%+ tarjetas reduce CAC |
| **Cumplimiento tributario** | Integracion SUNAT automatizada |
| **Escalabilidad regional** | Arquitectura preparada para LATAM |
| **Seguridad PCI** | SAQ-A minimiza responsabilidad |
| **Flexibilidad B2B** | Soporte para contratos y facturacion custom |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Dependencia Culqi** | Mercado Pago como backup |
| **Complejidad tributaria** | Nubefact automatiza emision |
| **Multiples integraciones** | Facade unificado para gateways |
| **Costo Nubefact** | ~S/150/mes por 1000 documentos |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Culqi downtime | Baja | Alto | Failover a Mercado Pago |
| Cambios regulatorios SUNAT | Media | Medio | Monitoreo constante, Nubefact actualiza |
| Fraude con Yape | Baja | Medio | Verificacion OTP, limites diarios |
| Disputas de tarjeta | Media | Bajo | Documentacion clara, 3D Secure |

---

## Metricas de Exito

| Metrica | Objetivo MVP | Objetivo Ano 1 |
|---------|--------------|----------------|
| Tasa de conversion pago | > 60% | > 75% |
| Tiempo promedio checkout | < 2 min | < 1 min |
| Pagos fallidos | < 5% | < 2% |
| Disputas/chargebacks | < 0.5% | < 0.3% |
| NPS proceso de pago | > 40 | > 60 |
| Emision SUNAT exitosa | > 99% | > 99.5% |

---

## Costos Estimados

### MVP (Mes 1-3)

| Concepto | Costo Mensual |
|----------|---------------|
| Culqi (comisiones) | Variable (~3% revenue) |
| Nubefact (1000 docs) | S/150 |
| Desarrollo integracion | 2 sprints (incluido en equipo) |
| **Total fijo** | **~S/150/mes** |

### Crecimiento (Mes 4-12)

| Concepto | Costo Mensual |
|----------|---------------|
| Culqi (comisiones) | Variable (~2.5% revenue negociado) |
| Nubefact (5000 docs) | S/500 |
| PayPal (premium) | Variable |
| **Total fijo** | **~S/500/mes** |

---

## Cronograma de Implementacion

### Fase 1: MVP (Semanas 1-4)

| Semana | Entregable |
|--------|------------|
| 1 | Integracion Culqi basica (tarjetas) |
| 2 | Yape/Plin + Suscripciones B2C |
| 3 | Integracion Nubefact (boletas) |
| 4 | Testing E2E + Go-live |

### Fase 2: B2B (Semanas 5-8)

| Semana | Entregable |
|--------|------------|
| 5 | Sistema de facturacion B2B |
| 6 | Transferencias bancarias |
| 7 | Retenciones + Notas de credito |
| 8 | Portal de administracion pagos |

### Fase 3: Optimizacion (Semanas 9-12)

| Semana | Entregable |
|--------|------------|
| 9-10 | Mercado Pago backup |
| 11 | PayPal para premium |
| 12 | Dashboards y reportes |

---

## Alternativas Consideradas

### Alternativa A: Stripe como Gateway Principal

**Pros:**
- API superior, mejor DX
- Suscripciones robustas
- Expansion global facil

**Contras:**
- No soporta Yape/Plin
- Comisiones en USD (tipo de cambio)
- Sin integracion SUNAT

**Veredicto:** Rechazado - No cubre metodos de pago locales criticos

### Alternativa B: Mercado Pago como Principal

**Pros:**
- Presencia regional fuerte
- PagoEfectivo integrado
- Marketplace ready

**Contras:**
- Comisiones mas altas
- Sin Yape/Plin nativo
- Menor integracion SUNAT

**Veredicto:** Rechazado como principal, mantenido como secundario

### Alternativa C: In-house Billing Completo

**Pros:**
- Control total
- Sin dependencias
- Costos marginales bajos

**Contras:**
- 4-6 meses desarrollo
- Mantenimiento regulatorio
- Complejidad PCI

**Veredicto:** Rechazado - Time-to-market inaceptable

---

## Referencias

- [Culqi Documentacion](https://docs.culqi.com/)
- [Nubefact API](https://docs.nubefact.com/)
- [SUNAT - Facturacion Electronica](https://www.sunat.gob.pe/ol-ti-itcpe/SEE)
- [PCI DSS SAQ-A Requirements](https://www.pcisecuritystandards.org/)
- [SBS Peru - Regulaciones de Pagos](https://www.sbs.gob.pe/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
