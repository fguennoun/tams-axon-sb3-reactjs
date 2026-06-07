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
      |   (status=OPEN)         |                          |
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

| Mode | Protocole | Cas d'usage | Avantage |
|------|-----------|-------------|----------|
| **Commandes/Événements** | Axon Server gRPC (port 8124) | Mutation d'aggregat, propagation d'événements | Couplage faible, asynchrone, journalisé |
| **Requêtes HTTP (lecture)** | API Gateway → Eureka → REST | GET queries, affichage portail | Réponse synchrone, familiarité REST |
| **Sagas** | Axon Saga Manager (gRPC) | Orchestration longue durée multi-services | Garantie de cohérence finale, compensation |

### Patterns de résilience

- **Bulkhead** : Axon segmente chaque Event Processor Group avec son propre pool de threads (`threadCount`), isolant les pannes entre projections.
- **Retry with exponential backoff** : Axon retente automatiquement les commandes échouées (configurable via `axon.command.retry.max-count` et `axon.command.retry.backoff-delay`).
- **Idempotency** : Les événements sont naturellement idempotents (traitement `@EventHandler` peut être rejoué sans effet de bord). Les commandes incluent un identifiant unique (`commandId`) pour la déduplication côté Aggregate.
- **Dead Letter Queue** : Axon Enterprise propose une DLQ pour les événements qui échouent après tous les tentatives ; en dev-mode, ils sont simplement loggés.

### API Versioning

Les événements dans l'Event Store sont immortels. La stratégie de versioning est implicite :

```java
// Version 1 : champ unique CoreSkill
public class CandidateSkills {
    private CoreSkill coreSkill;
    private SkillLevel skillLevel;
}

// Version 2 (future) : liste de compétences
// Upcaster requis pour migrer v1 → v2 dans l'Event Store
```

- **Backward-compatible** : ajout de champs optionnels uniquement (valeur par défaut dans `@EventSourcingHandler`)
- **Breaking change** : nécessite un `Upcaster` (cf. section 4)
- **Dép récation** : les vieux champs sont annotés `@Deprecated` mais jamais supprimés des classes d'événements

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

### Agrégation & Limites (Bounded Contexts)

Chaque aggregate est conçu dans son propre **Bounded Context** :

| Bounded Context | Aggregate | Responsabilité | Événements publiés |
|-----------------|-----------|---------------|-------------------|
| **Recrutement** (talent-request-service) | `TalentRequestAggregate` | Gérer le cycle de vie de la demande : création, mise à jour de statut | `TalentRequestCreatedEvent`, `TalentRequestStatusUpdatedEvent` |
| **Fulfillment** (talent-fulfillment-service) | `TalentFulfillmentAggregate` | Gérer l'approbation : attribution rôle, validation décision | `TalentFulfillmentCreatedEvent`, `TalentFulfillmentDecisionSubmittedEvent` |
| **Carrière** (career-portal-service) | `JobPostAggregate` | Publier l'offre sur le portail | `JobPostCreatedEvent` |

