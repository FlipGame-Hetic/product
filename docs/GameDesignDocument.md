# Game Design Document — Flipper Cyberpunk v1

---

## 1. Vision du jeu

**Flipper Cyberpunk** est un jeu de combat hybride basé sur des mécaniques de flipper classique, transposé dans un univers cyberpunk/hacking. Les joueurs s'affrontent en PvP ou progressent en PvE en utilisant des personnages aux capacités uniques, mêlant habileté au flipper et stratégie de combat.

---

## 2. Piliers du Game Design

1. **Skill-based** : Récompenser la maîtrise du flipper (timing, précision, combos)
2. **Combat asymétrique** : Chaque personnage possède un gameplay unique (1 bonus + 1 malus)
3. **Tension authentique** : Conservation du système de 3 billes pour maintenir l'intensité du flipper classique
4. **Stratégie** : Gestion des cooldowns, activation optimale des capacités au bon moment

---

## 3. Modes de jeu

### Mode Histoire (PvE)

- Progression solo contre des boss IA
- Déblocage de circuits, rampes et passages secrets
- Easter eggs cachés dans les niveaux

### Mode Versus (PvP)

- Affrontement 1v1 local (via système de room)
- Chaque joueur dispose de son propre client (flipper / web / mobile )
- **Conditions de victoire :**
  - Réduire les PV adverses à 0
  - Si les deux joueurs perdent leurs 3 billes : celui avec le plus de PV gagne
- Architecture serveur nécessaire pour la synchronisation

---

## 4. Système de Billes et Points de Vie

### Billes

- **3 billes par joueur** (en PvE et PvP)
- Préserve l'essence authentique du flipper
- Maintient la tension : chaque action compte

### Points de Vie (PV)

- Système de PV intégré au mode combat
- Les dégâts sont infligés via les capacités des personnages
- La perte d'une bille peut également impacter les PV (à définir)

---

## 5. Personnages

### Nombre

- **5 personnages** prévus au total
- Différenciation V1 : **changements de couleurs uniquement** (avant modèles 3D distincts)

### Structure des capacités

Chaque personnage possède :

- **1 Bonus** : Utilisable en PvE et PvP, aide à scorer ou à se protéger
- **1 Malus** : Inflige un handicap à l'adversaire (uniquement en PvP, disparaît en PvE)

### Inspirations et archétypes

- **Policier RoboCop** : Thématique lumière/protection, défensif
- **Judge Dredd** : "La loi, c'est moi" — justice implacable
- **Hacker cyberpunk** : Multiplicateurs, portails, manipulation du terrain
- **Cyborg agressif** : Thématique démoniaque, offensif (potentiellement légèrement plus fort)
- **Boss/Antagoniste GLaDOS** : Perturbe subtilement, décale la bille, "taunte" le joueur

**Note :** Les personnages définitifs restent à confirmer, mais la structure bonus/malus est validée.

---

## 6. Système de Capacités

### Activation

- **Barre de charge** à remplir en jouant bien
- Méthode principale : maintenir les deux flippers levés simultanément
- Relâcher les flippers = déclenchement de la capacité

### Cooldowns

- Chaque capacité possède un temps de rechargement
- Empêche le spam des capacités
- Force à choisir le bon moment d'activation

---

## 7. Bonus (Capacités Défensives/Utilitaires)

| Nom                          | Description                                            | Durée/Effet                 | Notes                                                                                            |
| ---------------------------- | ------------------------------------------------------ | --------------------------- | ------------------------------------------------------------------------------------------------ |
| **Bouclier**                 | Protection temporaire qui empêche la perte d'une bille | 10-30 secondes              | Disparaît dès qu'on perd une bille ; bloque l'accumulation d'énergie ultime pendant l'activation |
| **Ralentissement du temps**  | Ralentit la vitesse de la bille                        | ~ 5-10 secondes             | Associé au perso avec malus invisible pour équilibrer                                            |
| **Multiplicateur de combo**  | Multiplie les points marqués (x2 à x4)                 | Durée variable              | Mode "hacker" ; récompense le beau jeu (pour joueurs skillés type "Jett")                        |
| **Boost de dégâts/points**   | Augmentation basique des dégâts infligés               | Standard                    | Simple mais efficace                                                                             |
| **Flippers supplémentaires** | Ajoute 2 flippers (passe de 4 à 6)                     | Temporaire                  | Parallèle avec bioaugmentation cybernétique                                                      |
| **Portails/Téléportation**   | Place un checkpoint et peut revenir en arrière         | 2 charges : entrée + sortie | Conserve les points gagnés ; inspiration Portal/Apex Legends                                     |
| **Effet Freeze/Glace**       | Geler l'adversaire momentanément                       | Courte durée                | Effet glissement après (augmente vitesse) ; la bille s'arrête puis repart                        |
| **Bille supplémentaire**     | Ajoute une bille temporaire                            | Usage unique                | Bonus du flipper lui-même (pas un pouvoir de perso) ; doit être limité pour éviter abus          |

---

## 8. Malus (Capacités Offensives)

