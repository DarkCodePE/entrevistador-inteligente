# Guia de Implementacion OCR Hibrido para CV Parsing

## Tabla de Contenidos

1. [Setup del Entorno](#1-setup-del-entorno)
2. [Configuracion PaddleOCR para Espanol](#2-configuracion-paddleocr-para-espanol)
3. [PP-DocLayout para Secciones de CV](#3-pp-doclayout-para-secciones-de-cv)
4. [Pipeline Completo](#4-pipeline-completo)
5. [Manejo de Formatos](#5-manejo-de-formatos)
6. [Benchmarks Esperados](#6-benchmarks-esperados)
7. [Docker Compose para Produccion](#7-docker-compose-para-produccion)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Setup del Entorno

### 1.1 Dependencias Base (CPU)

```bash
# Crear entorno virtual
python -m venv cv-parser-env
source cv-parser-env/bin/activate  # Linux/Mac
# cv-parser-env\Scripts\activate   # Windows

# Dependencias principales
pip install paddleocr==2.7.0
pip install paddlex==3.0.0b1
pip install opencv-python==4.8.1.78
pip install spacy==3.7.2
pip install numpy==1.24.3
pip install Pillow==10.1.0

# Modelo spaCy para espanol
python -m spacy download es_core_news_lg
```

### 1.2 Soporte GPU (CUDA 11.7+)

```bash
# PaddlePaddle con GPU
pip install paddlepaddle-gpu==2.5.1.post117 -f https://www.paddlepaddle.org.cn/whl/linux/mkl/avx/stable.html

# Verificar instalacion GPU
python -c "import paddle; print(paddle.device.get_device())"
# Salida esperada: gpu:0
```

### 1.3 Dependencias Adicionales para Formatos

```bash
# Para PDF
pip install pdf2image==1.16.3
pip install PyMuPDF==1.23.6  # Alternativa mas rapida

# Para DOCX
pip install python-docx==1.1.0

# Para generacion de embeddings
pip install sentence-transformers==2.2.2
```

### 1.4 Dependencias del Sistema (Linux)

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    poppler-utils \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender-dev
```

---

## 2. Configuracion PaddleOCR para Espanol

### 2.1 Inicializacion Basica

```python
from paddleocr import PaddleOCR

# Configuracion optimizada para CVs en espanol
ocr = PaddleOCR(
    lang='es',                    # Idioma espanol
    use_angle_cls=True,           # Clasificador de angulo para texto rotado
    det_db_thresh=0.3,            # Umbral de deteccion (menor = mas sensible)
    det_db_box_thresh=0.5,        # Umbral de confianza de bounding box
    det_db_unclip_ratio=1.6,      # Expansion de bounding box
    rec_batch_num=6,              # Batch size para reconocimiento
    use_gpu=True,                 # Usar GPU si esta disponible
    show_log=False,               # Desactivar logs verbosos
    enable_mkldnn=True,           # Aceleracion Intel MKL-DNN (CPU)
    cpu_threads=4                 # Hilos CPU
)
```

### 2.2 Configuracion Avanzada

```python
class CVOCRConfig:
    """Configuracion optimizada para diferentes escenarios de CV."""

    @staticmethod
    def high_quality():
        """Para CVs con buena calidad de imagen."""
        return PaddleOCR(
            lang='es',
            use_angle_cls=True,
            det_db_thresh=0.3,
            det_db_box_thresh=0.6,
            det_db_unclip_ratio=1.5,
            rec_batch_num=8,
            use_gpu=True,
            det_limit_side_len=1920,  # Mayor resolucion
            rec_image_shape='3, 48, 320'
        )

    @staticmethod
    def low_quality():
        """Para CVs escaneados o con baja calidad."""
        return PaddleOCR(
            lang='es',
            use_angle_cls=True,
            det_db_thresh=0.2,        # Mas sensible
            det_db_box_thresh=0.4,    # Menor umbral
            det_db_unclip_ratio=1.8,  # Mayor expansion
            rec_batch_num=4,
            use_gpu=True,
            det_limit_side_len=2560,  # Mayor resolucion
            rec_image_shape='3, 48, 480'
        )

    @staticmethod
    def fast_processing():
        """Para procesamiento rapido en batch."""
        return PaddleOCR(
            lang='es',
            use_angle_cls=False,      # Desactivar para velocidad
            det_db_thresh=0.4,
            det_db_box_thresh=0.6,
            rec_batch_num=16,
            use_gpu=True,
            det_limit_side_len=960    # Menor resolucion
        )
```

### 2.3 Uso Basico del OCR

```python
import cv2
import numpy as np

def extract_text_from_image(image_path: str) -> list:
    """
    Extrae texto de una imagen de CV.

    Args:
        image_path: Ruta a la imagen

    Returns:
        Lista de tuplas (bbox, texto, confianza)
    """
    # Ejecutar OCR
    result = ocr.ocr(image_path, cls=True)

    extracted_data = []
    for line in result[0]:
        bbox = line[0]          # Coordenadas del bounding box
        text = line[1][0]       # Texto reconocido
        confidence = line[1][1] # Confianza (0-1)

        extracted_data.append({
            'bbox': bbox,
            'text': text,
            'confidence': confidence
        })

    return extracted_data

# Ejemplo de uso
results = extract_text_from_image('/path/to/cv.jpg')
for item in results:
    if item['confidence'] > 0.85:
        print(f"[{item['confidence']:.2f}] {item['text']}")
```

---

## 3. PP-DocLayout para Secciones de CV

### 3.1 Inicializacion del Modelo de Layout

```python
from paddlex import create_model

# Cargar modelo de layout (descarga automatica en primera ejecucion)
layout_model = create_model('PP-DocLayout_plus-L')

# Alternativas segun recursos:
# - 'PP-DocLayout_plus-L': Mayor precision, mas lento (~500ms/pagina GPU)
# - 'PP-DocLayout_plus-M': Balance precision/velocidad (~300ms/pagina GPU)
# - 'PP-DocLayout_plus-S': Mas rapido, menor precision (~150ms/pagina GPU)
```

### 3.2 Categorias Relevantes para CVs

```python
# Categorias detectadas por PP-DocLayout y su mapeo a secciones de CV
CV_LAYOUT_MAPPING = {
    'text': 'paragraph',           # Parrafos de texto (resumen, descripcion)
    'title': 'section_header',     # Encabezados de seccion
    'table': 'structured_data',    # Experiencia/educacion en formato tabla
    'list': 'skills_list',         # Listas de habilidades
    'figure': 'photo',             # Foto del candidato
    'header': 'document_header',   # Encabezado del documento
    'footer': 'document_footer',   # Pie de pagina
    'caption': 'label',            # Etiquetas de campos
}

# Secciones tipicas de un CV
CV_SECTIONS = [
    'datos_personales',
    'resumen_profesional',
    'experiencia_laboral',
    'educacion',
    'habilidades',
    'idiomas',
    'certificaciones',
    'proyectos',
    'referencias'
]
```

### 3.3 Deteccion de Layout

```python
import numpy as np
from PIL import Image
from dataclasses import dataclass
from typing import List, Tuple, Optional

@dataclass
class LayoutRegion:
    """Representa una region detectada en el layout."""
    category: str
    bbox: Tuple[float, float, float, float]  # x1, y1, x2, y2
    confidence: float
    text: Optional[str] = None

def detect_cv_layout(image_path: str) -> List[LayoutRegion]:
    """
    Detecta el layout de un CV identificando secciones.

    Args:
        image_path: Ruta a la imagen del CV

    Returns:
        Lista de regiones detectadas ordenadas por posicion Y
    """
    # Cargar imagen
    image = Image.open(image_path).convert('RGB')
    image_np = np.array(image)

    # Ejecutar deteccion de layout
    result = layout_model.predict(image_np)

    regions = []
    for detection in result['boxes']:
        region = LayoutRegion(
            category=detection['label'],
            bbox=tuple(detection['coordinate']),
            confidence=detection['score']
        )
        regions.append(region)

    # Ordenar por posicion vertical (top to bottom)
    regions.sort(key=lambda r: r.bbox[1])

    return regions

def classify_cv_section(title_text: str) -> str:
    """
    Clasifica el texto de un titulo en una seccion de CV.

    Args:
        title_text: Texto del encabezado

    Returns:
        Nombre de la seccion identificada
    """
    title_lower = title_text.lower().strip()

    section_keywords = {
        'datos_personales': ['datos personales', 'informacion personal', 'contacto'],
        'resumen_profesional': ['resumen', 'perfil', 'objetivo', 'sobre mi', 'acerca de'],
        'experiencia_laboral': ['experiencia', 'historial laboral', 'trayectoria'],
        'educacion': ['educacion', 'formacion', 'estudios', 'academico'],
        'habilidades': ['habilidades', 'competencias', 'skills', 'conocimientos'],
        'idiomas': ['idiomas', 'languages'],
        'certificaciones': ['certificaciones', 'cursos', 'capacitacion'],
        'proyectos': ['proyectos', 'portfolio', 'trabajos'],
        'referencias': ['referencias', 'recomendaciones']
    }

    for section, keywords in section_keywords.items():
        if any(kw in title_lower for kw in keywords):
            return section

    return 'other'
```

### 3.4 Visualizacion de Layout (Debug)

```python
import cv2
from PIL import Image, ImageDraw, ImageFont

def visualize_layout(image_path: str, regions: List[LayoutRegion], output_path: str):
    """Genera imagen con bounding boxes de las regiones detectadas."""

    # Colores por categoria
    COLORS = {
        'text': (0, 255, 0),       # Verde
        'title': (255, 0, 0),      # Rojo
        'table': (0, 0, 255),      # Azul
        'list': (255, 255, 0),     # Amarillo
        'figure': (255, 0, 255),   # Magenta
        'header': (0, 255, 255),   # Cyan
        'footer': (128, 128, 128), # Gris
    }

    image = cv2.imread(image_path)

    for region in regions:
        x1, y1, x2, y2 = map(int, region.bbox)
        color = COLORS.get(region.category, (128, 128, 128))

        # Dibujar rectangulo
        cv2.rectangle(image, (x1, y1), (x2, y2), color, 2)

        # Etiqueta
        label = f"{region.category} ({region.confidence:.2f})"
        cv2.putText(image, label, (x1, y1-10),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.5, color, 1)

    cv2.imwrite(output_path, image)
```

---

## 4. Pipeline Completo

### 4.1 Clase Principal del Pipeline

```python
import spacy
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, field
from datetime import datetime
import re

@dataclass
class CVData:
    """Estructura de datos extraidos del CV."""
    nombre: Optional[str] = None
    email: Optional[str] = None
    telefono: Optional[str] = None
    ubicacion: Optional[str] = None
    linkedin: Optional[str] = None
    resumen: Optional[str] = None
    experiencia: List[Dict[str, Any]] = field(default_factory=list)
    educacion: List[Dict[str, Any]] = field(default_factory=list)
    habilidades: List[str] = field(default_factory=list)
    idiomas: List[Dict[str, str]] = field(default_factory=list)
    certificaciones: List[Dict[str, Any]] = field(default_factory=list)
    raw_text: str = ""
    confidence_score: float = 0.0
    processing_time_ms: float = 0.0

class CVParserPipeline:
    """Pipeline completo para parsing de CVs."""

    def __init__(self, use_gpu: bool = True, quality_mode: str = 'balanced'):
        """
        Inicializa el pipeline.

        Args:
            use_gpu: Usar GPU si esta disponible
            quality_mode: 'fast', 'balanced', 'high_quality'
        """
        self.use_gpu = use_gpu
        self.quality_mode = quality_mode

        # Inicializar componentes
        self._init_ocr()
        self._init_layout_model()
        self._init_nlp()

    def _init_ocr(self):
        """Inicializa PaddleOCR segun modo de calidad."""
        config_map = {
            'fast': CVOCRConfig.fast_processing,
            'balanced': CVOCRConfig.high_quality,
            'high_quality': CVOCRConfig.low_quality
        }
        self.ocr = config_map.get(self.quality_mode, CVOCRConfig.high_quality)()

    def _init_layout_model(self):
        """Inicializa modelo de layout."""
        model_map = {
            'fast': 'PP-DocLayout_plus-S',
            'balanced': 'PP-DocLayout_plus-M',
            'high_quality': 'PP-DocLayout_plus-L'
        }
        model_name = model_map.get(self.quality_mode, 'PP-DocLayout_plus-M')
        self.layout_model = create_model(model_name)

    def _init_nlp(self):
        """Inicializa spaCy para NER."""
        self.nlp = spacy.load('es_core_news_lg')

    def preprocess_image(self, image: np.ndarray) -> np.ndarray:
        """
        Preprocesa imagen para mejorar OCR.

        Args:
            image: Imagen como numpy array (BGR)

        Returns:
            Imagen preprocesada
        """
        # Convertir a escala de grises
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

        # Aplicar CLAHE para mejorar contraste
        clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
        enhanced = clahe.apply(gray)

        # Denoising
        denoised = cv2.fastNlMeansDenoising(enhanced, None, 10, 7, 21)

        # Binarizacion adaptativa
        binary = cv2.adaptiveThreshold(
            denoised, 255,
            cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY, 11, 2
        )

        # Convertir de vuelta a BGR para OCR
        result = cv2.cvtColor(binary, cv2.COLOR_GRAY2BGR)

        return result

    def detect_layout(self, image: np.ndarray) -> List[LayoutRegion]:
        """Detecta layout del CV."""
        result = self.layout_model.predict(image)

        regions = []
        for detection in result['boxes']:
            region = LayoutRegion(
                category=detection['label'],
                bbox=tuple(detection['coordinate']),
                confidence=detection['score']
            )
            regions.append(region)

        return sorted(regions, key=lambda r: r.bbox[1])

    def ocr_region(self, image: np.ndarray, bbox: Tuple[float, float, float, float]) -> str:
        """
        Aplica OCR a una region especifica.

        Args:
            image: Imagen completa
            bbox: Bounding box (x1, y1, x2, y2)

        Returns:
            Texto extraido de la region
        """
        x1, y1, x2, y2 = map(int, bbox)

        # Recortar region con padding
        padding = 5
        h, w = image.shape[:2]
        x1 = max(0, x1 - padding)
        y1 = max(0, y1 - padding)
        x2 = min(w, x2 + padding)
        y2 = min(h, y2 + padding)

        region_image = image[y1:y2, x1:x2]

        # OCR en la region
        result = self.ocr.ocr(region_image, cls=True)

        if not result or not result[0]:
            return ""

        # Concatenar texto
        texts = [line[1][0] for line in result[0]]
        return " ".join(texts)

    def extract_entities(self, text: str) -> Dict[str, List[str]]:
        """
        Extrae entidades nombradas con spaCy.

        Args:
            text: Texto a analizar

        Returns:
            Diccionario con entidades por tipo
        """
        doc = self.nlp(text)

        entities = {
            'PER': [],   # Personas
            'ORG': [],   # Organizaciones
            'LOC': [],   # Ubicaciones
            'MISC': [],  # Miscelanea
            'DATE': []   # Fechas
        }

        for ent in doc.ents:
            if ent.label_ in entities:
                entities[ent.label_].append(ent.text)

        return entities

    def extract_patterns(self, text: str) -> Dict[str, Optional[str]]:
        """
        Extrae patrones especificos (email, telefono, etc).

        Args:
            text: Texto a analizar

        Returns:
            Diccionario con patrones encontrados
        """
        patterns = {
            'email': r'[\w\.-]+@[\w\.-]+\.\w+',
            'telefono': r'(?:\+?\d{1,3}[-.\s]?)?\(?\d{2,4}\)?[-.\s]?\d{3,4}[-.\s]?\d{3,4}',
            'linkedin': r'linkedin\.com/in/[\w-]+',
            'github': r'github\.com/[\w-]+',
            'fecha': r'\d{1,2}[-/]\d{1,2}[-/]\d{2,4}|\d{4}[-/]\d{1,2}|\w+\s+\d{4}'
        }

        results = {}
        for name, pattern in patterns.items():
            match = re.search(pattern, text, re.IGNORECASE)
            results[name] = match.group(0) if match else None

        return results

    def structure_cv_data(self, regions: List[LayoutRegion],
                          region_texts: Dict[str, str],
                          entities: Dict[str, List[str]],
                          patterns: Dict[str, Optional[str]]) -> CVData:
        """
        Estructura los datos extraidos en formato CVData.

        Args:
            regions: Regiones de layout detectadas
            region_texts: Textos por region
            entities: Entidades NER
            patterns: Patrones regex

        Returns:
            Objeto CVData estructurado
        """
        cv_data = CVData()

        # Datos de contacto
        cv_data.email = patterns.get('email')
        cv_data.telefono = patterns.get('telefono')
        cv_data.linkedin = patterns.get('linkedin')

        # Nombre (primera entidad de persona encontrada)
        if entities['PER']:
            cv_data.nombre = entities['PER'][0]

        # Ubicacion
        if entities['LOC']:
            cv_data.ubicacion = entities['LOC'][0]

        # Procesar secciones por tipo
        current_section = None
        for region in regions:
            if region.category == 'title':
                current_section = classify_cv_section(region.text or "")
            elif current_section and region.category in ['text', 'list', 'table']:
                text = region_texts.get(str(region.bbox), "")
                self._process_section(cv_data, current_section, text, entities)

        # Texto completo
        cv_data.raw_text = " ".join(region_texts.values())

        return cv_data

    def _process_section(self, cv_data: CVData, section: str,
                         text: str, entities: Dict[str, List[str]]):
        """Procesa una seccion especifica del CV."""

        if section == 'resumen_profesional':
            cv_data.resumen = text

        elif section == 'experiencia_laboral':
            # Parsear experiencia (simplificado)
            exp_entry = {
                'descripcion': text,
                'empresas': [org for org in entities['ORG'] if org in text],
                'raw_text': text
            }
            cv_data.experiencia.append(exp_entry)

        elif section == 'educacion':
            edu_entry = {
                'descripcion': text,
                'instituciones': [org for org in entities['ORG'] if org in text],
                'raw_text': text
            }
            cv_data.educacion.append(edu_entry)

        elif section == 'habilidades':
            # Dividir por comas, puntos o saltos de linea
            skills = re.split(r'[,\n;]', text)
            cv_data.habilidades.extend([s.strip() for s in skills if s.strip()])

        elif section == 'idiomas':
            # Formato esperado: "Idioma - Nivel"
            idioma_pattern = r'(\w+)\s*[-:]\s*([\w\s]+)'
            matches = re.findall(idioma_pattern, text)
            for idioma, nivel in matches:
                cv_data.idiomas.append({'idioma': idioma, 'nivel': nivel.strip()})

    def parse(self, image_path: str) -> CVData:
        """
        Pipeline principal: procesa un CV y extrae datos estructurados.

        Args:
            image_path: Ruta a la imagen del CV

        Returns:
            CVData con informacion estructurada
        """
        import time
        start_time = time.time()

        # 1. Cargar y preprocesar imagen
        image = cv2.imread(image_path)
        if image is None:
            raise ValueError(f"No se pudo cargar la imagen: {image_path}")

        processed_image = self.preprocess_image(image)

        # 2. Detectar layout
        regions = self.detect_layout(image)

        # 3. OCR por region
        region_texts = {}
        all_text = []

        for region in regions:
            text = self.ocr_region(processed_image, region.bbox)
            region.text = text
            region_texts[str(region.bbox)] = text
            all_text.append(text)

        full_text = " ".join(all_text)

        # 4. NER con spaCy
        entities = self.extract_entities(full_text)

        # 5. Extraccion de patrones
        patterns = self.extract_patterns(full_text)

        # 6. Estructurar datos
        cv_data = self.structure_cv_data(regions, region_texts, entities, patterns)

        # 7. Calcular metricas
        confidences = [r.confidence for r in regions if r.confidence]
        cv_data.confidence_score = sum(confidences) / len(confidences) if confidences else 0.0
        cv_data.processing_time_ms = (time.time() - start_time) * 1000

        return cv_data
```

### 4.2 Generacion de Embeddings

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class CVEmbeddingGenerator:
    """Genera embeddings para CVs parseados."""

    def __init__(self, model_name: str = 'paraphrase-multilingual-MiniLM-L12-v2'):
        """
        Inicializa el generador de embeddings.

        Args:
            model_name: Nombre del modelo de sentence-transformers
        """
        self.model = SentenceTransformer(model_name)

    def generate_cv_embedding(self, cv_data: CVData) -> np.ndarray:
        """
        Genera embedding combinado para un CV.

        Args:
            cv_data: Datos estructurados del CV

        Returns:
            Vector de embedding (384 dims por defecto)
        """
        # Construir texto representativo
        text_parts = []

        if cv_data.resumen:
            text_parts.append(f"Resumen: {cv_data.resumen}")

        if cv_data.experiencia:
            exp_text = " ".join([e.get('descripcion', '') for e in cv_data.experiencia])
            text_parts.append(f"Experiencia: {exp_text}")

        if cv_data.educacion:
            edu_text = " ".join([e.get('descripcion', '') for e in cv_data.educacion])
            text_parts.append(f"Educacion: {edu_text}")

        if cv_data.habilidades:
            text_parts.append(f"Habilidades: {', '.join(cv_data.habilidades)}")

        combined_text = " ".join(text_parts)

        # Generar embedding
        embedding = self.model.encode(combined_text, normalize_embeddings=True)

        return embedding

    def generate_skill_embeddings(self, cv_data: CVData) -> Dict[str, np.ndarray]:
        """
        Genera embeddings individuales para cada habilidad.

        Args:
            cv_data: Datos del CV

        Returns:
            Diccionario {skill: embedding}
        """
        if not cv_data.habilidades:
            return {}

        embeddings = self.model.encode(cv_data.habilidades, normalize_embeddings=True)

        return {skill: emb for skill, emb in zip(cv_data.habilidades, embeddings)}
```

### 4.3 Uso del Pipeline Completo

```python
# Ejemplo de uso
def process_cv(file_path: str) -> Dict[str, Any]:
    """Procesa un CV y retorna datos estructurados con embeddings."""

    # Inicializar pipeline
    pipeline = CVParserPipeline(use_gpu=True, quality_mode='balanced')
    embedding_gen = CVEmbeddingGenerator()

    # Parsear CV
    cv_data = pipeline.parse(file_path)

    # Generar embeddings
    cv_embedding = embedding_gen.generate_cv_embedding(cv_data)
    skill_embeddings = embedding_gen.generate_skill_embeddings(cv_data)

    # Preparar resultado
    result = {
        'datos_personales': {
            'nombre': cv_data.nombre,
            'email': cv_data.email,
            'telefono': cv_data.telefono,
            'ubicacion': cv_data.ubicacion,
            'linkedin': cv_data.linkedin
        },
        'resumen': cv_data.resumen,
        'experiencia': cv_data.experiencia,
        'educacion': cv_data.educacion,
        'habilidades': cv_data.habilidades,
        'idiomas': cv_data.idiomas,
        'embeddings': {
            'cv_vector': cv_embedding.tolist(),
            'skill_vectors': {k: v.tolist() for k, v in skill_embeddings.items()}
        },
        'metadata': {
            'confidence_score': cv_data.confidence_score,
            'processing_time_ms': cv_data.processing_time_ms
        }
    }

    return result

# Ejecutar
if __name__ == '__main__':
    result = process_cv('/path/to/cv.jpg')
    print(f"Nombre: {result['datos_personales']['nombre']}")
    print(f"Habilidades: {result['habilidades']}")
    print(f"Tiempo: {result['metadata']['processing_time_ms']:.0f}ms")
```

---

## 5. Manejo de Formatos

### 5.1 Procesamiento de PDF

```python
import fitz  # PyMuPDF
from pdf2image import convert_from_path
from typing import List, Generator
import tempfile

class PDFProcessor:
    """Procesador de CVs en formato PDF."""

    def __init__(self, dpi: int = 200):
        """
        Args:
            dpi: Resolucion para conversion a imagen
        """
        self.dpi = dpi

    def pdf_to_images(self, pdf_path: str) -> List[np.ndarray]:
        """
        Convierte PDF a lista de imagenes.

        Args:
            pdf_path: Ruta al PDF

        Returns:
            Lista de imagenes como numpy arrays
        """
        # Metodo 1: pdf2image (requiere poppler)
        images = convert_from_path(pdf_path, dpi=self.dpi)
        return [np.array(img) for img in images]

    def pdf_to_images_pymupdf(self, pdf_path: str) -> List[np.ndarray]:
        """
        Alternativa usando PyMuPDF (mas rapido, sin dependencias externas).

        Args:
            pdf_path: Ruta al PDF

        Returns:
            Lista de imagenes
        """
        doc = fitz.open(pdf_path)
        images = []

        for page in doc:
            # Zoom para mayor resolucion
            zoom = self.dpi / 72
            mat = fitz.Matrix(zoom, zoom)
            pix = page.get_pixmap(matrix=mat)

            # Convertir a numpy array
            img = np.frombuffer(pix.samples, dtype=np.uint8).reshape(
                pix.height, pix.width, pix.n
            )

            # Convertir de RGB a BGR para OpenCV
            if pix.n == 3:
                img = cv2.cvtColor(img, cv2.COLOR_RGB2BGR)
            elif pix.n == 4:
                img = cv2.cvtColor(img, cv2.COLOR_RGBA2BGR)

            images.append(img)

        doc.close()
        return images

    def extract_text_direct(self, pdf_path: str) -> str:
        """
        Extrae texto directamente del PDF (para PDFs con texto seleccionable).

        Args:
            pdf_path: Ruta al PDF

        Returns:
            Texto extraido
        """
        doc = fitz.open(pdf_path)
        text_parts = []

        for page in doc:
            text_parts.append(page.get_text())

        doc.close()
        return "\n".join(text_parts)

    def is_searchable_pdf(self, pdf_path: str) -> bool:
        """
        Determina si el PDF tiene texto seleccionable.

        Args:
            pdf_path: Ruta al PDF

        Returns:
            True si el PDF tiene texto seleccionable
        """
        doc = fitz.open(pdf_path)

        # Verificar primeras paginas
        for page in doc[:min(3, len(doc))]:
            text = page.get_text().strip()
            if len(text) > 50:  # Umbral minimo de texto
                doc.close()
                return True

        doc.close()
        return False


def process_pdf_cv(pdf_path: str, pipeline: CVParserPipeline) -> CVData:
    """
    Procesa un CV en formato PDF.

    Args:
        pdf_path: Ruta al PDF
        pipeline: Instancia del pipeline OCR

    Returns:
        CVData estructurado
    """
    processor = PDFProcessor(dpi=200)

    # Verificar si tiene texto seleccionable
    if processor.is_searchable_pdf(pdf_path):
        # Extraccion directa (mas rapido y preciso)
        text = processor.extract_text_direct(pdf_path)

        # Procesar con NLP unicamente
        entities = pipeline.extract_entities(text)
        patterns = pipeline.extract_patterns(text)

        cv_data = CVData(
            raw_text=text,
            email=patterns.get('email'),
            telefono=patterns.get('telefono'),
            linkedin=patterns.get('linkedin'),
            nombre=entities['PER'][0] if entities['PER'] else None,
            ubicacion=entities['LOC'][0] if entities['LOC'] else None
        )

        return cv_data

    # PDF escaneado: usar OCR
    images = processor.pdf_to_images_pymupdf(pdf_path)

    # Procesar cada pagina
    all_cv_data = []
    for i, image in enumerate(images):
        # Guardar temporalmente
        with tempfile.NamedTemporaryFile(suffix='.png', delete=False) as tmp:
            cv2.imwrite(tmp.name, image)
            page_data = pipeline.parse(tmp.name)
            all_cv_data.append(page_data)

    # Combinar datos de todas las paginas
    combined = merge_cv_data(all_cv_data)

    return combined


def merge_cv_data(cv_list: List[CVData]) -> CVData:
    """Combina datos de multiples paginas de CV."""
    if not cv_list:
        return CVData()

    merged = CVData()

    for cv in cv_list:
        # Tomar primer valor no nulo para campos simples
        if not merged.nombre and cv.nombre:
            merged.nombre = cv.nombre
        if not merged.email and cv.email:
            merged.email = cv.email
        if not merged.telefono and cv.telefono:
            merged.telefono = cv.telefono
        if not merged.ubicacion and cv.ubicacion:
            merged.ubicacion = cv.ubicacion
        if not merged.linkedin and cv.linkedin:
            merged.linkedin = cv.linkedin
        if not merged.resumen and cv.resumen:
            merged.resumen = cv.resumen

        # Agregar listas
        merged.experiencia.extend(cv.experiencia)
        merged.educacion.extend(cv.educacion)
        merged.habilidades.extend(cv.habilidades)
        merged.idiomas.extend(cv.idiomas)
        merged.certificaciones.extend(cv.certificaciones)
        merged.raw_text += " " + cv.raw_text

    # Eliminar duplicados en habilidades
    merged.habilidades = list(set(merged.habilidades))

    # Calcular confianza promedio
    confidences = [cv.confidence_score for cv in cv_list if cv.confidence_score > 0]
    merged.confidence_score = sum(confidences) / len(confidences) if confidences else 0.0

    # Sumar tiempos de procesamiento
    merged.processing_time_ms = sum(cv.processing_time_ms for cv in cv_list)

    return merged
```

### 5.2 Procesamiento de DOCX

```python
from docx import Document
from docx.shared import Inches
import io

class DOCXProcessor:
    """Procesador de CVs en formato DOCX."""

    def extract_text(self, docx_path: str) -> str:
        """
        Extrae todo el texto del documento.

        Args:
            docx_path: Ruta al archivo DOCX

        Returns:
            Texto completo del documento
        """
        doc = Document(docx_path)
        text_parts = []

        for para in doc.paragraphs:
            text_parts.append(para.text)

        # Extraer texto de tablas
        for table in doc.tables:
            for row in table.rows:
                row_text = [cell.text for cell in row.cells]
                text_parts.append(" | ".join(row_text))

        return "\n".join(text_parts)

    def extract_structured(self, docx_path: str) -> Dict[str, Any]:
        """
        Extrae contenido estructurado del DOCX.

        Args:
            docx_path: Ruta al archivo

        Returns:
            Diccionario con parrafos, tablas y estilos
        """
        doc = Document(docx_path)

        structure = {
            'paragraphs': [],
            'tables': [],
            'headers': []
        }

        for para in doc.paragraphs:
            para_info = {
                'text': para.text,
                'style': para.style.name if para.style else 'Normal',
                'is_heading': para.style.name.startswith('Heading') if para.style else False
            }

            if para_info['is_heading']:
                structure['headers'].append(para.text)

            structure['paragraphs'].append(para_info)

        for table in doc.tables:
            table_data = []
            for row in table.rows:
                table_data.append([cell.text for cell in row.cells])
            structure['tables'].append(table_data)

        return structure


def process_docx_cv(docx_path: str, pipeline: CVParserPipeline) -> CVData:
    """
    Procesa un CV en formato DOCX.

    Args:
        docx_path: Ruta al DOCX
        pipeline: Instancia del pipeline

    Returns:
        CVData estructurado
    """
    processor = DOCXProcessor()

    # Extraer texto
    text = processor.extract_text(docx_path)
    structured = processor.extract_structured(docx_path)

    # Procesar con NLP
    entities = pipeline.extract_entities(text)
    patterns = pipeline.extract_patterns(text)

    # Construir CVData
    cv_data = CVData(
        raw_text=text,
        email=patterns.get('email'),
        telefono=patterns.get('telefono'),
        linkedin=patterns.get('linkedin'),
        nombre=entities['PER'][0] if entities['PER'] else None,
        ubicacion=entities['LOC'][0] if entities['LOC'] else None
    )

    # Procesar secciones por headers
    current_section = None
    current_content = []

    for para in structured['paragraphs']:
        if para['is_heading']:
            # Procesar seccion anterior
            if current_section and current_content:
                section_text = " ".join(current_content)
                pipeline._process_section(cv_data, current_section, section_text, entities)

            # Nueva seccion
            current_section = classify_cv_section(para['text'])
            current_content = []
        else:
            current_content.append(para['text'])

    # Procesar ultima seccion
    if current_section and current_content:
        section_text = " ".join(current_content)
        pipeline._process_section(cv_data, current_section, section_text, entities)

    cv_data.confidence_score = 0.95  # Alta confianza para DOCX

    return cv_data
```

### 5.3 Detector de Formato Universal

```python
import magic  # python-magic
from pathlib import Path

class CVFormatDetector:
    """Detecta el formato de un archivo de CV."""

    SUPPORTED_FORMATS = {
        'application/pdf': 'pdf',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document': 'docx',
        'application/msword': 'doc',
        'image/jpeg': 'image',
        'image/png': 'image',
        'image/tiff': 'image',
        'image/webp': 'image'
    }

    @staticmethod
    def detect(file_path: str) -> str:
        """
        Detecta el formato del archivo.

        Args:
            file_path: Ruta al archivo

        Returns:
            Tipo de formato ('pdf', 'docx', 'image', 'unknown')
        """
        # Por extension
        ext = Path(file_path).suffix.lower()
        ext_map = {
            '.pdf': 'pdf',
            '.docx': 'docx',
            '.doc': 'doc',
            '.jpg': 'image',
            '.jpeg': 'image',
            '.png': 'image',
            '.tiff': 'image',
            '.webp': 'image'
        }

        if ext in ext_map:
            return ext_map[ext]

        # Por magic bytes
        try:
            mime = magic.from_file(file_path, mime=True)
            return CVFormatDetector.SUPPORTED_FORMATS.get(mime, 'unknown')
        except Exception:
            return 'unknown'


def process_cv_universal(file_path: str) -> CVData:
    """
    Procesa un CV en cualquier formato soportado.

    Args:
        file_path: Ruta al archivo

    Returns:
        CVData estructurado
    """
    pipeline = CVParserPipeline(use_gpu=True)
    format_type = CVFormatDetector.detect(file_path)

    if format_type == 'pdf':
        return process_pdf_cv(file_path, pipeline)
    elif format_type == 'docx':
        return process_docx_cv(file_path, pipeline)
    elif format_type == 'image':
        return pipeline.parse(file_path)
    elif format_type == 'doc':
        raise NotImplementedError("Formato DOC requiere conversion previa a DOCX")
    else:
        raise ValueError(f"Formato no soportado: {format_type}")
```

---

## 6. Benchmarks Esperados

### 6.1 Metricas de Rendimiento

| Metrica | Target | Descripcion |
|---------|--------|-------------|
| Precision texto | >95% | Accuracy de caracteres reconocidos |
| Precision secciones | >90% | Accuracy de clasificacion de secciones |
| Latencia Nivel 1 (GPU) | <3s | CVs de alta calidad, 1 pagina |
| Latencia Nivel 2 (GPU) | <5s | CVs escaneados, 1-2 paginas |
| Latencia Nivel 3 (GPU) | <10s | PDFs multipagina, calidad variable |
| Costo promedio | <$0.001/CV | Sin APIs externas |
| Throughput | >100 CVs/hora | Procesamiento en batch (GPU) |

### 6.2 Metricas por Componente

| Componente | Tiempo (GPU) | Tiempo (CPU) | Memoria |
|------------|--------------|--------------|---------|
| Preprocesamiento | ~50ms | ~100ms | 50MB |
| Layout Detection | ~200ms | ~800ms | 500MB |
| OCR (por pagina) | ~500ms | ~2s | 300MB |
| NER spaCy | ~100ms | ~150ms | 400MB |
| Embeddings | ~50ms | ~200ms | 200MB |
| **Total (1 pagina)** | **~900ms** | **~3.5s** | **1.5GB** |

### 6.3 Script de Benchmark

```python
import time
import statistics
from pathlib import Path
from typing import List, Dict

def benchmark_pipeline(test_files: List[str], iterations: int = 5) -> Dict[str, Any]:
    """
    Ejecuta benchmarks del pipeline.

    Args:
        test_files: Lista de archivos de prueba
        iterations: Numero de iteraciones por archivo

    Returns:
        Resultados del benchmark
    """
    pipeline = CVParserPipeline(use_gpu=True, quality_mode='balanced')

    results = {
        'files': [],
        'times': [],
        'confidences': [],
        'errors': []
    }

    for file_path in test_files:
        file_times = []
        file_confidences = []

        for i in range(iterations):
            try:
                start = time.perf_counter()
                cv_data = process_cv_universal(file_path)
                elapsed = (time.perf_counter() - start) * 1000

                file_times.append(elapsed)
                file_confidences.append(cv_data.confidence_score)

            except Exception as e:
                results['errors'].append({
                    'file': file_path,
                    'iteration': i,
                    'error': str(e)
                })

        if file_times:
            results['files'].append({
                'path': file_path,
                'mean_time_ms': statistics.mean(file_times),
                'std_time_ms': statistics.stdev(file_times) if len(file_times) > 1 else 0,
                'mean_confidence': statistics.mean(file_confidences)
            })
            results['times'].extend(file_times)
            results['confidences'].extend(file_confidences)

    # Estadisticas globales
    results['summary'] = {
        'total_files': len(test_files),
        'successful_runs': len(results['times']),
        'mean_time_ms': statistics.mean(results['times']) if results['times'] else 0,
        'p95_time_ms': sorted(results['times'])[int(0.95 * len(results['times']))] if results['times'] else 0,
        'mean_confidence': statistics.mean(results['confidences']) if results['confidences'] else 0,
        'error_rate': len(results['errors']) / (len(test_files) * iterations)
    }

    return results


# Ejecutar benchmark
if __name__ == '__main__':
    test_files = list(Path('/path/to/test_cvs').glob('*.*'))
    results = benchmark_pipeline(test_files, iterations=3)

    print("=== Benchmark Results ===")
    print(f"Archivos procesados: {results['summary']['total_files']}")
    print(f"Tiempo promedio: {results['summary']['mean_time_ms']:.0f}ms")
    print(f"Tiempo P95: {results['summary']['p95_time_ms']:.0f}ms")
    print(f"Confianza promedio: {results['summary']['mean_confidence']:.2%}")
    print(f"Tasa de error: {results['summary']['error_rate']:.2%}")
```

---

## 7. Docker Compose para Produccion

### 7.1 Dockerfile

```dockerfile
# Dockerfile
FROM paddlepaddle/paddle:2.5.1-gpu-cuda11.7-cudnn8.4-trt8.4

# Variables de entorno
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PADDLE_OCR_LANG=es

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender-dev \
    && rm -rf /var/lib/apt/lists/*

# Crear directorio de trabajo
WORKDIR /app

# Copiar requirements
COPY requirements.txt .

# Instalar dependencias Python
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# Descargar modelos de spaCy
RUN python -m spacy download es_core_news_lg

# Pre-descargar modelos de PaddleOCR
RUN python -c "from paddleocr import PaddleOCR; PaddleOCR(lang='es', show_log=False)"

# Pre-descargar modelo de layout
RUN python -c "from paddlex import create_model; create_model('PP-DocLayout_plus-M')"

# Pre-descargar modelo de embeddings
RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')"

# Copiar codigo
COPY src/ ./src/
COPY models/ ./models/

# Puerto por defecto
EXPOSE 8000

# Comando de inicio
CMD ["python", "-m", "uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 7.2 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  cv-parser:
    build:
      context: .
      dockerfile: Dockerfile
    image: cv-parser:latest
    container_name: cv-parser-service
    restart: unless-stopped
    ports:
      - "8000:8000"
    volumes:
      - ./models:/app/models:ro
      - ./data/uploads:/app/uploads
      - ./data/processed:/app/processed
      - ./logs:/app/logs
    environment:
      - PADDLE_OCR_LANG=es
      - USE_GPU=true
      - QUALITY_MODE=balanced
      - MAX_FILE_SIZE_MB=20
      - ALLOWED_EXTENSIONS=pdf,docx,jpg,jpeg,png
      - LOG_LEVEL=INFO
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
        limits:
          memory: 8G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - cv-network

  redis:
    image: redis:7-alpine
    container_name: cv-parser-redis
    restart: unless-stopped
    volumes:
      - redis-data:/data
    networks:
      - cv-network

  worker:
    build:
      context: .
      dockerfile: Dockerfile
    image: cv-parser:latest
    container_name: cv-parser-worker
    restart: unless-stopped
    command: ["python", "-m", "celery", "-A", "src.worker.celery_app", "worker", "--loglevel=info"]
    volumes:
      - ./models:/app/models:ro
      - ./data/uploads:/app/uploads
      - ./data/processed:/app/processed
    environment:
      - PADDLE_OCR_LANG=es
      - USE_GPU=true
      - REDIS_URL=redis://redis:6379/0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on:
      - redis
    networks:
      - cv-network

networks:
  cv-network:
    driver: bridge

volumes:
  redis-data:
```

### 7.3 Requirements.txt

```text
# requirements.txt

# Core OCR
paddleocr==2.7.0
paddlex==3.0.0b1
paddlepaddle-gpu==2.5.1.post117

# Computer Vision
opencv-python==4.8.1.78
Pillow==10.1.0
numpy==1.24.3

# NLP
spacy==3.7.2

# Document Processing
pdf2image==1.16.3
PyMuPDF==1.23.6
python-docx==1.1.0

# Embeddings
sentence-transformers==2.2.2

# API
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6

# Task Queue
celery==5.3.4
redis==5.0.1

# Utilities
python-magic==0.4.27
pydantic==2.5.2

# Monitoring
prometheus-client==0.19.0
```

### 7.4 API FastAPI

```python
# src/api/main.py
from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
import tempfile
import os
from typing import Optional

from ..pipeline import CVParserPipeline, CVEmbeddingGenerator
from ..formats import process_cv_universal, CVFormatDetector

app = FastAPI(
    title="CV Parser API",
    description="API para parsing de CVs con OCR hibrido",
    version="1.0.0"
)

# Inicializar pipeline globalmente
pipeline = None
embedding_gen = None

@app.on_event("startup")
async def startup():
    global pipeline, embedding_gen
    use_gpu = os.getenv("USE_GPU", "true").lower() == "true"
    quality_mode = os.getenv("QUALITY_MODE", "balanced")

    pipeline = CVParserPipeline(use_gpu=use_gpu, quality_mode=quality_mode)
    embedding_gen = CVEmbeddingGenerator()

@app.get("/health")
async def health_check():
    return {"status": "healthy", "gpu_available": pipeline.use_gpu if pipeline else False}

@app.post("/parse")
async def parse_cv(
    file: UploadFile = File(...),
    include_embeddings: bool = True
):
    """
    Parsea un CV y retorna datos estructurados.

    - **file**: Archivo del CV (PDF, DOCX, JPG, PNG)
    - **include_embeddings**: Incluir vectores de embedding
    """
    # Validar formato
    format_type = CVFormatDetector.detect_from_filename(file.filename)
    if format_type == 'unknown':
        raise HTTPException(400, f"Formato no soportado: {file.filename}")

    # Validar tamano
    max_size = int(os.getenv("MAX_FILE_SIZE_MB", "20")) * 1024 * 1024
    contents = await file.read()
    if len(contents) > max_size:
        raise HTTPException(413, "Archivo demasiado grande")

    # Guardar temporalmente
    with tempfile.NamedTemporaryFile(delete=False, suffix=f".{format_type}") as tmp:
        tmp.write(contents)
        tmp_path = tmp.name

    try:
        # Procesar CV
        cv_data = process_cv_universal(tmp_path)

        result = {
            'datos_personales': {
                'nombre': cv_data.nombre,
                'email': cv_data.email,
                'telefono': cv_data.telefono,
                'ubicacion': cv_data.ubicacion,
                'linkedin': cv_data.linkedin
            },
            'resumen': cv_data.resumen,
            'experiencia': cv_data.experiencia,
            'educacion': cv_data.educacion,
            'habilidades': cv_data.habilidades,
            'idiomas': cv_data.idiomas,
            'metadata': {
                'confidence_score': cv_data.confidence_score,
                'processing_time_ms': cv_data.processing_time_ms,
                'format': format_type
            }
        }

        # Agregar embeddings si se solicitan
        if include_embeddings:
            cv_embedding = embedding_gen.generate_cv_embedding(cv_data)
            result['embeddings'] = {
                'cv_vector': cv_embedding.tolist()
            }

        return JSONResponse(content=result)

    except Exception as e:
        raise HTTPException(500, f"Error procesando CV: {str(e)}")

    finally:
        os.unlink(tmp_path)

@app.post("/batch")
async def batch_parse(files: list[UploadFile] = File(...)):
    """Procesa multiples CVs en batch."""
    results = []

    for file in files:
        try:
            result = await parse_cv(file, include_embeddings=False)
            results.append({
                'filename': file.filename,
                'status': 'success',
                'data': result
            })
        except Exception as e:
            results.append({
                'filename': file.filename,
                'status': 'error',
                'error': str(e)
            })

    return {'results': results, 'total': len(files), 'successful': sum(1 for r in results if r['status'] == 'success')}
```

---

## 8. Troubleshooting

### 8.1 Problemas Comunes

#### Error: CUDA out of memory
```python
# Solucion 1: Reducir batch size
ocr = PaddleOCR(rec_batch_num=2, use_gpu=True)

# Solucion 2: Reducir resolucion
ocr = PaddleOCR(det_limit_side_len=960, use_gpu=True)

# Solucion 3: Usar CPU para documentos grandes
import paddle
paddle.set_device('cpu')
```

#### Error: Texto no detectado correctamente
```python
# Solucion 1: Ajustar umbrales
ocr = PaddleOCR(
    det_db_thresh=0.2,      # Bajar umbral de deteccion
    det_db_box_thresh=0.3,  # Bajar umbral de confianza
    lang='es'
)

# Solucion 2: Preprocesamiento adicional
def enhance_for_ocr(image):
    # Aumentar contraste
    lab = cv2.cvtColor(image, cv2.COLOR_BGR2LAB)
    l, a, b = cv2.split(lab)
    clahe = cv2.createCLAHE(clipLimit=3.0, tileGridSize=(8, 8))
    l = clahe.apply(l)
    lab = cv2.merge([l, a, b])
    return cv2.cvtColor(lab, cv2.COLOR_LAB2BGR)
```

#### Error: Secciones no detectadas
```python
# Solucion: Usar modelo mas grande
layout_model = create_model('PP-DocLayout_plus-L')

# O ajustar post-procesamiento
def merge_nearby_regions(regions, y_threshold=20):
    """Combina regiones verticalmente cercanas."""
    merged = []
    for region in sorted(regions, key=lambda r: r.bbox[1]):
        if merged and abs(region.bbox[1] - merged[-1].bbox[3]) < y_threshold:
            # Combinar con region anterior
            merged[-1].bbox = (
                min(merged[-1].bbox[0], region.bbox[0]),
                merged[-1].bbox[1],
                max(merged[-1].bbox[2], region.bbox[2]),
                region.bbox[3]
            )
            merged[-1].text += " " + region.text
        else:
            merged.append(region)
    return merged
```

### 8.2 Logs y Debugging

```python
import logging

# Configurar logging detallado
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Habilitar logs de PaddleOCR
ocr = PaddleOCR(lang='es', show_log=True)

# Guardar imagenes intermedias para debug
def debug_pipeline(image_path: str, output_dir: str):
    import os
    os.makedirs(output_dir, exist_ok=True)

    # Original
    image = cv2.imread(image_path)
    cv2.imwrite(f"{output_dir}/01_original.png", image)

    # Preprocesado
    processed = pipeline.preprocess_image(image)
    cv2.imwrite(f"{output_dir}/02_preprocessed.png", processed)

    # Layout
    regions = pipeline.detect_layout(image)
    visualize_layout(image_path, regions, f"{output_dir}/03_layout.png")

    # OCR por region
    for i, region in enumerate(regions):
        x1, y1, x2, y2 = map(int, region.bbox)
        region_img = image[y1:y2, x1:x2]
        cv2.imwrite(f"{output_dir}/04_region_{i}_{region.category}.png", region_img)
```

### 8.3 Optimizacion de Memoria

```python
import gc
import torch

def clear_gpu_memory():
    """Libera memoria GPU."""
    gc.collect()
    if torch.cuda.is_available():
        torch.cuda.empty_cache()

def process_large_batch(files: List[str], batch_size: int = 5):
    """Procesa archivos en lotes para evitar OOM."""
    results = []

    for i in range(0, len(files), batch_size):
        batch = files[i:i+batch_size]

        for file in batch:
            result = process_cv_universal(file)
            results.append(result)

        # Limpiar memoria entre lotes
        clear_gpu_memory()

    return results
```

---

## Referencias

- [PaddleOCR Documentation](https://github.com/PaddlePaddle/PaddleOCR)
- [PaddleX Documentation](https://github.com/PaddlePaddle/PaddleX)
- [spaCy Spanish Models](https://spacy.io/models/es)
- [Sentence Transformers](https://www.sbert.net/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
