# Presupuestador LIWA Módulos

Widget de autopresupuesto para construcción modular, en pasos. Es un único archivo (`index.html`) sin dependencias, listo para embeber en una landing de GoHighLevel (GHL) mediante un iframe y enviar los leads a un Workflow de GHL por webhook.

**Pasos:** Superficie → Ambientes (baño, cocina, dormitorios, living, A/C) → Terminación + envío (localidad y km) → Datos de contacto → Rango estimado en USD + aviso de que no es un presupuesto oficial + CTA para agendar.

**URL publicada:** https://florenciaar.github.io/liwa-presupuestador/ (GitHub Pages, rama `gh-pages`).

## Lógica de precios

Todos los valores están en el objeto `PRICING` dentro de `index.html` y se pueden editar ahí:

```
Costo Base = (m²_cub × 350 + m²_semi × 180 + Ambientes + A/C × 1100) × Factor_Terminación + 2500
Precio     = Costo Base × 1.50
Rango      = Precio × 0.95  →  Precio × 1.15   (redondeado a centenas)
```

Ambientes:
- Baño: completo $2,500 · tipo oficina $1,250
- Cocina: completa $2,000 · kitchenette $1,000
- Dormitorio (incluye placard): $800 por dormitorio
- Living (incluye cortinas y lámpara de techo): suma como mínimo 10 m² cubiertos (10 × $350 = $3,500)

Terminación: Standard ×1.0 · Premium ×1.35.

**Envío** (en ARS, se muestra aparte del rango en USD, sin margen): ida y vuelta, el mayor entre **ARS 1.300.000** y **ARS 7.000 × km**. El cliente carga los km de ida y se multiplican por 2. Ej.: 120 km → 240 km × 7.000 = ARS 1.680.000; 50 km → 100 km × 7.000 = ARS 700.000, por lo que se aplica el mínimo de ARS 1.300.000.

Ejemplo: 45 m² cubiertos + 12 m² de semicubierto, baño completo + cocina completa, 1 equipo de A/C, Standard → **$37,100 – $44,900 USD**.

---

## Integración con GoHighLevel

### 1. Publicar el widget (para tener una URL)

Un iframe necesita una URL pública. La opción gratuita más simple es **GitHub Pages**:

1. En el repo: *Settings → Pages → Build and deployment → Source: Deploy from a branch*.
2. Elegir la rama (por ejemplo `main`) y la carpeta `/ (root)`, y guardar.
3. Al minuto queda en `https://<usuario>.github.io/liwa-presupuestador/`.

(También sirve Netlify o Vercel, o subir `index.html` a cualquier hosting.)

### 2. Crear el webhook en GHL

1. **Automation → Workflows → Create Workflow → Start from scratch**.
2. Trigger: **Inbound Webhook**. Copiar la URL que genera (`https://services.leadconnectorhq.com/hooks/.../webhook-trigger/...`).
3. Completar una vez el presupuestador con el webhook ya configurado (paso 3) para que GHL reciba una muestra, y en el trigger hacer clic en **Fetch sample request** / elegir la muestra recibida.
4. Agregar la acción **Create/Update Contact** y mapear:
   - First Name → `first_name`
   - Last Name → `last_name`
   - Email → `email`
   - Phone → `phone`
   - Source → `source`
5. *(Recomendado)* Crear campos personalizados del contacto (*Settings → Custom Fields*) y mapearlos: `rango_texto`, `precio_estimado_usd`, `rango_min_usd`, `rango_max_usd`, `m2_cubiertos`, `m2_semicubiertos`, `ambientes`, `bano`, `cocina`, `dormitorios`, `living`, `aires_acondicionados`, `terminacion`, `localidad`, `distancia_km`, `envio_ars`, `envio_texto` y `presupuesto_detallado` (este último como campo de tipo **texto largo / multi line**: trae los datos de la persona y el presupuesto completo, línea por línea, listo para pegar en una nota, un email o un mensaje).
6. Agregar las acciones que quieran: **Add Tag** (`presupuestador`), crear una oportunidad en el pipeline, enviar un WhatsApp o email con el `rango_texto`, notificar al equipo, etc.
7. **Publicar** el workflow.

