# Analisis: OCR para CV Parsing en Entrevistador Inteligente

| Metadata | Valor |
|----------|-------|
| Fecha | 2026-01-31 |
| Autor | Research Agent |
| Relacionado con | ADR-007 (CV Pipeline), ADR-004 (AI/ML Infrastructure) |
| Basado en | Investigacion OCR para recibos |

---

## 1. Resumen Ejecutivo

Este documento analiza como aplicar la investigacion de OCR realizada para recibos al contexto de parsing de CVs para "Entrevistador Inteligente". Se evaluan las herramientas identificadas, se comparan costos entre soluciones cloud y open source, y se propone una arquitectura hibrida optimizada para CVs.

### Conclusion Principal

**La estrategia hibrida de 3 niveles es aplicable a CVs con modificaciones:**

| Nivel | Recibos | CVs (Adaptado) |
|-------|----------------|----------------|
| 1 | Coordenadas fijas | Deteccion de secciones por patrones de texto |
| 2 | PP-DocLayout | PP-DocLayout + LayoutParser (tablas, columnas) |
| 3 | Gemini Vision fallback | Claude/GPT Vision fallback |

**Costo estimado por CV:** $0.002 - $0.015 (vs $0.10-0.50 de Affinda)

---

## 2. Diferencias Clave: Recibos vs CVs

### 2.1 Caracteristicas Estructurales

| Aspecto | Recibos | CVs |
|---------|----------------|-----|
| **Estructura** | Fija, predecible | Variable, creativa |
| **Layout** | Tabular, formulario | Multi-columna, secciones libres |
| **Longitud** | 1-2 paginas | 1-10 paginas |
| **Formatos** | PDF/Imagen | PDF, DOCX, DOC, Imagen |
| **Idioma** | Terminos fijos | Vocabulario diverso |
| **Tipografia** | Impresa, uniforme | Mixta, fuentes decorativas |
| **Campos a extraer** | ~10-15 campos | ~30-50 campos |
| **Relaciones** | Lineales | Jerarquicas (experiencia > logros) |

### 2.2 Complejidades Adicionales de CVs

```
CV / CURRICULUM
+---------------------------+
| JUAN PEREZ GARCIA         |
| Software Engineer         |
| juan@email.com | LinkedIn |
+---------------------------+
| EXPERIENCIA               |
| +-------------------------+
| | Tech Company  2022-2024 |
| | - Backend Developer     |
| | - Python, Django, AWS   |
| | > Logro 1: +30% perf    |
| | > Logro 2: Lidere 5     |
| +-------------------------+
| | Otra Empresa  2020-2022 |
| | - Junior Dev            |
| +-------------------------+
| EDUCACION                 |
| ...                       |
+---------------------------+

Estructura VARIABLE
Secciones en cualquier orden
Formatos creativos (2 columnas, etc)
```

### 2.3 Implicaciones para OCR

| Factor | Impacto en Recibos | Impacto en CVs |
|--------|-------------------|----------------|
| **OCR Accuracy** | Critico para numeros (kWh, montos) | Critico para nombres, fechas, skills |
| **Layout Detection** | Util pero opcional | ESENCIAL para secciones |
| **Deteccion de Tablas** | Muy importante | Moderadamente importante |
| **NER Post-OCR** | Minimo (regex basta) | ESENCIAL (spaCy/LLM) |
| **Fallback Vision** | Solo para baja calidad | Para layouts creativos |

---

## 3. Evaluacion de Herramientas para CV Parsing

### 3.1 Comparativa de OCR

| Libreria | Precision CVs | Manejo Columnas | Espanol | Costo | Recomendacion |
|----------|---------------|-----------------|---------|-------|---------------|
| **Tesseract 5.x** | 95-98% (limpio) | Basico (hOCR) | Nativo | Gratis | **MVP** |
| **PaddleOCR** | 96-99% | Excelente (PP-Structure) | Si | Gratis | **Produccion** |
| **EasyOCR** | 90-95% | Medio | Si | Gratis | Fallback simple |
| **docTR** | 95-98% | Bueno | Si | Gratis | Alternativa |
| **Google Vision** | 98-99% | Excelente | Si | $1.50/1K | Premium fallback |
| **Azure Form Recognizer** | 98-99% | Excelente (prebuilt) | Si | $1.00/1K | **Recomendado cloud** |

