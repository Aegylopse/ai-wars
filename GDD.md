# Game Design Document — AI Wars

## Unités

| Nom | Bâtiment producteur | Dégâts | Vitesse d'attaque | Portée | Blindé | Points de vie | Capacités spéciales |
|---|---|---|---|---|---|---|---|
| Worker | QG | 3 | ~1,1 s / attaque | 5 (corps à corps) | Non | 40 | Récolte minéraux et vespène |
| Marine | Barracks | 10 | ~0,85 s / attaque | 35 | Non | 50 | **Stimulant** (×1,5 vitesse & att. pendant 15 s, coûte 33 % des PV actuels) ; **Kiting** (recul automatique après chaque tir) |
| Marauder | Barracks | 24 | ~1,35 s / attaque | 38 | Oui | 80 | **Stimulant** (×1,5 vitesse & att. pendant 15 s, coûte 33 % des PV actuels) |
| Tank | Factory | 20 (mobile) / 45 (siège, AOE r=30 px) | ~0,85 s (mobile) / 1,8 s (siège) | 55 (mobile) / 120 (siège) | Non | 220 | **Mode siège** : dégâts en zone, portée étendue ; ne bat pas en retraite sous PV critiques |
| Medivac | Starport | — | — | 100 (soin) | Non | 120 | Soigne les alliés (15 PV/s) ; **évacuation** des blessés critiques (< 25 % PV) vers le QG ; **transport tactique** (flanking / encerclement) : embarque 2-3 marines ou marauders en bonne santé et les dépose sur le flanc le moins défendu de l'ennemi |
| Viking | Starport | 14 (sol) / 18 (air) | ~0,85 s / attaque | 35 | Non | 80 | **Mode sol** (pentagone) : attaque uniquement les unités au sol ; **Mode air** (losange ✈) : attaque uniquement les unités aériennes ; bascule automatiquement selon les menaces visibles (3 s de transition) |
| Hélion | Factory | 8 (×2 vs unités légères) | ~0,55 s / attaque | 28 | Non | 90 | Lance-flamme en **cône (±40°)** devant l'unité touchant toutes les cibles terrestres dans l'angle ; **bonus ×2 vs unités légères** (marines, medivacs) ; rapide, prioritise les marines |

> **Unités légères** : Marine, Medivac

---

## Bâtiments

| Nom | Label | Coût | Coût Vespène | Points de vie | Rôle |
|---|---|---|---|---|---|
| QG | HQ | — | — | 600 | Base principale ; produit les Workers |
| Dépôt | DEP | 100 | — | 300 | Augmente la capacité de stockage de ressources |
| Barracks | BAR | 150 | — | 200 | Produit Marines et Marauders ; accepte un module (Tech Lab ou Réacteur) |
| Factory | FAC | 150 | 75 | 250 | Produit les Tanks et les Hélions |
| Starport | STP | 100 | 100 | 200 | Produit les Medivacs |
| Tourelle | TUR | 75 | — | 150 | Défense statique (dégâts 18, portée 120, cd 1,4 s) |

---

## Modules de Barracks

| Module | Effet |
|---|---|
| Tech Lab | Débloque la production de Marauders et la recherche du Stimulant |
| Réacteur | Permet la production de 2 Marines simultanément par cycle |

---

## Technologies

| Technologie | Bâtiment requis | Effet |
|---|---|---|
| Stimulant | Barracks + Tech Lab | Active le Stimulant sur Marines et Marauders |
