# TAMS: Axon + Spring Boot 3 back-end with React front-end

This repository contains a multi-module Spring Boot 3 back-end built with Axon Framework (CQRS/ES), and a React front-end for a Talent Acquisition Management System (TAMS). It is organized as:
- Back-end aggregator project: tams-backe-end-axon-sb3 (Maven multi-module/parent)
  - career-portal-service
  - talent-fulfillment-service
  - talent-request-service
  - tams-api-gateway
  - tams-core-api
  - tams-discovery-service
- Front-end: tams-uat-front-end-react

The docker-compose.yml at the repo root provides Axon Server for local event store and messaging.


## 1) Architecture overview

- DDD + CQRS/ES with Axon Framework
  - Commands mutate state on Aggregates and emit Events.
  - Events (event-sourced) are stored and distributed via Axon Server.
  - Query models (read side) are maintained via event handlers and exposed via REST.
- Microservices (Spring Boot 3)
  - Discovery via Netflix Eureka (tams-discovery-service)
  - API Gateway via Spring Cloud Gateway (tams-api-gateway)
- React front-end consuming the back-end APIs to support different personas:
  - Career Portal (viewing job posts)
  - Hiring Manager Portal (creating and tracking talent requests)
  - Talent Acquisition Portal (fulfilling talent requests)


## 2) Technology stack

Back-end:
- Java 17, Spring Boot 3.x
- Axon Framework 4.9.x (axon-spring-boot-starter), Axon Server connector
- Spring Data JPA, H2 in-memory databases for local dev
- Spring Cloud Netflix Eureka (discovery) and Spring Cloud Gateway
- Maven (multi-module aggregator)

Front-end:
- React (Create React App structure)
- Redux Toolkit for state management

DevOps / Local:
- Docker Compose for Axon Server


## 3) Module-by-module

Parent/aggregator (Maven):
- tams-backe-end-axon-sb3/pom.xml (packaging=pom) aggregates and builds all backend modules together.

Domain/Shared:
- tams-core-api: Core commands, events, domain enums and configuration (e.g., Axon XStream config). Consumed by other services.

Business services:
- career-portal-service
  - Aggregate: JobPostAggregate
  - Query side: JobPost entity/repository, query services, REST controllers
  - Exposes endpoints to list and view job posts (query side)
- talent-request-service
  - Aggregate: TalentRequestAggregate
  - Commands: CreateTalentRequestCommand; emits TalentRequestCreatedEvent, status update events
  - Query side for listing and retrieving talent requests
  - Saga: TalentRequestSaga orchestrates cross-service flows
- talent-fulfillment-service
  - Aggregate: TalentFulfillmentAggregate
  - Commands/events around fulfillment decisions
  - Query side to list/retrieve fulfillments
  - Saga: JobPostCreationSaga bridges to job post creation upon certain decisions

Edge services:
- tams-discovery-service (Eureka Server)
- tams-api-gateway (Spring Cloud Gateway): routes requests to services discovered via Eureka

Front-end:
- tams-uat-front-end-react: React app with feature folders (CareerPortal, TalentRequest, TalentFulfillment) and components/pages for each persona.


## 4) CQRS/ES with Axon (theory and how it’s applied here)

- Commands: Intent to change state, routed to Aggregates. Example: CreateTalentRequestCommand.
- Aggregates: Enforce invariants and apply events in command handlers. Example: TalentRequestAggregate.
- Events: Facts emitted by aggregates (e.g., TalentRequestCreatedEvent). Persisted to Axon Server event store; used for projection updates and inter-service communication.
- Event Handlers/Projections: Update JPA entities (e.g., JobPost, TalentRequest, TalentFulfillment) on the query side for fast reads. The project uses H2 for local development.
- Sagas/Process Managers: Coordinate long-running, cross-service processes. Example: TalentRequestSaga and JobPostCreationSaga.
- Axon Server: Provides event storage, subscription/query/command routing in dev. We provide docker-compose support for local use.


## 5) Project structure (high level)

C:\workspace\tams-axon-sb3-reactjs
- tams-backe-end-axon-sb3 (Maven parent, packaging=pom)
  - career-portal-service
  - talent-fulfillment-service
  - talent-request-service
  - tams-api-gateway
  - tams-core-api
  - tams-discovery-service
- tams-uat-front-end-react (React app)
- docker-compose.yml (Axon Server for local dev)
- README.md (this file)


## 6) Build and run

Prerequisites:
- Java 17+, Maven 3.9+
- Node.js 18+ (for the front-end)
- Docker Desktop (to run Axon Server via docker-compose)

Build all backend modules:
- mvn -f tams-backe-end-axon-sb3/pom.xml clean install

Run Axon Server locally (recommended before starting services):
- docker compose up -d
- Axon Server UI: http://localhost:8024
- Axon gRPC port: 8124

Run Eureka (discovery):
- From IDE or command line: mvn -f tams-backe-end-axon-sb3/tams-discovery-service/pom.xml spring-boot:run
- Accessible at: http://localhost:8761

Run API Gateway:
- mvn -f tams-backe-end-axon-sb3/tams-api-gateway/pom.xml spring-boot:run

Run business services (in any order after Eureka is up):
- career-portal-service: mvn -f tams-backe-end-axon-sb3/career-portal-service/pom.xml spring-boot:run
- talent-request-service: mvn -f tams-backe-end-axon-sb3/talent-request-service/pom.xml spring-boot:run
- talent-fulfillment-service: mvn -f tams-backe-end-axon-sb3/talent-fulfillment-service/pom.xml spring-boot:run

Run the front-end:
- cd tams-uat-front-end-react
- npm install
- npm start


## 7) Configure services to use Axon Server (local dev)

The code already includes Axon dependencies (see tams-core-api/pom.xml). Ensure each service has the following properties to connect to Axon Server (add if not present):

spring.application.name=<service-name>
axon.axonserver.servers=localhost:8124
axon.serializer.general=xstream

Notes:
- The axon.axonserver.servers property points to the gRPC port exposed by Axon Server in docker-compose.
- XStream is configured via tams-core-api configuration (AxonXStreamConfig). Adjust serializers as needed.


## 8) Practical end-to-end flow (example)

1. Hiring Manager creates a Talent Request (command) via talent-request-service REST endpoint.
2. TalentRequestAggregate handles the command and applies TalentRequestCreatedEvent.
3. Event is stored in Axon Server and published. Query handlers update TalentRequest projection for reads.
4. Saga reacts to events and may emit commands to other services (e.g., to create related JobPosts).
5. Career Portal reads JobPost projections through REST, consumed by React front-end.


## 9) Useful endpoints (local dev)

- Eureka Dashboard: http://localhost:8761
- Axon Server Dashboard: http://localhost:8024
- API Gateway (base): http://localhost:8080 (actual routes depend on gateway config)
- H2 Consoles (per service): /h2 (e.g., http://localhost:<service_port>/h2)


## 10) Troubleshooting

- If commands/queries aren’t routing, check Axon Server UI and ensure services can reach localhost:8124.
- Ensure docker compose is up: docker compose ps
- For dependency or build issues, rebuild parent: mvn -f tams-backe-end-axon-sb3/pom.xml clean install -U
- Port conflicts: services use server.port=0 for random ports (check logs) while Eureka and Gateway have fixed ports.


## 11) Future improvements

- Centralize dependencyManagement in the parent POM (versions for Spring Boot, Axon, Spring Cloud) to ensure alignment across modules.
- Add docker compose services for Eureka and the API Gateway to enable one-command startup.
- Introduce profiles for dev/test/prod, and persistent databases for projections.