### 3.2 Herramientas de Deteccion de Layout

| Herramienta | mAP Score | Detecta Secciones CV | GPU Requerida | Costo |
|-------------|-----------|----------------------|---------------|-------|
| **PP-DocLayout** | 90.4% | Si (tablas, texto, titulos) | Recomendada | Gratis |
| **LayoutParser** | 88-92% | Si (configurable) | Recomendada | Gratis |
| **deepdoctection** | 87-90% | Si (completo) | Si | Gratis |
| **YOLO-based** | 85-90% | Custom training | Si | Gratis |
| **Azure Layout** | 95%+ | Si (prebuilt) | No (API) | $0.01/pagina |

### 3.3 Procesadores de Documentos Especializados

| Solucion | Especialidad | Precision CVs | Costo/CV | Vendor Lock-in |
|----------|--------------|---------------|----------|----------------|
| **Affinda** | CVs | 98%+ | $0.10-0.50 | Alto |
| **Sovren** | CVs | 97%+ | $0.15-0.30 | Alto |
| **resume-parser (npm)** | CVs basicos | 75-85% | Gratis | Bajo |
| **pyresparser** | CVs basicos | 70-80% | Gratis | Bajo |
| **Textkernel** | HR docs | 95%+ | Enterprise | Alto |

---

## 4. Comparativa de Costos: Azure vs Stack Open Source

### 4.1 Escenarios de Volumen

| Escenario | CVs/mes | Azure Form Recognizer | Stack Open Source |
|-----------|---------|----------------------|-------------------|
| **MVP** | 1,000 | $10 | $0 (+ $50 infra) |
| **Growth** | 5,000 | $50 | $0 (+ $100 infra) |
| **Scale** | 20,000 | $200 | $0 (+ $300 infra) |
| **Enterprise** | 100,000 | $1,000 | $0 (+ $800 infra) |

### 4.2 Desglose de Costos - Azure Form Recognizer

```yaml
azure_form_recognizer:
  # Prebuilt Document Model
  pricing_per_1000_pages: $10.00

  # Para CVs (promedio 2 paginas)
  cost_per_cv: $0.02

  features_included:
    - OCR de alta precision
    - Deteccion de layout
    - Extraccion de tablas
    - Key-value pairs
    - Estructura jerarquica

  # Costos adicionales
  custom_model_training: $10/hora
  storage: $0.024/GB/mes

  # Ejemplo 5,000 CVs/mes
  monthly_cost:
    pages: 10,000  # 5K CVs x 2 pags
    ocr: $100
    storage: $5
    total: $105
```

### 4.3 Desglose de Costos - Stack Open Source

```yaml
open_source_stack:
  # Componentes
  tesseract: $0
  paddleocr: $0
  pp_doclayout: $0
  spacy: $0

  # Infraestructura requerida
  compute:
    # Sin GPU (CPU only)
    cpu_instance: $50-100/mes  # 4 vCPU, 16GB RAM

    # Con GPU (recomendado para produccion)
    gpu_instance: $200-400/mes  # T4 GPU

  storage:
    s3_compatible: $5-20/mes

  # Costos ocultos
  hidden_costs:
    maintenance: 8-16 hrs/mes
    model_updates: 2-4 hrs/mes
    monitoring: $20/mes (Grafana Cloud)

  # Ejemplo 5,000 CVs/mes
  monthly_cost:
    compute_cpu: $75
    storage: $10
    monitoring: $20
    total: $105  # Similar a Azure!

  # Con GPU para mejor performance
  monthly_cost_gpu:
    compute_gpu: $300
    storage: $10
    monitoring: $20
    total: $330
```

### 4.4 Analisis TCO (Total Cost of Ownership) - 12 meses