### 3. Pegar el iframe en la landing

En el editor de la landing de GHL, agregar un elemento **Custom JS/HTML** (Código personalizado) y pegar el contenido de [`embed-ghl.html`](embed-ghl.html), reemplazando:

| Parámetro | Valor |
|---|---|
| `src` base | La URL del paso 1 |
| `webhook=` *(opcional)* | URL del Inbound Webhook, **codificada**. El webhook de LIWA ya viene configurado por defecto en `CONFIG.webhookUrl`; este parámetro solo hace falta para usar otro |
| `wa=` | Número de WhatsApp del bot, con código de país y sin espacios ni `+` (ej: `5491112345678`). También se puede fijar en `CONFIG.whatsappNumber` |

Para codificar una URL, en la consola del navegador (F12): `encodeURIComponent("https://services.leadconnectorhq.com/hooks/...")`.

Ejemplo final:

```
https://florenciaar.github.io/liwa-presupuestador/?wa=5491112345678
```

El script incluido en `embed-ghl.html` ajusta el alto del iframe en cada paso, así no aparecen barras de scroll.

> Si prefieren no usar parámetros en la URL, pueden escribir los valores directamente en el objeto `CONFIG` de `index.html` (`webhookUrl`, `whatsappNumber`).

### Qué datos se envían

Cuando el usuario envía el paso 4, el widget:

1. Hace `console.log` del lead.
2. Hace un `POST` JSON a la URL del webhook. Si el navegador bloquea la petición por CORS, la reintenta en modo `no-cors` (GHL la recibe igual). Al usuario nunca se lo hace esperar más de 6 segundos.
3. Emite un `postMessage` `{ type: "liwa:lead", payload }` a la página padre (sirve, por ejemplo, para disparar el píxel de Meta `fbq('track','Lead')` desde la landing).

Payload de ejemplo:

```json
{
  "first_name": "Juan",
  "last_name": "Pérez",
  "full_name": "Juan Pérez",
  "name": "Juan Pérez",
  "email": "juan@test.com",
  "phone": "+54 9 11 1234 5678",
  "source": "Autopresupuestador LIWA",
  "tags": ["presupuestador", "terminacion-standard"],
  "m2_cubiertos": 45,
  "m2_semicubiertos": 12,
  "ambientes": "Baño completo, Cocina completa",
  "bano": "Baño completo",
  "cocina": "Cocina completa",
  "dormitorios": 0,
  "living": "No",
  "localidad": "Pilar, Buenos Aires",
  "distancia_km": 120,
  "envio_ars": 1680000,
  "envio_texto": "ARS 1.680.000 (ida y vuelta, 240 km)",
  "presupuesto_detallado": "PRESUPUESTO ESTIMADO - LIWA MÓDULOS\nFecha: ...\n\nDATOS DEL CLIENTE\nNombre: ...\n...\n\nDETALLE EN USD (incluye terminación Standard)\n- Superficie cubierta (45 m²): $23,625 USD\n...",
  "aires_acondicionados": 1,
  "terminacion": "Standard",
  "costo_base_usd": 26010,
  "precio_estimado_usd": 39015,
  "rango_min_usd": 37100,
  "rango_max_usd": 44900,
  "rango_texto": "$37,100 USD - $44,900 USD",
  "pagina": "https://...",
  "fecha": "2026-10-06T12:00:00.000Z"
}
```

El botón "Agendar llamada" (y el link "escribinos por WhatsApp" del disclaimer) abre un chat de WhatsApp con el bot, con un mensaje precargado que incluye nombre, rango en USD, envío, m², terminación y localidad.

## Personalizar

- **Colores:** las variables CSS en `:root` al principio de `index.html` (tema oscuro: `--brand` naranja, `--bg`, `--card`, `--field`, `--border`). Tipografías: Jost (textos) y Space Grotesk (títulos y precios).
- **Textos y precios:** directamente en el HTML y en `PRICING`.
- **Probar localmente:** abrir `index.html` en el navegador. Sin webhook configurado, el envío se simula y se ve en la consola.
