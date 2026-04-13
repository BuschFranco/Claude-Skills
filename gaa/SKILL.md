---
name: gaa
description: Audita un sitio web y determina si es apto para lanzar campañas en Google Ads. Evalúa cumplimiento legal por región (Global, América, Europa, Asia). Uso: /gaa <url>
argument-hint: <url>
user-invocable: true
allowed-tools:
  - WebFetch
  - WebSearch
  - Write
---

Vas a auditar el sitio web en la URL proporcionada para determinar si cumple los requisitos legales y técnicos necesarios para correr campañas de Google Ads.

**URL a auditar:** $ARGUMENTS

---

## Paso 1 — Fetch de la página principal

Usá WebFetch para obtener el contenido de la URL. El prompt debe pedir explícitamente:
- Título y meta description
- Todo el texto visible (headlines, body, CTAs, disclaimers, footer)
- **Todos los atributos `href` de cada elemento `<a>` de la página** — especialmente footer, navegación y disclaimers. Necesitás las URLs completas, no solo el texto del enlace.
- Scripts de tracking presentes (GTM, GA4, Meta Pixel, etc.) con sus IDs
- Formularios y sus campos
- Precio o descripción del producto/servicio
- Información de contacto visible
- Cualquier aviso legal, cookie banner o checkbox de consentimiento

## Paso 2 — Verificar recursos clave del dominio y links de la landing

### 2a — Verificar robots.txt y sitemap.xml

A partir del dominio de la URL auditada, fetcheá en paralelo:

- `[dominio]/robots.txt` — verificar si existe. Si existe, anotá si bloquea bots de Google Ads o si contiene directivas `Disallow` relevantes.
- `[dominio]/sitemap.xml` — verificar si existe y si es accesible.

Registrá para cada uno:
- ¿Existe y responde correctamente? (200 / 404 / error)
- Contenido relevante encontrado (directivas que afecten indexación o rastreo de anuncios)

Incluí los resultados en la sección **Tracking y Analytics** del reporte bajo un bloque separado "Indexabilidad", o como observación en los problemas detectados si corresponde.

### 2b — Obtener el listado completo de links de la landing

Del contenido obtenido en el Paso 1, extraé **todos** los valores `href` encontrados.

**Si el Paso 1 no devolvió las URLs completas (solo devolvió texto de links sin los href):** hacé un segundo WebFetch de la misma URL principal con el prompt: "Dame TODOS los atributos href de los elementos `<a>` de la página. Necesito las URLs completas de cada link, no el texto. Incluí absolutamente todos: footer, nav, body, CTAs, botones."

Con el listado completo, clasificá cada link en una de estas categorías:
- **Legal** → Privacy Policy, Terms, Cookies, Contact, About, Impressum, Unsubscribe
- **CTA** → botones de acción principal (Comprar, Subscribe, Get Started, Descargar, etc.)
- **Externo** → links que apuntan a dominios distintos al de la LP
- **Interno** → links dentro del mismo dominio
- **Ancla** → `#seccion` — no requieren verificación
- **Exit** → links de salida explícitos (ej. botón "Exit", "Back to Google")

Excluí de la verificación: links `#ancla`, `javascript:void`, `mailto:`, `tel:` y redes sociales conocidas (facebook.com, instagram.com, twitter.com, linkedin.com, youtube.com).

### 2c — Fetchear todos los links relevantes

Hacé WebFetch de **todos** los links de categoría Legal, CTA y Externo — en paralelo donde sea posible. **Usá exactamente el href encontrado, nunca construyas rutas.**

Para cada link fetcheado, registrá:
- ¿Carga correctamente? (✅ / ❌ con el error: 404, ECONNREFUSED, timeout, redirect)
- Si es un CTA: ¿a dónde redirige? ¿es coherente con el producto?
- Si es externo: ¿el dominio es legítimo o sospechoso?
- Si es legal: ¿el contenido existe y es suficiente?

Solo si no existe **ningún href** relacionado con páginas legales en todo el HTML, intentá las rutas comunes relativas al dominio principal: `/privacidad`, `/privacy`, `/terms`, `/terminos`, `/agb`, `/cookies`, `/legal`, `/contact`.

Si algún link devuelve error (404, timeout, ECONNREFUSED), documentalo exactamente — no intentes rutas alternativas por cuenta propia.

## Paso 3 — Consultar políticas vigentes de Google Ads

Antes de evaluar el sitio, fetcheá las políticas actuales de Google Ads para usar los criterios vigentes (no los de entrenamiento). Hacé estas llamadas **en paralelo**:

**Siempre fetchear:**
- `https://support.google.com/adspolicy/answer/6368661` — Requisitos de destino (Destination requirements)
- `https://support.google.com/adspolicy/answer/6008942` — Requisitos de landing page

