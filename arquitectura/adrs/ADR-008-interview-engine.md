# ADR-008: Motor de Simulacion de Entrevistas con IA

## Metadata

| Campo | Valor |
|-------|-------|
| **Estado** | Propuesto |
| **Fecha** | 2026-01-31 |
| **Decisores** | Equipo de Arquitectura, CTO, Product Lead |
| **Contexto** | Core del producto - Simulador de entrevistas |
| **Relacionado** | ADR-001 (Arquitectura General), ADR-003 (Modelo de IA) |

---

## Contexto

El simulador de entrevistas es el **nucleo diferenciador** de Entrevistador Inteligente. El 60% de candidatos peruanos fracasan en entrevistas por falta de preparacion, y no existe ninguna solucion escalable de practica con IA en espanol peruano.

### Flujo de Entrevista Objetivo

```
Setup (rol, industria, nivel)
    |
    v
Generacion de preguntas contextuales
    |
    v
+----------------------------------+
|  LOOP DE ENTREVISTA (5-10 Q)     |
|  +----------------------------+  |
|  | Pregunta N                 |  |
|  | Usuario responde           |  |
|  | Feedback inmediato         |  |
|  | Follow-up dinamico         |  |
|  +----------------------------+  |
+----------------------------------+
    |
    v
Analisis final consolidado
    |
    v
Score + Recomendaciones personalizadas
```

### Restricciones del Negocio

| Restriccion | Valor | Justificacion |
|-------------|-------|---------------|
| Latencia por respuesta | < 3 segundos | UX fluida tipo conversacion |
| Costo por entrevista | < $0.10 USD | Margenes viables en plan basico |
| Idioma | Espanol peruano | Localizacion como diferenciador |
| MVP Timeline | 8 semanas | Validacion rapida de mercado |

---

## Decisiones

### Decision 1: Seleccion de LLM - Claude 3.5 Sonnet (Primario) + GPT-4o-mini (Fallback)

#### Opciones Evaluadas

| Criterio | GPT-4o | Claude 3.5 Sonnet | GPT-4o-mini | Gemini 1.5 Pro | Llama 3.1 70B |
|----------|--------|-------------------|-------------|----------------|---------------|
| Calidad respuesta (1-10) | 9 | 9 | 7 | 8 | 7 |
| Latencia p95 | 2.5s | 1.8s | 0.8s | 2.0s | Variable |
| Costo/1M tokens (input) | $5.00 | $3.00 | $0.15 | $3.50 | $0.00* |
| Costo/1M tokens (output) | $15.00 | $15.00 | $0.60 | $10.50 | $0.00* |
| Espanol peruano | Bueno | Excelente | Bueno | Bueno | Aceptable |
| Consistencia formato | Alta | Muy Alta | Media | Alta | Media |
| Context window | 128K | 200K | 128K | 1M | 128K |
| Rate limits | 10K TPM | 20K TPM | 150K TPM | 20K TPM | N/A |

*Llama: Costo de infraestructura ~$3K-8K/mes para hosting

#### Decision

**Claude 3.5 Sonnet como modelo primario** con GPT-4o-mini como fallback.

#### Justificacion

1. **Calidad en espanol**: Claude muestra mejor comprension de matices del espanol latinoamericano
2. **Latencia competitiva**: 1.8s p95 cumple el requisito < 3s con margen
3. **Context window**: 200K tokens permite sesiones largas sin truncamiento
4. **Consistencia**: Respuestas mas estructuradas y predecibles para parsing
5. **Costo optimizado**: Modelo tiered reduce costo promedio a ~$0.05/entrevista

#### Arquitectura de Modelos Tiered

```
                    Costo/Entrevista

Tier 1 (Premium)    [$0.12]
+-------------------+
| Claude 3.5 Sonnet |  <- Evaluacion final, feedback detallado
+-------------------+     Casos complejos, preguntas tecnicas
        |
        v
Tier 2 (Standard)   [$0.04]
+-------------------+
| GPT-4o-mini       |  <- Preguntas estandar, follow-ups
+-------------------+     Flujo conversacional
        |
        v
Tier 3 (Cache)      [$0.001]
+-------------------+
| Respuestas cached |  <- Preguntas frecuentes
| + Templates       |     Intro/cierre de entrevista
+-------------------+
```

**Costo promedio por entrevista (30 min, 10 preguntas):**
- 3 preguntas Tier 1 (evaluacion, tecnicas): $0.036
- 5 preguntas Tier 2 (conductuales): $0.020
- 2 elementos Tier 3 (intro/cierre): $0.002
- **Total: ~$0.058 USD** (cumple < $0.10)

#### Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Anthropic rate limits | Media | Alto | Fallback a GPT-4o-mini automatico |
| Cambio de pricing | Media | Alto | Contratos anuales, modelo propio Q4 |
| Calidad inconsistente | Baja | Medio | Evaluacion continua, A/B testing |
| Latency spikes | Media | Medio | Circuit breaker, timeout 5s |

---

### Decision 2: Estrategia de Prompt Engineering - RAG + Few-Shot + Chain-of-Thought

#### Arquitectura de Prompts

