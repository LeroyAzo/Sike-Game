# Sike.io — Documentación completa

Juego de supervivencia 2D estilo arena. Controlas un cuadrado cian que recolecta gemas doradas mientras huye de enemigos rojos. **Cada gema reasigna aleatoriamente las teclas de movimiento**, volviendo el control más caótico con cada colección.

> ⚠️ **Solo para PC / teclado físico.** No funciona en móvil.

---

## Stack

- Un solo archivo: `index.html` (HTML + CSS + JS vanilla, ~1700 líneas)
- Canvas 2D (960×720, escalado responsive vía `95vmin`)
- Web Audio API + etiquetas `<audio>` para sonidos
- `localStorage` para persistencia del récord y volumen
- Git para control de versiones

## Estructura

```
juegoweb/
├── index.html              # Todo el juego
├── SIKE.IO.md              # Esta documentación
├── test_audio2.html        # Test de decodificación de audio
├── assets/
│   └── audio/
│       ├── gem1-5.wav      # Intro gema (5 variantes)
│       ├── gem.mp3         # Segundo sonido gema
│       ├── hover.wav       # Hover botones
│       ├── click.wav       # Click botones
│       ├── vibrate.wav     # Vibración reset score
│       ├── desolve.wav     # Desvanecimiento reset score
│       ├── death.wav       # Sonido de muerte
│       ├── dumbstep_fondo_3.wav  # Música de fondo (loop)
│       └── (otros archivos de música experimental)
└── .gitignore
```

---

## Mecánica de control por tiers

| Tier | Score | Pool de teclas | Nombre |
|---|---|---|---|
| 0 | 0 | WASD (fijo) | WASD |
| 1 | 1 | Flechas (fijo) | ARROWS |
| 2 | ≤3 | RTFGVB | NEAR |
| 3 | ≤5 | RTYFGHVBN | MID |
| 4 | ≤8 | EWRTYFGHJCVBN | WIDE |
| 5 | ≤12 | QWERTASDFGZXCVB | VERY WIDE |
| 6 | >12 | Todas las letras | ALL |

Al recoger una gema:
1. `state.keys` se vacía (el jugador debe soltar y presionar de nuevo)
2. Se genera nuevo mapping aleatorio del pool del tier actual
3. Popup "SIKE" muestra las teclas por 2s (ralentiza el juego 75%)
4. El popup en modo panel se muestra en la esquina izquierda con animación glitch

---

## Flujo del juego

### Menú principal (estilo gris/rojo/negro)
- Fondo `#1c1c1c` sólido — no se ve el juego atrás
- Título "Sike.io" en rojo `#ff2222` con glow pulsante
- **PLAY** (rojo, inicia partida)
- **OPTIONS** → SOUND (sliders MASTER / GEM INTRO / SIKE VOLUME / MUSIC) + RESET SCORE (animación)
- Botones con relleno `#A8A8A8`, borde 3px, hover rojizo
- BEST SCORE en dorado `#ffd700`
- Los sliders tienen thumb rojo

### HUD
TIME | GEMS | RECORD — esquina superior del game container.

### Sistema de peligro
- Cuando un enemigo se acerca, aparece un **vignette rojo** que pulsa al ritmo del beat
- **Screen shake** sincronizado con el latido cuando el peligro es alto (>15%)
- La música se distorsiona (WaveShaper) cuanto más cerca están los enemigos

### Rastro de enemigos (goo)
- Los enemigos dejan un rasto **negro con brillo rojo** al moverse
- El rastro se desvanece gradualmente
- Si el jugador pisa el rastro, se ralentiza al 35%

### Muerte
1. Colisión con enemigo → `showDeathScreen()`
2. Screen shake fuerte (0.4s, intensidad 35px)
3. Explosión: 50 partículas (paleta cian/teal/blanco) con gravedad y rotación
4. Canvas pasa a BN progresivo 0% → 100% en 1.5s
5. Suena `death.wav`, se detiene la música
6. Aparece pantalla DEFEATED con estadísticas
7. Opciones: PLAY AGAIN (reinicia con música) / BACK TO MENU

---

## Audio

