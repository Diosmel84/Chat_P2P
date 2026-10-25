# 🔥 Chat Efímero P2P

Un chat entre dos personas que **se conecta directo de navegador a navegador** (P2P) y donde **cada mensaje se autodestruye a los 5 segundos**. Todo está en un solo archivo, `index.html`: no usa backend, ni npm, ni pasos de build.

---

## Índice

1. [Qué hace](#qué-hace)
2. [Cómo usarlo](#cómo-usarlo)
3. [Cómo funciona por dentro](#cómo-funciona-por-dentro)
4. [Estructura del archivo](#estructura-del-archivo)
5. [Ciclo de vida de un mensaje](#ciclo-de-vida-de-un-mensaje)
6. [Privacidad y seguridad](#privacidad-y-seguridad)
7. [Publicarlo en internet](#publicarlo-en-internet)
8. [Limitaciones conocidas](#limitaciones-conocidas)
9. [Personalización rápida](#personalización-rápida)

---

## Qué hace

| Requisito | Cómo se cumple |
|---|---|
| Crear un Peer ID al cargar | `new Peer()` pide un ID aleatorio al servidor público de PeerJS |
| Mostrar "Tu link para chatear:" | Panel con la URL `...index.html?join=ID` y botón **Copiar link** |
| Conectarse automáticamente con `?join=ID` | Se lee con `URLSearchParams` y se llama a `peer.connect(id)` |
| Estado al conectar | `Conectado - mensajes se autodestruyen en 5s` (en verde) |
| Chat por DataChannel | `conexion.send(texto)` / evento `conexion.on('data')` |
| Contador 5-4-3-2-1 y borrado | `setInterval` de 1 s; al llegar a 0 se quema y se hace `remove()` del DOM |
| Sin historial | No hay arrays. Una sola variable `mensaje`, que se pone en `null` al borrar |
| 1 mensaje a la vez | Si llega otro, el anterior se quema rápido antes de mostrar el nuevo |
| Cerrar pestaña = destruir todo | `pagehide`/`beforeunload` cierran la conexión y destruyen el Peer. No usa `localStorage` |
| Diseño | Modo oscuro, mobile first, centrado, burbujas estilo iMessage, animación de quemado |

---

## Cómo usarlo

1. **Persona A** abre `index.html` (mejor desde una URL publicada; mirá la sección [Publicarlo](#publicarlo-en-internet)).
2. Aparece **"Tu link para chatear:"** con una URL del tipo:
   ```
   https://tu-sitio.com/index.html?join=3f9c2a1e-...
   ```
3. A toca **Copiar link** (en el celular se abre el menú de compartir) y se lo manda a **Persona B**.
4. B abre el link y se conecta sola. En las dos pantallas aparece:
   > 🟢 Conectado - mensajes se autodestruyen en 5s
5. Escriben. Cada mensaje cuenta 5 → 1, se prende fuego y desaparece.
6. Cuando alguien cierra la pestaña, la conversación termina y no queda nada.

### Probarlo en tu propia compu

Abrí dos pestañas: en la primera abrí `index.html`, copiá el link y pegalo en la segunda. Para que funcione bien conviene servirlo por HTTP local:

```bash
python -m http.server 8000
```

Después entrá a `http://localhost:8000`.

---

## Cómo funciona por dentro

```
   Navegador A                Servidor PeerJS              Navegador B
  ─────────────              (solo signaling)             ─────────────
  new Peer()  ───────────────►  asigna ID "abc"
  muestra ?join=abc
                                                         abre ?join=abc
                                 ◄─────────────────────  new Peer()  →  ID "xyz"
                                 ◄─────────────────────  peer.connect("abc")
  on('connection')  ◄──────────  intercambio de ofertas
                                 SDP / ICE (WebRTC)
  ════════════════════  DataChannel directo P2P  ════════════════════
          los mensajes van de navegador a navegador, sin pasar por el servidor
```

- **PeerJS** es una librería que simplifica **WebRTC**. Su servidor público (`0.peerjs.com`) se usa **solo para el "apretón de manos"** (signaling): las dos partes intercambian los datos necesarios para encontrarse.
- Una vez establecida la conexión, los mensajes viajan por un **RTCDataChannel**, cifrado de extremo a extremo con **DTLS** (obligatorio en WebRTC).
- La conexión se abre con `reliable: true`, así los mensajes llegan completos y en orden.

---

## Estructura del archivo

El `index.html` tiene tres partes, y está comentado **línea por línea en español**:

### 1. `<head>`: configuración y estilos
- `meta viewport` con `viewport-fit=cover` para celulares con notch.
- `meta referrer="no-referrer"` para que la URL con el ID no se filtre a otros sitios.
- Carga de PeerJS desde `https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js`.
- `<style>` con variables de color (`:root`), las burbujas, el contador, la barra de progreso y las animaciones `aparecer`, `vaciar`, `latido`, `quemar` y `llamas`.

### 2. `<body>`: la estructura visual
| Elemento | Función |
|---|---|
| `#estado` | Punto de color + texto del estado de la conexión |
| `#panel-link` | "Tu link para chatear:", la URL y el botón copiar |
| `#escenario` | Zona donde vive **el único mensaje** visible |
| `#vacio` | Texto de ayuda cuando no hay mensaje |
| `#formulario` | Input + botón enviar (deshabilitados hasta conectar) |

### 3. `<script>`: la lógica
| Función | Qué hace |
|---|---|
| `iniciar()` | Crea el `Peer`; si hay `?join=` se conecta y, si no, muestra el link y espera |
| `configurarConexion(c)` | Registra los eventos `open`, `data`, `close` y `error` de la conexión. Rechaza a un tercer participante |
| `mostrarMensaje(texto, deQuien)` | Crea la burbuja, el contador y la barra, y arranca la cuenta regresiva |
| `destruirMensaje(rapido)` | Frena el contador, pone `mensaje = null`, aplica la animación de quemado y saca el nodo del DOM |
| `ponerEstado(texto, tipo)` | Cambia el texto y el color del estado |
| `habilitarChat(bool)` | Activa o desactiva el input y el botón |
| `destruirTodo()` | Al cerrar la pestaña: borra el mensaje, envía la señal de despedida (`SENAL_ADIOS`), cierra la conexión y destruye el Peer |

**Detección de desconexión (doble):**
1. **Señal de despedida:** al cerrar la pestaña se envía `SENAL_ADIOS`, y el otro lado cierra en menos de 1 s.
2. **Respaldo por ICE:** si la pestaña muere de golpe (crash, sin batería, sin red), se vigila `iceconnectionstatechange` de la conexión WebRTC, que avisa en unos 5 a 10 s.

**Variables de estado (todas solo en memoria RAM):**
- `peer`: la identidad en la red PeerJS.
- `conexion`: el DataChannel con la otra persona.
- `mensaje`: `{ elemento, intervalo }` del único mensaje vivo, o `null`.
- `miLink`: el link de invitación.

---

## Ciclo de vida de un mensaje

```
 enviar/recibir
      │
      ▼
 ¿hay mensaje vivo? ──sí──► destruirMensaje(true)  (quemado rápido 0.35 s)
      │no
      ▼
 crear burbuja ─► contador "5" + barra que se vacía en 5 s
      │
      ▼  setInterval cada 1 s
 5 → 4 → 3 → 2 (naranja, late) → 1
      │
      ▼
 destruirMensaje(false)
   ├─ clearInterval
   ├─ mensaje = null            ← ya no existe referencia en JS
   ├─ clase .quemando           ← animación de fuego 0.9 s
   └─ animationend → elemento.remove()   ← sale del DOM para siempre
                     (setTimeout de respaldo por si la pestaña está en segundo plano)
```

**Animación de quemado:** la burbuja brilla y toma un tono anaranjado (`sepia` + `hue-rotate`), se oscurece como carbón, se desenfoca y sube como ceniza. Al mismo tiempo, el pseudo-elemento `::after` dibuja llamas que suben desde abajo.

---

## Privacidad y seguridad

- ✅ **Sin backend propio y sin base de datos.** No hay ningún lugar donde se guarden los mensajes.
- ✅ **Sin `localStorage`, `sessionStorage`, cookies ni IndexedDB.**
- ✅ **Sin historial:** no hay arrays; el texto vive solo dentro del nodo del DOM durante 5 s.
- ✅ **Cifrado:** WebRTC cifra el DataChannel con DTLS.
- ✅ **Anti-XSS:** el texto recibido se inserta con `textContent`, nunca con `innerHTML`.
- ✅ **Validación:** solo se aceptan strings, recortados a 500 caracteres.
- ✅ **Solo 2 personas:** si un tercero usa el link mientras hay una charla activa, se le cierra la conexión.

⚠️ **Lo que NO puede evitar:**
- Que la otra persona saque una **captura de pantalla** o filme la pantalla.
- El servidor de signaling de PeerJS ve **los IDs y las IPs** (no ve los mensajes).
- Si la conexión directa falla y se usa un servidor TURN, los datos pasan por ese servidor, pero siguen cifrados.
- Mientras dura la sesión, cualquiera que consiga tu link puede intentar conectarse antes que la persona invitada.

---

## Publicarlo en internet

Como es un único archivo estático, se puede subir a cualquier hosting gratuito con **HTTPS** (WebRTC funciona mejor en un contexto seguro):

- **GitHub Pages:** subí `index.html` a un repositorio → Settings → Pages.
- **Netlify Drop:** arrastrá la carpeta a <https://app.netlify.com/drop>.
- **Vercel / Cloudflare Pages:** importás la carpeta y listo.

> Si abrís el archivo con doble clic (`file:///...`), el link generado apunta a tu disco y **solo sirve en tu misma compu**. Para chatear con otra persona tiene que estar publicado.

---

## Limitaciones conocidas

- **Redes muy restrictivas** (algunas redes corporativas o 4G con CGNAT estricto) pueden bloquear la conexión directa, porque PeerJS por defecto solo trae servidores STUN públicos y no TURN. Se puede pasar un servidor TURN propio en `new Peer({ config: { iceServers: [...] } })`.
- **Depende del servidor público de PeerJS** para el signaling. Si está caído, no se puede conectar. Para producción se puede usar un servidor propio (`peerjs-server`).
- **Si alguien recarga la página**, cambia su ID y se corta la conversación (es a propósito: es efímero).
- Si la pestaña queda en **segundo plano**, el navegador puede retrasar los temporizadores; igual el respaldo con `setTimeout` asegura que el mensaje se borre.

---

## Personalización rápida

| Qué cambiar | Dónde |
|---|---|
| Duración de los mensajes | `const SEGUNDOS_DE_VIDA = 5;` **y** `animation: vaciar 5s` en `.barra` |
| Largo máximo | `const LARGO_MAXIMO = 500;` y `maxlength="500"` del input |
| Colores | Variables en `:root` (`--azul`, `--gris`, `--naranja`, etc.) |
| Velocidad del quemado | `animation: quemar 0.9s` en `.burbuja.quemando` |
| Ancho en escritorio | `max-width: 480px` en `.app` |

---

**Tecnologías:** HTML5 · CSS3 · JavaScript vanilla · WebRTC · [PeerJS 1.5.2](https://peerjs.com/)
