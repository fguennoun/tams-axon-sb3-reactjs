# EXPLAIN.md — Talent Acquisition Management System (TAMS)

## Architecture Microservices, DDD, EDA/CQRS & Axon Framework

---

## 1. Aspect Fonctionnel (Functional Overview)

### MVP — Produit Minimum Viable

Le système répond à 3 besoins métier fondamentaux :

| Acteur | Fonctionnalité | Description |
|--------|---------------|-------------|
| **Responsable du recrutement** (Hiring Manager) | Créer une demande de recrutement | Soumet une `TalentRequest` avec titre, description du poste, compétences requises, date de début |
| **Spécialiste du recrutement** (Talent Acquisition Specialist) | Examiner et approuver la demande | Consulte les demandes, ajoute le niveau de rôle et le type d'emploi, approuve (`APPROVED`) |
| **Candidat / Public** | Consulter les offres publiées | Visite le portail carrières pour voir les `JobPost` automatiquement créés après approbation |

### Workflow métier (End-to-End)

```
[Hiring Manager]         [Talent Acquisition]        [Career Portal]
      |                         |                          |
      |-- Crée TalentRequest -->|                          |
      |   (status=OPEN)        |                          |
      |                         |-- Examine la demande --> |
      |                         |-- Approuve (APPROVED) -> |
      |                         |                          |
      |                         |-- (automatique)          |
      |                         |   Crée JobPost           |
      |                         |   Met à jour status      |
      |                         |                          |
      |                         |                   [Candidat consulte]
```

### Flux de statuts

```
OPEN ──> ASSIGNED_TO_TA ──> APPROVED
  ^                            ^
  |                            |
Création par             Approbation par
Hiring Manager           Talent Acquisition
```

---

## 2. Architecture Microservices (MSA)

### Découpage en 6 modules

```
tams-backe-end-axon-sb3/
  |
  |-- tams-discovery-service    (Eureka Server - port 8761)
  |-- tams-api-gateway          (Spring Cloud Gateway - port 8080)
  |-- tams-core-api             (Librairie partagée : commandes, événements, value objects)
  |-- talent-request-service    (Gestion des demandes de recrutement)
  |-- talent-fulfillment-service (Gestion de l'approbation / fulfillment)
  |-- career-portal-service     (Portail carrières / consultation des offres)
```

### Rôles détaillés

| Microservice | Responsabilité | Port |
|-------------|----------------|------|
| **tams-discovery-service** | Registre de services (Netflix Eureka). Tous les services s'y enregistrent avec un port aléatoire (`server.port=0`) | 8761 |
| **tams-api-gateway** | Point d'entrée unique. Route les requêtes HTTP vers les services via Eureka. Active `discovery.locator.enabled=true` | 8080 |
| **tams-core-api** | Projet JAR partagé (non-exécutable). Contient : commandes inter-services, événements, énumérations, value objects embeddables, configuration XStream |
| **talent-request-service** | Aggregate `TalentRequestAggregate`, Saga `TalentRequestSaga`, query side pour lister/consulter les demandes | random |
| **talent-fulfillment-service** | Aggregate `TalentFulfillmentAggregate`, Saga `JobPostCreationSaga`, endpoints d'approbation | random |
| **career-portal-service** | Aggregate `JobPostAggregate`, lecture seule pour le portail carrières | random |

### Communication inter-services

- **Commandes/Événements** : via Axon Server (gRPC, port 8124)
- **Requêtes HTTP** : via API Gateway → Eureka → service
- **Sagas** : orchestration longue durée entre talent-request-service et talent-fulfillment-service

### Démarrage (ordre requis)

```
1. docker compose up -d        (Axon Server)
2. tams-discovery-service       (Eureka)
3. tams-api-gateway             (Gateway)
4. talent-request-service       (Business service)
5. talent-fulfillment-service   (Business service)
6. career-portal-service        (Business service)
7. npm start                    (React frontend)
```

---

## 3. Domain-Driven Design (DDD)

### Ubiquitous Language (Langage Ubiquitaire)

Les termes métier sont utilisés de manière cohérente dans tout le code :

