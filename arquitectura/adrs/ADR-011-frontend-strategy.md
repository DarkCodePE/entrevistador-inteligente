# ADR-011: Estrategia Frontend Mobile-First

## Estado
**Aceptado**

Fecha: 2026-01-31
Autores: Equipo de Arquitectura
Revisores: CTO, Tech Lead, UX Lead

---

## Contexto

### Descripcion del Problema
"Entrevistador Inteligente Peru" necesita una estrategia frontend que:
1. Priorice la experiencia movil (80%+ de usuarios peruanos acceden via smartphone)
2. Funcione en condiciones de red limitadas (3G en zonas rurales)
3. Soporte dispositivos low-end (Android Go, 2GB RAM)
4. Permita evolucion hacia apps nativas en el futuro
5. Maximice la velocidad de desarrollo con equipo reducido

### Plataformas Objetivo

| Plataforma | Prioridad | Timeline |
|------------|-----------|----------|
| Web App (mobile responsive) | P0 | MVP (semanas 1-12) |
| PWA (instalable, offline) | P0 | MVP (semanas 1-12) |
| Android App | P1 | Fase 2 (meses 6-12) |
| iOS App | P2 | Fase 3 (ano 2) |

### Componentes Criticos de UI

```
┌─────────────────────────────────────────────────────────────┐
│                    Componentes Core                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────┐  ┌───────────────────┐               │
│  │  Chat Interface   │  │   CV Viewer/      │               │
│  │  (Entrevistas)    │  │   Editor          │               │
│  │  - Streaming IA   │  │  - PDF render     │               │
│  │  - Voice input    │  │  - Inline edit    │               │
│  │  - Typing ind.    │  │  - Export         │               │
│  └───────────────────┘  └───────────────────┘               │
│                                                              │
│  ┌───────────────────┐  ┌───────────────────┐               │
│  │  Dashboard        │  │  Onboarding       │               │
│  │  (Metricas)       │  │  Wizard           │               │
│  │  - Charts         │  │  - Multi-step     │               │
│  │  - Progress       │  │  - Validacion     │               │
│  │  - Recommendations│  │  - Social login   │               │
│  └───────────────────┘  └───────────────────┘               │
│                                                              │
│  ┌───────────────────┐  ┌───────────────────┐               │
│  │  Payment Flow     │  │  WhatsApp         │               │
│  │  - Yape/Plin      │  │  Integration      │               │
│  │  - Tarjetas       │  │  - Share CV       │               │
│  │  - Invoicing PE   │  │  - Notificaciones │               │
│  └───────────────────┘  └───────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Restricciones del Mercado Peruano

| Factor | Realidad | Implicacion Tecnica |
|--------|----------|---------------------|
| Conectividad | 40% usuarios en 3G o inferior | Offline-first, lazy loading agresivo |
| Dispositivos | 60% Android low-end (<4GB RAM) | Bundle pequeno, sin frameworks pesados |
| Preferencias | WhatsApp como canal principal | Deep links, share intents |
| Pagos | Yape/Plin dominan (70%+) | Integracion con billeteras locales |
| Idioma | Espanol con regionalismos | i18n preparado, copy localizado |

### Performance Targets (Core Web Vitals)

| Metrica | Target | Criticidad |
|---------|--------|------------|
| LCP (Largest Contentful Paint) | < 2.5s | Alta (SEO + UX) |
| FID (First Input Delay) | < 100ms | Alta (interactividad) |
| CLS (Cumulative Layout Shift) | < 0.1 | Media (estabilidad visual) |
| TTI (Time to Interactive) | < 3.5s | Alta (engagement) |
| Bundle Size (initial) | < 200KB gzipped | Critica (3G users) |
| Bundle Size (total) | < 500KB gzipped | Alta |

---

## Decision

### Framework Principal: Next.js 14+ con App Router

**Adoptamos Next.js como framework principal para web y PWA.**

```
┌─────────────────────────────────────────────────────────────┐
│                     Stack Frontend                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Framework:        Next.js 14+ (App Router)                │
│   Rendering:        RSC + Streaming + ISR                   │
│   Styling:          Tailwind CSS + shadcn/ui                │
│   State:            Zustand + React Query (TanStack)        │
│   Forms:            React Hook Form + Zod                   │
│   Mobile Future:    React Native (code sharing parcial)     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Estrategia Mobile: PWA-First, React Native Futuro

