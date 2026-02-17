# UC-01 : Démarrer une nouvelle partie

| ID    | Nom                          | Acteur          | But                                                        |
| ----- | ---------------------------- | --------------- | ---------------------------------------------------------- |
| UC-01 | Démarrer une nouvelle partie | Joueur, Système | Initialiser une session de jeu et lancer la première bille |

### UC-01 : Démarrer une nouvelle partie (détaillé)

**Acteurs :** Joueur (via sélecion dans le menu et plunger), Système

**Préconditions :** La borne est sur l'écran d'accueil ou le menu principal, le serveur de jeu est joignable

**Scénario nominal :**

1. Le joueur appuie sur le bouton "Start" pour lancer une nouvelle session
2. Le système réinitialise l'état de partie : score à 0, nombre de vies par défaut, jauges remises à l'état initial
3. Le système prépare l'aire de jeu et place la bille dans le lanceur
4. Le joueur tire le plunger pour envoyer la première bille
5. Le système passe en état "partie en cours" et commence le suivi des événements de jeu

**Extensions :**

- 1a. Une partie est déjà active → le système demande confirmation avant réinitialisation
- 4a. Le joueur n'actionne pas le plunger dans le délai attendu → un rappel visuel/sonore est affiché, la partie reste en attente

**Postconditions :** Une nouvelle partie est active, la première bille est lancée, les compteurs sont initialisés

Arthur Jenck - v1.0.0 - 17/02/26