| Terme | Définition | Classe(s) |
|-------|-----------|-----------|
| **TalentRequest** | Demande de recrutement soumise par un Hiring Manager | `TalentRequestAggregate`, `TalentRequest` (entity) |
| **TalentFulfillment** | Traitement et approbation d'une demande par le Talent Acquisition | `TalentFulfillmentAggregate`, `TalentFulfillment` (entity) |
| **JobPost** | Offre d'emploi publiée sur le portail carrières | `JobPostAggregate`, `JobPost` (entity) |
| **RequestStatus** | Statut du cycle de vie : OPEN → ASSIGNED_TO_TA → APPROVED | `RequestStatus` (enum) |
| **CandidateSkills** | Compétence technique requise (CoreSkill + SkillLevel) | `CandidateSkills` (@Embeddable) |
| **JobDescription** | Description du poste (responsabilités + qualifications) | `JobDescription` (@Embeddable) |

### Aggregates (Racines d'Agrégat)

Chaque aggregate est défini par Axon `@Aggregate` avec `@AggregateIdentifier` :

```
TalentRequestAggregate
  @AggregateIdentifier: talentRequestId
  États: talentRequestTitle, jobDescription, candidateSkills, requestStatus, startDate
  Commandes gérées:
    - CreateTalentRequestCommand → TalentRequestCreatedEvent
    - UpdateTalentRequestStatusCommand → TalentRequestStatusUpdatedEvent

TalentFulfillmentAggregate
  @AggregateIdentifier: talentFulfillmentId
  États: jobPostId, talentRequestId, talentRequestTitle, startDate, jobDescription,
         candidateSkills, requestStatus, roleLevel, employmentType
  Commandes gérées:
    - CreateTalentFulfillmentCommand → TalentFulfillmentCreatedEvent
    - SubmitTalentFulfillmentDecisionCommand → TalentFulfillmentDecisionSubmittedEvent

JobPostAggregate
  @AggregateIdentifier: jobPostId
  États: talentFulfillmentId, talentRequestId, talentRequestTitle, startDate,
         jobDescription, candidateSkills, requestStatus, roleLevel, employmentType
  Commandes gérées:
    - CreateJobPostCommand → JobPostCreatedEvent
```

### Value Objects (Objets de Valeur)

- **`JobDescription`** : `responsibilities` + `qualifications` (tous deux String, @Embeddable)
- **`CandidateSkills`** : `coreSkill` (CoreSkill enum) + `skillLevel` (SkillLevel enum, @Embeddable)
- Enums : `CoreSkill` (JAVA, PYTHON, NODEJS, REACT, PROJECT_MANAGEMENT, AGILE_COACH), `SkillLevel` (STUDENT, JUNIOR, ENTRY, ADVANCED, EXPERT), `EmploymentType` (FULL_TIME, CONTRACT), `RoleLevel` (INDIVIDUAL_CONTRIBUTOR, LEADERSHIP), `RequestStatus` (OPEN, ASSIGNED_TO_TA, APPROVED)

### Domain Events (Événements du Domaine)

Chaque mutation d'aggregate produit un événement :

```
TalentRequestCreatedEvent
TalentRequestStatusUpdatedEvent
TalentFulfillmentCreatedEvent
TalentFulfillmentDecisionSubmittedEvent
JobPostCreatedEvent
```

### Sagas (Orchestrations du Domaine)

Deux sagas modélisent les processus métier transverses :

```
TalentRequestSaga (dans talent-request-service)
  1. @StartSaga: TalentRequestCreatedEvent
     → envoie CreateTalentFulfillmentCommand (statut → ASSIGNED_TO_TA)
  2. @EndSaga: TalentFulfillmentCreatedEvent
     → envoie UpdateTalentRequestStatusCommand (statut → ASSIGNED_TO_TA)

JobPostCreationSaga (dans talent-fulfillment-service)
  1. @StartSaga: TalentFulfillmentDecisionSubmittedEvent
     → envoie CreateJobPostCommand (vers career-portal-service)
  2. @EndSaga: JobPostCreatedEvent
     → envoie UpdateTalentRequestStatusCommand (statut → APPROVED)
```

---

## 4. Event-Driven Architecture (EDA) & Event Sourcing

### Principe

