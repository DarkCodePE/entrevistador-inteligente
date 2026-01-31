# GAP HUNTER - ENTREVISTADOR INTELIGENTE PERU

## USACF Gap Analysis Report
**Fecha:** 2026-01-30
**Version:** 1.0
**Metodologia:** USACF Gap Hunter Framework
**Mercado:** Peru

---

## RESUMEN EJECUTIVO

El mercado laboral peruano presenta oportunidades significativas para una plataforma de "Entrevistador Inteligente" debido a:

- **70.7%** de informalidad laboral (12.4M trabajadores)
- **88%** de empresas ya reclutan por internet
- **37%** de jovenes no pueden acceder al mercado laboral post-estudios
- **Solo 12%** de profesionales IT tienen experiencia en IA
- **Brecha salarial** de S/1,000+ entre expectativa (S/3,286) y realidad (S/2,294)

---

## 1. ANALISIS DE GAPS POR DIMENSION

### A. Gaps Funcionales

| Gap ID | Descripcion | Severidad | Impacto en Usuario | Solucion Potencial | Validacion |
|--------|-------------|-----------|-------------------|-------------------|------------|
| **GF01** | No hay prep de entrevistas integrada con plataformas locales | **ALTA** | Candidatos no preparados, ansiedad, bajo rendimiento | Simulador IA con feedback multimodal | 70% de usuarios se sienten mas preparados con IA |
| **GF02** | CVs no optimizados para ATS | **ALTA** | 99% Fortune 500 usa ATS; rechazo automatico silencioso | Parser + sugerencias de optimizacion + score ATS | Computrabajo ya educa sobre esto |
| **GF03** | Sin matching semantico de skills | **MEDIA** | Candidatos aplican a trabajos inadecuados | Algoritmo AI matching (embeddings + NLP) | Solo Krowdy ofrece algo similar en Peru |
| **GF04** | Falta de feedback post-entrevista | **ALTA** | Candidatos no saben por que fueron rechazados | Analisis de respuestas + areas de mejora | Pain point validado en encuestas |
| **GF05** | Preparacion generica, no por industria/rol | **MEDIA** | Entrevistas de tech vs finanzas son muy diferentes | Bancos de preguntas especializados + contexto sectorial | Demanda evidenciada |
| **GF06** | Sin analisis de video/voz | **ALTA** | Lenguaje corporal y tono son 55-93% de comunicacion | Computer vision + speech analysis | HireVue, VidCruiter lo ofrecen globalmente |
| **GF07** | No hay prep para entrevistas en ingles | **MEDIA** | Empresas internacionales requieren ingles | Simulador bilingue con correccion de pronunciacion | 532+ empleos en Peru requieren ingles |
| **GF08** | Sin verificacion de habilidades declaradas | **MEDIA** | Empresas desconfian de CVs inflados | Assessments tecnicos integrados + certificaciones | Tendencia global en HR tech |

### B. Gaps de Experiencia de Usuario

| Gap ID | Descripcion | Severidad | Segmento Afectado | Solucion Potencial |
|--------|-------------|-----------|-------------------|-------------------|
| **GU01** | Proceso fragmentado (multiples plataformas) | **ALTA** | Todos | One-stop-shop: CV + matching + prep + aplicacion |
| **GU02** | Feedback tardio o inexistente | **ALTA** | Candidatos activos | Feedback en tiempo real post-simulacion |
| **GU03** | Sin personalizacion real | **MEDIA** | Mid-level professionals | Algoritmos adaptativos basados en historial |
| **GU04** | Interfaces no mobile-first | **MEDIA** | Jovenes, trabajadores informales | PWA/app nativa optimizada |
| **GU05** | Barreras de idioma (solo espanol formal) | **MEDIA** | Regiones, migrantes venezolanos | Soporte multilingue + jerga local |
| **GU06** | Onboarding tedioso | **MEDIA** | Nuevos usuarios | LinkedIn import + wizard progresivo |
| **GU07** | Falta de gamificacion | **BAJA** | Gen Z | Badges, streaks, rankings anonimos |
| **GU08** | Sin tracking de progreso | **MEDIA** | Usuarios recurrentes | Dashboard personal con metricas |

### C. Gaps Tecnologicos