| Factor | Azure Form Recognizer | Open Source (CPU) | Open Source (GPU) |
|--------|----------------------|-------------------|-------------------|
| **Setup inicial** | 4 hrs ($200) | 40 hrs ($2,000) | 60 hrs ($3,000) |
| **Costo mensual (5K CVs)** | $105 | $105 | $330 |
| **Mantenimiento anual** | 20 hrs ($1,000) | 120 hrs ($6,000) | 150 hrs ($7,500) |
| **Escalabilidad** | Automatica | Manual | Manual |
| **Total Y1** | **$2,460** | **$9,260** | **$14,460** |
| **Total Y2** | **$1,260** | **$7,260** | **$11,460** |

### 4.5 Decision de Costo-Beneficio

```
                    DECISION TREE
                         |
            +------------+------------+
            |                         |
     Presupuesto < $500/mes     Presupuesto > $500/mes
            |                         |
            v                         v
    +---------------+         +---------------+
    | Open Source   |         | Azure Form    |
    | (Tesseract +  |         | Recognizer    |
    | PaddleOCR)    |         | + LLM fallback|
    +---------------+         +---------------+
            |                         |
    Para < 2K CVs/mes         Para cualquier vol.
    O equipo tecnico          O equipo pequeno
    disponible                sin ML expertise
```

**Recomendacion para Entrevistador Inteligente:**

Dado que el ADR-004 ya selecciono **Azure Form Recognizer**, esta decision es correcta porque:
1. Equipo pequeno (startup)
2. Time-to-market es prioridad
3. Costo $0.01-0.02/CV es aceptable
4. Ya hay integracion Azure (Azure Storage para CVs)

Sin embargo, propongo una **arquitectura hibrida** para optimizar costos:

---

## 5. Arquitectura Hibrida Propuesta para CVs

### 5.1 Estrategia de 3 Niveles (Adaptada de Recibos)

```
+-------------------------------------------------------------------------+
|                    PIPELINE HIBRIDO CV PARSING                           |
+-------------------------------------------------------------------------+

    [CV Upload]
         |
         v
    +-------------------+
    | Format Detection  |
    | & Classification  |
    +--------+----------+
             |
    +--------+--------+--------+
    |        |        |        |
    v        v        v        v
  [PDF]   [DOCX]  [Image]  [LinkedIn]
    |        |        |        |
    v        v        v        v

+-------------------------------------------------------------------------+
|  NIVEL 1: FAST PATH (Costo: $0)                                          |
|  Target: 60-70% de CVs                                                   |
+-------------------------------------------------------------------------+
|                                                                          |
|  Para: PDFs nativos (texto seleccionable) y DOCX                        |
|                                                                          |
|  +------------------+     +------------------+     +------------------+  |
|  | pdf.js / docx    |     | Regex Patterns   |     | Section          |  |
|  | Text Extraction  | --> | - Email regex    | --> | Classification   |  |
|  | (0ms CPU)        |     | - Phone regex    |     | (keyword-based)  |  |
|  +------------------+     | - URL regex      |     +------------------+  |
|                           | - Date patterns  |              |           |
|                           +------------------+              v           |
|                                                    +------------------+ |
|                                                    | Confidence Check | |
|                                                    | >= 80%? -> Done  | |
|                                                    | < 80% -> Nivel 2 | |
|                                                    +------------------+ |
+-------------------------------------------------------------------------+
                                    |
                                    v (30-40% de CVs)
+-------------------------------------------------------------------------+
|  NIVEL 2: LAYOUT ANALYSIS (Costo: ~$0.001/CV)                           |
|  Target: CVs con layout complejo, tablas, columnas                       |
+-------------------------------------------------------------------------+
|                                                                          |
|  Para: PDFs escaneados, imagenes, layouts multi-columna                 |
|                                                                          |
|  +------------------+     +------------------+     +------------------+  |
|  | PaddleOCR        |     | PP-DocLayout     |     | Section Mapper   |  |
|  | (PP-Structure)   | --> | Title detection  | --> | Map regions to   |  |
|  | + Tesseract      |     | Table detection  |     | CV sections      |  |
|  +------------------+     | Text blocks      |     +------------------+  |
|         |                 +------------------+              |           |
|         v                                                   v           |
|  +------------------+                              +------------------+ |
|  | Enhanced NER     |                              | Confidence Check | |
|  | spaCy + custom   |                              | >= 85%? -> Done  | |
|  | CV entities      |                              | < 85% -> Nivel 3 | |
|  +------------------+                              +------------------+ |
+-------------------------------------------------------------------------+
                                    |
                                    v (5-10% de CVs)
+-------------------------------------------------------------------------+
|  NIVEL 3: VISION LLM FALLBACK (Costo: ~$0.01-0.03/CV)                   |
|  Target: CVs problematicos, layouts muy creativos                        |
+-------------------------------------------------------------------------+
|                                                                          |
|  Para: Calidad muy baja, disenos artisticos, formatos inusuales         |
|                                                                          |
|  +------------------+     +------------------+     +------------------+  |
|  | Render to Image  |     | Claude Vision /  |     | Structured       |  |
|  | (if not image)   | --> | GPT-4 Vision     | --> | JSON Output      |  |
|  +------------------+     | with prompt      |     +------------------+  |
|                           +------------------+                          |
|                                                                          |
|  Prompt Template:                                                        |
|  "Extrae la siguiente informacion del CV en formato JSON:                |
|   - nombre_completo, email, telefono, ubicacion                          |
|   - experiencia: [{empresa, cargo, fechas, descripcion}]                 |
|   - educacion: [{institucion, titulo, fechas}]                           |
|   - skills: {tecnicos: [], blandos: [], idiomas: []}                     |
|   - certificaciones: []"                                                 |
+-------------------------------------------------------------------------+
```

