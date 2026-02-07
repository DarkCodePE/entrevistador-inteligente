# ADR-007: Pipeline de Procesamiento de CV

| Metadata | Value |
|----------|-------|
| Estado | Actualizado |
| Fecha | 2026-01-31 |
| Autores | Equipo de Arquitectura |
| Revisores | - |
| Relacionado con | ADR-005 (Almacenamiento), ADR-006 (AI/ML) |
| Revision | 2.0 - Pipeline OCR Hibrido |

## Contexto

"Entrevistador Inteligente" requiere un pipeline robusto para procesar CVs en multiples formatos y extraer informacion estructurada. El sistema debe soportar:

- Formatos diversos: PDF, DOCX, DOC, imagenes (fotos de CV)
- Documentos escaneados que requieren OCR
- Extraccion de entidades clave (nombre, contacto, experiencia, educacion, skills)
- Normalizacion de terminologia tecnica
- Generacion de embeddings para matching semantico
- Scoring de compatibilidad con sistemas ATS
- Sugerencias de optimizacion para el candidato

### Flujo del Pipeline (Actualizado v2.0)

```
Upload CV (PDF/DOCX/Image)
       |
       v
+--------------------------------------+
| NIVEL 1: Procesamiento Open Source   |
| -------------------------------------|
| 1. pdf2image / python-docx           |
| 2. PaddleOCR (texto espanol)         |
| 3. PP-DocLayout (secciones)          |
| 4. spaCy (NER: nombres, fechas)      |
|                                      |
| Costo: $0 | Latencia: <3s            |
+--------------------------------------+
       |
       v (si confianza < 85%)
+--------------------------------------+
| NIVEL 2: Cloud Vision Fallback       |
| -------------------------------------|
| Google Cloud Vision API              |
|                                      |
| Costo: $0.0015/img | Latencia: <2s   |
+--------------------------------------+
       |
       v (si aun falla)
+--------------------------------------+
| NIVEL 3: LLM Vision Fallback         |
| -------------------------------------|
| GPT-4 Vision / Gemini 1.5 Pro        |
| Prompt: "Extract structured CV data" |
|                                      |
| Costo: $0.01/img | Latencia: <5s     |
+--------------------------------------+
       |
       v
+--------------------------------------+
| Post-Procesamiento                   |
| -------------------------------------|
| - Normalizacion de skills            |
| - Generacion de embeddings           |
| - Calculo de ATS Score               |
| - Sugerencias de mejora              |
+--------------------------------------+
       |
       v
Datos Estructurados --> Embeddings --> PostgreSQL + pgvector
```

### Requisitos de Performance

| Metrica | Target |
|---------|--------|
| Tiempo de procesamiento completo | < 30 segundos |
| Accuracy en extraccion de datos | > 95% |
| Tamano maximo de documento | 10 paginas |
| Formatos soportados | PDF, DOCX, DOC, PNG, JPG, JPEG |

## Decisiones

### Decision 1: Parser de CV (Actualizado v2.0)

**Opciones evaluadas:**

| Opcion | Pros | Contras | Costo |
|--------|------|---------|-------|
| **Affinda** | Alta precision (98%+), API lista, soporta 100+ idiomas | Costo por documento ($0.10-0.50/CV), vendor lock-in | Alto |
| **AWS Textract** | Integracion AWS, buena para formularios | Requiere post-procesamiento, menos especializado en CVs | Medio |
| **resume-parser (OSS)** | Gratuito, control total, personalizable | Menor precision inicial, requiere mantenimiento | Bajo |
| **Hybrid (OSS + LLM)** | Flexibilidad, precision mejorable, control total | Complejidad de implementacion | Medio |
| **Hybrid 3-Tier (PaddleOCR + Cloud + LLM)** | Optimo costo/precision, espanol nativo, escalable | Complejidad de implementacion | Bajo-Medio |

**Decision: Hybrid 3-Tier (PaddleOCR + Google Cloud Vision + LLM Vision)**

**Justificacion:**
- **Nivel 1 gratuito**: PaddleOCR + PP-DocLayout eliminan costos de OCR en 85%+ de casos
- **Soporte nativo espanol**: PaddleOCR tiene modelos optimizados para espanol latinoamericano
- **Layout intelligence**: PP-DocLayout detecta secciones automaticamente (experiencia, educacion, skills)
- **Fallback progresivo**: Solo escala a servicios pagos cuando es necesario
- **Control total**: Sin vendor lock-in en la capa primaria

**Arquitectura del parser hibrido:**

