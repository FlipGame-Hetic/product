# UC-07 : Provoquer un tilt

| ID    | Nom               | Acteur          | But                                                             |
| ----- | ----------------- | --------------- | --------------------------------------------------------------- |
| UC-07 | Provoquer un tilt | Joueur, Système | Détecter une secousse via le gyroscope et sanctionner le joueur |

### UC-07 : Provoquer un tilt (détaillé)

**Acteurs :** Joueur (involontairement ou intentionnellement), Système (via le gyroscope de la borne)

**Préconditions :** Une partie est en cours, le gyroscope est actif et calibré

**Scénario nominal :**

1. Le joueur secoue ou pousse physiquement la borne au-delà du seuil de tolérance défini
2. Le gyroscope détecte le mouvement et envoie le signal au système
3. Le système déclenche immédiatement la séquence de tilt : "TILT" affiché sur le DMD, animation sur le Backglass
4. La bille en jeu est considérée comme perdue
5. UC-06 est déclenché (perte de bille / décrémentation des vies)

**Extensions :**

- 1a. Le mouvement est en dessous du seuil → avertissement "DANGER" affiché sur le DMD sans pénalité (comportement classique des vrais flippers)
- 2a. Le gyroscope détecte un faux positif (vibration externe) → le système applique quand même la pénalité (contrainte matérielle, à améliorer si possible)
- 5a. C'était la dernière vie → Game Over déclenché immédiatement

**Postconditions :** La bille est perdue, le compteur de vies est décrémenté, la partie continue ou se termine selon les vies restantes