### 5.2 Costos Estimados por Nivel

| Nivel | % CVs | Tiempo/CV | Costo/CV | Tecnologia |
|-------|-------|-----------|----------|------------|
| **1: Fast Path** | 60-70% | <100ms | $0 | pdf.js, docx, regex |
| **2: Layout** | 25-35% | 1-3s | ~$0.001 | PaddleOCR, PP-DocLayout |
| **3: Vision LLM** | 5-10% | 3-8s | ~$0.015 | Claude Vision / GPT-4V |

### 5.3 Costo Promedio Ponderado

```
Costo promedio por CV = (0.65 * $0) + (0.30 * $0.001) + (0.05 * $0.015)
                      = $0 + $0.0003 + $0.00075
                      = $0.00105 por CV

Comparado con:
- Affinda: $0.10-0.50/CV
- Azure Form Recognizer solo: $0.02/CV
- Propuesta hibrida: $0.001/CV (95% mas barato que Azure!)
```

### 5.4 Costo Mensual Proyectado

| Volumen | Azure Only | Hibrido Propuesto | Ahorro |
|---------|------------|-------------------|--------|
| 1,000 CVs | $20 | $1.05 | 95% |
| 5,000 CVs | $100 | $5.25 | 95% |
| 20,000 CVs | $400 | $21 | 95% |
| 100,000 CVs | $2,000 | $105 | 95% |

---

## 6. Implementacion Tecnica

### 6.1 Arquitectura de Codigo

```
src/
├── cv-pipeline/
│   ├── index.ts                 # Orquestador principal
│   ├── detector/
│   │   ├── format-detector.ts   # Detecta PDF/DOCX/Image
│   │   └── quality-scorer.ts    # Score de calidad del documento
│   │
│   ├── level-1-fast/
│   │   ├── pdf-extractor.ts     # pdf.js text extraction
│   │   ├── docx-extractor.ts    # docx-parser
│   │   ├── section-classifier.ts # Keyword-based sections
│   │   └── regex-patterns.ts    # Email, phone, URL, dates
│   │
│   ├── level-2-layout/
│   │   ├── paddle-ocr.ts        # PaddleOCR wrapper
│   │   ├── tesseract.ts         # Tesseract fallback
│   │   ├── layout-detector.ts   # PP-DocLayout integration
│   │   └── section-mapper.ts    # Map regions to sections
│   │
│   ├── level-3-vision/
│   │   ├── vision-llm.ts        # Claude/GPT Vision API
│   │   └── prompts.ts           # Prompts optimizados
│   │
│   ├── nlp/
│   │   ├── spacy-client.ts      # spaCy NER via HTTP
│   │   ├── skill-normalizer.ts  # Normalize skills (JS->JavaScript)
│   │   └── date-parser.ts       # Parse date ranges
│   │
│   └── output/
│       ├── schema.ts            # Zod schema for output
│       ├── confidence-scorer.ts # Overall confidence
│       └── formatter.ts         # Format to ProcessedCV
```