```
Fase 1 (MVP)              Fase 2                    Fase 3
─────────────────────────────────────────────────────────────►

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  Next.js     │         │  Next.js     │         │  Next.js     │
│  + PWA       │   =>    │  + PWA       │   =>    │  + PWA       │
│              │         │  + RN Android│         │  + RN iOS    │
│              │         │              │         │  + RN Android│
└──────────────┘         └──────────────┘         └──────────────┘

Cobertura:               Cobertura:               Cobertura:
- 100% Web               - 100% Web               - 100% Web
- PWA instalable         - PWA instalable         - PWA instalable
                         - Play Store             - Play Store
                                                  - App Store
```

### UI Library: Tailwind CSS + shadcn/ui

```
┌─────────────────────────────────────────────────────────────┐
│                    Design System Stack                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Base:           Tailwind CSS 3.4+                          │
│  Components:     shadcn/ui (copy-paste, no runtime)         │
│  Icons:          Lucide React (tree-shakeable)              │
│  Animations:     Framer Motion (lazy loaded)                │
│  Charts:         Recharts (lightweight)                     │
│  Theme:          CSS Variables (dark/light mode)            │
│                                                              │
│  Custom Components:                                          │
│  ├── ei-chat/         # Chat interface optimizado           │
│  ├── ei-cv-viewer/    # Visor de CV con PDF.js              │
│  ├── ei-dashboard/    # Widgets de metricas                 │
│  ├── ei-forms/        # Formularios con validacion          │
│  └── ei-payments/     # Componentes de pago Peru            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### State Management: Zustand + React Query

```typescript
// Arquitectura de Estado
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌─────────────────┐    ┌─────────────────┐                 │
│  │   React Query   │    │     Zustand     │                 │
│  │   (TanStack)    │    │                 │                 │
│  ├─────────────────┤    ├─────────────────┤                 │
│  │ Server State:   │    │ Client State:   │                 │
│  │ - API calls     │    │ - UI state      │                 │
│  │ - Caching       │    │ - User prefs    │                 │
│  │ - Mutations     │    │ - Chat history  │                 │
│  │ - Optimistic    │    │ - Form drafts   │                 │
│  │ - Background    │    │ - Theme         │                 │
│  │   refetch       │    │ - Offline queue │                 │
│  └─────────────────┘    └─────────────────┘                 │
│                                                              │
│  Persistencia:                                               │
│  - IndexedDB para offline (via Zustand persist)             │
│  - Service Worker para cache de assets                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Arquitectura de Componentes

### Estructura de Proyecto

```
src/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Grupo: rutas autenticadas
│   │   ├── dashboard/
│   │   ├── cv/
│   │   ├── entrevistas/
│   │   └── configuracion/
│   ├── (public)/                 # Grupo: rutas publicas
│   │   ├── login/
│   │   ├── registro/
│   │   └── landing/
│   ├── (onboarding)/             # Grupo: wizard inicial
│   │   └── [...step]/
│   ├── api/                      # API Routes (BFF)
│   │   ├── auth/
│   │   ├── cv/
│   │   └── payments/
│   ├── layout.tsx
│   └── globals.css
│
├── components/
│   ├── ui/                       # shadcn/ui base components
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── dialog.tsx
│   │   └── ...
│   ├── features/                 # Feature components
│   │   ├── chat/
│   │   │   ├── ChatContainer.tsx
│   │   │   ├── MessageBubble.tsx
│   │   │   ├── StreamingMessage.tsx
│   │   │   ├── VoiceInput.tsx
│   │   │   └── hooks/
│   │   ├── cv/
│   │   │   ├── CVViewer.tsx
│   │   │   ├── CVEditor.tsx
│   │   │   ├── PDFRenderer.tsx
│   │   │   └── hooks/
│   │   ├── dashboard/
│   │   │   ├── MetricsCard.tsx
│   │   │   ├── ProgressChart.tsx
│   │   │   └── Recommendations.tsx
│   │   ├── onboarding/
│   │   │   ├── StepIndicator.tsx
│   │   │   ├── ProfileForm.tsx
│   │   │   └── CVUpload.tsx
│   │   └── payments/
│   │       ├── PaymentSelector.tsx
│   │       ├── YapeQR.tsx
│   │       └── CardForm.tsx
│   └── layout/                   # Layout components
│       ├── Header.tsx
│       ├── MobileNav.tsx
│       ├── Sidebar.tsx
│       └── Footer.tsx
│
├── lib/
│   ├── api/                      # API client
│   │   ├── client.ts
│   │   ├── endpoints.ts
│   │   └── types.ts
│   ├── hooks/                    # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useOffline.ts
│   │   ├── useStreaming.ts
│   │   └── useWhatsApp.ts
│   ├── stores/                   # Zustand stores
│   │   ├── authStore.ts
│   │   ├── chatStore.ts
│   │   ├── cvStore.ts
│   │   └── uiStore.ts
│   └── utils/
│       ├── cn.ts                 # Class merge utility
│       ├── format.ts
│       └── validation.ts
│
├── styles/
│   └── themes/
│       ├── light.css
│       └── dark.css
│
└── public/
    ├── manifest.json             # PWA manifest
    ├── sw.js                     # Service Worker
    └── icons/
```

