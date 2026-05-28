# Postulaciones → Google Sheets (Excel) en 5 minutos

El formulario de `Register.astro` envía cada postulación a una hoja de cálculo de Google con dos pestañas:

- **`Equipos`** — una fila por postulación (datos del equipo).
- **`Personas`** — una fila por persona (líder + cada integrante), enlazadas al equipo.

La hoja se descarga como `.xlsx` cuando quieras (Archivo → Descargar → Microsoft Excel). No hace falta backend, ni servidores, ni base de datos.

---

## 1. Crear la hoja

1. Entra a <https://drive.google.com> con la cuenta que va a centralizar las postulaciones.
2. Crea una **Hoja de cálculo** nueva. Renómbrala a algo como `NEXIA 2026 — Postulaciones`.
3. La pestaña por defecto puede llamarse `Hoja 1`. El script crea automáticamente `Equipos` y `Personas` la primera vez que llegue un envío.

## 2. Pegar el script

1. En la misma hoja: **Extensiones → Apps Script**.
2. Borra el contenido de `Code.gs` y pega esto exactamente:

> Importante: Apps Script tiene dos motores (V8 moderno y Rhino legacy). Si tu proyecto está en Rhino, `const`/`let`/arrow no funcionan y verás `TypeError: "" is not a function`. El script de abajo está escrito en **ES5 puro** para que corra en ambos sin tocar nada.

```js
var EQUIPOS_HEADERS = [
  'Fecha y hora', 'ID equipo', 'Equipo', 'N° integrantes', 'Track elegido', 'Idea / tema del proyecto'
];

var PERSONAS_HEADERS = [
  'Fecha y hora', 'ID equipo', 'Equipo', 'Rol', 'N° de orden',
  'Nombre completo', 'Carrera', 'Universidad', 'Correo electrónico',
  'Celular', 'Perfil técnico', 'Perfil no técnico'
];

function doPost(e) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var equipos  = ensureSheet_(ss, 'Equipos',  EQUIPOS_HEADERS);
  var personas = ensureSheet_(ss, 'Personas', PERSONAS_HEADERS);

  var p = (e && e.parameter) ? e.parameter : {};
  var now = new Date();
  var teamId = Utilities.getUuid().slice(0, 8);

  equipos.appendRow([
    now, teamId, p.team || '', toInt_(p.size), p.track || '', p.project_idea || ''
  ]);

  personas.appendRow([
    now, teamId, p.team || '', 'Líder', 0,
    p.leader_name || '', p.leader_career || '',
    p.leader_university || '', p.leader_email || '',
    "'" + (p.leader_phone || ''),
    p.leader_tech || '', p.leader_nontech || ''
  ]);

  var idxs = collectMemberIdxs_(p);
  for (var i = 0; i < idxs.length; i++) {
    var idx = idxs[i];
    personas.appendRow([
      now, teamId, p.team || '', 'Integrante', idx,
      p['member_' + idx + '_name']       || '',
      p['member_' + idx + '_career']     || '',
      p['member_' + idx + '_university'] || '',
      p['member_' + idx + '_email']      || '',
      '',
      p['member_' + idx + '_tech']       || '',
      p['member_' + idx + '_nontech']    || ''
    ]);
  }

  return ContentService
    .createTextOutput(JSON.stringify({ ok: true, team_id: teamId }))
    .setMimeType(ContentService.MimeType.JSON);
}

function ensureSheet_(ss, name, headers) {
  var sheet = ss.getSheetByName(name);
  if (!sheet) sheet = ss.insertSheet(name);

  var range = sheet.getRange(1, 1, 1, headers.length);
  var current = range.getValues()[0];
  var matches = true;
  for (var i = 0; i < headers.length; i++) {
    if (current[i] !== headers[i]) { matches = false; break; }
  }
  if (!matches) {
    range.setValues([headers]);
    range.setFontWeight('bold');
    range.setBackground('#1a1a2e');
    range.setFontColor('#00e5ff');
    sheet.setFrozenRows(1);
    sheet.autoResizeColumns(1, headers.length);
  }
  return sheet;
}

function collectMemberIdxs_(p) {
  var seen = {};
  var keys = Object.keys(p);
  for (var i = 0; i < keys.length; i++) {
    var m = keys[i].match(/^member_(\d+)_/);
    if (m) seen[m[1]] = true;
  }
  var arr = [];
  for (var k in seen) arr.push(parseInt(k, 10));
  arr.sort(function (a, b) { return a - b; });
  return arr;
}

function toInt_(v) {
  var n = parseInt(v, 10);
  return isNaN(n) ? '' : n;
}
```

