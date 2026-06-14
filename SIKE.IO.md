# Sike.io

Juego de supervivencia 2D estilo arena. Controlas un cuadrado cian que recolecta gemas doradas mientras huye de enemigos rojos. **Cada gema reasigna aleatoriamente las teclas de movimiento**, volviendo el control más caótico con cada colección.

> ⚠️ **Solo para PC / teclado físico.** La mecánica central depende del teclado (reasignación WASD → letras aleatorias). No funciona en móvil.

---

## Stack

- Un solo archivo: `index.html` (HTML + CSS + JS vanilla, ~1388 líneas)
- Canvas 2D (960×720, escalado responsive vía `95vmin`)
- Web Audio API + etiquetas `<audio>` para sonidos
- `localStorage` para persistencia del récord
- Git para control de versiones (21 commits)

## Estructura

```
juegoweb/
├── index.html              # Todo el juego
├── SIKE.IO.md              # Esta documentación
├── assets/
│   └── audio/
│       ├── gem1.wav        # Intro gama variante 1
│       ├── gem2.wav        # Intro gema variante 2
│       ├── gem3.wav        # Intro gema variante 3
│       ├── gem4.wav        # Intro gema variante 4
│       ├── gem5.wav        # Intro gema variante 5
│       ├── gem.mp3         # Segundo sonido gema (importado)
│       ├── hover.wav       # Hover botones
│       ├── click.wav       # Click botones
│       ├── vibrate.wav     # Vibración reset score
│       ├── desolve.wav     # Desvanecimiento reset score
│       └── death.wav       # Sonido de muerte
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
2. Se genera un nuevo mapping aleatorio del pool del tier actual
3. Popup "SIKE" muestra las teclas por 2s (ralentiza el juego 75%)
4. El popup en modo panel se muestra en la esquina izquierda

---

## Flujo del juego

### Menú principal
- **PLAY**: inicia la partida
- **OPTIONS** → SOUND (MASTER / GEM INTRO / SIKE VOLUME) + RESET SCORE (animación)

### HUD
TIME | GEMS | RECORD — esquina superior del game container.

### Muerte
1. Colisión con enemigo → `showDeathScreen()`
2. Explosión: 50 partículas (paleta cian/teal/blanco) desde el área del jugador
3. Canvas pasa a BN progresivo 0% → 100% en 1.5s
4. Suena `death.wav`
5. Aparece pantalla DEFEATED con estadísticas
6. Opciones: PLAY AGAIN (reinicia directo) / BACK TO MENU

---

## Sonidos (10 archivos)

### Sintéticos generados con Python
| Archivo | Característica | Duración |
|---|---|---|
| `gem1-5.wav` | Onda cuadrada + armónicos, tonos graves variados (80-400Hz) | 0.25-0.5s |
| `hover.wav` | Senoidal 1000Hz, ataque rápido, decaimiento cuadrático | 0.04s |
| `click.wav` | Senoidal 450Hz, decaimiento exponencial | 0.05s |
| `vibrate.wav` | Pulsos rápidos 30Hz, tono 550Hz | 0.35s |
| `desolve.wav` | Descendente 400→100Hz + noise | 1.5s |
| `death.wav` | Golpe seco 100→40Hz + cola noise | 0.5s |

### Importado
| Archivo | Propósito |
|---|---|
| `gem.mp3` | Segundo sonido al recoger gema (se reproduce tras el intro) |

### Sistema de volumen
Tres sliders: MASTER × GEM INTRO, MASTER × SIKE VOLUME. Botones UI usan MASTER × 0.25/0.45.

---

## Animaciones destacadas

### Reset score
1. Número se desliza desde derecha (0.7s cubic-bezier)
2. Escala a enorme + blur de fondo (0.4s)
3. Vibración 0.5s (visual: shake CSS / sonido: vibrate.wav)
4. Vibración se detiene abruptamente
5. Fade-out del número 1.5s + sonido desolve.wav
6. Cleanup, score = 0

### Muerte
1. Partículas desde el área del jugador (física: gravedad 200px/s², fricción 0.97, rotación)
2. Filtro CSS `grayscale()` progresa 0→100% durante 1.5s
3. Jugador no se dibuja durante ni después de la explosión

---

## Eventos principales (JS)

### Game loop
```js
function gameLoop(timestamp) {
  var dt = Math.min((timestamp - state.lastTime) / 1000, 0.05);
  state.lastTime = timestamp;
  update(dt);
  render(timestamp / 1000);
  if (state.running || state.exploding) requestAnimationFrame(gameLoop);
}
```

### update(dt)
- Si `exploding`: mueve partículas, decrementa timer, muestra death screen al terminar
- Si `!running || dead`: return (durante death screen)
- Si popup visible: dt *= 0.25
- Movimiento del jugador según `state.controlMapping`
- Enemigos persiguen al jugador
- Colisión gema → score++ → nuevo mapping + popup + sonido
- Colisión enemigo → `showDeathScreen()`

### render(time)
- Si `exploding`: aplica `ctx.filter = grayscale(progress%)`
- Dibuja grid, gemas, enemigos, jugador
- Si `exploding`: dibuja partículas
- Resetea filter a 'none'

---

## Git — historial

```
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

## Cómo ejecutar

Abrir `index.html` en cualquier navegador moderno (Chrome, Firefox, Edge). Sin servidor necesario — todo es local.

Para desarrollo:
```bash
# Ver estado
git status

# Ver historial
git log --oneline

# Crear commit
git add -A
git commit -m "mensaje"
```

Para subir a GitHub:
```bash
git remote add origin https://github.com/tu-usuario/sike.io.git
git push -u origin master
```