```
+------------------------------------------------------------------+
|                    PROMPT ARCHITECTURE                            |
+------------------------------------------------------------------+
|                                                                  |
|  +--------------------+    +--------------------+                |
|  | SYSTEM PROMPT      |    | CONTEXT INJECTION  |                |
|  | (Rol de IA)        |    | (RAG)              |                |
|  +--------------------+    +--------------------+                |
|           |                         |                            |
|           +------------+------------+                            |
|                        |                                         |
|                        v                                         |
|           +------------------------+                             |
|           | CONTEXTO CANDIDATO     |                             |
|           | - CV parsed            |                             |
|           | - Rol objetivo         |                             |
|           | - Industria            |                             |
|           | - Nivel experiencia    |                             |
|           | - Historia sesion      |                             |
|           +------------------------+                             |
|                        |                                         |
|                        v                                         |
|           +------------------------+                             |
|           | FEW-SHOT EXAMPLES      |                             |
|           | (3-5 ejemplos)         |                             |
|           | - Pregunta tipo        |                             |
|           | - Respuesta ideal      |                             |
|           | - Rubrica evaluacion   |                             |
|           +------------------------+                             |
|                        |                                         |
|                        v                                         |
|           +------------------------+                             |
|           | CHAIN-OF-THOUGHT       |                             |
|           | Instructions           |                             |
|           | - Analizar respuesta   |                             |
|           | - Identificar STAR     |                             |
|           | - Evaluar dimensiones  |                             |
|           | - Generar feedback     |                             |
|           +------------------------+                             |
|                                                                  |
+------------------------------------------------------------------+
```

#### Estructura del System Prompt

```markdown
## ROL
Eres un entrevistador profesional de {industria} en Peru,
evaluando candidatos para el puesto de {rol} nivel {nivel}.

## PERSONALIDAD
- Profesional pero cercano (tuteo natural en Peru)
- Usa expresiones peruanas naturales (ej: "chevere", "ya")
- Evita frases de Espana (ej: NO "vale", NO "tio")
- Feedback constructivo, nunca humillante

## FORMATO DE RESPUESTA
Siempre responde en JSON estructurado:
{
  "pregunta": "texto de la pregunta",
  "tipo": "behavioral|technical|situational|case",
  "dificultad": 1-5,
  "tiempo_sugerido": "2-3 minutos",
  "rubrica": {
    "excelente": "descripcion",
    "bueno": "descripcion",
    "mejorable": "descripcion"
  }
}

## REGLAS DE EVALUACION
1. Estructura STAR para conductuales (Situacion, Tarea, Accion, Resultado)
2. Evaluar: Relevancia, Especificidad, Impacto cuantificable
3. Penalizar: Respuestas vagas, falta de ejemplos, exceso de "yo"
```

#### RAG para Contexto Empresarial

| Fuente | Contenido | Uso |
|--------|-----------|-----|
| Banco de preguntas | 500+ preguntas reales Peru | Personalizacion por industria |
| Cultura empresarial | Valores de empresas target | Preguntas de fit cultural |
| Competencias por rol | Skills requeridos por puesto | Evaluacion tecnica |
| Respuestas modelo | Ejemplos de respuestas A+ | Few-shot learning |

#### Riesgos y Mitigaciones

| Riesgo | Mitigacion |
|--------|------------|
| Prompt injection | Sanitizacion de inputs, guardrails |
| Respuestas genericas | Few-shot con ejemplos Peru especificos |
| Hallucinations | Grounding en banco de preguntas verificadas |
| Formato inconsistente | JSON schema validation, retry logic |

---

### Decision 3: Context Management - Sliding Window con Memoria Estructurada

#### Problema

Una entrevista de 30 minutos genera ~15,000 tokens de contexto. Mantener toda la conversacion es costoso e innecesario.

#### Solucion: Sliding Window + Summary Memory

```
+------------------------------------------------------------------+
|                   CONTEXT MANAGEMENT                              |
+------------------------------------------------------------------+
|                                                                  |
|  SESSION START                                                   |
|  +---------------------------+                                   |
|  | Static Context (fixed)    |  ~2,000 tokens                   |
|  | - System prompt           |                                   |
|  | - CV summary              |                                   |
|  | - Role/company context    |                                   |
|  | - Few-shot examples       |                                   |
|  +---------------------------+                                   |
|                                                                  |
|  DURANTE ENTREVISTA                                              |
|  +---------------------------+                                   |
|  | Rolling Summary           |  ~1,000 tokens                   |
|  | (actualizado cada 3 Q)    |                                   |
|  | - Temas cubiertos         |                                   |
|  | - Fortalezas detectadas   |                                   |
|  | - Areas a profundizar     |                                   |
|  | - Score parcial           |                                   |
|  +---------------------------+                                   |
|              +                                                   |
|  +---------------------------+                                   |
|  | Sliding Window            |  ~3,000 tokens                   |
|  | (ultimas 3 Q&A)           |                                   |
|  | - Pregunta N-2            |                                   |
|  | - Pregunta N-1            |                                   |
|  | - Pregunta N (actual)     |                                   |
|  +---------------------------+                                   |
|                                                                  |
|  TOTAL CONTEXTO: ~6,000 tokens (fijo)                           |
|                                                                  |
+------------------------------------------------------------------+
```

#### Estado de Sesion (Redis)

```json
{
  "session_id": "uuid",
  "candidate_id": "uuid",
  "started_at": "2026-01-31T10:00:00Z",
  "config": {
    "role": "Product Manager",
    "industry": "tech",
    "level": "mid",
    "duration_minutes": 30,
    "question_count": 10
  },
  "state": {
    "current_question": 5,
    "questions_asked": [...],
    "responses": [...],
    "scores": {
      "content": [7, 8, 6, 7, 0],
      "communication": [8, 7, 7, 8, 0],
      "technical": [0, 9, 0, 0, 0]
    },
    "rolling_summary": "Candidato muestra experiencia en...",
    "topics_covered": ["liderazgo", "conflictos", "priorización"],
    "topics_remaining": ["metricas", "stakeholders"]
  },
  "ttl": 7200
}
```

#### Persistencia y Recovery

| Componente | Storage | TTL | Recovery |
|------------|---------|-----|----------|
| Session state | Redis | 2 horas | Reconexion automatica |
| Q&A history | PostgreSQL | 1 ano | Replay de sesion |
| Audio/Video | S3 | 90 dias | Re-analisis |
| Feedback | PostgreSQL | Indefinido | Historico candidato |