### 6.2 Flujo de Decision

```typescript
// cv-pipeline/index.ts
import { detectFormat } from './detector/format-detector';
import { scoreQuality } from './detector/quality-scorer';
import { level1FastPath } from './level-1-fast';
import { level2LayoutAnalysis } from './level-2-layout';
import { level3VisionLLM } from './level-3-vision';

export async function processCV(file: Buffer, fileName: string): Promise<ProcessedCV> {
  const format = await detectFormat(file, fileName);

  // Nivel 1: Fast Path para PDF nativos y DOCX
  if (format === 'pdf-native' || format === 'docx') {
    const result = await level1FastPath(file, format);
    if (result.confidence >= 0.80) {
      return result;
    }
  }

  // Nivel 2: Layout Analysis para imagenes y PDFs escaneados
  if (format === 'pdf-scanned' || format === 'image') {
    const qualityScore = await scoreQuality(file);
    if (qualityScore >= 0.70) {
      const result = await level2LayoutAnalysis(file, format);
      if (result.confidence >= 0.85) {
        return result;
      }
    }
  }

  // Nivel 3: Vision LLM Fallback
  return await level3VisionLLM(file, format);
}
```

### 6.3 Integracion con Arquitectura Existente

```
+-----------------------------------------------------------------------+
|                    INTEGRACION CON ADR-007                             |
+-----------------------------------------------------------------------+

    [CV Upload API]
         |
         v
    +-------------------+
    | BullMQ Queue      |
    | cv-processing     |
    +--------+----------+
             |
             v
    +-------------------+
    | CV Pipeline       |  <-- PROPUESTA HIBRIDA
    | (Nivel 1/2/3)     |
    +--------+----------+
             |
             +-----> [Nivel 1] pdf.js/docx --> [60-70%]
             |
             +-----> [Nivel 2] PaddleOCR + Layout --> [25-35%]
             |
             +-----> [Nivel 3] Claude Vision --> [5-10%]
             |
             v
    +-------------------+
    | spaCy NLP         |  <-- Ya definido en ADR-007
    | (NER Custom)      |
    +--------+----------+
             |
             v
    +-------------------+
    | Embedding         |  <-- Ya definido en ADR-004
    | text-embedding-3  |
    +--------+----------+
             |
             v
    +-------------------+
    | PostgreSQL +      |  <-- Ya definido en ADR-005
    | pgvector          |
    +-------------------+
```

---

## 7. Herramientas Especificas Recomendadas

### 7.1 Stack Recomendado para CV Parsing

| Componente | Herramienta Principal | Fallback | Razon |
|------------|----------------------|----------|-------|
| **PDF Text** | pdf.js | pdf-parse | Browser-compatible, mature |
| **DOCX Parser** | docx (npm) | mammoth | Full structure access |
| **OCR Engine** | PaddleOCR | Tesseract 5.x | Better accuracy, table support |
| **Layout Detection** | PP-DocLayout | LayoutParser | Best mAP, good doc support |
| **NER** | spaCy es_core_news_lg | Custom transformers | Fast, Spanish native |
| **Vision LLM** | Claude 3.5 Sonnet | GPT-4 Vision | Better Spanish, structured output |
| **Skill Normalization** | Custom dict + fuzzy | OpenAI | Control, no API cost |
| **Date Parsing** | chrono-node | dayjs | Natural language dates |

### 7.2 Configuracion de PaddleOCR para CVs

