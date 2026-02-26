# Microservices TP1 - Architecture E-Commerce

## Architecture Globale

```
                          ┌──────────────────┐
                          │   Config Service  │
                          │    (port 8888)    │
                          └────────┬─────────┘
                                   │ fournit la config
                                   ▼
┌──────────┐          ┌──────────────────────┐
│  Client  │──────►   │   Gateway Service    │
│ (Browser │          │     (port 8080)      │
│ / curl)  │          └───┬──────┬───────┬───┘
└──────────┘              │      │       │
                          │      │       │        ┌────────────────────┐
                          │      │       │        │ Discovery Service  │
                          │      │       │        │   (Eureka 8761)    │
                          │      │       │        └────────────────────┘
                          │      │       │          ▲  tous les services
                          │      │       │          │  s'enregistrent ici
                          ▼      ▼       ▼
               ┌─────────┐ ┌──────────┐ ┌──────────┐
               │Catalogue │ │Commande  │ │Paiement  │
               │ Service  │ │ Service  │ │ Service  │
               │ (8081)   │ │ (8082)   │ │ (8083)   │
               └─────────┘ └──────────┘ └──────────┘
                     ▲           │             │
                     │  OpenFeign│   OpenFeign │
                     └───────────┘             │
                          ▲                    │
                          │    OpenFeign       │
                          └────────────────────┘
```

## Comment les services sont-ils liés ?

Les services communiquent entre eux via **OpenFeign** (appels HTTP synchrones) :

| Source | Destination | Pourquoi ? |
|--------|------------|------------|
| **commande-service** | **catalogue-service** | Pour récupérer les infos produit (nom, prix) lors de la création d'une commande |
| **paiement-service** | **commande-service** | Pour récupérer les infos commande (montant total) lors de la création d'un paiement |

La **gateway-service** route toutes les requêtes entrantes vers le bon microservice via **Eureka** (service discovery).

---

## Description de chaque service

### 1. Discovery Service (Eureka Server) - Port 8761

**Rôle** : Registre de services. Tous les microservices s'enregistrent auprès d'Eureka pour être découverts dynamiquement.

**Tester** : Ouvrir http://localhost:8761 dans un navigateur pour voir le dashboard Eureka et les services enregistrés.

---

### 2. Config Service (Spring Cloud Config) - Port 8888

**Rôle** : Fournit la configuration centralisée pour tous les microservices. Les fichiers de configuration sont stockés dans `config-service/src/main/resources/configurations/`.

**Tester** :
```bash
# Voir la config du catalogue-service
curl http://localhost:8888/catalogue-service/default
```

---

### 3. Gateway Service (Spring Cloud Gateway) - Port 8080

**Rôle** : Point d'entrée unique (API Gateway). Route les requêtes vers les bons microservices :
- `/api/v1/produits/**` → catalogue-service
- `/api/v1/commandes/**` → commande-service
- `/api/v1/paiements/**` → paiement-service

---

### 4. Catalogue Service - Port 8081

**Rôle** : Gestion du catalogue de produits (CRUD).

**Architecture hexagonale** :
- `domain/Produit.java` - Entité JPA
- `dto/ProduitRequestDTO.java` - DTO d'entrée avec validation
- `dto/ProduitResponseDTO.java` - DTO de sortie
- `dto/ProduitMapper.java` - Mapping entité ↔ DTO
- `repository/ProduitRepository.java` - Accès base de données (H2)
- `service/ProduitService.java` - Logique métier
- `controller/ProduitController.java` - Endpoints REST
- `exception/GlobalExceptionHandler.java` - Gestion globale des erreurs

**Endpoints** :

| Méthode | URL | Description |
|---------|-----|-------------|
| GET | `/api/v1/produits` | Lister tous les produits |
| GET | `/api/v1/produits/{id}` | Obtenir un produit par ID |
| POST | `/api/v1/produits` | Créer un produit |
| PUT | `/api/v1/produits/{id}` | Modifier un produit |
| DELETE | `/api/v1/produits/{id}` | Supprimer un produit |