La separation en 3 aggregates distincts (plutôt qu'un seul monolithe) est justifiée par :
- **Autonomie d'évolution** : chaque équipe peut modifier son aggregate sans affecter les autres
- **Granularité transactionnelle** : les invariants d'un aggregate ne verrouillent pas les autres
- **Séparation des préoccupations** : l'aggregate `TalentRequest` ne connaît pas `RoleLevel` ou `EmploymentType`

### Invariants & Validation

Les invariants sont enforceés dans les `@CommandHandler` avant `AggregateLifecycle.apply()` :

```java
@CommandHandler
public TalentRequestAggregate(CreateTalentRequestCommand command) {
    // Invariant : le titre est obligatoire
    if (command.getTalentRequestTitle() == null || command.getTalentRequestTitle().isBlank()) {
        throw new IllegalArgumentException("Talent request title is required");
    }
    // Invariant : la date de début doit être dans le futur
    if (command.getStartDate() != null && command.getStartDate().isBefore(LocalDate.now())) {
        throw new IllegalArgumentException("Start date must be in the future");
    }
    AggregateLifecycle.apply(new TalentRequestCreatedEvent(/* ... */));
}
```

**Règle fondamentale** : ne jamais émettre un événement représentant un état invalide. L'aggregat est le gardien de sa propre cohérence.

### Repository Patterns

Dans une architecture Event Sourcing, on distingue :

| Pattern | Usage dans TAMS | Implémentation |
|---------|----------------|----------------|
| **Repository (JPA)** | Côté Query (projection) | `JpaRepository<TalentRequest, String>` — opérations CRUD standard sur la BDD de lecture |
| **Event Sourcing Repository** | Côté Command (aggregate) | Fourni par Axon : `AggregateRepository<TalentRequestAggregate>`. Charge l'aggregat en rejouant les événements depuis l'Event Store |
| **Unit of Work** | Transactionnel | Axon garantit que `AggregateLifecycle.apply(event)` est atomique : soit tous les événements sont stockés, soit aucun |

```java
// Query Repository (JPA standard)
@Repository
public interface TalentRequestRepository extends JpaRepository<TalentRequest, String> {
    List<TalentRequest> findByRequestStatus(RequestStatus status);
}

// Command Repository (géré par Axon)
// Pas de code explicite — Axon utilise le AggregateRepository automatiquement
// Injection dans le service :
// @Autowired
// private Repository<TalentRequestAggregate> aggregateRepository;
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

### Event Store vs. Base de Projection

| Aspect | Event Store (Axon Server) | Base de Projection (H2) |
|--------|--------------------------|------------------------|
| **Nature** | Journal d'événements immuables (append-only) | Tables relationnelles (mises à jour, suppressions possibles) |
| **Source de vérité** | Oui — **source of truth** | Non — dérivée, peut être reconstruite |
| **Schéma** | Flexible — les événements évoluent avec le temps | Fixe — défini par l'entité JPA courante |
| **Requêtes** | Impossible (pas de query directe sur l'Event Store) | SQL standard via JPA |
| **Performance** | Écriture séquentielle rapide | Indexés pour les lectures |
| **Stockage** | Axon Server (fichier ou JDBC selon config) | H2 en mémoire (dev) ou PostgreSQL (prod) |

### Snapshots (Instantanés)

Pour éviter de rejouer des milliers d'événements à chaque chargement d'aggregat :

```yaml
# application.yml — chaque service business
axon:
  eventhandling:
    snapshot-trigger:
      threshold: 50  # Déclenche un snapshot tous les 50 événements
```

```java
@Configuration
public class AxonSnapshotConfig {
    @Bean
    public SnapshotTriggerDefinition snapshotTriggerDefinition(
            Snapshotter snapshotter) {
        return new ThresholdSnapshotTriggerDefinition(snapshotter, 50);
    }
}
```

- Un snapshot capture l'état complet de l'aggregat à un instant T
- Au chargement, Axon lit le snapshot le plus récent + les événements depuis ce snapshot
- Réduit drastiquement le temps de chargement pour les aggregates avec un historique long

### Upcasting (Migration d'Événements)

Quand la structure d'un événement change (ajout/suppression/renommage de champ), un **Upcaster** transforme les anciens événements vers le nouveau format au moment de la lecture :

```java
// Exemple : migration CandidateSkills v1 (champs séparés) → v2 (objet imbriqué)
public class CandidateSkillsUpcaster extends SingleEventUpcaster {

    @Override
    protected boolean canUpcast(IntermediateEventRepresentation event) {
        return event.getType().equals("TalentRequestCreatedEvent");
    }

    @Override
    protected IntermediateEventRepresentation doUpcast(
            IntermediateEventRepresentation event) {
        return event.upcast(
            Map.of("com.example.event", this::transformPayload),
            String.class
        );
    }

    private Document transformPayload(Document document) {
        // Extraire les champs v1, les imbriquer dans un objet
        return new Document(document)
            .remove("coreSkill")
            .remove("skillLevel");
    }
}
```

- Enregistré dans le package `upcaster/` de `tams-core-api`
- Se déclenche automatiquement à la lecture des événements depuis l'Event Store
- **Ne modifie jamais** les événements stockés (immutabilité)
- Versionné : chaque upcaster a son propre numéro de séquence

### Event Schema Evolution — Règles

| Type de changement | Compatible ? | Action requise |
|-------------------|-------------|----------------|
| Ajout d'un champ optionnel | Oui | Valeur par défaut dans `@EventSourcingHandler` |
| Ajout d'un champ obligatoire | Non | Upcaster requis, valeur par défaut |
| Renommage d'un champ | Non | Upcaster + `@Deprecated` sur l'ancien champ |
| Suppression d'un champ | Oui | Upcaster pour ignorer l'ancien champ, puis suppression dans la classe |
| Changement de type d'un champ | Non | Upcaster pour transformer le type |

---

## 5. CQRS (Command Query Responsibility Segregation)

### Architecture

```
                    +---------------------------+
                    |       API Gateway         |
                    |     (localhost:8080)      |
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
                    | Aggregate|     |  Query      |
                    | (Event   |     |  Handlers   |
                    | Sourced) |     |  (JPA Read) |
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

### Flux complet des données (End-to-End)

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
[CommandGateway]
       |
       v
[TalentRequestAggregate]
       | @CommandHandler
       | AggregateLifecycle.apply(event)
       v
[TalentRequestCreatedEvent]
       |
       +----------+-----------+----------+
       |          |           |          |
       v          v           v          v
   [Axon ES]  [EventSourcing]  [EventHandler]  [TalentRequestSaga]
   (stocke)   (reconstruit     (projette dans    (déclenche la
               état courant)    BDD H2)          prochaine étape)
                                                   |
                                                   v
                                      [CreateTalentFulfillmentCommand]
                                                   |
                                                   v
                              [TalentFulfillmentAggregate]
                                   | @CommandHandler
                                   | AggregateLifecycle.apply(event)
                                   v
                          [TalentFulfillmentCreatedEvent]
                                   |
                                   +----------+----------+
                                   |          |          |
                                   v          v          v
                               [Axon ES]  [EventSourcing]  [EventHandler]
                               (stocke)   (reconstruit     (projette dans
                                            état courant)    BDD H2)
                                                          |
                                   [TalentRequestSaga] ←--+ (fin)
                                   (reçoit événement,
                                    envoie UpdateStatus
                                    envoie UpdateStatusCmd)
                                          |
                                          v
                              [TalentRequestAggregate]
                              (status → ASSIGNED_TO_TA)
                                          |
                                          v
                                   [Nouvel événement]
                                   (TalentRequestStatusUpdatedEvent)

              ═══════ Étape 2 : Approbation par Talent Acquisition ═══════

[Talent Acquisition Specialist]
       |
       | POST /talent-fulfillment/job-post  (via API Gateway)
       v
[TalentFulfillmentCommandController]
       |
       v
[TalentFulfillmentService]
       |  submitFulfillmentDecision(command)
       |  → commandGateway.send(command)
       v
[CommandGateway]
       |
       v
[TalentFulfillmentAggregate]
       | @CommandHandler (SubmitTalentFulfillmentDecisionCommand)
       | AggregateLifecycle.apply(event)
       v
[TalentFulfillmentDecisionSubmittedEvent]
       |
       +----------+-----------+----------+
       |          |           |          |
       v          v           v          v
   [Axon ES]  [EventSourcing]  [EventHandler]  [JobPostCreationSaga]
   (stocke)   (reconstruit     (projette dans    (déclenche la
               état courant)    BDD H2)          création du job)
                                                   |
                                                   v
                                      [CreateJobPostCommand]
                                                   |
                                                   v
                                       [JobPostAggregate]
                                        (career-portal-service)
                                            | @CommandHandler
                                            | AggregateLifecycle.apply(event)
                                            v
                                       [JobPostCreatedEvent]
                                            |
                                            +----------+----------+
                                            |          |          |
                                            v          v          v
                                        [Axon ES]  [EventSourcing]  [EventHandler]
                                        (stocke)   (reconstruit     (projette dans
                                                     état courant)    BDD job_posts)
                                                                       |
                                                                 [Career Portal]
                                                                 (lecture H2)

                                            [JobPostCreationSaga] ←--+ (fin)
                                            (reçoit JobPostCreatedEvent)
                                                   |
                                                   v
                                      [UpdateTalentRequestStatusCommand]
                                      (status → APPROVED)
                                                   |
                                                   v
                                      [TalentRequestAggregate]
                                      (status mis à jour)
```

### Validation des Commandes

La validation intervient à 3 niveaux :

| Niveau | Lieu | Exemple | Comportement en cas d'échec |
|--------|------|---------|---------------------------|
| **Contrat HTTP** | `CommandController` / DTO | `@NotBlank talentRequestTitle` | 400 Bad Request avant d'atteindre Axon |
| **Métier (invariant)** | `@CommandHandler` de l'aggregate | Vérification `requestStatus` avant transition | Exception levée → commande annulée, aucun événement stocké |
| **Technique (Axon)** | Intercepteur `CommandDispatchInterceptor` | Validation de structure, logging, autorisation | Intercepteur rejette la commande avant routage |

```java
// Intercepteur de commande côté expéditeur
@Component
public class ValidationCommandDispatchInterceptor
        implements CommandDispatchInterceptor {

    @Override
    public CommandMessage<?> handle(CommandMessage<?> commandMessage) {
        // Logique de pré-validation transverse
        if (commandMessage.getPayload() instanceof CreateTalentRequestCommand cmd) {
            if (cmd.getTalentRequestTitle() == null) {
                throw new IllegalArgumentException("Title is required");
            }
        }
        return commandMessage;
    }
}
```

### Idempotence des Commandes

Dans un système distribué, une commande peut être émise plusieurs fois (retry, timeout, double-click). Stratégies :

```java
// 1. Identifiant unique de commande dans l'Event Store
// Axon déduplique automatiquement si le même événement est déjà stocké

// 2. Token d'idempotence côté client (frontend)
// Généré par React et envoyé dans le header X-Idempotency-Key
// Vérifié par un CommandDispatchInterceptor avant routage
```

- **Idempotence naturelle** : les commandes `Create*` échouent si l'aggregat existe déjà (contrainte d'unicité du identifiant)
- **Idempotence des événements** : `@EventSourcingHandler` est conçu pour être appelé plusieurs fois sans effet de bord
- **Token bucket** : en production, stocker les tokens d'idempotence dans Redis avec TTL

### Optimisation du Modèle de Requêtes

```java
// Query personnalisée avec filtres
public class FindTalentRequestsByStatusQuery {
    private RequestStatus status;
    // constructeur, getters
}

// Query Handler
@QueryHandler
public List<TalentRequest> handle(FindTalentRequestsByStatusQuery query) {
    return talentRequestRepository.findByRequestStatus(query.getStatus());
}
```

- **Requêtes dédiées** : chaque query sert un cas d'usage précis (pas de GenericQuery)
- **Indexation JPA** : `@Table(indexes = @Index(columnList = "requestStatus"))` sur l'entité de projection
- **DTO de réponse** : les `QueryResponseDTO` ne contiennent que les champs nécessaires à l'UI (pas de sur-fetching)
- **Pagination** : intégrer `Pageable` pour les listes volumineuses
- **Caching** : `@Cacheable` sur les queries en lecture fréquente (ex: liste des job posts publiés)

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

### Configuration avancée (application.yml)

```yaml
axon:
  # Bus & Event Store
  axonserver:
    servers: localhost:8124
    event-store:
      push-events: true          # Mode push pour les événements (vs polling)

  # Event Handling
  eventhandling:
    processors:
      talent-request-group:      # Nom du groupe de processeurs
        mode: TRACKING           # TRACKING = événements depuis le début (replayable)
        thread-count: 2          # Traitement parallèle des événements
        batch-size: 100          # Taille du lot par commit
        sequencer:               # Ordre de traitement (PER_AGGREGATE = séquentiel par aggregate)
          type: PER_AGGREGATE
    snapshot-trigger:
      threshold: 50

  # Command retry
  command:
    retry:
      max-count: 3               # Tentatives max pour une commande
      backoff-delay: 1000        # Délai initial (ms) entre les tentatives

  # Query
  query:
    default:
      timeout: 10000             # Timeout pour les queries (ms)
```

### Event Processor Groups — Modes

| Mode | Description | Cas d'usage dans TAMS |
|------|-------------|----------------------|
| **SUBSCRIBING** | Receive des événements en temps réel (live) | Processeurs en mode par défaut |
| **TRACKING** | Rejoue depuis le début ou depuis un jeton (token) | Projections configurées explicitement avec `mode: TRACKING` |
| **POOLED** | Pool de threads avec token auto-géré (via JPA/JDBC ou Embedded) | Production — haute disponibilité |

**Avantage TRACKING** : permet de **rejouer les projections** en cas de bug ou de nouvelle version du handler. Il suffit de réinitialiser le token :

```bash
# Via Axon Dashboard : Operations → Reset Token
# Ou via API REST d'Axon Server
```

### Gestion des Erreurs (Error Handling)

```java
// Intercepteur de commande côté réception
@Component
public class ExceptionWrappingHandlerInterceptor
        implements HandlerInterceptor {

    @Override
    public boolean handle(UnitOfWork<?> unitOfWork,
                          InterceptorChain interceptorChain) {
        try {
            return interceptorChain.proceed();
        } catch (IllegalArgumentException e) {
            // Transformer en CommandExecutionException (serialisable)
            throw new CommandExecutionException(e.getMessage(), e);
        }
    }
}
```

| Scénario d'erreur | Comportement Axon | Stratégie |
|-------------------|-------------------|-----------|
| Commande invalide | `CommandExecutionException` → retour au `CommandGateway.send()` | Controller advise → 400 Bad Request |
| Événement de projection échoué | Log + propagation (ou DLQ en Enterprise) | `@EventHandler` try/catch + compensation |
| Timeout gRPC | Axon retente selon `retry.max-count` | Configurer un backoff adapté |
| Saga bloquée | Saga reste en vie dans Axon Server | Dead-letter + notification ops |

### Dev-Mode Axon Server

La propriété `AXONIQ_AXONSERVER_DEVMODE_ENABLED=true` :

- Réinitialise l'Event Store au redémarrage du conteneur Docker
- Crée un seul contexte par défaut (pas de multi-context)
- Active un license de développement gratuite (limitée à 3 nœuds)
- **Ne pas utiliser en production** (perte de tous les événements au restart)

---

## 7. Schéma d'Architecture Complet

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Navigateur)                             │
│                         http://localhost:3000                                │
│                      React + Redux Toolkit + Axios                           │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │ HTTP
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    tams-api-gateway (Spring Cloud Gateway)                   │
│                          http://localhost:8080                               │
│                  discovery.locator.enabled=true                              │
│                  Routes automatiques via Eureka                              │
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
       │                  │                       │
       │    ┌─────────────┴──────────────┐        │
       │    │  Axon Server (gRPC :8124)  │        │
       │    │  - Event Store             │        │
       │    │  - Command Bus             │        │
       │    │  - Query Bus               │        │
       │    │  - Saga Manager            │        │
       │    └─────────────┬──────────────┘        │
       │                  │                       │
       │    ┌─────────────┴──────────────┐        │
       └────┤  tams-discovery-service    ├────────┘
            │  (Eureka - port 8761)      │
            └────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                        tams-core-api (Librairie partagée)                    │
│                                                                              │
│  Domain:        CoreSkill, SkillLevel, EmploymentType, RoleLevel,            │
│                 RequestStatus, JobDescription, CandidateSkills               │
│                                                                              │
│  Commands:      CreateJobPostCommand, CreateTalentFulfillmentCommand,        │
│                 UpdateTalentRequestStatusCommand                             │
│                                                                              │
│  Events:        JobPostCreatedEvent, TalentFulfillmentCreatedEvent           │
│                                                                              │
│  Config:        AxonXStreamConfig                                            │
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
     |                    |-- /talent- -->   |               |                  |                  |                  |
     |                    |   request        |               |                  |                  |                  |
     |                    |                  |-- POST ------>|                  |                  |                  |
     |                    |                  |   /talent-    |                  |                  |                  |
     |                    |                  |   request     |                  |                  |                  |
     |                    |                  |               |-- CreateTalent-->|                  |                  |
     |                    |                  |               |   RequestCommand |                  |                  |
     |                    |                  |               |                  |-- store event -->|                  |
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
  Talent Acquisition      |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |-- GET /talent ---->|                  |               |                  |                  |                  |
     |   fulfillment      |--- /talent- ---->|               |                  |                  |                  |
     |                    |   fulfillment    |-- GET ------->|                  |                  |                  |
     |                    |                  |               |-- Query -------->|                  |                  |
     |                    |                  |               |   all fulfill    |                  |                  |
     |                    |<--- list --------|               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |-- POST /approve -->|                  |               |                  |                  |                  |
     |                    |--- /talent- ---->|               |                  |                  |                  |
     |                    |   fulfillment    |-- POST ------>|                  |                  |                  |
     |                    |   /job-post      |               |   SubmitTalent-->|                  |                  |
     |                    |                  |               |   Fulfillment    |                  |                  |
     |                    |                  |               |   DecisionCmd    |                  |                  |
     |                    |                  |               |                  |-- TalentFulfill--|                  |
     |                    |                  |               |                  |   mentDecision   |                  |
     |                    |                  |               |                  |   SubmittedEvent |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |====== SAGA ======|                  |
     |                    |                  |               |                  | JobPostCreationSaga                 |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |-- CreateJobPost->|                  |
     |                    |                  |               |                  |   Command        |-- store event -> |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |<-- UpdateStatus -|                  |                  |
     |                    |                  |               |   Cmd (APPROVED) |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
  Candidat                |                  |               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
     |-- GET /career ---->|                  |               |                  |                  |                  |
     |   portal/job-posts |--- /career- ---->|               |                  |                  |                  |
     |                    |   portal/job-    |-- GET ------->|                  |                  |                  |
     |                    |   posts          |               |                  |                  |--- Query ------->|
     |                    |                  |               |                  |                  |    all job posts |
     |                    |<--- job posts ---|               |                  |                  |                  |
     |                    |                  |               |                  |                  |                  |
```

---

## 12. Stratégie de Test (Testing Strategy)

### Pyramide de test pour TAMS

```
              ╱╲
             ╱  ╲
            ╱E2E ╲          ← Tests end-to-end (Cypress, Playwright)
           ╱______╲
          ╱        ╲
         ╱  Saga    ╲       ← Tests de sagas (Axon Test Fixtures)
        ╱   Tests    ╲
       ╱______________╲
      ╱                ╲
     ╱  Integration     ╲   ← Tests Spring Boot slices (@WebMvcTest, @DataJpaTest)
    ╱      Tests         ╲
   ╱______________________╲
  ╱                        ╲
 ╱  Unit Tests (JUnit 5)    ╲  ← Tests d'aggregates, handlers, services
╱____________________________╲
```

### 1. Tests Unitaires — Aggregates (Axon Fixtures)

```java
class TalentRequestAggregateTest {

    private FixtureConfiguration<TalentRequestAggregate> fixture;

    @BeforeEach
    void setUp() {
        fixture = new AggregateTestFixture<>(TalentRequestAggregate.class);
    }

    @Test
    void shouldCreateTalentRequest() {
        String id = UUID.randomUUID().toString();
        fixture.givenNoPriorActivity()
               .when(new CreateTalentRequestCommand(id, "Dev Java", /* ... */))
               .expectSuccessfulHandlerExecution()
               .expectEvents(new TalentRequestCreatedEvent(/* ... */));
    }

    @Test
    void shouldRejectBlankTitle() {
        String id = UUID.randomUUID().toString();
        fixture.givenNoPriorActivity()
               .when(new CreateTalentRequestCommand(id, "", /* ... */))
               .expectException(IllegalArgumentException.class);
    }

    @Test
    void shouldTransitionFromOpenToApproved() {
        String id = UUID.randomUUID().toString();
        fixture.given(new TalentRequestCreatedEvent(id, /* ... */))
               .when(new UpdateTalentRequestStatusCommand(id, RequestStatus.APPROVED))
               .expectEvents(new TalentRequestStatusUpdatedEvent(id, RequestStatus.APPROVED));
    }
}
```

### 2. Tests Unitaires — Sagas (Axon Saga Test Fixtures)

```java
class TalentRequestSagaTest {

    private SagaTestFixture<TalentRequestSaga> fixture;

    @BeforeEach
    void setUp() {
        fixture = new SagaTestFixture<>(TalentRequestSaga.class);
        fixture.registerAggregate(TalentFulfillmentAggregate.class);
    }

    @Test
    void shouldStartSagaOnTalentRequestCreated() {
        String id = UUID.randomUUID().toString();
        fixture.givenAggregate(id).published()
               .whenAggregate(id).publishes(
                   new TalentRequestCreatedEvent(id, "Dev Java", /* ... */))
               .expectActiveSagas(1)
               .expectDispatchedCommands(
                   new CreateTalentFulfillmentCommand(/* ... */));
    }

    @Test
    void shouldEndSagaOnFulfillmentCreated() {
        // given: saga started
        // when: TalentFulfillmentCreatedEvent
        // expect: saga ends, status command dispatched
    }
}
```

### 3. Tests d'Intégration — Query Projections

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = Replace.ANY)
class TalentRequestEventHandlerTest {

