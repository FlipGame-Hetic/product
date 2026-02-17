# Cahier des Charges Technique — Flipper

---

## 1. Vision

**Flipper** est un jeu de pinball virtuel multijoueur au thème cyberpunk intégré dans une borne physique, l'objectif est de proposer une expérience arcade moderne alliant jeu combat et mécanique de flipper classique en proposant à la fois du PvE et du PvP.

---

## 2. Objectifs et périmètre

### Objectifs

1. Simuler une table de flipper avec physique réaliste (balle, flippers, rebonds, gravité)
2. Intégrer 4 personnages jouables avec chacun un buff et un malus activables
3. Synchroniser 3 écrans en temps réel via WebSocket
4. Permettre le contrôle via les capteurs physiques (ESP32/Raspberry Pi) et le web
5. Proposer un mode PvP local avec système de room par code

### Non-objectifs

1. Pas de matchmaking en ligne avec des inconnus
2. Pas d'éditeur de table personnalisé
3. Bot qui joue contre le joueur

### Personas

**Lucas**, 20 ans, étudiant — il veut tester la borne avec ses amis, sans lire de notice. Il s'attend à jouer en moins de 10 secondes après avoir appuyé sur le premier bouton.

**Camille**, 22 ans, étudiante dev web — elle contribue au projet, elle veut une architecture claire et un environnement de dev bien configuré pour ne pas perdre de temps.

---

## 3. Use Cases

| ID    | Nom               | Acteur | But                                      |
| ----- | ----------------- | ------ | ---------------------------------------- |
| UC-01 | Lancer une partie | Joueur | Démarrer une nouvelle partie solo ou PvP |

### UC-01 : Lancer une partie (détaillé)

**Acteurs :** Joueur (via contrôleurs physiques ou web)  
**Préconditions :** Serveur WS démarré, au moins 1 écran connecté, personnage sélectionné  
**Scénario nominal :**

1. Le joueur appuie sur le bouton de démarrage
2. Le système affiche un compte à rebours (3, 2, 1)
3. Le score est réinitialisé à 0
4. La bille est placée dans le lanceur
5. La partie commence

**Extensions :**

- 1a. Partie en cours → confirmation demandée avant réinitialisation
- 4a. Connexion ESP32 perdue → fallback clavier web activé

**Postconditions :** Une partie est en cours, les scores sont trackés en temps réel

---

## 4. Architecture technique

**Flux de données :**

- Les inputs physiques (flippers, boutons de capacité) sont captés par l'ESP32 et transmis au Raspberry Pi via Serial
- Le serveur Node.js gère la logique de jeu et diffuse l'état en temps réel aux 3 écrans via WebSocket
- Chaque écran exécute un client Three.js/Cannon.js qui rend la scène en fonction de son rôle (vue joueur, vue terrain)

---

## 5. Diagrammes UML

---

## 6. Stack technique
| Composant                | Techno                     | Justification                                                      |
| ------------------------ | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rendu / Physique         | Three.js (R3F) + Rapier.js | Simulation physique déterministe côté client en temps réel |
| UI / Style               | TailwindCSS                | Permet d’itérer vite sur l’interface borne et UI                                      |
| État global              | Zustand                    | Synchronisation simple sans re-render inutiles                                                        |
| Typage                   | TypeScript                 | Typage statique pour la sécurité et la lisibilité du code                                            |
| Framework front          | React                      | Découplage clair entre rendu du monde et overlay UI (scores, effets, feedback joueur)                                                                 |
| Backend / Logique métier | Rust                       | Serveur authoritative (gestion des ticks) capable de simuler la partie sans jitter ni pauses, avantage de la performance par rapport à un autre langage|
| Protocole IoT            | MQTT                       | Absorbe le bruit matériel des capteurs physiques et normalise les événements venant de l’ESP32                                                        |
| Temps réel front         | WebSocket                  | Transmission au serveur vers le renderer minimale                                                                  |
| Cache / Pub-Sub          | Redis                      | Relais interne entre instances de simulation pour le 1v1 (multiplayer) et le cache des données                                                         |
| Persistance              | PostgreSQL                 | Stockage fiable des scores, parties et stats compétitives                                                                                             |
| Déploiement / DevOps     | Docker                     | Reproduire localement la stack réseau complète du flipper jusqu’au serveur multi                                                                      |
| CI/CD                    | GitHub Actions             | Automatisation des tests et déploiements pour garantir la qualité et la stability du code                                                              |
| Déploiement / VPS    | Google Cloud Platform (GCP) | Avoir un contrôle sur les déploiements et les ressources disponibles, faciliter le scalling et la gestion du multijoueur                                     |


**Alternatives écartées :**

- `Socket.io` => trop lourd, la lib `ws` native suffit pour notre besoin
- `Unity` => pas web-natif, incompatible avec notre stack 3 écrans/browser
- `Cannon.js` => plus leger que Rapier.js mais moin performant.
- `BLE` pour l'ESP32 =>latence trop variable, Serial/GPIO plus fiable en local
- `Railway` => peut être instable, facile à mettre en place mais difficilement scalable par rapport à GCP
- `Supabase` => Facile à mettre en place mais pas adpater à nos besoin.
- `Python`=> langage accessible, syntaxe simple, mais pas adapter à un besoin de performance élevée et de contexte d'iot en temps réel
---

## 7. Risques et contraintes

### Contraintes

---

## 8. Conventions équipe

### Git

### Code

---

## 9. Roadmap et questions ouvertes

| Phase       | Semaines | Objectif                                                        |
| ----------- | -------- | --------------------------------------------------------------- |
| CDC + Setup | S1       | CDC validé,repos prêt                                           |
| POC         | S2-S3    | Flipper jouable (1 écran, physique de base, 1 flipper physique) |
| MVP         | S4-S7    | 3 écrans sync, 4 persos, buff/malus, mode PvP room              |
| Polish      | S8-S9    | Thème cyberpunk, effets néons/particules, TTP optimisé          |
| Demo        | S10      | Intégration finale, présentation, tag `v1.0.0`                  |

### Questions ouvertes

- [ ] Comment afficher les instructions de capacité sans sortir le joueur de sa partie ?
- [ ] Faut-il un système de compte / login pour le mode PvP, ou juste un code de room suffisamment court ?
- [ ] Gestion de l'auth ws / bridge interne à mqtt ou personalisé ?