---

### Decision 4: Algoritmo de Scoring - Rubrica Multi-dimensional con Pesos Adaptativos

#### Dimensiones de Evaluacion

```
+------------------------------------------------------------------+
|                    SCORING DIMENSIONS                             |
+------------------------------------------------------------------+
|                                                                  |
|  DIMENSION 1: CONTENIDO (35%)                                    |
|  +------------------------+                                      |
|  | - Relevancia (10%)     |  ¿Responde la pregunta?             |
|  | - Especificidad (10%)  |  ¿Ejemplos concretos?               |
|  | - Estructura (10%)     |  ¿Usa STAR method?                  |
|  | - Impacto (5%)         |  ¿Resultados cuantificables?        |
|  +------------------------+                                      |
|                                                                  |
|  DIMENSION 2: COMUNICACION (25%)                                 |
|  +------------------------+                                      |
|  | - Claridad (10%)       |  ¿Facil de entender?                |
|  | - Concision (8%)       |  ¿Sin rodeos?                       |
|  | - Profesionalismo (7%) |  ¿Tono adecuado?                    |
|  +------------------------+                                      |
|                                                                  |
|  DIMENSION 3: TECNICO (20%) - Solo si aplica                    |
|  +------------------------+                                      |
|  | - Precision (10%)      |  ¿Correcto tecnicamente?            |
|  | - Profundidad (5%)     |  ¿Nivel de detalle?                 |
|  | - Actualidad (5%)      |  ¿Conoce tendencias?                |
|  +------------------------+                                      |
|                                                                  |
|  DIMENSION 4: SOFT SKILLS (20%)                                  |
|  +------------------------+                                      |
|  | - Autoconocimiento(7%) |  ¿Conoce fortalezas/debilidades?    |
|  | - Adaptabilidad (7%)   |  ¿Flexible ante cambios?            |
|  | - Colaboracion (6%)    |  ¿Trabajo en equipo?                |
|  +------------------------+                                      |
|                                                                  |
+------------------------------------------------------------------+
```

#### Algoritmo de Evaluacion

```python
# Pseudocode del scoring engine

def evaluate_response(response: str, question: Question, context: Context) -> Score:

    # 1. Analisis con LLM (Chain-of-Thought)
    analysis = llm.analyze(
        prompt=EVALUATION_PROMPT,
        response=response,
        question=question,
        rubric=question.rubric,
        cv_context=context.cv
    )

    # 2. Extraccion de scores por dimension
    raw_scores = {
        "content": extract_content_score(analysis),
        "communication": extract_communication_score(analysis),
        "technical": extract_technical_score(analysis) if question.is_technical else None,
        "soft_skills": extract_soft_skills_score(analysis)
    }

    # 3. Aplicar pesos adaptativos segun rol
    weights = get_weights_for_role(context.role, context.level)

    # 4. Calcular score ponderado
    weighted_score = sum(
        raw_scores[dim] * weights[dim]
        for dim in raw_scores
        if raw_scores[dim] is not None
    )

    # 5. Normalizar a escala 0-100
    final_score = normalize(weighted_score, min=0, max=100)

    return Score(
        overall=final_score,
        dimensions=raw_scores,
        feedback=analysis.feedback,
        strengths=analysis.strengths,
        improvements=analysis.improvements
    )
```

#### Pesos por Rol y Nivel

| Rol | Nivel | Contenido | Comunicacion | Tecnico | Soft Skills |
|-----|-------|-----------|--------------|---------|-------------|
| Engineer | Junior | 30% | 20% | 35% | 15% |
| Engineer | Senior | 25% | 20% | 40% | 15% |
| PM | Mid | 35% | 30% | 15% | 20% |
| PM | Senior | 30% | 30% | 15% | 25% |
| Sales | Any | 25% | 40% | 5% | 30% |
| Leadership | Any | 25% | 30% | 10% | 35% |

#### Calibracion y Consistency

- **Benchmark dataset**: 200 respuestas evaluadas por expertos humanos
- **Correlation target**: > 0.85 con evaluacion humana
- **Inter-rater reliability**: Cohen's kappa > 0.7
- **Re-calibracion**: Mensual con nuevos datos

---

### Decision 5: Feedback Generation - Especifico y Accionable

#### Framework de Feedback

```
+------------------------------------------------------------------+
|                    FEEDBACK STRUCTURE                             |
+------------------------------------------------------------------+
|                                                                  |
|  INMEDIATO (despues de cada respuesta)                          |
|  +---------------------------+                                   |
|  | Quick Score: 7/10         |  Indicador visual                |
|  | Mejor momento: "Cuando    |  Refuerzo positivo especifico    |
|  |   mencionaste el 30%..."  |                                   |
|  | Tip rapido: "Agrega mas   |  1 mejora concreta               |
|  |   datos cuantitativos"    |                                   |
|  +---------------------------+                                   |
|                                                                  |
|  FINAL (al terminar entrevista)                                 |
|  +---------------------------+                                   |
|  | Score Global: 78/100      |                                   |
|  |                           |                                   |
|  | Desglose por Dimension:   |  Radar chart visual              |
|  | - Contenido: 82           |                                   |
|  | - Comunicacion: 75        |                                   |
|  | - Tecnico: 80             |                                   |
|  | - Soft Skills: 72         |                                   |
|  |                           |                                   |
|  | Top 3 Fortalezas:         |  Con ejemplos de la sesion       |
|  | 1. Ejemplos cuantificados |                                   |
|  | 2. Estructura STAR        |                                   |
|  | 3. Conocimiento tecnico   |                                   |
|  |                           |                                   |
|  | Top 3 Areas de Mejora:    |  Con acciones concretas          |
|  | 1. Reducir muletillas     |  -> Ejercicio de grabacion       |
|  | 2. Respuestas mas cortas  |  -> Practicar 2-min answers      |
|  | 3. Mas ejemplos de        |  -> Revisar STAR worksheet       |
|  |    colaboracion           |                                   |
|  |                           |                                   |
|  | Recursos Recomendados:    |  Links a contenido               |
|  | - Video: "STAR Method"    |                                   |
|  | - Ejercicio: "Elevator    |                                   |
|  |   Pitch"                  |                                   |
|  +---------------------------+                                   |
|                                                                  |
|  COMPARATIVO (si hay historial)                                 |
|  +---------------------------+                                   |
|  | vs. Sesion Anterior:      |  Progreso visible                |
|  | +5 puntos overall         |                                   |
|  | +8 en Comunicacion        |                                   |
|  | -2 en Tecnico (revisar)   |                                   |
|  +---------------------------+                                   |
|                                                                  |
+------------------------------------------------------------------+
```

