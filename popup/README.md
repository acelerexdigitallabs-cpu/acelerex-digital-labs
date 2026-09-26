# Popup "Más juegos" — contrato de integración (WebView)

Página estática servida en vivo desde GitHub Pages
(`https://acelerexdigitallabs-cpu.github.io/acelerex-digital-labs/popup/index.html`)
que los juegos cargan dentro de un `WebView` en lugar de traer su propia copia del modal.

**Importante:** esta página no está versionada por juego. Todos los juegos —viejos y
nuevos— apuntan a la misma URL en producción. Cualquier cambio al contrato tiene que ser
aditivo y retrocompatible: un juego que no manda un parámetro nuevo tiene que seguir
viéndose y comportándose exactamente igual que hoy.

Esto es distinto del paquete `@acelerexdigitallabs/more-games` (ver skill `more-games`),
que es un `Modal` nativo de React Native sin WebView. Son dos caminos de integración
separados — no mezclar.

## Cómo se embebe

```tsx
<WebView
  source={{
    uri:
      'https://acelerexdigitallabs-cpu.github.io/acelerex-digital-labs/popup/index.html' +
      `?currentGameId=${applicationId}&locale=${locale}`,
  }}
  onMessage={(event) => {
    const msg = JSON.parse(event.nativeEvent.data);
    if (msg.type === 'CLOSE') setVisible(false);
    if (msg.type === 'OPEN_URL') Linking.openURL(msg.url);
  }}
/>
```

`currentGameId` = el `applicationId` real de `android/app/build.gradle` de ESE proyecto
(no asumirlo del nombre del repo ni de `app.json`).

## Contrato implementado hoy

**Query params (al cargar la URL):**

| Param            | Valores          | Default si falta                          |
| ---------------- | ----------------- | ------------------------------------------ |
| `locale`         | `es` \| `en`       | detecta `navigator.language` del WebView    |
| `currentGameId`  | id del juego       | ninguno (no se excluye ningún juego de la lista) |

**postMessage saliente (popup → app, vía `onMessage`):**

| type       | payload           | Cuándo se dispara                          |
| ---------- | ----------------- | ------------------------------------------- |
| `CLOSE`    | —                  | el usuario toca el botón de cerrar (×)      |
| `OPEN_URL` | `{ url: string }`  | el usuario toca "Jugar", el footer o el banner beta |

## Tuto rápido

Inicial: `?theme=dark|light&accentColor=RRGGBB` en la URL del WebView.
Runtime: `webviewRef.postMessage(JSON.stringify({type:'THEME_UPDATE', theme, accentColor}))`.

## Extensión planeada: tema y color de acento (aún no implementado en `index.html`)

Diseño acordado para cuando se codifique. Documentado ahora para que los juegos que
integren desde hoy sepan qué esperar y puedan sumarlo sin esperar a un aviso aparte.

**Query params nuevos (estado inicial, antes de que el WebView termine de cargar):**

| Param         | Valores                        | Default si falta                                  |
| ------------- | ------------------------------- | --------------------------------------------------- |
| `theme`       | `dark` \| `light`                | detecta `prefers-color-scheme` del WebView; si tampoco hay, `dark` |
| `accentColor` | hex de 6 dígitos (`RRGGBB` o `#RRGGBB`) | paleta fija actual (cyan) |

**postMessage entrante nuevo (app → popup, para cambios en caliente sin recargar):**

```json
{ "type": "THEME_UPDATE", "theme": "light", "accentColor": "#FF6B00" }
```

- Ambos campos son opcionales dentro del mensaje — mandá solo el que cambió.
- `accentColor` inválido (no matchea `/^#?[0-9A-Fa-f]{6}$/`) se ignora sin romper el render.
- El acento solo tiñe **bordes** (header, rows, footer) — el botón "Jugar" y el badge
  "Beta" mantienen sus colores fijos (cyan/amber).
- Se manda con `webviewRef.current.postMessage(JSON.stringify(msg))` del lado RN; el
  popup escucha `window.addEventListener('message', ...)` (y `document`, por la
  inconsistencia de WebView en Android).

**Compatibilidad:** un juego legacy que no manda `theme`/`accentColor` ni el
`postMessage` de `THEME_UPDATE` no tiene que cambiar nada — el popup se sigue viendo
igual que en producción hoy (dark + bordes cyan).

## Probar localmente

Abrir `popup/index.html` directo en el navegador con query params, ej.:
`popup/index.html?locale=es&currentGameId=com.acelerex.sudokugame`. Sin
`ReactNativeWebView`, los botones caen al fallback `window.open`/no-op definido en el
script.