**Tester** :
```bash
# Créer un produit
curl -X POST http://localhost:8080/api/v1/produits \
  -H "Content-Type: application/json" \
  -d "{\"nom\":\"Laptop Dell\",\"description\":\"Ordinateur portable\",\"prix\":1299.99,\"quantiteStock\":15}"

# Lister les produits
curl http://localhost:8080/api/v1/produits

# Obtenir un produit
curl http://localhost:8080/api/v1/produits/1

# Modifier un produit
curl -X PUT http://localhost:8080/api/v1/produits/1 \
  -H "Content-Type: application/json" \
  -d "{\"nom\":\"Laptop Dell XPS\",\"description\":\"Ordinateur portable haut de gamme\",\"prix\":1499.99,\"quantiteStock\":10}"

# Supprimer un produit
curl -X DELETE http://localhost:8080/api/v1/produits/1
```

---

### 5. Commande Service - Port 8082

**Rôle** : Gestion des commandes. Appelle **catalogue-service** via OpenFeign pour récupérer les informations des produits commandés.

**Architecture hexagonale** :
- `domain/Commande.java` - Entité commande avec statut (EN_ATTENTE, CONFIRMEE, EXPEDIEE, LIVREE, ANNULEE)
- `domain/LigneCommande.java` - Ligne de commande (produit + quantité)
- `dto/CommandeRequestDTO.java`, `CommandeResponseDTO.java` - DTOs
- `dto/LigneCommandeRequestDTO.java`, `LigneCommandeResponseDTO.java` - DTOs lignes
- `dto/ProduitDTO.java` - DTO pour les données reçues du catalogue
- `client/CatalogueClient.java` - **Client OpenFeign** vers catalogue-service
- `service/CommandeService.java` - Logique métier
- `controller/CommandeController.java` - Endpoints REST

**Lien avec catalogue-service** : Quand on crée une commande, le service appelle `catalogue-service` pour chaque produit afin de récupérer son nom et son prix.

**Endpoints** :

| Méthode | URL | Description |
|---------|-----|-------------|
| GET | `/api/v1/commandes` | Lister toutes les commandes |
| GET | `/api/v1/commandes/{id}` | Obtenir une commande par ID |
| POST | `/api/v1/commandes` | Créer une commande |
| PATCH | `/api/v1/commandes/{id}/statut?statut=CONFIRMEE` | Modifier le statut |
| DELETE | `/api/v1/commandes/{id}` | Supprimer une commande |

**Tester** :
```bash
# Créer une commande (le produit avec id=1 doit exister dans catalogue-service)
curl -X POST http://localhost:8080/api/v1/commandes \
  -H "Content-Type: application/json" \
  -d "{\"lignes\":[{\"produitId\":1,\"quantite\":2}]}"

# Lister les commandes
curl http://localhost:8080/api/v1/commandes

# Modifier le statut
curl -X PATCH "http://localhost:8080/api/v1/commandes/1/statut?statut=CONFIRMEE"
```

---

### 6. Paiement Service - Port 8083

**Rôle** : Gestion des paiements. Appelle **commande-service** via OpenFeign pour récupérer le montant total de la commande.

**Architecture hexagonale** :
- `domain/Paiement.java` - Entité avec mode (CARTE_BANCAIRE, VIREMENT, PAYPAL, ESPECES) et statut (EN_ATTENTE, ACCEPTE, REFUSE, REMBOURSE)
- `dto/PaiementRequestDTO.java`, `PaiementResponseDTO.java` - DTOs
- `dto/CommandeDTO.java` - DTO pour les données reçues de commande-service
- `client/CommandeClient.java` - **Client OpenFeign** vers commande-service
- `service/PaiementService.java` - Logique métier
- `controller/PaiementController.java` - Endpoints REST

**Lien avec commande-service** : Quand on crée un paiement, le service appelle `commande-service` pour récupérer le montant total de la commande.

**Endpoints** :

