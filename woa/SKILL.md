---
name: woa
description: Audita un sitio web en términos de SEO, performance y accesibilidad. Genera un reporte detallado con checklist, problemas críticos y plan de acción. Uso: /woa <url>
argument-hint: <url>
user-invocable: true
allowed-tools:
  - Bash
  - WebFetch
  - WebSearch
  - Write
---

Vas a auditar el sitio web en la URL proporcionada para evaluar su estado en tres dimensiones: SEO, Performance y Accesibilidad.

**URL a auditar:** $ARGUMENTS

---

## Paso 1 — Fetch de la página principal

Usá WebFetch para obtener el contenido de la URL. El prompt debe pedir explícitamente:
- Título de la página (`<title>`) y meta description
- Todos los headings presentes (H1, H2, H3) con su texto exacto
- Atributo `lang` del elemento `<html>`
- Meta tags relevantes: `robots`, `canonical`, `viewport`, Open Graph (`og:title`, `og:description`, `og:image`), Twitter Card
- Todas las imágenes con sus atributos `src`, `alt`, `width`, `height`, `loading`
- Scripts externos (`<script src="...">`) y si tienen atributo `defer` o `async`
- Hojas de estilo externas (`<link rel="stylesheet">`)
- Fuentes externas (Google Fonts, Adobe Fonts, etc.)
- Formularios y sus campos, con `<label>` asociados
- Atributos `aria-label`, `aria-labelledby`, `role` presentes en el HTML
- Todos los links `<a>` con sus `href` y texto visible
- Presencia de `<noscript>` tags
- Structured data (`<script type="application/ld+json">`) si existe

Si el WebFetch no devuelve los atributos HTML completos, hacé un segundo fetch con el prompt: "Dame el HTML fuente completo de la página, incluyendo todos los atributos de cada elemento: meta tags, imágenes con alt/width/height/loading, scripts con defer/async, links con href, y cualquier bloque JSON-LD."

## Paso 1b — Extracción de HTML crudo via curl (fuente de verdad)

WebFetch convierte el HTML a markdown y puede perder atributos clave. Ejecutá los siguientes dos comandos Bash en paralelo para obtener el HTML real sin transformar:

**Comando 1 — HEAD, imágenes, scripts, headings, formularios, JSON-LD, ARIA:**

```bash
curl -s -L "[URL_AUDITADA]" -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" | node -e "
const chunks = [];
process.stdin.on('data', c => chunks.push(c));
process.stdin.on('end', () => {
  const html = chunks.join('');

  const headMatch = html.match(/<head[^>]*>([\s\S]*?)<\/head>/i);
  const head = headMatch ? headMatch[1] : '';

  console.log('=== HTML TAG ===');
  const htmlTag = html.match(/<html[^>]*>/i);
  console.log(htmlTag ? htmlTag[0] : 'NOT FOUND');

  console.log('\n=== FULL HEAD ===');
  console.log(head);

  console.log('\n=== TITLE ===');
  const titleMatch = head.match(/<title[^>]*>(.*?)<\/title>/i);
  console.log(titleMatch ? titleMatch[1] : 'NOT FOUND');

  console.log('\n=== META TAGS ===');
  const metas = head.match(/<meta[^>]+>/gi) || [];
  metas.forEach(m => console.log(m));

  console.log('\n=== CANONICAL & LINKS ===');
  const links = head.match(/<link[^>]+>/gi) || [];
  links.forEach(l => console.log(l));

  console.log('\n=== IMAGES ===');
  const imgs = html.match(/<img[^>]+>/gi) || [];
  imgs.forEach(i => console.log(i));

  console.log('\n=== HEADINGS ===');
  const headings = html.match(/<h[1-6][^>]*>[\s\S]*?<\/h[1-6]>/gi) || [];
  headings.forEach(h => console.log(h.replace(/<[^>]+>/g, '').trim() + ' [' + h.match(/<(h[1-6])/i)[1] + ']'));

  console.log('\n=== JSON-LD ===');
  const jsonld = html.match(/<script[^>]*type=[\"'\`]application\/ld\+json[\"'\`][^>]*>[\s\S]*?<\/script>/gi) || [];
  console.log(jsonld.length ? jsonld.join('\n') : 'NONE');

  console.log('\n=== ARIA & ROLES ===');
  const ariaMatches = html.match(/(?:aria-[a-z]+|role)=[\"'][^\"']+[\"']/gi) || [];
  console.log([...new Set(ariaMatches)].join('\n') || 'NONE');

  console.log('\n=== NOSCRIPT ===');
  const noscript = html.match(/<noscript[^>]*>[\s\S]*?<\/noscript>/gi) || [];
  noscript.forEach(n => console.log(n));
});
"
```

