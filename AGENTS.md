# AGENTS.md

Guía para agentes de IA (y humanos) que trabajan en este repositorio.
Léela completa antes de escribir código. Si algo aquí contradice una petición puntual, pregunta antes de romper una regla marcada como **OBLIGATORIO**.

---

## 1. Qué es este proyecto

Un **juego de pelea de plataformas tipo "Smash"** para navegador, en 3D con jugabilidad 2.5D (el movimiento ocurre en el plano X/Y; Z solo es visual).

- Hasta **8 jugadores** online por partida.
- **Servidor autoritativo**: el servidor decide el estado real; el cliente predice y se corrige.
- Objetivo de rendimiento: **60 FPS en cliente** y **tick de simulación fijo de 60 Hz** en servidor.

---

## 2. Stack tecnológico

| Componente | Tecnología | Uso |
|---|---|---|
| Gráficos 3D | Three.js | Render de personajes, escenarios y VFX |
| Físicas | Rapier.js (`@dimforge/rapier2d-compat` / `rapier3d-compat`) | Colisión con el escenario y controlador cinemático |
| Red | Geckos.io (WebRTC, canales no fiables) | Transporte tipo UDP cliente ↔ servidor |
| Servidor | Node.js (LTS) | Bucle de juego autoritativo |
| Lenguaje | TypeScript (`strict: true`) | Cliente, servidor y código compartido |
| Serialización | ArrayBuffer + DataView | Protocolo binario, nunca JSON en el bucle de juego |
| Build | Vite (cliente), tsx/tsup (servidor) | Desarrollo y empaquetado |
| Tests | Vitest | Unitarios y de determinismo |

No añadas dependencias nuevas sin justificarlo en el PR (tamaño, mantenimiento, alternativa nativa).

---

## 3. Estructura del monorepo

```
/
├── packages/
│   ├── shared/        # Lógica pura: simulación, tipos, protocolo, datos de personajes
│   │   ├── sim/       # step(state, inputs) → state. SIN Three.js, SIN red, SIN DOM
│   │   ├── combat/    # hitboxes, hurtboxes, daño, knockback, hitstun
│   │   ├── protocol/  # encoders/decoders binarios + IDs de mensaje
│   │   ├── data/      # frame data de personajes y escenarios (TS/JSON tipado)
│   │   └── math/      # vectores, fixed-point si se usa, utilidades
│   ├── client/        # Vite + Three.js: render, input, predicción, interpolación, UI, audio
│   └── server/        # Node + Geckos: salas, tick loop, validación, broadcast de snapshots
├── assets/            # Modelos glTF/GLB, texturas (KTX2), audio
├── tools/             # Scripts: editor de hitboxes, generación de assets, bots de carga
└── AGENTS.md
```

Gestor de paquetes: **pnpm workspaces**. Comandos:

```bash
pnpm install
pnpm dev            # cliente + servidor en paralelo
pnpm dev:client
pnpm dev:server
pnpm test           # vitest en todos los paquetes
pnpm typecheck      # tsc --noEmit en todos los paquetes
pnpm lint
pnpm build
```

Antes de dar una tarea por terminada: `pnpm typecheck && pnpm lint && pnpm test` deben pasar.

---

## 4. Reglas de arquitectura

### 4.1 Simulación (OBLIGATORIO)

- Toda la lógica de juego vive en `packages/shared/sim` y es **la misma** en cliente y servidor.
- La simulación avanza en **frames fijos** (`DT = 1/60`). Nunca uses `deltaTime` variable ni `Date.now()` dentro de `sim/`.
- `sim/` es una función pura en la práctica: `step(state, inputsDelFrame) → nuevoState`. Sin efectos secundarios, sin `Math.random()` (usa un PRNG con semilla guardada en el estado).
- Prohibido importar `three`, `@geckos.io/*` o APIs del DOM desde `shared/`.
- El render **lee** el estado; nunca lo modifica.

### 4.2 Físicas

- **Rapier se usa para colisión con el escenario**, no para el combate. Los luchadores son **cuerpos cinemáticos** movidos con `KinematicCharacterController` (o lógica AABB propia): la sensación de un juego de pelea requiere control total de velocidad, gravedad y fricción, no un rigidbody dinámico.
- Plataformas atravesables (*one-way*): se resuelven en lógica propia (ignorar colisión si el jugador sube o pulsa abajo).
- **Hitboxes y hurtboxes NO son colliders de Rapier.** Son volúmenes (círculos/cápsulas en 2D) definidos por frame en el frame data y comprobados manualmente en `combat/`. Esto mantiene el combate determinista y depurable.
- Inicializa Rapier una sola vez (`await RAPIER.init()`) tanto en cliente como en servidor.

### 4.3 Combate

- Los ataques se definen por **frame data**: `startup`, `active`, `recovery`, lista de hitboxes por frame (offset, radio, daño, ángulo, knockback base, crecimiento).
- Knockback escalado por porcentaje de daño (estilo Smash). Fórmula en `combat/knockback.ts`, con tests.
- Estados del personaje como **máquina de estados explícita** (`idle`, `run`, `jump`, `airborne`, `attack`, `hitstun`, `shield`, `ledge`, `ko`, …). Nada de booleanos sueltos.
- Buffer de input de ~5 frames para que los comandos no se pierdan.
- Un hit no conecta dos veces con el mismo objetivo en el mismo ataque (lista de "ya golpeados").

