# Cahier des Charges Technique — Flipper

---

## 1. Vision

**Flipper** est un jeu de pinball virtuel multijoueur au thème cyberpunk intégré dans une borne physique où l'objectif est de proposer une expérience arcade moderne alliant jeu combat et mécanique de flipper classique en proposant à la fois du PvE et du PvP.

---

## 2. Objectifs et périmètre

### Objectifs

1. Simuler une table de flipper avec une physique réaliste (balle, flippers, rebonds, gravité)
2. Intégrer 4 personnages jouables avec chacun un buff et un malus activables
3. Synchroniser 3 écrans en temps réel via WebSocket
4. Permettre le contrôle via les inputs physiques et virtuel (pour la partie web)
5. Mise en place d'un sounds design immercif
6. Intégration de l'IA (analyse, aide, taunt)
7. Proposer un mode PvP avec système de room par code

### Non-objectifs

1. Pas de matchmaking en ligne
2. Pas d'éditeur de table personnalisé
3. Pas de Bot qui joue contre le joueur

### Personas

**Lucas**, 20 ans, étudiant — il veut tester la borne avec ses amis en local, sans lire de notice. Il s'attend à jouer en moins de 10 secondes après avoir appuyé sur le premier bouton.

**Camille**, 22 ans, fan d'esport, ce qui l'interesse c'est la compétition. Elle veux une experience de combat équitable, sans lag et pouvoir facilement lancé un combat contre sont adversaire.

---

## 3. Use Cases

Une série de cas d'usage anticipés a été ajoutée et est disponible dans le dossiers `/docs/use-cases`. La liste n'est pas exhaustive, de futurs contraintes liées aux spécifications techniques pourront apparaître. D'autres cas seront ajoutés au fur et à mesure.

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

| Composant                | Techno                      | Justification                                                                                                                                           |
| ------------------------ | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rendu / Physique         | Three.js (R3F) + Rapier.js  | Simulation physique déterministe côté client en temps réel                                                                                              |
| UI / Style               | TailwindCSS                 | Permet d’itérer vite sur l’interface borne et UI                                                                                                        |
| État global              | Zustand                     | Synchronisation simple sans re-render inutiles                                                                                                          |
| Typage                   | TypeScript                  | Typage statique pour la sécurité et la lisibilité du code                                                                                               |
| Framework front          | React                       | Découplage clair entre rendu du monde et overlay UI (scores, effets, feedback joueur)                                                                   |
| Backend / Logique métier | Rust                        | Serveur authoritative (gestion des ticks) capable de simuler la partie sans jitter ni pauses, avantage de la performance par rapport à un autre langage |
| Protocole IoT            | MQTT                        | Protocole léger publish/subscribe pour l'ESP32                                                                                                          |
| IOT/ESP32                | C++                         | Rapidité et stabilité du C++ et de ses libs                                                                                                             |
| Temps réel front         | WebSocket                   | Transmission au serveur vers le renderer minimale                                                                                                       |
| Cache / Pub-Sub          | Redis                       | Relais interne entre instances de simulation pour le 1v1 (multiplayer) et le cache des données                                                          |
| Persistance              | PostgreSQL                  | Stockage fiable des scores, parties et stats compétitives                                                                                               |
| Déploiement / DevOps     | Docker                      | Reproduire localement la stack réseau complète du flipper jusqu’au serveur multi                                                                        |
| CI/CD                    | GitHub Actions              | Automatisation des tests et déploiements pour garantir la qualité et la stability du code                                                               |
| Déploiement / VPS        | Google Cloud Platform (GCP) | Avoir un contrôle sur les déploiements et les ressources disponibles, faciliter le scalling et la gestion du multijoueur                                |
| CND / Bucket             | S3 storage / CDN            | Hébergement des assets statiques (build front, textures, sons) avec CDN pour réduire la latence et améliorer les performances en multijoueur            |

**Alternatives écartées :**

