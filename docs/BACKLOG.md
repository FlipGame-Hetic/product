# 🎰 FlipGame — Backlog MVP

> **Légende priorités** : `P0` = bloquant / MVP core · `P1` = important pour l'expérience · `P2` = nice-to-have
> **Légende taille** : `S` ≤ 2j · `M` 3–5j · `L` 6–10j
> **Owners** : `Frontend` · `Backend` · `IoT` · `DevOps`

---

## 🗓️ Sprint S1–S2 — Cadrage & Setup (Semaines 1–2)

| #   | Tâche                                                                                            | Priorité | Owner    | DoD                                                                        | Dépendances | Taille |
| --- | ------------------------------------------------------------------------------------------------ | -------- | -------- | -------------------------------------------------------------------------- | ----------- | ------ |
| 1   | Initialiser le monorepo (frontend, backend, iot) avec conventions de branches et PR templates    | P0       | DevOps   | Repo GitHub structuré, README principal présent, branch protection activée | —           | S      |
| 2   | Dockeriser l'environnement de dev (docker-compose avec Rust, Redis, PostgreSQL, Node)            | P0       | DevOps   | `docker compose up` fonctionnel sur toutes les machines de l'équipe        | #1          | M      |
| 3   | Configurer CI/CD GitHub Actions (lint, build, tests unitaires auto sur PR)                       | P0       | DevOps   | Pipeline vert sur une PR de test, artefacts buildés                        | #1 #2       | M      |
| 4   | Provisionner le serveur GCP (instance, DNS, certificats TLS, accès SSH équipe)                   | P0       | DevOps   | Serveur accessible via SSH par tous les membres, HTTPS opérationnel        | —           | M      |
| 5   | Définir et documenter les contrats d'interface (format MQTT topics, API WebSocket JSON schema)   | P0       | Backend  | Document `contracts/` versionné dans le repo, validé par Frontend et IoT   | —           | S      |
| 6   | Scaffolding du projet Rust (structure crates, dépendances Tokio/Axum/Redis, hello world déployé) | P0       | Backend  | Serveur Rust compile et répond sur GCP                                     | #2 #4       | M      |
| 7   | Scaffolding React + Three.js R3F + Rapier.js + Zustand + TypeScript (Vite, config Tailwind)      | P0       | Frontend | App vide build sans erreurs, Rapier.js importé et initialisé               | #2          | M      |
| 8   | Scaffolding ESP32 : firmware de base, connexion Wi-Fi, publication MQTT premier message          | P0       | IoT      | Message MQTT reçu côté backend sur topic de test                           | #5          | M      |

---

## 🗓️ Sprint S3–S4 — POC Technique (Semaines 3–4)

| #   | Tâche                                                                                            | Priorité | Owner              | DoD                                                                                     | Dépendances | Taille |
| --- | ------------------------------------------------------------------------------------------------ | -------- | ------------------ | --------------------------------------------------------------------------------------- | ----------- | ------ |
| 9   | POC physique Rapier.js : bille, gravité, flippers articulés, rebonds sur bumpers                 | P0       | Frontend           | Bille se comporte de façon réaliste, les flippers bougent en réponse à un input clavier | #7          | L      |
| 10  | POC WebSocket Rust : connexion client, envoi/réception de messages JSON, ping/pong               | P0       | Backend            | Client de test envoie un message et reçoit une réponse structurée en < 50ms local       | #6          | M      |
| 11  | POC MQTT IoT → Backend : lecture des 2 boutons flippers physiques et transmission en temps réel  | P0       | IoT                | Appui physique sur flipper gauche/droite → event reçu côté backend en < 30ms            | #8 #5       | M      |
| 12  | Intégration inputs physiques → moteur physique frontend (bridge IoT → WS → R3F)                  | P0       | Frontend + Backend | Appui sur flipper physique fait bouger le flipper dans le rendu 3D                      | #9 #10 #11  | L      |
| 13  | Setup PostgreSQL : schéma initial (sessions, scores, joueurs) + migrations automatisées          | P1       | Backend            | Migrations s'appliquent proprement au démarrage du container, schéma validé             | #2          | M      |
| 14  | Synchronisation 3 écrans : détection des displays, affichage ciblé (Playfield / Backglass / DMD) | P0       | Frontend           | Chaque écran affiche le bon contenu sans conflit, testé sur 2 moniteurs minimum         | #7          | M      |
| 15  | Calibration gyroscope ESP32 : détection de tilt (seuils à définir)                               | P1       | IoT                | Un tilt physique déclenche un event MQTT distinct des inputs flippers                   | #8          | S      |