### Sistema de música de fondo
- Usa `<audio>` + `createMediaElementSource` + Web Audio API
- Flujo: `song (Audio)` → `MediaElementSource` → `AnalyserNode` → `WaveShaperNode` → `GainNode` → `destination`
- El `AnalyserNode` (FFT 256) alimenta el visualizer
- El `WaveShaperNode` distorsiona la música según la cercanía de enemigos
- El `GainNode` controla el volumen (master × music)
- Requiere servidor HTTP (Live Server, etc.) — no funciona con `file://` en Chrome/Brave por CORS

### Sonidos (archivos)
| Archivo | Propósito |
|---|---|
| `dumbstep_fondo_3.wav` | Música de fondo en loop |
| `gem1-5.wav` | 5 variantes de intro al recoger gema |
| `gem.mp3` | Segundo sonido que suena tras el intro |
| `hover.wav` | Hover sobre botones |
| `click.wav` | Click en botones |
| `vibrate.wav` | Vibración durante animación reset score |
| `desolve.wav` | Desvanecimiento al resetear score |
| `death.wav` | Sonido al morir |

### Sistema de volumen
- 4 sliders (persisten en localStorage):
  - MASTER: multiplicador global
  - GEM INTRO: volumen de los sonidos intro de gema
  - SIKE VOLUME: volumen del gem.mp3
  - MUSIC: volumen de la música de fondo
- Botones UI usan MASTER × 0.25 (hover) y MASTER × 0.45 (click)
- Volúmenes se guardan en localStorage y se restauran al cargar

---

## Visualizer (barras de frecuencia)

- Se dibuja en `#vizCanvas` (se extiende 100px más allá del game container en cada lado)
- 128 barras distribuidas en los 4 bordes (top 40, right 24, bottom 40, left 24)
- **Cada lado muestrea todo el espectro** (no bloques contiguos) para que todos reaccionen parejo
- Colores: gradiente verde-azul → azul con brillo variable según amplitud
- Sensibilidad: `freqArray[i] * 0.6`
- Se actualiza en cada frame del game loop

---

## Animaciones destacadas

### Reset score
1. Número se desliza desde derecha (0.7s cubic-bezier)
2. Escala a enorme + blur de fondo (0.4s)
3. Vibración 0.5s (visual + sonido vibrate.wav)
4. Vibración se detiene abruptamente
5. Fade-out del número 1.5s + sonido desolve.wav
6. Cleanup, score = 0

### Muerte
1. Partículas desde el área del jugador (física: gravedad 200px/s², fricción 0.97, rotación)
2. Filtro CSS `grayscale()` progresa 0→100% durante 1.5s
3. Screen shake (0.4s, amplitud 35px)
4. Jugador no se dibuja durante ni después de la explosión

---

## Eventos principales (JS)

### Game loop
```js
function gameLoop(timestamp) {
  var dt = Math.min((timestamp - state.lastTime) / 1000, 0.05);
  state.lastTime = timestamp;
  update(dt);
  render(timestamp / 1000);
  drawVisualizer();
  if (state.running || state.exploding) requestAnimationFrame(gameLoop);
}
```

### update(dt)
- Si `exploding`: mueve partículas, decrementa explosionTimer + shakeTimer, muestra death screen al terminar
- Si `!running || dead`: return
- Si popup visible: dt *= 0.25
- Movimiento del jugador según `state.controlMapping` (con ralentización si está sobre goo)
- Enemigos persiguen al jugador, dejan rastro goo si se mueven >6px
- Goo puddles se desvanecen con decay
- Colisión gema → score++ → nuevo mapping + popup + sonido + spawn más enemigos
- Cálculo de peligro (`state.danger`) basado en distancia al enemigo más cercano
- Screen shake sincronizado con beat si danger > 15%
- `updateSongSaturation()` distorsiona música según peligro
- Colisión enemigo → `showDeathScreen()`

### render(time)
- Si `exploding`: aplica `ctx.filter = grayscale(progress%)`
- Si `state.shakeTimer > 0`: aplica desplazamiento aleatorio (screen shake)
- Dibuja en orden: goo → grid → gemas → enemigos → jugador → partículas (si exploding)
- Si danger > 0 y no dead/exploding: dibuja vignette rojo pulsante
- Resetea filter a 'none'