**Comando 2 — Scripts inline, links y formularios:**

```bash
curl -s -L "[URL_AUDITADA]" -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" | node -e "
const chunks = [];
process.stdin.on('data', c => chunks.push(c));
process.stdin.on('end', () => {
  const html = chunks.join('');

  console.log('=== INLINE SCRIPTS (primeros 300 chars c/u) ===');
  const inlineScripts = html.match(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/gi) || [];
  inlineScripts.forEach((s, i) => console.log('Script ' + (i+1) + ' [' + (s.match(/type=[\"']([^\"']+)[\"']/i)||['',''])[1] + ']:', s.substring(0, 300)));

  console.log('\n=== SCRIPTS EXTERNOS ===');
  const extScripts = html.match(/<script[^>]+src=[^>]*>/gi) || [];
  extScripts.forEach(s => console.log(s));

  console.log('\n=== ALL LINKS (href + texto) ===');
  const links = html.match(/<a[^>]*href[^>]*>[\s\S]*?<\/a>/gi) || [];
  links.forEach(l => {
    const href = l.match(/href=[\"']([^\"']+)[\"']/i);
    const text = l.replace(/<[^>]+>/g, '').trim().substring(0, 80);
    if (href) console.log(href[1] + ' | ' + text);
  });

  console.log('\n=== FORMS ===');
  const forms = html.match(/<form[\s\S]*?<\/form>/gi) || [];
  if (forms.length === 0) console.log('No <form> tags in static HTML (may be JS-rendered)');
  forms.forEach((f, i) => {
    const inputs = f.match(/<input[^>]+>/gi) || [];
    const labels = f.match(/<label[^>]*>[\s\S]*?<\/label>/gi) || [];
    console.log('FORM ' + (i+1) + ': inputs=' + inputs.length + ' labels=' + labels.length);
    inputs.forEach(inp => console.log('  INPUT:', inp));
    labels.forEach(lbl => console.log('  LABEL:', lbl));
  });
});
"
```

Usá los datos de estos comandos como **fuente primaria** para completar las secciones de SEO, imágenes, scripts y accesibilidad del reporte. Los datos de curl reflejan el HTML real enviado por el servidor, independientemente del renderizado JS.

> **Nota:** Si el sitio requiere JS para renderizar contenido (SPA, Next.js, etc.), el HTML crudo puede mostrar menos elementos que el HTML renderizado. En ese caso, los audits de Lighthouse del Paso 2 tienen precedencia para los elementos que dependen de JS.

## Paso 2 — Análisis runtime con PageSpeed Insights

**IMPORTANTE — API Key:** Antes de fetchear, ejecutá via Bash: `echo $PAGESPEED_API_KEY` para obtener el valor real de la clave. Reemplazá `[API_KEY]` en las URLs con ese valor. Sin la clave real la API devuelve errores 429.

Fetcheá en paralelo las dos estrategias:

- `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=[URL_AUDITADA]&strategy=mobile&key=[API_KEY]`
- `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=[URL_AUDITADA]&strategy=desktop&key=[API_KEY]`

De cada respuesta, extraé:

**Métricas de laboratorio (Lighthouse):**
- `lighthouseResult.categories.performance.score` × 100 → Puntuación Lighthouse
- `lighthouseResult.audits["first-contentful-paint"].displayValue` → FCP
- `lighthouseResult.audits["largest-contentful-paint"].displayValue` → LCP
- `lighthouseResult.audits["total-blocking-time"].displayValue` → TBT (proxy de INP)
- `lighthouseResult.audits["cumulative-layout-shift"].displayValue` → CLS
- `lighthouseResult.audits["speed-index"].displayValue` → Speed Index

