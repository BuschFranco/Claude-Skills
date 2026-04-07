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
└── gaa/
    └── SKILL.md        # /gaa — Auditoría de sitios web para Google Ads
```

---

## Skills disponibles

### `/gaa` — Google Ads Auditor

Audita un sitio web y determina si cumple los requisitos legales y técnicos para correr campañas de Google Ads.

**Uso:** `/gaa <url>`

**Que hace:**
- Analiza el contenido de la landing page (textos, links, tracking, formularios)
- Verifica que todos los links de la pagina funcionen correctamente
- Consulta las politicas vigentes de Google Ads en tiempo real
- Detecta la region del sitio (Global, America, Europa, Asia) y evalua el cumplimiento legal correspondiente (GDPR, LGPD, APPI, DPDP, etc.)
- Genera un reporte estructurado con checklists, mapa de links y plan de accion
- Guarda el reporte como archivo `.md` en `C:/Claude/`

**Veredictos posibles:** `APTO ✅` / `APTO CON OBSERVACIONES ⚠️` / `NO APTO ❌`

---