| Méthode | URL | Description |
|---------|-----|-------------|
| GET | `/api/v1/paiements` | Lister tous les paiements |
| GET | `/api/v1/paiements/{id}` | Obtenir un paiement par ID |
| GET | `/api/v1/paiements/commande/{commandeId}` | Paiements d'une commande |
| POST | `/api/v1/paiements` | Créer un paiement |
| DELETE | `/api/v1/paiements/{id}` | Supprimer un paiement |

**Tester** :
```bash
# Créer un paiement (la commande avec id=1 doit exister dans commande-service)
curl -X POST http://localhost:8080/api/v1/paiements \
  -H "Content-Type: application/json" \
  -d "{\"commandeId\":1,\"modePaiement\":\"CARTE_BANCAIRE\"}"

# Lister les paiements
curl http://localhost:8080/api/v1/paiements

# Paiements d'une commande
curl http://localhost:8080/api/v1/paiements/commande/1
```

---

## Partie C — Configuration des services

### Variables d'environnement (`.env`)

Toutes les variables sont centralisées dans le fichier `.env` à la racine :

| Variable | Valeur par défaut | Description |
|----------|-------------------|-------------|
| `SPRING_PROFILES_ACTIVE` | `docker` | Profil Spring actif |
| `EUREKA_URL` | `http://discovery-service:8761/eureka` | URL Eureka |
| `CONFIG_SERVER_URL` | `http://config-service:8888` | URL Config Server |
| `POSTGRES_USER` | `postgres` | Utilisateur PostgreSQL |
| `POSTGRES_PASSWORD` | `postgres` | Mot de passe PostgreSQL |
| `CATALOGUE_DB_NAME` | `cataloguedb` | Base catalogue |
| `COMMANDE_DB_NAME` | `commandedb` | Base commande |
| `PAIEMENT_DB_NAME` | `paiementdb` | Base paiement |

### Réseau interne

Docker Compose utilise le **réseau par défaut**. Les services communiquent entre eux via leur nom de service :
- `http://discovery-service:8761`
- `http://config-service:8888`
- `http://catalogue-service:8081`
- `http://commande-service:8082`
- `http://paiement-service:8083`
- `http://gateway-service:8080`

### Images Docker

Chaque Dockerfile utilise un build multi-stage avec le Maven Wrapper :
1. **Build stage** (`eclipse-temurin:21-jdk`) : copie `.mvn` + `mvnw`, résout les dépendances offline, compile le JAR
2. **Runtime stage** (`eclipse-temurin:21-jre`) : image légère pour exécuter le JAR

### Bases de données (PostgreSQL)

Chaque service métier dispose de sa propre base PostgreSQL :

| Service | Container DB | Base | Port host |
|---------|-------------|------|-----------|
| catalogue-service | `catalogue-db` | `cataloguedb` | 5432 |
| commande-service | `commande-db` | `commandedb` | 5433 |
| paiement-service | `paiement-db` | `paiementdb` | 5434 |

Les données sont persistées via des **volumes Docker** (`catalogue-data`, `commande-data`, `paiement-data`).

### Healthchecks

Chaque service a un healthcheck configuré :
- **PostgreSQL** : `pg_isready`
- **Spring Boot services** : `wget` sur `/actuator/health`
- `depends_on` avec `condition: service_healthy` garantit l'ordre de démarrage

---

## Partie D — Lancement de l'architecture

### Tâche 7 : Démarrer tous les services

```bash
# Mode build automatique (Tâche 14 — dev mode)
docker compose up --build

# En arrière-plan
docker compose up --build -d
```

### Tâche 8 : Vérifier le bon fonctionnement

```bash
# Lister les conteneurs actifs
docker compose ps
```

Vérifier l'accès via navigateur :
- **Eureka Dashboard** : http://localhost:8761
- **Config Server** : http://localhost:8888
- **Gateway** : http://localhost:8080

---

## Partie E — Tests et validation

### Tâche 9 : Vérifier la communication inter-services

