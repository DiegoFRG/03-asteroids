# Asteroids — Overview

Clon del arcade **Asteroids** implementado en canvas HTML5 puro. Sin dependencias, bundler ni build: dos archivos (`index.html` + `game.js`) y un `favicon.svg`. Toda la lógica vive en `game.js` (423 líneas).

## Arquitectura general

- `index.html` — solo el shell: un `<canvas id="canvas">` fijo de 800x600 y la carga de `game.js` como **script clásico** (no módulo).
- `game.js` — todo el juego: input, utilidades, clases de entidades (`Bullet`, `Asteroid`, `Ship`, `Particle`), estado global, `update`/`draw` y el bucle principal.
- El tamaño del canvas está duplicado: `width="800" height="600"` en el HTML y `W = 800` / `H = 600` en `game.js:5-6`. Si cambias uno, cambia el otro.

## Entrada (`game.js:8-24`)

- Dos mapas de estado: `keys` (tecla mantenida) y `justPressed` (flanco, se activa solo al pulsar).
- `pressed(code)` (`game.js:20`) consume el flanco y lo pone a `false`. Acciones de un solo toque (disparar, reiniciar) usan `pressed()`; acciones continuas (rotar, propulsar) usan `keys`.
- `preventDefault` en flechas y `Espacio` (`game.js:15-16`) para evitar el scroll de la página.

## Utilidades (`game.js:27-30`)

- `wrap(v, max)` — módulo siempre positivo; implementa el espacio toroidal.
- `dist(a, b)` — distancia euclídea directa, **sin** tener en cuenta el envolvimiento (ver gotchas).
- `rand(min, max)` / `randInt(min, max)`.

## Entidades

### `Bullet` (`game.js:33-58`)
- Velocidad fija `SPEED = 520` px/s en la dirección del ángulo.
- `ttl = 1.1` s; `radius = 2`; se marca `dead` al expirar.
- Se crea en la nariz de la nave: offset `NOSE = 21` desde el centro (`game.js:165-168`).

### `Asteroid` (`game.js:65-119`)
- Tamaño indexado 1..3 en arrays paralelos (`game.js:61-63`):
  - `RADII  = [0, 16, 30, 50]`
  - `SPEEDS = [0, 85, 55, 32]` (velocidad base; más pequeños = más rápidos)
  - `POINTS = [0, 100, 50, 20]`
  - El índice 0 es un centinela sin uso.
- Velocidad real = base + `rand(-15, 15)`; rotación `rand(-1.2, 1.2)` rad/s.
- Dibujo: polígono irregular de 8-13 vértices, radio de cada vértice `rand(0.6, 1.0) * radius`.
- `split()` (`game.js:96-102`) devuelve 2 asteroides de `size - 1`; para en `size <= 1`. Los hijos nacen en la posición exacta del padre, sin separación.

### `Ship` (`game.js:122-204`)
- Constantes: `ROT = 3.5` rad/s, `THRUST = 260` px/s², `DRAG = 0.987` (aplicado por frame, no por segundo).
- Invencibilidad de reaparición: `invincible = 3` s; parpadea con `Math.floor(invincible * 8) % 2 === 0` (`game.js:174`).
- Cooldown de disparo: `0.2` s (`game.js:164`); `tryShoot()` devuelve `[]` si está en cooldown o muerta.
- Silueta clásica: triángulo con muesca trasera (`game.js:184-190`); llama de propulsor aleatoria cuando `thrusting` (`game.js:193-200`).

### `Particle` (`game.js:207-236`)
- Velocidad `rand(30, 130)`, `life = ttl = rand(0.4, 1.1)`.
- Alpha de dibujo = `ttl / life`; se dibuja como una pequeña estela en dirección opuesta a su velocidad.

## Estado global (`game.js:239-242`)

- `state`: `'playing' | 'dead' | 'gameover'`.
- `ship`, `bullets`, `asteroids`, `particles`, `score`, `lives`, `level`, `deadTimer`.
- `initGame()` (`game.js:256`) — partida nueva: `lives = 3`, `level = 1`, `state = 'playing'`, 4 asteroides.
- `nextLevel()` (`game.js:268`) — `level++`, limpia balas/partículas, `ship.reset()` y genera `3 + level` asteroides.
- `spawnAsteroids()` (`game.js:244`) evita generar en un radio de `SAFE_DIST = 130` px del centro (posición de reaparición de la nave).

## Lógica de `update(dt)` (`game.js:293-351`)

Ramas por estado:
- `'gameover'` — solo actualiza partículas; `Espacio` llama a `initGame()`.
- `'dead'` — cuenta `deadTimer` (2 s), actualiza partículas y asteroides (el mundo sigue vivo); al llegar a 0, `state = 'playing'` y `ship.reset()`.
- `'playing'` — orden de ejecución:
  1. Disparo si `pressed('Space')`.
  2. `ship.update`, `bullets/asteroids/particles.update`.
  3. Filtrado de balas y partículas muertas.
  4. **Colisión bala vs. asteroide** — doble bucle; si `dist(b, a) < a.radius`: mata bala y asteroide, suma `POINTS[a.size]`, genera `explode(a.x, a.y, a.size * 5)` partículas y acumula `a.split()`.
  5. **Colisión nave vs. asteroide** — solo si `ship.invincible <= 0`; umbral `ship.radius + a.radius * 0.82`; llama a `killShip()`.
  6. **Nivel completado** — si `asteroids.length === 0`, `nextLevel()`.
- `killShip()` (`game.js:280`) — 14 partículas, `ship.dead = true`, `lives--`; si `lives <= 0` → `gameover`, si no → `'dead'` con `deadTimer = 2`.

## Render (`game.js:353-409`)

- Fondo negro, luego capas: partículas → asteroides → balas → nave.
- `drawHUD()` (`game.js:371`) — `SCORE` (izq.), `NIVEL` (centro), iconos de vidas como mini-naves (der., `drawLifeIcon`).
- `drawOverlay()` (`game.js:386`) — "GAME OVER" + puntaje y "ESPACIO PARA REINICIAR".
- Texto de UI en español.

## Bucle principal (`game.js:414-420`)

- `requestAnimationFrame`; `dt = (ts - lastTime) / 1000` limitado con `Math.min(..., 0.05)` para evitar saltos tras pestañas inactivas.
- El primer frame usa `dt = 0`. `initGame()` + primer `requestAnimationFrame(loop)` al final (`game.js:422-423`).

## Puntuación y progresión

- Puntos: grande 20, mediano 50, pequeño 100 (matar lo pequeño vale más).
- Dificultad por nivel: número de asteroides = `3 + level`; el nivel avanza al limpiar el campo.

## Gotchas / casos límite

- **Colisiones sin wrap**: `dist()` es euclídea, así que un asteroide pegado al borde no colisiona con objetos del lado opuesto, aunque visualmente se tocan.
- **`justPressed` se consume al leerlo**: si dos sistemas llaman a `pressed()` en el mismo frame, el segundo no ve la tecla.
- **`dt` acotado a 0.05**: la física no es de paso fijo; un frame largo se trata como 50 ms.
- **`DRAG` por frame**: se aplica una vez por `update`, no escala con `dt` ni con la tasa de refresco.
- **Hijos de `split()` sin separación**: nacen superpuestos en la posición del padre.

## Discrepancias con el README

El `README.md` menciona "power-ups especiales" y "tipos de asteroides únicos como la estrella fugaz", pero **no existen en el código actual**. Son funcionalidades no implementadas.

## Verificación

No hay tests, lint ni build. La verificación es manual: abrir `index.html` en el navegador o `npx serve .` (`http://localhost:3000`) y jugar.