```
                    +------------------+
                    |  Documento Raw   |
                    | (PDF/DOCX/Image) |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Conversion       |
                    | pdf2image /      |
                    | python-docx      |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | NIVEL 1:         |
                    | PaddleOCR +      |
                    | PP-DocLayout     |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
              v                             v
     +----------------+            +----------------+
     | Confidence     |            | Confidence     |
     | >= 85%         |            | < 85%          |
     +-------+--------+            +-------+--------+
             |                             |
             v                             v
     +----------------+            +----------------+
     | Usar resultado |            | NIVEL 2:       |
     | directo        |            | Google Cloud   |
     | Costo: $0      |            | Vision API     |
     +----------------+            +-------+--------+
                                           |
                            +--------------+--------------+
                            |                             |
                            v                             v
                   +----------------+            +----------------+
                   | Confidence     |            | Confidence     |
                   | >= 85%         |            | < 85%          |
                   +-------+--------+            +-------+--------+
                           |                             |
                           v                             v
                   +----------------+            +----------------+
                   | Usar resultado |            | NIVEL 3:       |
                   | Costo: $0.0015 |            | LLM Vision     |
                   +----------------+            | (GPT-4V/Gemini)|
                                                | Costo: $0.01   |
                                                +----------------+
```

**Tabla de Costos por Nivel:**

| Nivel | Tecnologia | Costo/Imagen | Latencia | Precision Esperada |
|-------|------------|--------------|----------|-------------------|
| 1 | PaddleOCR + PP-DocLayout | $0 | <3s | 85-95% |
| 2 | Google Cloud Vision | $0.0015 | <2s | 95-98% |
| 3 | GPT-4 Vision / Gemini 1.5 Pro | $0.01 | <5s | 98-99% |

**Estimacion de distribucion de procesamiento:**

| Nivel | % de CVs procesados | Costo promedio |
|-------|---------------------|----------------|
| Nivel 1 (Open Source) | 70-80% | $0 |
| Nivel 2 (Cloud Vision) | 15-25% | $0.0015 |
| Nivel 3 (LLM Vision) | 2-5% | $0.01 |
| **Promedio ponderado** | - | **~$0.0006/CV** |

---

### Decision 2: Motor de OCR (Actualizado v2.0)

**Opciones evaluadas:**

| Opcion | Pros | Contras | Costo | Precision Espanol |
|--------|------|---------|-------|-------------------|
| **Tesseract** | Open source, amplio soporte | Precision variable, pre-procesamiento necesario | Gratuito | Media (75-85%) |
| **Google Vision** | Alta precision (99%+), handwriting support | Costo por imagen, latencia de red | $1.50/1000 imgs | Alta (95%+) |
| **Azure OCR** | Buena precision, integracion Azure | Costo similar a Google, menos features | $1.00/1000 imgs | Alta (93%+) |
| **PaddleOCR** | Open source, modelos espanol optimizados, deteccion de layout | Requiere mas memoria, GPU opcional | Gratuito | Alta (90-95%) |
| **PP-DocLayout** | Deteccion de secciones de documentos | Solo layout, no OCR | Gratuito | N/A |

**Decision: PaddleOCR + PP-DocLayout (Nivel 1) con fallbacks progresivos**

**Justificacion:**
- **PaddleOCR supera a Tesseract** en precision para espanol latinoamericano
- **PP-DocLayout** permite segmentacion inteligente de secciones del CV
- **Costo cero** en la capa primaria (70-80% de documentos)
- **Deteccion de angulos** automatica sin pre-procesamiento manual
- **GPU opcional**: funciona en CPU con rendimiento aceptable

**Implementacion del Parser Hibrido:**