**Metadatos detectados por Lighthouse (úsalos para corregir el análisis SEO del Paso 1):**
- `lighthouseResult.audits["document-title"].displayValue` → Título real de la página
- `lighthouseResult.audits["meta-description"].displayValue` → Meta description real
- `lighthouseResult.audits["hreflang"].displayValue` → Atributo lang / hreflang
- `lighthouseResult.audits["canonical"].displayValue` → URL canónica
- `lighthouseResult.audits["viewport"].displayValue` → Meta viewport
- `lighthouseResult.audits["document-title"].score` → 1 = presente, 0 = ausente
- `lighthouseResult.audits["meta-description"].score` → 1 = presente, 0 = ausente

Estos valores tienen precedencia sobre los detectados via WebFetch, ya que Lighthouse analiza el HTML renderizado real.

**Datos de campo reales (CrUX) — si están disponibles:**
- `loadingExperience.metrics.LARGEST_CONTENTFUL_PAINT_MS` → LCP real (p75)
- `loadingExperience.metrics.CUMULATIVE_LAYOUT_SHIFT_SCORE` → CLS real (p75)
- `loadingExperience.metrics.INTERACTION_TO_NEXT_PAINT` → INP real (p75)
- `loadingExperience.overall_category` → Veredicto general del campo (FAST / AVERAGE / SLOW)

Si la API devuelve error o la URL no tiene datos de campo reales (sitios con poco tráfico en CrUX), notalo en el reporte y usá solo los datos de laboratorio.

Incluí estos datos en la sección **⚡ Performance** del reporte, en un bloque separado llamado "Core Web Vitals (Runtime)".

## Paso 3 — Verificar recursos clave del dominio en paralelo

A partir del dominio detectado, fetcheá en paralelo:

- `[dominio]/robots.txt` — verificar si existe y qué directivas tiene
- `[dominio]/sitemap.xml` — verificar si existe y si es accesible
- Si hay links internos relevantes (página de inicio, página de contacto, categorías principales), fetcheá hasta 2 páginas adicionales para evaluar consistencia de SEO on-page

Registrá para cada uno:
- ¿Existe y responde correctamente?
- Contenido relevante encontrado

## Paso 4 — Consultar fuentes de referencia actualizadas

Fetcheá en paralelo las siguientes fuentes para usar criterios vigentes:

- `https://developers.google.com/search/docs/fundamentals/seo-starter-guide` — Guía SEO de Google
- `https://web.dev/performance` — Métricas de performance (Core Web Vitals)
- `https://www.w3.org/WAI/WCAG22/quickref/` — Referencia rápida WCAG 2.2

Extraé de cada una los criterios más relevantes y actualizados. Si alguna URL falla, continuá con el conocimiento disponible y notalo en el reporte.

## Paso 5 — Generar el reporte de auditoría

Producí un reporte estructurado con el siguiente formato exacto:

---

# Auditoría WOA — [dominio]

**URL auditada:** [url]
**Fecha:** [fecha actual]
**Tecnología detectada:** [CMS, framework o stack inferido si es posible]

---

## Resultado general

| Dimensión | Puntuación | Veredicto |
|---|---|---|
| SEO | [0-100] | [ÓPTIMO ✅ / MEJORABLE ⚠️ / CRÍTICO ❌] |
| Performance | [0-100] | [ÓPTIMO ✅ / MEJORABLE ⚠️ / CRÍTICO ❌] |
| Accesibilidad | [0-100] | [ÓPTIMO ✅ / MEJORABLE ⚠️ / CRÍTICO ❌] |
| **Puntuación global** | [promedio] | [veredicto general] |

> La puntuación es una estimación basada en análisis estático del HTML. Para métricas de runtime (LCP, CLS, FID) se recomienda complementar con Google PageSpeed Insights o Lighthouse.

---

## 🔍 SEO

### Meta y estructura básica

| Elemento | Estado | Detalle |
|---|---|---|
| `<title>` presente y optimizado | ✅/⚠️/❌ | [texto encontrado, longitud en caracteres] |
| Meta description presente | ✅/⚠️/❌ | [texto encontrado, longitud en caracteres] |
| H1 único y presente | ✅/⚠️/❌ | [texto del H1 o motivo de falla] |
| Jerarquía de headings correcta (H1→H2→H3) | ✅/⚠️/❌ | [estructura detectada] |
| Atributo `lang` declarado en `<html>` | ✅/⚠️/❌ | [valor encontrado o ausente] |
| Meta `viewport` presente | ✅/⚠️/❌ | [valor encontrado] |
| URL canónica (`rel=canonical`) | ✅/⚠️/❌ | [URL canónica encontrada o ausente] |
| Meta `robots` | ✅/⚠️/❌ | [directivas encontradas: index/noindex, follow/nofollow] |
| HTTPS activo | ✅/❌ | [basado en la URL] |