### Chat Interface con Streaming

```typescript
// components/features/chat/ChatContainer.tsx
// Optimizado para conexiones lentas con streaming progresivo

interface ChatMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  status: 'sending' | 'streaming' | 'complete' | 'error';
}

// Caracteristicas clave:
// 1. Streaming via Server-Sent Events (SSE)
// 2. Reconexion automatica en redes inestables
// 3. Cola de mensajes offline
// 4. Indicadores de typing optimizados
// 5. Virtualizacion para historial largo
```

```
┌─────────────────────────────────────────┐
│  Entrevista: Desarrollador Backend      │
│  ─────────────────────────────────────  │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ AI: Hola! Soy tu entrevistador  │    │
│  │ virtual. Cuéntame sobre tu      │    │
│  │ experiencia con APIs REST...    │    │
│  └─────────────────────────────────┘    │
│                                         │
│       ┌─────────────────────────────┐   │
│       │ He trabajado 3 años con     │   │
│       │ Node.js y Express...        │   │
│       └─────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ AI: Excelente! Y como manejas   │    │
│  │ la autenticacion en tus APIs?   │    │
│  │ ▊ (streaming...)                │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ Escribe tu respuesta...          🎤││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

### CV Viewer con Lazy Loading

```typescript
// components/features/cv/CVViewer.tsx
// PDF.js cargado dinamicamente solo cuando se necesita

// Estrategia de carga:
// 1. Skeleton mientras carga PDF.js (~200KB)
// 2. Renderizado progresivo pagina por pagina
// 3. Cache local en IndexedDB
// 4. Modo offline con version cacheada
```

---

## PWA Configuration

### Service Worker Strategy

```javascript
// public/sw.js
// Estrategia: Stale-While-Revalidate para assets
// Network-First para API calls criticos

const CACHE_STRATEGIES = {
  // Assets estaticos: cache primero
  static: 'CacheFirst',

  // API de entrevistas: network primero (datos frescos)
  api: 'NetworkFirst',

  // CV del usuario: stale-while-revalidate
  userContent: 'StaleWhileRevalidate',

  // Imagenes: cache con TTL de 7 dias
  images: 'CacheFirst'
};
```

### Manifest PWA

```json
{
  "name": "Entrevistador Inteligente",
  "short_name": "EntrevistadorAI",
  "description": "Prepara tus entrevistas con IA",
  "start_url": "/dashboard",
  "display": "standalone",
  "orientation": "portrait",
  "theme_color": "#0F172A",
  "background_color": "#FFFFFF",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/dashboard.png",
      "sizes": "1080x1920",
      "type": "image/png",
      "form_factor": "narrow"
    }
  ],
  "shortcuts": [
    {
      "name": "Nueva Entrevista",
      "url": "/entrevistas/nueva",
      "icons": [{"src": "/icons/interview.png", "sizes": "96x96"}]
    },
    {
      "name": "Mi CV",
      "url": "/cv",
      "icons": [{"src": "/icons/cv.png", "sizes": "96x96"}]
    }
  ]
}
```

### Offline Capabilities

```
┌─────────────────────────────────────────────────────────────┐
│                    Modo Offline                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Disponible Offline:                                         │
│  ├── Dashboard (datos cacheados)                            │
│  ├── CV viewer (PDFs descargados)                           │
│  ├── Historial de entrevistas                               │
│  ├── Configuracion de perfil                                │
│  └── Tips y recursos educativos                             │
│                                                              │
│  Requiere Conexion:                                          │
│  ├── Nueva entrevista (streaming IA)                        │
│  ├── Subir/modificar CV                                     │
│  ├── Pagos                                                  │
│  └── Matching con vacantes                                  │
│                                                              │
│  Queue Offline:                                              │
│  ├── Mensajes pendientes se sincronizan al reconectar       │
│  ├── Cambios de perfil en cola                              │
│  └── Indicador visual de estado de sincronizacion           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Performance Optimization