#### Ejemplos de Feedback Especifico vs Generico

| Tipo | Generico (MALO) | Especifico (BUENO) |
|------|-----------------|-------------------|
| Fortaleza | "Buena respuesta" | "Tu ejemplo del proyecto de migracion mostro liderazgo: coordinar 5 equipos y reducir downtime 40% es impresionante" |
| Mejora | "Se mas conciso" | "Tu respuesta sobre conflictos duro 4 minutos. Para este tipo de pregunta, apunta a 2-2.5 minutos. Tip: usa el formato STAR y cronometra tus practicas" |
| Tecnico | "Estudia mas" | "Mencionaste microservicios pero no profundizaste en patrones de comunicacion. Para roles senior, preparate para explicar diferencias entre REST vs gRPC vs eventos" |

#### Personalizacion por Industria Peruana

| Industria | Contexto Local | Ajuste en Feedback |
|-----------|---------------|-------------------|
| Banca | BCP, Interbank, BBVA | Enfasis en regulacion SBS, riesgos |
| Retail | Falabella, Ripley, Cencosud | Omnicanalidad, experiencia cliente |
| Tech | Startups Lima, nearshoring | Metodologias agiles, ingles tecnico |
| Mineria | Southern, Antamina | Seguridad, medio ambiente, comunidades |
| Consumo | Alicorp, Backus | Trade marketing, distribucion |

---

### Decision 6: Modos de Entrevista y Roadmap de Features

#### Modo 1: Texto (MVP - Semana 1-8)

```
+------------------------------------------------------------------+
|                    MODO TEXTO (MVP)                               |
+------------------------------------------------------------------+
|                                                                  |
|  INPUT                           OUTPUT                          |
|  +----------------+              +----------------+               |
|  | Texto escrito  |  ---------> | Pregunta texto |               |
|  | por usuario    |              | + Quick score  |               |
|  +----------------+              | + Tip          |               |
|                                  +----------------+               |
|                                                                  |
|  Tech Stack:                                                     |
|  - Frontend: React (responsive)                                  |
|  - Backend: FastAPI/Node                                         |
|  - LLM: Claude 3.5 Sonnet / GPT-4o-mini                         |
|  - State: Redis + PostgreSQL                                     |
|                                                                  |
|  Latencia Target: < 2 segundos                                   |
|  Costo Target: < $0.05 USD/entrevista                           |
|                                                                  |
+------------------------------------------------------------------+
```

#### Modo 2: Audio (V2 - Semana 9-16)

```
+------------------------------------------------------------------+
|                    MODO AUDIO (V2)                                |
+------------------------------------------------------------------+
|                                                                  |
|  INPUT                           OUTPUT                          |
|  +----------------+              +----------------+               |
|  | Audio grabado  |  ---------> | Pregunta voz   |               |
|  | (WebRTC/upload)|  STT        | (TTS)          |               |
|  +----------------+  Whisper    +----------------+               |
|         |                              |                         |
|         v                              v                         |
|  +----------------+              +----------------+               |
|  | Transcripcion  |              | Audio feedback |               |
|  | con timestamps |              | + Analisis de  |               |
|  | y confidence   |              |   fluency      |               |
|  +----------------+              +----------------+               |
|                                                                  |
|  Tech Stack Adicional:                                           |
|  - STT: Whisper large-v3 (fine-tuned espanol peruano)           |
|  - TTS: ElevenLabs / Azure Neural Voices (voz peruana)          |
|  - Audio processing: FFmpeg                                      |
|  - Storage: S3 para grabaciones                                  |
|                                                                  |
|  Metricas Adicionales:                                           |
|  - Palabras por minuto                                           |
|  - Muletillas detectadas                                         |
|  - Pausas y silencios                                            |
|  - Tono de voz (confianza)                                       |
|                                                                  |
|  Latencia Target: < 3 segundos (STT + LLM + TTS)                |
|  Costo Target: < $0.08 USD/entrevista                           |
|                                                                  |
+------------------------------------------------------------------+
```

#### Modo 3: Video (V3 - Semana 17-24)