```python
from paddleocr import PaddleOCR
from paddlex import create_model
import cv2
import numpy as np
from pdf2image import convert_from_path
from google.cloud import vision
import openai
from dataclasses import dataclass
from typing import Optional, Tuple, Literal
from enum import Enum

class OCRLevel(Enum):
    OPENSOURCE = "opensource"
    CLOUD_VISION = "cloud_vision"
    LLM_VISION = "llm_vision"

@dataclass
class OCRResult:
    text: str
    confidence: float
    sections: dict
    level_used: OCRLevel
    processing_time_ms: float

class HybridCVParser:
    """
    Parser de CV hibrido con 3 niveles de fallback.
    Optimiza costo y precision automaticamente.
    """

    def __init__(self, confidence_threshold: float = 0.85):
        # Nivel 1: Open Source
        self.ocr = PaddleOCR(
            lang='es',              # Espanol
            use_angle_cls=True,     # Detectar rotacion
            use_gpu=False,          # CPU por defecto
            show_log=False
        )
        self.layout_model = create_model('PP-DocLayout_plus-L')

        # Configuracion
        self.confidence_threshold = confidence_threshold

        # Nivel 2: Google Cloud Vision (lazy init)
        self._vision_client = None

    @property
    def vision_client(self):
        if self._vision_client is None:
            self._vision_client = vision.ImageAnnotatorClient()
        return self._vision_client

    def parse(self, file_path: str) -> Tuple[OCRResult, dict]:
        """
        Procesa un CV con fallback automatico entre niveles.

        Args:
            file_path: Ruta al archivo (PDF, DOCX, o imagen)

        Returns:
            Tuple[OCRResult, dict]: Resultado OCR y datos estructurados
        """
        import time
        start_time = time.time()

        # Convertir a imagen si es necesario
        images = self._convert_to_images(file_path)

        # Nivel 1: Open Source (PaddleOCR + PP-DocLayout)
        result = self._parse_opensource(images)
        if result.confidence >= self.confidence_threshold:
            result.processing_time_ms = (time.time() - start_time) * 1000
            return result, self._extract_structured_data(result)

        # Nivel 2: Google Cloud Vision
        result = self._parse_cloud_vision(images)
        if result.confidence >= self.confidence_threshold:
            result.processing_time_ms = (time.time() - start_time) * 1000
            return result, self._extract_structured_data(result)

        # Nivel 3: LLM Vision (GPT-4V / Gemini)
        result = self._parse_llm_vision(images)
        result.processing_time_ms = (time.time() - start_time) * 1000
        return result, self._extract_structured_data(result)

    def _convert_to_images(self, file_path: str) -> list:
        """Convierte PDF/DOCX a lista de imagenes."""
        if file_path.lower().endswith('.pdf'):
            return convert_from_path(file_path, dpi=300)
        elif file_path.lower().endswith(('.docx', '.doc')):
            # Usar libreoffice para convertir a PDF primero
            import subprocess
            subprocess.run([
                'libreoffice', '--headless', '--convert-to', 'pdf',
                '--outdir', '/tmp', file_path
            ])
            pdf_path = file_path.rsplit('.', 1)[0] + '.pdf'
            return convert_from_path(f'/tmp/{pdf_path.split("/")[-1]}', dpi=300)
        else:
            # Ya es imagen
            return [cv2.imread(file_path)]

    def _parse_opensource(self, images: list) -> OCRResult:
        """Nivel 1: PaddleOCR + PP-DocLayout."""
        all_text = []
        all_confidences = []
        sections = {}

        for img in images:
            # Convertir PIL a numpy si es necesario
            if hasattr(img, 'convert'):
                img_array = np.array(img)
            else:
                img_array = img

            # OCR con PaddleOCR
            ocr_result = self.ocr.ocr(img_array, cls=True)

            if ocr_result and ocr_result[0]:
                for line in ocr_result[0]:
                    text = line[1][0]
                    confidence = line[1][1]
                    all_text.append(text)
                    all_confidences.append(confidence)

            # Deteccion de layout con PP-DocLayout
            layout_result = self.layout_model.predict(img_array)
            for item in layout_result.get('boxes', []):
                section_type = item.get('label', 'unknown')
                if section_type not in sections:
                    sections[section_type] = []
                sections[section_type].append(item)

        avg_confidence = np.mean(all_confidences) if all_confidences else 0.0

        return OCRResult(
            text='\n'.join(all_text),
            confidence=avg_confidence,
            sections=sections,
            level_used=OCRLevel.OPENSOURCE,
            processing_time_ms=0  # Se actualiza despues
        )

    def _parse_cloud_vision(self, images: list) -> OCRResult:
        """Nivel 2: Google Cloud Vision API."""
        from google.cloud.vision_v1 import types

        all_text = []
        all_confidences = []

        for img in images:
            # Convertir a bytes
            if hasattr(img, 'tobytes'):
                import io
                buffer = io.BytesIO()
                img.save(buffer, format='PNG')
                content = buffer.getvalue()
            else:
                _, encoded = cv2.imencode('.png', img)
                content = encoded.tobytes()

            image = types.Image(content=content)
            response = self.vision_client.document_text_detection(image=image)

            if response.full_text_annotation:
                all_text.append(response.full_text_annotation.text)
                # Calcular confianza promedio
                for page in response.full_text_annotation.pages:
                    for block in page.blocks:
                        all_confidences.append(block.confidence)

        avg_confidence = np.mean(all_confidences) if all_confidences else 0.0

        return OCRResult(
            text='\n'.join(all_text),
            confidence=avg_confidence,
            sections={},  # Cloud Vision no detecta secciones
            level_used=OCRLevel.CLOUD_VISION,
            processing_time_ms=0
        )

    def _parse_llm_vision(self, images: list) -> OCRResult:
        """Nivel 3: LLM Vision (GPT-4V o Gemini 1.5 Pro)."""
        import base64
        import io

        # Convertir primera imagen a base64
        img = images[0]
        if hasattr(img, 'tobytes'):
            buffer = io.BytesIO()
            img.save(buffer, format='PNG')
            img_base64 = base64.b64encode(buffer.getvalue()).decode()
        else:
            _, encoded = cv2.imencode('.png', img)
            img_base64 = base64.b64encode(encoded).decode()

        # Prompt optimizado para extraccion de CV
        prompt = """Analiza esta imagen de CV y extrae toda la informacion estructurada.

        Responde en formato JSON con las siguientes secciones:
        {
            "nombre_completo": "",
            "email": "",
            "telefono": "",
            "ubicacion": "",
            "resumen_profesional": "",
            "experiencia": [
                {
                    "empresa": "",
                    "cargo": "",
                    "fecha_inicio": "",
                    "fecha_fin": "",
                    "descripcion": ""
                }
            ],
            "educacion": [
                {
                    "institucion": "",
                    "titulo": "",
                    "fecha": ""
                }
            ],
            "habilidades": [],
            "idiomas": [],
            "certificaciones": []
        }

        Extrae TODA la informacion visible. Si algo no esta claro, indica [ilegible]."""

        response = openai.chat.completions.create(
            model="gpt-4-vision-preview",
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": prompt},
                        {
                            "type": "image_url",
                            "image_url": {
                                "url": f"data:image/png;base64,{img_base64}"
                            }
                        }
                    ]
                }
            ],
            max_tokens=4096
        )

        return OCRResult(
            text=response.choices[0].message.content,
            confidence=0.95,  # LLM siempre alta confianza
            sections={},
            level_used=OCRLevel.LLM_VISION,
            processing_time_ms=0
        )

    def _extract_structured_data(self, result: OCRResult) -> dict:
        """Extrae datos estructurados del texto OCR usando spaCy."""
        import spacy

        nlp = spacy.load('es_core_news_lg')
        doc = nlp(result.text)

        structured = {
            'nombres': [],
            'emails': [],
            'telefonos': [],
            'fechas': [],
            'organizaciones': [],
            'ubicaciones': []
        }

        for ent in doc.ents:
            if ent.label_ == 'PER':
                structured['nombres'].append(ent.text)
            elif ent.label_ == 'ORG':
                structured['organizaciones'].append(ent.text)
            elif ent.label_ == 'LOC':
                structured['ubicaciones'].append(ent.text)
            elif ent.label_ == 'DATE':
                structured['fechas'].append(ent.text)

        # Extraer emails y telefonos con regex
        import re
        emails = re.findall(r'[\w\.-]+@[\w\.-]+\.\w+', result.text)
        phones = re.findall(r'[\+]?[\d\s\-\(\)]{10,}', result.text)

        structured['emails'] = emails
        structured['telefonos'] = [p.strip() for p in phones]

        return structured
```