**Fetchear según el tipo de servicio detectado en el Paso 1:**
- Si hay suscripción o cobro recurrente → `https://support.google.com/adspolicy/answer/6368661` (Subscription services)
- Si hay facturación por operador / carrier billing / DCB → `https://support.google.com/adspolicy/answer/6020954` y `https://support.google.com/adspolicy/answer/6021546` (Billing by operator)
- Si es servicio financiero (crédito, inversión, seguros) → `https://support.google.com/adspolicy/answer/2464998` (Financial services)
- Si es salud o medicamentos → `https://support.google.com/adspolicy/answer/176038` (Healthcare)
- Si es juego de azar → `https://support.google.com/adspolicy/answer/1645359` (Gambling)

Para cada página, extraé: los requisitos obligatorios listados, qué está permitido/prohibido, y cualquier cambio o nota de actualización reciente.

Usá esta información como fuente de verdad para completar la sección "Políticas específicas de Google Ads" del reporte. Si alguna URL falla o redirige, notalo pero continuá con el resto.

## Paso 4 — Detectar región y ley aplicable

Determiná la región del sitio basándote en:
- Dominio TLD (`.at`, `.de`, `.fr`, `.es` → Europa | `.com.ar`, `.com.br`, `.mx`, `.co` → América | `.jp`, `.cn`, `.sg`, `.in`, `.pk` → Asia | `.com` → Global)
- Idioma del contenido
- Moneda y métodos de pago mencionados
- País del proveedor si está indicado

## Paso 5 — Generar el reporte de auditoría

Producí un reporte estructurado con el siguiente formato exacto:

---

# Auditoría Google Ads — [dominio]

**URL auditada:** [url]
**Fecha:** [fecha actual]
**Región detectada:** [Global / América / Europa / Asia]
**Producto/Servicio:** [descripción breve]

---

## Resultado general

[APTO ✅ / APTO CON OBSERVACIONES ⚠️ / NO APTO ❌]

[Una línea explicando el veredicto principal]

---

## Checklist Universal (aplica a todas las regiones)

| Requisito | Estado | Detalle |
|---|---|---|
| Política de Privacidad accesible | ✅/⚠️/❌ | [dónde está o por qué falta] |
| Términos y Condiciones | ✅/⚠️/❌ | [dónde está o por qué falta] |
| Información de contacto visible | ✅/⚠️/❌ | [qué info hay: email, tel, dirección] |
| Descripción clara del producto/servicio | ✅/⚠️/❌ | [qué se ofrece] |
| Precio visible (si aplica) | ✅/⚠️/❌ | [precio encontrado o N/A] |
| HTTPS activo | ✅/❌ | [basado en la URL] |
| Tracking de Google (GTM/GA4) | ✅/⚠️/❌ | [IDs encontrados] |
| Página funcional (no 404/error) | ✅/❌ | [estado] |
| Todos los links de la LP funcionales | ✅/⚠️/❌ | [N links verificados, N rotos — ver Mapa de links] |

---

## Checklist por Región

### 🌎 América (Argentina, Brasil, México, Colombia, etc.)

| Requisito | Estado | Detalle |
|---|---|---|
| Mención a ley local de datos (Ley 25.326 AR / LGPD BR / LFPDPPP MX) | ✅/⚠️/❌ | [encontrado o no] |
| Aviso de privacidad en formularios | ✅/⚠️/❌ | [descripción] |
| Información del responsable de datos | ✅/⚠️/❌ | [empresa/persona identificada] |
| Derechos del titular (acceso, rectificación, cancelación) | ✅/⚠️/❌ | [mencionados o no] |
| Cookie banner (recomendado, no obligatorio) | ✅/⚠️/❌ | [presente o no] |

**Nivel de riesgo América:** [BAJO / MEDIO / ALTO]

---

### 🇪🇺 Europa (UE/EEA — GDPR)

| Requisito | Estado | Detalle |
|---|---|---|
| Cookie banner con consentimiento explícito | ✅/⚠️/❌ | [descripción] |
| GTM/GA4 condicional (no carga sin consent) | ✅/⚠️/❌ | [Consent Mode v2 detectado o no] |
| Base legal del tratamiento declarada | ✅/⚠️/❌ | [consentimiento / interés legítimo / contrato] |
| Mención a GDPR / RGPD / DSGVO | ✅/⚠️/❌ | [encontrado o no] |
| Derechos GDPR (acceso, portabilidad, supresión, oposición) | ✅/⚠️/❌ | [mencionados o no] |
| Identificación del DPO o responsable EU | ✅/⚠️/❌ | [encontrado o no] |
| Transferencias internacionales declaradas | ✅/⚠️/❌ | [mencionado o no] |
| Formularios con checkbox de consentimiento | ✅/⚠️/❌ | [presente o no] |
| Política de Cookies separada o sección dedicada | ✅/⚠️/❌ | [encontrada o no] |

**Nivel de riesgo Europa:** [BAJO / MEDIO / ALTO]

---

### 🌏 Asia

#### General Asia
| Requisito | Estado | Detalle |
|---|---|---|
| Política de privacidad disponible | ✅/⚠️/❌ | [descripción] |
| Información del proveedor/empresa local | ✅/⚠️/❌ | [encontrado o no] |

