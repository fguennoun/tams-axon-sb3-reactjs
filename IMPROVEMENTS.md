# TAMS Improvements Guide

Purpose: This document lists pragmatic improvements that will make this demo more professional, robust, and explanatory as a Spring Boot DDD + Axon microservices system with a React front end. Each section provides short rationale and actionable steps/checklists.

Note: Treat this as a backlog. Pick the most valuable items first for your context (local demo, team education, or production-readiness simulation).


## 1) Documentation and developer experience

What to add:
- Architecture diagrams
  - Context diagram (services, Axon Server, front-end, API Gateway, Eureka)
  - Component diagrams per service (Aggregates, Sagas, Projections, REST)
  - Sequence diagrams for key flows (Create Talent Request, Publish Job Post)
- ADRs (Architecture Decision Records) to document choices (Axon, Eureka, Gateway, serializer, DB)
- Per-module READMEs: quick start, endpoints, configs, domain glossary
- CONTRIBUTING.md: branch strategy, commit message style, code review checklist
- CODE_OF_CONDUCT.md: community rules if public
- API documentation: Springdoc OpenAPI + Swagger UI
  - Dependency (per service):
    - org.springdoc:springdoc-openapi-starter-webmvc-ui
  - Visit http://localhost:<port>/swagger-ui/index.html
- Makefile or Taskfile for common actions (build, run all, seed data)

Quick actions:
- Add OpenAPI to each service.
- Create /docs folder with PNG/SVG diagrams and ADRs.
- Enhance root README with “Runbook: first 10 minutes” and links to module READMEs.


## 2) Maven parent and dependency management

Goals: consistent versions, reproducible and faster builds.

Actions:
- Centralize versions using dependencyManagement and pluginManagement in tams-backe-end-axon-sb3/pom.xml.
- Import BOMs: Spring Boot, Spring Cloud, Axon.
- Enforce Java and plugin versions via maven-enforcer-plugin.

Example snippet for parent POM:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.5</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.axonframework</groupId>
      <artifactId>axon-bom</artifactId>
      <version>4.9.3</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <release>17</release>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-failsafe-plugin</artifactId>
        <version>3.2.5</version>
      </plugin>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>versions-maven-plugin</artifactId>
        <version>2.16.2</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-enforcer-plugin</artifactId>
        <version>3.4.1</version>
        <executions>
          <execution>
            <id>enforce</id>
            <goals><goal>enforce</goal></goals>
            <configuration>
              <rules>
                <requireJavaVersion><version>[17,)</version></requireJavaVersion>
              </rules>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

Then, in child modules remove version tags where managed and inherit plugins.


## 3) Configuration and profiles

Goals: clear separation of dev/test/prod and self-documenting configuration.

Actions:
- Use application.yml and profiles:
  - application.yml (shared)
  - application-dev.yml (H2, localhost Axon)
  - application-test.yml (Testcontainers)
  - application-prod.yml (Postgres/MySQL, remote Axon, hardened settings)
- Externalize secrets via environment variables and Spring Cloud Config (optional).
- Standardize Axon properties across services:
  - axon.axonserver.servers
  - axon.eventhandling.processors.<name>.mode=tracking
  - axon.eventhandling.processors.<name>.initial-segment-count=1..n
  - axon.eventhandling.token-store.
- Database migrations with Flyway or Liquibase for query models.

Example properties (dev):

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:demo
  h2:
    console:
      enabled: true