---

## 🗓️ Sprint S5–S8 — Core PvE Loop (Semaines 5–8)

| #   | Tâche                                                                                          | Priorité | Owner              | DoD                                                                                    | Dépendances | Taille |
| --- | ---------------------------------------------------------------------------------------------- | -------- | ------------------ | -------------------------------------------------------------------------------------- | ----------- | ------ |
| 16  | Monnayeur : détection insertion pièce via ESP32 → démarrage session PvE                        | P0       | IoT + Backend      | Insertion pièce → session créée en BDD → frontend reçoit l'event de démarrage          | #11 #13     | M      |
| 17  | Écran menu principal (idle) : affichage sur Backglass, animation attract mode                  | P1       | Frontend           | Menu animé affiché en l'absence de session active, disparaît à l'insertion d'une pièce | #14         | M      |
| 18  | Écran choix de personnage : 4 personnages affichés, sélection via boutons de navigation        | P0       | Frontend           | Joueur peut sélectionner un personnage en < 10s, buff/malus affichés, choix confirmé   | #14 #12     | M      |
| 19  | Système de vies : 3 vies par partie, détection perte de bille, affichage sur DMD               | P0       | Frontend + Backend | Perte de bille → vie retirée → affichage mis à jour → game over si 0 vie               | #9 #12      | M      |
| 20  | Gestion du tilt en jeu : pénalité (perte de vie ou score) + animation Backglass                | P1       | Frontend + IoT     | Tilt détecté → pénalité appliquée → feedback visuel immédiat                           | #15 #19     | S      |
| 21  | Système de score : comptage en temps réel, affichage sur DMD, persistance en BDD               | P0       | Frontend + Backend | Score s'incrémente correctement, stocké en BDD à fin de session                        | #13 #19     | M      |
| 22  | Plunger : lecture analogique de la force d'éjection ESP32 → lancement de bille proportionnel   | P0       | IoT + Frontend     | Plunger tiré à 50% → bille éjectée à ~50% de la force max, testé sur 3 niveaux         | #12         | M      |
| 23  | Boss 1 — structure : HP bar, logique de dégâts (points flipper → HP boss), conditions victoire | P0       | Frontend + Backend | Boss 1 perd des HP proportionnellement au score du joueur, passage au boss 2 déclenché | #21         | L      |

---

## 🗓️ Sprint S9–S12 — Boss, Ultimes & PvP Core (Semaines 9–12)

| #   | Tâche                                                                                            | Priorité | Owner              | DoD                                                                                        | Dépendances | Taille |
| --- | ------------------------------------------------------------------------------------------------ | -------- | ------------------ | ------------------------------------------------------------------------------------------ | ----------- | ------ |
| 24  | Boss 2 & Boss 3 — implémentation complète (design mécanique différencié par boss)                | P1       | Frontend + Backend | Les 3 boss sont jouables à la suite, chacun avec un comportement distinct                  | #23         | L      |
| 25  | Système d'ultime : détection combinaison de boutons (haut gauche + haut droite), jauge d'ultime  | P0       | Frontend + IoT     | Combinaison détectée → ultime déclenché si jauge pleine → effet appliqué                   | #12 #18     | M      |
| 26  | Buffs/malus par personnage : implémentation des 4 modificateurs de gameplay                      | P1       | Frontend + Backend | Chaque personnage modifie effectivement le gameplay (vitesse, dégâts, etc.) selon sa fiche | #18 #23     | M      |
| 27  | Lobby PvP — création de room : génération code 4 chiffres, attente adversaire                    | P0       | Backend            | Code généré → room créée en Redis → état "waiting" visible sur frontend                    | #10 #13     | M      |
| 28  | Lobby PvP — rejoindre une room : saisie code via boutons de navigation, connexion WS             | P0       | Frontend + Backend | Joueur 2 entre code → rejoint la room → les 2 joueurs voient "Match ready"                 | #27 #18     | M      |
| 29  | Match PvP — synchronisation temps réel : score joueur → dégâts adversaire via WS + Redis pub-sub | P0       | Backend            | Score J1 publié → J2 reçoit les dégâts en < 100ms, testé avec latence réseau simulée       | #27 #10     | L      |
| 30  | Match PvP — affichage HP adversaire en temps réel sur le Backglass                               | P1       | Frontend           | HP de l'adversaire se mettent à jour en < 200ms, sans freeze du rendu 3D                   | #29 #14     | M      |
| 31  | Gestion déconnexion en PvP : forfait automatique si déconnexion > 5s                             | P1       | Backend            | Déconnexion simulée → l'adversaire gagne après timeout, état proprement nettoyé en Redis   | #29         | S      |

