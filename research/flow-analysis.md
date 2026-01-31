# Entrevistador Inteligente - Analisis Completo de Flujos

## Tabla de Contenidos
1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [User Journey - Candidato](#user-journey---candidato)
3. [User Journey - Empresa (B2B)](#user-journey---empresa-b2b)
4. [Flujo de Datos](#flujo-de-datos)
5. [Flujo de Monetizacion](#flujo-de-monetizacion)
6. [Dependencias Criticas](#dependencias-criticas)
7. [Cuellos de Botella Potenciales](#cuellos-de-botella-potenciales)
8. [Diagramas de Arquitectura](#diagramas-de-arquitectura)
9. [Recomendaciones de Optimizacion](#recomendaciones-de-optimizacion)

---

## Resumen Ejecutivo

El Entrevistador Inteligente es una plataforma de IA que conecta candidatos con oportunidades laborales mediante:
- Parsing inteligente de CVs
- Matching algoritmico con ofertas de empleo
- Simulacion de entrevistas con IA
- Feedback personalizado para mejora continua

### Metricas Clave del Analisis

| Metrica | Valor |
|---------|-------|
| Flujos de Usuario (Candidato) | 6 etapas |
| Flujos de Usuario (Empresa) | 5 etapas |
| Flujos de Datos Principales | 4 |
| Revenue Streams Identificados | 5 |
| Dependencias Criticas | 5 |
| Cuellos de Botella Potenciales | 8 |

### Contexto de Mercado Peru

| Indicador | Valor |
|-----------|-------|
| Poblacion Economicamente Activa | 18.5M |
| Tasa de Desempleo Urbano | 7.2% |
| Subempleo | 42% |
| Busquedas mensuales "trabajo" en Google | 1.2M |
| Penetracion smartphones | 78% |
| Usuarios LinkedIn Peru | 7.5M |

---

## User Journey - Candidato

### AWARENESS -> CONSIDERATION -> ACTIVATION -> ENGAGEMENT -> MONETIZATION -> RETENTION

### Etapa 1: AWARENESS (Conocimiento)

**Objetivo:** Candidato descubre la plataforma

**Acciones del Usuario:**
- Busca "como mejorar mi CV" o "practica entrevistas" en Google
- Ve anuncio en redes sociales (Facebook, TikTok, Instagram)
- Recibe recomendacion de amigo o colega
- Encuentra la app en Play Store/App Store
- Ve contenido educativo (YouTube, blog)

**Puntos de Dolor:**
- No sabe que herramientas de IA existen para busqueda de empleo
- Desconfianza inicial hacia plataformas nuevas
- Saturacion de apps de empleo (Bumeran, Computrabajo, LinkedIn)
- Percepcion de "otra app mas que no funciona"

**Oportunidades de Valor:**
- Contenido educativo gratuito (tips de CV, preparacion entrevistas)
- Casos de exito verificables de usuarios peruanos
- Demo interactiva sin registro
- Integracion con WhatsApp para acceso facil

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| CAC | < S/15 | Costo de adquisicion de candidato |
| CTR Ads | > 2.5% | Click-through rate en publicidad |
| Brand Awareness | 15% | Reconocimiento en target (6 meses) |
| Organic Traffic | 30% | Porcentaje de trafico organico |

**Canales de Adquisicion:**
```
                    ┌─────────────────────────────────────────┐
                    │           CANDIDATO POTENCIAL           │
                    └────────────────────┬────────────────────┘
                                         │
        ┌────────────┬──────────┬───────┴───────┬──────────┬────────────┐
        ▼            ▼          ▼               ▼          ▼            ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Google  │ │  Redes  │ │Referidos│   │App Store│ │WhatsApp │ │Universi-│
   │ Search  │ │Sociales │ │ Word of │   │  Play   │ │ Viral   │ │  dades  │
   │  (35%)  │ │  (25%)  │ │ Mouth   │   │  Store  │ │  (10%)  │ │  (5%)   │
   │         │ │         │ │  (15%)  │   │  (10%)  │ │         │ │         │
   └─────────┘ └─────────┘ └─────────┘   └─────────┘ └─────────┘ └─────────┘
```

---

### Etapa 2: CONSIDERATION (Consideracion)

**Objetivo:** Candidato evalua si la plataforma le sirve

**Acciones del Usuario:**
- Explora landing page y features
- Lee testimonios y reviews
- Compara con alternativas (Bumeran, LinkedIn Premium)
- Busca reviews en Play Store
- Pregunta en grupos de WhatsApp/Facebook

**Puntos de Dolor:**
- "Sera seguro subir mi CV con mis datos personales?"
- "Vale la pena pagar cuando hay opciones gratis?"
- "Funciona realmente la IA o es marketing?"
- "Tendre que aprender algo complicado?"

**Oportunidades de Valor:**
- Free trial completo (3 entrevistas simuladas)
- Certificaciones de seguridad visibles (GDPR-like)
- Video demos de 60 segundos
- Comparativa transparente vs competencia
- Chat en vivo para resolver dudas

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Time on Site | > 3 min | Tiempo promedio en pagina |
| Bounce Rate | < 45% | Tasa de rebote |
| Feature Exploration | > 4 clicks | Interaccion con features |
| Trial Signup Rate | > 25% | Conversion a trial |

**Decision Matrix (Usuario):**
```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         MATRIZ DE DECISION CANDIDATO                          │
├────────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│ Factor             │ Peso            │ Evaluacion      │ Accion Requerida    │
├────────────────────┼─────────────────┼─────────────────┼─────────────────────┤
│ Precio             │ 30%             │ Comparar plans  │ Pricing claro       │
│ Facilidad de uso   │ 25%             │ Demo interactiva│ Onboarding simple   │
│ Resultados reales  │ 25%             │ Testimonios     │ Social proof        │
│ Seguridad datos    │ 15%             │ Certificaciones │ Trust badges        │
│ Soporte            │ 5%              │ Disponibilidad  │ Chat 24/7           │
└────────────────────┴─────────────────┴─────────────────┴─────────────────────┘
```

---

### Etapa 3: ACTIVATION (Activacion)

**Objetivo:** Candidato se registra y completa primera accion de valor

**Acciones del Usuario:**
1. Registro (email, Google, Facebook, Apple)
2. Verificacion (OTP WhatsApp preferido en Peru)
3. Upload de CV (PDF, Word, foto)
4. Completar perfil basico
5. Primera simulacion de entrevista o analisis de CV

**Puntos de Dolor:**
- Formularios largos de registro
- Friccion en verificacion
- CV en formato no compatible
- No entiende que hacer despues del registro
- Proceso toma mas de 5 minutos

**Oportunidades de Valor:**
- Registro con 2 clicks (Google/Facebook)
- Verificacion via WhatsApp (no SMS)
- OCR avanzado para cualquier formato de CV
- Wizard guiado de onboarding
- Primer valor en < 3 minutos (analisis CV instantaneo)

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Signup Completion | > 80% | Usuarios que completan registro |
| Time to First Value | < 5 min | Tiempo hasta primera entrevista |
| CV Upload Success | > 95% | CVs procesados exitosamente |
| Activation Rate | > 60% | Usuarios que completan 1 entrevista |

**Flujo de Activacion:**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  REGISTRO   │────▶│VERIFICACION │────▶│  UPLOAD CV  │────▶│  ANALISIS   │
│  (30 seg)   │     │   (20 seg)  │     │   (60 seg)  │     │   (30 seg)  │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                    │
                    ┌───────────────────────────────────────────────┘
                    ▼
           ┌─────────────────┐     ┌─────────────────┐
           │ PRIMERA SIMULA- │────▶│    FEEDBACK     │
           │ CION ENTREVISTA │     │  PERSONALIZADO  │
           │    (5 min)      │     │    (instant)    │
           └─────────────────┘     └─────────────────┘
                                            │
                    TIEMPO TOTAL: < 8 minutos hasta valor
```

---

### Etapa 4: ENGAGEMENT (Compromiso)

**Objetivo:** Candidato usa la plataforma regularmente

**Acciones del Usuario:**
- Practica entrevistas semanalmente
- Mejora CV basado en feedback
- Explora ofertas de empleo matcheadas
- Recibe y acepta preparacion para entrevistas reales
- Comparte progreso en redes

**Puntos de Dolor:**
- Feedback repetitivo o generico
- Ofertas no relevantes para su perfil
- No ve progreso tangible
- Olvida usar la app (no hay habito)
- Simulaciones no reflejan entrevistas reales

**Oportunidades de Valor:**
- Gamificacion (badges, niveles, streaks)
- Feedback cada vez mas especifico con IA
- Notificaciones inteligentes de ofertas relevantes
- Tracking de progreso visual (antes/despues)
- Entrevistas especializadas por industria/rol
- Comunidad de candidatos (peer support)

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| WAU/MAU | > 40% | Usuarios activos semanales vs mensuales |
| Sessions/Week | > 2 | Sesiones por semana por usuario |
| Interview Completion | > 85% | Entrevistas iniciadas vs completadas |
| Improvement Score | > 15% | Mejora en puntaje entre sesiones |
| NPS | > 50 | Net Promoter Score |

**Loop de Engagement:**
```
                         ┌────────────────────────────────────┐
                         │                                    │
                         ▼                                    │
            ┌─────────────────────────┐                       │
            │   TRIGGER EXTERNO       │                       │
            │ (oferta, recordatorio)  │                       │
            └───────────┬─────────────┘                       │
                        │                                     │
                        ▼                                     │
            ┌─────────────────────────┐                       │
            │      ACCION             │                       │
            │ (practica entrevista)   │                       │
            └───────────┬─────────────┘                       │
                        │                                     │
                        ▼                                     │
            ┌─────────────────────────┐                       │
            │    RECOMPENSA           │──────────────────────┘
            │ (feedback, badge,       │
            │  match con oferta)      │
            └───────────┬─────────────┘
                        │
                        ▼
            ┌─────────────────────────┐
            │    INVERSION           │
            │ (tiempo, datos,        │
            │  personalizacion)      │
            └─────────────────────────┘
```

---

### Etapa 5: MONETIZATION (Monetizacion)

**Objetivo:** Candidato se convierte en cliente pago

**Acciones del Usuario:**
- Agota trial gratuito
- Necesita prepararse para entrevista real importante
- Quiere acceso a ofertas premium
- Busca coaching personalizado
- Requiere certificacion de habilidades

**Puntos de Dolor:**
- "El precio es muy alto para un desempleado"
- "No se si valdra la pena pagar"
- "Que pasa si pago y no consigo trabajo?"
- "Prefiero LinkedIn Premium que ya conozco"

**Oportunidades de Valor:**
- Planes flexibles (semanal, mensual, anual)
- Garantia de satisfaccion / devolucion
- Pago en cuotas (tarjeta o Yape)
- Plan "paga cuando consigas trabajo"
- Descuentos para estudiantes/desempleados
- Paquetes empresariales (company sponsorship)

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Free-to-Paid | > 8% | Conversion de trial a pago |
| ARPU | S/35/mes | Revenue promedio por usuario |
| LTV | S/180 | Lifetime value por candidato |
| Payback Period | < 2 meses | Tiempo para recuperar CAC |
| Churn Rate | < 8%/mes | Tasa de abandono mensual |

**Pricing Tiers Propuestos:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PLANES CANDIDATO                                   │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│     GRATIS      │     BASICO      │       PRO       │       PREMIUM         │
├─────────────────┼─────────────────┼─────────────────┼───────────────────────┤
│ S/0             │ S/29/mes        │ S/59/mes        │ S/99/mes              │
│                 │ (S/290/ano)     │ (S/590/ano)     │ (S/990/ano)           │
├─────────────────┼─────────────────┼─────────────────┼───────────────────────┤
│ 3 entrevistas   │ 15 entrevistas  │ Ilimitadas      │ Ilimitadas            │
│ Analisis CV     │ Analisis CV     │ Analisis CV     │ Analisis CV           │
│ basico          │ detallado       │ avanzado        │ premium + coach IA    │
│                 │ 5 ofertas/dia   │ 20 ofertas/dia  │ Ofertas ilimitadas    │
│                 │                 │ Prep. empresa   │ Prep. empresa         │
│                 │                 │ especifica      │ + video feedback      │
│                 │                 │                 │ Prioridad soporte     │
│                 │                 │                 │ Certificaciones       │
└─────────────────┴─────────────────┴─────────────────┴───────────────────────┘
```

---

### Etapa 6: RETENTION (Retencion)

**Objetivo:** Candidato permanece activo incluso despues de encontrar empleo

**Acciones del Usuario:**
- Consigue empleo y pausa suscripcion
- Recomienda plataforma a amigos
- Vuelve cuando busca nuevo empleo
- Usa para negociar ascensos/salarios
- Se convierte en mentor de otros candidatos

**Puntos de Dolor:**
- "Ya consegui trabajo, para que sigo pagando?"
- "No hay razon para volver"
- "Olvide que tenia cuenta"

**Oportunidades de Valor:**
- Modo "empleado" (preparacion para reviews, ascensos)
- Referral program con recompensas reales
- Comunidad alumni activa
- Contenido de desarrollo profesional continuo
- Notificaciones de oportunidades "demasiado buenas"
- Creditos que no expiran

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Retention 30d | > 45% | Usuarios activos a 30 dias |
| Retention 90d | > 25% | Usuarios activos a 90 dias |
| Reactivation Rate | > 20% | Usuarios que vuelven despues de churn |
| Referral Rate | > 15% | Usuarios que refieren al menos 1 persona |
| NPS Post-Hire | > 60 | Satisfaccion despues de conseguir empleo |

---

## User Journey - Empresa (B2B)

### DISCOVERY -> EVALUATION -> ONBOARDING -> USAGE -> EXPANSION

### Etapa 1: DISCOVERY (Descubrimiento)

**Objetivo:** Empresa descubre la plataforma como herramienta de reclutamiento

**Actores:**
- HR Manager
- Talent Acquisition Lead
- CEO/Founder (en startups/pymes)
- Hiring Manager (areas especificas)

**Acciones del Usuario:**
- Busca "plataforma de reclutamiento Peru" o "pre-screening candidatos"
- Recibe outreach de sales team
- Ve caso de estudio de empresa similar
- Asiste a webinar o evento HR
- Referido por otra empresa

**Puntos de Dolor:**
- Alto costo de reclutamiento tradicional
- Muchos CVs, pocos candidatos calificados
- Tiempo excesivo en screening inicial
- Entrevistas ineficientes (candidatos no preparados)
- Rotacion alta por mal fit

**Oportunidades de Valor:**
- ROI calculator (ahorro vs metodo actual)
- Case studies de empresas peruanas
- Demo personalizada sin compromiso
- Integracion con ATS existentes
- Candidatos pre-evaluados y rankeados

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Lead Generation | 50/mes | Leads calificados B2B |
| Demo Request Rate | > 30% | Conversion de visita a demo |
| Sales Cycle | < 45 dias | Tiempo de cierre |
| CAC B2B | < S/500 | Costo de adquisicion empresa |

---

### Etapa 2: EVALUATION (Evaluacion)

**Objetivo:** Empresa evalua si la plataforma cumple sus necesidades

**Acciones del Usuario:**
- Participa en demo personalizada
- Solicita piloto gratuito
- Compara con competencia (LinkedIn, Bumeran Premium)
- Evalua integraciones con sistemas actuales
- Consulta con equipo legal sobre datos

**Puntos de Dolor:**
- "Como se integra con nuestro ATS?"
- "Los candidatos de nuestra industria estaran en la plataforma?"
- "Cumple con regulaciones de proteccion de datos?"
- "Cual es el ROI real demostrable?"

**Oportunidades de Valor:**
- Piloto de 30 dias con metricas claras
- Integraciones pre-construidas (principales ATS)
- Compliance documentado (GDPR, Ley Peru)
- Dashboard de metricas en tiempo real
- Soporte dedicado durante piloto

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Pilot Conversion | > 50% | Pilotos que se convierten en clientes |
| Time in Evaluation | < 30 dias | Duracion de evaluacion |
| Stakeholder Approval | > 3 | Personas que aprueban internamente |
| Technical Validation | 100% | Integraciones funcionando |

---

### Etapa 3: ONBOARDING (Incorporacion)

**Objetivo:** Empresa configura y empieza a usar la plataforma

**Acciones del Usuario:**
- Firma contrato y activa cuenta
- Configura roles y permisos de equipo
- Integra con ATS/HRIS
- Carga/define perfiles de puesto
- Entrena a equipo de HR
- Publica primeras posiciones

**Puntos de Dolor:**
- Proceso de setup muy largo
- Equipo no sabe usar la herramienta
- Integraciones fallan
- Primeros resultados tardan en llegar

**Oportunidades de Valor:**
- Setup asistido en < 1 dia
- Biblioteca de perfiles de puesto pre-definidos
- Entrenamiento in-app interactivo
- Customer Success Manager dedicado
- Quick wins en primera semana

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Time to First Hire | < 21 dias | Primera contratacion via plataforma |
| Setup Completion | > 95% | Configuracion completa |
| Team Adoption | > 80% | Usuarios activos del equipo |
| First Value | < 7 dias | Primer shortlist generado |

**Flujo de Onboarding B2B:**
```
    Dia 1               Dia 2-3             Dia 4-5             Dia 6-7
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   KICKOFF   │───▶│   CONFIG    │───▶│ INTEGRACION │───▶│ENTRENAMIENTO│
│   MEETING   │    │   CUENTA    │    │    ATS      │    │   EQUIPO    │
└─────────────┘    └─────────────┘    └─────────────┘    └──────┬──────┘
                                                                │
    Dia 8-14            Dia 15-21           Dia 22-30          │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  PRIMERA    │───▶│  PRIMERA    │───▶│   REVIEW    │◀─────────┘
│  POSICION   │    │ CONTRATACION│    │  RESULTADOS │
└─────────────┘    └─────────────┘    └─────────────┘
```

---

### Etapa 4: USAGE (Uso)

**Objetivo:** Empresa usa la plataforma como herramienta principal de reclutamiento

**Acciones del Usuario:**
- Publica posiciones regularmente
- Revisa candidatos pre-calificados
- Programa entrevistas via plataforma
- Mide metricas de hiring funnel
- Solicita candidatos para roles especificos

**Puntos de Dolor:**
- Calidad de candidatos inconsistente
- Reportes no se alinean con necesidades
- Soporte lento para resolver issues
- Features que no usan (complejidad)

**Oportunidades de Valor:**
- Algoritmo que mejora con uso
- Reportes customizables
- Soporte prioritario por tier
- Features "just in time" (mostrar cuando relevante)
- Benchmarks de industria

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Positions/Month | > 5 | Posiciones publicadas por mes |
| Time to Fill | -30% | Reduccion vs baseline |
| Quality of Hire | > 4.2/5 | Rating de managers |
| Platform NPS | > 50 | Satisfaccion de HR |
| Feature Adoption | > 60% | Features utilizados |

---

### Etapa 5: EXPANSION (Expansion)

**Objetivo:** Empresa aumenta uso y se convierte en promotor

**Acciones del Usuario:**
- Agrega mas usuarios/departamentos
- Upgrade a plan superior
- Usa features avanzados (analytics, AI interview)
- Recomienda a otras empresas
- Participa en case studies

**Puntos de Dolor:**
- Limitaciones del plan actual
- Necesidades especificas no cubiertas
- Justificar expansion a directivos

**Oportunidades de Valor:**
- Upsell basado en uso real
- ROI dashboard para justificar expansion
- Beta access a nuevos features
- Partnership/referral benefits
- Co-marketing opportunities

**Metricas Clave:**
| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Net Revenue Retention | > 110% | Expansion vs churn |
| Upsell Rate | > 25% | Clientes que upgraden |
| Referral Rate | > 20% | Clientes que refieren |
| Logo Retention | > 90% | Clientes que renuevan |

**Pricing Tiers B2B:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PLANES EMPRESA                                     │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│     STARTUP     │      GROWTH     │   ENTERPRISE    │       CUSTOM          │
├─────────────────┼─────────────────┼─────────────────┼───────────────────────┤
│ S/299/mes       │ S/799/mes       │ S/1,999/mes     │ Negociable            │
├─────────────────┼─────────────────┼─────────────────┼───────────────────────┤
│ 3 usuarios      │ 10 usuarios     │ 25 usuarios     │ Ilimitados            │
│ 5 posiciones    │ 20 posiciones   │ Ilimitadas      │ Ilimitadas            │
│ Matching basico │ Matching IA     │ Matching IA+    │ Matching custom       │
│ Email support   │ Chat support    │ Dedicated CSM   │ Dedicated team        │
│                 │ Basic analytics │ Full analytics  │ Custom analytics      │
│                 │ ATS integration │ All integrations│ API access            │
│                 │                 │ White label opt │ SLA garantizado       │
└─────────────────┴─────────────────┴─────────────────┴───────────────────────┘
```

---

## Flujo de Datos

### Flujo Principal: CV Upload -> Matching -> Interview -> Feedback

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          FLUJO DE DATOS PRINCIPAL                            │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         1. CV UPLOAD & PARSING                           │
    └─────────────────────────────────────────────────────────────────────────┘

    [Usuario sube CV]          [Formato detectado]        [Parsing/OCR]
          │                           │                        │
          ▼                           ▼                        ▼
    ┌───────────┐              ┌───────────┐            ┌───────────┐
    │  PDF/     │──────────────│  Format   │────────────│   OCR     │
    │  Word/    │              │  Detector │            │  Engine   │
    │  Imagen   │              │           │            │ (Tesseract│
    │           │              │           │            │  /Azure)  │
    └───────────┘              └───────────┘            └─────┬─────┘
                                                              │
                                                              ▼
                                                       ┌───────────┐
                                                       │   NLP     │
                                                       │ Extraction│
                                                       │ (spaCy/   │
                                                       │  BERT)    │
                                                       └─────┬─────┘
                                                              │
              ┌────────────────────────────────────────────────┤
              │                                               │
              ▼                                               ▼
    ┌───────────────────────┐                    ┌───────────────────────┐
    │   DATOS EXTRAIDOS     │                    │   DATOS INFERIDOS     │
    ├───────────────────────┤                    ├───────────────────────┤
    │ - Nombre              │                    │ - Nivel experiencia   │
    │ - Contacto            │                    │ - Skills implicitos   │
    │ - Educacion           │                    │ - Fit cultural        │
    │ - Experiencia         │                    │ - Gaps detectados     │
    │ - Skills explicitos   │                    │ - Red flags           │
    │ - Idiomas             │                    │ - Fortalezas          │
    │ - Certificaciones     │                    │ - Areas de mejora     │
    └───────────────────────┘                    └───────────────────────┘
                    │                                        │
                    └───────────────────┬────────────────────┘
                                        │
                                        ▼
                              ┌───────────────────┐
                              │  PERFIL UNIFICADO │
                              │  (Vector Store)   │
                              └─────────┬─────────┘
                                        │
    ┌───────────────────────────────────┼───────────────────────────────────┐
    │                                   │                                   │
    │                                   │                                   │
    ▼                                   ▼                                   ▼

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         2. MATCHING ALGORITHM                            │
    └─────────────────────────────────────────────────────────────────────────┘

    ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
    │   PERFIL          │    │   BASE OFERTAS    │    │   PREFERENCIAS    │
    │   CANDIDATO       │    │   LABORALES       │    │   CANDIDATO       │
    │   (vector)        │    │   (vectors)       │    │                   │
    └─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
              │                        │                        │
              └────────────────────────┼────────────────────────┘
                                       │
                                       ▼
                           ┌───────────────────────┐
                           │   MATCHING ENGINE     │
                           ├───────────────────────┤
                           │ - Semantic similarity │
                           │ - Skill gap analysis  │
                           │ - Experience level    │
                           │ - Location/remote     │
                           │ - Salary expectations │
                           │ - Cultural fit score  │
                           └───────────┬───────────┘
                                       │
                                       ▼
                           ┌───────────────────────┐
                           │   RANKED MATCHES      │
                           │   (Top 20 offers)     │
                           │   + Fit Score 0-100   │
                           │   + Gap Analysis      │
                           └───────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      3. INTERVIEW SIMULATION                             │
    └─────────────────────────────────────────────────────────────────────────┘

    [Usuario inicia entrevista]
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                    CONFIGURACION                               │
    ├───────────────────────────────────────────────────────────────┤
    │ - Puesto objetivo (general o especifico)                      │
    │ - Empresa objetivo (opcional)                                  │
    │ - Tipo: Tecnica / Conductual / Case Study / HR               │
    │ - Nivel: Junior / Mid / Senior / Executive                    │
    │ - Idioma: Espanol / Ingles / Mixto                           │
    │ - Duracion: 10 / 20 / 30 / 45 minutos                        │
    └───────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                 GENERACION DE PREGUNTAS                        │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
    │   │   LLM       │   │  Banco de   │   │  Contexto   │        │
    │   │  (GPT-4/    │ + │  Preguntas  │ + │  Candidato  │        │
    │   │  Claude)    │   │  Reales     │   │  (CV+hist)  │        │
    │   └─────────────┘   └─────────────┘   └─────────────┘        │
    │                                                               │
    │   → Preguntas personalizadas al perfil                       │
    │   → Preguntas tipicas de la empresa objetivo                 │
    │   → Follow-ups dinamicos basados en respuestas               │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                    SESION DE ENTREVISTA                        │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │   [Pregunta IA]  →  [Respuesta Usuario]  →  [Analisis]       │
    │         │                    │                    │           │
    │         │                    │                    │           │
    │         │              ┌─────┴─────┐        ┌────┴────┐      │
    │         │              │   Audio   │        │ Tiempo  │      │
    │         │              │   Video   │        │ Fluency │      │
    │         │              │   Texto   │        │ Content │      │
    │         │              └─────┬─────┘        └────┬────┘      │
    │         │                    │                   │           │
    │         └─────── [Follow-up dinamico] ◀─────────┴───────────│
    │                                                               │
    │   Repetir 5-15 preguntas segun duracion                      │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         4. FEEDBACK & IMPROVEMENT                        │
    └─────────────────────────────────────────────────────────────────────────┘

    ┌───────────────────────────────────────────────────────────────┐
    │                    ANALISIS MULTI-DIMENSIONAL                  │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
    │  │   CONTENIDO  │  │  COMUNICACION│  │   TECNICO    │        │
    │  ├──────────────┤  ├──────────────┤  ├──────────────┤        │
    │  │ - Relevancia │  │ - Claridad   │  │ - Precision  │        │
    │  │ - Estructura │  │ - Concision  │  │ - Profundidad│        │
    │  │ - Ejemplos   │  │ - Confianza  │  │ - Actualidad │        │
    │  │ - STAR method│  │ - Tono       │  │ - Aplicacion │        │
    │  └──────────────┘  └──────────────┘  └──────────────┘        │
    │                                                               │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
    │  │  LENGUAJE    │  │   PRESENCIA  │  │   CULTURAL   │        │
    │  │   CORPORAL   │  │    EJECUTIVA │  │     FIT      │        │
    │  ├──────────────┤  ├──────────────┤  ├──────────────┤        │
    │  │ - Eye contact│  │ - Liderazgo  │  │ - Valores    │        │
    │  │ - Postura    │  │ - Impacto    │  │ - Trabajo eq │        │
    │  │ - Gestos     │  │ - Profesional│  │ - Adaptabil. │        │
    │  │ - Nerviosismo│  │              │  │              │        │
    │  └──────────────┘  └──────────────┘  └──────────────┘        │
    │                                                               │
    └────────────────────────────┬──────────────────────────────────┘
                                 │
                                 ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                      REPORTE FINAL                             │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │   SCORE GLOBAL: 78/100  [========      ] BUENO                │
    │                                                               │
    │   Por Dimension:                                              │
    │   ├── Contenido:      82/100  ████████░░                     │
    │   ├── Comunicacion:   75/100  ███████░░░                     │
    │   ├── Tecnico:        80/100  ████████░░                     │
    │   ├── Presencia:      70/100  ███████░░░                     │
    │   └── Cultural Fit:   85/100  ████████░░                     │
    │                                                               │
    │   Top 3 Fortalezas:                                          │
    │   1. Ejemplos concretos y cuantificables                     │
    │   2. Dominio tecnico del area                                │
    │   3. Alineacion con valores de la empresa                    │
    │                                                               │
    │   Top 3 Areas de Mejora:                                     │
    │   1. Reducir uso de muletillas ("este", "o sea")            │
    │   2. Estructurar mejor respuestas (usar STAR)               │
    │   3. Mejorar contacto visual con camara                      │
    │                                                               │
    │   Recursos Recomendados:                                     │
    │   - Video: "Como eliminar muletillas"                        │
    │   - Practica: Ejercicio STAR (10 min)                        │
    │   - Articulo: "Tips para video-entrevistas"                  │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                   LOOP DE MEJORA CONTINUA                      │
    │                                                               │
    │   [Feedback] → [Recursos] → [Practica] → [Re-evaluacion]    │
    │       ↑                                          │           │
    │       └──────────────────────────────────────────┘           │
    │                                                               │
    │   Tracking de progreso entre sesiones                        │
    │   Comparativa: Sesion 1 vs Sesion N                         │
    │   Alertas cuando score mejora significativamente             │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
```

### Flujo Secundario: Empresa -> Candidato Matching

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FLUJO B2B: EMPRESA → CANDIDATO                           │
└─────────────────────────────────────────────────────────────────────────────┘

    [Empresa crea posicion]
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                    JOB DESCRIPTION PARSING                     │
    ├───────────────────────────────────────────────────────────────┤
    │ Input: JD en texto libre o estructurado                       │
    │                                                               │
    │ Output:                                                       │
    │ - Skills requeridos (hard + soft)                            │
    │ - Nivel de experiencia                                        │
    │ - Educacion minima                                           │
    │ - Rango salarial                                             │
    │ - Ubicacion/remoto                                           │
    │ - Cultura/valores de empresa                                 │
    │ - Deal breakers                                              │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                    CANDIDATE SOURCING                          │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
    │   │ Candidatos  │   │ Candidatos  │   │ Candidatos  │        │
    │   │  Activos    │ + │  Pasivos    │ + │  Externos   │        │
    │   │ (buscando)  │   │ (no buscan) │   │ (referidos) │        │
    │   └─────────────┘   └─────────────┘   └─────────────┘        │
    │                                                               │
    │   Filtros:                                                   │
    │   - Match score > umbral (ej: 70%)                           │
    │   - Disponibilidad temporal                                  │
    │   - Expectativa salarial compatible                          │
    │   - Sin red flags                                            │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                    PRE-SCREENING AUTOMATICO                    │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │   Candidatos rankeados reciben invitacion a:                 │
    │                                                               │
    │   1. Completar perfil (si incompleto)                        │
    │   2. Tomar entrevista simulada especifica para el rol        │
    │   3. Responder preguntas knockout de la empresa              │
    │                                                               │
    │   Resultado:                                                  │
    │   - Score de entrevista simulada                             │
    │   - Respuestas a knockout questions                          │
    │   - Video clips de mejores respuestas                        │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌───────────────────────────────────────────────────────────────┐
    │                    SHORTLIST PARA EMPRESA                      │
    ├───────────────────────────────────────────────────────────────┤
    │                                                               │
    │   Top 10-20 candidatos con:                                  │
    │   - Ranking por fit score                                    │
    │   - Resumen de perfil                                        │
    │   - Score de entrevista simulada                             │
    │   - Video highlights (opcional)                              │
    │   - Gap analysis vs JD                                       │
    │   - Risk/red flag indicators                                 │
    │                                                               │
    │   Empresa puede:                                             │
    │   - Ver perfil completo                                      │
    │   - Solicitar mas informacion                                │
    │   - Invitar a entrevista real                                │
    │   - Rechazar con feedback                                    │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
```

---

## Flujo de Monetizacion

### Revenue Streams Analysis

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       MODELO DE MONETIZACION                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  1. FREEMIUM (Candidatos)                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Gratis:                           Pago:                                    │
│  - 3 entrevistas simuladas         - Entrevistas ilimitadas                │
│  - Analisis CV basico              - Analisis CV avanzado                  │
│  - Ofertas limitadas               - Ofertas personalizadas ilimitadas     │
│                                    - Prep especifica por empresa           │
│                                    - Video feedback                        │
│                                                                             │
│  Conversion Target: 8% free → paid                                         │
│  ARPU: S/35-45/mes                                                         │
│  LTV: S/180-250                                                            │
│                                                                             │
│  Proyeccion Y1: 50,000 usuarios, 4,000 pagos = S/1.7M ARR                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  2. SUBSCRIPTION B2B (Empresas)                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Startup:    S/299/mes   →  S/3,588/ano                                    │
│  Growth:     S/799/mes   →  S/9,588/ano                                    │
│  Enterprise: S/1,999/mes →  S/23,988/ano                                   │
│  Custom:     Negociable  →  S/50,000+/ano                                  │
│                                                                             │
│  Target Y1: 50 empresas                                                     │
│  Mix: 25 Startup, 15 Growth, 8 Enterprise, 2 Custom                        │
│  ARR Y1: S/750K                                                            │
│                                                                             │
│  Upsell path:                                                              │
│  - Mas usuarios                                                            │
│  - Mas posiciones                                                          │
│  - Features avanzados (AI interview, analytics)                            │
│  - Integraciones custom                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  3. PAY-PER-USE (Entrevistas Simuladas)                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Para candidatos sin suscripcion:                                          │
│  - Entrevista basica (10 min): S/9.90                                      │
│  - Entrevista completa (30 min): S/19.90                                   │
│  - Pack 5 entrevistas: S/69.90                                             │
│  - Pack 10 entrevistas: S/119.90                                           │
│                                                                             │
│  Para empresas (pre-screening):                                            │
│  - Por candidato evaluado: S/15-25                                         │
│  - Bulk (100+ candidatos): S/10-15                                         │
│                                                                             │
│  Proyeccion Y1: 10,000 transacciones = S/150K                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  4. MARKETPLACE (Comision por Contratacion)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Modelo success fee:                                                        │
│  - Comision: 8-12% del salario anual                                       │
│  - Solo se cobra si hay contratacion                                       │
│  - Garantia: reemplazo gratis si renuncia en 90 dias                       │
│                                                                             │
│  Ejemplo:                                                                   │
│  - Salario candidato: S/60,000/ano                                         │
│  - Comision (10%): S/6,000                                                 │
│                                                                             │
│  Target Y1: 100 contrataciones via plataforma                              │
│  Revenue Y1: S/500K (promedio S/5K/contratacion)                           │
│                                                                             │
│  Riesgo: Desintermediacion (empresa contacta candidato fuera)             │
│  Mitigacion: Valor agregado en proceso, contratos claros                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  5. B2B PREMIUM SERVICES                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Servicios adicionales:                                                     │
│                                                                             │
│  - Entrevistas custom con IA de empresa: S/5,000-15,000 setup              │
│  - White label de plataforma: S/50,000+ /ano                               │
│  - API access para integraciones: S/10,000-30,000/ano                      │
│  - Reportes de mercado laboral: S/2,000-5,000/reporte                      │
│  - Training corporativo (soft skills): S/3,000-10,000/sesion               │
│                                                                             │
│  Target Y1: 10 servicios premium = S/150K                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESUMEN PROYECCION REVENUE Y1                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Freemium (B2C):           S/1,700,000   (51%)                          │
│  2. Subscriptions (B2B):      S/  750,000   (23%)                          │
│  3. Pay-per-use:              S/  150,000   ( 5%)                          │
│  4. Marketplace fees:         S/  500,000   (15%)                          │
│  5. Premium services:         S/  150,000   ( 5%)                          │
│  ─────────────────────────────────────────────                             │
│  TOTAL Y1:                    S/3,250,000   (~$850K USD)                   │
│                                                                             │
│  Proyeccion Y3:               S/15,000,000  (~$4M USD)                     │
│  Proyeccion Y5:               S/50,000,000  (~$13M USD)                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Unit Economics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         UNIT ECONOMICS B2C                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CAC (Costo Adquisicion Candidato):                                        │
│  ├── Paid ads:        S/12                                                 │
│  ├── Organic:         S/3                                                  │
│  ├── Referral:        S/5                                                  │
│  └── Blended:         S/8                                                  │
│                                                                             │
│  LTV (Lifetime Value):                                                      │
│  ├── ARPU mensual:    S/38                                                 │
│  ├── Avg lifetime:    5 meses                                              │
│  ├── Gross margin:    85%                                                  │
│  └── LTV:             S/161                                                │
│                                                                             │
│  LTV/CAC Ratio:       20x (excelente, target >3x)                         │
│  Payback Period:      < 1 mes                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         UNIT ECONOMICS B2B                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CAC (Costo Adquisicion Empresa):                                          │
│  ├── Sales team:      S/300                                                │
│  ├── Marketing:       S/150                                                │
│  └── Blended:         S/450                                                │
│                                                                             │
│  LTV (Lifetime Value):                                                      │
│  ├── ACV promedio:    S/8,000                                              │
│  ├── Avg lifetime:    2.5 anos                                             │
│  ├── Gross margin:    80%                                                  │
│  └── LTV:             S/16,000                                             │
│                                                                             │
│  LTV/CAC Ratio:       35x (excelente)                                      │
│  Payback Period:      < 1 mes                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Dependencias Criticas

### DC-001: APIs de Parsing de CV

| Atributo | Detalle |
|----------|---------|
| **Criticidad** | MUY ALTA |
| **Descripcion** | Servicios de OCR y NLP para extraer datos de CVs |
| **Opciones Principales** | Azure Form Recognizer, Google Document AI, Amazon Textract, Affinda |
| **Fallback** | Parser propio con Tesseract + spaCy |
| **SLA Requerido** | 99.9% uptime, < 5 seg procesamiento |
| **Costo Estimado** | $0.01-0.05 por documento |
| **Riesgo** | Cambios de pricing, deprecacion de APIs |

**Mitigacion:**
- Multi-provider strategy (primary + backup)
- Cache de resultados de parsing
- Parser propio como fallback
- Contratos con SLAs garantizados

---

### DC-002: Modelos de IA para Matching

| Atributo | Detalle |
|----------|---------|
| **Criticidad** | ALTA |
| **Descripcion** | Embeddings y similarity search para matching candidato-oferta |
| **Opciones Principales** | OpenAI Embeddings, Cohere, Sentence-BERT |
| **Vector Store** | Pinecone, Weaviate, Qdrant, pgvector |
| **SLA Requerido** | 99.9% uptime, < 500ms query |
| **Costo Estimado** | $0.0001 por embedding, $0.10-0.50 por 1K queries |
| **Riesgo** | Drift en calidad de embeddings, costos escalables |

**Mitigacion:**
- Evaluacion continua de calidad de matches
- A/B testing de providers
- Modelo propio fine-tuneado como backup
- Caching agresivo de embeddings

---

### DC-003: Modelos de IA para Simulacion de Entrevistas

| Atributo | Detalle |
|----------|---------|
| **Criticidad** | MUY ALTA |
| **Descripcion** | LLMs para generar preguntas y evaluar respuestas |
| **Opciones Principales** | GPT-4, Claude, Gemini, Llama (self-hosted) |
| **Especializacion** | Fine-tuning para contexto peruano/LATAM |
| **SLA Requerido** | 99.5% uptime, < 3 seg respuesta |
| **Costo Estimado** | $0.03-0.10 por entrevista de 30 min |
| **Riesgo** | Rate limits, cambios de pricing, hallucinations |

**Mitigacion:**
- Multi-model architecture (primary + fallback)
- Prompt engineering riguroso
- Human-in-the-loop para casos edge
- Modelo fine-tuneado propio a largo plazo
- Guardrails para contenido inapropiado

---

### DC-004: Infraestructura Cloud

| Atributo | Detalle |
|----------|---------|
| **Criticidad** | ALTA |
| **Descripcion** | Hosting, compute, storage, networking |
| **Provider Principal** | AWS / GCP / Azure |
| **Componentes Clave** | Kubernetes, PostgreSQL, Redis, S3 |
| **Region Primaria** | us-east-1 o sa-east-1 (Sao Paulo) |
| **SLA Requerido** | 99.9% uptime |
| **Costo Estimado** | $3,000-8,000/mes inicial |
| **Riesgo** | Outages regionales, costos no controlados |

**Mitigacion:**
- Multi-AZ deployment
- Disaster recovery plan
- Cost monitoring y alertas
- Reserved instances para costos predecibles

---

### DC-005: Base de Datos de Ofertas de Empleo

| Atributo | Detalle |
|----------|---------|
| **Criticidad** | ALTA |
| **Descripcion** | Fuente de ofertas laborales actualizadas |
| **Fuentes Principales** | LinkedIn Jobs API, Indeed API, Scraping de Bumeran/Computrabajo, Empresas directas |
| **Volumen Necesario** | 10,000+ ofertas activas Peru |
| **Frecuencia Update** | Cada 4-6 horas |
| **Costo Estimado** | $500-2,000/mes en APIs + infraestructura |
| **Riesgo** | ToS violations en scraping, APIs cerradas |

**Mitigacion:**
- Mix de fuentes (APIs oficiales + partnerships)
- Partnerships con job boards
- Base de datos propia de empresas directas
- Compliance con ToS de cada fuente

---

## Cuellos de Botella Potenciales

### CB-001: Latencia en Procesamiento de CV

| Atributo | Valor |
|----------|-------|
| **Severidad** | ALTA |
| **Probabilidad** | Media |
| **Impacto** | Abandono en onboarding, mala primera impresion |
| **Causa Raiz** | OCR lento, NLP complejo, formatos no estandar |

**Sintomas:**
- Procesamiento > 30 segundos
- Usuarios abandonan durante upload
- Errores en extraccion de datos

**Mitigacion:**
1. Procesamiento asincrono con notificacion
2. Preview instantaneo mientras procesa
3. Pre-procesamiento de formatos comunes
4. Cache de CVs similares ya procesados
5. CDN para uploads (edge processing)

**Metricas de Seguimiento:**
- p50 < 5 seg, p95 < 15 seg, p99 < 30 seg
- Error rate < 2%
- Abandono en upload < 10%

---

### CB-002: Calidad de Matching Inconsistente

| Atributo | Valor |
|----------|-------|
| **Severidad** | CRITICA |
| **Probabilidad** | Alta |
| **Impacto** | Ofertas irrelevantes, perdida de confianza |
| **Causa Raiz** | Embeddings genericos, falta de contexto Peru |

**Sintomas:**
- Usuarios reportan "ofertas no relevantes"
- Baja tasa de aplicacion a ofertas sugeridas
- Alto churn por insatisfaccion

**Mitigacion:**
1. Fine-tuning de embeddings con datos Peru
2. Feedback loop (thumbs up/down en ofertas)
3. Pesos personalizados por usuario
4. Factores explicitos (ubicacion, salario, remoto)
5. A/B testing continuo de algoritmos

**Metricas de Seguimiento:**
- Click-through rate en ofertas > 15%
- Aplicacion rate > 5%
- User feedback score > 4/5

---

### CB-003: Costo de LLM por Entrevista

| Atributo | Valor |
|----------|-------|
| **Severidad** | ALTA |
| **Probabilidad** | Alta |
| **Impacto** | Margenes negativos en plan basico |
| **Causa Raiz** | Uso intensivo de GPT-4/Claude para cada entrevista |

**Sintomas:**
- Costo por entrevista > precio del plan
- Margenes < 50% en B2C
- Imposibilidad de ofrecer free tier

**Mitigacion:**
1. Modelo tiered (GPT-3.5 para basico, GPT-4 para premium)
2. Cache de respuestas a preguntas frecuentes
3. Modelo propio fine-tuneado para preguntas estandar
4. Limite de tokens por sesion
5. Batch processing para feedback

**Analisis de Costos:**
```
Entrevista 30 min (15 preguntas):
- Input tokens: ~5,000 (preguntas + contexto)
- Output tokens: ~8,000 (preguntas + feedback)
- Total: ~13,000 tokens

Costo GPT-4: $0.03/1K input + $0.06/1K output = $0.63/entrevista
Costo GPT-3.5: $0.0005/1K input + $0.0015/1K output = $0.01/entrevista

Estrategia: GPT-3.5 para flujo, GPT-4 solo para evaluacion final
Costo optimizado: ~$0.15/entrevista
```

---

### CB-004: Escalamiento de Video/Audio Processing

| Atributo | Valor |
|----------|-------|
| **Severidad** | MEDIA |
| **Probabilidad** | Media |
| **Impacto** | Degradacion de servicio en horas pico |
| **Causa Raiz** | Analisis de video en tiempo real consume recursos |

**Sintomas:**
- Lag en video durante entrevista
- Feedback de video tardio
- Costos de compute disparados

**Mitigacion:**
1. Procesamiento de video post-sesion
2. Analisis de audio solo (no video) para tier basico
3. Edge processing donde sea posible
4. Compresion agresiva de video
5. Auto-scaling de workers

**Metricas de Seguimiento:**
- Latencia video < 500ms
- Processing time post-sesion < 5 min
- Costo compute/entrevista < $0.05

---

### CB-005: Onboarding Friction - WhatsApp Verification

| Atributo | Valor |
|----------|-------|
| **Severidad** | ALTA |
| **Probabilidad** | Media |
| **Impacto** | Alto abandono en registro |
| **Causa Raiz** | Usuarios no reciben OTP, cambian de contexto |

**Sintomas:**
- Tasa de verificacion < 70%
- Usuarios abandonan en paso de OTP
- Quejas de "no recibo codigo"

**Mitigacion:**
1. WhatsApp como primera opcion (mayor deliverability en Peru)
2. SMS como fallback inmediato
3. Email como tercera opcion
4. "Magic link" alternativo
5. Registro sin verificacion (verificar despues)

**Metricas de Seguimiento:**
- OTP delivery rate > 95%
- Verificacion completion > 85%
- Time to verify < 60 seg

---

### CB-006: Cold Start Problem - Nuevos Usuarios

| Atributo | Valor |
|----------|-------|
| **Severidad** | MEDIA |
| **Probabilidad** | Alta |
| **Impacto** | Ofertas no personalizadas, engagement bajo |
| **Causa Raiz** | Sin datos para matching en usuarios nuevos |

**Sintomas:**
- Ofertas genericas para nuevos usuarios
- Bajo engagement en primeras sesiones
- "Esta app no entiende lo que busco"

**Mitigacion:**
1. Onboarding wizard que capture preferencias
2. Inferir preferencias del CV
3. Ofertas "populares" como default
4. Quick survey de 3 preguntas
5. Aprender rapido de primeras interacciones

**Metricas de Seguimiento:**
- Time to personalized recommendations < 3 acciones
- Satisfaction nuevos usuarios > 3.5/5
- Retention Day 1 > 50%

---

### CB-007: Dependency en Job Boards Externos

| Atributo | Valor |
|----------|-------|
| **Severidad** | ALTA |
| **Probabilidad** | Media |
| **Impacto** | Sin ofertas = sin propuesta de valor |
| **Causa Raiz** | APIs de terceros pueden cambiar/cerrar |

**Sintomas:**
- Reduccion repentina de ofertas
- Ofertas desactualizadas
- Bloqueo por ToS violation

**Mitigacion:**
1. Diversificacion de fuentes (5+ fuentes activas)
2. Partnerships formales con job boards
3. Empresas publicando directamente en plataforma
4. Scraping etico con rate limiting
5. Base de datos propia creciente

**Metricas de Seguimiento:**
- Ofertas activas > 10,000 Peru
- Freshness < 24 horas
- Diversidad de fuentes > 5

---

### CB-008: Privacidad y Seguridad de Datos

| Atributo | Valor |
|----------|-------|
| **Severidad** | CRITICA |
| **Probabilidad** | Baja |
| **Impacto** | Breach = perdida de confianza total, multas |
| **Causa Raiz** | CVs contienen datos personales sensibles |

**Sintomas:**
- Data breach
- Usuarios preocupados por privacidad
- Reguladores investigando

**Mitigacion:**
1. Encryption at rest y in transit
2. Compliance con Ley 29733 (Peru) y GDPR
3. Data retention policies claras
4. Penetration testing regular
5. SOC 2 compliance a mediano plazo
6. Anonimizacion de datos para analytics

**Metricas de Seguimiento:**
- Zero breaches
- Compliance audit passed
- Security incidents < 1/trimestre

---

## Diagramas de Arquitectura

### Arquitectura de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA - ENTREVISTADOR INTELIGENTE                  │
└─────────────────────────────────────────────────────────────────────────────┘

                                   USUARIOS
                    ┌─────────────────────────────────────┐
                    │   Candidatos        Empresas        │
                    │   (Mobile/Web)      (Web/API)       │
                    └──────────┬───────────────┬──────────┘
                               │               │
                               ▼               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              CDN (CloudFront)                                 │
│                         Static Assets + Edge Caching                         │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY (Kong/AWS)                              │
│                    Rate Limiting, Auth, Routing                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
           ▼                           ▼                           ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│   AUTH SERVICE   │      │   CORE API       │      │  WEBHOOK SERVICE │
│                  │      │                  │      │                  │
│ - JWT/OAuth      │      │ - User mgmt      │      │ - Payment hooks  │
│ - Social login   │      │ - CV processing  │      │ - Job updates    │
│ - MFA            │      │ - Job matching   │      │ - Notifications  │
│                  │      │ - Analytics      │      │                  │
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           MESSAGE QUEUE (SQS/RabbitMQ)                        │
│                         Async Processing, Event-Driven                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
         ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  CV PROCESSOR    │      │  INTERVIEW       │      │  MATCHING        │
│  SERVICE         │      │  ENGINE          │      │  SERVICE         │
│                  │      │                  │      │                  │
│ - OCR (Azure)    │      │ - LLM orchestr.  │      │ - Embedding gen  │
│ - NLP extraction │      │ - Question gen   │      │ - Vector search  │
│ - Validation     │      │ - Response eval  │      │ - Ranking algo   │
│ - Enrichment     │      │ - Video analysis │      │ - Personalization│
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                       │
├──────────────────┬───────────────────┬───────────────────┬───────────────────┤
│   PostgreSQL     │    Redis          │   Pinecone/       │   S3              │
│   (Primary DB)   │    (Cache)        │   pgvector        │   (Files)         │
│                  │                   │   (Vectors)       │                   │
│ - Users          │ - Sessions        │ - CV embeddings   │ - CVs             │
│ - Companies      │ - Rate limits     │ - Job embeddings  │ - Videos          │
│ - Jobs           │ - Temp data       │ - User prefs      │ - Reports         │
│ - Interviews     │ - Hot data        │                   │ - Assets          │
│ - Analytics      │                   │                   │                   │
└──────────────────┴───────────────────┴───────────────────┴───────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL SERVICES                                   │
├──────────────────┬───────────────────┬───────────────────┬───────────────────┤
│   LLM APIs       │   Job APIs        │   Payment         │   Notifications   │
│                  │                   │                   │                   │
│ - OpenAI         │ - LinkedIn        │ - Stripe          │ - SendGrid        │
│ - Anthropic      │ - Indeed          │ - Yape/Plin       │ - Twilio          │
│ - Azure OpenAI   │ - Job boards      │ - MercadoPago     │ - Firebase        │
└──────────────────┴───────────────────┴───────────────────┴───────────────────┘
```

### Flujo de Entrevista - Arquitectura

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INTERVIEW ENGINE - ARQUITECTURA DETALLADA                 │
└─────────────────────────────────────────────────────────────────────────────┘

         Usuario                                              Backend
            │                                                    │
            │  1. Start Interview                                │
            │ ─────────────────────────────────────────────────► │
            │    {job_type, difficulty, duration}                │
            │                                                    │
            │                                         ┌──────────┴──────────┐
            │                                         │   INTERVIEW         │
            │                                         │   ORCHESTRATOR      │
            │                                         │                     │
            │                                         │ - Session mgmt      │
            │                                         │ - State machine     │
            │                                         │ - Flow control      │
            │                                         └──────────┬──────────┘
            │                                                    │
            │                                                    │ 2. Load context
            │                                                    ▼
            │                                         ┌──────────────────────┐
            │                                         │   CONTEXT BUILDER    │
            │                                         │                      │
            │                                         │ - CV data            │
            │                                         │ - Job description    │
            │                                         │ - Company culture    │
            │                                         │ - Past interviews    │
            │                                         └──────────┬───────────┘
            │                                                    │
            │                                                    │ 3. Generate Q
            │                                                    ▼
            │                                         ┌──────────────────────┐
            │                                         │   QUESTION           │
            │  4. Question (text/audio)               │   GENERATOR          │
            │ ◄───────────────────────────────────────│                      │
            │                                         │ - LLM (GPT-4)        │
            │                                         │ - Question bank      │
            │                                         │ - Personalization    │
            │                                         │ - Follow-up logic    │
            │                                         └──────────────────────┘
            │
            │  5. Answer (text/audio/video)
            │ ─────────────────────────────────────────────────►
            │
            │                                         ┌──────────────────────┐
            │                                         │   RESPONSE           │
            │                                         │   PROCESSOR          │
            │                                         │                      │
            │                                         │ - Speech-to-text     │
            │                                         │ - Sentiment analysis │
            │                                         │ - Content extraction │
            │                                         │ - Video analysis     │
            │                                         └──────────┬───────────┘
            │                                                    │
            │                                                    │ 6. Evaluate
            │                                                    ▼
            │                                         ┌──────────────────────┐
            │                                         │   EVALUATION         │
            │                                         │   ENGINE             │
            │                                         │                      │
            │                                         │ - Content scoring    │
            │                                         │ - Communication      │
            │                                         │ - Technical accuracy │
            │                                         │ - Soft skills        │
            │                                         └──────────┬───────────┘
            │                                                    │
            │                                                    │ Loop: Next Q
            │                                         ┌──────────┴───────────┐
            │                                         │                      │
            │  7. Next Question                       │   Until complete     │
            │ ◄───────────────────────────────────────│   (N questions)      │
            │                                         │                      │
            │  ...                                    │                      │
            │                                         └──────────────────────┘
            │
            │  8. Interview Complete
            │ ─────────────────────────────────────────────────►
            │
            │                                         ┌──────────────────────┐
            │                                         │   FEEDBACK           │
            │  9. Detailed Report                     │   GENERATOR          │
            │ ◄───────────────────────────────────────│                      │
            │    {scores, strengths, improvements,    │ - Aggregate scores   │
            │     resources, comparison}              │ - Identify patterns  │
            │                                         │ - Generate insights  │
            │                                         │ - Recommend actions  │
            │                                         └──────────────────────┘
```

---

## Recomendaciones de Optimizacion

### Prioridad 1: Criticas (Sprint 1-2)

#### OPT-001: CV Processing Pipeline Asincrono
**Problema:** Procesamiento de CV bloquea UX
**Solucion:**
```
- Upload instantaneo a S3
- Job queue para procesamiento
- Notificacion push cuando listo
- Preview parcial mientras procesa
```
**Impacto:** Time-to-value reducido 50%

#### OPT-002: Modelo LLM Tiered
**Problema:** Costos de GPT-4 insostenibles
**Solucion:**
```
- GPT-3.5 para preguntas estandar
- GPT-4 solo para evaluacion y feedback
- Cache de preguntas frecuentes
- Modelo fine-tuneado propio (Q3)
```
**Impacto:** Reduccion costos 70%, margen viable

#### OPT-003: Multi-Source Job Aggregation
**Problema:** Dependencia de fuentes unicas de empleo
**Solucion:**
```
- APIs oficiales (LinkedIn, Indeed)
- Partnerships con job boards locales
- Scraping etico con compliance
- Base de datos propia (empresas directas)
```
**Impacto:** 10,000+ ofertas activas, resiliencia

### Prioridad 2: Importantes (Sprint 3-4)

#### OPT-004: Feedback Loop de Matching
**Problema:** Algoritmo no mejora con uso
**Solucion:**
```
- Thumbs up/down en cada oferta
- Tracking de aplicaciones
- Implicit signals (tiempo en oferta, clicks)
- Retraining semanal del modelo
```
**Impacto:** Relevancia +25%, engagement +15%

#### OPT-005: Gamificacion y Streaks
**Problema:** Bajo engagement recurrente
**Solucion:**
```
- Daily streaks (practica diaria)
- Badges por logros
- Leaderboards (opcional)
- Progress tracking visual
```
**Impacto:** WAU/MAU +20%, retention +15%

#### OPT-006: Onboarding Personalizado por Industria
**Problema:** Experiencia generica no resuena
**Solucion:**
```
- Deteccion de industria del CV
- Templates de entrevista por sector
- Ofertas filtradas por industria
- Contenido educativo especializado
```
**Impacto:** Activation rate +15%, NPS +10

### Prioridad 3: Mejoras (Sprint 5+)

#### OPT-007: Video Analysis Completo
**Problema:** Solo texto, sin feedback de presencia
**Solucion:**
```
- Eye contact tracking
- Postura analysis
- Gesture recognition
- Emotion detection (confianza, nerviosismo)
```
**Impacto:** Diferenciador, premium feature

#### OPT-008: API Publica para Empresas
**Problema:** Integracion manual con ATS
**Solucion:**
```
- REST API documentada
- Webhooks para eventos
- SDKs (Python, JS)
- Sandbox environment
```
**Impacto:** Enterprise adoption, stickiness

#### OPT-009: Marketplace de Entrenadores
**Problema:** Candidatos quieren coaching humano
**Solucion:**
```
- Network de coaches certificados
- Booking integrado
- Revenue share (20%)
- Reviews y ratings
```
**Impacto:** Nuevo revenue stream, diferenciador

---

## Anexos

### A1: Competidores en Peru/LATAM

| Competidor | Tipo | Fortaleza | Debilidad | Precio |
|------------|------|-----------|-----------|--------|
| LinkedIn Premium | Job search | Brand, network | Caro, no local | $30/mes |
| Bumeran | Job board | Inventario Peru | Sin IA, basico | Gratis/empresa paga |
| Computrabajo | Job board | Popular Peru | UX anticuada | Gratis/empresa paga |
| Big Interview | Mock interview | UI, feedback | No LATAM, ingles | $79/mes |
| Pramp | Peer practice | Gratis, tech | Solo tech, peers | Gratis |
| Interview Warmup | Google AI | Gratis, Google | Basico, no matching | Gratis |

### A2: Metricas de Exito por Fase

| Fase | Timeline | Metricas Clave | Target |
|------|----------|----------------|--------|
| MVP | M1-M3 | Usuarios registrados | 5,000 |
| | | Entrevistas completadas | 10,000 |
| | | NPS | > 30 |
| Growth | M4-M9 | MAU | 25,000 |
| | | Paid users | 2,000 |
| | | ARR | S/500K |
| Scale | M10-M18 | MAU | 100,000 |
| | | Empresas activas | 50 |
| | | ARR | S/3M |

### A3: Roadmap de Features

| Q1 2026 | Q2 2026 | Q3 2026 | Q4 2026 |
|---------|---------|---------|---------|
| MVP Core | Matching v2 | B2B Platform | Video Analysis |
| CV Parser | Gamification | API Publica | Multi-country |
| Basic Interview | Industry Templates | Enterprise Features | Marketplace |
| Subscription | Mobile App | Integrations | AI Coach v2 |

---

## Control de Versiones

| Version | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-30 | Flow Analyst | Documento inicial completo |

---

*Documento generado como parte del analisis USACF - Entrevistador Inteligente Peru*
