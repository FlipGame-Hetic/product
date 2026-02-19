# FlipGame — Backlog

> **Legendes priorites** : `P0` = bloquant / MVP core · `P1` = important pour l'experience · `P2` = nice-to-have
> **Legendes taille** : `XS` < 1j · `S` 1–2j · `M` 3–5j · `L` 6–10j
> **Owners** : `Frontend` · `Backend` · `IoT` · `DevOps` · `Equipe`

---

# MVP — Semaines 1 a 8

---

## Sprint S1–S2 — Cadrage, Setup & RnD (Semaines 1–2)

| #   | Tache                                                                                                                                                                                  | Priorite | Owner    | DoD                                                                                                                             | Dependances | Taille |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------- | ------------------------------------------------------------------------------------------------------------------------------- | ----------- | ------ |
| 1   | Initialiser les 3 repos GitHub (repo-iot, repo-back, monorepo-front) avec conventions de branches, PR templates et branch protection                                                   | P0       | DevOps   | 3 repos structures, README present sur chacun, branch protection activee, acces equipe configure                                | —           | XS     |
| 2   | Configurer CI/CD GitHub Actions sur chacun des 3 repos (lint, build, tests unitaires auto sur PR)                                                                                      | P0       | DevOps   | Pipeline vert sur une PR de test par repo, artefacts buildes                                                                    | #1          | M      |
| 3   | Definir et documenter les contrats d'interface (format MQTT topics, API WebSocket JSON schema)                                                                                         | P0       | Backend  | Document `contracts/` versionne dans les repos concernes, valide par Frontend et IoT                                            | —           | S      |
| 4   | Scaffolding repo-back : structure crates Rust, dependances Tokio/Axum/Redis, Swagger documente, docker-compose dev inclus                                                              | P0       | Backend  | Serveur Rust compile, `docker compose up` fonctionnel sur toutes les machines, Swagger accessible en local                      | #2          | M      |
| 5   | Scaffolding monorepo-front : deux apps (Backglass, Playfield) + lib partagee (Turborepo), React + R3F + Rapier.js + Zustand + TypeScript + Tailwind, deploiement temporaire sur Vercel | P0       | Frontend | Les deux apps buildent sans erreurs, Rapier.js initialise, lib partagee importable depuis chaque app, preview Vercel accessible | #2          | M      |
| 6   | Scaffolding repo-iot : firmware ESP32 de base, connexion Wi-Fi, publication MQTT premier message, procedure de flash documentee                                                        | P0       | IoT      | Message MQTT recu cote backend sur topic de test, flash reproductible par n'importe quel membre                                 | #3          | M      |
| 7   | RnD Game Design & Direction Artistique : brainstorming gameplay, regles du boss unique MVP, comportement de la bille, feedback visuels cibles, references DA cyberpunk                 | P0       | Equipe   | Document de game design MVP valide (plateau, 1 boss, 1 perso, mecanique d'ulti), moodboard DA produit                           | —           | L      |

> **Note** : La tache #7 est transversale et non assignee a un seul role. Elle conditionne les choix techniques des sprints suivants et doit etre finalisee avant la fin de S2.

---

## Sprint S3–S4 — POC & Integration physique (Semaines 3–4)

> **Note** : Le POC MQTT/WS est une validation technique rapide (1–2 jours en debut de S3). Le reste du sprint est consacre a l'integration et a la physique.

| #   | Tache                                                                                                                                                         | Priorite | Owner                    | DoD                                                                                               | Dependances | Taille |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------ | ------------------------------------------------------------------------------------------------- | ----------- | ------ |
| 8   | POC MQTT/WS : valider le pipeline complet ESP32 → MQTT → Rust → WebSocket → frontend sur un input et un affichage de test                                     | P0       | IoT + Backend + Frontend | Message envoye depuis l'ESP32 visible dans le rendu frontend en < 50ms, pipeline documente        | #4 #5 #6    | S      |
| 9   | Integration complete des inputs physiques : 2 boutons gauche, 2 boutons droite, 3 boutons verticaux, plunger solenoid, gyroscope (tilt + secousse), monnayeur | P0       | IoT                      | Chaque input genere un event MQTT distinct et correctement type, recu cote backend en < 30ms      | #6 #3       | L      |
| 10  | Moteur physique Rapier.js : bille, gravite, flippers articules, rebonds sur bumpers                                                                           | P0       | Frontend                 | Bille se comporte de facon realiste, les flippers repondent aux inputs clavier en attendant l'IoT | #5          | L      |
| 11  | Synchronisation 3 ecrans : detection des displays, routage Playfield / Backglass / DMD                                                                        | P0       | Frontend                 | Chaque ecran affiche le bon contenu sans conflit, teste sur 2 moniteurs minimum                   | #5          | M      |
| 12  | Bridge IoT → WS → moteur physique : les inputs physiques pilotent le rendu 3D en temps reel                                                                   | P0       | Frontend + Backend       | Appui sur un flipper physique fait bouger le flipper dans le rendu 3D                             | #9 #10 #11  | M      |
| 13  | Calibration gyroscope : seuils de tilt et secousse definis avec le game design, events MQTT distincts des autres inputs                                       | P1       | IoT                      | Tilt et secousse declenchent des events separes, seuils documentes                                | #9 #7       | S      |

---

## Sprint S5–S6 — Boucle PvE core (Semaines 5–6)

| #   | Tache                                                                                                | Priorite | Owner              | DoD                                                                                                      | Dependances | Taille |
| --- | ---------------------------------------------------------------------------------------------------- | -------- | ------------------ | -------------------------------------------------------------------------------------------------------- | ----------- | ------ |
| 14  | Monnayeur : insertion piece → demarrage session PvE                                                  | P0       | IoT + Backend      | Insertion piece → session initiee → frontend recoit l'event de demarrage                                 | #9          | S      |
| 15  | Ecran menu (idle) : affichage simple sur Backglass en attente de piece                               | P1       | Frontend           | Menu affiche en l'absence de session, disparait a l'insertion d'une piece                                | #11         | S      |
| 16  | Ecran choix de personnage : 1 personnage MVP, selection via boutons de navigation                    | P0       | Frontend           | Joueur confirme son choix en < 10s, ulti du personnage affiche, choix transmis au backend                | #12 #11     | M      |
| 17  | Systeme de vies : 3 vies par partie, detection perte de bille, affichage sur DMD                     | P0       | Frontend + Backend | Perte de bille → vie retiree → affichage mis a jour → game over si 0 vie                                 | #10 #12     | M      |
| 18  | Systeme de score : comptage en temps reel, affichage sur DMD, persistance localStorage               | P0       | Frontend           | Score s'incremente correctement, sauvegarde en localStorage a fin de session                             | #17         | M      |
| 19  | Boss unique MVP : HP bar, degats (points flipper → HP boss), conditions victoire, sprite placeholder | P0       | Frontend + Backend | Boss perd des HP proportionnellement au score, victoire declenchee quand HP = 0, affichage sur Backglass | #18 #7      | L      |
| 20  | Fin de partie MVP : ecran game over ou victoire, score final, retour menu idle                       | P0       | Frontend           | Resultat affiche, score sauvegarde en localStorage, retour idle apres delai ou clic                      | #18 #19     | S      |

---

## Sprint S7–S8 — Ulti, calibration & validation MVP (Semaines 7–8)

| #   | Tache                                                                                                             | Priorite | Owner          | DoD                                                                                     | Dependances   | Taille |
| --- | ----------------------------------------------------------------------------------------------------------------- | -------- | -------------- | --------------------------------------------------------------------------------------- | ------------- | ------ |
| 21  | Systeme d'ulti : jauge, detection combinaison boutons bas gauche + bas droite, declenchement et effet sur le boss | P0       | Frontend + IoT | Combinaison detectee → ulti declenche si jauge pleine → effet visible sur le boss       | #12 #16 #19   | M      |
| 22  | Gestion du tilt en jeu : penalite selon game design (vie ou score) + feedback visuel sur Backglass                | P1       | Frontend + IoT | Tilt detecte → penalite appliquee → feedback immediat                                   | #13 #17       | S      |
| 23  | Calibration finale Rapier.js : gravite, masse bille, restitution flippers, ajustements post-playtest              | P0       | Frontend       | Playtest de 30min sans comportement physique aberrant, parametres documentes            | #10           | M      |
| 24  | Playtests MVP : sessions completes de la boucle PvE, correction des bugs bloquants                                | P0       | Equipe         | Aucun bug P0 ouvert, boucle complete jouable de l'insertion de piece a la fin de partie | Toutes MVP P0 | M      |

---

# Post-MVP — Semaines 9 a 18

---

## Phase 1 Post-MVP — Infrastructure & Contenu (Semaines 9–12)

> Priorite : poser les fondations techniques stables et enrichir le contenu de jeu avant d'attaquer le multi.

| #   | Tache                                                                                                       | Priorite | Owner              | DoD                                                                                            | Dependances | Taille |
| --- | ----------------------------------------------------------------------------------------------------------- | -------- | ------------------ | ---------------------------------------------------------------------------------------------- | ----------- | ------ |
| 25  | Deploiement GCP : provisioning serveur, DNS, TLS, pipeline de deploiement continu                           | P0       | DevOps             | Serveur accessible publiquement en HTTPS, deploiement automatise depuis main                   | —           | M      |
| 26  | Migration localStorage → PostgreSQL : schema BDD (sessions, scores, joueurs), migrations auto               | P0       | Backend            | Migrations appliquees au demarrage, localStorage remplace par appels BDD, donnees persistantes | #25         | M      |
| 27  | RnD & creation d'assets : benchmark pipeline 3D/illustration/son, production des premiers assets definitifs | P0       | Equipe             | Pipeline de production d'assets defini, premiers livrables valides par l'equipe                | #7          | L      |
| 28  | Integration assets finales DA : plateau, boss 1, personnage 1, animations de base                           | P0       | Frontend           | Assets definitives integrees en remplacement des placeholders, sans regression sur les perfs   | #27         | L      |
| 29  | Boss 2 et Boss 3 : mecaniques differenciees, assets, conditions de victoire                                 | P0       | Frontend + Backend | Les 3 boss sont jouables a la suite avec des comportements distincts                           | #28         | L      |
| 30  | 5 personnages jouables : ajout des 4 personnages restants avec buff et malus distincts                      | P0       | Frontend + Backend | Chaque personnage modifie effectivement le gameplay, selectionnable depuis l'ecran de choix    | #28         | L      |
| 31  | Anti-cheat serveur : validation des scores cote Rust, detection de valeurs impossibles, logs                | P1       | Backend            | Score aberrant rejete, log d'anomalie enregistre, aucun impact sur le gameplay normal          | #26         | M      |

---

## Phase 2 Post-MVP — Multijoueur PvP (Semaines 12–16)

| #   | Tache                                                                                                     | Priorite | Owner              | DoD                                                                                           | Dependances | Taille |
| --- | --------------------------------------------------------------------------------------------------------- | -------- | ------------------ | --------------------------------------------------------------------------------------------- | ----------- | ------ |
| 32  | Lobby PvP : creation de room par code 4 chiffres, attente adversaire                                      | P0       | Backend            | Code genere → room creee → etat "waiting" visible sur frontend, expiration si pas de joueur 2 | #25 #26     | M      |
| 33  | Rejoindre une room : saisie code via boutons de navigation, connexion WS                                  | P0       | Frontend + Backend | Joueur 2 entre le code → rejoint la room → les 2 joueurs voient "Match ready"                 | #32         | M      |
| 34  | Match PvP temps reel : score J1 → degats J2 via WS + Redis pub-sub, affichage HP adversaire sur Backglass | P0       | Backend + Frontend | Score publie → adversaire recoit les degats en < 100ms, HP affiches en temps reel             | #33         | L      |
| 35  | Gestion fin de match PvP : gagnant/perdant sur les 2 bornes, gestion deconnexion et match nul             | P1       | Backend + Frontend | Resultat coherent affiche sur les 2 bornes en < 500ms apres l'event final                     | #34         | M      |
| 36  | Mode degrade reseau : basculement si latence WS > seuil defini, feedback utilisateur                      | P1       | Backend + Frontend | Latence > seuil → mode degrade active → UI informe le joueur, partie non interrompue          | #34         | M      |
| 37  | Tests de charge WS : simulation 2 clients simultanes, mesure latence p95                                  | P1       | DevOps + Backend   | p95 latence < 100ms sur GCP avec 2 sessions actives                                           | #34 #25     | M      |

---

## Phase 3 Post-MVP — Polish & Soutenance (Semaines 16–18)

| #   | Tache                                                                                                | Priorite | Owner    | DoD                                                                   | Dependances     | Taille |
| --- | ---------------------------------------------------------------------------------------------------- | -------- | -------- | --------------------------------------------------------------------- | --------------- | ------ |
| 38  | Optimisation rendering : 60fps stables sur les 3 ecrans en session complete                          | P1       | Frontend | 60fps maintenus sur les 3 ecrans pendant une session PvE complete     | Toutes frontend | M      |
| 39  | Sound design (si enceintes disponibles) : sons flippers, bumpers, boss, ulti (< 20ms de latence)     | P2       | Frontend | Sons declenches au bon moment sans impact sur les perfs               | #28             | M      |
| 40  | Animations Backglass : animations thematiques pour les evenements de jeu (boss attacks, transitions) | P2       | Frontend | Animations jouees aux bons moments sans regression perfs              | #28             | M      |
| 41  | Playtests finaux : 5 sessions PvE + 3 sessions PvP, correction bugs avant soutenance                 | P0       | Equipe   | Aucun bug P0 ouvert, liste P1/P2 connue et documentee                 | Toutes P0       | M      |
| 42  | Build de prod et validation borne : deploiement stable sur GCP, borne testee en conditions reelles   | P0       | DevOps   | Build stable deploye, borne physique fonctionnelle pour la soutenance | #41             | M      |

---

## Recapitulatif

| Phase                      | Taches P0 | Taches P1 | Taches P2 | Total  |
| -------------------------- | --------- | --------- | --------- | ------ |
| MVP (S1–S8)                | 16        | 5         | 0         | 21     |
| Post-MVP Phase 1 (S9–S12)  | 4         | 1         | 0         | 5      |
| Post-MVP Phase 2 (S12–S16) | 3         | 3         | 0         | 6      |
| Post-MVP Phase 3 (S16–S18) | 2         | 1         | 2         | 5      |
| **Total**                  | **25**    | **10**    | **2**     | **37** |

---

## Questions ouvertes (a trancher avant S5)

- **Penalite tilt** : perte de vie ou malus de score ? → impact sur #22
- **Condition regagner une vie en PvE** : prevu ou hors scope MVP ?
- **Combinaison exacte de l'ulti** : boutons bas gauche + bas droite a confirmer avec l'IoT → impact sur #21
- **Seuils gyroscope** : valeurs de tilt et secousse a calibrer pendant le POC → impact sur #13 et #22
- **Match nul PvP** : round supplementaire ou egalite acceptee ? → impact sur #35
- **Seuil mode degrade** : quelle latence WS declenche le basculement ? → impact sur #36 et #37

_Auteur : Arthur Jenck - v1.0.0 - 19/02/2026_