### Indexabilidad

| Elemento | Estado | Detalle |
|---|---|---|
| `robots.txt` accesible | ✅/⚠️/❌ | [existe y permite indexación o bloquea] |
| `sitemap.xml` accesible | ✅/⚠️/❌ | [existe o no encontrado] |
| Sin directivas `noindex` problemáticas | ✅/⚠️/❌ | [análisis de robots y meta robots] |

### Open Graph y redes sociales

| Elemento | Estado | Detalle |
|---|---|---|
| `og:title` | ✅/⚠️/❌ | [texto encontrado o ausente] |
| `og:description` | ✅/⚠️/❌ | [texto encontrado o ausente] |
| `og:image` | ✅/⚠️/❌ | [URL de imagen encontrada o ausente] |
| Twitter Card | ✅/⚠️/❌ | [tipo encontrado o ausente] |

### Structured Data

| Elemento | Estado | Detalle |
|---|---|---|
| JSON-LD presente | ✅/⚠️/❌ | [tipos de schema encontrados: Article, Product, Organization, etc.] |
| Schema coherente con el contenido | ✅/⚠️/❌ | [evaluación de coherencia] |

**Puntuación SEO estimada:** [X/100]
**Issues críticos SEO:** [N]

---

## ⚡ Performance

### Core Web Vitals (Runtime)

**Datos de laboratorio — Mobile:**

| Métrica | Valor | Estado |
|---|---|---|
| Puntuación Lighthouse Performance | [0-100] | ✅/⚠️/❌ |
| LCP (Largest Contentful Paint) | [valor] | ✅/⚠️/❌ |
| CLS (Cumulative Layout Shift) | [valor] | ✅/⚠️/❌ |
| TBT (Total Blocking Time) | [valor] | ✅/⚠️/❌ |
| FCP (First Contentful Paint) | [valor] | ✅/⚠️/❌ |
| Speed Index | [valor] | ✅/⚠️/❌ |

**Datos de laboratorio — Desktop:**

| Métrica | Valor | Estado |
|---|---|---|
| Puntuación Lighthouse Performance | [0-100] | ✅/⚠️/❌ |
| LCP | [valor] | ✅/⚠️/❌ |
| CLS | [valor] | ✅/⚠️/❌ |
| TBT | [valor] | ✅/⚠️/❌ |

**Datos de campo reales (CrUX):** [disponibles / no disponibles — motivo]

| Métrica | p75 | Veredicto de campo |
|---|---|---|
| LCP real | [valor o N/A] | FAST / AVERAGE / SLOW |
| CLS real | [valor o N/A] | FAST / AVERAGE / SLOW |
| INP real | [valor o N/A] | FAST / AVERAGE / SLOW |
| Veredicto general | — | [overall_category] |

> Umbrales Google: LCP ≤2.5s ✅ / ≤4s ⚠️ / >4s ❌ · CLS ≤0.1 ✅ / ≤0.25 ⚠️ / >0.25 ❌ · INP ≤200ms ✅ / ≤500ms ⚠️ / >500ms ❌

### Imágenes

| Elemento | Estado | Detalle |
|---|---|---|
| Imágenes en formato moderno (WebP / AVIF) | ✅/⚠️/❌ | [formatos detectados, cantidad en PNG/JPG legacy] |
| Atributos `width` y `height` en imágenes | ✅/⚠️/❌ | [N imágenes sin dimensiones declaradas] |
| `loading="lazy"` en imágenes below-the-fold | ✅/⚠️/❌ | [N imágenes sin lazy load] |
| Imágenes con `alt` descriptivo | ✅/⚠️/❌ | [N imágenes sin alt o con alt vacío] |

### Scripts y CSS

| Elemento | Estado | Detalle |
|---|---|---|
| Scripts con `defer` o `async` | ✅/⚠️/❌ | [N scripts bloqueantes sin defer/async] |
| CSS crítico inline o en `<head>` | ✅/⚠️/❌ | [hojas de estilo externas que bloquean render] |
| Fuentes externas optimizadas (`display=swap`) | ✅/⚠️/❌ | [fuentes detectadas y si usan font-display:swap] |
| Sin recursos de terceros innecesarios | ✅/⚠️/❌ | [scripts y recursos externos detectados] |