axon:
  axonserver:
    servers: localhost:8124
  serializer:
    general: xstream
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```


## 4) Observability (metrics, logs, tracing, health)

Actions:
- Add Spring Boot Actuator to all services.
- Metrics: Micrometer + Prometheus endpoint; add Grafana dashboard.
- Tracing: OpenTelemetry (OTel) with Zipkin/Jaeger; propagate trace IDs through Gateway and React.
- Structured logging: Logback JSON encoder; include traceId/spanId in MDC.
- Health checks: readiness/liveness endpoints; Docker/Compose healthchecks for all services.

Quick add (dependencies):
- org.springframework.boot:spring-boot-starter-actuator
- io.micrometer:micrometer-registry-prometheus
- io.opentelemetry:opentelemetry-exporter-zipkin (or use Spring Cloud Sleuth equivalent for Boot 3 via Micrometer Tracing)


## 5) Testing strategy

- Unit tests: domain logic, Aggregates, Sagas
  - Use Axon test fixtures (AggregateTestFixture, SagaTestFixture)
- Slice tests: @DataJpaTest for projections, @WebMvcTest for controllers
- Contract tests: Pact or Spring Cloud Contract between Gateway and services
- Integration tests: Testcontainers (Axon Server, Postgres/H2, Eureka) to run end-to-end
- UI tests: React Testing Library and Cypress for E2E

Example: Axon aggregate fixture
```java
var fixture = new AggregateTestFixture<>(TalentRequestAggregate.class);
fixture.givenNoPriorActivity()
       .when(new CreateTalentRequestCommand("id1", ...))
       .expectEvents(new TalentRequestCreatedEvent("id1", ...));
```

Example: Testcontainers (JUnit 5)
```java
@Container
static GenericContainer<?> axon = new GenericContainer<>(DockerImageName.parse("axoniq/axonserver:latest"))
  .withExposedPorts(8024, 8124);
```


## 6) Resilience and routing

- Gateway: add Resilience4j for circuit breaking, rate limiting, and retries.
- Eureka: set lease renewal/expiration appropriately; prefer IP address in dev.
- Axon: tracking processors with token store in DB; enable retries and DLQ (Axon 4.9 has Dead Letter Queue support).
- Timeouts and backoff policies for commands/queries.

Sample Gateway route with circuit breaker (YAML):
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: talent-request
          uri: lb://talent-request-service
          predicates:
            - Path=/talent-request/**
          filters:
            - name: CircuitBreaker
              args:
                name: tr-cb
                fallbackUri: forward:/fallback
```


## 7) Events, projections, and versioning

- Event versioning strategy: add revision metadata and Upcasters.
- Snapshotting for large aggregates; tune thresholds.
- Query model rebuild strategy: from scratch or from snapshot/last token.
- Idempotency in event handlers to handle replays.
- Use persistent DB for projections in non-demo mode (Postgres) with indices.


## 8) Docker and local orchestration

Extend docker-compose to run the whole stack:
- Add Eureka, Gateway, and each service with healthchecks and dependencies
- Use a shared network, assign service names, and externalize ports
- Optional: Postgres services for projections

Example extensions (sketch):
```yaml
services:
  eureka:
    image: your/eureka:latest
    ports: ["8761:8761"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8761"]
  api-gateway:
    image: your/gateway:latest
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka:8761/eureka
    depends_on: [eureka]
  talent-request-service:
    image: your/tr-service:latest
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka:8761/eureka
      - AXON_AXONSERVER_SERVERS=axonserver:8124
    depends_on: [eureka, axonserver]
```

Tip: Use .env file to centralize ports and versions.


## 9) CI/CD pipeline

- Build matrix: backend modules and front-end
- Caching: Maven and npm caches
- Static analysis: Spotless, Checkstyle, PMD, OWASP Dependency-Check or Snyk
- Tests: unit + integration + UI
- Docker build and push to registry
- Security scans: Trivy image scan
- Artifacts: SBOM generation (CycloneDX)

GitHub Actions workflow outline:
```yaml
name: ci
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17', cache: 'maven' }
      - name: Build backend
        run: mvn -f tams-backe-end-axon-sb3/pom.xml -B -ntp clean verify
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm', cache-dependency-path: 'tams-uat-front-end-react/package-lock.json' }
      - name: Build front-end
        run: |
          cd tams-uat-front-end-react
          npm ci
          npm run build
```


## 10) Security baseline

