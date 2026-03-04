# Game Design Document — AI Wars

## Unités

| Nom | Bâtiment producteur | Dégâts | Vitesse d'attaque | Portée | Blindé | Points de vie | Capacités spéciales |
|---|---|---|---|---|---|---|---|
| Worker | QG | 3 | ~1,1 s / attaque | 15 | Non | 40 | Récolte minéraux et vespène |
| Marine | Barracks | 12 | ~0,85 s / attaque | 35 | Non | 50 | **Stimulant** (×1,5 vitesse & att. pendant 15 s, coûte 33 % des PV actuels) ; **Kiting** (recul automatique après chaque tir) |
| Marauder | Barracks | 20 | ~1,35 s / attaque | 38 | Oui | 80 | **Stimulant** (×1,5 vitesse & att. pendant 15 s, coûte 33 % des PV actuels) |
| Tank | Factory | 20 (mobile) / 45 (siège, AOE r=30 px) | ~0,85 s (mobile) / 1,8 s (siège) | 55 (mobile) / 165 (siège) | Non | 220 | **Mode siège** : dégâts en zone, portée étendue ; ne bat pas en retraite sous PV critiques |
| Medivac | Starport | — | — | 100 (soin) | Non | 120 | Soigne les alliés (15 PV/s) ; **évacuation** des blessés critiques (< 25 % PV) vers le QG ; **transport tactique** (flanking / encerclement) : embarque 2-3 marines ou marauders en bonne santé et les dépose sur le flanc le moins défendu de l'ennemi |

---

## Bâtiments

| Nom | Label | Coût | Coût Vespène | Points de vie | Rôle |
|---|---|---|---|---|---|
| QG | HQ | — | — | 600 | Base principale ; produit les Workers |
| Dépôt | DEP | 100 | — | 300 | Augmente la capacité de stockage de ressources |
| Barracks | BAR | 150 | — | 200 | Produit Marines et Marauders ; accepte un module (Tech Lab ou Réacteur) |
| Factory | FAC | 150 | 75 | 250 | Produit les Tanks |
| Starport | STP | 100 | 100 | 200 | Produit les Medivacs |
| Tourelle | TUR | 75 | — | 150 | Défense statique (dégâts 18, portée 90, cd 1,4 s) |

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
