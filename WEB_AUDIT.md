# WEB_AUDIT — liberticorporation.com

**Fecha:** 2026-06-09 · **Auditado:** sitio live + código fuente (`v2-apple/index.html`, 2038 líneas)
**Tipo:** one-page corporativo estático (HTML/CSS/JS vanilla, single file, sin build). Hosting GitHub Pages + Cloudflare proxy + dominio custom. Propósito: verificación institucional para bancos y socios.

## Resumen ejecutivo

Sitio funcionalmente sólido: 0 errores de consola, 0 requests fallidos, 0 overflow horizontal en 5 viewports (320–1440px), menú móvil operativo, HTTPS con redirects correctos. Los problemas reales están en **peso de página (~19–20 MB, dominado por hero.mp4 de 8.3 MB y Spline 3.4 MB)**, **SEO incompleto (sin Open Graph, canonical, sitemap ni schema.org — crítico para un sitio cuyo propósito es verificación institucional)**, **un link externo roto por SSL (www.clandestinobarber.com)** y **contenido invisible si JavaScript falla**. Nada compromete seguridad; no hay secrets expuestos ni mixed content.

---

## Hallazgos

### 🔴 Crítico

**C1. Sin Open Graph ni Twitter Cards**

- **Ubicación:** `v2-apple/index.html:6-14` (head solo tiene title, description, theme-color, favicons)
- **Impacto:** al compartir el link por WhatsApp/iMessage/LinkedIn/email (el caso de uso #1 de un sitio de verificación bancaria) no aparece preview: ni título, ni imagen, ni descripción. Primera impresión pobre exactamente donde más importa.
- **Fix:** agregar `og:title`, `og:description`, `og:image` (crear imagen 1200×630 con logo sobre negro), `og:url`, `og:type=website`, `twitter:card=summary_large_image`.

**C2. Contenido invisible sin JavaScript**

- **Ubicación:** `v2-apple/index.html:70-76` (`.fi { opacity: 0 }`) + `:1899-1916` (IntersectionObserver agrega `.v`)
- **Impacto:** con JS bloqueado/fallido (corporate proxies, extensiones, error de red en parse), todo el contenido queda permanentemente invisible — solo se ve el video del hero. Googlebot renderiza JS, pero otros crawlers/preview-bots no.
- **Fix:** `<noscript><style>.fi{opacity:1;transform:none}</style></noscript>`, una línea.

### 🟠 Alto

**A1. Link roto: www.clandestinobarber.com (SSL inválido)**

- **Ubicación:** `v2-apple/index.html:1223` (`href="https://www.clandestinobarber.com"`)
- **Evidencia:** `curl https://www.clandestinobarber.com` → `SSL: no alternative certificate subject name matches target host name` (cert emitido para `livekit-us.zigsa.app`). Sin `www` responde 200.
- **Impacto:** clic en la card de Clandestino Barber → error SSL a pantalla completa en el navegador del visitante. Pésimo para un sitio de verificación (parece phishing).
- **Fix:** cambiar href a `https://clandestinobarber.com` (y el texto visible ya dice `clandestinobarber.com`, así que solo el href).

**A2. Peso de carga inicial ~12 MB (hero.mp4 8.3 MB + Spline 3.4 MB) — ~20 MB con scroll completo**

- **Ubicación:** `v2-apple/index.html:1054-1061` (hero video sin `poster` ni `preload`), `:21-24` (spline-viewer 2.12 MB desde unpkg, cargado siempre), `:1485-1488` (escena `.splinecode` 1.29 MB)
- **Evidencia:** hero.mp4 = 8,672,982 bytes; spline-viewer.js = 2.12 MB; scene.splinecode = 1.29 MB; videos mosaico externos 1.5 + 2.4 + 4.1 MB (estos sí con `preload="metadata"`, bien).
- **Impacto:** en 4G/datos móviles (Venezuela) la carga completa es lenta y cara; el script de Spline se descarga aunque el usuario nunca llegue a la sección AI.
- **Fix:** (1) recomprimir hero.mp4 a 1080p CRF 28 (~2-3 MB) + agregar `poster` (frame JPEG); (2) lazy-load del script Spline solo cuando la sección `#ai` entra al viewport (dynamic import en el IntersectionObserver existente); (3) considerar `<source>` AV1/HEVC.

**A3. 8 logos con `alt=""` — empresas anónimas para lectores de pantalla**

- **Ubicación:** `v2-apple/index.html:1731, 1743, 1757, 1766, 1775, 1782, 1789, 1796` (org chart: el logo es el ÚNICO identificador de cada empresa; con `alt=""` el screen reader lee solo "100% — Marketing" sin saber de quién)
- **Impacto:** WCAG 1.1.1; en el org chart la información de qué empresa es se pierde por completo.
- **Fix:** poner nombre de empresa en cada alt (`alt="1bite Studio"`, etc.).

### 🟡 Medio

**M1. Sin canonical, sitemap.xml (404) ni datos estructurados**

- **Ubicación:** head (`:6-24`); `https://liberticorporation.com/sitemap.xml` → 404. robots.txt existe pero es el auto-generado de Cloudflare (content signals), sin directiva `Sitemap:`.
- **Impacto:** SEO subóptimo; sin schema.org `Organization` (nombre, logo, teléfono, sameAs a las redes) Google no arma knowledge panel — relevante para verificación corporativa.
- **Fix:** `<link rel="canonical" href="https://liberticorporation.com/">`, sitemap.xml de 1 URL, JSON-LD `Organization`.

**M2. Jerarquía de encabezados rota: títulos de sección son `<div>`**

- **Ubicación:** `.sec-title` en `:1112, 1137, 1250, 1467, 1575, 1612, 1689, 1826` son `<div>`; el único heading aparte del H1 es un `<h3>` (`:1478`). Documento = H1 → H3, sin H2.
- **Impacto:** SEO (estructura semántica) + navegación por headings de screen readers.
- **Fix:** convertir `.sec-title` a `<h2>` (estilos ya van por clase, cero cambio visual).

**M3. Logos PNG gigantes sin formato moderno**

- **Ubicación:** `assets/logos/` — 1pixel-logo.png 3238px/134 KB, clandestino 4500px/88 KB, apex 4500px/84 KB, 1bite 4500px/71 KB… total ~634 KB para mostrarse a ≤200px de ancho.
- **Impacto:** ~600 KB desperdiciados; viola además el pipeline propio (regla global: WebP antes de deploy).
- **Fix:** resize a 2x del tamaño render (400px) + WebP → ~30-50 KB total.

**M4. Headers de seguridad ausentes**

- **Evidencia:** respuesta live sin `Strict-Transport-Security`, `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`.
- **Impacto:** bajo en un estático sin auth, pero HSTS y X-Frame-Options (anti-embedding/clickjacking) son gratis. GitHub Pages no permite headers custom; Cloudflare sí.
- **Fix:** Cloudflare → Rules → Response Header Transform: `Strict-Transport-Security: max-age=31536000`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`. HSTS también activable en SSL/TLS → Edge Certificates.

**M5. Script de terceros sin SRI ni versión inmutable garantizada**

- **Ubicación:** `:21-24` — `https://unpkg.com/@splinetool/viewer@1.9.48/build/spline-viewer.js`
- **Impacto:** unpkg comprometido o caído = sección AI muerta; sin `integrity` no hay verificación. Único script externo del sitio.
- **Fix:** agregar atributo `integrity` + `crossorigin`, o self-hostear el viewer en `/assets/`.

**M6. Touch target del menú hamburguesa: 30×33 px**

- **Ubicación:** `:142-153` (`.nav-tog` padding 6px + svg 18px) — medido 30×33 px en live; mínimo recomendado 44×44.
- **Ubicación adicional:** `#mt` sin `aria-expanded` (`:1029`).
- **Fix:** `padding: 13px` y `aria-expanded` toggled en el listener (`:1919-1921`).

**M7. Videos hot-linkeados de sitios hermanos = dependencia frágil**

- **Ubicación:** `:1628, 1648, 1668` (thestudio4.io, 1pixelve.com, detailprocar.com)
- **Evidencia:** hoy los 3 responden 206 y reproducen; pero un redeploy de cualquier sitio hermano que renombre el archivo rompe el tile silenciosamente (sin error visible, tile negro).
- **Impacto:** decisión consciente (memoria de proyecto: "hot-linked, NOT copied"), pero sin monitoreo nadie se enteraría.
- **Fix mínimo:** poster/fallback por tile; ideal: copiar versiones comprimidas al repo.

### 🟢 Bajo

**B1. `href="#"` en logo del nav** (`:1015`) — scroll-to-top accidentado + `#` en URL. Fix: `href="#top"` o listener con `scrollTo`.
**B2. Videos autoplay ignoran `prefers-reduced-motion`** — el parallax sí lo respeta (`:1931-1934`), los 4 videos no. Fix: pausar autoplay bajo media query.
**B3. CSS/JS inline sin minificar** — documento 13.6 KB; impacto marginal. No tocar salvo que se agregue build step.
**B4. `meta description` única y title genérico** — "Liberti Global Corp" sin descriptor. Fix: `Liberti Global Corp — Diversified Holding Company`.
**B5. Favicon ICO referenciado con ruta relativa `../favicon.ico` en v2-apple** (`:12-13`) — funciona en root deployado (sed lo corrige), correcto pero frágil si el sync sed cambia.

### ✅ Verificado OK

- HTTPS válido, HTTP→HTTPS 301, www→apex 301, certificado Cloudflare vigente
- 0 errores de consola, 0 requests fallidos (24 requests totales)
- 0 overflow horizontal en 320/375/768/1024/1440
- Contraste AA: `.st-l` 5.80, `.card-emp` 4.80 — pasan
- `lang="en"` declarado, 1 solo H1, anchors internos (6/6 existen), menú móvil abre/cierra
- LCP 140 ms (primer frame hero video), FCP 124 ms — excelentes en buena conexión
- Links externos: 7/8 OK (solo falla www.clandestinobarber.com, ver A1)
- Sin secrets/API keys en el código; sin mixed content; fonts con `display=swap` + preconnect

---

## Quick wins vs. mayor esfuerzo

| Quick win (≤15 min)                        | Hallazgo |
| ------------------------------------------ | -------- |
| Fix href clandestinobarber (quitar www)    | A1       |
| `<noscript>` fallback para `.fi`           | C2       |
| Meta OG + Twitter Cards (+ crear og-image) | C1       |
| alt en 8 logos del org chart               | A3       |
| canonical + JSON-LD Organization + sitemap | M1       |
| `.sec-title` div → h2                      | M2       |
| Padding hamburguesa + aria-expanded        | M6       |
| Headers vía Cloudflare Transform Rules     | M4       |

| Mayor esfuerzo                                    | Hallazgo |
| ------------------------------------------------- | -------- |
| Recomprimir hero.mp4 + poster + lazy Spline       | A2       |
| Logos → WebP redimensionados (12 archivos + refs) | M3       |
| SRI o self-host del Spline viewer                 | M5       |
| Fallback/copia local de videos hermanos           | M7       |
