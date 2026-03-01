# AI Wars 4 — Contexte du projet

## Vue d'ensemble

Jeu de stratégie temps réel (RTS) entièrement dans le navigateur, en mode **spectateur** : 4 factions IA s'affrontent sans intervention humaine. Les agents apprennent et évoluent entre les parties grâce à un algorithme génétique.

## Architecture

- **Un seul fichier** : `rts-ai-spectator.html` (~2000 lignes, HTML + CSS + JS inline)
- **Serveur** : `server.js` — serveur HTTP Node.js minimal pour servir le fichier en local (port 3001)
- **Persistance** : `localStorage` (`aiwars4_v2`) — sauvegarde la mémoire des agents entre sessions

## Factions

| ID | Couleur | Position |
|----|---------|----------|
| `nord` | Cyan `#00d4ff` | Haut-gauche |
| `est` | Orange `#ff9500` | Haut-droite |
| `sud` | Rose `#ff3a7a` | Bas-droite |
| `ouest` | Vert `#44ff88` | Bas-gauche |

## Unités

| Type | PV | Vitesse | Attaque | Portée | Notes |
|------|-----|---------|---------|--------|-------|
| `worker` | 40 | 0.6 | 3 | 15 | Récolte et construit |
| `marine` | 50 | 0.9 | 12 | 35 | Infanterie légère |
| `marauder` | 80 | 0.7 | 20 | 38 | Blindé, anti-armure |
| `tank` | 220 | 0.45 | 20 | 55 | Mode siège (×3 portée, AOE) |
| `medivac` | 120 | 1.1 | — | 100 | Transport + soins |

## Bâtiments

| Type | Coût | Notes |
|------|------|-------|
| `hq` | — | Quartier général, production workers |
| `barracks` | 150 min | Production marines/marauders + modules |
| `factory` | 150 min + 75 vesp | Production tanks |
| `starport` | 100 min + 100 vesp | Production medivacs |
| `depot` | 100 min | Avant-poste de récolte |
| `turret` | 75 min | Défense fixe, attaque 18, portée 90 |

## Système de modules (Barracks)

Une caserne peut recevoir **un seul module** (coût : 50 min + 50 vesp) :

- **Centre Technique (techlab)** — débloque la production de marauders + permet de rechercher le Stimulant (60s de recherche)
- **Réacteur (reactor)** — les marines sont produits 2 par 2

## Stimulant

Technologie recherchée dans le Centre Technique. Une fois débloquée :
- S'active **automatiquement** quand un marine/marauder entre à portée d'attaque
- **Coût** : -33% des PV actuels à chaque usage (min 1 PV)
- **Effet** : +50% vitesse de déplacement et cadence d'attaque pendant **15 secondes**
- **Condition** : l'unité doit avoir > 40% de ses PV max pour s'activer
- **Visuel** : halo blanc sur l'unité stimulée

## IA et apprentissage

Chaque faction est pilotée par un agent avec 6 paramètres :

| Paramètre | Rôle |
|-----------|------|
| `assaultThreshold` | Nb de soldats requis avant d'attaquer |
| `attackInterval` | Secondes entre deux offensives |
| `defenseRadius` | Rayon de surveillance du QG |
| `expansionDrive` | Tendance à construire des dépôts (0–1) |
| `targetPriority` | `military` / `workers` / `buildings` |
| `harassEarly` | Harcèlement avant le premier assaut |

**Algorithme génétique** (après chaque partie) :
- Vainqueur : légère mutation (6%)
- Perdants : croisement avec le vainqueur + mutation croissante (17% / 24% / 31% selon le rang)

**Comportements IA** :
- Phases : `economy` → `expand` → `assault` → `pressure`
- Éclaireurs pour explorer la carte
- Alliances temporaires entre factions faibles contre le leader
- Adaptation de la composition (anti-tank / anti-infanterie)
- Construction automatique de modules et recherche du stimulant

## Ressources

- **Minéraux** : ressource principale (clusters de 6 cristaux × 2000 unités)
- **Vespène** : ressource secondaire (2 cristaux × 1500 par cluster)
- 8 clusters répartis sur la carte (4 aux bases, 4 au centre)

## Commandes de jeu

| Bouton | Action |
|--------|--------|
| `×½` à `×8` | Vitesse de simulation |
| `VIEW` | Changer la faction observée (brouillard de guerre) |
| `■ FIN` | Terminer la partie en cours |
| `📊 STATS` | Afficher les statistiques détaillées |
| `⚡ SIMULER ×100` | Lancer 100 parties en arrière-plan |
| `↺ Effacer apprentissage` | Réinitialiser la mémoire des agents |

## Lancer le jeu

```bash
node server.js
# Ouvrir http://localhost:3001
```

Ou via Claude Preview avec la config `AI Wars` (`.claude/launch.json`).