**Pipeline de pre-procesamiento (opcional para Nivel 1):**

```python
def preprocess_for_ocr(image):
    """Pre-procesamiento opcional para imagenes de baja calidad."""
    import cv2

    # 1. Convertir a escala de grises
    if len(image.shape) == 3:
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    else:
        gray = image

    # 2. Deskew - corregir inclinacion
    coords = np.column_stack(np.where(gray > 0))
    angle = cv2.minAreaRect(coords)[-1]
    if angle < -45:
        angle = 90 + angle
    (h, w) = gray.shape[:2]
    center = (w // 2, h // 2)
    M = cv2.getRotationMatrix2D(center, angle, 1.0)
    gray = cv2.warpAffine(gray, M, (w, h),
                          flags=cv2.INTER_CUBIC,
                          borderMode=cv2.BORDER_REPLICATE)

    # 3. Binarizacion adaptativa
    binary = cv2.adaptiveThreshold(
        gray, 255,
        cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
        cv2.THRESH_BINARY, 11, 2
    )

    # 4. Eliminacion de ruido
    denoised = cv2.fastNlMeansDenoising(binary, None, 10, 7, 21)

    # 5. Mejora de contraste (CLAHE)
    clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
    enhanced = clahe.apply(gray)

    return enhanced
```

