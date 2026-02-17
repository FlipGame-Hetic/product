# UC-08 : Jouer un match PvP

| ID    | Nom                | Acteur             | But                                                   |
| ----- | ------------------ | ------------------ | ----------------------------------------------------- |
| UC-08 | Jouer un match PvP | Joueur 1, Joueur 2 | S'affronter en temps réel, chaque joueur sur sa borne |

### UC-08 : Jouer un match PvP (détaillé)

**Acteurs :** Joueur 1 (Borne A), Joueur 2 (Borne B sur Ordinateur)

**Préconditions :** Les deux bornes sont appairées sur la même session PvP, chaque joueur a sélectionné son personnage (UC-02), le serveur distant valide la session

**Scénario nominal :**

1. Le serveur synchronise les deux bornes et affiche un compte à rebours de départ sur les deux DMD
2. Les deux joueurs lancent leur partie locale puis tirent leur bille simultanément via leur plunger respectif (UC-01)
3. Chaque joueur joue sur sa propre borne ; les points marqués sont convertis en dégâts infligés à la barre de vie de l'adversaire
4. Le serveur centralise et redistribue l'état en temps réel via WebSocket : HP de chaque joueur, score, événements d'ultime
5. Chaque borne affiche les HP de l'adversaire en temps réel sur son DMD et Backglass
6. Un joueur peut déclencher son ultime (UC-05) pour infliger un burst de dégâts à l'adversaire
7. Quand un joueur perd une bille, son compteur de vies décrémente (UC-06) ; quand il n'a plus de vies, il ne peut plus jouer
8. La partie se termine quand les deux joueurs ont épuisé leurs vies ; le vainqueur est celui dont la barre de vie adverse est la plus basse
9. Le résultat est affiché simultanément sur les deux bornes, l'XP est distribué (UC-09)

**Extensions :**

- 4a. Latence réseau trop élevée → mode dégradé, synchronisation ralentie, joueurs avertis sur leur DMD
- 7a. Un joueur provoque un tilt → UC-07 déclenché, la bille est perdue immédiatement
- 8a. Les deux joueurs terminent avec des barres de vie adverses identiques → match nul (**question ouverte** : round supplémentaire ou égalité acceptée ?)
- 9a. Un joueur déconnecte en cours de partie → forfait déclaré après un délai de grâce, l'adversaire est déclaré vainqueur

**Postconditions :** Résultats enregistrés sur le serveur, les deux bornes retournent à l'écran d'accueil de façon synchronisée

Arthur Jenck - v1.0.0 - 17/02/26