Au lieu de stocker l'état courant d'un aggregate, **chaque changement est stocké comme un événement immuable** dans l'Event Store d'Axon Server. L'état courant est reconstruit en rejouant tous les événements de l'aggregate.

### Implémentation

```
                  Command
                    |
                    v
            +-----------------+
            |  CommandGateway |
            +-----------------+
                    |
                    v
            +-----------------+
            |   Aggregate     |  ← @CommandHandler
            |                 |
            |  AggregateLifecycle.apply(event)
            +-----------------+
                    |
                    v
            +-----------------+
            |  Axon Server    |  ← Stocke l'événement (Event Store)
            +-----------------+
                    |
                    v
            +-----------------+
            | @EventSourcing  |  ← Reconstruit l'état de l'aggregate
            | Handler         |
            +-----------------+
                    |
                    v
            +-----------------+
            | @EventHandler   |  ← Met à jour la BDD de lecture (projection)
            | (Query side)    |
            +-----------------+
```

### Exemple concret : Création d'une TalentRequest

```java
// 1. Commande reçue
@CommandHandler
public TalentRequestAggregate(CreateTalentRequestCommand command) {
    TalentRequestCreatedEvent event = new TalentRequestCreatedEvent();
    BeanUtils.copyProperties(command, event);
    AggregateLifecycle.apply(event);  // Publie et stocke l'événement
}

// 2. Event Sourcing - reconstruction de l'état
@EventSourcingHandler
public void on(TalentRequestCreatedEvent event) {
    this.talentRequestId = event.getTalentRequestId();
    this.talentRequestTitle = event.getTalentRequestTitle();
    // ... copie de tous les champs
}

// 3. Projection - mise à jour de la base de lecture
@EventHandler
public void on(TalentRequestCreatedEvent event) {
    TalentRequest entity = new TalentRequest();
    BeanUtils.copyProperties(event, entity);
    talentRequestRepository.save(entity);
}
```

### Avantages de l'Event Sourcing dans TAMS

- **Audit complet** : chaque modification est tracée dans Axon Server
- **Reconstruction** : possibilité de rejouer les événements pour recréer l'état à un instant T
- **Séparation commande/requête** : la BDD de lecture (H2) est optimisée pour les requêtes
- **Communication inter-services** : les événements servent de contrat entre services

---

## 5. CQRS (Command Query Responsibility Segregation)

### Architecture

```
                    +---------------------------+
                    |       API Gateway         |
                    |     (localhost:8080)       |
                    +----+------------------+---+
                         |                  |
                    POST /               GET /
               (Command Side)      (Query Side)
                         |                  |
                    +----+----+      +------+------+
                    | Command |      | Query       |
                    | Gateway |      | Gateway     |
                    +----+----+      +------+------+
                         |                  |
                    +----+----+      +------+------+
                    | Aggregate|      | Query      |
                    | (Event   |      | Handlers   |
                    | Sourced) |      | (JPA Read) |
                    +---------+      +------------+
                         |
                    +----+----+
                    | Axon    |
                    | Server  |
                    +---------+
```

### Separation stricte dans le code

Chaque service a une séparation physique en packages :

```
talent-request-service/
  command/
    aggregate/  (TalentRequestAggregate)
    command/    (CreateTalentRequestCommand)
    controller/ (TalentRequestCommandController - POST)
    dto/        (DTOs d'entrée/sortie)
    service/    (TalentRequestService - envoie les commandes)
  query/
    controller/ (TalentRequestQueryController - GET)
    dto/        (TalentRequestQueryResponseDTO)
    eventhandler/ (TalentRequestEventHandler - projecte les événements)
    query/      (FindTalentRequestsQuery, FindTalentRequestByTalentRequestIdQuery)
    repository/ (TalentRequest entity JPA + repository)
    service/    (TalentRequestQueryService - interroge le QueryGateway)
  saga/
    TalentRequestSaga
```

### Command Side (Écriture)

| Composant | Rôle |
|-----------|------|
| `CommandController` | Reçoit les requêtes HTTP POST, appelle le service |
| `CommandService` | Construit la commande, l'envoie via `CommandGateway` |
| `Command` (objet) | Implémente `@TargetAggregateIdentifier`, transporté par Axon |
| `Aggregate` | `@CommandHandler` valide et applique `AggregateLifecycle.apply(event)` |
| `Event` | Objet immuable représentant un fait accompli |

