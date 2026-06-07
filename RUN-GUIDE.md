# RUN-GUIDE.md — Guide d'exécution TAMS

## Manuel technique pour lancer et utiliser l'application

---

## 1. Prérequis

| Outil | Version | Vérification |
|-------|---------|-------------|
| Java JDK | 17+ | `java -version` |
| Maven | 3.9+ | `mvn -version` |
| Node.js | 18+ | `node -v` |
| Docker Desktop | latest | `docker compose version` |
| IntelliJ IDEA | 2023+ | — |

---

## 2. Infrastructure : Axon Server (Docker)

**Déjà fait** — Axon Server tourne sur http://localhost:8024 (Dashboard) et gRPC :8124.

```bash
# Vérifier le statut
docker compose ps

# Logs en direct
docker compose logs -f

# Arrêt
docker compose down
```

---

## 3. Backend : Build Maven

### 3.1 Premier build (depuis la racine du projet)

```bash
# Depuis C:\workspace\tams-axon-sb3-reactjs
mvn -f tams-backe-end-axon-sb3/pom.xml clean install -DskipTests
```

> **Important** : Ce build compile d'abord `tams-core-api` (JAR partagé), puis les autres modules. Obligatoire avant le premier lancement.

### 3.2 Build d'un seul module

```bash
mvn -f tams-backe-end-axon-sb3/talent-request-service/pom.xml clean install -DskipTests
```

---

## 4. IntelliJ : Configuration et lancement des microservices

### 4.1 Ouvrir le projet dans IntelliJ

- **File → Open** → sélectionner `C:\workspace\tams-axon-sb3-reactjs\tams-backe-end-axon-sb3`
- IntelliJ détecte automatiquement le POM parent et les 6 modules
- Laissez IntelliJ indexer et télécharger les dépendances Maven (barre en bas à droite)

### 4.2 Configuration des Run Configurations

Pour chaque module, créez une configuration **Application** :

| Module | Classe main | Port |
|--------|-------------|------|
| `tams-discovery-service` | `TamsDiscoveryServiceApplication` | 8761 |
| `tams-api-gateway` | `TamsApiGatewayApplication` | 8080 |
| `talent-request-service` | `TalentRequestServiceApplication` | 0 (aléatoire) |
| `talent-fulfillment-service` | `TalentFulfillmentServiceApplication` | 0 (aléatoire) |
| `career-portal-service` | `CareerPortalServiceApplication` | 0 (aléatoire) |

**Procédure pour chaque module :**

1. **Run → Edit Configurations**
2. Cliquez **+** → **Application**
3. Remplissez :
   - **Name** : `tams-discovery-service` (ou le nom du module)
   - **Main class** : cliquez sur l'icône dossier → naviguez jusqu'à la classe
   - **Working directory** : `C:\workspace\tams-axon-sb3-reactjs\tams-backe-end-axon-sb3`
   - **Use classpath of module** : sélectionnez le module correspondant
   - **JRE** : Java 17+
4. **OK**

**Alternative rapide** : ouvrez chaque classe `*Application.java` → cliquez sur la flèche verte à côté de `public static void main` → **Run**

### 4.3 Ordre de lancement (IMPORTANT)

```
1. tams-discovery-service   → http://localhost:8761
2. tams-api-gateway         → http://localhost:8080
3. talent-request-service
4. talent-fulfillment-service
5. career-portal-service
```

Attendez que chaque service soit complètement démarré avant de lancer le suivant.

### 4.4 Vérification des logs

À la fin des logs de chaque service, cherchez :

```
Started TamsDiscoveryServiceApplication in X seconds
```

Pour les services avec `server.port=0`, cherchez dans les logs :

```
Tomcat started on port(s): 54321 (http)
```

Notez le port aléatoire de chaque service pour les consoles H2.

### 4.5 Vérification Eureka

Ouvrez http://localhost:8761 — vous devez voir 3 services enregistrés :

| Service | Statut |
|---------|--------|
| `TALENT-REQUEST-SERVICE` | UP (1 instance) |
| `TALENT-FULFILLMENT-SERVICE` | UP (1 instance) |
| `CAREER-PORTAL-SERVICE` | UP (1 instance) |

---

## 5. Frontend : Application React

### 5.1 Installation des dépendances

```bash
cd tams-uat-front-end-react
npm install
```

### 5.2 Lancement

```bash
npm start
# → http://localhost:3000
```

### 5.3 Structure des appels API

Le frontend appelle l'API Gateway sur `http://localhost:8080` :