```
+------------------------------------------------------------------+
|                    MODO VIDEO (V3)                                |
+------------------------------------------------------------------+
|                                                                  |
|  INPUT                           OUTPUT                          |
|  +----------------+              +----------------+               |
|  | Video stream   |  ---------> | Avatar IA      |               |
|  | (WebRTC)       |  + Audio    | interactivo    |               |
|  +----------------+  + Vision   +----------------+               |
|         |                              |                         |
|         v                              v                         |
|  +----------------+              +----------------+               |
|  | Analisis:      |              | Feedback:      |               |
|  | - Expresion    |              | - Eye contact  |               |
|  | - Postura      |              | - Postura      |               |
|  | - Gestos       |              | - Nerviosismo  |               |
|  | - Eye contact  |              | - Presencia    |               |
|  +----------------+              +----------------+               |
|                                                                  |
|  Tech Stack Adicional:                                           |
|  - Video processing: MediaPipe / OpenCV                          |
|  - Facial analysis: Azure Face API / custom model                |
|  - Avatar: HeyGen / D-ID / custom                                |
|  - WebRTC: Twilio / Daily.co                                     |
|                                                                  |
|  Analisis de Video:                                              |
|  - Eye contact score (% tiempo mirando camara)                   |
|  - Postura score (hombros, inclinacion)                          |
|  - Expresion facial (confianza, nerviosismo, sonrisa)            |
|  - Gestos (manos, movimientos distractores)                      |
|  - Background profesionalismo                                     |
|                                                                  |
|  Latencia Target: < 500ms video processing                       |
|  Costo Target: < $0.15 USD/entrevista                           |
|                                                                  |
+------------------------------------------------------------------+
```

---

### Decision 7: Tipos de Preguntas y Banco de Preguntas

#### Taxonomia de Preguntas

```
+------------------------------------------------------------------+
|                    QUESTION TAXONOMY                              |
+------------------------------------------------------------------+
|                                                                  |
|  1. BEHAVIORAL (40% de entrevista)                              |
|  +---------------------------+                                   |
|  | Evaluan: Comportamiento   |  Ejemplo:                        |
|  | pasado como predictor     |  "Cuentame de una vez que        |
|  | de futuro                 |   tuviste un conflicto con       |
|  |                           |   un companero. Como lo          |
|  | Framework: STAR method    |   resolviste?"                   |
|  +---------------------------+                                   |
|                                                                  |
|  2. TECHNICAL (30% - segun rol)                                 |
|  +---------------------------+                                   |
|  | Evaluan: Conocimientos    |  Ejemplo (PM):                   |
|  | especificos del area      |  "Como priorizarias features     |
|  |                           |   con recursos limitados?        |
|  | Variantes:                |   Que framework usarias?"        |
|  | - Conceptual              |                                   |
|  | - Problem-solving         |  Ejemplo (Eng):                  |
|  | - System design           |  "Disena un sistema de           |
|  |                           |   notificaciones que maneje      |
|  |                           |   1M usuarios"                   |
|  +---------------------------+                                   |
|                                                                  |
|  3. SITUATIONAL (15%)                                           |
|  +---------------------------+                                   |
|  | Evaluan: Juicio en        |  Ejemplo:                        |
|  | situaciones hipoteticas   |  "Si tu jefe te pide entregar    |
|  |                           |   un proyecto en 1 semana pero   |
|  | Focus: Toma de decisiones |   sabes que necesitas 3,         |
|  |                           |   que harias?"                   |
|  +---------------------------+                                   |
|                                                                  |
|  4. CASE STUDY (10% - consultoria/finanzas)                     |
|  +---------------------------+                                   |
|  | Evaluan: Pensamiento      |  Ejemplo:                        |
|  | estructurado, analisis    |  "Un cliente de retail quiere    |
|  |                           |   expandirse a provincia.        |
|  | Industrias: Consulting,   |   Como estructurarias el         |
|  | Banking, Strategy         |   analisis?"                     |
|  +---------------------------+                                   |
|                                                                  |
|  5. CODING (Tech roles only)                                    |
|  +---------------------------+                                   |
|  | Evaluan: Habilidad de     |  Ejemplo:                        |
|  | programacion en vivo      |  "Implementa una funcion que     |
|  |                           |   encuentre el segundo           |
|  | Formato: Editor embebido  |   elemento mas grande en         |
|  | Lenguajes: Python, JS,    |   un array"                      |
|  | Java, etc.                |                                   |
|  +---------------------------+                                   |
|                                                                  |
+------------------------------------------------------------------+
```

#### Estructura del Banco de Preguntas

```sql
-- Schema para banco de preguntas

CREATE TABLE questions (
    id UUID PRIMARY KEY,
    question_text TEXT NOT NULL,
    question_type ENUM('behavioral', 'technical', 'situational', 'case', 'coding'),

    -- Targeting
    industries TEXT[],           -- ['tech', 'banking', 'retail']
    roles TEXT[],                -- ['pm', 'engineer', 'analyst']
    levels ENUM('junior', 'mid', 'senior', 'executive')[],

    -- Metadata
    difficulty INT CHECK (1 <= difficulty <= 5),
    estimated_time_seconds INT,
    competencies TEXT[],         -- ['leadership', 'problem-solving']

    -- Evaluation
    rubric JSONB,                -- Criterios de evaluacion
    example_excellent TEXT,      -- Respuesta A+
    example_good TEXT,           -- Respuesta B
    example_poor TEXT,           -- Respuesta a evitar

    -- Usage
    times_asked INT DEFAULT 0,
    avg_score DECIMAL(5,2),
    is_active BOOLEAN DEFAULT true,

    -- Localization
    region TEXT DEFAULT 'peru',
    language TEXT DEFAULT 'es-PE',

    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP
);

-- Ejemplo de pregunta
INSERT INTO questions VALUES (
    'uuid-123',
    'Cuentame de un proyecto donde tuviste que liderar un equipo bajo presion.
     Cual fue el contexto, que acciones tomaste y cual fue el resultado?',
    'behavioral',
    ARRAY['tech', 'banking', 'retail'],
    ARRAY['pm', 'tech-lead', 'manager'],
    ARRAY['mid', 'senior'],
    3,
    180,
    ARRAY['leadership', 'pressure-handling', 'results-orientation'],
    '{
        "excellent": "Usa STAR completo, resultados cuantificados, refleja aprendizaje",
        "good": "STAR parcial, resultados claros pero no cuantificados",
        "poor": "Vago, sin estructura, culpa a otros"
    }',
    'En mi rol como PM en [empresa], lideramos una migracion de...',
    'Tuve un proyecto dificil donde...',
    'Siempre trabajo bien bajo presion, me gusta...',
    150,
    72.5,
    true,
    'peru',
    'es-PE',
    NOW(),
    NOW()
);
```