```bash
# Vérifier que les services sont enregistrés dans Eureka
curl http://localhost:8761/eureka/apps

# Tester les appels via la Gateway
curl http://localhost:8080/api/v1/produits
curl http://localhost:8080/api/v1/commandes
curl http://localhost:8080/api/v1/paiements
```

### Tâche 10 : Consulter les logs

```bash
# Tous les logs
docker compose logs -f

# Logs d'un service spécifique
docker compose logs -f catalogue-service
docker compose logs -f commande-service
```

---

## Partie F — Arrêt de l'environnement

### Tâche 11 : Arrêter les services

```bash
# Arrêter les services (conserve les volumes)
docker compose down

# Arrêter et supprimer les volumes (reset complet)
docker compose down -v
```

### Construire une image individuellement

```bash
docker build -t discovery-service ./discovery-service
docker build -t config-service ./config-service
docker build -t gateway-service ./gateway-service
docker build -t catalogue-service ./catalogue-service
docker build -t commande-service ./commande-service
docker build -t paiement-service ./paiement-service
```

---

## Démarrage manuel (sans Docker)

Les services doivent être démarrés dans cet ordre :

1. **discovery-service** (Eureka doit être prêt en premier)
2. **config-service** (optionnel, les configs sont aussi embarquées)
3. **catalogue-service** (doit être prêt avant commande-service)
4. **commande-service** (doit être prêt avant paiement-service)
5. **paiement-service**
6. **gateway-service**

### Depuis IntelliJ IDEA
1. Ouvrir le projet comme projet Maven multi-modules
2. Attendre que Maven télécharge toutes les dépendances
3. Lancer chaque service dans l'ordre ci-dessus (Run → Run 'Application')

### Depuis le terminal
```bash
# Pour chaque service, dans son dossier
mvn spring-boot:run
```

## Scénario de test complet (End-to-End)

```bash
# Étape 1 : Créer un produit
curl -X POST http://localhost:8080/api/v1/produits \
  -H "Content-Type: application/json" \
  -d "{\"nom\":\"Laptop Gaming\",\"description\":\"PC portable gamer\",\"prix\":1500.00,\"quantiteStock\":10}"

# Étape 2 : Créer une commande avec ce produit (utilise OpenFeign → catalogue-service)
curl -X POST http://localhost:8080/api/v1/commandes \
  -H "Content-Type: application/json" \
  -d "{\"lignes\":[{\"produitId\":1,\"quantite\":2}]}"

# Étape 3 : Payer la commande (utilise OpenFeign → commande-service)
curl -X POST http://localhost:8080/api/v1/paiements \
  -H "Content-Type: application/json" \
  -d "{\"commandeId\":1,\"modePaiement\":\"CARTE_BANCAIRE\"}"

# Étape 4 : Vérifier le résultat
curl http://localhost:8080/api/v1/produits
curl http://localhost:8080/api/v1/commandes
curl http://localhost:8080/api/v1/paiements
```

## Stack technique

| Technologie | Usage |
|-------------|-------|
| **Spring Boot 3.2.5** | Framework principal |
| **Spring Cloud 2023.0.2** | Microservices (Gateway, Config, Eureka, Feign) |
| **Java 21** | Langage |
| **H2 Database** | Base de données en mémoire (dev local) |
| **PostgreSQL 16** | Base de données (Docker) |
| **Docker / Docker Compose** | Conteneurisation et orchestration |
| **OpenFeign** | Communication inter-services |
| **JUnit 5 + Mockito** | Tests unitaires |
| **Jakarta Validation** | Validation des DTOs |
| **SLF4J** | Logging structuré |

## Tests unitaires

Chaque service métier a des tests unitaires pour la couche service :
- `catalogue-service/src/test/.../service/ProduitServiceTest.java`
- `commande-service/src/test/.../service/CommandeServiceTest.java`
- `paiement-service/src/test/.../service/PaiementServiceTest.java`

Lancer les tests :
```bash
# Depuis le dossier de chaque service
mvn test
```