### Query Side (Lecture)

| Composant | Rôle |
|-----------|------|
| `QueryController` | Reçoit les requêtes HTTP GET, appelle le service |
| `QueryService` | Envoie la query via `QueryGateway`, retourne la réponse |
| `Query` (objet) | Marqueur pour la query dispatchée (ex: `FindAllTalentRequestsQuery`) |
| `QueryHandler` | `@QueryHandler` exécute la query sur la BDD de lecture |
| `EventHandler` | `@EventHandler` écoute les événements et met à jour la BDD de lecture |
| `Repository` | Interface JPA standard pour les opérations de lecture |
| Entity | `@Entity` JPA mappée sur la table de projection |

### Flux complet des données

```                   
[Hiring Manager]
       |
       | POST /talent-request  (via API Gateway → talent-request-service)
       v
[TalentRequestCommandController]
       |
       v
[TalentRequestService]
       |  createTalentRequest(command)
       |  → commandGateway.send(command)
       v
[CommandGateway] ─────────────────────────────────────────┐
       |                                                   |
       v                                                   |
[TalentRequestAggregate]                                   |
       | @CommandHandler                                    |
       | AggregateLifecycle.apply(event)                    |
       v                                                   |
[TalentRequestCreatedEvent]                                |
       |                                                   |
       +----------+-----------+----------+                 |
       |          |           |          |                 |
       v          v           v          v                 |
   [Axon ES]  [EventSourcing]  [EventHandler]  [TalentRequestSaga]
   (stocke)   (reconstruit     (projette dans    (déclenche la
               état courant)    BDD H2)          prochaine étape)
                                                  |
                                                  v
                                     [CreateTalentFulfillmentCommand]
                                                  |
                                                  v
                                        [TalentFulfillmentAggregate]
                                                  .
                                                  .
                                                  .
```

---

## 6. Axon Framework — Détails Techniques

### Dépendances (pom.xml)

```xml
<!-- Chaque service business -->
<dependency>
    <groupId>org.axonframework</groupId>
    <artifactId>axon-spring-boot-starter</artifactId>
    <version>4.9.0</version>
</dependency>
<dependency>
    <groupId>com.axoniq</groupId>
    <artifactId>axonserver-connector-java</artifactId>
    <version>2023.2.0</version>
</dependency>
```

### Annotations clés utilisées

| Annotation | Utilisation | Exemple |
|-----------|-------------|---------|
| `@Aggregate` | Marque une classe comme racine d'agrégat Axon | `TalentRequestAggregate` |
| `@AggregateIdentifier` | Champ identifiant unique de l'agrégat | `private String talentRequestId` |
| `@CommandHandler` | Méthode qui traite une commande | Constructeur ou méthode handle() |
| `@EventSourcingHandler` | Méthode qui reconstruit l'état depuis un événement | `public void on(TalentRequestCreatedEvent)` |
| `@EventHandler` | Méthode de projection (query side) | `TalentRequestEventHandler.on()` |
| `@QueryHandler` | Méthode qui répond à une query | `TalentRequestsQueryHandler` |
| `@Saga` | Marque une classe comme saga | `TalentRequestSaga` |
| `@StartSaga` | Démarre une saga sur un événement | `handle(TalentRequestCreatedEvent)` |
| `@EndSaga` | Termine une saga | `handle(TalentFulfillmentCreatedEvent)` |
| `@SagaEventHandler` | Gestionnaire d'événement dans une saga | `associationProperty = "talentRequestId"` |

### Configuration XStream (tams-core-api)

```java
@Configuration
public class AxonXStreamConfig {
    @Bean
    public XStream xStream() {
        XStream xStream = new XStream();
        xStream.allowTypesByWildcard(
            new String[]{
                "io.fullstackbasics.**",
                "fullstackbasics.io.**"
            }
        );
        return xStream;
    }
}
```

### Axon Server (docker-compose)

```yaml
services:
  axonserver:
    image: axoniq/axonserver:latest
    ports:
      - "8024:8024"   # Dashboard HTTP
      - "8124:8124"   # gRPC clients
```