#### Cobertura de Preguntas MVP

| Categoria | Cantidad | Industrias | Ejemplo |
|-----------|----------|------------|---------|
| Behavioral - Liderazgo | 25 | General | Conflictos, feedback, delegacion |
| Behavioral - Logros | 20 | General | Mayor exito, fracaso, presion |
| Behavioral - Colaboracion | 20 | General | Equipos, stakeholders, cross-functional |
| Technical - PM | 30 | Tech | Priorizacion, metricas, roadmap |
| Technical - Eng | 40 | Tech | System design, coding, debugging |
| Technical - Finance | 25 | Banca | Analisis financiero, riesgos, regulacion |
| Situational | 30 | General | Escenarios eticos, trade-offs |
| Case Study | 20 | Consulting | Market sizing, estrategia |
| **TOTAL MVP** | **210** | | |

---

## Arquitectura del Interview Engine

### Diagrama de Componentes

```
+------------------------------------------------------------------+
|                    INTERVIEW ENGINE ARCHITECTURE                  |
+------------------------------------------------------------------+

                    +-------------------+
                    |   API Gateway     |
                    |  /api/interview   |
                    +--------+----------+
                             |
                             v
+----------------------------+----------------------------+
|                    INTERVIEW SERVICE                    |
+----------------------------+----------------------------+
|                            |                            |
|  +------------------+      |      +------------------+  |
|  | Session Manager  |      |      | State Machine    |  |
|  |                  |      |      |                  |  |
|  | - Create session |      |      | - SETUP         |  |
|  | - Load context   |      |      | - QUESTIONING   |  |
|  | - Persist state  |      |      | - EVALUATING    |  |
|  | - Handle timeout |      |      | - COMPLETE      |  |
|  +------------------+      |      +------------------+  |
|                            |                            |
+---------+------------------+------------------+---------+
          |                                     |
          v                                     v
+---------+----------+               +----------+---------+
| QUESTION GENERATOR |               | RESPONSE EVALUATOR |
+--------------------+               +--------------------+
|                    |               |                    |
| +----------------+ |               | +----------------+ |
| | Context        | |               | | LLM Evaluator  | |
| | Builder        | |               | | (Claude 3.5)   | |
| +----------------+ |               | +----------------+ |
|        |           |               |        |           |
| +----------------+ |               | +----------------+ |
| | Question       | |               | | Scoring        | |
| | Selector       | |               | | Engine         | |
| | (RAG + rules)  | |               | +----------------+ |
| +----------------+ |               |        |           |
|        |           |               | +----------------+ |
| +----------------+ |               | | Feedback       | |
| | LLM Generator  | |               | | Generator      | |
| | (GPT-4o-mini)  | |               | +----------------+ |
| +----------------+ |               |                    |
+--------------------+               +--------------------+
          |                                     |
          v                                     v
+---------+----------+               +----------+---------+
|   QUESTION BANK    |               |   SCORING RULES    |
|   (PostgreSQL)     |               |   (Config/Code)    |
+--------------------+               +--------------------+

          |                                     |
          +-----------------+-------------------+
                            |
                            v
                  +---------+----------+
                  |   DATA LAYER       |
                  +--------------------+
                  | PostgreSQL: Users, |
                  |   Sessions, Q&A    |
                  | Redis: State, Cache|
                  | S3: Audio, Video   |
                  +--------------------+
```

### State Machine de Entrevista

```
+------------------------------------------------------------------+
|                    INTERVIEW STATE MACHINE                        |
+------------------------------------------------------------------+

    +--------+
    | SETUP  |  <- Usuario configura: rol, industria, nivel, duracion
    +---+----+
        |
        | config_complete
        v
    +--------+     timeout (30min)     +-----------+
    |QUESTION| ------------------->    | ABANDONED |
    | ING    |                         +-----------+
    +---+----+
        |
        | all_questions_answered
        | OR user_ends_early
        v
    +----------+
    |EVALUATING|  <- Generando reporte final
    +----+-----+
         |
         | evaluation_complete
         v
    +----------+
    | COMPLETE |  <- Mostrando resultados
    +----+-----+
         |
         | user_reviews OR timeout (1hr)
         v
    +----------+
    | ARCHIVED |  <- Sesion guardada en historial
    +----------+
```

### Flujo de Datos Detallado

```
+------------------------------------------------------------------+
|                    DATA FLOW - SINGLE QUESTION                    |
+------------------------------------------------------------------+

1. GENERATE QUESTION
   +------------------+     +------------------+
   | Session Context  | --> | Context Builder  |
   | - CV summary     |     | (merge + format) |
   | - Config         |     +--------+---------+
   | - History        |              |
   +------------------+              v
                            +--------+---------+
                            | Question Selector|
   +------------------+     | - Filter by type |
   | Question Bank    | --> | - Filter by level|
   +------------------+     | - Avoid repeats  |
                            +--------+---------+
                                     |
                                     v
                            +--------+---------+
                            | LLM Personalizer |
                            | - Adapt to CV    |
                            | - Add follow-up  |
                            | - Format output  |
                            +--------+---------+
                                     |
                                     v
                            +------------------+
                            | Question JSON    |
                            | {                |
                            |   text: "...",   |
                            |   type: "behav", |
                            |   time: 120,     |
                            |   rubric: {...}  |
                            | }                |
                            +------------------+

2. EVALUATE RESPONSE
   +------------------+     +------------------+
   | User Response    | --> | Pre-processor    |
   | (text/audio/vid) |     | - Transcribe     |
   +------------------+     | - Clean text     |
                            | - Extract audio  |
                            +--------+---------+
                                     |
                                     v
                            +--------+---------+
   +------------------+     | LLM Evaluator    |
   | Question Context | --> | - Analyze STAR   |
   | - Original Q     |     | - Check rubric   |
   | - Rubric         |     | - Score dims     |
   | - CV context     |     +--------+---------+
   +------------------+              |
                                     v
                            +--------+---------+
   +------------------+     | Scoring Engine   |
   | Role Weights     | --> | - Apply weights  |
   | {content: 0.35}  |     | - Normalize 0-100|
   +------------------+     +--------+---------+
                                     |
                                     v
                            +------------------+
                            | Response Score   |
                            | {                |
                            |   overall: 78,   |
                            |   content: 82,   |
                            |   comm: 75,      |
                            |   feedback: "...",|
                            |   tip: "..."     |
                            | }                |
                            +------------------+
```

