# Sistema de guardado — Jodedor Valley

Este documento explica cómo funciona el sistema de guardado del juego (`index.html`),
cómo está pensado para sobrevivir a futuras actualizaciones de contenido, y qué debes
hacer cada vez que añadas una mecánica nueva.

Todo el sistema vive dentro de `index.html`, en un bloque claramente delimitado
justo antes de `</script>`, con el comentario `SISTEMA DE GUARDADO`. No sustituye ni
reescribe ninguna función de juego existente: solo **lee** las variables del juego para
guardar y las **reescribe** para cargar.

## 1. Dónde se guardan las partidas

- El juego es una página estática (pensada para GitHub Pages), sin servidor propio, así
  que las partidas se guardan en `localStorage` del navegador del jugador.
- Clave de la partida activa: `jodedorValley.save.slot1`
- Clave de la copia de seguridad automática: `jodedorValley.save.slot1.backup`
- Si una partida llega corrupta, se conserva además una copia con una clave del tipo
  `jodedorValley.save.slot1.corrupt-<timestamp>` (nunca se sobrescribe ni se borra sola).

El `slot1` es el hueco de partida que se usa hoy. La arquitectura ya admite varios huecos
(`slot2`, `slot3`, ...) sin cambios de fondo — ver la sección 9.

Como `localStorage` es local al navegador, cada jugador tiene su propia partida en su
propio ordenador/navegador. Por eso existen también **exportar/importar** (sección 8),
para que puedan mover o respaldar su partida manualmente.

## 2. Estructura de una partida (`saveVersion`)

Toda partida guardada es un objeto JSON con esta forma:

```json
{
  "saveVersion": 1,
  "savedAt": "2026-01-01T10:00:00.000Z",
  "player": { "worldId": "ciudad", "x": 1182, "y": 706, "facing": "down" },
  "inventory": ["kebab", "llaves"],
  "discovered": { "npcs": ["Xokas"], "items": ["kebab"], "buildings": ["tecnocasa"] },
  "flags": { "money": 100, "questXokasDone": true, "patoCount": 3, "...": "..." },
  "quests": [{ "id": "xokas", "done": true, "hidden": false }, "..."],
  "flagPoles": [{ "id": "bandera-roja", "filled": false }, "..."],
  "museumDisplays": ["museoShirtDisplay"],
  "collectedWorldItems": ["gilletteItem", "patoPlaya"]
}
```

- **`saveVersion`**: número de versión de la ESTRUCTURA de la partida (no de la versión
  del juego). Solo se incrementa cuando se hace un cambio **incompatible** en cómo se
  guardan los datos (por ejemplo, cambiar la forma de un campo, no simplemente añadir uno
  nuevo — añadir campos nuevos NO requiere subir la versión, ver sección 3).
- La constante hermana en el código es `CURRENT_SAVE_VERSION` (hoy vale `1`, porque este
  es el primer sistema de guardado que tiene el juego).

### Qué NO se guarda (a propósito)

Siguiendo el principio de "guarda solo el estado, no los datos estáticos del juego":

- Los objetos, NPCs, edificios, precios, sprites, diálogos, etc. siguen viviendo en las
  constantes del juego (`codexItems`, `codexNpcs`, `codexBuildings`, arrays de líneas de
  diálogo, etc.). La partida solo guarda **ids** (`"kebab"`, `"tecnocasa"`, `"Xokas"`...) y,
  cuando aplica, cantidades (`patoCount`, `zanahoriaCount`...). Nunca se duplica un icono,
  un precio o un texto dentro de la partida.
- Estado puramente de interfaz (menú de objetivos abierto, cuadro de contraseña abierto,
  test de trivia en curso...) no se guarda: no tiene sentido "recordar" un menú abierto.
- El orden exacto en que el jugador arrastró los objetos dentro de la barra de acceso
  rápido no se guarda tal cual: al cargar, los objetos se recolocan automáticamente en la
  barra (mismos objetos y cantidades, pero no necesariamente en el mismo hueco). Es una
  simplificación consciente para no tener que guardar y validar 10 punteros de UI.