    @Autowired
    private TalentRequestRepository repository;

    @Autowired
    private TalentRequestEventHandler eventHandler;

    @Test
    void shouldProjectTalentRequestCreatedEvent() {
        String id = UUID.randomUUID().toString();
        eventHandler.on(new TalentRequestCreatedEvent(id, "Dev Java", /* ... */));

        TalentRequest entity = repository.findById(id).orElseThrow();
        assertThat(entity.getTalentRequestTitle()).isEqualTo("Dev Java");
        assertThat(entity.getRequestStatus()).isEqualTo(RequestStatus.OPEN);
    }
}
```

### 4. Tests d'Intégration — Contrôleurs REST

```java
@WebMvcTest(TalentRequestCommandController.class)
class TalentRequestCommandControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private TalentRequestService talentRequestService;

    @Test
    void shouldReturn201OnCreate() throws Exception {
        String body = """
            {
                "talentRequestTitle": "Dev Java",
                "startDate": "2026-07-01"
            }
            """;

        mockMvc.perform(post("/talent-request")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body))
               .andExpect(status().isCreated());
    }

    @Test
    void shouldReturn400OnInvalidInput() throws Exception {
        String body = """
            { "talentRequestTitle": "" }
            """;

        mockMvc.perform(post("/talent-request")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body))
               .andExpect(status().isBadRequest());
    }
}
```

### 5. Tests End-to-End

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class TalentRequestE2ETest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void fullFlowShouldCreateAndReadTalentRequest() {
        // 1. Create
        var createPayload = Map.of(
            "talentRequestTitle", "Dev Java",
            "startDate", "2026-07-01"
        );
        var createResponse = restTemplate.postForEntity(
            "http://localhost:" + port + "/talent-request",
            createPayload,
            Map.class
        );
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // 2. Read
        String id = (String) createResponse.getBody().get("talentRequestId");
        var getResponse = restTemplate.getForEntity(
            "http://localhost:" + port + "/talent-request/" + id,
            Map.class
        );
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().get("talentRequestTitle")).isEqualTo("Dev Java");
    }
}
```