---

### Decision 3: Motor de NLP

**Opciones evaluadas:**

| Opcion | Pros | Contras | Costo |
|--------|------|---------|-------|
| **spaCy** | Rapido, eficiente, bueno para NER | Modelos pre-entrenados limitados para CVs | Gratuito |
| **Hugging Face** | Modelos especializados, comunidad activa | Requiere GPU para mejor performance | Gratuito/GPU |
| **OpenAI API** | Alta precision, zero-shot, multilingue | Costo por token, latencia | $0.002/1K tokens |
| **spaCy + Custom NER** | Control total, optimizado para dominio | Requiere datos de entrenamiento | Gratuito |

**Decision: spaCy con modelos custom + OpenAI fallback**

**Justificacion:**
- spaCy es extremadamente rapido para NER local
- Entrenamiento de modelos custom para skills tecnicos
- OpenAI como fallback para contextos ambiguos
- Balance optimo entre costo y precision

**Entidades a extraer:**

```yaml
entities:
  - PERSON_NAME      # Nombre del candidato
  - EMAIL            # Correo electronico
  - PHONE            # Telefono
  - LOCATION         # Ubicacion
  - COMPANY          # Empresas donde trabajo
  - JOB_TITLE        # Titulos de puesto
  - EDUCATION        # Instituciones educativas
  - DEGREE           # Titulos academicos
  - SKILL_TECHNICAL  # Habilidades tecnicas
  - SKILL_SOFT       # Habilidades blandas
  - LANGUAGE         # Idiomas
  - CERTIFICATION    # Certificaciones
  - DATE_RANGE       # Periodos de tiempo
```

---

### Decision 4: Modelo de Embeddings

**Opciones evaluadas:**

| Opcion | Dimensiones | Costo | Latencia | Precision |
|--------|-------------|-------|----------|-----------|
| **OpenAI text-embedding-3-small** | 1536 | $0.02/1M tokens | ~100ms | Alta |
| **OpenAI text-embedding-3-large** | 3072 | $0.13/1M tokens | ~150ms | Muy Alta |
| **sentence-transformers (local)** | 384-768 | Gratuito | ~50ms | Media-Alta |
| **Cohere embed-v3** | 1024 | $0.10/1M tokens | ~100ms | Alta |

**Decision: OpenAI text-embedding-3-small**

**Justificacion:**
- Excelente balance precision/costo
- 1536 dimensiones suficientes para matching semantico
- Consistencia con otros componentes del sistema
- Facil integracion y mantenimiento

**Estrategia de embedding:**

```yaml
embedding_strategy:
  # Embeddings separados por seccion para matching granular
  sections:
    - skills_embedding      # Solo skills
    - experience_embedding  # Experiencia laboral
    - education_embedding   # Educacion
    - full_cv_embedding     # CV completo

  # Normalizacion antes de embedding
  preprocessing:
    - lowercase
    - remove_stopwords
    - skill_normalization
    - acronym_expansion
```

---

### Decision 5: Modelo de Procesamiento

**Opciones evaluadas:**

| Opcion | Pros | Contras | Complejidad |
|--------|------|---------|-------------|
| **Sincrono** | Simple, facil debugging | Bloquea UI, timeout en archivos grandes | Baja |
| **Async (Queue)** | No bloquea, escalable, retry | Complejidad, necesita workers | Media |
| **Hybrid** | Rapido para archivos simples, robusto para complejos | Logica condicional | Media |

**Decision: Async con Queue (Bull/BullMQ)**

**Justificacion:**
- Permite procesar CVs pesados sin bloquear
- Escalabilidad horizontal con workers
- Reintentos automaticos en fallos
- Visibilidad del estado del procesamiento

**Arquitectura de procesamiento:**

```
+-------------+     +------------------+     +------------------+
|   Cliente   |     |   API Gateway    |     |   Redis Queue    |
|   Upload CV | --> |   /api/cv/upload | --> |   cv-processing  |
+-------------+     +------------------+     +--------+---------+
                                                      |
                           +--------------------------|
                           |                          |
                           v                          v
                    +-------------+            +-------------+
                    |  Worker 1   |            |  Worker 2   |
                    |  (OCR+Parse)|            |  (NLP+Embed)|
                    +------+------+            +------+------+
                           |                          |
                           v                          v
                    +------------------------------------------+
                    |              PostgreSQL                   |
                    |  (Resultados + pgvector para embeddings)  |
                    +------------------------------------------+
                                      |
                                      v
                    +------------------------------------------+
                    |           WebSocket Notification          |
                    |         (Notifica al cliente)             |
                    +------------------------------------------+
```

