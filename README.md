# Isekai Rank System — Documentación Técnica

Sistema de rol para **Foundry VTT v13** basado en el esquema de rangos F → SSR de los isekai, con un motor de combate d100% diseñado para escalar hasta el nivel 1000.

> **Versión del sistema:** 1.0.5
> **Compatibilidad Foundry:** v13 mínimo · v13 verificado

---

## Índice

- [1. Estructura del sistema](#1-estructura-del-sistema)
- [2. Categorías y Sub-estadísticas](#2-categorías-y-sub-estadísticas)
- [3. Estadísticas Principales](#3-estadísticas-principales)
- [4. Fórmula de cálculo de estadísticas](#4-fórmula-de-cálculo-de-estadísticas)
- [5. Sistema de Rangos](#5-sistema-de-rangos)
- [6. Arquetipos (clases automáticas)](#6-arquetipos-clases-automáticas)
- [7. Motor de tiradas d100%](#7-motor-de-tiradas-d100)
- [8. Motor de combate](#8-motor-de-combate)
- [9. Sistema de buffs y Rule Elements](#9-sistema-de-buffs-y-rule-elements)
- [10. Equipamiento](#10-equipamiento)
- [11. Escalado de items por nivel y rareza](#11-escalado-de-items-por-nivel-y-rareza)
- [12. Consumibles](#12-consumibles)
- [13. Tags de items (Tagify)](#13-tags-de-items-tagify)
- [14. Raza, Origen y Trasfondo](#14-raza-origen-y-trasfondo)
- [15. Accesorio](#15-accesorio)
- [16. Modificadores de edad y longevidad](#16-modificadores-de-edad-y-longevidad)
- [17. IWR (Inmunidades, Resistencias, Debilidades)](#17-iwr-inmunidades-resistencias-debilidades)
- [18. Elementos y Perfiles](#18-elementos-y-perfiles)
- [19. Energía Mística](#19-energía-mística)
- [20. Sistema de XP y Subida de Nivel](#20-sistema-de-xp-y-subida-de-nivel)
- [21. Editor de NPC](#21-editor-de-npc)
- [22. Sistema de idiomas del mundo](#22-sistema-de-idiomas-del-mundo)
- [23. Sistema de bulk y moneda](#23-sistema-de-bulk-y-moneda)
- [24. Macros disponibles](#24-macros-disponibles)
- [25. Configuración del sistema](#25-configuración-del-sistema)
- [26. Instalación](#26-instalación)
- [27. API pública](#27-api-pública)

---

## 1. Estructura del sistema

```
isekai-rank-system/
├── system.json                   ← Manifiesto de Foundry VTT
├── template.json                 ← Definición de todos los campos de datos (Actor/Item)
├── isekai.js                     ← Punto de entrada: hooks init/ready, helpers Handlebars, API pública
├── styles/
│   └── isekai.css                ← Hoja de estilos tema dark-fantasy
├── templates/
│   ├── actor/
│   │   ├── character-sheet.hbs   ← Hoja de personaje jugador (PC)
│   │   └── npc-sheet.hbs         ← Hoja de NPC
│   ├── item/
│   │   └── item-sheet.hbs        ← Hoja de objetos unificada (todos los tipos)
│   └── apps/
│       ├── level-up.hbs          ← Diálogo de subida de nivel (PC)
│       ├── npc-editor.hbs        ← Editor de NPC con presets
│       └── idiomas-config.hbs    ← Configurador de idiomas del mundo
├── lang/
│   └── es.json                   ← Localización completa en español
├── lib/
│   ├── tagify.min.js             ← Librería de tags (cargada via system.json)
│   └── tagify.min.css
└── module/
    ├── config.js                 ← Motor completo: constantes, fórmulas, combate, utilidades
    ├── actor.js                  ← Clases IsekaiActor e IsekaiItem
    ├── actor-sheet.js            ← Hoja de actor (ApplicationV2)
    ├── item-sheet.js             ← Hoja de item (ApplicationV2)
    ├── macros.js                 ← Macros de ejemplo para Foundry
    └── tags.js                   ← TAG_PROFILES (re-exporta desde config.js)
```

---

## 2. Categorías y Sub-estadísticas

Las **18 sub-estadísticas** se dividen en 3 categorías. Cada una tiene clave interna, etiqueta, abreviatura y color de UI.

### ⚔️ Físico

| Clave          | Etiqueta      | Abrev. | Color    | Rol principal                                               |
|----------------|---------------|--------|----------|-------------------------------------------------------------|
| `fuerza`       | Fuerza        | FUE    | #e87040  | Daño bruto melee, carga, empuje físico                      |
| `destreza`     | Destreza      | DES    | #e87040  | Precisión de ataque físico, trabajos de mano                |
| `velocidad`    | Velocidad     | VEL    | #e87040  | Iniciativa, evasión base, huida                             |
| `reflejos`     | Reflejos      | REF    | #e87040  | Evasión instintiva, reacción a peligros inesperados         |
| `fortaleza`    | Fortaleza     | FOR    | #e87040  | DR base, resistencia física, HP máximo parcial              |
| `constitucion` | Constitución  | CON    | #e87040  | HP máximo (mayor peso), recuperación, resistencia a venenos |

### 🧠 Mental

| Clave          | Etiqueta      | Abrev. | Color    | Rol principal                                               |
|----------------|---------------|--------|----------|-------------------------------------------------------------|
| `inteligencia` | Inteligencia  | INT    | #5a8fd4  | Precisión mágica, análisis, fabricación                     |
| `sabiduria`    | Sabiduría     | SAB    | #5a8fd4  | Evasión táctica, percepción espiritual                      |
| `voluntad`     | Voluntad      | VOL    | #5a8fd4  | Guardia (resistir impactos), resistencia mental, DR mental  |
| `conocimiento` | Conocimiento  | CON2   | #5a8fd4  | Precisión técnica de combate, identificación                |
| `carisma`      | Carisma       | CAR    | #5a8fd4  | Negociación, liderazgo, intimidación, Suerte                |
| `percepcion`   | Percepción    | PER    | #5a8fd4  | Detección, iniciativa adicional, exploración                |

### ✨ Especial

| Clave          | Etiqueta        | Abrev. | Color    | Rol principal                                               |
|----------------|-----------------|--------|----------|-------------------------------------------------------------|
| `afinidad`     | Afinidad        | AFI    | #b565e0  | Potencia mágica base, sintonía elemental                    |
| `reserva`      | Reserva Mística | RSV    | #b565e0  | Pool de energía máxima (Magia), resistencia mágica          |
| `canalizacion` | Canalización    | CAN    | #b565e0  | Velocidad de lanzamiento, coste reducido de conjuros        |
| `dominio`      | Dominio         | DOM    | #b565e0  | Precisión de conjuros, control de área                      |
| `sensibilidad` | Sensibilidad    | SEN    | #b565e0  | Percepción mística, detección de peligros ocultos, Suerte   |
| `nexo`         | Nexo            | NEX    | #b565e0  | Vínculo con entidades, poder de sellos/contratos            |

---

## 3. Estadísticas Principales

Se calculan automáticamente desde las sub-estadísticas en `derivarEstadisticasPrincipales()`:

| Stat         | Fórmula                                                             | Uso principal                         |
|--------------|---------------------------------------------------------------------|---------------------------------------|
| **Ataque**   | `⌊FUE×0.55 + DES×0.35 + FOR×0.10⌋`                                  | Potencia ofensiva, daño general       |
| **Defensa**  | `⌊FOR×0.45 + CON×0.35 + VOL×0.20⌋`                                  | Resistencia al daño, tankeo           |
| **Agilidad** | `⌊VEL×0.40 + REF×0.40 + PER×0.20⌋`                                  | Evasión, iniciativa, movilidad        |
| **Magia**    | `⌊AFI×0.30 + RSV×0.25 + INT×0.25 + CAN×0.20⌋`                       | Potencia mística general              |
| **Suerte**   | `⌊SEN×0.30 + CAR×0.25 + PER×0.25 + NEX×0.20⌋`                       | Críticos, eventos fortuitos           |
| **DR**       | `⌊(FOR + VOL) / 2⌋ + armor_dr (de armadura equipada)`               | Reducción de daño recibido            |

### Scores de Combate

Determinan la probabilidad de golpear, esquivar y bloquear mediante la función logística:

| Score        | Fórmula                             | Uso en combate                          |
|--------------|-------------------------------------|-----------------------------------------|
| `accuracy`   | `DES + conocimiento`                | Precisión con armas físicas             |
| `acc_mag`    | `DOM + INT`                         | Precisión de conjuros                   |
| `evasion`    | `VEL + SAB`                         | Resistir esquiva                        |
| `guard`      | `FOR + VOL`                         | Resistir bloqueo                        |
| `iniciativa` | `VEL + PER + ⌊Suerte÷10⌋`           | Orden de acción en combate              |

El Origen puede añadir bonos planos a cualquiera de estos scores via `bonus_scores`.

### HP Máximo

```js
HP_max = max(1, ⌊ (FOR×5 + CON×8 + VOL×3) × (1 + nivel×0.05) × (1 + hp_bonus_pct/100) ⌋)
```

- La ancestría puede aportar `hp_bonus_pct` (porcentaje adicional sobre el resultado final).
- Escala lineal del 5% por nivel (un nivel 100 tiene ×6 HP base, nivel 1000 ×51).

---

## 4. Fórmula de cálculo de estadísticas

Cada sub-estadística sigue esta cadena de cálculo (`calcSubstat`):

```
flat    = base + ancFlat + oriFlat + traFlat + eqFlat
growthMult = (100 + ancPct + oriPct) / 100
buffMult   = (100 + Σbuffs_deduplicados) / 100
accMult    = (100 + accPct) / 100         ← accPct de accesorios
ageMult    = según franja de edad y longevidad de ancestría
value      = max(1, ⌊ flat × growthMult × buffMult × ageMult × accMult ⌋)
```

**Orden de fuentes (aditivo dentro de cada tipo):**

| Fuente                  | Campo               | Tipo      | Siempre activa |
|-------------------------|---------------------|-----------|----------------|
| Puntos de personaje     | `base`              | Flat      | ✓              |
| Ancestría               | `ancFlat` / `ancPct`| Flat / %  | ✓              |
| Origen                  | `oriFlat` / `oriPct`| Flat / %  | ✓              |
| Trasfondo               | `traFlat` / `traPct`| Flat / %  | ✓              |
| Equipamiento (RE)       | `eqFlat` / `eqPct`  | Flat / %  | Solo equipado  |
| Accesorios              | `accPct`            | % final   | Solo equipado  |
| Efectos/Buffs activos   | buffs deduplicados  | %         | Si activo      |
| Modificador de edad     | `ageMult`           | Mult.     | ✓              |

El Trasfondo también puede bonificar un único **stat principal** (Ataque, Defensa, etc.) con un valor plano adicional.

---

## 5. Sistema de Rangos

El **Poder de Combate (CP)** determina el rango automáticamente:

```js
CP = Ataque + Defensa + Agilidad + Magia + Suerte + ⌊HP/1000⌋ + DR×2
```

| Rango | CP mínimo   | Color    |
|-------|-------------|----------|
| F     | 0           | #6b7280  |
| E     | 100         | #22c55e  |
| D     | 500         | #3b82f6  |
| C     | 2.000       | #a855f7  |
| B     | 8.000       | #f97316  |
| A     | 25.000      | #ef4444  |
| S     | 80.000      | #eab308  |
| SS    | 250.000     | #fbbf24  |
| SSR   | 1.000.000   | #f472b6  |

Los umbrales son configurables en tiempo de juego desde **Configuración del Sistema → Umbrales de Rango** (JSON editable).

---

## 6. Arquetipos (clases automáticas)

El arquetipo se asigna automáticamente al calcular las estadísticas. Se evalúan en orden de prioridad:

| Arquetipo           | Icono | Condición de activación                                                 |
|---------------------|-------|-------------------------------------------------------------------------|
| Destructor          | ⚔️    | ATQ > DEF×2 y MAG < ATQ×0.3                                            |
| Bastión             | 🛡️    | DEF > ATQ×1.5 y DEF > AGI×1.5                                          |
| Fantasma            | 💨    | AGI > ATQ×1.5 y AGI > DEF×2                                            |
| Archimago           | 🔮    | MAG > ATQ×1.8 y MAG > DEF                                              |
| Embaucador          | 🎲    | SUE > ATQ×1.5 y SUE > MAG                                              |
| Espadachín Mágico   | ✨    | gap(ATQ, MAG) < 25% y ATQ > DEF                                        |
| Campeón             | 🏆    | ATQ > MAG×1.4 y gap(ATQ, DEF) < 35%                                    |
| Pícaro              | 🗡️    | AGI > DEF×1.3 y ATQ > DEF                                              |
| Paladín             | ⚡    | gap(ATQ, DEF) < 20% y MAG > AGI                                        |
| Oráculo             | 👁️    | SUE > DEF y MAG > ATQ×1.3                                              |
| All-Rounder         | ⭕    | Fallback — siempre se activa si ninguno anterior coincide               |

`gap(a, b) = |a − b| / max(a, b, 1)` — mide la diferencia relativa entre dos estadísticas.

---

## 7. Motor de tiradas d100%

Toda tirada de sub-estadística usa `rollStat(statKey)`:

1. Se lanza **1d100** en Foundry.
2. El resultado se normaliza como porcentaje del valor de la sub-estadística:
   `resultado = valor_substat × (d100 / 100)`
3. Se publican los detalles en el chat con toggle de detalles.

**Tiradas de habilidad** usan la misma lógica, con el stat definido por `hab_stat_key`.

---

## 8. Motor de combate

### Flujo de un ataque (`attackWith`)

```
1. Calcular effAccuracy del atacante
   → accuracy (física) o acc_mag (mágica)
   → + bonos de origen (oriBonusScores)
   → + Rule Elements AccuracyBonus
   → × accuracyMult de los tags del arma/habilidad (mergeTagProfiles)

2. Lanzar 1d100 para evasión y 1d100 para bloqueo

3. resolveOutcome(effAccuracy, evasion, guard, rollEvasion, rollBlock, elemProfile, tagProfile, blockMult)
   → Determina: dodge | graze | block | hit

4. Para cada elemento del arma (hab_elementos[]):
   a. Lanzar 1d100 de daño
   b. fullDamagePipeline(atacanteStats, powerPct×fracción, powerFlat×fracción, d100, ...)
      → calcDamageBase → dmgBase (stat × power% + flat) × (d100/100)
      → × powerMult de tags
      → × blockMult del outcome (0 dodge, 0.20 graze, mult block, 1.0 hit)
      → applyRichIWR (div → resistPct → weakPct → resistFlat → weakFlat)
   c. Acumular dmgTotal

5. mitigateDR(dmgTotal, DR_efectiva)
   → DR_efectiva = DR × (1 − penetracionPct/100)
   → daño_final  = max(⌊dmg×20%⌋, ⌊dmg²/(dmg+DR_efec)⌋)

6. Aplicar vampPct (tag vampiro): HP atacante += dmgFinal × vampPct/100

7. Publicar mensaje de chat y actualizar HP del objetivo
```

### Función logChance

```js
logChance(attackerScore, defenderScore, s=0.50, K=10) =
  clamp(0.05, 0.95,  0.5 + ln((atk+K)/(def+K)) × s)
```

- `K=10` suaviza la curva a valores bajos.
- `s=0.50` controla la pendiente (bloqueo usa `s=0.30` — curva más suave).
- Una ventaja ×10 en accuracy da ~91% de impacto; nunca alcanza el 100%.

### Resultados posibles

| Outcome | Condición                                                               | blockMult          |
|---------|-------------------------------------------------------------------------|--------------------|
| `dodge` | `rollEvasion/100 ≥ hitChance` y `margen > 0.12` (glanceMargin)         | ×0 (sin daño)      |
| `graze` | `rollEvasion/100 ≥ hitChance` y `margen ≤ 0.12`                        | ×0.20 (daño rozante) |
| `block` | Impacta pero `rollBlock/100 ≥ hitChanceBlock` y hay escudo              | ×blockMult_escudo  |
| `hit`   | Impacta y no bloquea                                                    | ×1.0 (daño completo) |

El margen de roce (0.12) es configurable en **Configuración del Sistema → Umbral de Roce**.

### Mitigación de DR (fórmula cuadrática)

```js
mitigateDR(damage, dr, penetracionPct = 0):
  drEfectivo = ⌊dr × (1 − penetracionPct/100)⌋
  mitigado   = ⌊damage² / (damage + drEfectivo)⌋
  mínimo     = max(1, ⌊damage × 0.20⌋)
  return     = max(mínimo, mitigado)
```

- El daño nunca cae por debajo del **20% del bruto**, evitando personajes indestructibles.
- La penetración reduce la DR efectiva antes del cálculo cuadrático.

### Críticos

Los críticos se activan por combinación de tags en `mergeTagProfiles`:

```
critThreshold (d100) = ⌊hitChance × critRangePct × 100⌋
Si d100 ≤ critThreshold Y el ataque impacta → daño × critMult
```

Cuando múltiples tags tienen `critRangePct`, se conserva el mayor (`critMult` asociado a ese tag).

---

## 9. Sistema de buffs y Rule Elements

Los items equipados (armas, armaduras, accesorios) pueden tener un array `reglas[]`. Cada regla tiene `type`, `selector` y `value`.

### Tipos de Rule Element disponibles

| Tipo               | Descripción                                                             | Selector formato              |
|--------------------|-------------------------------------------------------------------------|-------------------------------|
| `PercentRollBonus` | Suma X% al d100 al tirar esa sub-estadística                           | `roll.<substat>`              |
| `StatBonusFlat`    | Añade X puntos fijos a la sub-estadística                              | `stat.<substat>`              |
| `StatBonusPct`     | Multiplica la sub-estadística por (100+X)%                             | `stat.<substat>`              |
| `DamagePowerPct`   | Aumenta el Power% del arma para un tipo de daño                        | `damage.powerPct.<tipo>`      |
| `PenetrationPct`   | Ignora X% de DR del defensor para ese elemento                         | `penetration.<elemento>`      |
| `AccuracyBonus`    | Suma X puntos al score de precisión física o mágica                    | `attack.accuracy` / `acc_mag` |
| `DefenseBonus`     | Suma X puntos al score de evasión o guardia                            | `defense.evasion` / `guard`   |
| `BlockMult`        | Modifica cuánto daño pasa al bloquear (0=nada, 1=todo)                | `defense.block`               |
| `Resistance`       | Reduce el daño del tipo a la mitad (booleano, div 2)                  | `iwr.<elemento>`              |
| `Weakness`         | Duplica el daño del tipo (booleano, ×2)                               | `iwr.<elemento>`              |
| `Immunity`         | Anula completamente el daño del tipo                                   | `iwr.<elemento>`              |
| `ResistancePct`    | Reduce el daño del elemento en X% (acumulable, cap 100%)              | `iwr.<elemento>`              |
| `ResistanceFlat`   | Resta X puntos fijos de daño (tras divisiones y porcentajes)           | `iwr.<elemento>`              |
| `WeaknessPct`      | Amplifica el daño del elemento en +X%                                 | `iwr.<elemento>`              |
| `WeaknessFlat`     | Suma X puntos fijos de daño extra                                     | `iwr.<elemento>`              |

### Deduplicación de buffs

Los buffs con el mismo `slug` o `uuid` solo se aplican una vez (`dedupBuffs`). Los efectos de tipo `efecto` también llevan `slug` para evitar duplicados al reaplir.

---

## 10. Equipamiento

### Arma

| Campo                | Tipo    | Descripción                                                          |
|----------------------|---------|----------------------------------------------------------------------|
| `tipo_arma`          | string  | `melee` / `ranged` / `magic`                                         |
| `familia`            | string  | Espada, Hacha, Maza, Lanza, Arco, Ballesta, Arma de fuego, Vara, Bastón, Daga, Guante, Látigo, Exótica |
| `stat_key`           | string  | Sub-estadística de daño (default: `fuerza`)                          |
| `stat_key_accuracy`  | string  | Sub-estadística para precisión (default: `destreza`)                 |
| `power_pct`          | number  | % del stat que se convierte en daño base (escala con tier/rareza)    |
| `power_flat`         | number  | Daño plano adicional (escala con tier² × 0.6 × rarMult)             |
| `penetracion_pct`    | number  | % de penetración DR base                                             |
| `accuracy_bonus_pct` | number  | Bono de precisión base                                               |
| `elemento`           | string  | Tipo elemental del ataque (`physical`, `fire`, etc.)                 |
| `traits`             | array   | Array de claves de TAG_PROFILES                                      |
| `block_mult`         | number  | Si >0, el arma puede bloquear (como escudo). Ratio 0-1               |
| `manos_arma`         | string  | `"1"` o `"2"` manos                                                  |
| `danos_adicionales`  | array   | Fuentes adicionales de daño con su propio elemento y powerPct        |
| `habilidades_otorgadas` | array | UUIDs de habilidades concedidas al equipar                         |
| `rareza`             | string  | `comun` / `incomun` / `raro` / `epico` / `legendario` / `unico`     |

### Armadura

| Campo                       | Tipo    | Descripción                                                   |
|-----------------------------|---------|---------------------------------------------------------------|
| `tipo_armadura`             | string  | `ligera` / `media` / `pesada` / `escudo_ligero` / `escudo_pesado` |
| `familia`                   | string  | Ropa, Ligera, Media, Pesada, Escudo ligero, Escudo pesado, Exótica |
| `armor_dr`                  | number  | DR aportada (escala fuerte: ×tierMult × 1.2 × rarMult)       |
| `evasion_bonus`             | number  | Bono plano a evasión (+1 cada 50 niveles automático)          |
| `guard_bonus`               | number  | Bono plano a guardia (+1 cada 40 niveles automático)          |
| `block_mult`                | number  | Ratio de bloqueo para escudos (0.15–0.25 recomendado)        |
| `traits`                    | array   | Tags de la armadura                                           |
| `resistencias_elementales`  | object  | Resistencia % a fuego, rayo, hielo, oscuridad, luz, veneno, arcano, viento, tierra, agua, vacío |

### Habilidad

| Campo                  | Tipo    | Descripción                                                         |
|------------------------|---------|---------------------------------------------------------------------|
| `categoria`            | string  | `fisico` / `magico` / `especial` / `pasiva`                         |
| `coste`                | number  | Coste de Energía Mística (escala con `nivel/120`)                   |
| `tipo_recurso`         | string  | `energia` / `ninguno`                                               |
| `activa` / `pasiva`    | boolean | Tipo de habilidad                                                   |
| `alcance`              | string  | `self` / `objetivo` / `area`                                        |
| `rango_pies`           | number  | Rango en pies                                                       |
| `hab_stat_key`         | string  | Stat de daño (default: `afinidad`)                                  |
| `hab_stat_accuracy`    | string  | Score de precisión: `acc_mag` (default) o `accuracy`               |
| `hab_power_pct`        | number  | % de potencia (escala igual que arma)                               |
| `hab_power_flat`       | number  | Daño plano de habilidad                                             |
| `hab_penetracion_pct`  | number  | Penetración DR de la habilidad                                      |
| `hab_elementos`        | array   | Multi-elemento: `[{ elemento: "fire", pct: 60 }, { elemento: "arcane", pct: 40 }]` |
| `tipo_area`            | string  | Explosión, Cono, Cubo, Cilindro, Emanación, Línea, Cuadrado, Dónut |
| `efectos_adicionales`  | array   | Efectos de status aplicados al impactar                             |
| `tipo_ataque`          | string  | `ninguno` / `fisico` / `magico`                                     |

---

## 11. Escalado de items por nivel y rareza

Todos los items de combate (armas, armaduras, habilidades) escalan automáticamente. Los valores base del item actúan como "perfil" y los valores escalados se calculan en `prepareDerivedData()` y se cachean en `item.system._scaled`.

### Tiers de nivel

| Tier | Niveles     |
|------|-------------|
| T0   | 1 – 20      |
| T1   | 21 – 50     |
| T2   | 51 – 100    |
| T3   | 101 – 200   |
| T4   | 201 – 350   |
| T5   | 351 – 500   |
| T6   | 501 – 700   |
| T7   | 701 – 850   |
| T8   | 851 – 950   |
| T9   | 951 – 1000  |

### Fórmula de escalado

```js
tierMult = 1 + nivel / 80          // escala continua
rarMult  = RAREZA_MULT[rareza]      // multiplicador de rareza
```

### Multiplicadores de rareza

| Rareza     | rarMult | Color    |
|------------|---------|----------|
| Común      | ×1.00   | #888888  |
| Incomún    | ×1.08   | #55aa88  |
| Raro       | ×1.18   | #6699ff  |
| Épico      | ×1.32   | #cc8844  |
| Legendario | ×1.50   | #ff8800  |
| Único      | ×1.75   | #ffcc00  |

### Escalado de arma

```js
powerPctFinal   = round(power_pct  × tierMult × rarMult)
powerFlatFinal  = round(power_flat × tierMult² × 0.6 × rarMult)
penetracionFinal = penetracion_pct + ⌊nivel/100⌋ × 1.5   // +1.5% cada 100 niveles, máx +15%
accuracyFinal   = accuracy_bonus_pct + ⌊nivel/200⌋        // +1 cada 200 niveles
```

### Escalado de armadura

```js
armorDrFinal = round(armor_dr × tierMult × 1.2 × rarMult)  // DR escala fuerte
evasionFinal = evasion_bonus + ⌊nivel/50⌋                   // +1 por cada 50 niveles
guardFinal   = guard_bonus   + ⌊nivel/40⌋                   // +1 por cada 40 niveles
blockMultFinal = block_mult                                   // no escala (es ratio)
```

### Escalado de habilidad

```js
habPowerPctFinal   = round(hab_power_pct × tierMult × rarMult)
habPowerFlatFinal  = round(hab_power_flat × tierMult² × 0.6 × rarMult)
costeFinal = (coste > 0 && tipo_recurso === "energia")
             ? round(coste × (1 + nivel/120))
             : coste
```

---

## 12. Consumibles

| Tipo        | Emoji | Mecánica principal                                                     |
|-------------|-------|------------------------------------------------------------------------|
| `pocion`    | 🧪    | Restaura HP (`hp_change`) y/o EM (`em_change`). Sin tirada de combate. |
| `veneno`    | ☠️    | Aplica efectos temporales al objetivo. Puede usar pipeline de daño.    |
| `pergamino` | 📜    | Funciona como habilidad de un solo uso. Usa `fullDamagePipeline`.      |
| `bomba`     | 💣    | Auto-impacto: no hay tirada de acierto, solo evasión reduce el daño. Si se esquiva, se aplica graze (×0.20) en vez de dodge. |
| `comida`    | 🍖    | Efectos de rol, sin mecánica de combate directa.                       |
| `otro`      | 📦    | Genérico.                                                              |

### Campos del consumible

| Campo               | Tipo   | Descripción                                                           |
|---------------------|--------|-----------------------------------------------------------------------|
| `tipo`              | string | `pocion` / `veneno` / `pergamino` / `bomba` / `comida` / `otro`      |
| `usos.value`        | number | Usos restantes                                                        |
| `usos.max`          | number | Usos máximos                                                          |
| `hp_change`         | number | Curación de HP (positivo) o daño directo (negativo) para pociones     |
| `em_change`         | number | Restauración de EM                                                    |
| `modo_bomba`        | string | `objetivo` (impacto directo) / `explosion` (radio AOE)               |
| `rango_explosion`   | number | Radio en pies si `modo_bomba = explosion`                             |
| `stat_key`          | string | Stat atacante para pergaminos/bombas (default: `afinidad`)            |
| `elemento`          | string | Elemento del daño                                                     |
| `hab_elementos`     | array  | Multi-elemento igual que habilidades                                  |
| `power_pct`         | number | Escala igual que arma (vía `calcArmaScaling`)                         |
| `efectos_temporales`| array  | Rule Elements temporales que se aplican al objetivo                   |

---

## 13. Tags de items (Tagify)

Los tags se definen en `TAG_PROFILES` (en `config.js`, accesible también como `globalThis.ISEKAI_TAGS`). Cada tag puede tener los siguientes campos mecánicos:

| Campo                | Efecto                                                                   |
|----------------------|--------------------------------------------------------------------------|
| `evasionMult`        | Multiplicador sobre la evasión efectiva del defensor (< 1 = más difícil esquivar) |
| `blockMult`          | Multiplicador sobre la guardia efectiva del defensor                     |
| `accuracyMult`       | Multiplicador sobre la precisión efectiva del atacante                   |
| `powerMult`          | Multiplicador sobre el daño base calculado                               |
| `penetrationBonus`   | % adicional de penetración DR (aditivo con penetración del arma)         |
| `vampPct`            | % del daño final recuperado como HP del atacante                         |
| `guardBreakPct`      | % que reduce el guard del defensor solo para este ataque                 |
| `grazeMult`          | Reemplaza el ×0.20 del graze (ej: 0.50 = rozante vale ×0.50)            |
| `noGraze`            | Los graze se convierten en hit completo (×1.0)                          |
| `forceHitOnBlock`    | Si es bloqueado, el daño no se reduce                                    |
| `ignoreGuard`        | El defensor no puede bloquear este ataque                                |
| `critRangePct`       | Fracción del rango de impacto donde ocurre crítico                       |
| `critMult`           | Multiplicador de daño en crítico                                         |
| `requiresConcentration` | Flag de rol, sin mecánica de combate                                  |
| `requiresReload`     | Flag de rol, requiere acción de recarga                                  |
| `silent`             | Flag de rol, sin ruido                                                   |

### Tags de combate disponibles

**Velocidad/Manejo:** `ligero`, `pesado`, `lento`
**Técnica:** `finesse`, `preciso`, `torpe`
**Potencia:** `brutal`, `forceful`, `debilitado`, `deadly`
**Penetración:** `twoHand`, `stab`, `tearing`
**Proyectiles:** `firearm`, `hitscan`, `volley`, `scatter`
**Alcance:** `reach`, `ranged`, `thrown`
**Área:** `aoe`, `sweep`, `cleave`
**Magia:** `magic_pierce`, `channeled`
**Críticos:** `critico`, `fatal`, `fatal_aim`, `concussive`
**Outcomes:** `sangrienta`, `imparable`, `contundente`, `unblockable`
**Vampirismo:** `vampirico`
**Flags de rol:** `concentracion`, `recarga`, `silencioso`, `sagrado`, `corrupto`

Cuando un arma/habilidad tiene múltiples tags, los multiplicativos se combinan por **producto** y los aditivos por **suma** (`mergeTagProfiles`).

---

## 14. Raza, Origen y Trasfondo

### Ancestría (Raza)

Define la raza del personaje. Siempre activa (no requiere equipar).

| Campo                    | Descripción                                                             |
|--------------------------|-------------------------------------------------------------------------|
| `bonos_planos`           | Planos a cada una de las 18 sub-estadísticas                            |
| `porcentajes_crecimiento`| % de crecimiento adicional sobre cada sub-estadística                   |
| `hp_bonus_pct`           | % adicional al HP máximo calculado                                      |
| `longevidad`             | `corta` (~80 años) / `longevo` (~300 años) / `muy_longevo` (1000+ años)|
| `velocidad_pies`         | Velocidad de movimiento base (pies)                                     |
| `velocidad_nadar`        | Velocidad nadando (0 = sin capacidad)                                   |
| `velocidad_trepar`       | Velocidad trepando                                                      |
| `velocidad_excavar`      | Velocidad excavando                                                     |
| `velocidad_volar`        | Velocidad volando                                                       |
| `manos`                  | Número de manos (default: 2)                                            |
| `alcance`                | Alcance de ataque cuerpo a cuerpo en pies (default: 5)                  |
| `vision`                 | `normal` / `oscura` / `ciega` / etc.                                    |
| `habilidades_otorgadas`  | Habilidades pasivas/activas que otorga la raza                          |
| `bonos_especiales`       | Texto libre de rasgos raciales                                          |

### Origen

El origen representa el trasfondo cultural o regional. Siempre activo.

| Campo                | Descripción                                                              |
|----------------------|--------------------------------------------------------------------------|
| `bonos_planos`       | Planos a sub-estadísticas                                                |
| `porcentajes_crecimiento` | % de crecimiento adicional                                          |
| `bonus_scores`       | Bonos planos a los 5 scores de combate (`accuracy`, `acc_mag`, `evasion`, `guard`, `iniciativa`) |
| `idiomas`            | Lista de idiomas conocidos por este origen                               |
| `relems`             | Array de resistencias/debilidades/inmunidades elementales regionales. Formato: `{ elemento, tipo: "resistencia"/"debilidad"/"inmunidad", valor }` |

Las resistencias de origen se acumulan con las de armadura (cap 75%).

### Trasfondo

Representa el pasado social del personaje. Siempre activo.

| Campo                | Descripción                                                              |
|----------------------|--------------------------------------------------------------------------|
| `bonos_planos`       | Planos a sub-estadísticas                                                |
| `porcentajes_crecimiento` | % de crecimiento adicional                                          |
| `nivel_social`       | Impacta en el % de crecimiento de Carisma (ver tabla abajo)             |
| `bonus_stat_principal` | Bonus plano a **un** stat principal (`{ stat, valor }`)               |

**Nivel Social y bono a Carisma:**

| Nivel social  | Bonus CAR |
|---------------|-----------|
| `marginado`   | −20%      |
| `plebeyo`     | +0%       |
| `artesano`    | +8%       |
| `comerciante` | +15%      |
| `nobleza`     | +30%      |
| `realeza`     | +50%      |

---

## 15. Accesorio

Los accesorios aplican `porcentajes_finales` (multiplicadores %) sobre el valor ya calculado de cada sub-estadística (la última capa del cálculo). Solo actúan si están equipados.

| Campo                | Descripción                                                              |
|----------------------|--------------------------------------------------------------------------|
| `slot`               | Slot de equipamiento (para gestión de ranuras)                          |
| `subslot`            | Sub-ranura dentro del slot                                               |
| `porcentajes_finales`| % final sobre el valor de cada sub-estadística (se aplica como `accMult`) |
| `habilidades_otorgadas` | Habilidades concedidas al equipar el accesorio                       |

---

## 16. Modificadores de edad y longevidad

La edad del personaje modifica las categorías de sub-estadísticas mediante `getAgeMods(edad, longevidad)`. Solo aplica a personajes de tipo `character`.

### Parámetros de longevidad

| Longevidad      | Centro neutro | Zona muerta (±) | Ejemplo racial        |
|-----------------|---------------|------------------|-----------------------|
| `corta`         | 27.5 años     | ±2.5 años        | Humanos (~80 años)    |
| `longevo`       | 100 años      | ±10 años         | Elfos (~300 años)     |
| `muy_longevo`   | 350 años      | ±35 años         | Dragones (1000+ años) |

### Efectos por edad

- **Zona neutra:** sin modificador (×1.0 en todo).
- **Joven** (por debajo del centro): físico aumenta hasta +30%, mental decrece hasta −30%.
- **Anciano** (por encima del centro): físico decrece hasta −20%, mental aumenta hasta +30%.
- **Especial** (afinidades mágicas): sin modificador de edad (siempre ×1.0).

Las tasas de cambio escalan **inversamente** con `neutralCenter` para que la misma fracción de vida produzca el mismo efecto en todas las razas.

---

## 17. IWR (Inmunidades, Resistencias, Debilidades)

El sistema usa un **IWR Rico** (`richIWR`) que combina todas las fuentes:

### Fuentes de IWR (por orden de aplicación)

1. **IWR booleano heredado:** `resistencia` (÷2), `debilidad` (×2), `inmunidad` (×0).
2. **Resistencias % de origen:** acumulables, cap 75%.
3. **Resistencias % de armadura equipada:** acumulables, cap 75%.
4. **Rule Elements:** `ResistancePct`, `ResistanceFlat`, `WeaknessPct`, `WeaknessFlat`.

### Pipeline de aplicación IWR (`applyRichIWR`)

```
Para cada elemento atacante:
1. Inmunidad → daño = 0 (stop)
2. División (÷div, booleana)
3. −resistPct% (cap 100%)
4. +weakPct%
5. −resistFlat (plano)
6. +weakFlat (plano)
```

El slot `"todo"` afecta a todos los elementos excepto sí mismo.

---

## 18. Elementos y Perfiles

Cada elemento modifica los multiplicadores de evasión y bloqueo del defensor:

| Elemento      | Clave       | evasionMult | blockMult | glanceBonus |
|---------------|-------------|-------------|-----------|-------------|
| Físico        | `physical`  | ×1.00       | ×1.00     | 0           |
| Fuego         | `fire`      | ×1.00       | ×0.95     | 0           |
| Hielo         | `ice`       | ×1.00       | ×1.00     | 0           |
| **Rayo**      | `lightning` | **×0.85**   | ×0.95     | **−3**      |
| Viento        | `wind`      | ×1.05       | ×1.00     | +2          |
| Tierra        | `earth`     | ×1.00       | ×1.10     | 0           |
| Agua          | `water`     | ×1.00       | ×1.00     | 0           |
| Oscuridad     | `darkness`  | ×0.90       | ×0.90     | −2          |
| Luz           | `light`     | ×0.90       | ×1.05     | −2          |
| Veneno        | `poison`    | ×1.00       | ×1.00     | 0           |
| Arcano        | `arcane`    | ×0.95       | ×1.00     | 0           |
| **Balístico** | `ballistic` | **×0.80**   | **×0.90** | **−4**      |
| **Vacío**     | `void`      | **×0.70**   | **×0.60** | **−5**      |

Los elementos `void` y `ballistic` son los más difíciles de esquivar y bloquear. `wind` es el único que beneficia la evasión del defensor.

---

## 19. Energía Mística

Recurso de magia. El **máximo se calcula automáticamente**:

```js
EM_max = max(1, ⌊ (RSV×5 + CAN×3 + AFI×2) × (1 + nivel×0.03) ⌋)
```

- Escala al **3% por nivel** (más lento que HP que escala al 5%).
- El valor actual se clampea al máximo si los cambios de stats lo superan.
- En `system.json`, `secondaryTokenAttribute` = `"recursos.energia"` para mostrar en tokens.

---

## 20. Sistema de XP y Subida de Nivel

### XP requerida por nivel

```js
XP_para_subir_de_N_a_N+1  = N² × 10
XP_total_hasta_nivel_N     = Σ(n=1 to N-1) [ n² × 10 ]
```

| Nivel   | XP para subir | XP total acumulada |
|---------|---------------|--------------------|
| 1→2     | 10            | 10                 |
| 5→6     | 250           | 550                |
| 10→11   | 1.000         | 2.850              |
| 50→51   | 25.000        | 408.250            |
| 100→101 | 100.000       | 3.283.350          |

### Puntos por nivel

```js
puntosEnNivel(N) = 5 + ⌊N / 10⌋
```

| Nivel      | Puntos ganados |
|------------|----------------|
| 2 – 9      | 5 pts          |
| 10 – 19    | 6 pts          |
| 100 – 109  | 15 pts         |
| 1000       | 105 pts        |

### Subida de nivel (PC)

El botón **▲ Subir nivel** abre un diálogo con:

- Puntos disponibles: `puntosEnNivel(nivel_actual)`
- Contadores en tiempo real de puntos usados/sobrantes
- Distribución libre entre las 18 sub-estadísticas
- Validación de XP: si el personaje no tiene XP suficiente se avisa, pero el GM puede continuar.

---

## 21. Editor de NPC

Los NPCs tienen el botón **✏️ Editar** en lugar del de subir nivel. Abre un diálogo con:

- Campo de nivel editable directamente
- Total de puntos a repartir: `totalPuntosParaNivel(nivel)`
- Distribución manual por sub-estadística con validación en tiempo real
- Campo de **XP recompensa** para los jugadores al derrotar al NPC

### Presets de NPC

| Preset               | Clave             | Descripción                                       |
|----------------------|-------------------|---------------------------------------------------|
| 🦾 Bruto             | `brute`           | ATQ 40%, DEF 35%, AGI 8%, MAG 8%, SUE 9%         |
| ⚡ Atacante Mágico   | `magical_striker` | ATQ 20%, DEF 10%, AGI 15%, MAG 45%, SUE 10%      |
| 🎭 Maestro Habils.   | `skill_paragon`   | ATQ 20%, DEF 20%, AGI 25%, MAG 15%, SUE 20%      |
| 💨 Escaramuzador     | `skirmisher`      | ATQ 25%, DEF 10%, AGI 50%, MAG 5%, SUE 10%       |
| 🎯 Francotirador     | `sniper`          | ATQ 35%, DEF 10%, AGI 35%, MAG 5%, SUE 15%       |
| ⚔️ Soldado          | `soldier`         | ATQ 35%, DEF 30%, AGI 15%, MAG 10%, SUE 10%      |
| 🔮 Lanzador Hechizos | `spellcaster`     | ATQ 5%, DEF 8%, AGI 12%, MAG 65%, SUE 10%        |

La distribución de sub-estadísticas desde los presets usa `distribuirPuntosPorTargets()`, que aplica varianza aleatoria (configurable por preset, típicamente 10–20%) para que cada NPC sea único. La suma siempre es exactamente igual al total de puntos.

---

## 22. Sistema de idiomas del mundo

Accesible desde **Configuración del Sistema → Gestionar Idiomas del Mundo** (solo GM).

### Idiomas base predefinidos

| Idioma      | Rareza       |
|-------------|--------------|
| Común       | Común        |
| Élfico      | Común        |
| Enano       | Común        |
| Dracónico   | Común        |
| Gnómico     | Infrecuente  |
| Orco        | Infrecuente  |
| Infernal    | Raro         |
| Celestial   | Raro         |
| Primordial  | Raro         |
| Abisal      | Secreto      |
| Profundo    | Secreto      |

### Rareza de idiomas

| Clave         | Etiqueta     | Color    |
|---------------|--------------|----------|
| `comun`       | Común        | #8a9aaa  |
| `infrecuente` | Infrecuente  | #e07820  |
| `raro`        | Raro         | #4080e0  |
| `secreto`     | Secreto      | #8b1520  |

- Los idiomas **base** no pueden eliminarse, solo desactivarse.
- Los idiomas añadidos por el GM pueden eliminarse.
- Los personajes aprenden idiomas a través de su Origen (`idiomas[]`).

---

## 23. Sistema de bulk y moneda

### Bulk

El bulk representa el peso/volumen de los items. Se muestra en la hoja del personaje con barra de capacidad.

| Valor bulk | Peso equivalente |
|------------|------------------|
| `"-"`      | 0 (no pesa)      |
| `"L"`      | 0.1 unidades     |
| `"1"`      | 1 unidad         |
| … hasta `"20"` | 20 unidades   |

**Capacidad máxima de bulk:**
```js
BulkMax = 5 + ⌊FUE / 10⌋
```

### Moneda

El sistema usa 3 denominaciones:

| Moneda  | Clave   | Símbolo | Ratio de conversión | Color   |
|---------|---------|---------|---------------------|---------|
| Cobre   | `cobre` | ◈       | × 1                 | #c87533 |
| Plata   | `plata` | ✦       | × 10                | #c0c8d8 |
| Oro     | `oro`   | ☀️      | × 100               | #f0c040 |

---

## 24. Macros disponibles

En `module/macros.js` (copiar manualmente en Foundry como macros de tipo Script):

| Macro               | Función                                                              |
|---------------------|----------------------------------------------------------------------|
| `MACRO_rollStat`    | Diálogo para elegir sub-estadística y tirar con bono opcional        |
| `MACRO_quickAttack` | Ataca con el arma equipada hacia el token objetivo seleccionado      |
| `MACRO_applyBuff`   | Aplica un efecto con slug para deduplicación automática              |
| `MACRO_levelUp`     | Sube de nivel y distribuye puntos con diálogo interactivo            |
| `MACRO_rankInfo`    | Muestra resumen de rango, CP y stats principales en el chat          |

---

## 25. Configuración del sistema

Accesible desde **Foundry → Configuración → Isekai Rank System**:

| Opción                      | Clave                | Descripción                                                          | Default        |
|-----------------------------|----------------------|----------------------------------------------------------------------|----------------|
| **Umbrales de Rango**       | `rankThresholds`     | JSON editable con los CP mínimos para cada rango (F…SSR)            | Ver §5         |
| **Umbral de Roce (Glance)** | `glanceMarginBase`   | Margen de evasión a partir del cual el golpe es un roce              | `0.12`         |
| **Idiomas del Mundo**       | `worldIdiomas`       | Gestionado vía menú de configuración (no editable directamente)     | Lista base     |

Todas las opciones se aplican **en tiempo de juego** sin necesidad de reiniciar.

---

## 26. Instalación

1. Descarga o clona el repositorio.
2. Coloca la carpeta `isekai-rank-system/` en `Data/systems/` de tu instancia de Foundry VTT.
3. En Foundry, ve a **Configuración → Sistemas de juego** y selecciona o instala el sistema.
4. Crea un nuevo Mundo usando **Isekai Rank System** como sistema.
5. Copia las macros de `module/macros.js` en Foundry (**Macros → Nueva Macro → Script**).

### Configuración recomendada post-instalación

- Ajusta los **umbrales de rango** si quieres que el progreso sea más rápido o lento.
- Crea los ítems de **Ancestría** y **Origen** para las razas de tu mundo con sus bonos y porcentajes.
- Añade o desactiva **idiomas** del mundo según el lore de tu campaña.
- Ajusta el `power_pct` de las armas según el nivel de la campaña:
  - Armas de nivel bajo (T0–T1): 40–80
  - Armas épicas (T3–T5): 100–150
  - Armas míticas (T7–T9): 150–300+
- Configura el `block_mult` de los escudos (recomendado: 0.15–0.25).

---

## 27. API pública

El sistema expone utilidades en `game.isekai` accesibles desde macros y módulos externos:

```javascript
// ── Tiradas ─────────────────────────────────────────────────────────────────
await game.isekai.roll("Nombre del Actor", "fuerza");

// ── Daño y curación ─────────────────────────────────────────────────────────
await game.isekai.applyDamage("Nombre del Actor", 5000, "rayo");
  // Aplica IWR booleano y actualiza HP. Para IWR rico usar el pipeline directamente.
await game.isekai.heal("Nombre del Actor", 2000);

// ── Motor de combate ────────────────────────────────────────────────────────
game.isekai.engine.logChance(attackScore, defenseScore);
game.isekai.engine.mitigateDR(damage, dr, penetracionPct);
game.isekai.engine.applyIWR(damage, "fuego", actor.system.iwr);    // legacy booleano
game.isekai.engine.fullDamagePipeline({ /* opts */ });              // pipeline completo

// ── XP y niveles ────────────────────────────────────────────────────────────
game.isekai.engine.xpTotalParaNivel(nivel);
game.isekai.engine.puntosEnNivel(nivel);
game.isekai.engine.totalPuntosParaNivel(nivel);
game.isekai.engine.distribuirPuntosPorTargets(total, targets, variance);

// ── Rangos y arquetipos ─────────────────────────────────────────────────────
game.isekai.calcRango(cp);         // → "F" | "E" | … | "SSR"
game.isekai.CONFIG;                // RANGOS, RANGO_UMBRALES, ELEMENT_PROFILES, NPC_PRESETS…

// ── Utilidades de buffs ──────────────────────────────────────────────────────
game.isekai.engine.dedupBuffs(buffs, statKey);
```

### Funciones de escalado de items (accesibles desde config.js)

```javascript
import { calcArmaScaling, calcArmaduraScaling, calcHabilidadScaling,
         calcTierMult, calcTierPenBonus, getItemTier } from "./module/config.js";

// Obtener stats escalados de un arma
const scaled = calcArmaScaling(item.system);
// → { tier, tierMult, rarMult, powerPctFinal, powerFlatFinal, penetracionFinal, accuracyFinal }
```

---

> **Nota de diseño:** Este sistema no usa combate en mesa clásico con +X al golpe. Todo escala mediante porcentajes y logaritmos para que números de varios millones de HP sigan siendo manejables. Una diferencia de ×10 en accuracy da solo ~91% de chance de golpear, nunca el 100% automático. La DR usa una fórmula cuadrática que garantiza que el daño siempre llega (mínimo 20% del bruto), evitando personajes indestructibles a escala alta. El escalado de items (Tiers T0–T9 × rareza) permite que los items de nivel 1 y los de nivel 1000 coexistan con diferencias de potencia predecibles y controladas.