### Résumé des technologies de test

| Niveau | Framework | Artefact testé |
|--------|-----------|---------------|
| Unitaire (Aggregate) | Axon `AggregateTestFixture` | `@CommandHandler`, `@EventSourcingHandler`, invariants |
| Unitaire (Saga) | Axon `SagaTestFixture` | `@SagaEventHandler`, `@StartSaga`, `@EndSaga` |
| Intégration (Projection) | `@SpringBootTest` + JPA | `@EventHandler`, `@QueryHandler` |
| Intégration (API) | `@WebMvcTest` + MockMvc | Contrôleurs REST, validation |
| E2E | `@SpringBootTest(RANDOM_PORT)` | Flow complet commande + query |

---

## 13. Considérations de Production (Production Considerations)

### Scalabilité

| Dimension | Stratégie TAMS | Détails |
|-----------|---------------|---------|
| **Horizontal scaling** | Instances multiples de chaque service | Eureka équilibre les appels HTTP ; Axon Server distribue les événements via gRPC |
| **Event Processor parallelism** | `thread-count: N` par Event Processor Group | Permet de traiter N événements en parallèle (attention à l'ordre par aggregate) |
| **Axon Server cluster** | 3+ nœuds en production | Haute disponibilité de l'Event Store, réplication synchrone des événements |
| **Snapshot threshold** | `threshold: 50` (ajuster selon l'historique) | Réduit le temps de chargement des aggregates longs |
| **CQRS read scaling** | Répliquer les bases de projection | Lecture scalable indépendamment de l'écriture |

### Base de données persistante (Production vs Dev)

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/talent_requests
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate     # Ne pas créer les tables en prod (utiliser Flyway/Liquibase)
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

axon:
  axonserver:
    servers: axonserver-1:8124,axonserver-2:8124,axonserver-3:8124  # Cluster
  eventhandling:
    processors:
      talent-request-group:
        mode: POOLED         # Pooled Streaming pour production
        thread-count: 4
```

### Sécurité

```java
// Intercepteur de commande pour validation d'autorisation
@Component
public class SecurityCommandDispatchInterceptor
        implements CommandDispatchInterceptor {

    @Override
    public CommandMessage<?> handle(CommandMessage<?> commandMessage) {
        // Récupérer le token JWT depuis les métadonnées de la commande
        String token = commandMessage.getMetaData().get("jwt_token");
        if (!isAuthorized(token, commandMessage.getPayloadType())) {
            throw new CommandExecutionException("Unauthorized", null);
        }
        return commandMessage;
    }
}
```

| Aspect | Implémentation |
|--------|---------------|
| **Authentification** | Spring Security + JWT (Bearer token dans headers HTTP) |
| **Autorisation des commandes** | `CommandDispatchInterceptor` vérifie les droits avant routage |
| **Sécurité de l'Event Store** | Axon Server Enterprise supporte TLS + ACL par contexte |
| **Protection des endpoints** | API Gateway valide le token avant routage vers les services |
| **Secrets management** | Variables d'environnement (pas de secrets dans `application.yml`) |

### Monitoring & Observabilité

```yaml
# Actuator + Micrometer
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

**Métriques clés à surveiller :**

| Métrique | Source | Seuil d'alerte |
|----------|--------|---------------|
| `axon.command.duration` | Micrometer | > 1s (latence anormale) |
| `axon.event.processor.progress` | Axon Dashboard | Retard > 1000 événements |
| `axon.snapshot.load.duration` | Micrometer | > 500ms |
| `jvm.memory.used` | Actuator | > 80% heap |
| `discovery.health` | Eureka | Instance down |

### Déploiement

#### Docker Compose Production

```yaml
services:
  axonserver:
    image: axoniq/axonserver:enterprise   # Version Enterprise pour cluster
    environment:
      - AXONIQ_AXONSERVER_NAME=axonserver-1
      - AXONIQ_AXONSERVER_HOSTNAME=axonserver-1
      - AXONIQ_AXONSERVER_DOMAIN=axonserver-cluster
      - spring.datasource.url=jdbc:postgresql://postgres:5432/axon_event_store
    volumes:
      - axon-events:/data/events
    ports:
      - "8124:8124"
      - "8024:8024"

  talent-request-service:
    build: ./talent-request-service
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - AXONIQ_AXONSERVER_SERVERS=axonserver-1:8124
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://discovery-service:8761/eureka/
    deploy:
      replicas: 3    # 3 instances pour la résilience

  # ... autres services
```

#### Pipeline CI/CD (GitHub Actions)

```yaml
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with: { java-version: '21' }
      - run: mvn clean test

  build:
    needs: test
    steps:
      - run: mvn package -DskipTests
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/tams/${{ matrix.service }}:${{ github.sha }}
```

### Gestion des erreurs en production

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CommandExecutionException.class)
    public ResponseEntity<ErrorResponse> handleCommandException(
            CommandExecutionException ex) {
        return ResponseEntity
            .badRequest()
            .body(new ErrorResponse("COMMAND_ERROR", ex.getMessage()));
    }

    @ExceptionHandler(AxonNonTransientException.class)
    public ResponseEntity<ErrorResponse> handleNonTransient(
            AxonNonTransientException ex) {
        // Erreur fatale — alerter les ops
        log.error("Non-transient Axon error", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("AXON_FATAL", "Event Store unreachable"));
    }
}
```

### Checklist de mise en production

- [ ] Remplacer H2 par PostgreSQL (ou autre base relationnelle)
- [ ] Configurer Axon Server en mode cluster (3 nœuds minimum)
- [ ] Activer TLS pour les connexions gRPC
- [ ] Configurer les snapshots (threshold adapté au volume d'événements)
- [ ] Mettre en place les upcasters pour la version courante des événements
- [ ] Ajouter Spring Security + JWT
- [ ] Configurer les health checks Eureka (période, seuil)
- [ ] Activer les métriques Prometheus + Grafana
- [ ] Définir les alertes Ops (latence, erreurs, stockage)
- [ ] Tests de résilience (chaos engineering : tuer des instances, simuler des pannes gRPC)

---

## 14. Points d'Extension & Améliorations Futures

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