- **Port 8024** : Interface de gestion (visualiser événements, commandes, queries)
- **Port 8124** : Communication gRPC pour les clients Axon
- **Volumes** : `axon-data`, `axon-events`, `axon-config` pour la persistance

---

## 7. Schéma d'Architecture Complet

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Navigateur)                             │
│                         http://localhost:3000                                 │
│                      React + Redux Toolkit + Axios                           │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │ HTTP
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    tams-api-gateway (Spring Cloud Gateway)                     │
│                          http://localhost:8080                                 │
│                  discovery.locator.enabled=true                               │
│                  Routes automatiques via Eureka                               │
└─────┬─────────────────────┬─────────────────────┬────────────────────────────┘
      │                     │                     │
      ▼                     ▼                     ▼
┌─────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ talent-     │   │ talent-          │   │ career-portal-   │
│ request     │   │ fulfillment      │   │ service          │
│ -service    │   │ -service         │   │                  │
│             │   │                  │   │                  │
│ POST /      │   │ POST /           │   │ GET /job-post    │
│ talent-     │   │ talent-fulfill-  │   │                  │
│ request     │   │ ment/job-post    │   │                  │
│             │   │                  │   │                  │
│ GET /       │   │ GET /talent-     │   │                  │
│ talent-     │   │ fulfillment      │   │                  │
│ request     │   │                  │   │                  │
└──────┬──────┘   └───────┬──────────┘   └────────┬─────────┘
       │                  │                        │
       │    ┌─────────────┴──────────────┐         │
       │    │  Axon Server (gRPC :8124)  │         │
       │    │  - Event Store             │         │
       │    │  - Command Bus             │         │
       │    │  - Query Bus               │         │
       │    │  - Saga Manager            │         │
       │    └─────────────┬──────────────┘         │
       │                  │                        │
       │    ┌─────────────┴──────────────┐         │
       └────┤  tams-discovery-service    ├─────────┘
            │  (Eureka - port 8761)      │
            └────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                        tams-core-api (Librairie partagée)                     │
│                                                                              │
│  Domain:        CoreSkill, SkillLevel, EmploymentType, RoleLevel,            │
│                 RequestStatus, JobDescription, CandidateSkills               │
│                                                                              │
│  Commands:      CreateJobPostCommand, CreateTalentFulfillmentCommand,        │
│                 UpdateTalentRequestStatusCommand                             │
│                                                                              │
│  Events:        JobPostCreatedEvent, TalentFulfillmentCreatedEvent           │
│                                                                              │
│  Config:        AxonXStreamConfig                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Frontend React — Architecture

### Stack technique

| Technologie | Version | Utilisation |
|-------------|---------|-------------|
| React | 18.2.0 | UI Components |
| Redux Toolkit | 1.9.7 | State management |
| React Router DOM | 6.21.3 | Navigation |
| Axios | 1.6.5 | HTTP client |
| React Toastify | 10.0.4 | Notifications |
| React Icons | 5.0.1 | Icônes |

### Structure Redux (3 slices)

```
store.js
  ├── talentRequests   → talentRequestSlice.js
  │                      talentRequestService.js  (axios → /talent-request-service/talent-request)
  │
  ├── talentFulfillments → talentFulfillmentSlice.js
  │                        talentFulfillmentService.js (axios → /talent-fulfillment-service/talent-fulfillment)
  │
  └── jobPosts          → careerPortalSlice.js
                         careerPortalService.js  (axios → /career-portal-service/job-post)
```

### Routes de l'application

| Route | Component | Portail |
|-------|-----------|---------|
| `/` | `TAMSHome` | Accueil |
| `/hiring-manager` | `HiringManagerPortalHome` | Hiring Manager |
| `/create-talent-request` | `CreateTalentRequestForm` | Hiring Manager |
| `/get-all-talent-requests` | `TalentRequestsView` | Hiring Manager |
| `/talent-request/:id` | `TalentRequestByIdView` | Hiring Manager |
| `/talent-fulfillment` | `TalentAcquisitionPortalHome` | Talent Acquisition |
| `/get-all-talent-fulfillment-requests` | `TalentFulfillmentsView` | Talent Acquisition |
| `/talent-fulfillment/:id` | `TalentFulfillmentByIdView` | Talent Acquisition |
| `/career-portal` | `CareerPortalHome` | Career Portal |
| `/career-portal/job-posts` | `JobPostsView` | Career Portal |
| `/career-portal/job-post/:id` | `JobPostByIdView` | Career Portal |