### Bundle Splitting Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    Bundle Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Initial Bundle (< 200KB gzipped):                          │
│  ├── React runtime (~45KB)                                  │
│  ├── Next.js core (~30KB)                                   │
│  ├── Zustand (~3KB)                                         │
│  ├── Core UI components (~40KB)                             │
│  ├── Layout & routing (~20KB)                               │
│  └── Critical CSS (~10KB)                                   │
│                                                              │
│  Lazy Loaded Chunks:                                         │
│  ├── Chat module (~80KB) - on /entrevistas                  │
│  ├── PDF.js (~200KB) - on CV view                           │
│  ├── Charts (~50KB) - on dashboard metrics                  │
│  ├── Payment SDK (~40KB) - on checkout                      │
│  ├── Framer Motion (~30KB) - on animation-heavy pages       │
│  └── Rich text editor (~60KB) - on CV edit                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Image Optimization

```typescript
// next.config.js
const config = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [320, 420, 640, 768, 1024],
    minimumCacheTTL: 60 * 60 * 24 * 30, // 30 dias
    remotePatterns: [
      { hostname: 'storage.entrevistadorinteligente.pe' }
    ]
  }
};

// Uso con responsive images
// <Image
//   src={avatar}
//   sizes="(max-width: 768px) 64px, 96px"
//   placeholder="blur"
// />
```

### Critical CSS

```css
/* globals.css - Solo estilos criticos inline */
/* Tailwind se purga agresivamente */

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    /* ... theme tokens ... */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... dark theme ... */
  }
}
```

---

## Integracion WhatsApp

### Deep Links y Compartir

```typescript
// lib/hooks/useWhatsApp.ts

interface WhatsAppShare {
  // Compartir CV
  shareCV: (cvUrl: string, message: string) => void;

  // Notificacion de entrevista completada
  shareResults: (resultUrl: string) => void;

  // Invitar amigos
  shareReferral: (referralCode: string) => void;
}

// Genera links de WhatsApp optimizados para Peru
const generateWhatsAppLink = (phone: string, message: string) => {
  const encodedMessage = encodeURIComponent(message);
  return `https://wa.me/51${phone}?text=${encodedMessage}`;
};

// Web Share API con fallback a WhatsApp
const shareWithFallback = async (data: ShareData) => {
  if (navigator.share && navigator.canShare(data)) {
    await navigator.share(data);
  } else {
    // Fallback: WhatsApp direct
    window.open(generateWhatsAppLink('', data.text || ''));
  }
};
```

### Notificaciones via WhatsApp Business API

```
┌─────────────────────────────────────────────────────────────┐
│              WhatsApp Integration Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Usuario                   App                    WhatsApp   │
│     │                       │                         │      │
│     │  Completa entrevista  │                         │      │
│     │──────────────────────>│                         │      │
│     │                       │                         │      │
│     │                       │  Envia notificacion     │      │
│     │                       │────────────────────────>│      │
│     │                       │                         │      │
│     │                       │     "Tu entrevista      │      │
│     │                       │      esta lista!        │      │
│     │                       │      Ver resultados:    │      │
│     │                       │      [link]"            │      │
│     │<──────────────────────────────────────────────────     │
│     │                                                        │
│     │  Click en link                                         │
│     │───────────────────────>│                              │
│     │                       │                               │
│     │      Deep link a      │                               │
│     │      resultados       │                               │
│     │<──────────────────────│                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Dark/Light Mode

### Theme Implementation

```typescript
// lib/stores/uiStore.ts
interface UIStore {
  theme: 'light' | 'dark' | 'system';
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
}

// Detecta preferencia del sistema
// Persiste eleccion del usuario
// Aplica sin flash (script blocking en <head>)
```

