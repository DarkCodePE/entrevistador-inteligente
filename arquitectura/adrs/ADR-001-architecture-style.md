# ADR-001: Estilo de Arquitectura General - Modular Monolith

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead

---

## Contexto

### Descripcion del Sistema
"Entrevistador Inteligente Peru" es una plataforma SaaS que integra:
- Gestion y optimizacion de CVs con IA
- Matching inteligente entre candidatos y vacantes
- Simulacion de entrevistas con IA conversacional
- Panel B2B para empresas reclutadoras

### Modelo de Negocio
- **B2C**: Candidatos que buscan empleo (freemium + premium)
- **B2B**: Empresas que publican vacantes y acceden a candidatos

### Restricciones del Proyecto
| Factor | Detalle |
|--------|---------|
| Equipo | 2-4 desarrolladores iniciales |
| Presupuesto | Pre-seed $50-150K USD |
| Timeline | MVP en 8-12 semanas |
| Mercado inicial | Peru |
| Expansion | LATAM (Colombia, Mexico, Chile) |

### Problema a Resolver
Necesitamos definir el estilo arquitectonico que permita:
1. Entregar un MVP funcional en 8-12 semanas
2. Mantener costos de infraestructura bajos en etapa inicial
3. Soportar cargas de trabajo de IA/ML (procesamiento de CVs, NLP, simulaciones)
4. Escalar horizontalmente cuando el negocio lo requiera
5. Facilitar la expansion a nuevos mercados

---

## Decision

**Adoptamos Modular Monolith como estilo arquitectonico para el MVP y fase de crecimiento inicial.**

### Estructura de Modulos Propuesta

```
entrevistador-inteligente/
├── src/
│   ├── modules/
│   │   ├── identity/           # Autenticacion, usuarios, roles
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── api/
│   │   ├── cv-management/      # CRUD CVs, parsing, optimizacion
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── api/
│   │   ├── job-matching/       # Algoritmos de matching, scoring
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── api/
│   │   ├── interview-sim/      # Simulador de entrevistas IA
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── api/
│   │   ├── company-portal/     # Panel B2B, vacantes
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── api/
│   │   ├── analytics/          # Metricas, reportes
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── api/
│   │   └── shared/             # Kernel compartido
│   │       ├── domain/
│   │       └── infrastructure/
│   ├── api-gateway/            # Punto de entrada unificado
│   └── workers/                # Jobs asincrónos (IA processing)
├── infrastructure/
│   ├── docker/
│   ├── terraform/
│   └── k8s/
└── tests/
```

### Principios de Modularidad

1. **Boundaries claros**: Cada modulo tiene su propio dominio, sin dependencias circulares
2. **Comunicacion via interfaces**: Modulos se comunican a traves de contratos definidos
3. **Base de datos compartida con schemas separados**: Un PostgreSQL con schemas por modulo
4. **Event Bus interno**: Para comunicacion asincrona entre modulos
5. **Preparado para extraccion**: Cualquier modulo puede convertirse en microservicio

### Patron de Comunicacion Interna

```
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   Identity    │   │ CV Management │   │ Job Matching  │
│    Module     │◄──│    Module     │──►│    Module     │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        │           ┌─────────┴─────────┐           │
        │           ▼                   ▼           │
        │   ┌───────────────┐   ┌───────────────┐   │
        │   │ Interview Sim │   │Company Portal │   │
        │   │    Module     │   │    Module     │   │
        │   └───────────────┘   └───────────────┘   │
        │                                           │
        └───────────────────┬───────────────────────┘
                            ▼
                   ┌───────────────┐
                   │   Analytics   │
                   │    Module     │
                   └───────────────┘
                            │
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
┌───────────────┐                       ┌───────────────┐
│  Event Bus    │                       │    Workers    │
│  (Internal)   │                       │  (AI/ML Jobs) │
└───────────────┘                       └───────────────┘
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Velocidad de desarrollo** | Un solo repositorio, un solo deploy, ciclos rapidos de iteracion |
| **Costo operacional bajo** | Una instancia de aplicacion, un cluster de base de datos |
| **Simplicidad de debugging** | Stack traces completos, logs centralizados |
| **Equipo pequeno eficiente** | No requiere expertise en sistemas distribuidos |
| **Refactoring seguro** | IDE puede rastrear todas las dependencias |
| **Testing integrado** | Tests de integracion simples entre modulos |
| **Time-to-market** | MVP entregable en 8-12 semanas |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **Escalamiento acoplado** | Workers de IA separados para cargas pesadas |
| **Riesgo de monolito acoplado** | Reglas estrictas de boundaries, linting arquitectonico |
| **Un solo punto de fallo** | Multiples replicas, health checks, auto-healing |
| **Deploy atomico** | Feature flags para releases graduales |
| **Stack tecnologico unico** | Aceptable para MVP, extraer modulos si se requiere otro stack |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Degradacion a Big Ball of Mud | Media | Alto | ArchUnit/similar para validar boundaries |
| Cuello de botella en BD | Baja | Medio | Read replicas, caching agresivo |
| Latencia en picos de IA | Media | Medio | Queue para procesamiento asincrono |
| Dificultad de escalar equipo | Baja | Bajo | Ownership por modulo, documentacion clara |

---

## Alternativas Consideradas

### Opcion A: Microservicios desde el inicio

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Identity │ │    CV    │ │ Matching │ │Interview │
│ Service  │ │ Service  │ │ Service  │ │ Service  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
     │            │            │            │
     └────────────┴────────────┴────────────┘
                       │
              ┌────────────────┐
              │  API Gateway   │
              │ + Service Mesh │
              └────────────────┘
```