- Si guardas estando dentro de un edificio (interior), al cargar reapareces justo fuera de
  ese edificio, en el mapa exterior, en vez de reconstruir la escena interior exacta. Los
  interiores son escenas cortas (tiendas, casas) y no tiene sentido "recordar" que estabas
  a mitad de una conversación.
- Un viaje en curso al safari con acompañantes (Mara/Zeling siguiéndote) no se conserva
  tal cual; es un estado transitorio de una "escena", no de progreso.

## 3. Compatibilidad hacia atrás (añadir cosas nuevas sin migración)

Hay dos maneras de que una partida antigua reciba datos nuevos, y el sistema ya soporta
ambas sin que tengas que tocar partidas guardadas a mano:

### 3.1 Flags/contadores nuevos (el caso más común)

Todas las variables sueltas de progreso (`questXxxDone`, contadores, "ya conocí a
fulano"...) se guardan y se cargan mediante dos funciones gemelas:

```js
function collectFlags(){ return { money, questXokasDone, /* ... */ }; }
function applyFlags(f){
  if(f.money!==undefined) money=f.money;
  if(f.questXokasDone!==undefined) questXokasDone=f.questXokasDone;
  /* ... */
}
```

`applyFlags` solo **sobreescribe** una variable si la partida cargada trae esa clave. Si
una partida antigua no la trae, la variable se queda con el valor inicial que ya tiene en
su `let` de declaración (ej. `let questNuevaMision=false;`). Es decir: **el valor por
defecto de tu nueva variable, tal y como la declares, ya es el valor que recibirán las
partidas antiguas.** No hace falta escribir ninguna migración para esto.

**Qué hacer cuando añades una variable de progreso nueva:**
1. Declárala donde ya se declaran las demás (`let miNuevaVariable=false;`).
2. Añade una línea en `collectFlags()`: `miNuevaVariable,`
3. Añade una línea en `applyFlags()`: `if(f.miNuevaVariable!==undefined) miNuevaVariable=f.miNuevaVariable;`

Eso es todo. No hace falta migración ni subir `CURRENT_SAVE_VERSION`.

### 3.2 Misiones, NPCs, edificios, banderas, objetos del museo — genérico

Estas partes NO usan una lista fija de nombres: se leen y se escriben inspeccionando el
propio DOM/HTML:

- **Misiones** (`.quest-item[data-quest]`): se guarda `{id, done, hidden}` de cada una
  leyendo directamente sus clases CSS. Si añades una misión nueva en el HTML
  (`<div class="quest-item hidden-quest" data-quest="mi-mision-nueva">...`), una partida
  antigua simplemente no traerá esa entrada, así que la misión se queda tal cual está en
  el HTML (oculta, sin completar) — que es exactamente el estado inicial correcto.
- **Postes de banderas** (`.flag-pole[data-flag-item]`): igual, por `data-flag-item`.
- **Objetos donados al museo**: cualquier elemento `id="museo...Display"` que esté
  visible se guarda por su id. Un nuevo objeto donable que siga esa misma convención de
  nombre de id ya funciona automáticamente, sin tocar el sistema de guardado.
- **Objetos del catálogo** (`codexItems`): el inventario y la barra de acceso rápido solo
  guardan **ids**. Cualquier objeto nuevo que añadas a `codexItems` (icono, nombre) ya es
  visible para el sistema de guardado en cuanto lo añades ahí — no hay que tocar nada más.

### 3.3 Zonas nuevas (mapas)

Cada mapa es un `<div class="world" id="...">` con su propio id. La posición del jugador
se guarda como `{ worldId, x, y, facing }`. Si en el futuro añades una zona nueva:

1. Añade una entrada en `WORLD_CONFIG` (dentro del bloque de guardado) con su elemento,
   sus dimensiones y los flags de "en qué mapa estoy" que ya usa el juego para esa zona
   (mira cómo están las demás entradas, es siempre el mismo patrón de 3-4 líneas).
2. Si la zona debe empezar bloqueada para partidas antiguas, gestiona ese bloqueo con el
   mismo patrón que ya usa el juego para zonas bloqueadas (una clase CSS
   `hidden-until-...` + una condición al entrar), y añade la línea correspondiente en
   `applyRevealsAndRemovals()` para que se vuelva a aplicar al cargar una partida antigua
   que SÍ haya cumplido esa condición.

### 3.4 Ejemplo completo: añadir "mascotas" (`pets`)

Esto es un ejemplo de una función nueva que SÍ necesita su propio campo en el save
(porque no encaja en "flags sueltos" ni en nada genérico ya existente, sería una lista de
objetos con su propio estado):

```js
// 1. Variable de estado real del juego (donde ya declares el resto):
let pets = []; // [{id:'perro1', name:'Toby', hunger:80}, ...]

// 2. En buildSaveObject(), añade una línea:
pets: pets,

// 3. En applySaveObject(), añade una línea (con valor por defecto []):
pets = save.pets || [];
```

Con esto, una partida vieja que no tiene mascotas simplemente carga `pets = []` (el valor
por defecto razonable que pide la mecánica), y no hace falta ninguna migración porque no
estás cambiando la FORMA de datos que ya existían — solo estás añadiendo un campo nuevo,
opcional, con valor por defecto.

## 4. Cuándo SÍ hace falta una migración

Solo cuando cambias la **forma** de datos que una partida antigua ya tenía guardados de
otra manera. Ejemplos reales de "esto sí necesita migración":

- Cambiar `inventory` de ser un array de strings a ser un array de objetos
  `{id, quantity}`.
- Renombrar un campo (`money` → `coins`).
- Cambiar cómo se identifica el mapa activo (`worldId` de string a un índice numérico).

Cuando eso pase:

1. Sube `CURRENT_SAVE_VERSION` en 1 (por ejemplo, de `1` a `2`).
2. Añade una función en la tabla `MIGRATIONS`, indexada por la versión DESTINO:

```js
const MIGRATIONS = {
  2: function(save){
    // save.saveVersion todavía vale 1 en este punto; transforma lo que haga falta.
    save.inventory = (save.inventory||[]).map(id => ({ id, quantity: 1 }));
    return save;
  }
};
```

3. No hace falta tocar nada más: `migrateSave()` ya se encarga de:
   - Detectar `save.saveVersion` (si no existe, asume `1`, nunca rechaza la partida).
   - Ejecutar `MIGRATIONS[2]`, luego marcar `save.saveVersion = 2`.
   - Si más adelante subes a `CURRENT_SAVE_VERSION = 3`, seguirá encadenando:
     ejecuta `MIGRATIONS[2]` → `MIGRATIONS[3]` → ... hasta llegar a la versión actual,
     siempre en orden, nunca salta pasos.
   - Una partida ya actualizada (`saveVersion === CURRENT_SAVE_VERSION`) no ejecuta
     ninguna migración: el `while` de `migrateSave()` no entra ni una vez.

Este es el mismo patrón que pedías en el encargo original:

```js
if (save.saveVersion < 2) { /* adaptar a v2 */ save.saveVersion = 2; }
if (save.saveVersion < 3) { /* adaptar a v3 */ save.saveVersion = 3; }
```//
`migrateSave()` hace exactamente esto pero en un bucle genérico, para no tener que
repetir el mismo `if` a mano cada vez.

## 5. Partidas corruptas

Si el JSON guardado no se puede parsear:

1. **No se sobrescribe** nada todavía.
2. Se intenta cargar automáticamente la copia de seguridad (`...backup`, ver sección 6).
   Si esa copia es válida, el juego arranca con ella y avisa
   ("Partida cargada (desde copia de seguridad)").
3. Si ni la partida ni la copia de seguridad son legibles, el JSON dañado se guarda tal
   cual bajo una clave `...corrupt-<timestamp>` (para poder rescatarlo a mano más
   adelante si hiciera falta) y el juego arranca una partida nueva, mostrando un aviso
   claro en el panel 💾: *"Tu partida guardada no se pudo leer (dañada). Se ha guardado
   una copia y se ha iniciado una partida nueva."* El juego nunca se queda colgado ni
   lanza un error sin control por esto.

## 6. Copias de seguridad automáticas

Cada vez que se guarda (`saveGame()`), si ya había una partida previa en esa clave, esa
partida previa se copia primero a `jodedorValley.save.slot1.backup` **antes** de
sobrescribirla. Así, un bug en una actualización que corrompa el guardado más reciente
como mucho te deja con la copia de seguridad del guardado anterior, nunca con nada.

## 7. Cuándo se guarda (autoguardado)

Este juego no tenía ningún sistema de guardado antes de esto, así que no había
autoguardado que "mantener". Se ha añadido:

- **Autoguardado periódico**: cada 60 segundos mientras el juego está abierto.
- **Guardado al salir**: al cerrar la pestaña o navegar fuera (`beforeunload` /
  `pagehide`), intenta guardar una última vez.
- **Guardado manual**: botón "Guardar ahora" en el panel 💾 (arriba a la izquierda).

Esto no cambia la jugabilidad ni el ritmo del juego — es puramente en segundo plano.

## 8. Exportar / Importar partida

En el panel que se abre con el botón 💾 (arriba a la izquierda, junto al de música):

- **Exportar partida**: descarga la partida actual como un archivo
  `jodedor-valley-save-slot1-<timestamp>.json`.
- **Importar partida**: abre un selector de archivos; el JSON elegido se valida, se migra
  si hace falta, se guarda como la partida activa y se aplica inmediatamente al juego en
  curso. Si el archivo no es válido, se avisa sin tocar la partida actual.

## 9. Varias partidas — activado mediante LOGIN por usuario

La API interna (`saveGame(slot)`, `loadGame(slot)`, `exportSave(slot)`,
`importSaveFromFile(file, slot)`, `readSaveFromStorage(slot)`,
`writeSaveToStorage(slot, ...)`) siempre aceptó un `slot` como parámetro opcional, y
`listSaveSlots()` enumera todos los huecos guardados en `localStorage`. Esa arquitectura
es la que hace posible el sistema de login: cada usuario registrado tiene su propio
`slot` (`user_<nombre normalizado>`), así que la partida de cada persona vive en un hueco
de `localStorage` completamente separado.

## 9.1 Login / registro (pantalla inicial)

Al abrir `index.html` siempre aparece primero una pantalla de login (`#authGate`) con
pestañas **Iniciar sesión** / **Registrarte**. El jugador tiene que volver a escribir su
usuario y contraseña cada vez que abre la página (no hay sesión persistente entre
recargas, tal y como se pidió) — esto es intencional, no un fallo.

**Cómo funciona por dentro** (todo en el mismo bloque de `index.html`, sección
`LOGIN / REGISTRO`):

- Los usuarios registrados en este navegador se guardan en `localStorage`, clave
  `jodedorValley.users`, como `{ "<usuario normalizado>": { displayName, salt,
  passwordHash, createdAt } }`.
- La contraseña **nunca** se guarda en texto plano: se combina con una sal aleatoria por
  usuario y se aplica SHA-256 (`crypto.subtle.digest`) antes de guardarla.
- Al iniciar sesión o registrarte con éxito, se llama a
  `startSessionForSlot('user_'+usuario)`, que es exactamente el mismo mecanismo de
  "arrancar sesión de guardado" que antes se ejecutaba solo al cargar la página — ahora
  se dispara tras el login en vez de automáticamente.
- Una cuenta nueva (sin partida previa en su slot) arranca siempre desde
  `DEFAULT_SAVE_SNAPSHOT`, una foto exacta del estado "de fábrica" del juego tomada en el
  momento en que se carga la página (antes de que nadie inicie sesión). Esto evita que el
  progreso de una cuenta se filtre a otra si, dentro de la misma pestaña, alguien prueba
  varias cuentas seguidas sin recargar.

**⚠️ Importante — qué NO es este sistema:** el juego se publica como páginas estáticas en
GitHub Pages, sin servidor ni base de datos propia. Este login es una separación de
partidas *local a cada navegador*, pensada para que varias personas que comparten el
mismo ordenador tengan cada una su propia partida — **no** es una cuenta real verificada
en un servidor, no sincroniza entre dispositivos, y no protege la contraseña frente a
alguien con acceso a las herramientas de desarrollador de ese navegador. Si en el futuro
hiciera falta una cuenta "de verdad" (multi-dispositivo, recuperación de contraseña,
etc.), habría que añadir un backend — el sistema de guardado en sí (versionado,
migraciones, export/import) seguiría funcionando igual por encima de eso.

**Añadir "cerrar sesión y cambiar de usuario" sin recargar:** hoy no existe ese botón (no
se pidió); la forma de "cambiar de cuenta" es recargar la página, lo cual ya limpia todo
el estado en memoria. Si algún día se añade un botón de cambio de cuenta en caliente,
debe llamar primero a `applySaveObject(JSON.parse(JSON.stringify(DEFAULT_SAVE_SNAPSHOT)))`
antes de `startSessionForSlot(nuevoSlot)` — igual que ya hace el login internamente —
para no arrastrar el estado de la cuenta anterior.

## 10. Qué NO tocar / filosofía general

- El sistema de guardado vive en un bloque separado al final del `<script>`. No sustituye
  ninguna función de juego existente (`addItem`, `talk`, `enterHouse`, etc.) — solo las
  usa como punto de referencia (o llama a alguna, como `removeGemitaStandIfDone()`, para
  no duplicar lógica que ya existe).
- Guarda **estado**, no datos estáticos. Antes de guardar un dato nuevo pregúntate: "¿esto
  puede reconstruirse a partir de un id y una constante del juego?" Si la respuesta es sí,
  guarda el id (y cantidad si aplica), no el dato completo.

## 11. Cómo probar que una actualización no rompe partidas antiguas

En la consola del navegador (con el juego abierto):

```js
runSaveSystemSelfTest()
```

Esto comprueba, entre otras cosas:
- Que guardar y volver a cargar una partida actual no pierde datos (round-trip).
- Que una partida de ejemplo v1 migra correctamente hasta `CURRENT_SAVE_VERSION`.
- Que el motor de migraciones encadena correctamente varios pasos en orden
  (prueba sintética v1→v2→v3→v4, independiente de si esas versiones existen hoy).
- Que un JSON corrupto no rompe el juego.
- Que aplicar una partida sin campos nuevos (`applyFlags({})`) no borra progreso ya hecho.

También hay partidas de ejemplo ya preparadas en `test-saves/`:

- `test-save-v1.json` — partida real con la forma actual (v1).
- `test-save-legacy-no-version.json` — simula una partida sin campo `saveVersion` en
  absoluto, para comprobar que no se rechaza (se asume v1).
- `test-save-missing-fields.json` — partida v1 a la que le faltan campos enteros
  (`inventory`, `discovered`, `quests`...), para comprobar que todo recibe su valor por
  defecto sin lanzar errores.
- `test-save-corrupt.json` — JSON deliberadamente inválido, para probar el botón
  "Importar partida" y comprobar que el juego avisa sin romperse.

Para probarlas manualmente: abre el panel 💾 → "Importar partida" → elige uno de estos
archivos. También puedes cargarlas por consola:

```js
fetch('test-saves/test-save-v1.json').then(r=>r.json()).then(applySaveObject);
```

Antes de publicar una actualización que cambie el `saveVersion`, añade una migración
nueva a `MIGRATIONS`, sube `CURRENT_SAVE_VERSION`, y vuelve a ejecutar
`runSaveSystemSelfTest()` — si sigue en verde, las partidas antiguas están a salvo.

## 12. Referencia rápida de funciones (dentro de `index.html`)

| Función | Qué hace |
|---|---|
| `buildSaveObject()` | Lee el estado actual del juego y devuelve el objeto de partida. |
| `applySaveObject(save)` | Migra si hace falta y aplica una partida al juego en curso. |
| `saveGame(slot?)` | Guarda la partida actual en `localStorage` (con copia de seguridad). |
| `loadGame(slot?)` | Carga una partida desde `localStorage`. |
| `exportSave(slot?)` | Descarga la partida actual como archivo `.json`. |
| `importSaveFromFile(file, slot?)` | Valida, migra y aplica un archivo de partida. |
| `migrateSave(save)` | Ejecuta las migraciones necesarias, en orden, hasta `CURRENT_SAVE_VERSION`. |
| `runSaveSystemSelfTest()` | Ejecuta las pruebas de compatibilidad descritas arriba. |
| `window.__saveSystem` | Expone las funciones anteriores para depurar desde la consola. |
