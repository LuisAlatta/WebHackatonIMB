# Postulaciones → Google Sheets (Excel) en 5 minutos

El formulario de `Register.astro` envía cada postulación a una hoja de cálculo de Google. Esa hoja se descarga como `.xlsx` cuando quieras (Archivo → Descargar → Microsoft Excel). No hace falta backend, ni servidores, ni base de datos.

---

## 1. Crear la hoja

1. Entra a <https://drive.google.com> con la cuenta que va a centralizar las postulaciones.
2. Crea una **Hoja de cálculo** nueva. Renómbrala a algo como `NEXIA 2026 — Postulaciones`.
3. La pestaña por defecto puede llamarse `Hoja 1`. El script de abajo crea automáticamente una pestaña `Postulaciones` la primera vez que llegue un envío, así que no toques nada más en la hoja.

## 2. Pegar el script

1. En la misma hoja: **Extensiones → Apps Script**.
2. Borra el contenido de `Code.gs` y pega esto exactamente:

```js
function doPost(e) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName('Postulaciones');
  if (!sheet) sheet = ss.insertSheet('Postulaciones');

  if (sheet.getLastRow() === 0) {
    sheet.appendRow([
      'timestamp',
      'team',
      'size',
      'university',
      'cycle',
      'leaderName',
      'leaderEmail',
      'leaderPhone',
      'track',
      'idea',
    ]);
  }

  const p = e.parameter;
  sheet.appendRow([
    new Date(),
    p.team || '',
    p.size || '',
    p.university || '',
    p.cycle || '',
    p.leaderName || '',
    p.leaderEmail || '',
    p.leaderPhone || '',
    p.track || '',
    p.idea || '',
  ]);

  return ContentService
    .createTextOutput(JSON.stringify({ ok: true }))
    .setMimeType(ContentService.MimeType.JSON);
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

1. Abre la web, baja a **Postulaciones**, llena el form con datos de prueba y envía.
2. Vuelve a la hoja → la pestaña **Postulaciones** debe tener una fila nueva con tus datos.
3. Para el "Excel": **Archivo → Descargar → Microsoft Excel (.xlsx)**.

## 6. Despliegue en Vercel

En Vercel: **Project → Settings → Environment Variables → Add**, con nombre `PUBLIC_SHEETS_ENDPOINT` y la misma URL. Marca los entornos *Production* y *Preview*. Vuelve a desplegar.

---

## FAQ

**¿Y si necesito cambiar el script?**
Cada vez que edites `doPost`, vuelve a `Implementar → Gestionar implementaciones → Editar (lápiz) → Nueva versión → Implementar`. La URL no cambia.

**¿Puedo recibir un email cada vez que alguien postula?**
Añade `MailApp.sendEmail('tu@correo.com', 'Nueva postulación NEXIA', JSON.stringify(p, null, 2))` al final de `doPost`, antes del `return`.

**¿Y los datos sensibles?**
La URL `/exec` es pública pero opaca: cualquiera con la URL puede *enviar* datos, no leerlos. Si te llega spam, añade un `if (!p.team || !p.leaderEmail) return;` o un campo honeypot.

**Modo demo (sin endpoint)**
Si `PUBLIC_SHEETS_ENDPOINT` está vacío, el form se ve y simula el envío (loading + pantalla de éxito) pero loguea los datos en la consola del navegador en lugar de enviarlos. Útil para revisar la UX antes de configurar Apps Script.
