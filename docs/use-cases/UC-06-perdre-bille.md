# UC-06 : Perdre une bille

| ID    | Nom              | Acteur          | But                                                |
| ----- | ---------------- | --------------- | -------------------------------------------------- |
| UC-06 | Perdre une bille | Joueur, Système | Gérer la perte d'une bille et décrémenter les vies |

### UC-06 : Perdre une bille (détaillé)

**Acteurs :** Joueur, Système

**Préconditions :** Une partie est en cours, le joueur possède au moins 1 vie restante

**Scénario nominal :**

1. La bille passe entre les flippers et tombe dans la gouttière
2. Le système détecte la perte de bille
3. Le système décrémente le compteur de vies du joueur (affiché sur le DMD)
4. Si le joueur a encore des vies restantes, la bille est replacée dans le lanceur après un court délai
5. Le joueur reprend la partie depuis l'état actuel (HP du boss conservés, score conservé, barre d'ultime conservée)

**Extensions :**

- 3a. C'était la dernière vie du joueur → le système déclenche la séquence "Game Over" (affichée sur le DMD et le Backglass), la partie se termine
- 4a. En PvP, le joueur n'a plus de billes mais son adversaire en a encore → le joueur ne peut plus jouer, il attend la fin de partie (UC-08)
- 5a. Une désynchronisation des vies est détectée entre borne et serveur → le système resynchronise le compteur avant la reprise

**Postconditions :** Le compteur de vies est décrémenté, la bille est relancée ou la partie se termine selon les vies restantes