**Pros:**
- Escalamiento independiente por servicio
- Equipos autonomos por servicio
- Flexibilidad tecnologica

**Contras:**
- Complejidad operacional prematura
- Costo de infraestructura 3-5x mayor
- Requiere DevOps dedicado
- Latencia de red entre servicios
- Debugging distribuido complejo
- Timeline de MVP se extiende a 16-20 semanas

**Veredicto:** Rechazado - Complejidad prematura para equipo y presupuesto actuales

### Opcion B: Serverless / FaaS

```
┌─────────────────────────────────────┐
│           AWS Lambda / GCP          │
│  ┌────────┐ ┌────────┐ ┌────────┐   │
│  │Func A  │ │Func B  │ │Func C  │   │
│  └────────┘ └────────┘ └────────┘   │
└─────────────────────────────────────┘
              │
      ┌───────────────┐
      │  API Gateway  │
      └───────────────┘
```

**Pros:**
- Pago por uso, ideal para trafico variable
- Sin gestion de servidores
- Escalamiento automatico

**Contras:**
- Cold starts afectan UX en simulaciones de entrevista
- Vendor lock-in significativo
- Dificil para workloads de IA con estado
- Costos impredecibles en escala
- Debugging mas complejo

**Veredicto:** Rechazado - Cold starts inaceptables para simulaciones en tiempo real

### Opcion C: Monolito Tradicional (sin modulos)

**Pros:**
- Maxima simplicidad inicial
- Desarrollo mas rapido en semana 1-4

**Contras:**
- Deuda tecnica acumulada rapidamente
- Refactoring costoso post-MVP
- Dificil de dividir en equipos
- No prepara para escala

**Veredicto:** Rechazado - Crea deuda tecnica inaceptable para producto con ambicion de escala

---

## Plan de Evolucion

### Fase 1: MVP (Semanas 1-12)
- Modular Monolith completo
- Un deploy, una base de datos
- Workers separados solo para procesamiento de IA pesado

### Fase 2: Crecimiento (Meses 4-12)
- Extraer `interview-sim` si requiere escalamiento independiente
- Agregar read replicas para reportes
- Cache distribuido (Redis)

### Fase 3: Escala Regional (Ano 2+)
- Evaluar extraccion de modulos criticos a microservicios
- Multi-region deployment
- Event-driven architecture para sincronizacion

```
Fase 1          Fase 2              Fase 3
────────────────────────────────────────────────►

┌──────────┐    ┌──────────┐       ┌──────────┐
│ Modular  │    │ Modular  │       │  Hybrid  │
│ Monolith │ => │ Monolith │  =>   │ Modular  │
│          │    │ + Cache  │       │ + Microservices │
│          │    │ + Workers│       │ Selectivos      │
└──────────┘    └──────────┘       └──────────┘
```

---

## Criterios de Exito

| Metrica | Objetivo MVP | Objetivo Ano 1 |
|---------|--------------|----------------|
| Time to deploy | < 10 min | < 5 min |
| Latencia P95 API | < 200ms | < 150ms |
| Latencia simulacion IA | < 2s | < 1s |
| Disponibilidad | 99% | 99.5% |
| Costo infra mensual | < $500 | < $2000 |

---

## Decision Final

**Modular Monolith** es la arquitectura optima para Entrevistador Inteligente Peru porque:

1. **Alineado con recursos**: Equipo pequeno, presupuesto limitado
2. **Optimizado para velocidad**: MVP en 8-12 semanas es alcanzable
3. **Preparado para escala**: Boundaries claros permiten extraccion futura
4. **Bajo riesgo tecnico**: Patron probado, tooling maduro
5. **Soporta IA/ML**: Workers dedicados manejan cargas pesadas

---

## Referencias

- [MonolithFirst - Martin Fowler](https://martinfowler.com/bliki/MonolithFirst.html)
- [Modular Monolith: A Primer - Kamil Grzybek](https://www.kamilgrzybek.com/design/modular-monolith-primer/)
- [Strategic Monolith to Microservices - Sam Newman](https://samnewman.io/books/monolith-to-microservices/)
- [Shopify: Deconstructing the Monolith](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity)
- [DHH: The Majestic Monolith](https://signalvnoise.com/svn3/the-majestic-monolith/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