3. Guarda con **Ctrl + S** (ponle nombre al proyecto, p. ej. `nexia-postulaciones`).

## 3. Publicar como Web App

1. Pulsa **Implementar → Nueva implementación** (Deploy → New deployment).
2. En el icono de la rueda elige **Aplicación web** (Web app).
3. Configura:
   - **Descripción**: `nexia v1`
   - **Ejecutar como**: tu cuenta (Me)
   - **Quién tiene acceso**: **Cualquier persona** (Anyone)
4. Pulsa **Implementar**. Google pedirá permisos la primera vez — acéptalos.
5. Copia la **URL de la aplicación web**. Tiene esta forma:
   ```
   https://script.google.com/macros/s/AKfycbx.../exec
   ```

## 4. Pegarla en el proyecto

En la raíz del repo:

1. Copia `.env.example` a `.env` (sólo se hace una vez).
2. Pega la URL en la variable:
   ```
   PUBLIC_SHEETS_ENDPOINT=https://script.google.com/macros/s/AKfycbx.../exec
   ```
3. Reinicia el servidor de dev:
   ```
   npm run dev
   ```

## 5. Probar

1. Abre la web, baja a **Postular**, llena el form con datos de prueba y envía.
2. Vuelve a la hoja:
   - Pestaña **Equipos** debe tener una fila nueva con `team_id`, `team`, `size`, `track`.
   - Pestaña **Personas** debe tener `size` filas nuevas (líder + integrantes), todas con el mismo `team_id` y el mismo `team`.
3. Para Excel: **Archivo → Descargar → Microsoft Excel (.xlsx)**.

## 6. Despliegue en Vercel / Netlify

En **Project → Settings → Environment Variables → Add**, con nombre `PUBLIC_SHEETS_ENDPOINT` y la misma URL. Marca *Production* y *Preview*. Vuelve a desplegar.

---

## Estructura de datos enviada

El form actual manda estos campos (URL-encoded en `FormData`):

| Campo                  | Origen                       |
| ---------------------- | ---------------------------- |
| `team`                 | Nombre del equipo            |
| `size`                 | "3", "4" o "5"               |
| `track`                | Track elegido (texto libre)  |
| `project_idea`         | Idea / tema del proyecto (opcional) |
| `leader_name`          | Líder · nombre completo      |
| `leader_career`        | Líder · carrera              |
| `leader_university`    | Líder · universidad          |
| `leader_email`         | Líder · email                |
| `leader_phone`         | Líder · celular              |
| `leader_tech`          | Líder · perfil técnico       |
| `leader_nontech`       | Líder · perfil no técnico    |
| `member_N_name`        | Integrante N · nombre        |
| `member_N_career`      | Integrante N · carrera       |
| `member_N_university`  | Integrante N · universidad   |
| `member_N_email`       | Integrante N · email         |
| `member_N_tech`        | Integrante N · técnico       |
| `member_N_nontech`     | Integrante N · no técnico    |
| `terms`                | "on" si aceptó bases         |

`N` va de `1` a `size - 1` (el líder cuenta como uno; los integrantes son los demás).

---

## FAQ

**¿Y si necesito cambiar el script?**
Cada vez que edites `doPost`, vuelve a `Implementar → Gestionar implementaciones → Editar (lápiz) → Nueva versión → Implementar`. La URL no cambia.

**¿Puedo recibir un email cada vez que alguien postula?**
Añade al final de `doPost`, antes del `return`:
```js
MailApp.sendEmail('rrpp.marcaperuana@gmail.com',
  'Nueva postulación NEXIA: ' + (p.team || 'sin nombre'),
  JSON.stringify(p, null, 2));
```

**¿Y los datos sensibles?**
La URL `/exec` es pública pero opaca: cualquiera con la URL puede *enviar* datos, no leerlos. Si te llega spam, añade un `if (!p.team || !p.leader_email) return;` o un campo honeypot.

**Modo demo (sin endpoint)**
Si `PUBLIC_SHEETS_ENDPOINT` está vacío, el form se ve y simula el envío (loading + pantalla de éxito) sin enviar nada. Útil para revisar la UX antes de configurar Apps Script.