- `Socket.io` => trop lourd, la lib `ws` native suffit pour notre besoin
- `Unity` => pas web-natif, incompatible avec notre stack 3 écrans/browser
- `Cannon.js` => plus leger que Rapier.js mais moin performant.
- `BLE` pour l'ESP32 =>latence trop variable, Serial/GPIO plus fiable en local
- `Railway` => peut être instable, facile à mettre en place mais difficilement scalable par rapport à GCP
- `Supabase` => Facile à mettre en place mais pas adpater à nos besoin.
- `Python`=> langage accessible, syntaxe simple, mais pas adapter à un besoin de performance élevée et de contexte d'iot en temps réel
- `C` => langage plus complexe et temps de développement accru pour résultat équivalent voire meilleur

---

## 7. Risques et contraintes

### Contraintes

| Risque                                                     | Probabilité | Impact | Mitigation                                                            |
| ---------------------------------------------------------- | ----------- | ------ | --------------------------------------------------------------------- |
| Latence WebSocket trop élevée en PvP (2 bornes distantes)  | Haute       | Fort   | POC réseau S3-S4, seuil de tolérance défini                           |
| Physique Rapier.js difficile à calibrer (rebonds, gravité) | Haute       | Fort   | Tests dès S3, paramètres ajustables via config, itérations courtes    |
| Désynchronisation des 3 écrans en cours de partie          | Moyenne     | Fort   | Serveur autoritatif en Rust, état de jeu unique, resync automatique   |
| Gyroscope : faux positifs de tilt                          | Haute       | Moyen  | Seuil de sensibilité calibrable, filtre logiciel côté ESP32           |
| Panne de GCP                                               | Faible      | Fort   | Switch vers une backup local                                          |
| IA trop faible/forte                                       | Moyenne     | Fort   | Entraîner l'IA avec de vrais joueurs                                  |
| Monnayeur physique non détecté (défaut matériel)           | Moyenne     | Moyen  | Tests matériels dès S3, procédure de fallback manuel                  |
| Surcharge du serveur GCP en pic de charge PvP              | Faible      | Fort   | Redis en pub-sub, architecture stateless, load testing avant release  |
| Rust : courbe d'apprentissage                              | Haute       | Moyen  | Remise à niveau                                                       |
| Panne d'un écran pendant une démo                          | Faible      | Fort   | Tests de reconnexion automatique, prévoir écran de remplacement       |
| Intégration MQTT ↔ WebSocket plus complexe que prévue      | Moyenne     | Moyen  | POC bridge MQTT/WS dès S3, découplage strict des responsabilités      |

### Contraintes

- **Délai :** 18 semaines (soutenance finale S19)
- **Équipe :** 5 personnes (frontend, backend, IoT, DevOps, design)
- **Matériel :** 1 borne physique avec 3 écrans intégrés, 1 ESP32, 1 gyroscope, 1 monnayeur physique, breadboard et composants électroniques
- **Infrastructure :** 1 VPS GCP géré par le DevOps de l'équipe
- **Budget matériel :** à définir
- **Contrainte réseau :** le PvP nécessite que les deux bornes soient connectées au même serveur distant — une coupure réseau met fin à la session
- **Contrainte physique :** la borne est un équipement unique partagé — les tests IoT et hardware ne peuvent pas se faire en parallèle du développement front

---

## 8. Conventions équipe

### Git

```markdown
- Branches : <issueId>-<type><scope>-<name> (ex: 18-featplayfield-implement physics engine) à noter, un / peut être ajouté entre le <type> et le <scope>
- Main ne peut être supprimé
- Commits : Conventional Commits with scope (feat/backscreen:, fix/DMD:, docs/use-cases:)
- PR obligatoire + 1 review minimum, nouvelle review si commit plus récent, push direct sur main bloqué par ruleset même avec -force
- Conflits : rebase sur main, resolution en binome
- Langue : l'anglais est utilisé partout, excepté dans la documentation et les Discussions GitHub de l'organisation
```

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

Maxime Bidan - Arnaud Fischer - Louis Dondey - Arthure Jenck - Alexis Gontier - v1.0.0 - 17/02/26