---

## Normalizacion de Skills

### Diccionario de Normalizacion

```json
{
  "skill_aliases": {
    "javascript": ["js", "ecmascript", "es6", "es2015", "es2020"],
    "typescript": ["ts"],
    "python": ["py", "python3", "python2"],
    "react": ["reactjs", "react.js", "react js"],
    "node.js": ["node", "nodejs"],
    "postgresql": ["postgres", "psql", "pg"],
    "mongodb": ["mongo"],
    "amazon web services": ["aws", "amazon aws"],
    "google cloud platform": ["gcp", "google cloud"],
    "kubernetes": ["k8s"],
    "docker": ["docker-compose", "dockerfile"],
    "machine learning": ["ml"],
    "artificial intelligence": ["ai"],
    "continuous integration": ["ci", "ci/cd"],
    "continuous deployment": ["cd"]
  },
  "skill_categories": {
    "programming_languages": ["javascript", "python", "java", "go", "rust"],
    "frameworks": ["react", "angular", "vue", "django", "fastapi"],
    "databases": ["postgresql", "mongodb", "mysql", "redis"],
    "cloud": ["aws", "gcp", "azure"],
    "devops": ["docker", "kubernetes", "terraform", "jenkins"]
  }
}
```

---

## ATS Score Algorithm

### Factores de Puntuacion

```yaml
ats_scoring:
  # Peso de cada factor (total = 100)
  factors:
    keyword_match: 35        # Match con job description
    formatting: 20           # Formato limpio, parseable
    section_completeness: 15 # Secciones requeridas presentes
    experience_relevance: 15 # Experiencia relevante al puesto
    education_match: 10      # Educacion requerida
    certifications: 5        # Certificaciones relevantes

  # Penalizaciones
  penalties:
    - tables_detected: -10   # Tablas dificultan parsing
    - images_in_cv: -5       # Imagenes no parseables
    - missing_contact: -15   # Sin info de contacto
    - no_dates: -10          # Experiencia sin fechas
    - inconsistent_format: -5

  # Bonus
  bonuses:
    - quantified_achievements: +5  # Logros con numeros
    - linkedin_url: +2
    - github_url: +3
    - portfolio_url: +2
```

### Sugerencias de Mejora

```yaml
improvement_suggestions:
  categories:
    - formatting:
        - "Evitar uso de tablas para mejor compatibilidad ATS"
        - "Usar formato cronologico inverso"
        - "Incluir fechas en todas las experiencias"

    - content:
        - "Agregar logros cuantificables (%, $, numeros)"
        - "Incluir palabras clave del puesto objetivo"
        - "Expandir seccion de skills tecnicos"

    - missing_sections:
        - "Agregar resumen profesional"
        - "Incluir informacion de contacto completa"
        - "Agregar seccion de certificaciones"

    - optimization:
        - "Reducir longitud a 2 paginas maximo"
        - "Usar verbos de accion al inicio de bullets"
        - "Remover experiencia no relevante (>10 anos)"
```

---

## Modelo de Datos

### Schema de CV Procesado

```typescript
interface ProcessedCV {
  id: string;
  userId: string;
  originalFile: {
    name: string;
    type: 'pdf' | 'docx' | 'doc' | 'image';
    size: number;
    url: string;
  };

  extractedData: {
    personalInfo: {
      fullName: string;
      email: string;
      phone: string;
      location: string;
      linkedinUrl?: string;
      githubUrl?: string;
      portfolioUrl?: string;
    };

    summary?: string;

    experience: Array<{
      company: string;
      title: string;
      location?: string;
      startDate: string;
      endDate?: string;
      current: boolean;
      description: string;
      achievements: string[];
    }>;

    education: Array<{
      institution: string;
      degree: string;
      field: string;
      startDate?: string;
      endDate?: string;
      gpa?: number;
    }>;

    skills: {
      technical: Array<{
        name: string;
        normalizedName: string;
        category: string;
        confidence: number;
      }>;
      soft: string[];
      languages: Array<{
        language: string;
        proficiency: string;
      }>;
    };

    certifications: Array<{
      name: string;
      issuer: string;
      date?: string;
      expirationDate?: string;
      credentialId?: string;
    }>;
  };

  embeddings: {
    fullCv: number[];      // 1536 dims
    skills: number[];      // 1536 dims
    experience: number[];  // 1536 dims
  };

  atsScore: {
    overall: number;       // 0-100
    breakdown: {
      keywordMatch: number;
      formatting: number;
      completeness: number;
      relevance: number;
    };
    penalties: string[];
    bonuses: string[];
  };

  suggestions: Array<{
    category: string;
    priority: 'high' | 'medium' | 'low';
    message: string;
    affectedSection?: string;
  }>;

  metadata: {
    processingTime: number;  // ms
    ocrApplied: boolean;
    llmFallbackUsed: boolean;
    confidence: number;      // 0-1
    version: string;
  };

  createdAt: Date;
  updatedAt: Date;
}
```