---

## API Specification

### Endpoints Principales

```yaml
# Interview Engine API - OpenAPI 3.0 Summary

/api/v1/interviews:
  POST:
    summary: Create new interview session
    request:
      body:
        role: string (required)
        industry: string (required)
        level: enum[junior, mid, senior, executive]
        company_target: string (optional)
        duration_minutes: int (default: 30)
        question_types: string[] (default: ["behavioral", "situational"])
    response:
      201:
        session_id: uuid
        first_question: Question
        config: InterviewConfig
        expires_at: datetime

/api/v1/interviews/{session_id}/respond:
  POST:
    summary: Submit response to current question
    request:
      body:
        response_text: string (required if mode=text)
        audio_url: string (required if mode=audio)
        video_url: string (required if mode=video)
    response:
      200:
        score: QuickScore
        feedback: string
        next_question: Question | null
        progress: int (1-10)

/api/v1/interviews/{session_id}/complete:
  POST:
    summary: End interview and generate final report
    response:
      200:
        final_score: int (0-100)
        dimension_scores: DimensionScores
        strengths: string[]
        improvements: ImprovementItem[]
        resources: Resource[]
        comparison: HistoricalComparison | null

/api/v1/interviews/{session_id}/status:
  GET:
    summary: Get current session status
    response:
      200:
        state: enum[questioning, evaluating, complete, abandoned]
        current_question: int
        total_questions: int
        time_remaining_seconds: int

/api/v1/interviews/history:
  GET:
    summary: Get user's interview history
    query:
      limit: int (default: 10)
      offset: int
      role: string (filter)
    response:
      200:
        interviews: InterviewSummary[]
        total: int
        has_more: boolean
```

### Modelos de Datos

```typescript
// Core Types

interface Question {
  id: string;
  text: string;
  type: 'behavioral' | 'technical' | 'situational' | 'case' | 'coding';
  difficulty: 1 | 2 | 3 | 4 | 5;
  estimated_time_seconds: number;
  competencies: string[];
  follow_up_context?: string;
}

interface QuickScore {
  overall: number;  // 0-100
  highlight: string;  // Best part of response
  tip: string;  // One improvement
}

interface DimensionScores {
  content: number;
  communication: number;
  technical?: number;
  soft_skills: number;
}

interface ImprovementItem {
  area: string;
  description: string;
  action: string;
  resources: Resource[];
}

interface Resource {
  type: 'video' | 'article' | 'exercise' | 'template';
  title: string;
  url: string;
  duration_minutes?: number;
}

interface InterviewConfig {
  role: string;
  industry: string;
  level: string;
  company_target?: string;
  duration_minutes: number;
  question_count: number;
  question_types: string[];
}
```

---

## Performance y Costo Targets

### Latencia

| Operacion | Target p50 | Target p95 | Max Allowed |
|-----------|------------|------------|-------------|
| Generate question | 800ms | 1.5s | 3s |
| Evaluate response | 1.2s | 2.0s | 4s |
| Final report | 3s | 5s | 10s |
| Audio transcription | 1s | 2s | 5s |

### Costo por Entrevista

| Componente | Texto (MVP) | Audio (V2) | Video (V3) |
|------------|-------------|------------|------------|
| LLM (questions) | $0.02 | $0.02 | $0.02 |
| LLM (evaluation) | $0.03 | $0.03 | $0.03 |
| STT (Whisper) | - | $0.006 | $0.006 |
| TTS (voice) | - | $0.01 | $0.01 |
| Video processing | - | - | $0.03 |
| Infrastructure | $0.01 | $0.015 | $0.025 |
| **TOTAL** | **$0.06** | **$0.08** | **$0.12** |

### Escalabilidad

| Metrica | MVP | 6 meses | 12 meses |
|---------|-----|---------|----------|
| Entrevistas/dia | 500 | 5,000 | 20,000 |
| Concurrent sessions | 50 | 200 | 500 |
| LLM requests/min | 100 | 1,000 | 4,000 |
| Audio hours/dia | 0 | 200 | 800 |

---

## Testing Strategy

### Unit Tests

```python
# Ejemplo de test para scoring engine

def test_scoring_applies_correct_weights():
    # Given
    raw_scores = {
        "content": 80,
        "communication": 70,
        "technical": 90,
        "soft_skills": 75
    }
    role = "senior_engineer"

    # When
    result = scoring_engine.calculate_weighted_score(raw_scores, role)

    # Then
    # Senior Eng weights: content=25%, comm=20%, tech=40%, soft=15%
    expected = 0.25*80 + 0.20*70 + 0.40*90 + 0.15*75
    assert result == expected  # 81.25

def test_feedback_is_specific_not_generic():
    # Given
    response = "Trabaje en un proyecto importante y lo termine"

    # When
    feedback = evaluator.generate_feedback(response, question, cv)

    # Then
    assert "proyecto importante" in feedback.improvement  # Specific reference
    assert "cuantifica" in feedback.tip.lower()  # Actionable advice
    assert len(feedback.highlight) > 20  # Not just "buen trabajo"
```