### General

| Elemento | Estado | Detalle |
|---|---|---|
| HTTPS activo | ✅/❌ | [basado en la URL] |
| Sin redirecciones evidentes en la URL | ✅/⚠️/❌ | [redirecciones detectadas o no] |
| Viewport configurado para mobile | ✅/⚠️/❌ | [meta viewport encontrado] |

**Puntuación Performance estimada:** [X/100]
**Issues críticos Performance:** [N]

> ⚠️ Para métricas reales de Core Web Vitals (LCP, CLS, INP) complementar con: https://pagespeed.web.dev/

---

## ♿ Accesibilidad

### Imágenes y medios

| Elemento | Estado | Detalle |
|---|---|---|
| Todas las imágenes tienen `alt` | ✅/⚠️/❌ | [N imágenes sin alt, listar src si es posible] |
| Imágenes decorativas con `alt=""` | ✅/⚠️/❌ | [evaluación de uso correcto de alt vacío] |

### Estructura y navegación

| Elemento | Estado | Detalle |
|---|---|---|
| Un único H1 por página | ✅/⚠️/❌ | [cantidad de H1 encontrados] |
| Links con texto descriptivo (no "click aquí") | ✅/⚠️/❌ | [links genéricos detectados] |
| Idioma declarado (`lang` en `<html>`) | ✅/⚠️/❌ | [valor encontrado] |
| Elementos interactivos con foco visible | ✅/⚠️/❌ | [evaluación de elementos `<a>`, `<button>`, `<input>`] |
| Skip links presentes | ✅/⚠️/❌ | [presente o no] |

### Formularios

| Elemento | Estado | Detalle |
|---|---|---|
| Todos los inputs tienen `<label>` asociado | ✅/⚠️/❌ | [N inputs sin label o aria-label] |
| Campos requeridos marcados explícitamente | ✅/⚠️/❌ | [uso de `required` o `aria-required`] |
| Mensajes de error descriptivos (si aplica) | ✅/⚠️/❌ | [evaluación estática] |

### ARIA y semántica

| Elemento | Estado | Detalle |
|---|---|---|
| Roles ARIA usados correctamente | ✅/⚠️/❌ | [roles detectados y evaluación] |
| `aria-label` en elementos sin texto visible | ✅/⚠️/❌ | [botones de ícono, etc.] |
| Estructura semántica (nav, main, header, footer) | ✅/⚠️/❌ | [landmarks HTML5 detectados] |

**Puntuación Accesibilidad estimada:** [X/100]
**Issues críticos Accesibilidad:** [N]

> ⚠️ Para validación completa de accesibilidad complementar con axe DevTools o WAVE.

---

## Problemas críticos (requieren acción inmediata)

[Lista numerada de los issues más graves detectados en cualquier dimensión. Si no hay ninguno, escribir "Ninguno".]

## Observaciones (mejoras recomendadas)

[Lista de mejoras que no son bloqueantes pero impactan positivamente en rankings, conversiones o experiencia de usuario.]

## Plan de acción recomendado

[Lista ordenada por impacto estimado (Alto / Medio / Bajo) de lo que hay que corregir o implementar.]

---

*Auditoría generada con /woa · Análisis estático basado en HTML — complementar con Lighthouse y axe para validación completa*

---

## Paso 6 — Guardar el reporte como Markdown

Una vez mostrado el reporte completo en pantalla, guardalo como archivo Markdown.

### 6a — Determinar el nombre de archivo

Extraé el dominio limpio de la URL auditada (ej: `ejemplo-com` de `https://ejemplo.com/...`) y usalo como nombre:
- `C:/Claude/woa_[dominio].md`

### 6b — Escribir el archivo Markdown

Usando la herramienta `Write`, escribí exactamente el mismo contenido del reporte mostrado en pantalla (el texto completo del Paso 4, con todas las tablas, secciones y listas), sin modificar ni resumir nada.

### 6c — Confirmar al usuario

Una vez escrito el archivo, informá al usuario:

```
Markdown generado: C:\Claude\woa_[dominio].md
```