---

## Diagrama de Secuencia

```
Usuario          Frontend         API           Queue          Worker         DB
   |                |              |              |               |            |
   |-- Upload CV -->|              |              |               |            |
   |                |-- POST /cv ->|              |               |            |
   |                |              |-- Validate ->|               |            |
   |                |              |-- Store raw -|---------------|----------->|
   |                |              |-- Enqueue -->|               |            |
   |                |<- 202 + jobId|              |               |            |
   |<- "Processing" |              |              |               |            |
   |                |              |              |-- Pick job -->|            |
   |                |              |              |               |-- OCR? --->|
   |                |              |              |               |-- Parse -->|
   |                |              |              |               |-- NLP ---->|
   |                |              |              |               |-- Embed -->|
   |                |              |              |               |-- Score -->|
   |                |              |              |               |-- Save --->|
   |                |              |<-- WebSocket "complete" ------|            |
   |<-- WS notify --|              |              |               |            |
   |-- GET result ->|              |              |               |            |
   |                |-- GET /cv/id |--------------|---------------|----------->|
   |<-- CV data ----|<-------------|              |               |            |
```

---

## Metricas y Monitoreo

### KPIs del Pipeline

```yaml
metrics:
  # Latencia
  - cv_processing_duration_seconds:
      type: histogram
      buckets: [1, 5, 10, 20, 30, 60]
      labels: [file_type, ocr_applied]

  # Throughput
  - cv_processed_total:
      type: counter
      labels: [status, file_type]

  # Accuracy
  - extraction_confidence_score:
      type: histogram
      buckets: [0.5, 0.7, 0.8, 0.9, 0.95, 1.0]

  # Errores
  - cv_processing_errors_total:
      type: counter
      labels: [error_type, stage]

  # Queue
  - cv_queue_depth:
      type: gauge
  - cv_queue_wait_time_seconds:
      type: histogram

alerts:
  - name: HighProcessingLatency
    condition: cv_processing_duration_seconds_p95 > 30
    severity: warning

  - name: LowExtractionConfidence
    condition: avg(extraction_confidence_score) < 0.8
    severity: warning

  - name: HighErrorRate
    condition: rate(cv_processing_errors_total[5m]) > 0.1
    severity: critical
```

---

## Metricas de Fallback

### Dashboard de Monitoreo OCR

```yaml
ocr_metrics:
  # Distribucion por nivel
  - ocr_level_usage:
      type: counter
      labels: [level]  # opensource, cloud_vision, llm_vision
      description: "Numero de CVs procesados por nivel"

  # Confidence scores
  - ocr_confidence_distribution:
      type: histogram
      buckets: [0.5, 0.6, 0.7, 0.8, 0.85, 0.9, 0.95, 1.0]
      labels: [level]

  # Fallback rate
  - ocr_fallback_rate:
      type: gauge
      description: "Porcentaje de CVs que requieren fallback"

  # Costo acumulado
  - ocr_cost_total:
      type: counter
      labels: [level]
      description: "Costo acumulado en USD por nivel"

  # Latencia por nivel
  - ocr_latency_seconds:
      type: histogram
      buckets: [0.5, 1, 2, 3, 5, 10]
      labels: [level]

alerts:
  - name: HighFallbackRate
    condition: ocr_fallback_rate > 0.30
    severity: warning
    message: "Mas del 30% de CVs requieren fallback a niveles superiores"

  - name: LowOpensourceConfidence
    condition: avg(ocr_confidence_distribution{level="opensource"}) < 0.75
    severity: warning
    message: "Confianza promedio de PaddleOCR por debajo del umbral"

  - name: HighLLMUsage
    condition: rate(ocr_level_usage{level="llm_vision"}[1h]) > 10
    severity: warning
    message: "Alto uso de LLM Vision - revisar calidad de documentos entrantes"
```

### Ejemplo de Dashboard Grafana

