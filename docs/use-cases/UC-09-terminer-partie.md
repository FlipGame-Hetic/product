# UC-09 : Terminer une partie

| ID    | Nom                 | Acteur  | But                                          |
| ----- | ------------------- | ------- | -------------------------------------------- |
| UC-09 | Terminer une partie | Système | Évaluer le résultat final et distribuer l'XP |

### UC-09 : Terminer une partie (détaillé)

**Acteurs :** Système

**Préconditions :** Une condition de fin de partie est remplie (dernier boss vaincu en PvE, ou vies épuisées pour tous les joueurs en PvP, ou Game Over)

**Scénario nominal :**

1. Le système détecte la condition de fin de partie
2. Le système calcule le score final, les dégâts infligés et l'XP à distribuer
3. Le système affiche l'écran de résultats sur le Backglass : score, XP gagné, résultat (victoire / défaite / Game Over)
4. Le DMD affiche un récapitulatif chiffré (score, niveau atteint)
5. Le système enregistre les données sur le serveur (score, XP, statistiques de la partie)
6. Après un délai ou une action du joueur, la borne repasse en mode veille/accueil

**Extensions :**

- 2a. En PvP, les résultats des deux joueurs sont réconciliés via le serveur avant affichage
- 5a. Le serveur est indisponible → les données sont conservées localement si possible, un message avertit le joueur que la progression n'a pas pu être sauvegardée
- 6a. Le joueur appuie sur "Start" pendant l'écran de résultats → la borne démarre une nouvelle session (UC-01)

**Postconditions :** La progression est sauvegardée, la borne est revenue en mode veille, prête pour une nouvelle partie

Arthur Jenck - v1.0.0 - 17/02/26