| Appel React | URL Gateway | Service cible |
|------------|-------------|---------------|
| `GET /talent-request` | `http://localhost:8080/talent-request-service/talent-request` | talent-request-service |
| `POST /talent-request` | `http://localhost:8080/talent-request-service/talent-request` | talent-request-service |
| `GET /talent-request/{id}` | `http://localhost:8080/talent-request-service/talent-request/{id}` | talent-request-service |
| `GET /talent-fulfillment` | `http://localhost:8080/talent-fulfillment-service/talent-fulfillment` | talent-fulfillment-service |
| `GET /talent-fulfillment/{id}` | `http://localhost:8080/talent-fulfillment-service/talent-fulfillment/{id}` | talent-fulfillment-service |
| `POST /talent-fulfillment/job-post` | `http://localhost:8080/talent-fulfillment-service/talent-fulfillment/job-post` | talent-fulfillment-service |
| `GET /job-post` | `http://localhost:8080/career-portal-service/job-post` | career-portal-service |
| `GET /job-post/{id}` | `http://localhost:8080/career-portal-service/job-post/{id}` | career-portal-service |

> Le Gateway utilise `discovery.locator.enabled=true` pour router automatiquement vers les services via leur nom Eureka (en lowercase).

---

## 6. Parcours utilisateur pas à pas (Test manuel)

### 6.1 Créer une demande de recrutement

| # | Action | Résultat attendu |
|---|--------|-----------------|
| 1 | Ouvrir http://localhost:3000 | Page d'accueil TAMS |
| 2 | Cliquer **"Hiring Managers click here"** | Portail Hiring Manager |
| 3 | Cliquer **"Create New Talent Request"** | Formulaire de création |
| 4 | Remplir : Titre = "Développeur React Senior", Responsabilités = "Développer des composants React", Qualifications = "5 ans d'expérience", Core Skill = "REACT", Skill Level = "ADVANCED", Start Date = une date future | — |
| 5 | Cliquer **"Submit Talent Request"** | Redirection vers la liste des demandes |
| 6 | Voir la demande avec le statut **OPEN** | Une ligne apparaît dans le tableau |

### 6.2 Consulter la demande (Hiring Manager)

| # | Action | Résultat attendu |
|---|--------|-----------------|
| 7 | Cliquer **"View Talent Request"** sur la ligne | Détails complets de la demande |
| 8 | Vérifier : Titre, Description, Compétences, Statut **OPEN** | Toutes les infos sont correctes |

### 6.3 Approuver la demande (Talent Acquisition)

| # | Action | Résultat attendu |
|---|--------|-----------------|
| 9 | Menu → **"Talent Fulfillment Portal"** | Portail Talent Acquisition |
| 10 | Cliquer **"View All Talent Requests"** | Liste des demandes à traiter |
| 11 | Vérifier le statut **ASSIGNED_TO_TA** | La demande a été automatiquement assignée |
| 12 | Cliquer **"View & Approve Talent Request"** | Formulaire d'approbation |
| 13 | Sélectionner : Role Level = "Individual Contributor", Employment Type = "Full Time", Approved = "APPROVED" | — |
| 14 | Cliquer **"Approve Job Post"** | Redirection vers la liste |
| 15 | Vérifier le statut **APPROVED** | La demande est approuvée |

### 6.4 Voir l'offre publiée (Career Portal)

| # | Action | Résultat attendu |
|---|--------|-----------------|
| 16 | Menu → **"Careers Portal"** | Portail carrières |
| 17 | Cliquer **"View All Job Openings"** | Liste des offres publiées |
| 18 | Voir la nouvelle offre avec le titre "Développeur React Senior" | L'offre est automatiquement créée |
| 19 | Cliquer **"View Job Post"** | Détails complets de l'offre |

---

## 7. Consoles H2 (Bases de données de projection)

Chaque service a sa propre base H2 en mémoire. Elles sont accessibles **via l'API Gateway** (le `discovery.locator.enabled=true` route automatiquement `/h2` vers chaque service) :

| Service | URL Console (via Gateway) | JDBC URL |
|---------|---------------------------|----------|
| talent-request-service | http://localhost:8080/talent-request-service/h2 | `jdbc:h2:mem:talent_requests` |
| talent-fulfillment-service | http://localhost:8080/talent-fulfillment-service/h2 | `jdbc:h2:mem:talent_fulfillment` |
| career-portal-service | http://localhost:8080/career-portal-service/h2 | `jdbc:h2:mem:job_posts` |