| Gap ID | Descripcion | Estado Actual Peru | Oportunidad | Complejidad |
|--------|-------------|-------------------|-------------|-------------|
| **GT01** | IA limitada en plataformas locales | Krowdy unico con IA basica | Diferenciador masivo | Alta |
| **GT02** | Sin analisis de video/voz para entrevistas | Inexistente | First-mover advantage | Muy Alta |
| **GT03** | Sin matching semantico de skills | Keyword-based | Precision 2-3x superior | Media |
| **GT04** | Ausencia de LLMs en preparacion | Solo ChatGPT generico | Experiencia conversacional | Media |
| **GT05** | Sin integraciones ATS locales | Fragmentado | B2B differentiator | Media |
| **GT06** | Falta de analytics predictivos | Basico | Valor para empleadores | Alta |
| **GT07** | Sin verificacion de credenciales | Manual | Trust layer | Media |
| **GT08** | Ausencia de HNSW/vector search | N/A | Velocidad de matching | Media |

### D. Gaps de Mercado

| Gap ID | Segmento Desatendido | Tamano Estimado | Dolor Principal | Disposicion a Pagar | Solucion |
|--------|---------------------|-----------------|-----------------|---------------------|----------|
| **GM01** | Recien graduados (18-24) | 2.1M+ | Sin experiencia, 85%+ informalidad | Baja ($5-10/mes) | Freemium + first-job prep |
| **GM02** | Cambio de carrera | ~1M (52% considerando cambiar) | Reskilling, validar nueva direccion | Media ($15-25/mes) | Career pivot simulator |
| **GM03** | Trabajadores informales formalizandose | 12.4M | No saben como entrar al formal | Muy Baja (subsidio) | Programa gratuito + gov partnership |
| **GM04** | Profesionales tech | ~500K | Entrevistas tecnicas especializadas | Alta ($30-50/mes) | Coding interviews + system design |
| **GM05** | Mujeres profesionales | 8M+ PEA (75% informalidad) | Brecha salarial 11.74% | Media | Negociacion salarial + coaching |
| **GM06** | Migrantes venezolanos | ~1.5M | Homologacion de titulos, cultura laboral | Baja | Programa de integracion |
| **GM07** | Ejecutivos senior (35+) | ~500K | Entrevistas de liderazgo, C-level | Alta ($50-100/mes) | Executive coaching AI |
| **GM08** | Zonas rurales | 5M+ (94% informalidad) | Acceso, conectividad, relevancia | Muy Baja | Version lite, offline-first |

### E. Gaps por Industria

| Industria | % Empleo Peru | Gap Especifico | Oportunidad |
|-----------|--------------|----------------|-------------|
| **Agricultura** | 25%+ | 94% informal, sin prep digital | Programa basico formalizacion |
| **Comercio/Retail** | 20%+ | 71.8% informal, alta rotacion | Quick-prep, entrevistas express |
| **Tecnologia** | 3-5% | Solo 12% con exp. IA, alta demanda | Tech interviews especializadas |
| **Mineria** | 2% | Alta especializacion, safety focus | Simulaciones tecnicas + seguridad |
| **Finanzas/Banca** | 5% | Regulado, compliance focus | Prep especifica banca |
| **Salud** | 4% | Telemedicina creciendo | Entrevistas medicas especializadas |
| **Call Centers/BPO** | 3% | Alta rotacion, idiomas | Prep rapida + ingles |

---

## 2. MATRIZ DE OPORTUNIDAD

| Gap | Tamano Mercado | Dificultad Resolver | Competencia | Monetizacion | **SCORE** |
|-----|----------------|---------------------|-------------|--------------|-----------|
| **GF01** Simulador IA | Grande (17M PEA) | Media | Baja local | Alta | **9.2** |
| **GF02** Optimizador ATS | Grande | Baja | Media | Media | **8.5** |
| **GM01** Recien graduados | Grande (2.1M) | Baja | Media | Baja | **7.8** |
| **GF06** Video analysis | Medio | Alta | Nula local | Alta | **8.0** |
| **GM04** Tech professionals | Medio (500K) | Media | Baja | Muy Alta | **8.7** |
| **GT03** Semantic matching | Grande | Media | Baja | Alta | **8.8** |
| **GU01** One-stop-shop | Grande | Alta | Media | Alta | **8.3** |
| **GM02** Career change | Medio (1M) | Media | Nula | Media | **8.1** |
| **GM05** Mujeres prof. | Grande (8M) | Baja | Nula | Media | **8.4** |
| **GF04** Feedback post-ent | Grande | Baja | Nula | Media | **8.6** |

### Top 5 Oportunidades Priorizadas:

1. **Simulador IA de Entrevistas** (Score 9.2) - Mercado grande, competencia local baja
2. **Semantic Matching Algorithm** (Score 8.8) - Diferenciador tecnico clave
3. **Tech Professionals Segment** (Score 8.7) - Alta disposicion a pagar
4. **Feedback Post-Entrevista** (Score 8.6) - Pain point no resuelto
5. **Optimizador ATS** (Score 8.5) - Quick win, facil implementacion

---

## 3. VALIDACION DE GAPS

### Gap GF01: Simulador IA de Entrevistas

**Evidencia de Demanda:**
- 70% de usuarios se sienten mas preparados usando IA para practicar ([Huru AI](https://huru.ai/es/que-es-una-entrevista-con-inteligencia-artificial-y-como-prepararse/))
- 98% satisfaccion entre candidatos que usan evaluaciones automatizadas
- En taller online de prep: "mitad abandono cuando llego momento de activar camara" - evidencia de necesidad de practica segura
- Solo Talently ofrece algo similar pero enfocado en tech

**Cuanto Pagarian:**
- Candidatos mid-level: S/30-50/mes (~$10-15 USD)
- Tech professionals: S/100-150/mes (~$30-45 USD)
- Empresas: S/500-2,000/mes por licencia bulk

**Viabilidad Tecnica:**
- APIs disponibles: GPT-4, Claude, Gemini
- Speech recognition: Whisper, Google Speech
- Video analysis: AWS Rekognition, Azure Face API
- Tiempo estimado MVP: 8-12 semanas

### Gap GF02: Optimizador ATS

**Evidencia de Demanda:**
- 99% de Fortune 500 usa ATS ([LinkedIn](https://es.linkedin.com/pulse/tu-cv-pas%C3%B3-el-applicant-tracking-systems))
- 84% de reclutadores peruanos considera "muy importante" CV actualizado ([Bumeran](https://www.bumeran.com.pe/blog/mercado-laboral/salario-en-peru-2025-analisis-completo-del-panorama-laboral/))
- Computrabajo ya educa sobre ATS - validacion de awareness

**Cuanto Pagarian:**
- Candidatos: S/10-20 one-time o incluido en suscripcion
- Empresas (bulk screening): S/0.50-2 por CV analizado

**Viabilidad Tecnica:**
- Parsers existentes: Affinda, Textract, open-source
- Complejidad baja-media
- Tiempo estimado: 4-6 semanas

### Gap GM04: Tech Professionals

**Evidencia de Demanda:**
- Demanda tech aumento 60% en Peru ([Nucamp](https://www.nucamp.co/blog/coding-bootcamp-peru-per-most-in-demand-tech-job-in-peru-in-2025))
- Solo 12% de IT professionals tienen experiencia IA - gap de habilidades
- Data scientists ganan ~$80,000 USD anuales
- Talently valida demanda (exits a empresas globales)

**Cuanto Pagarian:**
- $30-50 USD/mes por prep especializada
- Empresas: $500-2,000/mes por hiring assessments

**Viabilidad Tecnica:**
- Coding environments: Judge0, HackerRank API
- System design simulations: mas complejo
- Tiempo estimado: 12-16 semanas

### Gap GM01: Recien Graduados

**Evidencia de Demanda:**
- 37% no puede acceder al mercado post-estudios ([RPP](https://rpp.pe/peru/actualidad/los-jovenes-peruanos-son-los-que-tienen-mas-problemas-para-conseguir-trabajo-de-todo-america-latina-y-el-caribe-noticia-1519687))
- Peru tiene la MAYOR desventaja laboral juvenil de LATAM
- 85%+ de jovenes 14-24 en informalidad
- 70%+ de trabajadores jovenes en condiciones informales ([INEI](https://www.infobae.com/peru/2026/01/03/el-empleo-formal-e-informal-en-peru-durante-2025-balance-completo/))

**Cuanto Pagarian:**
- Limitado: $5-10 USD/mes maximo
- Modelo freemium recomendado
- Potencial de subsidio gubernamental

**Viabilidad Tecnica:**
- Reutilizar core del simulador
- Contenido adaptado (entry-level)
- Tiempo adicional: 2-4 semanas sobre MVP

---

## 4. ANALISIS COMPETITIVO LOCAL

### Competidores Directos en Peru

| Competidor | Fortaleza | Debilidad | Gap que Resuelve |
|------------|-----------|-----------|------------------|
| **Bumeran** | Base de usuarios grande | Sin prep entrevistas, sin IA | Solo job board |
| **Computrabajo** | Cobertura LATAM | UX anticuada, sin IA | Solo job board |
| **LinkedIn** | Red profesional | Costoso premium, generico | Networking, no prep local |
| **Krowdy** | IA en reclutamiento | Solo B2B, no prepara candidatos | Solo lado empresa |
| **Talently** | Tech talent + training | Nicho (solo tech) | Solo tech, no entrevistas |
| **Empleos Peru (Gob)** | Gratis, oficial | Sin tecnologia, basico | Solo listado |

### Competidores Internacionales (No en Peru)

| Competidor | Oferta | Por que No Esta en Peru |
|------------|--------|-------------------------|
| **HireVue** | Video interviews + AI | Enfoque enterprise global |
| **Pymetrics** | Games + AI assessment | B2B only |
| **Interview.io** | Mock interviews | Solo USA/tech |
| **Pramp** | Peer practice | Comunidad pequena LATAM |

### Oportunidad de White Space

**Nadie en Peru ofrece:**
1. Simulador de entrevistas con IA en espanol peruano
2. Feedback en tiempo real sobre respuestas
3. Analisis de video/voz para candidatos
4. Preparacion especializada por industria local
5. Integracion CV + matching + prep en una plataforma

---

## 5. INSIGHTS CLAVE DEL MERCADO PERUANO

### Datos Criticos

| Metrica | Valor | Fuente | Implicacion |
|---------|-------|--------|-------------|
| PEA total | 17M+ | INEI | TAM masivo |
| Tasa informalidad | 70.7% | INEI 2025 | Mercado de formalizacion |
| Reclutamiento digital | 88% | Bumeran 2025 | Canal digital dominante |
| Desempleo juvenil | 37% no accede | RPP | Segmento desatendido |
| Salario pretendido | S/3,286 | Bumeran | Pricing reference |
| Salario real | S/2,294 | INEI | Gap de expectativas |
| Brecha genero | 11.74% | Infobae | Oportunidad inclusion |
| % considera cambiar trabajo | 52% | Gestion | Alto churn potencial |
| Insatisfaccion laboral | 57% | Gestion | Demanda de mejora |
| Empresas con aumento salarial 2026 | 25% | Infobae | Presion economica |

### Tendencias Relevantes

1. **Digitalizacion acelerada post-COVID**: 88% recluta online
2. **IA en HR**: Solo 12% tiene experiencia, oportunidad de educacion
3. **Reskilling urgente**: Skills vida media 2-3 anos
4. **Trabajo remoto**: Abre mercado a empresas internacionales
5. **Formalizacion como prioridad nacional**: Meta 50% formal para 2040

---

## 6. RECOMENDACIONES ESTRATEGICAS

### Fase 1: MVP (Q1 2026)

**Enfoque:** Simulador IA + Optimizador ATS

| Feature | Gap que Resuelve | Prioridad |
|---------|------------------|-----------|
| Chatbot entrevistador IA | GF01 | P0 |
| Parser y scorer ATS | GF02 | P0 |
| Banco preguntas por rol | GF05 | P1 |
| Feedback instantaneo | GF04 | P1 |
| Dashboard progreso | GU08 | P2 |

**Target:** Mid-level professionals Lima (25-35, tech + finanzas)

### Fase 2: Expansion (Q2-Q3 2026)

| Feature | Gap que Resuelve | Prioridad |
|---------|------------------|-----------|
| Analisis de voz | GF06 | P0 |
| Semantic matching | GT03 | P0 |
| Especializacion tech | GM04 | P1 |
| Programa recien graduados | GM01 | P1 |
| Integraciones ATS | GT05 | P2 |

### Fase 3: Escala (Q4 2026+)

| Feature | Gap que Resuelve | Prioridad |
|---------|------------------|-----------|
| Video analysis completo | GT02 | P0 |
| B2B assessment platform | - | P0 |
| Programa formalizacion (gov) | GM03 | P1 |
| Expansion regional | GM08 | P2 |
| Career pivot module | GM02 | P2 |

---

## 7. METRICAS DE EXITO POR GAP

| Gap | KPI Principal | Target MVP | Target 12 meses |
|-----|---------------|------------|-----------------|
| GF01 Simulador | Simulaciones completadas/mes | 1,000 | 50,000 |
| GF02 ATS Optimizer | CVs optimizados | 500 | 25,000 |
| GF04 Feedback | NPS score | >40 | >60 |
| GM01 Graduados | Usuarios 18-24 | 200 | 10,000 |
| GM04 Tech | Usuarios tech | 100 | 5,000 |
| GT03 Matching | Match rate (apply after reco) | 15% | 30% |

---

## 8. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Baja adopcion IA | Media | Alto | Educacion + freemium |
| Competencia rapida | Media | Medio | First-mover + network effects |
| Costos IA insostenibles | Media | Alto | Caching + modelos locales |
| Regulacion datos | Baja | Alto | Compliance LPDP desde dia 1 |
| Resistencia cultural | Media | Medio | Localizacion + testimonios |

---

## 9. CONCLUSIONES

### Gaps de Mayor Impacto:

1. **Preparacion de entrevistas con IA** - Pain point universal, competencia local nula
2. **Optimizacion ATS** - Quick win, alta demanda latente
3. **Segmento tech** - Alta disposicion a pagar, crecimiento 60%
4. **Jovenes/recien graduados** - Mayor desventaja de LATAM, impacto social

### Diferenciadores Recomendados:

1. IA conversacional en espanol peruano (jerga local)
2. Feedback multimodal (texto + voz + video roadmap)
3. Especializacion por industria local (mineria, finanzas, retail)
4. Integracion CV-matching-prep en una plataforma
5. Precio accesible para mercado peruano

### Modelo de Negocio Sugerido:

- **B2C Freemium**: 3 simulaciones/mes gratis, premium S/29.90/mes
- **B2C Tech**: S/79.90/mes (coding interviews incluidas)
- **B2B Assessment**: S/500-2,000/mes por empresa
- **B2B Enterprise**: Personalizado

---

## FUENTES

### Mercado Laboral Peru
- [Infobae - Empleo formal e informal Peru 2025](https://www.infobae.com/peru/2026/01/03/el-empleo-formal-e-informal-en-peru-durante-2025-balance-completo/)
- [Infobae - 88% empresas reclutan por internet](https://www.infobae.com/peru/2025/06/02/el-88-de-empresas-en-peru-ya-recluta-por-internet-asi-puedes-destacar-en-este-nuevo-mercado-laboral/)
- [RPP - Desventaja laboral juvenil Peru](https://rpp.pe/peru/actualidad/los-jovenes-peruanos-son-los-que-tienen-mas-problemas-para-conseguir-trabajo-de-todo-america-latina-y-el-caribe-noticia-1519687)
- [Gestion - Tendencias laborales Peru 2026](https://gestion.pe/economia/empresas/talento-en-peru-tendencias-laborales-y-claves-para-las-empresas-en-2026-noticia/)

### Salarios y Expectativas
- [Bumeran - Salario Peru 2025](https://www.bumeran.com.pe/blog/mercado-laboral/salario-en-peru-2025-analisis-completo-del-panorama-laboral/)
- [La Republica - Salario pretendido](https://larepublica.pe/economia/2025/07/28/salario-pretendido-promedio-en-peru-llega-a-s3286-en-junio-2025-y-crece-094-en-el-semestre-hnews-358876)
- [Infobae - Brecha salarial genero](https://www.infobae.com/peru/2025/07/13/igual-trabajo-menor-sueldo-brecha-salarial-en-peru-persiste-mujeres-aspiran-a-menores-ingresos-que-los-hombres/)

### Tecnologia y Reclutamiento
- [Nucamp - Demanda tech Peru 2025](https://www.nucamp.co/blog/coding-bootcamp-peru-per-most-in-demand-tech-job-in-peru-in-2025)
- [Adecco - Tendencias reclutamiento 2025](https://www.adecco.com/es-pe/blog/tendencias-en-reclutamiento-2025)
- [Computrabajo - Claves superar filtros ATS](https://pe.computrabajo.com/desarrollo-profesional/busqueda-empleo/claves-superar-filtros-ats)
- [Factorial - Herramientas IA seleccion 2025](https://factorial.es/blog/herramientas-ai-seleccion-personal/)

### Informalidad y Formalizacion
- [Ministerio de Trabajo - Observatorio Formalizacion](https://www2.trabajo.gob.pe/estadisticas/observatorio-de-la-formalizacion-laboral/)
- [Buenos Empleos - Informalidad Peru 2025](https://buenosempleos.com/blog-informalidad-laboral-peru-2025-estadisticas/)

### HR Tech y Startups
- [Startupik - Top startups Peru](https://startupik.com/top-startups-peru/)
- [Wellfound - Tech startups Peru](https://wellfound.com/startups/location/peru)
- [9cv9 - Recruitment agencies Peru 2026](https://blog.9cv9.com/top-10-best-recruitment-agencies-in-peru-in-2026/)

---

**Documento generado por:** Research Agent - USACF Gap Hunter
**Siguiente paso:** Validacion de supuestos con user interviews (ver ambiguity-analysis.md)
