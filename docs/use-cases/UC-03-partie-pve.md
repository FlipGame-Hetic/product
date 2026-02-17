# UC-03 : Lancer et jouer une partie PvE

| ID    | Nom                   | Acteur | But                                    |
| ----- | --------------------- | ------ | -------------------------------------- |
| UC-03 | Lancer une partie PvE | Joueur | Démarrer un run solo contre les 3 boss |

### UC-03 : Lancer et jouer une partie PvE (détaillé)

**Acteurs :** Joueur (via boutons de navigation, flippers, plunger, boutons d'ultime et gyroscope)

**Préconditions :** Le joueur a sélectionné son personnage (UC-02), le mode PvE est choisi, les 3 écrans sont synchronisés, le serveur est joignable

**Scénario nominal :**

1. Le joueur lance une nouvelle partie sur la borne (UC-01)
2. Le système initialise le run PvE : 3 vies, HP du premier boss, score à 0, barre d'ultime à 0 — l'état est affiché sur le DMD et le Backglass
3. Le joueur joue au flipper ; chaque contact avec les zones du plateau génère des points et inflige des dégâts au boss (certaines zones spéciales infligent des dégâts bonifiés)
4. La barre d'ultime se charge progressivement au fil des points marqués
5. Quand les HP du boss atteignent 0, le système affiche un écran de transition sur le Backglass puis initialise le boss suivant avec plus de HP et de nouveaux malus — répété pour les 3 boss
6. Une fois les 3 boss vaincus, le système affiche la victoire finale sur le Backglass et le DMD, puis calcule l'XP gagné

**Extensions :**

- 3a. Le joueur perd sa bille → UC-06 déclenché (perte de vie)
- 4a. La barre d'ultime est pleine → le joueur peut déclencher UC-05 (ultime)
- 5a. Le joueur agite la borne → UC-07 déclenché (tilt)
- 6a. Le joueur perd sa dernière vie avant de vaincre les 3 boss → "Game Over" affiché sur le DMD, le run se termine, XP partiel attribué

**Postconditions :** Score et XP enregistrés sur le serveur, la borne repasse en mode veille/accueil