| Nom                      | Description                                      | Impact   | Notes                                                                   |
| ------------------------ | ------------------------------------------------ | -------- | ----------------------------------------------------------------------- |
| **Invisible/Intangible** | Rend la bille de l'adversaire invisible          | Fort     | Considéré très puissant/déséquilibré mais drôle ; nécessite équilibrage |
| **Tache d'encre**        | Obscurcit la vision de l'adversaire              | Moyen    | Style Mario Kart ; obstrue une partie de l'écran                        |
| **Réduction bumpers**    | Réduit la taille des bumpers adverses            | Moyen    | Rend le scoring plus difficile                                          |
| **Black hole**           | Spawn d'un trou à position aléatoire             | Fort     | Jugé trop punitif — à équilibrer ou retirer                             |
| **Modifier le rebond**   | Augmente ou rend aléatoire le rebond de la bille | Variable | Perturbe les trajectoires prévisibles                                   |
| **Bumpers collants**     | La bille colle aux bumpers                       | Moyen    | Nécessite de spammer les flippers pour décoller                         |

---

## 9. Mécaniques de Flipper Classiques

### Éléments de terrain

- **Circuits et rampes** : Débloquer des bonus en passant par certains chemins
- **Interrupteurs** : Activer 4 interrupteurs pour ouvrir des bonus spéciaux
- **Bumpers** : Zones de rebond pour scorer
- **Boucliers latéraux** : Protection temporaire des trous latéraux
- **Easter eggs** : Passages secrets et zones cachées

### Système de Combos

- Enchaînement de rampes ou de cibles pour multiplier les points
- Récompense la précision et la planification
- Fonctionne avec le bonus "Multiplicateur de combo"

---

## 10. Boss et Antagonistes

### Boss Principal Envisagé : GLaDOS

- **Comportement** : Perturbe subtilement le joueur avec des taunts
- **Mécanique** : Décale légèrement la trajectoire de la bille pour frustrer
- **Inspiration** : Portal

### Autres Antagonistes Potentiels

- **HAL 9000** (2001: L'Odyssée de l'espace)
- **AUTO** (Wall-E)
- Personnages de **Blade Runner**, **Ghost in the Shell**, **Akira**

---

## 11. Direction Artistique

### Univers

- **Thème principal** : Cyberpunk/Hacking
- Références : Blade Runner, Ghost in the Shell, Akira, Total Recall, RoboCop

### Esthétique

- **Néons** et **effets de lumière** : Ambiance cyberpunk premium
- **Particules** : Renforcer l'aspect futuriste
- Style visuel inspiré de **Octopath Traveler** pour les backgrounds (à confirmer)
- Possibilité de pixel art pour certains assets

---

## 12. Équilibrage et Méta

### Philosophie

- **Garder simple dans un premier temps** : Valider les mécaniques de base avant d'ajouter de la complexité
- **Sondage communautaire** : Prévu pour affiner les mécaniques et l'équilibrage
- **Différenciation progressive** : V1 en couleurs uniquement, modèles distincts plus tard

### Points d'attention

- **Risque de frustration pour débutants** : Un expert peut dominer un novice en 30 secondes
  - **Solution** : Passer par le mode histoire avant le PvP pour apprendre
- **Capacités déséquilibrées identifiées** :
  - Invisible : très puissant, nécessite équilibrage
  - Black hole : trop punitif, à revoir
  - Bouclier : risque de stalling si mal calibré

---

## 13. Contraintes Techniques Identifiées

### Architecture

- **Architecture serveur obligatoire** : Joueur 1 ↔ Serveur ↔ Joueur 2
- **Lag anticipé** : Nécessite optimisation réseau et fallback local

### Performance

- **Calcul de trajectoires** : Complexité pour certaines mécaniques (portails, téléportation)
- **Hardware minimum** :
  - **32 Go de RAM** (DDR4)
  - Focus sur la RAM plutôt que carte graphique
  - Jeu fonctionnant dans le navigateur

### Priorisation

- Certaines idées mises de côté pour raisons techniques (à réévaluer après POC)

---

## 14. Roadmap Game Design

| Phase           | Objectif                                                                                            |
| --------------- | --------------------------------------------------------------------------------------------------- |
| **V1 (MVP)**    | Mode solo fonctionnel, 1-2 personnages avec 1 bonus + 1 malus, flipper de base jouable              |
| **V2**          | 5 personnages, système de barre de charge finalisé, équilibrage bonus/malus                         |
| **V3**          | Mode PvP avec système de room, boss GLaDOS, easter eggs                                             |
| **V4 (Polish)** | Thème cyberpunk complet (néons, particules), mode histoire structuré, sondage communautaire intégré |

---

## 15. Questions Ouvertes

- [ ] La perte d'une bille impacte-t-elle directement les PV, ou seulement via les capacités ?
- [ ] Combien de PV par défaut pour chaque joueur ?
- [ ] Durées exactes des bonus/malus (nécessite playtesting)
- [ ] Le Multiball est-il un bonus de personnage ou un bonus global du flipper ?
- [ ] Comment afficher visuellement l'activation des capacités sans sortir le joueur de sa partie ?
- [ ] Faut-il un système de "super" (ultime) en plus du bonus/malus classique ?
- [ ] Répartition exacte des 3 écrans en mode PvP (vue joueur 1 / terrain central / vue joueur 2 ?)
- [ ] Quelle capacité pour quel personnage ? (attribution à définir après création des archétypes)

---

## 16. Prochaines Étapes

1. **Valider les 5 personnages définitifs** et leurs archétypes
2. **Attribuer 1 bonus + 1 malus** à chaque personnage

---

**Statut :** Version 1 — Document de référence pour le développement. À mettre à jour après chaque session de playtesting et validation d'équipe.

Maxime Bidan - V0.1.0 - 20/02/26