```
┌─────────────────────────────────────────────────────────────┐
│                    Theme System                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CSS Variables (design tokens):                             │
│  ├── --background                                           │
│  ├── --foreground                                           │
│  ├── --primary                                              │
│  ├── --secondary                                            │
│  ├── --accent                                               │
│  ├── --muted                                                │
│  └── --border                                               │
│                                                              │
│  Implementacion:                                             │
│  1. Script bloqueante en <head> detecta preferencia         │
│  2. Aplica clase 'dark' al <html> antes de render           │
│  3. CSS usa variables, 0 flash de contenido                 │
│  4. Toggle guarda en localStorage + Zustand                 │
│                                                              │
│  Accesibilidad:                                              │
│  - Contraste minimo WCAG AA (4.5:1)                         │
│  - respeta prefers-reduced-motion                           │
│  - Focus visible en todos los interactivos                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Consecuencias

### Positivas

| Beneficio | Descripcion |
|-----------|-------------|
| **Performance optimizada** | Bundle <200KB, LCP <2.5s alcanzable |
| **PWA nativa** | Instalable, offline, push notifications |
| **Developer Experience** | Next.js + TypeScript + Tailwind = productividad alta |
| **Code sharing futuro** | Logica en hooks reutilizable con React Native |
| **SEO optimizado** | SSR/SSG para landing pages publicas |
| **Costo reducido** | No requiere desarrollo nativo inicial |
| **Accesibilidad** | shadcn/ui tiene ARIA integrado |

### Negativas

| Desventaja | Mitigacion |
|------------|------------|
| **No es app nativa** | PWA cubre 90% de casos, RN en Fase 2 |
| **Sin acceso a APIs nativas** | Web APIs cubren camara, mic, geolocalizacion |
| **Play Store ausente inicial** | PWA instalable, TWA posible si necesario |
| **Ecosistema fragmentado** | shadcn evita dependencia de biblioteca UI |
| **Learning curve RSC** | Equipo ya tiene experiencia React |

### Riesgos

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Bundle size crece | Media | Alto | Webpack bundle analyzer, CI checks |
| PWA no adoptada | Baja | Medio | Onboarding claro, beneficios visibles |
| Streaming falla en 3G | Media | Alto | Queue offline, retry logic |
| shadcn breaking changes | Baja | Bajo | Components son copy-paste, no dependencia |
| React Native migration compleja | Media | Medio | Hooks compartidos desde dia 1 |

---

## Alternativas Consideradas

### Opcion A: Remix

**Pros:**
- Mejor manejo de forms y mutations
- Nested routes elegantes
- Loader/action pattern limpio

**Contras:**
- Ecosistema mas pequeno
- Menos integraciones enterprise
- Equipo sin experiencia previa
- PWA requiere config adicional

**Veredicto:** Rechazado - Next.js tiene mejor ecosistema y experiencia de equipo

### Opcion B: Nuxt (Vue)

**Pros:**
- Vue mas simple para juniors
- Nuxt 3 con Nitro es performante
- Buen soporte TypeScript

**Contras:**
- Equipo es React-first
- Menor pool de talento en Peru
- React Native no comparte codigo con Vue
- Menos componentes UI maduros

**Veredicto:** Rechazado - Reentrenamiento costoso, no alinea con estrategia mobile

### Opcion C: Flutter Web + Mobile

**Pros:**
- Un codebase para todo
- UI consistente cross-platform
- Dart es type-safe

**Contras:**
- Bundle size web enorme (2MB+)
- SEO casi imposible
- Rendering custom, no HTML semantico
- Accesibilidad limitada
- Talento Flutter escaso en Peru

**Veredicto:** Rechazado - Inaceptable para web mobile-first con restricciones de red

### Opcion D: React Native Web desde inicio

**Pros:**
- Un codebase real para web y mobile
- Componentes nativos en mobile

**Contras:**
- DX web inferior a Next.js
- SSR complejo de implementar
- Bundle size mayor
- Menos optimizaciones web disponibles

**Veredicto:** Rechazado - Sacrifica demasiado en web para beneficio mobile futuro

### Opcion E: MUI (Material UI)

**Pros:**
- Componentes completos out-of-the-box
- Theming robusto
- Documentacion extensa

**Contras:**
- Bundle size significativo (~100KB+ base)
- Estilo "Google" generico
- Customizacion requiere override de estilos
- Runtime CSS-in-JS impacta performance

**Veredicto:** Rechazado - shadcn/ui ofrece misma funcionalidad sin runtime cost

### Opcion F: Redux Toolkit

**Pros:**
- Estandar de la industria
- DevTools excelentes
- RTK Query para server state

**Contras:**
- Boilerplate mayor que Zustand
- Bundle size mayor (~15KB vs ~3KB)
- Overkill para app de este tamano

**Veredicto:** Rechazado - Zustand + React Query es mas ligero y suficiente

---

## Plan de Implementacion

### Fase 1: MVP Web + PWA (Semanas 1-12)

```
Semana 1-2:   Setup proyecto, design system base
Semana 3-4:   Auth, onboarding wizard
Semana 5-6:   CV upload/viewer
Semana 7-8:   Chat interface con streaming
Semana 9-10:  Dashboard, metricas
Semana 11:    Payments (Yape, tarjetas)
Semana 12:    PWA config, testing, optimizacion
```

### Fase 2: Optimizacion + Android (Meses 4-8)

```
Mes 4:    Performance audit, optimizaciones
Mes 5:    React Native setup, shared hooks
Mes 6-7:  Android app desarrollo
Mes 8:    Play Store launch
```

### Fase 3: iOS + Expansion (Ano 2)

```
Q1:       iOS app desarrollo
Q2:       App Store launch
Q3-Q4:    Expansion regional (Colombia, Mexico)
```

---

## Criterios de Exito

| Metrica | Target MVP | Target Ano 1 |
|---------|------------|--------------|
| LCP | < 2.5s | < 2.0s |
| FID | < 100ms | < 50ms |
| CLS | < 0.1 | < 0.05 |
| Bundle initial | < 200KB | < 150KB |
| Lighthouse score | > 85 | > 95 |
| PWA install rate | > 10% | > 25% |
| Offline usage | Disponible | > 15% sessions |
| Mobile traffic | > 70% | > 80% |

---

## Herramientas de Monitoreo

```
┌─────────────────────────────────────────────────────────────┐
│                    Observability Stack                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Performance:                                                │
│  ├── Vercel Analytics (Core Web Vitals)                     │
│  ├── Sentry Performance (tracing)                           │
│  └── Lighthouse CI (PRs)                                    │
│                                                              │
│  Error Tracking:                                             │
│  ├── Sentry (frontend errors)                               │
│  └── LogRocket (session replay)                             │
│                                                              │
│  Analytics:                                                  │
│  ├── Mixpanel (product analytics)                           │
│  ├── Google Analytics 4 (traffic)                           │
│  └── Hotjar (heatmaps, recordings)                          │
│                                                              │
│  Uptime:                                                     │
│  └── Checkly (synthetic monitoring)                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Decision Final