### 4.4 Red (OBLIGATORIO)

- **Cliente → servidor:** solo inputs (bitmask de botones + stick cuantizado + número de frame). Nunca posiciones.
- **Servidor → cliente:** snapshots del estado a 20–30 Hz, binarios, con delta respecto al último snapshot confirmado cuando sea posible.
- **Jugador local:** predicción en cliente + reconciliación (re-simular desde el último snapshot con los inputs pendientes).
- **Jugadores remotos:** interpolación con ~100 ms de buffer. Si se nota retraso en golpes, evaluar extrapolación corta.
- **Hit confirmation:** el servidor decide los golpes. El cliente puede mostrar efectos predichos, pero el daño/knockback real llega del servidor.
- Usa canales **no fiables** para inputs y snapshots; canal **fiable** solo para eventos discretos (unirse a sala, KO, fin de partida, chat).
- El servidor valida todo input (rangos, frecuencia, frame plausible). Jamás confíes en el cliente.
- WebRTC necesita un rango de puertos UDP abierto en el servidor y, en algunas redes, TURN. Documenta la configuración en `packages/server/README.md`.

### 4.5 Protocolo binario

- Cada mensaje empieza con un `uint8` de tipo (`protocol/messageIds.ts`).
- Cada mensaje tiene un `encodeX` y `decodeX` en `shared/protocol`, con **test de ida y vuelta** obligatorio.
- Cuantiza: posiciones a `int16` (cm o unidad fija), ángulos a `uint8`, porcentaje a `uint16`.
- Little-endian siempre (`DataView` con `true`).
- Si cambias el formato, incrementa `PROTOCOL_VERSION` y el servidor rechaza clientes de otra versión.

### 4.6 Cliente / render

- Modelos en **glTF/GLB**; animaciones con `AnimationMixer`, sincronizadas al frame de la simulación (no al reloj de pared).
- Reutiliza geometrías y materiales; nada de `new` en el bucle de render. Usa pools para partículas y VFX.
- Cámara: sigue al centro del grupo de jugadores y hace zoom para encuadrarlos a todos.
- Modo debug (tecla `F3`): dibuja hitboxes (rojo), hurtboxes (amarillo), colliders del escenario (verde), frame actual, ping y estado de la máquina.

---

## 5. Convenciones de código

- TypeScript `strict`, sin `any` (usa `unknown` + validación). Sin `// @ts-ignore` salvo justificado en comentario.
- Nombres: `camelCase` para variables/funciones, `PascalCase` para tipos/clases, `SCREAMING_SNAKE_CASE` para constantes de gameplay.
- Constantes de gameplay (gravedad, velocidad, frames) en `shared/data`, nunca "números mágicos" en la lógica.
- Archivos pequeños y con un propósito. Preferir funciones a clases en `shared/sim`.
- En el hot path (simulación, serialización, render): evita crear objetos/arrays por frame; reutiliza buffers.
- Comentarios en español; identificadores en inglés.
- Commits: Conventional Commits (`feat:`, `fix:`, `perf:`, `refactor:`, `test:`, `docs:`).

---

## 6. Testing

- **Determinismo:** test que ejecuta N frames con una secuencia de inputs fija dos veces y compara el estado byte a byte.
- **Combate:** tests de knockback, hitstun y colisión hitbox/hurtbox con casos conocidos.
- **Protocolo:** ida y vuelta (encode → decode) para cada mensaje, incluidos valores límite.
- **Carga:** `tools/bots` lanza 8 clientes simulados contra el servidor; el tick debe mantenerse < 4 ms de media.

---

## 7. Rendimiento (presupuestos)

| Métrica | Objetivo |
|---|---|
| Tick del servidor (8 jugadores) | < 4 ms |
| Frame del cliente | < 16,6 ms (60 FPS) en portátil medio |
| Ancho de banda por cliente (bajada) | < 10 KB/s |
| Draw calls | < 150 |
| Tamaño inicial de descarga | < 15 MB (resto lazy) |

Si un cambio empeora alguna métrica, indícalo en el PR con mediciones.

---

## 8. Hoja de ruta (orden recomendado)

1. Simulación local 1 jugador: movimiento, salto, doble salto, plataformas (sin red, sin modelos: cápsulas).
2. Combate local 2 jugadores en el mismo teclado/mando: un ataque, daño, knockback, KO por salir de pantalla.
3. Protocolo binario + servidor autoritativo con 2 jugadores.
4. Predicción, reconciliación e interpolación. Probar con latencia simulada (100–150 ms, 2 % de pérdida).
5. Escalar a 8 jugadores y pasar el test de carga.
6. Modelos glTF, animaciones, VFX, audio, UI, lobby/salas.

No saltes al paso 6 antes de que el paso 4 se sienta bien.

---

## 9. Qué NO hacer

- No uses rigidbodies dinámicos de Rapier para los luchadores.
- No envíes JSON en el bucle de juego.
- No pongas lógica de juego en `client/` ni en `server/`: va en `shared/`.
- No uses `setInterval` como único reloj del servidor sin corregir la deriva (usa acumulador de tiempo).
- No metas assets pesados en el bundle principal.
- No dejes `console.log` en el hot path.