#### Japón (APPI)
| Requisito | Estado | Detalle |
|---|---|---|
| Aviso de manejo de información personal (個人情報の取り扱い) | ✅/⚠️/❌ | [encontrado o no] |
| Nombre del responsable del tratamiento | ✅/⚠️/❌ | [encontrado o no] |
| Propósito del uso declarado | ✅/⚠️/❌ | [encontrado o no] |

#### China (PIPL)
| Requisito | Estado | Detalle |
|---|---|---|
| Política de privacidad en chino simplificado | ✅/⚠️/❌ | [encontrado o no] |
| Consentimiento explícito para datos sensibles | ✅/⚠️/❌ | [encontrado o no] |
| ICP License o registro local (si aplica) | ✅/⚠️/❌ | [encontrado o no] |
| Aviso de transferencia transfronteriza | ✅/⚠️/❌ | [encontrado o no] |

#### India (DPDP Act 2023)
| Requisito | Estado | Detalle |
|---|---|---|
| Aviso de datos en inglés o idioma local | ✅/⚠️/❌ | [encontrado o no] |
| Propósito de procesamiento declarado | ✅/⚠️/❌ | [encontrado o no] |
| Mecanismo de consentimiento | ✅/⚠️/❌ | [encontrado o no] |

#### Sudeste Asiático (Singapur PDPA, Indonesia PDP, etc.)
| Requisito | Estado | Detalle |
|---|---|---|
| Política de privacidad accesible | ✅/⚠️/❌ | [encontrado o no] |
| Propósito de recopilación declarado | ✅/⚠️/❌ | [encontrado o no] |
| Canal de contacto para datos personales | ✅/⚠️/❌ | [encontrado o no] |

**Nivel de riesgo Asia:** [BAJO / MEDIO / ALTO]

---

## Políticas específicas de Google Ads

| Política | Estado | Detalle |
|---|---|---|
| Política de servicios de suscripción (si aplica) | ✅/⚠️/❌ | [precio recurrente visible, cancelación clara] |
| Política de facturación por operador (DCB, si aplica) | ✅/⚠️/❌ | [método de cobro explícito] |
| Política de servicios financieros (si aplica) | ✅/⚠️/❌ | [disclaimers de riesgo si corresponde] |
| Política de servicios de salud (si aplica) | ✅/⚠️/❌ | [N/A o descripción] |
| Política de juegos de azar (si aplica) | ✅/⚠️/❌ | [N/A o descripción] |
| Landing page coherente con el anuncio esperado | ✅/⚠️/❌ | [descripción de coherencia] |
| Sin técnicas de engaño o presión artificial | ✅/⚠️/❌ | [observaciones] |

---

## Tracking y Analytics

| Elemento | Estado | Detalle |
|---|---|---|
| GTM ID detectado | ✅/❌ | [ID o "no encontrado"] |
| GA4 detectado | ✅/❌ | [confirmado via GTM o directo] |
| Google Ads Conversion Tag | ✅/❌/⚠️ | [detectado o no confirmable desde frontend] |
| Google Consent Mode v2 | ✅/❌ | [gtag consent default detectado o no] |
| Otros trackers (Meta, Clarity, TikTok, etc.) | ℹ️ | [lista de trackers encontrados] |

---

## Mapa de links verificados

Lista todos los links encontrados en la landing y el resultado de su verificación. Incluí una fila por cada link fetcheado (excluí anclas, mailto, tel y redes sociales estándar).

| Link | Categoría | Estado | Detalle |
|---|---|---|---|
| [URL o texto del link] | Legal / CTA / Externo / Exit | ✅/⚠️/❌ | [carga OK / error / redirige a X / contenido insuficiente] |

Al final de la tabla, agregá un resumen:
- **Links verificados:** N
- **Links funcionales:** N
- **Links rotos o inaccesibles:** N (listar cuáles)
- **Dominios externos detectados:** [lista de dominios distintos al principal]

---

## Problemas críticos (bloquean la campaña)

[Lista numerada de los issues que impiden correr Google Ads ahora. Si no hay ninguno, escribir "Ninguno".]

## Observaciones (no bloquean pero conviene corregir)

[Lista de mejoras recomendadas que no son bloqueantes.]

## Plan de acción recomendado

[Lista ordenada por prioridad de lo que hay que hacer para llegar a APTO si no lo está, o para fortalecer la compliance si ya lo está.]

---

*Auditoría generada con /gaa · Revisión humana recomendada antes de lanzar campañas*

---

## Paso 6 — Guardar el reporte como Markdown

Una vez mostrado el reporte completo en pantalla, guardalo como archivo Markdown.

### 6a — Determinar el nombre de archivo

Extraé el dominio limpio de la URL auditada (ej: `ejemplo-com` de `https://ejemplo.com/...`) y usalo como nombre:
- `C:/Claude/gaa_[dominio].md`

### 6b — Escribir el archivo Markdown

Usando la herramienta `Write`, escribí exactamente el mismo contenido del reporte mostrado en pantalla (el texto completo del Paso 5, con todas las tablas, secciones y listas), sin modificar ni resumir nada.

### 6c — Confirmar al usuario

Una vez escrito el archivo, informá al usuario:

```
Markdown generado: C:\Claude\gaa_[dominio].md
```
