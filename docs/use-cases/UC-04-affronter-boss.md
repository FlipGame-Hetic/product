# UC-04 : Affronter un boss

| ID    | Nom               | Acteur          | But                                                           |
| ----- | ----------------- | --------------- | ------------------------------------------------------------- |
| UC-04 | Affronter un boss | Joueur, Système | Infliger des dégâts au boss via le flipper jusqu'à sa défaite |

### UC-04 : Affronter un boss (détaillé)

**Acteurs :** Joueur (via les flippers, plunger et boutons de la borne), Système (gestion des HP, des dégâts et des malus)

**Préconditions :** Une partie PvE est en cours (UC-03), le boss courant est initialisé avec ses HP et ses malus disponibles

**Scénario nominal :**

1. Le système affiche les HP du boss sur le Backglass et le DMD
2. Le joueur joue au flipper ; la bille entre en contact avec les différentes zones du plateau
3. Chaque zone touchée génère un nombre de points correspondant, convertis en dégâts infligés au boss
4. Les zones spéciales du plateau infligent des dégâts bonifiés au boss et sont mises en évidence visuellement sur le Playfield
5. Le boss réagit aux dégâts reçus : animations sur le Backglass, effets visuels sur le Playfield
6. Quand les HP du boss atteignent 0, le système déclenche une animation de défaite sur le Backglass et valide le passage au boss suivant (UC-03, étape 6)

**Extensions :**

- 2a. Le joueur perd sa bille en cours de combat → UC-06 déclenché, le boss conserve ses HP actuels
- 3a. Le joueur active son ultime (UC-05) → les dégâts du prochain contact sont massivement augmentés
- 5a. Le boss applique un malus au joueur (boss 2 et 3) → le système modifie temporairement le comportement du plateau (ex. flippers ralentis, zones neutralisées)

**Postconditions :** Le boss est vaincu, ses HP sont à 0, le système prépare la transition vers le boss suivant ou la victoire finale

Arthur Jenck - v1.0.0 - 17/02/26