- Authentication/Authorization via OIDC (Keycloak) across Gateway and services
- Propagate JWT roles to services; method-level security @PreAuthorize
- CORS policy: restrict origins and methods in Gateway and services
- Secrets: never commit; use env vars, Vault, or GitHub Actions secrets
- HTTPS locally (mkcert) and in production; HSTS at Gateway
- Dependency scanning and container image scanning in CI


## 11) Front-end improvements (React)

- Central API client with Axios + interceptors for auth and tracing headers
- Use Redux Toolkit Query (RTK Query) for API data fetching/caching
- Environment separation: .env.development, .env.production
- UI state monitoring: show connection status to Axon/Gateway
- Component Storybook for visual documentation
- Testing: Jest + React Testing Library, Cypress E2E

Example .env.development:
```
REACT_APP_API_BASE=http://localhost:8080
REACT_APP_AXON_DASHBOARD=http://localhost:8024
```


## 12) Production-readiness simulation

- Health/readiness probes and graceful shutdown (server.shutdown=graceful)
- JVM sizing and container-friendly flags
- Horizontal scaling: configure Axon tracking processors concurrency and segments
- Split command and query workloads into dedicated microservices (optional)
- Kubernetes manifests or Helm chart (Deployment, Service, Ingress, ConfigMap, Secret)


## 13) Demo data and runbook

- Seed script that emits demo events (commands) to populate projections
- Postman or Bruno collection with sample requests
- One-command startup for demo: docker compose up -d eureka axonserver gateway services; or ./run-demo.ps1
- Troubleshooting guide: common failure modes and where to look (Eureka UI, Axon UI, logs)


## 14) Code quality and style

- Apply Spotless (formatting) and Checkstyle rules in parent POM
- Consistent package naming and module boundaries
- DTO validation with Jakarta Validation (e.g., @NotBlank)
- Mapping with MapStruct for clean separation between API and domain
- Error handling: standardized error response model from Gateway and services

Spotless example (parent POM):
```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>2.43.0</version>
  <configuration>
    <java>
      <googleJavaFormat/>
      <removeUnusedImports/>
      <importOrder />
    </java>
  </configuration>
  <executions>
    <execution>
      <goals><goal>apply</goal><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```


## 15) Axon-specific tuning tips

- Prefer Jackson serializer for production (better performance and versioning support) and define upcasters; keep XStream only for demo if desired
- Configure Dead Letter Queue for problematic events/commands
- Use a persistent token store (JPA/JDBC) so tracking processors can resume across restarts
- Monitor backlog and latency in Axon Server UI; scale processors by increasing segment count

Example JPA token store bean:
```java
@Bean
public TokenStore tokenStore(EntityManagerProvider emp, TransactionManager tm) {
  return JpaTokenStore.builder().entityManagerProvider(emp).transactionManager(tm).build();
}
```


## 16) Roadmap (suggested order)

1) Observability basics (Actuator + Prometheus + logs) and per-service Swagger UI
2) Parent POM dependencyManagement and linting (Spotless)
3) Profiles and Flyway; switch projections to Postgres (optionally via Docker)
4) Testcontainers for integration tests; Axon test fixtures coverage
5) CI GitHub Actions pipeline
6) Gateway resilience patterns (Resilience4j)
7) Security with Keycloak and JWT propagation
8) Full docker-compose for entire stack and demo seeding script
9) Event upcasting + snapshotting for stable evolution


## 17) References

- Axon Framework: https://docs.axoniq.io/
- Spring Boot Actuator: https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html
- Micrometer + Prometheus: https://micrometer.io/
- OpenTelemetry: https://opentelemetry.io/
- Spring Cloud Gateway: https://spring.io/projects/spring-cloud-gateway
- Eureka: https://spring.io/projects/spring-cloud-netflix
- Testcontainers: https://testcontainers.com/
- Springdoc OpenAPI: https://springdoc.org/
- Resilience4j: https://resilience4j.readme.io/
- Spotless: https://github.com/diffplug/spotless
- MapStruct: https://mapstruct.org/
