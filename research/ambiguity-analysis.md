# USACF Ambiguity Clarification Analysis
## Entrevistador Inteligente Peru

**Fecha de Analisis:** 2026-01-30
**Version:** 1.0
**Metodologia:** USACF (Unified Specification and Ambiguity Clarification Framework)

---

## 1. RESUMEN EJECUTIVO

**Concepto de Negocio:**
Plataforma donde usuarios suben CV, encuentra puestos de trabajo adecuados, y simula entrevistas para preparacion profesional en el mercado peruano.

**Propuesta de Valor Central:**
Reducir la brecha entre candidatos y empleadores mediante IA que optimiza el matching laboral y prepara candidatos con simulaciones de entrevistas personalizadas.

---

## 2. MATRIZ DE TERMINOS AMBIGUOS

### 2.1 Subir CV (CV Upload)

| Interpretacion | Descripcion | Complejidad Tecnica | Experiencia Usuario |
|----------------|-------------|---------------------|---------------------|
| **A: Upload Manual** | PDF/DOCX upload tradicional | Baja | Familiar pero tedioso |
| **B: LinkedIn Import** | OAuth integration con LinkedIn | Media | Conveniente, datos estructurados |
| **C: Parser Automatico** | OCR + NLP para extraccion automatica | Alta | Magico pero requiere validacion |
| **D: Perfil Progresivo** | Wizard guiado + enriquecimiento gradual | Media | Engagement alto, completitud variable |

**Supuesto Provisional:**
- **Interpretacion Asumida:** B + C (LinkedIn import como opcion principal, parser como fallback)
- **Confianza:** 70%
- **Riesgo si esta mal:** MEDIO - Afecta onboarding y calidad de datos

---

### 2.2 Encontrar Puesto (Job Matching)

| Interpretacion | Descripcion | Modelo de Negocio | Diferenciacion |
|----------------|-------------|-------------------|----------------|
| **A: Matching AI** | Algoritmo propio de recomendacion | B2C/B2B SaaS | Alta si el matching es superior |
| **B: Agregador** | Scraping de Bumeran, Computrabajo, Indeed | Advertising/Freemium | Baja, commodity |
| **C: Headhunting** | Curated matching con empresas partner | B2B placement fees | Media, requiere relaciones |
| **D: Marketplace** | Empresas publican + candidatos aplican | Transaccional | Requiere masa critica |

**Supuesto Provisional:**
- **Interpretacion Asumida:** A + B (AI matching sobre data agregada inicialmente, transicion a marketplace)
- **Confianza:** 60%
- **Riesgo si esta mal:** ALTO - Define modelo de monetizacion completo

---

### 2.3 Simular Entrevistas (Interview Simulation)

| Interpretacion | Descripcion | Tecnologia | Valor Percibido |
|----------------|-------------|------------|-----------------|
| **A: Chatbot IA** | Conversacion texto con LLM | GPT-4/Claude API | Medio, escalable |
| **B: Video Mock** | Avatar AI + speech recognition | ElevenLabs + vision | Alto, inmersivo |
| **C: Preguntas Escritas** | Banco de preguntas + respuestas modelo | Base de datos | Bajo, commoditizado |
| **D: Roleplay Avanzado** | Simulacion con feedback multimodal | Multi-modal AI | Muy alto, diferenciador |

**Supuesto Provisional:**
- **Interpretacion Asumida:** A (MVP chatbot IA, roadmap hacia B y D)
- **Confianza:** 80%
- **Riesgo si esta mal:** MEDIO - Afecta diferenciacion y pricing

---

### 2.4 Mercado Peru (Geographic Scope)

| Interpretacion | TAM Estimado | Competencia | Regulacion |
|----------------|--------------|-------------|------------|
| **A: Lima Metro** | ~5M trabajadores | Alta concentracion | LPDP aplica |
| **B: Nacional** | ~17M PEA | Diversa, regional | Uniforme |
| **C: Peru + LATAM** | ~200M (region) | Players establecidos | Multijurisdiccional |
| **D: Hispanoamerica** | ~400M | Muy competido | Compleja |

**Supuesto Provisional:**
- **Interpretacion Asumida:** A inicial, expansion B en 18 meses
- **Confianza:** 85%
- **Riesgo si esta mal:** BAJO - Expansion es decision tactica

---

### 2.5 Modelo de Negocio (Revenue Model)

| Interpretacion | Revenue Stream | CAC | LTV Esperado |
|----------------|---------------|-----|--------------|
| **A: B2C Freemium** | Suscripciones individuales | $5-15 | $50-200/ano |
| **B: B2B Enterprise** | Licencias corporativas | $500-2K | $5K-50K/ano |
| **C: Marketplace** | Comision por placement | Variable | $200-500/colocacion |
| **D: Hibrido** | Freemium + B2B + comisiones | Mixto | Variable |