### Flux React → Backend

```
React Component
  → dispatch(actionCreator)          ← Redux async thunk
    → service.axiosCall()            ← Axios HTTP request
      → http://localhost:8080/       ← API Gateway
        → [service-name]/[route]     ← Eureka routing
          → Controller (REST)        ← Spring Boot
            → Command/Query Gateway  ← Axon
```

---

## 9. Base de Données (Projections)

Chaque service utilise sa propre base H2 en mémoire pour la projection (query side) :

| Service | BDD H2 (JDBC URL) | Console |
|---------|-------------------|---------|
| `talent-request-service` | `jdbc:h2:mem:talent_requests` | `/h2` |
| `talent-fulfillment-service` | `jdbc:h2:mem:talent_fulfillment` | `/h2` |
| `career-portal-service` | `jdbc:h2:mem:job_posts` | `/h2` |

### Tables de projection

```
talent_requests (talent-request-service)
  talentRequestId     VARCHAR PRIMARY KEY
  talentRequestTitle  VARCHAR
  responsibilities    VARCHAR (@Embedded JobDescription)
  qualifications      VARCHAR
  coreSkill           VARCHAR (@Embedded CandidateSkills)
  skillLevel          VARCHAR
  requestStatus       VARCHAR (Enum: OPEN, ASSIGNED_TO_TA, APPROVED)
  startDate           DATE

talent_fulfillment (talent-fulfillment-service)
  talentFulfillmentId VARCHAR PRIMARY KEY
  jobPostId           VARCHAR
  talentRequestId     VARCHAR
  talentRequestTitle  VARCHAR
  startDate           DATE
  responsibilities    VARCHAR
  qualifications      VARCHAR
  coreSkill           VARCHAR
  skillLevel          VARCHAR
  requestStatus       VARCHAR
  roleLevel           VARCHAR
  employmentType      VARCHAR

job_posts (career-portal-service)
  jobPostId           VARCHAR PRIMARY KEY
  talentFulfillmentId VARCHAR
  talentRequestId     VARCHAR
  talentRequestTitle  VARCHAR
  startDate           DATE
  responsibilities    VARCHAR
  qualifications      VARCHAR
  coreSkill           VARCHAR
  skillLevel          VARCHAR
  requestStatus       VARCHAR
  roleLevel           VARCHAR
  employmentType      VARCHAR
```

---

## 10. Résumé des Concepts Appliqués

| Concept | Application dans TAMS |
|---------|----------------------|
| **MSA** (Microservices) | 6 modules indépendants, chacun avec sa propre BDD, son propre cycle de vie, discovery via Eureka, routage via Gateway |
| **DDD** (Domain-Driven Design) | Aggregates (TalentRequest, TalentFulfillment, JobPost), Value Objects (JobDescription, CandidateSkills), Ubiquitous Language, Sagas métier |
| **EDA** (Event-Driven Architecture) | Communication asynchrone via événements (TalentRequestCreatedEvent, TalentFulfillmentDecisionSubmittedEvent, JobPostCreatedEvent) |
| **Event Sourcing** | Stockage des événements dans Axon Server comme source de vérité, reconstruction de l'état via `@EventSourcingHandler` |
| **CQRS** (Command Query Responsibility Segregation) | Séparation physique en packages `command/` et `query/`, bases de lecture dédiées, `CommandGateway` vs `QueryGateway` |
| **Axon Framework** | `@Aggregate`, `@CommandHandler`, `@EventSourcingHandler`, `@EventHandler`, `@QueryHandler`, `@Saga`, `@StartSaga`, `@EndSaga`, Axon Server |
| **Sagas** | `TalentRequestSaga` (orchestre création fulfillment), `JobPostCreationSaga` (orchestre création job post + mise à jour statut) |
| **API Gateway** | Point d'entrée unique, routage dynamique via Eureka, load balancing |
| **Service Discovery** | Enregistrement automatique, health checks, localisation des instances |

