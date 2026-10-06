# Presupuestador LIWA Módulos

Widget de autopresupuesto para construcción modular, en pasos. Es un único archivo (`index.html`) sin dependencias, listo para embeber en una landing de GoHighLevel (GHL) mediante un iframe y enviar los leads a un Workflow de GHL por webhook.

**Pasos:** Superficie → Ambientes (+ A/C) → Terminación → Datos de contacto → Rango estimado en USD + CTA para agendar.

## Lógica de precios

Todos los valores están en el objeto `PRICING` dentro de `index.html` y se pueden editar ahí:

```
Costo Base = (m²_cub × 350 + m²_semi × 180 + Ambientes + A/C × 1100) × Factor_Terminación + 2500
Precio     = Costo Base × 1.50
Rango      = Precio × 0.95  →  Precio × 1.15   (redondeado a centenas)
```

Ambientes: baño $2,500 · cocina $2,000 · dormitorio $800. Terminación: Standard ×1.0 · Premium ×1.35.

Ejemplo: 45 m² cubiertos + 12 m² de semicubierto, baño + cocina, 1 equipo de A/C, Standard → **$37,100 – $44,900 USD**.

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
5. *(Recomendado)* Crear campos personalizados del contacto (*Settings → Custom Fields*) y mapearlos: `rango_texto`, `precio_estimado_usd`, `rango_min_usd`, `rango_max_usd`, `m2_cubiertos`, `m2_semicubiertos`, `ambientes`, `aires_acondicionados`, `terminacion`.
6. Agregar las acciones que quieran: **Add Tag** (`presupuestador`), crear una oportunidad en el pipeline, enviar un WhatsApp o email con el `rango_texto`, notificar al equipo, etc.
7. **Publicar** el workflow.

### 3. Pegar el iframe en la landing

En el editor de la landing de GHL, agregar un elemento **Custom JS/HTML** (Código personalizado) y pegar el contenido de [`embed-ghl.html`](embed-ghl.html), reemplazando:

| Parámetro | Valor |
|---|---|
| `src` base | La URL del paso 1 |
| `webhook=` | URL del Inbound Webhook (paso 2), **codificada** |
| `cta=` | Link del calendario de GHL (*Calendars → Share → Permanent link*), **codificado** |
| `logo=` *(opcional)* | URL de un PNG/SVG para reemplazar el logo incluido |

Para codificar una URL, en la consola del navegador (F12): `encodeURIComponent("https://services.leadconnectorhq.com/hooks/...")`.

Ejemplo final:

```
https://usuario.github.io/liwa-presupuestador/?webhook=https%3A%2F%2Fservices.leadconnectorhq.com%2Fhooks%2FABC%2Fwebhook-trigger%2FXYZ&cta=https%3A%2F%2Fapi.leadconnectorhq.com%2Fwidget%2Fbooking%2FCAL123
```

El script incluido en `embed-ghl.html` ajusta el alto del iframe en cada paso, así no aparecen barras de scroll.

> Si prefieren no usar parámetros en la URL, pueden escribir los valores directamente en el objeto `CONFIG` de `index.html` (`webhookUrl`, `ctaUrl`, `logoUrl`).

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
  "bano": true, "cocina": true, "dormitorio": false,
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

El botón "Agendar llamada" abre el calendario en la página completa (`target="_top"`) y le pasa `first_name`, `last_name`, `email` y `phone` en la URL para que el formulario de reserva de GHL ya aparezca precargado.

## Personalizar

- **Colores:** las variables CSS en `:root` al principio de `index.html` (`--brand` naranja, `--ink` negro).
- **Textos y precios:** directamente en el HTML y en `PRICING`.
- **Probar localmente:** abrir `index.html` en el navegador. Sin webhook configurado, el envío se simula y se ve en la consola.