**Supuesto Provisional:**
- **Interpretacion Asumida:** D (Hibrido con enfasis inicial en B2C freemium)
- **Confianza:** 55%
- **Riesgo si esta mal:** ALTO - Define unit economics completos

---

## 3. SUPUESTOS PROVISIONALES CONSOLIDADOS

| ID | Supuesto | Interpretacion | Confianza | Riesgo | Validacion Requerida |
|----|----------|----------------|-----------|--------|---------------------|
| SP-01 | CV Upload | LinkedIn import + parser | 70% | Medio | User testing, conversion rates |
| SP-02 | Job Matching | AI matching sobre agregacion | 60% | Alto | Piloto con empresas, NPS |
| SP-03 | Entrevistas | Chatbot IA (MVP) | 80% | Medio | Prototype testing |
| SP-04 | Geografia | Lima Metro inicial | 85% | Bajo | Market sizing validation |
| SP-05 | Revenue | Hibrido freemium-first | 55% | Alto | Willingness-to-pay research |
| SP-06 | Target User | Mid-level professionals (25-35) | 65% | Medio | Persona research |
| SP-07 | Industrias | Tech + servicios financieros primero | 70% | Medio | Industry analysis |

---

## 4. PREGUNTAS CLAVE PARA VALIDACION

### 4.1 Segmento de Usuario

1. **Target Principal:**
   - Recien graduados (18-24, sin experiencia)?
   - Mid-level (25-35, 3-8 anos experiencia)?
   - Ejecutivos senior (35+, C-level)?
   - **Hipotesis:** Mid-level es el sweet spot (disposicion a pagar + volumen)

2. **Motivacion Primaria:**
   - Busqueda activa de empleo?
   - Mejora continua / preparacion pasiva?
   - Transicion de carrera?
   - **Hipotesis:** 60% busqueda activa, 40% preparacion

3. **Industrias Prioritarias:**
   - Tech / Startups?
   - Banca / Finanzas?
   - Retail / Consumo?
   - Mineria / Energia?
   - **Hipotesis:** Tech + Finanzas = early adopters, Retail = escala

### 4.2 Modelo de Monetizacion

4. **Quien Paga:**
   - Candidato individual?
   - Empresa empleadora?
   - Ambos (two-sided)?
   - **Hipotesis:** MVP B2C, escala via B2B

5. **Pricing Strategy:**
   - Freemium con simulaciones limitadas?
   - Suscripcion mensual/anual?
   - Pago por uso (pay-per-interview)?
   - **Hipotesis:** Freemium (3 simulaciones/mes) + Premium ($9.99/mes)

6. **B2B Offering:**
   - Hiring assessment tool?
   - Training platform para empleados?
   - Employer branding?
   - **Hipotesis:** Assessment tool con analytics

### 4.3 Producto - Entrevistas

7. **Especializacion:**
   - Genericas (soft skills)?
   - Por industria (ej: finanzas, tech)?
   - Por rol (ej: PM, developer, ventas)?
   - Por empresa especifica?
   - **Hipotesis:** Por rol + industria, empresa-especifica es premium

8. **Modalidad:**
   - Solo texto (chatbot)?
   - Audio + texto?
   - Video completo (avatar)?
   - **Hipotesis:** MVP texto, V2 audio, V3 video

9. **Feedback:**
   - Score numerico?
   - Analisis cualitativo detallado?
   - Comparacion con candidatos similares?
   - **Hipotesis:** Cualitativo + score + benchmarking anonimo

### 4.4 Ecosistema e Integraciones

10. **Fuentes de Empleos:**
    - Base propia (empresas publican)?
    - Agregacion (Bumeran, Computrabajo, LinkedIn)?
    - Partnership exclusivos?
    - **Hipotesis:** Agregacion MVP, partnerships fase 2

11. **Integraciones ATS:**
    - Greenhouse, Lever?
    - SAP SuccessFactors?
    - Sistemas locales?
    - **Hipotesis:** API generica, Greenhouse prioritario

12. **Verificacion:**
    - Verificacion de identidad?
    - Background checks?
    - Validacion de experiencia?
    - **Hipotesis:** Identidad basica, background via partners

---

## 5. MATRIZ DE RIESGO POR SUPUESTO

```
                    IMPACTO
                    Bajo    Medio    Alto
              +------+-------+-------+
        Alta  |      |       | SP-02 |
              |      |       | SP-05 |
   PROB.      +------+-------+-------+
   ERROR      |      | SP-01 |       |
        Media |      | SP-03 | SP-06 |
              |      | SP-07 |       |
              +------+-------+-------+
        Baja  | SP-04|       |       |
              +------+-------+-------+
```

