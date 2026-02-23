# Isekai System — Documentación técnica

> Sistema de juego de rol para Foundry VTT v13+  
> Versión: 0.1.0 · Autor: Zinner

---

## Índice

1. [Visión general](#1-visión-general)
2. [Estructura del proyecto](#2-estructura-del-proyecto)
3. [Flujo de inicialización](#3-flujo-de-inicialización)
4. [Estadísticas y datos derivados](#4-estadísticas-y-datos-derivados)
5. [Sistema de crecimiento por nivel](#5-sistema-de-crecimiento-por-nivel)
6. [Sistema de XP y niveles](#6-sistema-de-xp-y-niveles)
7. [Arquetipos](#7-arquetipos)
8. [Modificadores (A/O y buffs)](#8-modificadores-ao-y-buffs)
9. [Tiradas porcentuales](#9-tiradas-porcentuales)
10. [Subida de nivel (LevelUp)](#10-subida-de-nivel-levelup)
11. [Items](#11-items)
12. [Settings del sistema](#12-settings-del-sistema)
13. [Logging](#13-logging)
14. [Convenciones de código](#14-convenciones-de-código)

---

## 1. Visión general

**Isekai System** es un sistema de juego de rol de fantasía ambientado en un mundo de "isekai" (otro mundo). Los personajes tienen estadísticas físicas, mentales y especiales que crecen con el nivel, y el sistema calcula automáticamente valores derivados, arquetipos y recursos.

---

## 2. Estructura del proyecto

```
isekai-system/
├── system.json                   # Manifiesto de Foundry
├── lang/
│   ├── es.json                   # Traducciones español
│   └── en.json                   # Traducciones inglés
├── styles/
│   └── isekai-system-v2.css      # Estilos (temas scifi, modern, fantasy)
├── templates/                    # Plantillas Handlebars
│   ├── actor-sheet-pf2e.hbs      # Ficha de personaje (activa)
│   ├── actor-sheet.hbs           # Ficha legacy
│   ├── level-up.hbs              # Diálogo de subida de nivel
│   ├── special-stats-settings.hbs
│   ├── special-stats-config.hbs
│   ├── add-placeholders.hbs
│   ├── chat/
│   │   └── percent-roll.hbs      # Mensaje de tirada al chat
│   └── item/
│       ├── item-sheet-generic.hbs
│       └── weapon-sheet.hbs
└── module/
    ├── isekai.mjs                # Punto de entrada — registra hooks
    ├── config/
    │   └── index.js              # Constantes globales del sistema
    ├── data/
    │   ├── stats.js              # Definición de stats (físicas/mentales/especiales)
    │   └── item-defaults.js      # Valores por defecto al crear items
    ├── documents/
    │   ├── actor/actor.js        # IsekaiActor — datos derivados
    │   └── item/item.js          # IsekaiItem — datos derivados, armas
    ├── apps/
    │   ├── level-up.js           # LevelUpDialogV2
    │   └── special-stats-settings.js
    ├── hooks/                    # Handlers de hooks de Foundry
    │   ├── index.js
    │   ├── actor.js
    │   ├── chat.js
    │   └── item.js
    ├── rolls/
    │   └── percent-roll.js       # Lógica de tiradas porcentuales
    ├── rules/
    │   ├── growth.js             # Crecimiento universal y decadeSpike
    │   ├── modifiers.js          # Colectores de A/O bonuses y buffs
    │   ├── rule-engine.js        # Motor de reglas (activos, stacking)
    │   └── weapon-branches.js    # Branches y stats de escalado de armas
    ├── settings/
    │   └── settings.js           # Registro de settings de Foundry
    └── utils/
        ├── archetypes.js         # Cálculo de arquetipos y título especialista
        ├── dragdrop.js           # Helper de drag & drop v13
        ├── logger.js             # Sistema de logging con niveles
        ├── math.js               # avg, clampInt
        ├── rolls.js              # buildRollMessageHTML, statKeyFromPath
        ├── stats-catalog.js      # getAllStatKeys, statKeyToPath
        └── xp.js                 # totalXPForLevel, levelFromTotalXP
```

---

## 3. Flujo de inicialización

```
Foundry init
  └─ isekai.mjs (hook "init")
       ├─ registerSettings()          → settings/settings.js
       ├─ Actors.registerSheet(...)   → IsekaiActorSheetV2
       ├─ Items.registerSheet(...)    → hojas de item por tipo
       └─ CONFIG.Actor.documentClass  = IsekaiActor
          CONFIG.Item.documentClass   = IsekaiItem

Foundry ready
  └─ hook "ready" (en settings.js)
       └─ normalizeSpecialStatsConfig() si GM
```

---

## 4. Estadísticas y datos derivados

### 4.1 Grupos de estadísticas

| Grupo     | Stats                                                    | Recurso derivado                          |
|-----------|----------------------------------------------------------|-------------------------------------------|
| Físico    | Fuerza, Agilidad, Constitución, Fortaleza, Destreza      | HP (Constitución), Aguante ((Agi+Sab)/2)  |
| Mental    | Inteligencia, Sabiduría, Voluntad, Conocimiento, Carisma | Magia (Voluntad)                          |
| Especial  | Configurable por el GM (ej. Fuego, Necromancia…)         | —                                         |

### 4.2 Fórmula de stat total

Cada stat sigue este pipeline en `actor.prepareDerivedData()`:

```
base           → valor editado manualmente (entero, mínimo 0)
raw            = base + universalGrowth(nivel) + decadeSpike(nivel) + bonusFor(key)
pre            = raw + ao.flat[key]       ← bonus flat de Ancestría/Origen
total          = floor(pre × (1 + ao.growthPct[key]/100) × (1 + buffPct[key]/100))
total_clamped  = max(1, total)
```

`bonusFor(key)` suma los bonuses de `system.bonuses.priority` y `system.bonuses.impulses`.

### 4.3 Stats primarias (para arquetipos)

```
stats.primary.fisico   = avg(Object.values(stats.physical))
stats.primary.mental   = avg(Object.values(stats.mental))
stats.primary.especial = avg(statsEspecialesActivas)
```

### 4.4 Recursos

| Recurso  | Máximo                           |
|----------|----------------------------------|
| HP       | `max(1, floor(Constitución))`   |
| Magia    | `max(1, floor(Voluntad))`        |
| Aguante  | `max(1, trunc((Agi + Sab) / 2))`|

El `value` de cada recurso se clampea a `[0, max]` en cada recálculo. Si el value es null/NaN se resetea al máximo.

---

## 5. Sistema de crecimiento por nivel

Definido en `rules/growth.js`. Hay dos componentes que se **suman**:

### 5.1 Crecimiento universal (`universalGrowth`)

Puntos acumulados que se añaden a **todas** las stats en función del nivel:

| Rango de niveles | Puntos/nivel |
|------------------|-------------|
| 2 – 9            | 1           |
| 10 – 49          | 2           |
| 50 – 99          | 3           |
| 100 – 199        | 5           |
| 200 – 299        | 7           |
| 300 – 499        | 10          |
| 500 – 699        | 14          |
| 700 – 899        | 19          |
| 900 – 1000       | 25          |

### 5.2 Picos de década (`decadeSpike`)

Bonus fijo acumulado al superar ciertos hitos:

| Umbral | Bonus |
|--------|-------|
| ≥  20  |    50 |
| ≥  50  |   150 |
| ≥ 100  |   400 |
| ≥ 300  | 1.200 |
| ≥ 600  | 3.000 |
| ≥ 900  | 6.000 |

### 5.3 `radarMax`

Usado para normalizar los gráficos de radar en la ficha:

```js
radarMax = max(20, 1 + universalGrowth(lvl) + decadeSpike(lvl) + max(0, (lvl - 1) * 2))
```

---

## 6. Sistema de XP y niveles

Definido en `utils/xp.js`.

**XP necesaria para nivel L:**
```
totalXPForLevel(L) = floor(100 × (L-1) × L × (L+1) / 6)
```

Algunos valores de referencia:

| Nivel | XP total |
|-------|----------|
| 1     | 0        |
| 2     | 200      |
| 5     | 2.000    |
| 10    | 16.500   |
| 20    | 133.400  |
| 50    | 2.082.500|

**Nivel desde XP:** búsqueda binaria sobre el rango 1–1000, O(log n).

### Estado de level-up

El sistema detecta niveles pendientes comparando el XP acumulado con el `targetLevel` almacenado en `system.levelUp`:

```js
system.levelUp = {
  pending:      boolean,   // hay niveles sin aplicar
  targetLevel:  number,    // nivel objetivo actual
  remaining:    number,    // niveles por aplicar aún
  picks:        number,    // puntos por nivel (default: 5)
  counts:       object,    // asignaciones en curso { "physical.fuerza": 3 }
  priorityKey:  string|null,
  impulse:      { value, statKey } | null,
  impulseOffer: number     // valor de decadeSpike para el siguiente hito
}
```

---

## 7. Arquetipos

Calculados en `utils/archetypes.js` a partir de las stats base del personaje.

**Lógica:**
1. Se fusionan las stats físicas, mentales y especiales en un solo mapa y se ordenan de mayor a menor.
2. Si `min/max ≥ 0.85` → **All-Rounder** (estadísticas equilibradas).
3. En caso contrario: el arquetipo primario corresponde a la stat más alta; los dos secundarios a la 2ª y 3ª.

**Tabla de arquetipos estándar:**

| Stat              | Arquetipo        |
|-------------------|------------------|
| Fuerza            | Bárbaro          |
| Constitución      | Juggernaut       |
| Fortaleza         | Paladín          |
| Agilidad          | Explorador       |
| Destreza          | Pícaro           |
| Inteligencia      | Sabio            |
| Sabiduría         | Oráculo          |
| Voluntad          | Místico          |
| Conocimiento      | Erudito          |
| Carisma           | Bardo            |
| Manipulación Energía | Mago          |
| Control Elemental | Elementalista    |
| Alteración Realidad | Hechicero      |
| Invocación        | Invocador        |
| Sanación          | Clérigo          |
| Necromancia       | Nigromante       |
| Psiónica          | Psíquico         |

Las stats especiales personalizadas muestran "Especialista en {nombre}" salvo que tengan un `title` configurado en el GM panel.

---

## 8. Modificadores (A/O y buffs)

### 8.1 Bonuses de Ancestría/Origen

Definidos en `rules/modifiers.js`. Se leen de los items de tipo `ancestry` u `origin` equipados al actor.

Soporta dos formatos en `item.system`:
- **Nuevo:** `modifiers.flat` / `modifiers.percent`
- **Legacy:** `statBonuses.flat` / `statBonuses.growthPct`

Ambos formatos aceptan tanto objetos planos (`{ "physical.fuerza": 5 }`) como anidados (`{ physical: { fuerza: 5 } }`).

### 8.2 Buffs (efectos activos)

Los buffs se almacenan en `actor.system.buffs[]`. Cada buff tiene:

```js
{
  enabled: boolean,
  slug:    string,      // identificador único preferido
  originUuid: string,   // fallback si no hay slug
  percent: { "physical.fuerza": 10, ... }
}
```

Si dos buffs tienen el mismo `slug` u `originUuid`, **solo se aplica el más fuerte** (mayor suma de |percent|), evitando stacking involuntario.

### 8.3 Rule Engine (`rules/rule-engine.js`)

Motor genérico para items/efectos con `rules[]`. Evalúa si un documento está "activo" y gestiona el stacking por `slug`, `originUUID` o `uuid`. Pendiente de conectar al pipeline principal.

---

## 9. Tiradas porcentuales

### Mecanismo

1. Se lanza **1d100**.
2. Se suman los bonuses de porcentaje (`getPercentRollBonuses`).
3. `final = trunc(statValue × totalPct / 100)`

```
d100 = 73, bonusPct = 10 → totalPct = 83
stat = Fuerza = 150
final = trunc(150 × 83/100) = trunc(124.5) = 124
```

### Escalas de rango (statRank)

La ficha muestra la letra de rango de cada stat total:

| Rango | Hasta | Rango | Hasta  |
|-------|-------|-------|--------|
| F     | 10    | B++   | 160    |
| F+    | 20    | A     | 170    |
| …     | …     | …     | …      |
| S++   | 210   | SSR++ | 30.000 |
| SS    | 2.200 | ∞     | >30.000|

### Iniciativa

```
init = trunc((Destreza + Agilidad + Sabiduría + Conocimiento) / 4)
tirada = 1d100 + init
```

---

## 10. Subida de nivel (LevelUp)

Implementado en `apps/level-up.js` (`LevelUpDialogV2`).

### Flujo

1. El actor acumula XP → `levelFromTotalXP()` detecta que el nivel derivado supera `targetLevel`.
2. Se marca `system.levelUp.pending = true`.
3. El jugador abre la ficha → botón "Subir de nivel" → abre `LevelUpDialogV2`.
4. El jugador distribuye `picksPerLevel × pendingLevels` puntos entre stats.
5. Opciones adicionales:
   - **Prioridad:** una stat recibe +1 por cada nivel aplicado (requiere tener puntos asignados en ella).
   - **Impulso:** bonus único disponible en hitos de múltiplos de 20 (usa `decadeSpike`).
   - **Bonus por uso:** stats usadas ≥3 veces reciben +1; ≥10 veces +2.
6. Al confirmar, se actualizan las stats, se consume `remaining`, se limpia el estado de level-up y se borra el registro de uso.
7. Si quedan niveles pendientes y no hay combate activo, el diálogo se reabre automáticamente.

### Modo bulk

Si hay varios niveles pendientes, el diálogo muestra inputs numéricos en lugar de botones (cuando `maxTotal > 20`), permitiendo asignar múltiples puntos a la vez por stat.

---

## 11. Items

### Tipos de item

| Tipo        | Descripción                        |
|-------------|------------------------------------|
| `ancestry`  | Ancestría del personaje (máx. 1)   |
| `origin`    | Origen del personaje (máx. 1)      |
| `race`      | Legacy de `ancestry`               |
| `background`| Legacy de `origin`                 |
| `weapon`    | Arma con modos de ataque           |
| `spell`     | Hechizo                            |
| `skill`     | Habilidad                          |
| `equipment` | Equipamiento genérico              |
| `ammo`      | Munición                           |
| `buff`      | Efecto de buff                     |

Solo puede haber **un** item de tipo ancestría y **uno** de tipo origen por actor. Al arrastrar uno nuevo se reemplaza el existente.

### Armas (`weapon`)

Los armas tienen:
- **Branches:** `["physical"]` por defecto; si tiene `"magic"`, permite escalado mental/especial.
- **Stat de escalado:** validada contra las branches permitidas; si no es válida se resetea a `physical.fuerza`.
- **Modos de ataque:** array de modos con `kind` (melee/ranged), `element`, `accuracyPct`, `powerPct`, `penFlat`, `penPct`.

---

## 12. Settings del sistema

| Setting              | Scope  | Tipo    | Default   | Descripción                               |
|----------------------|--------|---------|-----------|-------------------------------------------|
| `debug`              | client | Boolean | false     | Activa logs de nivel debug en consola     |
| `logLevel`           | client | String  | "info"    | Nivel de log: error/warn/info/debug       |
| `specialStatsConfig` | world  | Object  | (defaults)| Config de stats especiales activas        |
| `showMagic`          | world  | Boolean | true      | Muestra recurso de magia en la ficha      |
| `showStamina`        | world  | Boolean | true      | Muestra recurso de aguante en la ficha    |
| `energyLabel`        | world  | String  | "Magic"   | Etiqueta de la energía (localización)     |
| `sheetTheme`         | world  | String  | "Sci-Fi"  | Tema visual: Sci-Fi / Modern / Fantasy    |

### specialStatsConfig

```js
{
  "manipulacionEnergia": {
    name:   "Manipulación de Energía",
    active: true,
    color:  "hsl(220 80% 60%)",
    title:  "Especialista en Manipulación de Energía"
  },
  // … una entrada por stat especial
}
```

Al arrancar, el GM ejecuta automáticamente una migración que convierte formatos legacy (array, claves numéricas) al formato objeto canonico.

---

## 13. Logging

El sistema usa un logger centralizado en `utils/logger.js`.

### Uso

```js
import { LOG } from "../../utils/logger.js";

LOG.debug("mensaje", { datos: "opcionales" });
LOG.info("mensaje");
LOG.warn("advertencia");
LOG.error("error", errorObject);

// Grupos colapsados (solo en debug)
LOG.groupCollapsed("Sección");
// … código
LOG.groupEnd();
```

### Prefijo visual

Todos los mensajes muestran `%cIsekai` en color violeta (#7a5cff) para identificarlos fácilmente en la consola del navegador.

### Niveles

| Nivel   | Cuándo se muestra                              |
|---------|------------------------------------------------|
| `error` | Siempre                                        |
| `warn`  | Si `logLevel ≥ warn`                           |
| `info`  | Si `logLevel ≥ info` (default)                 |
| `debug` | Solo si `logLevel = debug` **y** `debug = true`|

Para activar el modo debug en partida: Configuración del juego → Sistema → activar "Debug" y establecer nivel "debug".

---

## 14. Convenciones de código

- **Imports:** siempre rutas relativas con extensión `.js`.
- **Null safety:** se usa `??` y optional chaining `?.` en lugar de comprobaciones verbosas.
- **Números:** `Number(x) || 0` para conversiones seguras; `Math.floor` para enteros; `Math.trunc` para truncar hacia cero.
- **Stat keys:** formato `"grupo.statId"` — ej. `"physical.fuerza"`, `"mental.voluntad"`, `"special.sanacion"`.
- **Paths de actor:** formato `"system.stats.physical.fuerza"` para base, `"system.statsTotal.physical.fuerza"` para totales.
- **Logging:** usar `LOG.debug` para valores intermedios de cálculo, `LOG.info` para acciones del usuario, `LOG.warn/error` para condiciones anómalas.
- **No `console.log` directo:** siempre a través de `LOG` para respetar el nivel de log configurado.

---

---

## 15. Hojas de item (sheets)

### 15.1 IsekaiItemSheetV2 (`sheets/item/item-sheet.js`)

Hoja genérica para tipos sin hoja especializada (equipment, consumable, spell, skill, etc.). Lee todos los elementos del DOM con `[data-path]` y los persiste automáticamente al cambio. Convierte el valor según el tipo del input (`checkbox → boolean`, `number → Number`, resto `string`).

### 15.2 IsekaiAncestrySheetV2 (`sheets/item/ancestry-sheet.js`)

Hoja para ítems de tipo `ancestry` (o `race` legacy). Muestra los modificadores flat y % que la ancestría aplica a las stats físicas y mentales. Internamente usa los helpers compartidos de `_modifier-sheet-helpers.js`.

### 15.3 IsekaiOriginSheetV2 (`sheets/item/origin-sheet.js`)

Hoja para ítems de tipo `origin` (o `background` legacy). Idéntica en estructura a la ancestría pero incluye también las stats especiales activas en el mundo.

### 15.4 Helpers compartidos de modificadores (`sheets/item/_modifier-sheet-helpers.js`)

Módulo interno (prefijo `_`) que evita duplicación entre `ancestry-sheet.js` y `origin-sheet.js`. Exporta:

| Función | Descripción |
|---------|-------------|
| `getActiveSpecialRows()` | Lista de stats especiales activas según la config del mundo. |
| `readModifierValue(root, key)` | Lee un valor de un objeto de modificadores en formato plano o anidado. |
| `buildModifierRows(flatRoot, percentRoot)` | Construye las filas de modificadores para physical, mental y special. |
| `attachModifierListeners(element, sheet)` | Registra listeners de cambio y borrado de modificadores en el DOM. |
| `attachTextFieldListeners(element, sheet)` | Registra listeners para inputs de texto con `[data-path]`. |

### 15.5 IsekaiWeaponSheetV2 (`sheets/item/weapon-sheet.js`)

Hoja completa para armas con:

- **Modos de ataque:** se añaden/eliminan dinámicamente. Cada modo tiene `kind` (melee/ranged), `stat` de escalado, `damageType`, `powerPct`, `flatPower`, `reloadActions` y `tags`.
- **Opciones de stat de escalado:** filtradas dinámicamente según las branches del arma (si tiene `"magic"` incluye mentales y especiales).
- **Reglas del rule-engine:** CRUD de objetos `{ type, selector, value, stackKey, enabled }` que el motor de reglas evalúa al preparar el actor.
- **Debounce:** los cambios en inputs se consolidan con un timer de 120ms para reducir llamadas a `document.update`.
- **`_commitAllDataPathFields()`:** antes de mutar arrays (añadir/eliminar modos o reglas), hace commit de todos los inputs sin blur para evitar pérdida de datos.

---

## 16. SpecialStatsSettingsV2 (`apps/special-stats-settings.js`)

Panel de GM para gestionar el catálogo global de stats especiales. Accesible desde la ficha del personaje (botón ⚙️ en la sección de stats especiales).

**Operaciones disponibles:**
- **Añadir:** crea una fila nueva con id, nombre, color (color picker) y título de especialista.
- **Desactivar/Activar:** checkbox por fila. Las stats inactivas no aparecen en las fichas.
- **Eliminar:** solo para stats personalizadas; las predeterminadas no se pueden borrar.
- **Guardar:** persiste todo en `specialStatsConfig` y re-renderiza todas las fichas de actores.

Los IDs se sanitizan eliminando espacios al guardar. Los defaults siempre se reinyectan aunque falten en el formulario.

---

## 17. Hooks de chat (`hooks/chat.js`)

Registra el hook `renderChatMessageHTML` (Foundry v13+). Al renderizar un mensaje de chat:
1. Busca elementos `[data-isekai-roll="percent"]` con `[data-roll-payload="..."]`.
2. Al hacer click, parsea el JSON del payload y abre un `Dialog` con el detalle de la tirada.

El payload esperado:
```js
{
  statLabel: "Fuerza",
  statValue: 150,
  d100:      73,
  bonusPct:  10,
  finalPct:  83,
  result:    124
}
```

---

## 18. Flujo completo de una tirada

```
Usuario hace click en "Tirada: Fuerza" en la ficha
  └─ IsekaiActorSheetV2.#onRollStat
       ├─ Lee statValue desde dataset.value o actor path
       ├─ Registra un uso en actor flags (SYS_ID, "usage")
       ├─ new Roll("1d100").evaluate() → d100
       ├─ getPercentRollBonuses(actor, statKey) → bonuses
       ├─ buildPercentRollData({ actor, statKey, statLabel, base, d100, bonuses })
       │    └─ final = trunc(base × (d100 + bonusPct) / 100)
       └─ postPercentRollToChat({ actor, label, rollData })
            ├─ renderTemplate("chat/percent-roll.hbs", ...)
            └─ ChatMessage.create({ flags: { [SYS_ID]: { percentRoll: rollData } } })

Foundry renderiza el mensaje
  └─ renderChatMessageHTML hook (hooks/chat.js)
       └─ Click en card → Dialog con desglose de la tirada
```

---

## 19. Flujo completo de subida de nivel

```
Actor acumula XP
  └─ prepareDerivedData() → lvl = levelFromTotalXP(xp)
       └─ Si lvl > targetLevel → levelUp.pending = true

Usuario abre la ficha
  └─ Botón "Subir de nivel" (acción openLevelUp)
       ├─ Comprueba levelUp.pending === true
       ├─ Comprueba !isInCombat()
       └─ new LevelUpDialogV2(actor).render(true)

Diálogo abierto
  ├─ Distribuye N × picksPerLevel puntos entre stats
  ├─ (Opcional) marca stat prioritaria (+1 por nivel)
  ├─ (Opcional) asigna impulso de década
  └─ Confirmar → #onConfirm
       ├─ Aplica stats + bonus por uso
       ├─ Aplica prioridad e impulso
       ├─ Decrementa levelUp.remaining
       ├─ Limpia flags de uso
       └─ Si remaining > 0 y !combat → reabre el diálogo
```

---

## 20. Sistema de logging (`utils/logger.js`)

### Arquitectura

El logger es el único módulo sin dependencias propias del sistema. Puede llamarse desde cualquier punto, incluso antes de que las settings estén registradas (devuelve `info` como nivel por defecto si `game.settings` no está listo aún).

### Niveles y visibilidad

| Método            | Nivel | `console.*` usado | Condición para mostrarse                            |
|-------------------|-------|-------------------|-----------------------------------------------------|
| `LOG.error(...)`  | 0     | `console.error`   | Siempre                                             |
| `LOG.warn(...)`   | 1     | `console.warn`    | `logLevel ≥ warn`                                   |
| `LOG.info(...)`   | 2     | `console.info`    | `logLevel ≥ info` *(default)*                       |
| `LOG.debug(...)`  | 3     | `console.debug`   | `logLevel = debug` **y** `debug = true` en cliente  |

### `createLogger(scope)` vs `LOG`

`createLogger` permite crear loggers con un prefijo de scope diferente, útil si en el futuro se quiere distinguir por módulo:

```js
// Logger específico para una clase (scope personalizado)
const log = createLogger("WeaponSheet");
log.debug("_prepareContext", { name: item.name });
// → %cWeaponSheet [violeta] "_prepareContext" { name: "..." }

// Logger global (el más común)
import { LOG } from "./logger.js";
LOG.info("Sistema iniciado");
// → %cIsekai [violeta] "Sistema iniciado"
```

### Grupos de debug

```js
LOG.groupCollapsed("Drop", data);  // abre grupo colapsado
// ... lógica ...
LOG.groupEnd();                    // cierra el grupo
```

Los grupos solo se muestran con `logLevel=debug` y `debug=true`. Son útiles para inspeccionar flujos con varios pasos (drag & drop, evaluación de reglas) sin contaminar la consola en producción.

---

## 21. Hoja de personaje (`sheets/actor/actor-sheet.js`)

### Visión general

`IsekaiActorSheetV2` extiende `DocumentSheetV2` con la mixin `HandlebarsApplicationMixin`. Usa el template `actor-sheet-pf2e.hbs` y es la única hoja registrada para el tipo `character`.

### `_prepareContext()` — datos enviados al template

| Clave               | Contenido                                                          |
|---------------------|--------------------------------------------------------------------|
| `actor`             | `{ id, name, img, type }`                                          |
| `system`            | `actor.system` completo                                            |
| `raceItem`          | Item de ancestría encontrado (o `null`)                            |
| `backgroundItem`    | Item de origen encontrado (o `null`)                               |
| `lists.physical`    | Stats físicas con `{ id, name, color, base, value, path }`         |
| `lists.mental`      | Stats mentales con el mismo shape                                  |
| `lists.special`     | Stats especiales activas                                           |
| `lists.inventory`   | `{ weapons, armor, equipment, consumables, treasure, containers }` |
| `lists.specials`    | `{ spells, skills }`                                               |
| `lists.passives`    | Array de items de tipo `passive`                                   |
| `lists.effects`     | Array de items de tipo `effect`                                    |
| `resources`         | `{ hp, magic, stamina }` con `{ value, max }`                      |
| `radar`             | `{ physical, mental, special }` — HTML strings SVG                 |
| `derived`           | `{ xpThreshold, initiative, armorDR }`                             |
| `ui`                | `{ showLevelUpButton, inCombat, levelUpPending }`                  |
| `settings`          | `{ sheetTheme, showMagic, showStamina, energyLabel }`              |

### `buildRadarSVG` — radar interactivo

Genera un SVG con anillos de referencia, ejes coloreados, un polígono animado y **hotspots triangulares** sobre cada eje que capturan el hover del ratón. Al hacer hover se muestra un HUD con nombre, valor y rango de la stat. Los eventos se manejan por delegación sobre el SVG completo, no por listener individual en cada hotspot.

### Sistema de tabs

DocumentSheetV2 no conecta tabs automáticamente. El sistema implementa:

- **`_initTabs(root)`** — tabs primarias: `nav.tabs[data-group="primary"]` ↔ `.tab[data-group="primary"]`. Persiste la pestaña activa en `this._isekaiActiveTab`.
- **`_initTabsByGroup(root, group)`** — sub-tabs genéricas para cualquier grupo.

### Acciones

| Acción              | Descripción                                              |
|---------------------|----------------------------------------------------------|
| `rollStat`          | Tira 1d100, aplica bonuses y publica en chat             |
| `openSpecialConfig` | Abre `SpecialStatsSettingsV2`                            |
| `openLevelUp`       | Abre `LevelUpDialogV2` (si pendiente y sin combate)      |
| `openItem`          | Abre la hoja del item embebido indicado                  |
| `deleteEmbeddedItem`| Elimina un item embebido del actor                       |
| `rollInitiative`    | Tira 1d100+init y publica en chat                        |

---

## 22. Hooks del sistema (`hooks/`)

### 22.1 `hooks/index.js` — punto de entrada

`registerHooks()` debe llamarse desde `isekai.mjs` durante el hook `init`. Delega en los registradores específicos y además registra directamente el hook `renderChatMessage` para el inspector de tiradas.

```
registerHooks()
  ├─ registerActorHooks()
  ├─ registerItemHooks()
  ├─ registerChatHooks()       ← renderChatMessageHTML (roll cards simples)
  ├─ registerTitleHooks()      ← ⚠ inactivo (ver 22.3)
  └─ renderChatMessage inline  ← inspector completo de tiradas porcentuales
```

**Dos hooks de chat, propósitos distintos:**

| Hook                    | Archivo    | Propósito                                                                                                                  |
|-------------------------|------------|----------------------------------------------------------------------------------------------------------------------------|
| `renderChatMessageHTML` | `chat.js`  | Engancha el click en cards `[data-isekai-roll="percent"]` para mostrar un Dialog simple                                    |
| `renderChatMessage`     | `index.js` | Añade clase CSS y un click listener que abre `openPercentRollDetails` con el desglose completo desde los flags del mensaje |

`openPercentRollDetails` lee `flags.[SYS_ID].percentRoll` del ChatMessage y construye un Dialog con la fórmula completa: base stat, d100, breakdown de bonuses por fuente, total y resultado final.

### 22.2 `hooks/actor.js`

**`preCreateActor`** — Inyecta el estado inicial del actor al crearlo, respetando cualquier dato que ya venga en `data.system` (mergeObject con `inplace: false`):

```js
{
  level: 1, xp: 0, xpThreshold: 0,
  arquetipo: "", arquetiposSec: [],
  levelUp: { pending: false, targetLevel: 1, picks: 5 },
  stats: { primary: {…}, physical: {…}, mental: {…}, special: {} }
}
```

**`preUpdateActor`** — Detecta cambios en `system.xp` y calcula la cola de level-up:

```
xp cambia
  └─ levelFromTotalXP(nextXP) → newLevel
       ├─ Actualiza system.level y system.xpThreshold siempre
       └─ Si newLevel > oldLevel:
            ├─ Respeta currentTarget (no salta niveles en cola existente)
            ├─ remaining = newLevel - (currentTarget - 1)
            ├─ offer = isImpulseLevel(currentTarget) ? decadeSpike(currentTarget) : 0
            └─ Escribe levelUp.{ pending, remaining, targetLevel, picks, impulseOffer }
```

**`createItem` / `deleteItem`** — Re-renderizan la ficha del actor propietario cuando cambia su colección de items embebidos. Filtran por `documentName === "Actor"` y `type === "character"`.

---

## 23. TODOs

### Pendientes anotados en el código

| Archivo            |                                         Pendiente                                                   |
|--------------------|-----------------------------------------------------------------------------------------------------|
| `item-defaults.js` | Todos los tipos tienen campos `"TODO"` (spell: tradition, dc, attack)                               |
| `percent-roll.js`  | `getPercentRollBonuses` aún no recorre items/effects directamente (usa el cache del rule-engine)    |
| `actor.js`         | `maxFromImpulses = 0` — pendiente conectar la tabla real de impulsos a `radarMax`                   |
| `rule-engine.js`   | Solo procesa reglas `key === "PercentRollBonus"` — extensible a otros tipos                         |
| `weapon-sheet.js`  | `ctx.statKeys` tiene una lista hardcoded en desuso — ya usa `getAllStatKeys()` vía `scalingOptions` |

### Extensión del rule-engine

Para añadir un nuevo tipo de regla:
1. Definir `rule.key = "NuevoTipoRegla"` en los items.
2. Añadir un bloque `if (rule.key !== "NuevoTipoRegla") continue;` en `evaluateActorRules`.
3. Guardar el resultado en `out.nuevoTipo[selector]` y consumirlo en `prepareDerivedData`.

### Añadir una nueva stat especial global

1. Añadirla a `DEFAULT_SPECIAL_STATS` en `data/stats.js`.
2. Opcionalmente mapear su arquetipo en `ARCHETYPE_NAMES` en `utils/archetypes.js`.
3. La migración automática del hook `ready` en `settings.js` la añadirá a la config de mundos existentes.

---