### Integration Tests

```python
def test_full_interview_flow():
    # 1. Create session
    session = client.post("/interviews", json={
        "role": "Product Manager",
        "industry": "tech",
        "level": "mid"
    })
    assert session.status_code == 201
    session_id = session.json()["session_id"]

    # 2. Answer questions
    for i in range(5):
        response = client.post(f"/interviews/{session_id}/respond", json={
            "response_text": f"Example response {i}"
        })
        assert response.status_code == 200
        assert "score" in response.json()

    # 3. Complete interview
    final = client.post(f"/interviews/{session_id}/complete")
    assert final.status_code == 200
    assert 0 <= final.json()["final_score"] <= 100
    assert len(final.json()["strengths"]) >= 1
    assert len(final.json()["improvements"]) >= 1
```

### Evaluation Benchmark

| Metrica | Target | Medicion |
|---------|--------|----------|
| Score correlation vs human | > 0.85 | 200 respuestas evaluadas |
| Feedback relevance | > 4.0/5 | User survey post-interview |
| Question relevance | > 4.5/5 | "Esta pregunta fue relevante?" |
| STAR detection accuracy | > 90% | vs manual annotation |

---

## Security y Privacy

### Datos Sensibles

| Dato | Clasificacion | Retencion | Encriptacion |
|------|---------------|-----------|--------------|
| CV contenido | PII | 1 ano | AES-256 at rest |
| Respuestas texto | PII | 1 ano | AES-256 at rest |
| Audio grabaciones | PII | 90 dias | AES-256 at rest |
| Video grabaciones | PII Sensible | 30 dias | AES-256 at rest |
| Scores | Anonimizable | 2 anos | Standard |

### Compliance

- **Ley 29733 (Peru)**: Consentimiento explicito para CV y grabaciones
- **GDPR-like**: Derecho a eliminacion, exportacion de datos
- **Data minimization**: Solo recolectar lo necesario

### Guardrails de IA

```python
# Validacion de outputs de LLM

FORBIDDEN_PATTERNS = [
    r"discrimin",  # No discriminar
    r"edad|genero|raza",  # No mencionar caracteristicas protegidas
    r"embaraz",  # No preguntar por embarazo
    r"religion|politica",  # Temas sensibles
]

def validate_question(question: str) -> bool:
    for pattern in FORBIDDEN_PATTERNS:
        if re.search(pattern, question, re.IGNORECASE):
            logger.warning(f"Blocked question with pattern: {pattern}")
            return False
    return True
```

---

## Rollout Plan

### Fase 1: Alpha (Semanas 1-4)

- Equipo interno + 20 beta testers
- Solo modo texto
- 50 preguntas behavioral
- Feedback manual para calibracion

### Fase 2: Beta Cerrada (Semanas 5-8)

- 200 usuarios invitados
- Expansion a 150 preguntas
- Scoring automatizado calibrado
- Metricas de engagement

### Fase 3: Beta Abierta (Semanas 9-12)

- Waitlist abierta
- Modo audio (V2) en beta
- 210+ preguntas (target MVP)
- Pricing validation

### Fase 4: GA (Semana 13+)

- Launch publico
- Todos los modos activos
- Planes de suscripcion
- B2B pilots

---

## Metricas de Exito

### Product Metrics

| Metrica | Target MVP | Target 6 meses |
|---------|------------|----------------|
| Interviews completed/day | 50 | 500 |
| Completion rate | > 70% | > 80% |
| Return rate (2+ interviews) | > 40% | > 50% |
| NPS | > 30 | > 50 |
| Avg score improvement | +8 pts | +12 pts |

### Business Metrics

| Metrica | Target MVP | Target 6 meses |
|---------|------------|----------------|
| Free users | 1,000 | 10,000 |
| Paid conversion | 5% | 10% |
| Cost per interview | < $0.08 | < $0.05 |
| Revenue per user | $5 | $15 |

---

## Consecuencias

### Positivas

1. **Diferenciacion clara**: Primera solucion de practica con IA en espanol peruano
2. **Escalabilidad**: Arquitectura permite 20K+ entrevistas/dia
3. **Costo controlado**: $0.06/entrevista permite free tier sostenible
4. **Mejora continua**: Datos de sesiones alimentan optimizacion del modelo

### Negativas

1. **Dependencia de LLM providers**: Riesgo de cambios de pricing/disponibilidad
2. **Complejidad de calibracion**: Requiere iteracion continua para accuracy
3. **Expectativas de usuarios**: IA puede no reemplazar 100% feedback humano

### Riesgos Residuales

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| LLM hallucinations | Media | Alto | Guardrails, human review |
| Latency spikes | Media | Medio | Circuit breaker, fallback models |
| Scoring bias | Baja | Alto | Diverse test set, fairness audits |
| Data breach | Baja | Critico | Encryption, pen testing, SOC 2 |

---

## Referencias

1. [STAR Method for Behavioral Interviews](https://www.themuse.com/advice/star-interview-method)
2. [OpenAI Pricing](https://openai.com/pricing)
3. [Anthropic Claude Documentation](https://docs.anthropic.com)
4. [Whisper Speech Recognition](https://openai.com/research/whisper)
5. [Peru Labor Law - Ley 29733](https://www.gob.pe/institucion/mtpe/normas-legales)

---

## Historial de Decisiones

| Version | Fecha | Cambio | Autor |
|---------|-------|--------|-------|
| 1.0 | 2026-01-31 | Documento inicial | System Architect |

---

*ADR generado para Entrevistador Inteligente Peru - Motor de Simulacion de Entrevistas*