### drawVisualizer()
- Si `!analyserNode`: return
- `getByteFrequencyData(freqArray)`
- Limpia vizCanvas, dibuja barras en los 4 bordes
- Cada barra usa `freqIdx(j, count)` para muestrear uniformemente del espectro
- Función `barColor()`: gradiente verde-azul con brillo variable

---

## Variables clave del state

```js
const state = {
  player: { x, y, size },
  enemies: [{ x, y, lastX, lastY, size, speed }],
  gems: [{ x, y, size, bob }],
  gooPuddles: [{ x1, y1, x2, y2, size, life, decay }],
  particles: [{ x, y, vx, vy, size, color, life, decay, rot, rotSpeed }],
  score, time, record,
  keys: {}, controlMapping,
  dead, running, lastTime,
  exploding, explosionTimer,
  danger, shakeTimer, shakeDuration, shakeIntensity,
  controlTimeout, gemSpawnTimer,
};
```

---

## Variables del audio

```js
var audioCtx, gainNode, shaperNode, analyserNode;
var freqArray, songBuffer, songSourceNode;
var audioReady = false, songPending = false;
```

- `initAudioSystem()`: crea AudioContext, analizador, waveshaper, gain. Carga canción con XHR + decodeAudioData
- `startSong()`: reanuda context si suspendido, crea BufferSource, conecta, reproduce
- `stopSong()`: detiene y desconecta BufferSource
- `updateSongVolume()`: ajusta gainNode = master × music
- `updateSongSaturation()`: calcula distorsión WaveShaper según peligro

---

## Cómo ejecutar

**Requiere servidor HTTP** (el visualizer no funciona con `file://` por CORS de Chrome/Brave).

### VS Code + Live Server (recomendado)
1. Instalar extensión "Live Server" (Ritwick Dey)
2. Abrir carpeta `juegoweb`
3. Right-click `index.html` → "Open with Live Server"
4. Se abre en `http://127.0.0.1:5500/`

### Terminal
```bash
npx http-server
# o: python -m http.server
# Abrir http://localhost:8080
```

### Alternativa: Firefox
Firefox permite `createMediaElementSource` y XHR con `file://`. Abrir `index.html` directamente.

---

## Git — historial

```
6d62643 gameplay_checkpoint
5f51a04 Add background music, danger vignette, screen shake, death shake
4d47202 Increase grayscale to 100%, apply to particles too
d78bf57 Fix player reappearing on death screen
e4fc22f Shatter player into cyan/teal particles on death
dff52d7 Add death explosion: red/black particles, grayscale fade, death sound
4a08236 Make vibrate sound drier (rapid short pulses)
3cdf2de Add vibrate and desolve sounds for reset score animation
b9749ba Make click sound drier (sine 450Hz, exponential decay)
07e589c Add hover and click UI sounds
db410e0 Add sound menu with master/intro/sike volume sliders
f642ee6 Prevent same gem intro from playing twice in a row
0c2bc86 Make gem variants more distinct
3a46469 Add 5 random gem intro variants (dark tones)
69fc5e5 Play synth intro WAV then main MP3 with fade
41a8d25 Fix audio playback: restore Audio element with volume fade-out
04c2334 Add audio fade-out and enlarge SIKE text
4ecad5b Swap audio file
6009c8a Update audio reference to gem.mp3
375374c Replace gem sound with user's audio file
8d165a5 Make gem sound lower and darker
616c33a Replace gem sound with harsher square-wave tone
7518dfb Add CC0 gem pickup sound
eb22e85 Initial commit - Sike.io
```

---

## Bugs conocidos / pendientes

- [ ] **Visualizer solo funciona con HTTP** — Chrome/Brave bloquea `createMediaElementSource` y XHR en `file://`
- [ ] **Saturación del WaveShaper** puede generar distorsión excesiva con enemies muy cercanos (falta clamping)
- [ ] **El goo puddle** usa `gg.size` como radio lineal — podría refinarse la detección de colisión
- [ ] **Dificultad** escala lineal pero no hay tope máximo de enemigos — puede volverse injugable