**Next.js 14+ con Tailwind/shadcn, Zustand/React Query, y PWA-first** es la estrategia optima porque:

1. **Performance optimizada para Peru**: Bundle <200KB, offline-capable, funciona en 3G
2. **Velocidad de desarrollo**: Stack familiar, componentes listos, DX excelente
3. **Costo reducido**: No requiere desarrollo nativo inicial
4. **Escalabilidad tecnica**: RSC, streaming, ISR para diferentes casos de uso
5. **Preparado para mobile nativo**: Hooks compartibles con React Native futuro
6. **Accesibilidad incluida**: shadcn/ui con ARIA, contraste WCAG
7. **SEO para crecimiento organico**: SSR/SSG para landing pages

---

## Referencias

- [Next.js 14 Documentation](https://nextjs.org/docs)
- [shadcn/ui Components](https://ui.shadcn.com)
- [Zustand State Management](https://zustand-demo.pmnd.rs)
- [TanStack Query](https://tanstack.com/query)
- [Web.dev Core Web Vitals](https://web.dev/vitals/)
- [PWA Builder](https://www.pwabuilder.com)
- [React Native for Web](https://necolas.github.io/react-native-web/)

---

## Registro de Cambios

| Version | Fecha | Autor | Cambio |
|---------|-------|-------|--------|
| 1.0 | 2026-01-31 | Equipo Arquitectura | Creacion inicial |