```
+----------------------------------+----------------------------------+
|  Distribucion por Nivel (Pie)   |  Costo Acumulado (Line)         |
|                                  |                                  |
|   [75%] Open Source             |  $0.15 -----------------.        |
|   [20%] Cloud Vision            |                        /         |
|   [5%]  LLM Vision              |  $0.10 ---------------/          |
|                                  |                      /           |
+----------------------------------+----------------------------------+
|  Confianza Promedio (Gauge)     |  Latencia p95 (Bar)              |
|                                  |                                  |
|   OpenSource: [======85%]       |  OS:     [==] 1.2s              |
|   Cloud:      [========95%]     |  Cloud:  [===] 1.8s             |
|   LLM:        [=========98%]    |  LLM:    [=====] 3.5s           |
|                                  |                                  |
+----------------------------------+----------------------------------+
```

---

## Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| OCR falla en documentos de baja calidad | Media | Alto | Pipeline 3 niveles con fallback automatico |
| Extraccion incorrecta de datos | Media | Alto | Confidence score + revision manual + LLM fallback |
| Timeout en archivos grandes | Baja | Medio | Queue async, limites de tamano (10 paginas max) |
| Costos de API escalan | Baja | Medio | Nivel 1 gratuito procesa 70-80%, caching agresivo |
| Ataques de inyeccion via CV | Baja | Alto | Sanitizacion, sandbox de parsing, validacion de formato |
| PaddleOCR no disponible (GPU/memoria) | Baja | Medio | Fallback directo a Cloud Vision, mode CPU |
| Google Cloud Vision rate limits | Baja | Medio | Queue con backoff exponencial, cache de resultados |

---

## Plan de Implementacion (Actualizado v2.0)

### Fase 1: MVP con OCR Hibrido (Semanas 1-2)
- [ ] Setup de queue (BullMQ + Redis)
- [ ] Instalar PaddleOCR con modelos espanol
- [ ] Instalar PP-DocLayout para deteccion de secciones
- [ ] Implementar HybridCVParser clase base
- [ ] Configurar Google Cloud Vision como Nivel 2
- [ ] Almacenamiento en PostgreSQL

### Fase 2: NLP + Embeddings (Semanas 3-4)
- [ ] Integracion spaCy con NER custom (es_core_news_lg)
- [ ] Normalizacion de skills con diccionario
- [ ] Generacion de embeddings (OpenAI text-embedding-3-small)
- [ ] Almacenamiento en pgvector
- [ ] Implementar extraccion estructurada post-OCR

### Fase 3: Scoring + Sugerencias (Semana 5)
- [ ] Algoritmo de ATS scoring
- [ ] Generacion de sugerencias contextuales
- [ ] UI para visualizacion de resultados
- [ ] Dashboard de metricas OCR

### Fase 4: LLM Vision + Optimizacion (Semana 6)
- [ ] Implementar Nivel 3 con GPT-4 Vision
- [ ] Agregar soporte Gemini 1.5 Pro como alternativa
- [ ] Metricas de fallback y alertas
- [ ] Performance tuning (caching, pre-procesamiento condicional)
- [ ] A/B testing de umbrales de confianza

### Dependencias Tecnicas

```bash
# Python dependencies
pip install paddleocr paddlex pdf2image python-docx
pip install google-cloud-vision openai
pip install spacy opencv-python numpy
python -m spacy download es_core_news_lg

# System dependencies (Ubuntu/Debian)
apt-get install poppler-utils libreoffice
```

---

## Referencias

- [PaddleOCR Documentation](https://github.com/PaddlePaddle/PaddleOCR)
- [PaddleX PP-DocLayout](https://github.com/PaddlePaddle/PaddleX)
- [Google Cloud Vision API](https://cloud.google.com/vision/docs)
- [OpenAI Vision API](https://platform.openai.com/docs/guides/vision)
- [spaCy NER](https://spacy.io/usage/linguistic-features#named-entities)
- [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [BullMQ Documentation](https://docs.bullmq.io/)
- [pgvector](https://github.com/pgvector/pgvector)
- [pdf2image](https://github.com/Belval/pdf2image)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-31 | Equipo de Arquitectura | Version inicial |
| 2.0 | 2026-01-31 | Equipo de Arquitectura | Pipeline OCR hibrido 3 niveles: PaddleOCR + Cloud Vision + LLM Vision. Reemplazo de Tesseract por PaddleOCR. Agregado PP-DocLayout para deteccion de secciones. Metricas de fallback. Tabla de costos actualizada. |