# AI Wars 4 — Documentation Technique

## Table des matières

1. [Architecture générale](#1-architecture-générale)
2. [Boucle de jeu](#2-boucle-de-jeu)
3. [État global](#3-état-global)
4. [Entités](#4-entités)
5. [Système de vision et brouillard de guerre](#5-système-de-vision-et-brouillard-de-guerre)
6. [Exploration de carte](#6-exploration-de-carte)
7. [Comportements des unités](#7-comportements-des-unités)
8. [Système de combat](#8-système-de-combat)
9. [Système économique](#9-système-économique)
10. [Système de modules (Barracks)](#10-système-de-modules-barracks)
11. [Stimulant](#11-stimulant)
12. [IA stratégique](#12-ia-stratégique)
13. [Algorithme génétique](#13-algorithme-génétique)
14. [Rendu](#14-rendu)
15. [Simulation headless](#15-simulation-headless)
16. [Persistance](#16-persistance)
17. [Serveur](#17-serveur)

---

## 1. Architecture générale

Le projet est un **fichier HTML unique** (`rts-ai-spectator.html`) contenant tout le CSS et le JavaScript inline. Aucune dépendance externe (pas de framework, pas de bundler).

```
rts-ai-spectator.html   ← jeu complet (~2100 lignes)
server.js               ← serveur HTTP Node.js minimal
.claude/launch.json     ← config Claude Preview (port 3001)
context.md              ← aperçu du projet
technical_documentation.md
```

**Polices externes** (Google Fonts, CDN) :
- `Orbitron` — titres, HUD
- `Share Tech Mono` — texte général, stats

---

## 2. Boucle de jeu

```
requestAnimationFrame(loop)
  │
  ├─ rawDt = (timestamp - lastTime) / 1000   (capped à 50ms)
  ├─ totalDt = rawDt × speed
  ├─ steps = ceil(totalDt / 0.05)            (sous-pas max 50ms)
  │
  └─ for each step:
       gameTime += stepDt
       updateGame(stepDt)
  │
  ├─ updateTimer()
  ├─ render()
  ├─ updateHUD()
  └─ échantillonnage ressources toutes les 5s → updateResourceGraph()
```

**Variable `speed`** : multiplicateur de simulation (×0.5 à ×8). Les sous-pas évitent les artefacts de collision à haute vitesse.

### updateGame(dt)

Ordre d'exécution à chaque pas :

1. **Bâtiments** — tourelles, spawns HQ/dépôt/caserne/usine/starport
2. **Unités** — chaque unité exécute son comportement selon son état
3. **Particules** — déplacement et expiration
4. **IA** — `updateExploreGrid` + `runAI` pour chaque faction vivante
5. **Éliminations** — `checkEliminations()`
6. **Victoire** — `checkWin()`

---

## 3. État global

```javascript
// Carte
let W, H, COLS, ROWS          // dimensions canvas en pixels / tiles
const TILE = 20                // taille d'une tuile en pixels
let map = []                   // map[y][x] = 0 (libre) | 1 (obstacle)

// Entités
let buildings = []             // tableau de tous les bâtiments
let units = []                 // tableau de toutes les unités
let particles = []             // projectiles et effets visuels
let resources = []             // nœuds de ressources

// Factions
let factions = {}              // factions[fid] = objet faction
let aiState = {}               // aiState[fid] = état IA de la faction

// Temps
let gameRunning = false
let speed = 1
let gameTime = 0               // secondes de jeu écoulées
let uid = 0                    // compteur d'ID unique

// Apprentissage (persistant via localStorage)
let agentMemory = {
  generation: Number,
  agents: {
    [fid]: {
      params: {...},           // paramètres stratégiques
      history: [...],          // 10 dernières parties
      wins: Number,
      totalGames: Number,
      fitness: Number          // score EMA de performance
    }
  }
}

// Graphes
let resourceHistory = {}       // resourceHistory[fid] = [{t, m, v}, ...]
let lastResourceSample = 0
```

---

## 4. Entités

### 4.1 Factions

```javascript
factions[fid] = {
  resources: Number,           // minéraux
  vespene: Number,             // gaz vespène
  hqId: Number,                // ID du bâtiment HQ
  color: String,               // couleur hex
  dim: String,                 // couleur sombre pour remplissage
  name: String,
  eliminated: Boolean,
  eliminatedAt: Number|null,   // gameTime d'élimination
  knownEnemyBuildings: Set,    // IDs des bâtiments ennemis mémorisés
  stats: {
    resourcesGathered: Number,
    killCount: Number,
    lossCount: Number,
  },
  // Stimulant
  stimResearched: Boolean,
  stimResearching: Boolean,
  stimResearchTimer: Number,   // secondes restantes (60s total)
}
```

### 4.2 Unités

Créées par `makeUnit(fid, type, x, y)` :

```javascript
{
  id, faction, type, x, y,
  tx, ty,                      // position cible (non utilisé activement)
  hp, maxHp,
  speed,                       // unités/px par frame à 60fps
  attack, range, size,
  attackCooldown,
  state,                       // 'idle'|'harvest'|'build'|'attack'|'retreat'|'scout'
  target,                      // ID de la cible
  resourceTarget,              // index dans resources[]
  buildTarget,                 // ID du bâtiment en construction
  carrying, carryingVespene,   // minerais transportés
  angle,                       // radians, direction de déplacement
  harvestPhase,                // 'goToRes'|'mining'|'returnToBase'
  killCount,
  // Médivac uniquement
  cargo, healCooldown, medState, dropTarget,
  energy,                      // énergie de soin (max 100)
  // Stimulant (marines et marauders)
  stimActive: Boolean,
  stimTimer: Number,           // secondes restantes (15s total)
}
```

**Stats par type :**

| Type | PV | Vitesse | Attaque | Portée | Taille | Spécial |
|------|-----|---------|---------|--------|--------|---------|
| worker | 40 | 0.6 | 3 | 15 | 4 | Récolte + construction |
| marine | 50 | 0.9 | 12 | 35 | 5 | Infanterie légère |
| marauder | 80 | 0.7 | 20 | 38 | 6 | `armored: true` |
| tank | 220 | 0.45 | 20 | 55 | 7 | Mode siège (×3 portée, AOE 30px) |
| medivac | 120 | 1.1 | 0 | 100 | 6 | Soin 40HP/s, énergie max 100 |

### 4.3 Bâtiments

Créés par `makeBuilding(fid, type, x, y)` :

```javascript
{
  id, faction, type, x, y,
  hp, maxHp, size,
  buildProgress,               // 0→1 pendant la construction
  spawnTimer,
  attack, range, attackCooldown,
  label,                       // texte affiché sur la map
  // Casernes uniquement
  module,                      // null | 'techlab' | 'reactor'
  modulePending,               // module en cours de construction
  moduleBuildTimer,            // secondes restantes (30s total)
}
```

**Stats par type :**

| Type | PV | Taille | Coût min | Coût vesp | Label | Notes |
|------|-----|--------|----------|-----------|-------|-------|
| hq | 600 | 28 | 0 | 0 | HQ | Spawn workers (max 10) |
| barracks | 200 | 20 | 150 | 0 | BAR | Spawn marines/marauders |
| factory | 250 | 22 | 150 | 75 | FAC | Spawn tanks |
| starport | 200 | 22 | 100 | 100 | STP | Spawn medivacs |
| depot | 300 | 20 | 100 | 0 | DEP | Avant-poste de récolte |
| turret | 150 | 16 | 75 | 0 | TUR | Attaque 18, portée 90, cooldown 1.4s |

---

## 5. Système de vision et brouillard de guerre

### Rayons de vision (pixels)

```javascript
const VISION_RADIUS = {
  worker: 90, marine: 130 (via 'soldier'), tank: 160,
  hq: 160, depot: 100, barracks: 110,
  factory: 110, starport: 110, medivac: 140, turret: 100
}
```

### Fonctions

```javascript
getVisionCircles(fid)          // → [{x, y, r}, ...] pour tous les bâtiments et unités
isVisible(fid, x, y)           // → Boolean
visibleEnemyUnits(fid)         // → unités ennemies dans la vision
visibleEnemyBuildings(fid)     // → bâtiments ennemis connus (avec mémoire)
```

**Mémoire des bâtiments** : une fois un bâtiment ennemi repéré, il est ajouté à `factions[fid].knownEnemyBuildings` (Set d'IDs). Il reste mémorisé jusqu'à sa destruction (retrait du Set via `damageEntity`).

### Brouillard de guerre (rendu)

Activé quand `fogFaction !== null`. Implémenté via un canvas offscreen :

1. Fond opaque (`rgba(5,10,15,0.92)`) sur tout le canvas
2. `globalCompositeOperation = 'destination-out'` : effacement des zones de vision via dégradé radial
3. Composition du canvas offscreen par-dessus la scène principale

---

## 6. Exploration de carte

Grille basse résolution (`EXPLORE_CELL = 80px`) mémorisant les cellules visitées par faction.

```javascript
exploreGrid[fid][cy][cx] = true   // cellule (cx, cy) visitée par fid
```

**`updateExploreGrid(fid)`** : marque toutes les cellules couvertes par les cercles de vision actuels.

**`getUnexploredTarget(fid)`** : retourne une destination non explorée en tirant 40 candidats aléatoires et en favorisant les zones éloignées du HQ allié.

**`exploreRatio(fid)`** → ratio 0–1 de la carte couverte.

---

## 7. Comportements des unités

### Automate d'état (state machine)

```
worker  : idle → harvest ↔ mining ↔ returnToBase
                idle → build
                idle/harvest → attack (si menace à <45px)

marine/marauder/tank : attack ↔ retreat
                       attack → scout (assigné par IA)

medivac : comportement autonome (pas de state machine explicite)
```

### `doHarvest(u, dt)`

Cycle : `goToRes` → arrivée à <10px → `mining` (50 unités/s) → plein ou épuisé → `returnToBase` → dépôt à <18px → livraison → `goToRes`.

Sélection du dépôt de retour : bâtiment (`hq` ou `depot`) le plus proche appartenant à la faction.

### `doBuild(u, dt)`

Worker se déplace vers le bâtiment cible. À <20px : `buildProgress += 0.28 × dt` (construction complète en ~3.5s).

### `doScout(u, dt)`

Éclaireur se déplace vers `getUnexploredTarget`. Si ennemi visible à portée ×2.5 : bascule en `attack`.

### `doMedivac(u, dt)`

1. Soigne tous les alliés blessés à portée (40 HP/s, consomme 20 énergie/s)
2. Se déplace vers l'allié le plus blessé (< 80% PV)
3. Orbite autour de lui quand à portée
4. Si aucun blessé : suit le centre de gravité des combattants
5. Si personne : retourne au HQ

Régénération d'énergie : 10/s quand pas de soin actif.

### `doCombatUnit(u, dt)`

```
1. Décompte stimActive (stimTimer -= dt)
2. Retraite si PV < 25% (sauf tank) → moveTo(HQ) + régén 2 HP/s
3. Sortir de retraite si PV > 60%
4. Tank : gestion mode siège (deploying / siege / undeploying, 2s de transition)
5. Résolution de cible via resolveTarget()
6. Si distance > range : déplacement (×1.5 si stimActive)
7. Si distance ≤ range :
   - Auto-stim si conditions remplies
   - Attaque si attackCooldown ≤ 0
```

---

## 8. Système de combat

### Résolution de cible (`resolveTarget`)

1. Ennemi visible à portée ×1.5 (opportuniste, priorité absolue)
2. Cible actuelle (si encore valide et visible)
3. `pickCombatTarget` : HQ ennemi > bâtiments de production > autres bâtiments > unités

### Dégâts (`damageEntity`)

```javascript
// Modificateurs d'armure
marauder vs tank attacker  → dmg × 0.80   (blindage)
tank vs non-armored        → dmg × 1.25   (bonus anti-infanterie)
```

Si HP ≤ 0 :
- **Unité** : supprimée de `units[]`, `lossCount++`
- **Bâtiment** : supprimé de `buildings[]`, retiré de tous les `knownEnemyBuildings`

### Mode siège du tank

- Portée normale : 55px ; portée siège : 165px (×3)
- Dégâts siège : 45 (vs 20 en mobile)
- AOE : 70% des dégâts siège sur toutes les entités ennemies dans un rayon 30px autour de la cible
- Transition deploying/undeploying : 2 secondes d'immobilité
- Le tank se déploie si un ennemi est dans la portée siège mais pas dans la portée normale

### Élimination

Faction éliminée si son HQ est détruit. Toutes ses unités et bâtiments sont immédiatement supprimés.

---

## 9. Système économique

### Ressources sur la carte

8 clusters répartis : 4 aux coins (bases), 4 au centre.

Par cluster :
- 6 cristaux de **minéraux** (2000 unités chacun)
- 2 cristaux de **vespène** (1500 unités chacun)

**Taux de récolte** : 50 unités/s par worker (les deux types).

### Coûts de production

| Unité | Minéraux | Vespène | Temps spawn | Max |
|-------|----------|---------|-------------|-----|
| worker | 50 | 0 | 20s | 10 |
| marine | 50 (×count) | 0 | 10s | 18 (infantry) |
| marauder | 75 | 25 | 13s | 18 (infantry) |
| tank | 150 | 100 | 22s | 6 |
| medivac | 100 | 100 | 28s | 3 |

### Règles de spawn

- **HQ** : workers jusqu'à 10, coût 50 min, timer 20s
- **Barracks** : marines/marauders jusqu'à 18 infantry au total. Ratio : 1 marauder pour 2 marines si techlab présent
- **Factory** : tanks jusqu'à 6
- **Starport** : medivacs jusqu'à 3
- Si ressources insuffisantes : retry dans 4s

---

## 10. Système de modules (Barracks)

### Principe

Une caserne ne peut avoir **qu'un seul module** à la fois. Le module est permanent une fois construit.

### Construction

- **Coût** : 50 minéraux + 50 vespène
- **Durée** : 30 secondes (`moduleBuildTimer`)
- L'IA ordonne la construction via la logique dans `runAI` (timer `moduleTimer` toutes les 5–10s)

### Types de modules

| Module | Clé | Effets |
|--------|-----|--------|
| Centre Technique | `techlab` | Débloque la production de marauders + permet la recherche Stimulant |
| Réacteur | `reactor` | Marines produits 2 par 2 (cost×2, timer identique) |

### Recherche Stimulant

Déclenchée automatiquement par l'IA dès qu'un techlab est opérationnel.
- **Durée** : 60 secondes
- Progression via `f.stimResearchTimer` décrémenté dans le loop des bâtiments
- Une seule recherche par faction (irréversible)

### Logique IA (module ordering)

```
toutes les 5–10s :
  si barracks libre (pas de module, pas de pending) ET resources >= 50 ET vespene >= 50 :
    si aucun techlab → construire techlab
    sinon            → construire reactor
  si stimResearched=false ET stimResearching=false ET techlab opérationnel :
    lancer recherche stimulant
```

### Rendu visuel

- Badge **CT** (orange `#ff9500`) ou **RÉ** (cyan `#00cfff`) dans le coin supérieur droit de la caserne
- Barre de progression sous le badge pendant la construction du module (30s)
- Barre jaune en haut de la caserne pendant la recherche Stimulant
- Icône **⚡** jaune si stimulant débloqué

---

## 11. Stimulant

### Activation

Déclenchée **automatiquement** par `doCombatUnit` quand une unité entre à portée d'attaque :

```javascript
conditions :
  !u.stimActive
  && factions[u.faction].stimResearched
  && (u.type === 'marine' || u.type === 'marauder')
  && u.hp > u.maxHp * 0.4    // seuil de sécurité (40% max)
```

### Effets

```javascript
// À l'activation
u.hp = Math.max(1, u.hp * 0.67)    // -33% PV actuels
u.stimActive = true
u.stimTimer = 15                    // durée : 15 secondes

// Pendant le stim
vitesse     × 1.5
cadence attaque × 1.5  (attackCooldown / 1.5)
```

### Comportement multi-stim

- Le stim peut se réactiver après expiration tant que `hp > 40% maxHp`
- Chaque activation coûte 33% des PV **courants** → chaque usage réduit la capacité future
- Une unité à 40% PV exactement ne peut plus se stimmer

### Rendu

Halo blanc (`shadowBlur=14/18px`, `shadowColor='#fff'`) sur les marines et marauders stimulés.

---

## 12. IA stratégique

### Phases de jeu

```
economy → expand → assault → pressure ↔ expand
```

| Phase | Condition d'entrée | Comportement |
|-------|--------------------|--------------|
| economy | départ | Construction pure, pas d'attaque (sauf harcèlement si `harassEarly`) |
| expand | ≥ 1 caserne | Production militaire, harcèlement possible |
| assault | `milPower >= assaultThreshold` | Attaque totale |
| pressure | après assault | Attaques partielles (50% des unités) |

### Puissance militaire (`milPower`)

```javascript
marines×1 + marauders×1.5 + tanks×3
```

### Comportements permanents

**Défense d'urgence** : si ennemis visibles à `< defenseRadius` du HQ, les soldats proches sont redirigés sur ces menaces.

**Éclaireurs** : 1–2 marines envoyés en exploration si le HQ ennemi n'est pas encore localisé (`explored < 30%` → 2 éclaireurs, sinon 1).

**Alliances** : si une faction est nettement plus forte, les deux factions les plus faibles font tacitement cause commune contre elle (ciblent le leader).

**Composition adaptative** :
- ≥ 3 tanks ennemis visibles → `compositionHint = 'marauder'` (anti-armure)
- ≥ 8 fantassins ennemis visibles → `compositionHint = 'tank'` (anti-infanterie)

**Regroupement** : avant l'assaut, si les unités sont dispersées (écart-type > seuil), elles se rallient au centre de gravité du groupe avant d'attaquer.

### Logique de construction

Priorité décroissante :
1. Caserne si aucune (min 150)
2. Usine si ≥ 3 soldats (min 150 + 75 vesp)
3. Starport si usine présente (min 100 + 100 vesp)
4. Tourelles défensives (max 3, 75 min)
5. Casernes/usines supplémentaires selon `compositionHint` et seuils de ressources
6. Dépôts selon `expansionDrive`

### Priorité de cibles (`targetPriority`)

| Valeur | Ordre d'attaque |
|--------|----------------|
| military | HQ > unités de combat > bâtiments de prod > autres |
| workers | HQ > workers > bâtiments de prod > autres |
| buildings | HQ > bâtiments de prod > autres > unités |

---

## 13. Algorithme génétique

Exécuté à chaque fin de partie dans `endGame()` et `simEndGame()`.

### Classement

Basé sur l'ordre d'élimination (dernier en vie = 1er). En cas d'égalité temporelle : ordre de `eliminatedAt`.

### Score de fitness (EMA)

```javascript
fitness_brut = (4 - place) × 10 + killCount × 0.5 - lossCount × 0.2
fitness = fitness_ancien × 0.6 + fitness_brut × 0.4
```

### Adaptation des paramètres

```
Gagnant (place = 0) :
  mutateParams(params, 0.06)          // mutation légère 6%

Perdant N (place = 1, 2, 3) :
  params = crossoverParams(params, winner.params)
  mutateParams(params, 0.10 + place × 0.07)
  // 2e → 17%, 3e → 24%, 4e → 31%
```

### Croisement (`crossoverParams`)

Pour chaque paramètre : sélection aléatoire 50/50 entre les deux parents.

### Mutation (`mutateParams`)

Chaque paramètre muté avec probabilité `rate` :

| Paramètre | Perturbation | Bornes |
|-----------|-------------|--------|
| assaultThreshold | ±2 | [3, 20] |
| attackInterval | ±3 | [6, 30] |
| defenseRadius | ±30 | [100, 400] |
| expansionDrive | ±0.125 | [0, 1] |
| targetPriority | tirage aléatoire | military/workers/buildings |
| harassEarly | flip aléatoire | true/false |

---

## 14. Rendu

### Pipeline

```
render()
  ├─ fillRect fond (#050a0f)
  ├─ drawMap()          tuiles obstacles (gris sombre)
  ├─ drawResources()    cristaux (bleu cyan / vert)
  ├─ drawBuildings()    bâtiments + modules + barres HP
  ├─ drawUnits()        unités + barres HP + effets stim
  ├─ drawParticles()    projectiles et explosions
  └─ drawFog(fogFaction) si mode faction actif
```

### Particules

| Type | Couleur | Durée | Comportement |
|------|---------|-------|--------------|
| bullet | couleur faction | distance/280s | vitesse 280px/s |
| spark | couleur faction | 0.5–0.9s | explosion radiale |
| heal | `#00ff88` | 0.35s | particule verte montante |

### HUD (sidebar)

- **Classement** : position de chaque faction avec icône de rang
- **Apprentissage** : paramètres actuels et fitness de chaque agent
- **Ressources** : graphe canvas `196×70px`, historique 50 points sur 5s, courbe par faction
- **Battle Log** : événements horodatés (constructions, éliminations, assauts…)

---

## 15. Simulation headless

`startSimulation(N)` lance N parties sans rendu, à la vitesse maximale, via `setTimeout` (une partie par tick pour ne pas bloquer l'UI).

**Dimensions fixes** : `W=1200, H=700` (indépendant du canvas visible).

`updateGameHeadless(dt)` : identique à `updateGame` sans les appels de rendu ni les particules visuelles. Duplique la logique de spawn des bâtiments et des unités (marauder cost, modules, etc.).

Progression affichée dans `#simOverlay` avec barre de progression et logs intermédiaires.

---

## 16. Persistance

**Clé localStorage** : `aiwars4_v2`

**Structure sauvegardée** :
```javascript
{
  generation: Number,
  agents: {
    [fid]: {
      params: {assaultThreshold, attackInterval, defenseRadius, expansionDrive, targetPriority, harassEarly},
      history: [{place, survivalTime, resourcesGathered, killCount, lossCount}],  // 10 max
      wins: Number,
      totalGames: Number,
      fitness: Number
    }
  }
}
```

**Chargement** : `loadMemory()` avec merge des agents manquants et complétion des paramètres absents (compatibilité avec les anciennes sauvegardes).

**Réinitialisation** : bouton "Effacer apprentissage" → `freshMemory()` + `saveMemory()`.

---

## 17. Serveur

`server.js` — serveur HTTP Node.js sans dépendance externe :

```javascript
const PORT = process.env.PORT || 3000
// Sert les fichiers statiques du répertoire courant
// Types MIME : .html, .js, .css, .png, .jpg
// Fichier par défaut : rts-ai-spectator.html
```

Le `.claude/launch.json` configure le port à **3001** pour éviter les conflits :

```json
{
  "name": "AI Wars",
  "runtimeExecutable": "node",
  "runtimeArgs": ["server.js"],
  "port": 3001
}
```