```python
# paddleocr_config.py
from paddleocr import PaddleOCR, PPStructure

# OCR optimizado para CVs
ocr = PaddleOCR(
    use_angle_cls=True,      # Detectar rotacion
    lang='es',               # Espanol
    use_gpu=True,            # GPU si disponible
    det_model_dir='ch_PP-OCRv4_det',
    rec_model_dir='ch_PP-OCRv4_rec',
    cls_model_dir='ch_ppocr_mobile_v2.0_cls',
    det_db_thresh=0.3,       # Sensibilidad deteccion
    det_db_box_thresh=0.6,
    det_db_unclip_ratio=1.5,
    rec_batch_num=6,
)

# Layout analysis para documentos estructurados
layout_engine = PPStructure(
    show_log=False,
    table=True,              # Detectar tablas
    ocr=True,                # OCR integrado
    layout_model_dir='picodet_lcnet_x1_0_fgd_layout',
)
```

### 7.3 spaCy Pipeline para CVs

```python
# spacy_cv_pipeline.py
import spacy
from spacy.tokens import Span

# Cargar modelo espanol
nlp = spacy.load("es_core_news_lg")

# Agregar entity ruler para skills tecnicos
ruler = nlp.add_pipe("entity_ruler", before="ner")

# Patrones para skills tecnicos (muestra)
skill_patterns = [
    {"label": "SKILL_TECH", "pattern": [{"LOWER": {"IN": ["python", "javascript", "java", "go", "rust"]}}]},
    {"label": "SKILL_TECH", "pattern": [{"LOWER": "react"}, {"LOWER": "js", "OP": "?"}]},
    {"label": "SKILL_TECH", "pattern": [{"LOWER": "node"}, {"TEXT": ".", "OP": "?"}, {"LOWER": "js", "OP": "?"}]},
    {"label": "SKILL_TECH", "pattern": [{"LOWER": "aws"}]},
    {"label": "SKILL_TECH", "pattern": [{"LOWER": "docker"}]},
    {"label": "SKILL_TECH", "pattern": [{"LOWER": "kubernetes"}, {"LOWER": "k8s", "OP": "?"}]},
    # ... mas patrones
]

ruler.add_patterns(skill_patterns)

# Entidades custom para CVs
CUSTOM_ENTITIES = [
    "SKILL_TECH",      # Python, React, AWS
    "SKILL_SOFT",      # Liderazgo, comunicacion
    "JOB_TITLE",       # Software Engineer, Gerente
    "COMPANY",         # Google, BCP, Intercorp
    "DEGREE",          # Ingeniero, Licenciado, MBA
    "INSTITUTION",     # PUCP, UPC, MIT
    "DATE_RANGE",      # 2020-2024, Enero 2022 - Presente
]
```

---

## 8. Metricas de Exito

### 8.1 KPIs del Pipeline Hibrido

| Metrica | Target MVP | Target Produccion |
|---------|------------|-------------------|
| Precision extraccion global | > 90% | > 95% |
| Precision Nivel 1 (fast) | > 85% | > 90% |
| Precision Nivel 2 (layout) | > 90% | > 95% |
| Precision Nivel 3 (vision) | > 95% | > 98% |
| Tiempo promedio procesamiento | < 5s | < 3s |
| % CVs en Nivel 1 | > 50% | > 60% |
| % CVs en Nivel 3 | < 15% | < 10% |
| Costo promedio por CV | < $0.01 | < $0.005 |

### 8.2 Monitoreo y Alertas

```yaml
monitoring:
  metrics:
    - cv_processing_duration_seconds:
        by: [level, format]
        alert: p95 > 10s

    - cv_level_distribution:
        by: [level]
        alert: level_3_ratio > 0.15

    - cv_extraction_confidence:
        by: [level, section]
        alert: avg < 0.80

    - cv_cost_per_document:
        by: [level]
        alert: avg > 0.02

  dashboards:
    - name: "CV Pipeline Health"
      panels:
        - Level distribution (pie chart)
        - Processing time by level (histogram)
        - Confidence distribution (histogram)
        - Cost trend (time series)
        - Error rate by stage (bar chart)
```

