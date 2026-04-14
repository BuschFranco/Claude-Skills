# Claude Skills
---

## Instalacion

Para usar estas skills en Claude Code, copiá la carpeta de la skill dentro del directorio de skills de tu proyecto o usuario:

```
~/.claude/skills/     # skills globales (disponibles en cualquier proyecto)
.claude/skills/       # skills locales (solo en el proyecto actual)
```

Cada skill se activa como un comando slash: `/nombre-de-la-skill`.

---

## Mapa del repositorio

```
Claude Skills/
├── gaa/
│   └── SKILL.md        # /gaa — Auditoría de sitios web para Google Ads
└── woa/
    └── SKILL.md        # /woa — Auditoría web: SEO, Performance y Accesibilidad
```

---

## Skills disponibles

### `/gaa` — Google Ads Auditor

Audita un sitio web y determina si cumple los requisitos legales y técnicos para correr campañas de Google Ads.

**Uso:** `/gaa <url>`

**Que hace:**
- Analiza el contenido de la landing page (textos, links, tracking, formularios)
- Verifica `robots.txt` y `sitemap.xml` del dominio auditado
- Verifica que todos los links de la pagina funcionen correctamente
- Consulta las politicas vigentes de Google Ads en tiempo real
- Detecta la region del sitio (Global, America, Europa, Asia) y evalua el cumplimiento legal correspondiente (GDPR, LGPD, APPI, DPDP, etc.)
- Genera un reporte estructurado con checklists, mapa de links y plan de accion
- Guarda el reporte como archivo `.md` en `C:/Claude/`

**Veredictos posibles:** `APTO ✅` / `APTO CON OBSERVACIONES ⚠️` / `NO APTO ❌`

---

### `/woa` — Web Optimization Audit

Audita un sitio web en terminos de SEO, performance y accesibilidad. Genera un reporte detallado con checklist, problemas criticos y plan de accion.

**Uso:** `/woa <url>`

**Que hace:**
- Extrae metadatos SEO via WebFetch y HTML crudo via curl (fuente de verdad)
- Analiza Core Web Vitals con PageSpeed Insights API (mobile y desktop)
- Verifica `robots.txt`, `sitemap.xml` y paginas adicionales del dominio
- Evalua accesibilidad: alt en imagenes, jerarquia de headings, labels en formularios, ARIA
- Consulta guias de referencia actualizadas (Google, web.dev, WCAG 2.2)
- Genera un reporte con tablas de estado por categoria y plan de accion priorizado
- Guarda el reporte como archivo `.md` en `C:/Claude/`

**Dimensiones evaluadas:** SEO · Performance · Accesibilidad

**Variable de entorno requerida:**

```
PAGESPEED_API_KEY=tu_clave_de_google_cloud
```

La skill usa la [PageSpeed Insights API](https://developers.google.com/speed/docs/insights/v5/get-started) para obtener metricas de Core Web Vitals. Sin la clave la API devuelve errores 429. Obtené una clave gratuita en Google Cloud Console habilitando la API `pagespeedonline`. Defini la variable en tu entorno antes de correr la skill (`~/.bashrc`, `~/.zshrc` o equivalente en Windows).

---

*Skills desarrolladas para uso con Claude Code · JAMO*
