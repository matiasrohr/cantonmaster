# CantonMaster — Contexto completo para Claude Code

## ¿Qué es esto?
CantonMaster es una PWA (Progressive Web App) para iPhone diseñada para capturar información de proveedores durante la Feria de Cantón (Guangzhou, China). Una sola persona opera todo en la feria y necesita capturar datos rápidamente.

## Flujo de la app
1. **Tarjeta** — Foto de la tarjeta del proveedor → Claude OCR extrae nombre, empresa, cargo, email, teléfono, WeChat
2. **Fotos** — Hasta 5 fotos del producto + 2 del stand
3. **Audio conversación** — Grabación de la conversación con el proveedor → Whisper transcribe
4. **Audio notas privadas** — Notas personales del operador (no van al informe del cliente)
5. **Informe** — Claude genera informe estructurado con todos los datos → descarga PDF + ZIP + sube a Google Drive

## Stack técnico
- HTML/CSS/JS puro — un solo archivo `index.html`
- Hosted en GitHub Pages: `https://matiasrohr.github.io/cantonmaster`
- API de Anthropic (Claude) via proxy en Cloudflare Workers
- API de OpenAI (Whisper) para transcripción de audio
- Google Drive API para guardar informes
- JSZip para generar archivos ZIP

## Infraestructura configurada
- **GitHub repo:** `github.com/matiasrohr/cantonmaster`
- **Cloudflare Worker (proxy):** `https://cantonmaster-proxy.matirohr.workers.dev`
  - Resuelve CORS de Safari iOS llamando a Anthropic desde el worker
  - Código del worker: recibe POST, reenvía a `api.anthropic.com/v1/messages`, devuelve respuesta
- **Google OAuth Client ID:** `61672469231-ec2f4ehmkclce04jf480brduofash7a4.apps.googleusercontent.com`
  - Proyecto: "My First Project" en Google Cloud Console
  - Usuario de prueba autorizado: `matias@logian.dev`
  - Scopes: `https://www.googleapis.com/auth/drive.file`

## Estado actual
- ✅ App funciona en Safari iOS (modo privado y normal tras limpiar caché)
- ✅ OCR de tarjeta funciona
- ✅ Generación de informe funciona
- ✅ PDF se abre en nueva pestaña (Safari iOS no descarga PDFs directamente)
- ✅ ZIP descarga correctamente
- ✅ Navegación libre con barra inferior (5 tabs)
- ❌ Google Drive upload — NO funciona aún (en desarrollo)
- ❌ PDF no se descarga automáticamente en iOS Safari (limitación del browser)

## Problema principal a resolver
**Google Drive upload no funciona.** El flujo OAuth implementado usa `window.open()` para el login pero en iOS Safari PWA los popups están bloqueados. Necesita una solución alternativa.

### Solución recomendada para Google Drive en iOS Safari PWA
En lugar de `window.open()` para OAuth, usar **redirect flow**:
1. Guardar estado en localStorage antes de redirigir
2. Redirigir a Google OAuth con `window.location.href`
3. Google redirige de vuelta a la app con el token en el hash de la URL
4. La app detecta el token al cargar y continúa

Implementar en `googleLogin()`:
```javascript
function googleLogin() {
  // Check if we already have a token
  if (state.gToken) return Promise.resolve(state.gToken);
  
  // Check if we just got redirected back from Google with a token
  const hash = window.location.hash;
  if (hash.includes('access_token')) {
    const token = hash.match(/access_token=([^&]+)/)?.[1];
    if (token) {
      state.gToken = token;
      localStorage.setItem('cc_gtoken', token);
      window.location.hash = ''; // clean URL
      return Promise.resolve(token);
    }
  }
  
  // Save current state to resume after redirect
  localStorage.setItem('cc_pending_upload', 'true');
  
  // Redirect to Google OAuth
  window.location.href = 'https://accounts.google.com/o/oauth2/v2/auth?' +
    'client_id=61672469231-ec2f4ehmkclce04jf480brduofash7a4.apps.googleusercontent.com' +
    '&redirect_uri=' + encodeURIComponent('https://matiasrohr.github.io/cantonmaster') +
    '&response_type=token' +
    '&scope=' + encodeURIComponent('https://www.googleapis.com/auth/drive.file') +
    '&prompt=consent';
    
  return new Promise(() => {}); // never resolves, page redirects
}
```

También en el `window.addEventListener('load')` hay que detectar el retorno de Google:
```javascript
// Check if returning from Google OAuth
const hash = window.location.hash;
if (hash.includes('access_token')) {
  const token = hash.match(/access_token=([^&]+)/)?.[1];
  if (token) {
    state.gToken = token;
    localStorage.setItem('cc_gtoken', token);
    window.location.hash = '';
    if (localStorage.getItem('cc_pending_upload') === 'true') {
      localStorage.removeItem('cc_pending_upload');
      // Show message that they need to tap Drive button again
      showToast('Google conectado. Tocá "Guardar en Drive" de nuevo.');
    }
  }
}
```

## Convenciones del proyecto
- ID de requerimiento: `RFQ-YYYY-NNN` (ej: RFQ-2026-031)
- Nombres de archivo generados: `rfq-2026-031-empresa-2026-04-17.txt`
- Colores: azul oscuro `0D2137`, azul medio `1A5C96`, naranja `E67E22`, verde `1E8449`
- Idioma: español para UI, inglés para datos que van a contactos en China
- Documentos internos: español

## Campos del informe generado
```json
{
  "contacto": {"nombre","empresa","cargo","email","telefono","wechat","web"},
  "producto": {"descripcion","categoria","especificaciones","certificaciones","variantes"},
  "comercial": {"precio_fob","moq","incoterm","lead_time","muestra_disponible","costo_muestra","terminos_pago"},
  "evaluacion": {"score (1-5)","fortalezas","debilidades","fit_clientes","proximos_pasos"},
  "privado": {"impresion_personal","nivel_confianza","observaciones_internas"},
  "resumen_ejecutivo": ""
}
```

## Modelo de Claude usado
`claude-opus-4-5-20251101` — via proxy Cloudflare

## Cómo testear localmente
```bash
cd Desktop/cantonmaster
python3 -m http.server 8080
```
Abrir en Chrome: `http://localhost:8080`

**Importante:** Las llamadas a la API de Anthropic solo funcionan desde `https://` (GitHub Pages) o via el proxy de Cloudflare. Desde `localhost` va a dar error CORS para las llamadas a Claude. Para testear el flujo de Google Drive sí funciona desde localhost.

## Próximos pasos priorizados
1. **Arreglar Google Drive upload** — implementar redirect OAuth flow (ver solución arriba)
2. **Mejorar PDF en iOS** — considerar generar PDF con jsPDF library en vez de window.print()
3. **Historial de proveedores** — guardar lista de informes generados en localStorage
4. **Modo offline** — Service Worker para que funcione sin internet (fotos y audio se guardan, sync después)

## Archivos en el proyecto
```
cantonmaster/
  index.html          ← toda la app en un solo archivo
  CLAUDE.md           ← este archivo de contexto
```

## Comando para deployar a GitHub
```bash
# Después de modificar index.html:
git add index.html
git commit -m "descripción del cambio"
git push origin main
# GitHub Pages se actualiza en ~1 minuto
```
