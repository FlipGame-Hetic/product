# UC-05 : Activer l'ultime

| ID    | Nom              | Acteur | But                                                       |
| ----- | ---------------- | ------ | --------------------------------------------------------- |
| UC-05 | Activer l'ultime | Joueur | Déclencher la capacité spéciale une fois la barre chargée |

### UC-05 : Activer l'ultime (détaillé)

**Acteurs :** Joueur (via la combinaison de boutons dédiée sur la borne)

**Préconditions :** Une partie est en cours (PvE ou PvP), la barre d'ultime du joueur est pleine

**Scénario nominal :**

1. Le système indique que la barre d'ultime est pleine via une animation sur le Backglass et le DMD
2. Le joueur appuie simultanément sur les boutons d'ultime (bouton haut gauche + bouton haut droite)
3. Le système valide la combinaison et déclenche l'ultime du personnage sélectionné
4. Une animation dans le thème cyberpunk est jouée sur le Backglass
5. L'ultime inflige une rafale de dégâts massifs à la cible (boss en PvE, adversaire en PvP)
6. La barre d'ultime est remise à 0 après l'activation

**Extensions :**

- 2a. Le joueur appuie sur la combinaison avec une barre incomplète → aucune action, la combinaison est ignorée, aucun retour visuel perturbant
- 3a. En PvP, l'adversaire reçoit les dégâts en temps réel via le serveur → si la latence est trop élevée, les dégâts sont appliqués avec un léger délai et un indicateur visuel avertit les deux joueurs
- 5a. La cible est déjà à 0 HP au moment de l'impact (mort simultanée) → les dégâts excédentaires sont ignorés

**Postconditions :** La barre d'ultime est vide, les dégâts sont appliqués à la cible, le jeu reprend normalement