---

## 11. Diagramme de Séquence — Flux Complet

```
Hiring Manager          React            API Gateway    talent-request    talent-fulfillment   career-portal     Axon Server
     |                    |                  |               |                  |                  |                  |
     |-- POST /create --->|                  |               |                  |                  |                  |
     |                    |-- /talent- -->    |               |                  |                  |                  |
     |                    |   request         |               |                  |                  |                  |
     |                    |                  |-- POST ------->|                  |                  |                  |
     |                    |                  |   /talent-     |                  |                  |                  |
     |                    |                  |   request      |                  |                  |                  |
     |                    |                  |               |-- CreateTalent--->|                  |                  |
     |                    |                  |               |   RequestCommand  |                  |                  |
     |                    |                  |               |                  |-- store event --->|                  |
     |                    |                  |               |-- TalentRequest--|                  |                  |
     |                    |                  |               |   CreatedEvent   |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |====== SAGA ======|                  |                  |
     |                    |                  |               |   TalentRequestSaga                 |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |-- CreateTalent-->|                  |                  |
     |                    |                  |               |   FulfillmentCmd |                  |                  |
     |                    |                  |               |                  |-- TalentFulfill--|                  |
     |                    |                  |               |                  |   mentCreated    |                  |
     |                    |                  |               |                  |   Event          |                  |
     |                    |                  |               |<-- UpdateStatus -|                  |                  |
     |                    |                  |               |   Cmd            |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
  Talent Acquisition     |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |-- GET /talent ---->|                  |               |                  |                  |                  |
     |   fulfillment      |-- /talent- ---->|               |                  |                  |                  |
     |                    |   fulfillment    |-- GET ------->|                  |                  |                  |
     |                    |                  |               |-- Query -------->|                  |                  |
     |                    |                  |               |   all fulfill    |                  |                  |
     |                    |<-- list --------|               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |-- POST /approve -->|                  |               |                  |                  |                  |
     |                    |-- /talent- ---->|               |                  |                  |                  |
     |                    |   fulfillment    |-- POST ------>|                  |                  |                  |
     |                    |   /job-post      |               |   SubmitTalent--->|                  |                  |
     |                    |                  |               |   Fulfillment    |                  |                  |
     |                    |                  |               |   DecisionCmd    |                  |                  |
     |                    |                  |               |                  |-- TalentFulfill--|                  |
     |                    |                  |               |                  |   mentDecision   |                  |
     |                    |                  |               |                  |   SubmittedEvent |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |====== SAGA ======|                  |
     |                    |                  |               |                  | JobPostCreationSaga                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |-- CreateJobPost->|                  |
     |                    |                  |               |                  |   Command        |-- store event ->|
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |<-- UpdateStatus -|                  |                  |
     |                    |                  |               |   Cmd (APPROVED) |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
  Candidat               |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |-- GET /career ---->|                  |               |                  |                  |                  |
     |   portal/job-posts |-- /career- ---->|               |                  |                  |                  |
     |                    |   portal/job-    |-- GET ------->|                  |                  |                  |
     |                    |   posts          |               |                  |                  |-- Query ------->|
     |                    |                  |               |                  |                  |   all job posts |
     |                    |<-- job posts ---|               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
```

---

## 12. Points d'Extension & Améliorations Futures

### Court terme
- Centraliser la gestion des dépendances dans le POM parent (spring-boot-dependencies, axon-bom)
- Ajouter la validation (Bean Validation / `jakarta.validation`)
- Tests unitaires et d'intégration (JUnit 5, Axon Test Fixtures)
- Gestion des erreurs et exceptions (ControllerAdvice)

### Moyen terme
- Authentification et autorisation (Spring Security + JWT)
- Base de données persistante (PostgreSQL) au lieu de H2
- Monitoring distribués (Spring Boot Actuator + Micrometer)
- Docker Compose complet (incluant Eureka + Gateway + services)

### Long terme
- Event versioning et upcasting
- CI/CD pipeline
- Déploiement Kubernetes
- Event sourcing avec snapshotting
- Tests de résilience (chaos engineering)
