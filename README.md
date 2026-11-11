# MeteoSphere

MeteoSphere est une plateforme météo intelligente en temps réel. Elle collecte les données de stations, de capteurs et d'APIs, produit des prévisions, calcule les risques naturels, envoie des alertes et permet aux utilisateurs de signaler la météo réelle.

## Problématique

Les données météo sont dispersées, peu fiables à l'échelle locale et mal exploitées pour l'alerte. MeteoSphere les réunit sur une seule plateforme et les transforme en informations utiles : prévisions, niveaux de risque par zone et notifications.

## Architecture

La plateforme repose sur 7 microservices (6 en Spring Boot, 1 en Django), une API Gateway, RabbitMQ, Redis et un frontend Angular avec une carte Leaflet.

Le diagramme global se trouve dans [Documentation/Diagrams/MeteoSphere-architecture-globale.drawio](Documentation/Diagrams/MeteoSphere-architecture-globale.drawio). Pour l'ouvrir, aller sur [app.diagrams.net](https://app.diagrams.net/), puis choisir **Fichier > Ouvrir depuis > Appareil** et sélectionner ce fichier.

### Flux global

```text
APIs externes ──► Ingestion ──┐
                              ├──► RabbitMQ ──► Prévisions ──► Risques ──► Alertes ──► Notifications
Stations IoT ─────────────────┤                     │            │
Signalements ─────────────────┘                     └────────────┴──► API Gateway ──► Frontend

                         Auth (Django) : JWT valide pour tous les services
```

### Microservices

| # | Microservice | Framework | Base de données | Responsable | Port prévu |
|---|---|---|---|---|---|
| 1 | Auth & Utilisateurs (commun) | Django REST | PostgreSQL | Toute l'équipe | 8081 |
| 2 | Ingestion des données météo | Spring Boot + Scheduler | MongoDB | Membre 1 | 8082 |
| 3 | Prévisions météo | Spring Boot + JPA | PostgreSQL | Membre 2 | 8083 |
| 4 | Alertes & Notifications | Spring Boot + Spring Mail/AMQP | MySQL | Membre 3 | 8084 |
| 5 | Stations & Capteurs IoT | Spring Boot WebFlux + MQTT | MongoDB | Membre 4 | 8085 |
| 6 | Risques & Catastrophes naturelles | Spring Boot + Spring Batch | PostgreSQL | Membre 5 | 8086 |
| 7 | Signalements | Spring Boot + WebSocket | H2 | Membre 6 | 8087 |

### Infrastructure

| Composant | Rôle | Port prévu |
|---|---|---|
| API Gateway (Spring Cloud Gateway) | Point d'entrée unique du frontend | 8080 |
| Eureka Server | Annuaire des services | 8761 |
| Config Server | Configuration centralisée depuis `config-repo` | 8888 |
| RabbitMQ | Événements asynchrones, exchange `meteosphere.events` | 5672 (UI 15672) |
| Redis | Cache des prévisions et de la carte des risques | 6379 |

### Bases de données

- PostgreSQL ×3 : Auth, Prévisions, Risques.
- MySQL ×1 : Alertes.
- H2 ×1 : Signalements (base embarquée dans le service).
- MongoDB ×2 : Ingestion, IoT.
- Redis : cache.

Chaque service a sa propre base, et aucun service ne lit la base d'un autre.

## Détail des services

### 1. Auth & Utilisateurs

Inscription, connexion JWT, rôles (citoyen, météorologue, admin) et préférences. Tout le monde contribue à ce service.

**Tables (5) :** `users`, `roles`, `user_preferences`, `api_keys`, `refresh_tokens`

### 2. Ingestion

Appelle chaque minute OpenWeatherMap et Open-Meteo, normalise les données et les publie sur RabbitMQ.

**Collections (4) :** `raw_observations`, `providers`, `ingestion_jobs`, `ingestion_errors`

**Trafic :** des milliers d'écritures par minute.

### 3. Prévisions

Prévisions horaires et sur 7 jours par ville ou par coordonnées, avec calcul de la précision des modèles.

**Tables (5) :** `cities`, `forecasts`, `forecast_models`, `forecast_accuracy`, `weather_conditions`

**Trafic :** service le plus consulté, avec un cache Redis.

### 4. Alertes & Notifications

Règles personnalisées de l'utilisateur (par exemple « alerte si > 40 °C ») et envoi par email, SMS ou push.

**Tables (5) :** `alert_rules`, `alerts`, `subscriptions`, `notification_channels`, `notification_logs`

**Trafic :** envois massifs lors des tempêtes et des canicules.

### 5. Stations & Capteurs IoT

Gère les stations et reçoit les mesures (température, humidité, vent, pression) en HTTP ou MQTT.

**Collections (4) :** `stations`, `sensors`, `measurements`, `maintenance_logs`

