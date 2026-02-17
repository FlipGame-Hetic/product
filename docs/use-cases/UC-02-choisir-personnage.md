# UC-02 : Choisir un personnage

| ID    | Nom                   | Acteur | But                                         |
| ----- | --------------------- | ------ | ------------------------------------------- |
| UC-02 | Choisir un personnage | Joueur | Sélectionner un personnage avant une partie |

### UC-02 : Choisir un personnage (détaillé)

**Acteurs :** Joueur (via les boutons de navigation de la borne)

**Préconditions :** Le joueur est sur l'écran de sélection avant le lancement d'une partie, aucune partie n'est active

**Scénario nominal :**

1. Le système affiche l'écran de sélection de personnage sur le Backglass avec les personnages disponibles et leurs capacités
2. Le joueur utilise les boutons de navigation pour parcourir les personnages
3. Pour chaque personnage mis en avant, le système affiche ses statistiques et sa capacité spéciale (ultime) sur le Backglass
4. Le joueur valide son choix avec le bouton de confirmation
5. Le système enregistre le personnage sélectionné et redirige vers le mode de jeu correspondant

**Extensions :**

- 2a. Un seul personnage est disponible → la sélection est ignorée, le personnage est attribué automatiquement

**Postconditions :** Le personnage est sélectionné, ses statistiques et son ultime sont chargés pour le lancement de la partie (UC-01)

Arthur Jenck - v1.0.0 - 17/02/26
