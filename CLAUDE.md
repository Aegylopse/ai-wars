# AI Wars — Instructions Claude

## Projet
- Repo GitHub : `Aegylopse/ai-wars`
- Fichier principal : `rts-ai-spectator.html` (jeu RTS entier en un seul fichier HTML/JS)
- Serveur de dev : Node.js port 3001 — lancer via `.claude/launch.json` (preview_start)
- Branche de travail : `claude/wonderful-tesla` → PR vers `main`

## Règles de workflow
- **Committer automatiquement** après chaque ensemble de modifications, sans attendre la demande
- **Langue** : français dans le jeu, français dans les échanges
- **Vitesse par défaut** du jeu : ×3
- Après chaque modification de code, vérifier via le serveur de preview (preview_screenshot, preview_console_logs)

## Règle GDD ↔ Jeu (CRITIQUE)
`GDD.md` est la source de vérité pour toutes les stats (HP, dégâts, vitesses, coûts, portées).
- Toute modif dans `GDD.md` → répercuter immédiatement dans `rts-ai-spectator.html`
- Toute modif de stats dans le jeu → mettre à jour `GDD.md`
- Ne jamais laisser les deux fichiers désynchronisés

## Architecture du jeu (`rts-ai-spectator.html`)

### Constantes & config
- `TILE = 20` — taille d'une tuile de la grille
- `UNIT_TYPES` (~ligne 185) — stats des unités (hp, speed, attack, range, size)
- `BUILDING_TYPES` (~ligne 192) — stats des bâtiments (hp, size, cost, costVespene, attack, range)
- `FACTION_DEFS` — 4 factions : nord (TL), est (TR), sud (BR), ouest (BL)
- HQ positions : `W*0.08, H*0.10` (TL) / `W*0.92, H*0.10` (TR) / `W*0.92, H*0.90` (BR) / `W*0.08, H*0.90` (BL)

### Fonctions clés
- `generateMap()` — clusters de rochers organiques, nettoie les zones autour des bases et ressources
- `spawnResources()` — arc de 6 minéraux + 2 vespène autour de chaque HQ (R=60, vers le coin)
- `makeUnit(fid, type, x, y)` — factory unités ; champs : `stimActive, stimTimer, kiteTimer, kiteAngle, path[], pathTarget, pathTimer, _stuckT, _stuckX, _stuckY`
- `doCombatUnit(u, dt)` — logique combat (stim, kiting, siège tank, retraite)
- `moveTo(u, tx, ty, dt)` — déplacement avec A*, détection de blocage, stim ×1.5
- `moveUnit(u, angle, speed, dt)` — déplacement bas niveau avec collision obstacles (unités volantes passent librement)
- `tileBlocked(px, py)` — vérifie si une case pixel est un obstacle
- `aStar(x0, y0, x1, y1)` — A* 8 directions, MinHeap, max 2000 iter, retourne `[{x,y}...]`
- `isAirUnit(u)` — medivac ou viking en mode air → ignore obstacles et A*
- `doMedivac(u, dt)` — soin, évacuation, transport tactique
- `doViking(u, dt)` — bascule sol/air selon menaces
- `doHellion(u, dt)` — cône de flamme, bonus ×2 vs unités légères
- `doScout(u, dt)` — exploration carte, transition vers combat si menace proche
- `runAI(fid, dt)` — IA faction (phases économie/militaire, modules, stim research)
- `drawDominanceBar()` — barre de dominance en bas du canvas

### Boucles de jeu
Il y a **deux boucles** : `updateGame` (avec rendu) et `updateGameHeadless` (simulation rapide).
Toute modification de logique doit s'appliquer aux deux — utiliser `replace_all: true` pour les blocs identiques.

### Carte & obstacles
- `map[row][col]` : 0 = libre, 1 = obstacle
- Taille grille : `COLS = W/TILE`, `ROWS = H/TILE`
- Clusters de rochers générés par random walk ; zones de base (14×14 tuiles) et ressources (±5 tuiles) nettoyées
- Unités terrestres contournent via A* ; unités volantes (medivac, viking air) passent librement

## Cooldowns d'attaque
- Marine : `0.7 + random*0.35` s (~0.85 s moy.)
- Marauder : `1.2 + random*0.3` s (~1.35 s moy.)
- Tank mobile : ~0.85 s (même formule que marine)
- Tank siège : `1.8` s fixe
- Worker : `0.9 + random*0.4` s (~1.1 s moy.)
- Tourelle : `1.4` s fixe