> **Connexion** : Laissez `User Name: sa`, `Password: vide`, entrez la **JDBC URL** du tableau ci-dessus, puis **Connect**.

Exemple de requêtes SQL pour vérifier les données :

```sql
SELECT * FROM TALENT_REQUEST;
SELECT * FROM TALENT_FULFILLMENT;
SELECT * FROM JOB_POST;
```

---

## 8. Dashboard Axon Server

Axon Server tourne sur Docker avec le **dev-mode activé** (`AXONIQ_AXONSERVER_DEVMODE_ENABLED=true`). L'Event Store est automatiquement vidé à chaque démarrage du conteneur. Pour forcer un nettoyage manuel :

```bash
docker compose down -v
docker compose up -d
```

Ouvrez http://localhost:8024 :

| Onglet | Utilisation |
|--------|-------------|
| **Dashboard** | Vue d'ensemble, statut du serveur |
| **Events** | Liste des événements stockés (TalentRequestCreatedEvent, etc.) |
| **Commands** | Commandes exécutées et leur statut |
| **Queries** | Requêtes de lecture et réponses |
| **Aggregates** | Instances des aggregates avec leurs événements |

---

## 9. Dépannage

### 9.1 Le service ne démarre pas

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| `Port 8761 already in use` | Eureka déjà lancé | Changez le port ou arrêtez l'autre instance |
| `Connection refused: localhost/127.0.0.1:8124` | Axon Server pas démarré | `docker compose up -d` |
| `No qualifying bean` | Build Maven pas fait | `mvn clean install -DskipTests` |
| `ClassNotFoundException` | tams-core-api pas compilé | Build le parent POM d'abord |

### 9.2 Erreurs Eureka

- Vérifiez que `tams-discovery-service` est lancé en premier
- Vérifiez http://localhost:8761 — les services doivent être `UP`
- Les autres services doivent pointer vers `http://localhost:8761/eureka`

### 9.3 Erreurs Axon

- Vérifiez le Dashboard Axon : http://localhost:8024
- Vérifiez que les services ont bien les dépendances Axon (pom.xml)
- Le port gRPC (8124) doit être accessible depuis les conteneurs/services

### 9.4 `NoClassDefFoundError: OutOfDirectMemoryError`

| Cause | Solution |
|-------|----------|
| Incompatibilité entre Netty (Spring Boot 3.2) et `axonserver-connector-java 2023.2.0` | Ajouter `io.netty:netty-common:4.1.100.Final` dans les POMs des 3 services business |

### 9.5 `AggregateNotFoundException: The aggregate was not found in the event store`

| Cause | Solution |
|-------|----------|
| Redémarrage d'Axon Server sans redémarrer les services → Event Store vide mais projections H2 encore présentes | Redémarrer les 3 services (talent-request, talent-fulfillment, career-portal) dans IntelliJ, puis refaire le parcours depuis le début |

### 9.6 Erreur 400 Bad Request lors de l'approbation (Job Post)

| Cause | Solution |
|-------|----------|
| Les champs `employmentType`, `roleLevel` ou `requestStatus` sont vides ou `ASSIGNED_TO_TA` au lieu de `APPROVED` | Sélectionner obligatoirement `APPROVED`, un `Employment Type` et un `Role Level` dans les dropdowns avant de soumettre |

### 9.7 Le frontend n'affiche pas les données

- Vérifiez que l'API Gateway est sur http://localhost:8080
- Vérifiez les appels dans les DevTools (F12 → Network)
- Vérifiez les CORS (si erreur, configurer dans le Gateway)

---

## 10. Résumé des URLs

| Service | URL |
|---------|-----|
| Application React | http://localhost:3000 |
| API Gateway | http://localhost:8080 |
| Eureka Dashboard | http://localhost:8761 |
| Axon Server Dashboard | http://localhost:8024 |
| H2 talent-request-service | http://localhost:8080/talent-request-service/h2 |
| H2 talent-fulfillment-service | http://localhost:8080/talent-fulfillment-service/h2 |
| H2 career-portal-service | http://localhost:8080/career-portal-service/h2 |

---

## 11. Commandes utiles

```bash
# Build complet
mvn -f tams-backe-end-axon-sb3/pom.xml clean install -DskipTests

# Lancer un service (sans IntelliJ)
mvn -f tams-backe-end-axon-sb3/talent-request-service/pom.xml spring-boot:run

# Logs Axon Server
docker compose logs -f

# Redémarrer Axon Server
docker compose restart

# Arrêter Axon Server
docker compose down
```