**Prioridad de Validacion:**
1. SP-02 (Job Matching Model) - CRITICO
2. SP-05 (Revenue Model) - CRITICO
3. SP-06 (Target User) - ALTO
4. SP-01 (CV Upload) - MEDIO
5. SP-03 (Entrevistas) - MEDIO
6. SP-07 (Industrias) - MEDIO
7. SP-04 (Geografia) - BAJO

---

## 6. PLAN DE VALIDACION RECOMENDADO

### Fase 1: Discovery (2 semanas)

| Actividad | Objetivo | Supuestos a Validar | Metodo |
|-----------|----------|---------------------|--------|
| User Interviews | Entender pain points | SP-06, SP-02 | 15-20 entrevistas |
| Competitor Analysis | Mapear landscape | SP-02, SP-05 | Desk research |
| Willingness-to-Pay | Pricing validation | SP-05 | Encuesta + conjoint |

### Fase 2: Prototype Testing (3 semanas)

| Actividad | Objetivo | Supuestos a Validar | Metodo |
|-----------|----------|---------------------|--------|
| Landing Page Test | Validar propuesta | SP-06, SP-02 | Paid ads + conversion |
| Chatbot Prototype | Test interview UX | SP-03 | 50 usuarios beta |
| CV Parser Test | Evaluar precision | SP-01 | 100 CVs diversos |

### Fase 3: MVP Validation (4 semanas)

| Actividad | Objetivo | Supuestos a Validar | Metodo |
|-----------|----------|---------------------|--------|
| Beta Launch | Product-market fit | Todos | 500 usuarios Lima |
| B2B Pilots | Validar enterprise | SP-05, SP-02 | 5 empresas partner |
| Retention Analysis | Engagement patterns | SP-03, SP-06 | Cohort analysis |

---

## 7. DEFINICIONES OPERATIVAS PROPUESTAS

Para eliminar ambiguedad en desarrollo, se proponen las siguientes definiciones:

### 7.1 Usuario Activo
> Usuario que ha completado al menos 1 simulacion de entrevista en los ultimos 30 dias.

### 7.2 Match Exitoso
> Candidato que aplica a una posicion recomendada Y recibe respuesta del empleador (positiva o negativa).

### 7.3 Simulacion Completa
> Sesion de entrevista donde el usuario responde al menos 5 preguntas Y recibe feedback.

### 7.4 Perfil Completo
> CV con: nombre, email, telefono, experiencia (min 1), educacion (min 1), skills (min 3).

### 7.5 Premium User
> Usuario con suscripcion activa pagada (mensual o anual).

---

## 8. DEPENDENCIAS Y DECISIONES BLOQUEANTES

### Decisiones que Bloquean Desarrollo

| Prioridad | Decision | Bloqueado | Deadline Sugerido |
|-----------|----------|-----------|-------------------|
| P0 | Target user definition | Arquitectura, UX, marketing | Semana 1 |
| P0 | Revenue model confirmation | Pricing, features, roadmap | Semana 2 |
| P1 | Interview format (text/audio/video) | Tech stack, infra costs | Semana 3 |
| P1 | Job source strategy | Partnerships, legal, data | Semana 4 |
| P2 | B2B feature set | Product roadmap, sales | Semana 6 |

### Dependencias Tecnicas

| Componente | Depende De | Riesgo |
|------------|------------|--------|
| CV Parser | SP-01 confirmation | Medio |
| Matching Algorithm | SP-02, job source decision | Alto |
| Interview Engine | SP-03 format decision | Medio |
| Analytics Dashboard | SP-05 metrics definition | Bajo |

---

## 9. PROXIMOS PASOS

### Inmediatos (Esta Semana)
- [ ] Agendar 5 user interviews con profesionales mid-level Lima
- [ ] Crear landing page test para validar propuesta de valor
- [ ] Mapear competidores locales e internacionales

### Corto Plazo (2 Semanas)
- [ ] Completar 15 user interviews
- [ ] Analisis willingness-to-pay
- [ ] Definir target user persona final
- [ ] Confirmar revenue model

### Antes de MVP
- [ ] Validar los 7 supuestos provisionales
- [ ] Documentar decisiones en ADRs
- [ ] Crear especificacion funcional v1.0

---

## 10. REGISTRO DE CAMBIOS

| Version | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-30 | Research Agent | Documento inicial |

---

## 11. REFERENCIAS

- USACF Framework v2.0
- Peru Labor Market Report 2025 (INEI)
- HR Tech Landscape LATAM 2025
- Competitive Analysis: Bumeran, Computrabajo, LinkedIn Peru

---

**Nota:** Este documento debe actualizarse semanalmente durante la fase de discovery conforme se validen o invaliden los supuestos provisionales.