**Trafic :** très élevé en écriture, d'où le choix de WebFlux non bloquant.

### 6. Risques & Catastrophes naturelles

Ce service transforme la météo en niveau de risque par zone géographique. Il calcule pour chaque zone un indice de 0 à 100 pour chaque type de risque.

| Risque | Données utilisées |
|---|---|
| Canicule | Températures mesurées et prévues sur plusieurs jours |
| Inondation | Pluie cumulée sur 24 h et prévisions de pluie |
| Tempête | Vitesse du vent et pression |
| Feux de forêt | Chaleur, sécheresse et vent |
| Grand froid | Températures minimales |

| Niveau | Indice |
|---|---|
| 🟢 Vert | 0-39 |
| 🟡 Jaune | 40-59 |
| 🟠 Orange | 60-79 |
| 🔴 Rouge | 80-100 |

**Différence avec le service Alertes :** Alertes envoie des notifications selon les règles personnelles de chaque utilisateur. Risques analyse les données de chaque zone pour calculer un indice objectif, puis prévient Alertes quand le niveau change.

**Tables (5) :**

- `risk_types` : types de risque et leurs paramètres.
- `risk_zones` : zones géographiques (polygones).
- `risk_thresholds` : seuils par risque et par niveau.
- `risk_assessments` : indice calculé par zone, risque et date.
- `risk_events` : historique des catastrophes et épisodes à risque.

**Trafic :** chaque nouvelle mesure ou prévision déclenche un recalcul, et la carte des risques est très consultée. Redis sert de cache pour la carte.

**API (MVP) :**

| Méthode | Route | Description |
|---|---|---|
| GET | `/risks/zones/{id}` | Risques actuels d'une zone |
| GET | `/risks/map?type=heatwave` | Carte des risques (GeoJSON) |
| GET | `/risks/events` | Historique des épisodes |
| POST | `/risks/thresholds` | Modification des seuils (admin) |

### 7. Signalements

Signalement de la météo réelle (pluie, grêle, brouillard) avec photo, commentaires et votes de fiabilité, plus un fil en direct par WebSocket.

**Tables (5) :** `reports`, `report_photos`, `comments`, `votes`, `badges`

**Trafic :** de type réseau social, avec beaucoup de lectures et d'écritures.

## Dépendances entre membres

| Membre | Service | Consomme | Fournit |
|---|---|---|---|
| 1 | Ingestion | APIs externes | `measurement.received` |
| 2 | Prévisions | `measurement.received` | `forecast.updated` |
| 3 | Alertes | `risk.level.changed`, Auth | Emails, SMS, push |
| 4 | IoT | Capteurs simulés | `measurement.received` |
| 5 | Risques | `measurement.received`, `forecast.updated` | `risk.level.changed`, carte des risques |
| 6 | Signalements | Auth | `report.created` |

## Événements RabbitMQ

| Événement | Émis par | Lu par |
|---|---|---|
| `measurement.received` | Ingestion, IoT | Prévisions, Risques |
| `forecast.updated` | Prévisions | Risques |
| `report.created` | Signalements | Risques (optionnel) |
| `risk.level.changed` | Risques | Alertes |

Exemple `measurement.received` :

```json
{
  "source": "iot",
  "sourceId": "station-12",
  "zoneId": "TUN-01",
  "type": "temperature",
  "value": 41.2,
  "unit": "C",
  "timestamp": "2026-11-08T10:00:00Z"
}
```

Exemple `risk.level.changed` :

```json
{
  "zoneId": "TUN-01",
  "riskType": "heatwave",
  "previousLevel": "yellow",
  "newLevel": "orange",
  "index": 68,
  "timestamp": "2026-11-08T10:05:00Z"
}
```

## Travailler séparément sans se bloquer

Les contrats sont figés avant de coder (S4-S5), dans `Documentation/Contracts/` :

- Un fichier OpenAPI par service, écrit avant le code.
- Un catalogue des événements RabbitMQ.

Règles d'indépendance :

- Chaque service a son dossier, son Dockerfile et sa base dans le `docker-compose`.
- Chacun simule (mock) les services dont il dépend. Par exemple, le Membre 5 envoie lui-même de faux `measurement.received` dans RabbitMQ.
- Chacun écrit un script seed pour remplir sa base de test.
- Le JWT est décidé dès S4 (algorithme, clé partagée, contenu du token : `userId`, `role`) pour que les services Spring valident les tokens de Django.

## Structure du dépôt

```text
MeteoSphere/
├── Backend/            Microservices et infrastructure
├── Frontend/           Application Angular et carte Leaflet
├── Documentation/
│   ├── Contracts/      Contrats OpenAPI et catalogue des événements (à créer en S4-S5)
│   ├── Diagrams/       Diagrammes du projet
│   └── Presentation/   Supports de présentation
└── Readme.md
```