---

## 🗓️ Sprint S13–S16 — Finalisation Gameplay & Robustesse (Semaines 13–16)

| #   | Tâche                                                                                         | Priorité | Owner              | DoD                                                                                   | Dépendances | Taille |
| --- | --------------------------------------------------------------------------------------------- | -------- | ------------------ | ------------------------------------------------------------------------------------- | ----------- | ------ |
| 32  | Fin de partie PvE : écran de score final, leaderboard, retour menu                            | P1       | Frontend + Backend | Game over → score affiché sur DMD → top 5 scores chargés depuis BDD → retour idle     | #21 #17     | M      |
| 33  | Fin de partie PvP : affichage gagnant/perdant sur les 2 bornes simultanément                  | P1       | Frontend + Backend | Les 2 bornes affichent le résultat cohérent en < 500ms après l'événement final        | #29 #32     | M      |
| 34  | Mode dégradé réseau : basculement si latence WS > seuil (à définir) avec feedback utilisateur | P2       | Backend + Frontend | Latence > seuil → mode dégradé activé → UI prévient le joueur, partie non interrompue | #29         | M      |
| 35  | Anti-cheat serveur : validation des scores côté Rust (limites de score par cycle physique)    | P1       | Backend            | Score impossible (ex. +10 000 en 1 cycle) rejeté, log d'anomalie enregistré           | #21 #29     | M      |
| 36  | Tests de charge WebSocket : simulation 2 clients simultanés, mesure latence p95               | P1       | DevOps + Backend   | p95 latence < 100ms sur GCP avec 2 sessions simultanées, rapport de test produit      | #29 #4      | M      |
| 37  | Calibration finale Rapier.js : ajustement gravité, masse bille, restitution flippers          | P0       | Frontend           | Playtest de 30min sans comportement physique aberrant, paramètres documentés          | #9          | M      |
| 38  | Optimisation rendering 3 écrans : 60fps stable sur le hardware cible                          | P1       | Frontend           | 60fps maintenus sur les 3 écrans simultanément pendant une session PvE complète       | #14 #9      | M      |
| 39  | Gestion match nul PvP (round supplémentaire ou égalité — à trancher)                          | P2       | Backend + Frontend | Comportement défini et implémenté, testé sur scénario de score identique              | #33         | S      |

---

## 🗓️ Sprint S17–S18 — Polish & Soutenance (Semaines 17–18)

| #   | Tâche                                                                              | Priorité | Owner            | DoD                                                                                     | Dépendances  | Taille |
| --- | ---------------------------------------------------------------------------------- | -------- | ---------------- | --------------------------------------------------------------------------------------- | ------------ | ------ |
| 40  | Animations Backglass thème cyberpunk (boss attacks, transitions, idle)             | P2       | Frontend         | Animations jouées aux bons moments, sans impact sur les performances du Playfield       | #24          | M      |
| 41  | Sound design : sons flippers, bumpers, boss, ultime (Web Audio API ou fichiers)    | P2       | Frontend         | Sons déclenchés en < 20ms après l'event correspondant                                   | #23 #25      | M      |
| 42  | Playtests complets : 5 sessions PvE + 3 sessions PvP, correction bugs bloquants    | P0       | Toute l'équipe   | Aucun bug P0 ouvert, liste des bugs P1/P2 connue et priorisée                           | Toutes P0/P1 | L      |
| 43  | Documentation technique finale (architecture, API, MQTT topics, guide déploiement) | P1       | DevOps + Backend | Doc dans `/docs`, relue par 2 membres, build Docker reproductible depuis zéro documenté | —            | M      |
| 44  | Démo day : build de prod déployé sur GCP, borne testée en conditions réelles       | P0       | DevOps           | Build stable déployé, borne physique fonctionnelle pour la soutenance                   | #42 #43      | M      |

---

## 📊 Récapitulatif

| Priorité  | Nombre de tâches |
| --------- | ---------------- |
| **P0**    | 19               |
| **P1**    | 17               |
| **P2**    | 5                |
| **Total** | **41**           |

### Questions ouvertes à trancher avant S5

- **Regagner une vie en PvE** : score seuil, zone spéciale sur le plateau, ou boss vaincu ? → impact sur #19 et #23
- **Nombre de billes en PvP** : fixe (3) ou configurable à la création de la room ? → impact sur #27
- **Match nul PvP** : round supplémentaire ou égalité acceptée ? → impact sur #39
- **Seuil mode dégradé** : quelle latence WS déclenche le basculement ? → impact sur #34 et #36