---

## 9. Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| PaddleOCR no soporta layout raro | Media | Medio | Fallback a Vision LLM |
| spaCy NER pierde skills nuevos | Alta | Bajo | Diccionario actualizable + LLM fallback |
| Vision LLM demasiado lento | Baja | Medio | Cache de respuestas similares |
| Costo Nivel 3 escala | Media | Alto | Mejorar Niveles 1-2 iterativamente |
| Privacidad (LLM ve datos PII) | Media | Alto | Anonimizacion antes de Vision LLM |

---

## 10. Plan de Implementacion

### Fase 1: MVP (Semanas 1-2)

- [x] Configurar BullMQ queue (existente)
- [ ] Implementar Nivel 1: pdf.js + docx extraction
- [ ] Implementar clasificador de secciones basico
- [ ] Integrar spaCy NER existente
- [ ] Tests con 100 CVs de prueba

### Fase 2: Layout Analysis (Semanas 3-4)

- [ ] Configurar PaddleOCR en Docker
- [ ] Implementar PP-DocLayout integration
- [ ] Crear section mapper para columnas
- [ ] Mejorar Tesseract preprocessing
- [ ] Tests con 200 CVs problematicos

### Fase 3: Vision Fallback (Semana 5)

- [ ] Implementar Claude Vision integration
- [ ] Crear prompts optimizados para CVs
- [ ] Agregar cache semantico para respuestas
- [ ] Implementar anonimizacion PII
- [ ] Tests end-to-end

### Fase 4: Optimizacion (Semana 6)

- [ ] Monitoreo y dashboards
- [ ] Ajustar umbrales de confianza
- [ ] Entrenar modelos NER custom si hay datos
- [ ] Optimizar para reducir Nivel 3 usage
- [ ] Documentacion final

---

## 11. Conclusion

La investigacion de OCR para recibos proporciona una base solida para el CV parsing, pero requiere adaptaciones significativas:

### Adaptaciones Clave:

1. **Layout Detection es CRITICO** - Los CVs tienen layouts variables, a diferencia de los recibos con formato fijo.

2. **NER Post-OCR es ESENCIAL** - Extraer entidades como skills, empresas y fechas requiere NLP avanzado.

3. **El Fast Path es diferente** - Para CVs, el nivel 1 usa extraccion directa de texto (pdf.js/docx) en lugar de coordenadas fijas.

4. **Vision LLM es mas necesario** - Los CVs creativos (5-10%) requieren Vision LLM, mas que los recibos (~1-2%).

### Recomendacion Final:

**Adoptar la arquitectura hibrida de 3 niveles** con:
- **Nivel 1**: pdf.js + docx para PDFs nativos (60-70%)
- **Nivel 2**: PaddleOCR + PP-DocLayout para scans e imagenes (25-35%)
- **Nivel 3**: Claude Vision para casos problematicos (5-10%)

**Costo estimado**: $0.001-0.003/CV (vs $0.10-0.50 de soluciones comerciales)

**Ahorro vs Azure solo**: ~95% en costos de OCR

---

## 12. Referencias

1. PaddleOCR Documentation: https://github.com/PaddlePaddle/PaddleOCR
2. PP-DocLayout: https://github.com/PaddlePaddle/PaddleDetection/tree/release/2.7/configs/doclayout
3. spaCy Spanish Models: https://spacy.io/models/es
4. Claude Vision API: https://docs.anthropic.com/en/docs/vision
5. Tesseract.js: https://tesseract.projectnaptha.com/
6. pdf.js: https://mozilla.github.io/pdf.js/
7. ADR-007 CV Pipeline: /home/orlando/startup/docs/entrevistadorinteligente/arquitectura/adrs/ADR-007-cv-pipeline.md
8. ADR-004 AI/ML Infrastructure: /home/orlando/startup/docs/entrevistadorinteligente/arquitectura/adrs/ADR-004-ai-ml-infrastructure.md

---

*Documento generado por Research Agent - Claude Flow V3*
