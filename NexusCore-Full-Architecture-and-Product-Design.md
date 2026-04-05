# NexusCore -- Enterprise Operations Intelligence Platform

## Full Architecture & Product Design

---

## 1. PRODUCT VISION

**What this actually is:** An enterprise-grade orchestration platform that unifies database operations, system administration, secure external communication, and AI-augmented knowledge retrieval into a single programmable surface. Think of it as the control plane for IT operations -- a layer that sits between your teams and the sprawling mess of databases, servers, external services, and operational knowledge that every enterprise accumulates.

**Why this is a product, not just a tool:**

Most enterprises run 3-8 different database engines, dozens of servers, and hundreds of integration points. The operational knowledge for managing all of this lives in wikis nobody reads, Slack threads that disappear, and the heads of senior engineers who eventually leave. NexusCore solves three real business problems:

1. **Fragmented operations tooling** -- DBA tools don't talk to monitoring tools don't talk to communication tools. NexusCore provides a unified adapter layer that abstracts all of them behind clean contracts.

2. **Knowledge decay** -- When a senior DBA resolves an incident at 2 AM, that resolution is lost. With the vector/RAG layer, NexusCore captures, embeds, and retrieves that knowledge for future incidents automatically.

3. **Integration brittleness** -- Every point-to-point integration is a liability. NexusCore's adapter architecture turns each integration into a pluggable module with its own circuit breaker, health check, and audit trail.

**Monetization paths (in order of realism):**

| Path | Model | Timeline | Revenue Type |
|---|---|---|---|
| Internal platform | Cost center (saves headcount) | Immediate | Cost avoidance |
| Enterprise license | Per-node or per-adapter seat license | 6-12 months | Recurring |
| Adapter marketplace | Revenue share on third-party adapters | 12-18 months | Platform tax |
| AI/RAG premium tier | Per-query or per-seat for AI features | 12-18 months | High-margin recurring |
| Managed SaaS | Multi-tenant hosted offering for SMBs | 18-24 months | MRR |
| Professional services | Custom adapter dev, onboarding, training | Immediate | Services margin |

**The hidden product angle:** The adapter marketplace is the real moat. Once third parties build adapters for NexusCore (SAP, Salesforce, ServiceNow connectors), switching costs become enormous. This is the MuleSoft/Zapier playbook applied to operations infrastructure.

---

## 2. SYSTEM ARCHITECTURE

### Architecture Style: Hexagonal (Ports & Adapters)

I am deliberately NOT recommending microservices. Here is why:

| Approach | Verdict | Reasoning |
|---|---|---|
| Monolith | Too rigid | Can't deploy adapters independently |
| Microservices | Premature | Operational overhead kills a small team; you don't have the traffic to justify it yet |
| **Modular monolith** | **Correct** | Single deployment unit, but strict module boundaries enforce adapter independence. Extract to services later when traffic/team size demands it. |
| Hexagonal + modular monolith | **Optimal** | The hexagonal pattern forces clean port interfaces. The modular monolith keeps deployment simple. You get the architectural benefits of microservices without the operational tax. |

### High-Level Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              INBOUND PORTS                   │
                    │  REST API │ gRPC │ CLI │ WebSocket │ Events │
                    └──────────────────┬──────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────┐
                    │           CORE DOMAIN SERVICES               │
                    │                                              │
                    │  ┌─────────────┐ ┌──────────────────────┐   │
                    │  │  Support    │ │  Orchestration       │   │
                    │  │  Workflow   │ │  Engine              │   │
                    │  │  Engine     │ │  (Pipeline/Routing)  │   │
                    │  └─────────────┘ └──────────────────────┘   │
                    │                                              │
                    │  ┌─────────────┐ ┌──────────────────────┐   │
                    │  │  Adapter    │ │  Event Bus           │   │
                    │  │  Registry & │ │  (Audit, Metrics,    │   │
                    │  │  Router     │ │   Alerting)          │   │
                    │  └─────────────┘ └──────────────────────┘   │
                    │                                              │
                    └──────────────────┬──────────────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
┌─────────▼──────────┐  ┌─────────────▼───────────┐  ┌─────────────▼──────────┐
│   DATABASE PORTS   │  │      OS PORTS           │  │   EXTERNAL PORTS       │
│                    │  │                         │  │                        │
│ PostgreSQL Adapter │  │ Command Executor        │  │ TLS Client Adapter     │
│ MySQL Adapter      │  │ Process Monitor         │  │ mTLS Mutual Auth       │
│ SQL Server Adapter │  │ File Operations         │  │ Secure API Gateway     │
│ Oracle Adapter     │  │ Service Health          │  │ Machine Identity       │
│ (future adapters)  │  │ Env Discovery           │  │ (future integrations)  │
└────────────────────┘  └─────────────────────────┘  └────────────────────────┘
          │
┌─────────▼──────────┐  ┌─────────────────────────┐
│   VECTOR / AI PORT │  │   OBSERVABILITY PORT     │
│                    │  │                          │
│ pgvector Store     │  │ Micrometer Metrics       │
│ Embedding Pipeline │  │ OpenTelemetry Tracing    │
│ Semantic Retriever │  │ Structured Logging       │
│ RAG Orchestrator   │  │ Health Aggregator        │
└────────────────────┘  └──────────────────────────┘
```

### Key Architectural Properties

**Internal event bus** -- When a database query executes, the adapter does NOT directly call the audit logger, the metrics collector, or the alerting system. Instead, it publishes a `QueryExecutedEvent`. Subscribers react independently. This gives us:

- Zero coupling between adapters and cross-cutting concerns
- Easy addition of new subscribers (billing, compliance, analytics) without touching adapter code
- Natural async processing for non-critical paths (audit writes, metric updates)

**Adapter registry with capabilities introspection** -- Each adapter registers itself with a `capabilities()` descriptor. The orchestration engine queries capabilities before routing. This means the platform can automatically discover what each adapter supports, generate accurate API documentation, and route requests only to capable adapters.

**Virtual threads throughout** -- Every I/O operation (DB query, OS command, external call, vector search) runs on a virtual thread. This eliminates the traditional thread pool sizing problem. A single NexusCore instance can handle thousands of concurrent operations without any pool tuning.

---

## 3. CORE MODULES

### Module Dependency Graph

```
nexuscore-core  (depends on: nothing -- pure domain)
    ↑
nexuscore-security  (depends on: core)
    ↑
nexuscore-adapter-db-common  (depends on: core, security)
    ↑
nexuscore-adapter-db-postgresql  (depends on: db-common)
nexuscore-adapter-db-mysql       (depends on: db-common)
nexuscore-adapter-db-sqlserver   (depends on: db-common)
nexuscore-adapter-db-oracle      (depends on: db-common)
    ↑
nexuscore-adapter-os       (depends on: core, security)
nexuscore-adapter-external (depends on: core, security)
nexuscore-vector           (depends on: core, adapter-db-postgresql)
nexuscore-observability    (depends on: core)
nexuscore-workflow         (depends on: core)
nexuscore-api              (depends on: core, security, workflow)
    ↑
nexuscore-platform  (depends on: everything -- assembly module)
nexuscore-test      (depends on: everything -- test-only)
```

### Module Details

| Module | Purpose | Key Classes |
|---|---|---|
| `nexuscore-core` | Domain model, port interfaces, domain services. ZERO infrastructure dependencies. | `DatabasePort`, `OsPlatformPort`, `ExternalComPort`, `VectorPort`, `AdapterRegistry`, `AdapterRouter`, domain events, value objects |
| `nexuscore-security` | TLS config, mTLS, auth, RBAC, machine identity, audit | `TlsContextFactory`, `MachineIdentityProvider`, `RbacEnforcer`, `AuditLogger`, `SecretsProvider` SPI |
| `nexuscore-adapter-db-common` | Shared utilities for all DB adapters | `ConnectionPoolManager`, `QuerySanitizer`, `SchemaIntrospector`, `CredentialResolver`, `DatabaseHealthProbe` |
| `nexuscore-adapter-db-postgresql` | PostgreSQL-specific adapter | `PostgreSQLAdapter implements DatabasePort`, dialect handling, PG-specific features |
| `nexuscore-adapter-db-mysql` | MySQL-specific adapter | `MySQLAdapter implements DatabasePort` |
| `nexuscore-adapter-db-sqlserver` | SQL Server-specific adapter | `SQLServerAdapter implements DatabasePort` |
| `nexuscore-adapter-db-oracle` | Oracle-specific adapter | `OracleAdapter implements DatabasePort` |
| `nexuscore-adapter-os` | OS interaction layer | `OsAdapter implements OsPlatformPort`, `CommandExecutor`, `ProcessMonitor`, `FileOperator`, `ServiceHealthChecker` |
| `nexuscore-adapter-external` | Secure external communication | `TlsClientAdapter implements ExternalComPort`, `MtlsHandshakeManager`, `EndpointRegistry` |
| `nexuscore-vector` | pgvector + AI/RAG | `PgVectorStore implements VectorPort`, `EmbeddingPipeline`, `SemanticRetriever`, `RagOrchestrator` |
| `nexuscore-observability` | Metrics, tracing, health | `MetricsRegistry`, `TraceContextPropagator`, `HealthAggregator`, `StructuredLogEnricher` |
| `nexuscore-workflow` | Orchestration engine | `WorkflowEngine`, `PipelineDefinition`, `StepExecutor`, `WorkflowScheduler` |
| `nexuscore-api` | REST/gRPC/WebSocket surface | Controllers, DTOs, request validators, response mappers, API versioning |
| `nexuscore-platform` | Spring Boot assembly | `NexusCoreApplication`, auto-configuration, profile-based adapter loading |
| `nexuscore-test` | Test infrastructure | Custom Testcontainers, adapter test harnesses, fixture generators |

---

## 4. BEST TECHNOLOGY STACK

### Core Runtime

| Concern | Technology | Version | Why This Over Alternatives |
|---|---|---|---|
| Language | Java | 21 LTS | Virtual threads, records, sealed interfaces, pattern matching. Kotlin is tempting but adds a dependency and reduces hiring pool. |
| Framework | Spring Boot | 3.3.x | Nothing else comes close in ecosystem maturity. Quarkus startup advantage is irrelevant for a long-running platform service. |
| Build | Gradle | 8.x (Kotlin DSL) | Multi-module support is dramatically better than Maven. Version catalogs for dependency management. |

### Data Access

| Concern | Technology | Why |
|---|---|---|
| Platform entities | Spring Data JPA (Hibernate 6) | Productivity for the platform's own 15-20 entities |
| Adapter queries | JdbcTemplate + plain JDBC | Raw SQL control is non-negotiable when you're executing arbitrary queries across 4+ database engines with different SQL dialects |
| Migrations | Flyway | Simpler mental model than Liquibase, version-based |
| Connection pool | HikariCP | Fastest, most battle-tested pool. Default in Spring Boot. |
| Vectors | pgvector + pgvector-java | Native PostgreSQL vector type, HNSW/IVFFlat indexes, no external vector DB needed |

**Why not MyBatis?** MyBatis sits in a middle ground -- more control than JPA, less than raw JDBC. For the adapter layer where you're executing user-provided or dynamically constructed queries across heterogeneous databases, you need the full control of JDBC. MyBatis's XML mapping files add ceremony without benefit when you're not mapping to fixed entity classes. For the platform's own entities, JPA is more productive. MyBatis would be the right choice if this were a single-database application with complex but static queries.

### Resilience & Communication

| Concern | Technology | Why |
|---|---|---|
| Circuit breaking | Resilience4j | Lightweight, composable, no Hystrix baggage |
| HTTP client | Spring WebClient (reactive) or Java HttpClient | Virtual-thread-friendly, TLS-configurable |
| gRPC | grpc-java + grpc-spring-boot-starter | High-throughput inter-service communication, future-ready |
| Messaging (optional) | Apache Kafka or RabbitMQ | Event streaming if async patterns demand it. Start without it. |
| Scheduling | Spring Scheduler + Quartz (for persistence) | Simple scheduled jobs escalate to Quartz when you need cluster-safe persistence |

### Security

| Concern | Technology | Why |
|---|---|---|
| TLS/mTLS | Java SSLContext + Bouncy Castle (for cert generation) | JDK's built-in TLS is production-grade; Bouncy Castle for programmatic cert ops |
| Authentication | Spring Security 6 + JWT | Standard bearer token auth for API consumers |
| Secrets | HashiCorp Vault (or Azure Key Vault via Spring Cloud Vault) | Dynamic secrets, automatic rotation, audit trail on access |
| Certificate mgmt | Cert-Manager (K8s) or ACME protocol | Automated cert renewal in production |

### Observability

| Concern | Technology | Why |
|---|---|---|
| Metrics | Micrometer | Vendor-neutral facade. Export to Prometheus, Datadog, CloudWatch, whatever. |
| Tracing | OpenTelemetry Java agent | Automatic instrumentation for Spring, JDBC, HTTP. Correlation IDs across adapters. |
| Logging | SLF4J + Logback with structured JSON | Machine-parseable logs, shipped to ELK/Loki/Splunk |
| Health | Spring Boot Actuator + custom HealthIndicators per adapter | /actuator/health with per-adapter granularity |
| Dashboarding | Grafana | Free, integrates with Prometheus/Loki/Tempo |

### Testing

| Concern | Technology | Why |
|---|---|---|
| Unit | JUnit 5 + Mockito | Standard, well-known |
| Integration | Testcontainers | Real PostgreSQL, MySQL, SQL Server, Oracle containers. No mocks for DB tests. |
| Contract | Spring Cloud Contract or Pact | API backward compatibility |
| Architecture | ArchUnit | Enforce module boundaries at build time (e.g., "core must not import adapter classes") |

### Deployment

| Concern | Technology | Why |
|---|---|---|
| Containerization | Docker (multi-stage build, distroless base) | Minimal attack surface, ~80MB image |
| Orchestration | Kubernetes (or Docker Compose for dev/small deployments) | Production standard |
| CI/CD | GitHub Actions or GitLab CI | Automated build, test, deploy pipeline |
| Config | Spring Cloud Config or Kubernetes ConfigMaps/Secrets | Externalized, environment-specific config |

---

## 5. ADAPTER ARCHITECTURE

This is the most important section. The adapter layer is what makes NexusCore a platform rather than just an app.

### Core Port Interfaces (in `nexuscore-core`)

```java
// === DATABASE PORT ===

public sealed interface DatabasePort permits
        ReadableDatabasePort, WritableDatabasePort, IntrospectableDatabasePort {

    DatabaseCapabilities capabilities();
    ConnectionTestResult testConnection(TargetDatabase target);
    HealthCheckResult healthCheck(TargetDatabase target);
}

public non-sealed interface ReadableDatabasePort extends DatabasePort {
    QueryResult executeQuery(TargetDatabase target, ParameterizedQuery query);
    StreamedQueryResult streamQuery(TargetDatabase target, ParameterizedQuery query);
}

public non-sealed interface WritableDatabasePort extends DatabasePort {
    MutationResult executeMutation(TargetDatabase target, ParameterizedQuery mutation);
    BatchMutationResult executeBatch(TargetDatabase target, List<ParameterizedQuery> mutations);
}

public non-sealed interface IntrospectableDatabasePort extends DatabasePort {
    SchemaMetadata introspectSchema(TargetDatabase target, String schemaName);
    List<TableMetadata> listTables(TargetDatabase target, String schemaName);
    TableMetadata describeTable(TargetDatabase target, String schemaName, String tableName);
}
```

```java
// === OS PLATFORM PORT ===

public interface OsPlatformPort {
    CommandResult executeCommand(CommandSpec command);
    ProcessSnapshot getProcess(ProcessIdentifier pid);
    List<ProcessSnapshot> listProcesses(ProcessFilter filter);
    List<ServiceStatus> listServices(ServiceFilter filter);
    FileOperationResult performFileOp(FileOperation operation);
    SystemMetrics collectMetrics();
    OsCapabilities capabilities();
}
```

```java
// === EXTERNAL COMMUNICATION PORT ===

public interface ExternalCommunicationPort {
    <T> SecureResponse<T> send(SecureRequest request, TypeReference<T> responseType);
    EndpointHealth probeEndpoint(EndpointDescriptor endpoint);
    void registerEndpoint(EndpointDescriptor endpoint, TlsProfile profile);
    CommunicationCapabilities capabilities();
}
```

```java
// === VECTOR / AI PORT ===

public interface VectorPort {
    void storeEmbedding(EmbeddingRecord record);
    void storeBatch(List<EmbeddingRecord> records);
    List<SimilarityResult> searchSimilar(float[] queryVector, int topK, SearchFilter filter);
    void deleteByMetadata(Map<String, String> metadata);
    VectorCapabilities capabilities();
}
```

### Capabilities Pattern (the key extensibility trick)

Every port returns a capabilities object. This is what makes the platform self-describing:

```java
public record DatabaseCapabilities(
    DatabaseType databaseType,
    String adapterVersion,
    Set<SqlFeature> supportedFeatures,   // STREAMING, BATCH, DDL, EXPLAIN_PLAN, etc.
    Set<DataType> supportedTypes,
    boolean supportsSchemaIntrospection,
    boolean supportsStreamingResults,
    boolean supportsCancelation,
    int maxConcurrentConnections
) {}
```

The orchestration engine queries capabilities before routing:

```java
public class AdapterRouter {

    private final AdapterRegistry registry;

    public QueryResult route(TargetDatabase target, ParameterizedQuery query) {
        DatabasePort adapter = registry.resolve(target.databaseType());

        if (!(adapter instanceof ReadableDatabasePort readable)) {
            throw new AdapterCapabilityException(
                "Adapter %s does not support read operations".formatted(target.databaseType()));
        }

        if (query.isStreaming() && !adapter.capabilities().supportsStreamingResults()) {
            throw new AdapterCapabilityException(
                "Adapter %s does not support streaming".formatted(target.databaseType()));
        }

        return readable.executeQuery(target, query);
    }
}
```

### Adapter Registration (SPI-style with Spring)

```java
// Each adapter module declares a Spring @Configuration
@Configuration
@ConditionalOnProperty(name = "nexuscore.adapters.postgresql.enabled", havingValue = "true", matchIfMissing = true)
public class PostgreSQLAdapterAutoConfiguration {

    @Bean
    public DatabasePort postgresqlAdapter(
            ConnectionPoolManager poolManager,
            MeterRegistry meterRegistry) {
        return new PostgreSQLAdapter(poolManager, meterRegistry);
    }
}
```

```java
// The registry collects all adapters via Spring injection
@Component
public class SpringAdapterRegistry implements AdapterRegistry {

    private final Map<DatabaseType, DatabasePort> databaseAdapters;

    public SpringAdapterRegistry(List<DatabasePort> adapters) {
        this.databaseAdapters = adapters.stream()
            .collect(Collectors.toMap(
                a -> a.capabilities().databaseType(),
                Function.identity()));
    }

    @Override
    public DatabasePort resolve(DatabaseType type) {
        return Optional.ofNullable(databaseAdapters.get(type))
            .orElseThrow(() -> new AdapterNotFoundException(
                "No adapter registered for database type: " + type));
    }
}
```

### Extension Strategy

Adding a new database adapter (e.g., CockroachDB):

1. Create a new Gradle module: `nexuscore-adapter-db-cockroachdb`
2. Add dependency on `nexuscore-adapter-db-common`
3. Implement `ReadableDatabasePort`, `WritableDatabasePort` (and optionally `IntrospectableDatabasePort`)
4. Create a `CockroachDBAdapterAutoConfiguration` with `@ConditionalOnProperty`
5. Add the JDBC driver to the module's dependencies
6. Write Testcontainers-based integration tests
7. Drop the JAR on the classpath -- Spring auto-discovers it

No changes to core. No changes to other adapters. No changes to the API layer. The registry picks it up automatically.

This is the adapter marketplace model: third parties can build and distribute adapter JARs independently.

---

## 6. PACKAGE / FOLDER STRUCTURE

```
nexuscore/
├── settings.gradle.kts
├── build.gradle.kts                          # Root: dependency version catalog, shared config
├── gradle/
│   └── libs.versions.toml                    # Gradle version catalog
│
├── nexuscore-core/
│   └── src/main/java/com/nexuscore/core/
│       ├── domain/
│       │   ├── model/                        # TargetDatabase, ParameterizedQuery, EmbeddingRecord
│       │   ├── event/                        # QueryExecutedEvent, CommandExecutedEvent, AuditEvent
│       │   └── exception/                    # AdapterNotFoundException, CapabilityException
│       ├── port/
│       │   ├── inbound/                      # Use-case interfaces (driven side)
│       │   │   ├── DatabaseQueryUseCase.java
│       │   │   ├── OsCommandUseCase.java
│       │   │   ├── SecureCommunicationUseCase.java
│       │   │   └── SemanticSearchUseCase.java
│       │   └── outbound/                     # SPI interfaces (driving side)
│       │       ├── DatabasePort.java
│       │       ├── OsPlatformPort.java
│       │       ├── ExternalCommunicationPort.java
│       │       ├── VectorPort.java
│       │       ├── SecretsProvider.java
│       │       └── AuditSink.java
│       └── service/                          # Domain services (pure logic, port injection)
│           ├── AdapterRegistry.java
│           ├── AdapterRouter.java
│           ├── QueryOrchestrator.java
│           └── SupportWorkflowService.java
│
├── nexuscore-security/
│   └── src/main/java/com/nexuscore/security/
│       ├── tls/
│       │   ├── TlsContextFactory.java
│       │   ├── MtlsHandshakeManager.java
│       │   └── CertificateStore.java
│       ├── auth/
│       │   ├── JwtTokenProvider.java
│       │   ├── RbacEnforcer.java
│       │   └── ApiKeyAuthenticator.java
│       ├── identity/
│       │   ├── MachineIdentityProvider.java
│       │   └── HostContextResolver.java
│       ├── secrets/
│       │   ├── VaultSecretsProvider.java
│       │   ├── AzureKeyVaultSecretsProvider.java
│       │   └── EnvironmentSecretsProvider.java
│       └── audit/
│           ├── AuditLogWriter.java
│           └── AuditEventEnricher.java
│
├── nexuscore-adapter-db-common/
│   └── src/main/java/com/nexuscore/adapter/db/common/
│       ├── ConnectionPoolManager.java        # Creates/manages HikariCP pools per target
│       ├── CredentialResolver.java           # Resolves credentials from SecretsProvider
│       ├── QuerySanitizer.java               # SQL injection prevention
│       ├── ResultSetMapper.java              # Generic ResultSet → QueryResult mapping
│       ├── DatabaseHealthProbe.java          # Standard health check logic
│       └── DialectNormalizer.java            # Cross-dialect type mapping
│
├── nexuscore-adapter-db-postgresql/
│   └── src/main/java/com/nexuscore/adapter/db/postgresql/
│       ├── PostgreSQLAdapter.java
│       ├── PostgreSQLDialect.java
│       └── PostgreSQLAdapterAutoConfiguration.java
│
├── nexuscore-adapter-db-mysql/
│   └── src/main/java/com/nexuscore/adapter/db/mysql/
│       ├── MySQLAdapter.java
│       ├── MySQLDialect.java
│       └── MySQLAdapterAutoConfiguration.java
│
├── nexuscore-adapter-db-sqlserver/
│   └── src/main/java/com/nexuscore/adapter/db/sqlserver/
│       ├── SQLServerAdapter.java
│       ├── SQLServerDialect.java
│       └── SQLServerAdapterAutoConfiguration.java
│
├── nexuscore-adapter-db-oracle/
│   └── src/main/java/com/nexuscore/adapter/db/oracle/
│       ├── OracleAdapter.java
│       ├── OracleDialect.java
│       └── OracleAdapterAutoConfiguration.java
│
├── nexuscore-adapter-os/
│   └── src/main/java/com/nexuscore/adapter/os/
│       ├── OsAdapter.java
│       ├── command/
│       │   ├── CommandExecutor.java
│       │   ├── SandboxedCommandExecutor.java # Process isolation, timeout enforcement
│       │   └── CommandWhitelist.java         # Allowed command registry
│       ├── process/
│       │   └── ProcessMonitor.java
│       ├── file/
│       │   └── SecureFileOperator.java       # Path traversal prevention
│       ├── service/
│       │   └── ServiceHealthChecker.java
│       └── OsAdapterAutoConfiguration.java
│
├── nexuscore-adapter-external/
│   └── src/main/java/com/nexuscore/adapter/external/
│       ├── TlsClientAdapter.java
│       ├── EndpointRegistry.java
│       ├── SecureRequestBuilder.java
│       ├── MachineContextEnricher.java       # Adds hostname/userID/machine context
│       └── ExternalAdapterAutoConfiguration.java
│
├── nexuscore-vector/
│   └── src/main/java/com/nexuscore/vector/
│       ├── store/
│       │   └── PgVectorStore.java            # VectorPort implementation
│       ├── embedding/
│       │   ├── EmbeddingProvider.java         # SPI for embedding generation
│       │   ├── OpenAiEmbeddingProvider.java
│       │   ├── OllamaEmbeddingProvider.java
│       │   └── EmbeddingPipeline.java
│       ├── retrieval/
│       │   ├── SemanticRetriever.java
│       │   └── HybridRetriever.java          # Combines vector + keyword search
│       ├── rag/
│       │   ├── RagOrchestrator.java
│       │   ├── ContextAssembler.java
│       │   └── PromptTemplate.java
│       └── VectorAutoConfiguration.java
│
├── nexuscore-observability/
│   └── src/main/java/com/nexuscore/observability/
│       ├── metrics/
│       │   ├── AdapterMetricsCollector.java
│       │   └── BusinessMetricsCollector.java
│       ├── tracing/
│       │   └── TraceContextPropagator.java
│       ├── health/
│       │   ├── HealthAggregator.java
│       │   └── AdapterHealthIndicator.java
│       └── logging/
│           └── StructuredLogEnricher.java
│
├── nexuscore-workflow/
│   └── src/main/java/com/nexuscore/workflow/
│       ├── engine/
│       │   ├── WorkflowEngine.java
│       │   └── StepExecutor.java
│       ├── definition/
│       │   ├── PipelineDefinition.java
│       │   └── StepDefinition.java
│       └── scheduler/
│           └── WorkflowScheduler.java
│
├── nexuscore-api/
│   └── src/main/java/com/nexuscore/api/
│       ├── rest/
│       │   ├── v1/
│       │   │   ├── DatabaseController.java
│       │   │   ├── OsController.java
│       │   │   ├── ExternalComController.java
│       │   │   ├── VectorController.java
│       │   │   └── AdminController.java
│       │   ├── dto/                          # Request/response DTOs
│       │   ├── mapper/                       # Domain ↔ DTO mappers (MapStruct)
│       │   └── validation/                   # Custom validators
│       ├── websocket/
│       │   └── StreamingResultHandler.java
│       └── error/
│           └── GlobalExceptionHandler.java
│
├── nexuscore-platform/
│   └── src/main/java/com/nexuscore/platform/
│       ├── NexusCoreApplication.java         # @SpringBootApplication
│       └── config/
│           ├── SecurityConfig.java
│           ├── JacksonConfig.java
│           ├── AsyncConfig.java              # Virtual thread executor
│           ├── ObservabilityConfig.java
│           └── AdapterLoadingConfig.java
│
├── nexuscore-test/
│   └── src/
│       ├── main/java/com/nexuscore/test/
│       │   ├── containers/                   # Custom Testcontainers (PG+pgvector, etc.)
│       │   ├── fixtures/                     # Test data generators
│       │   └── harness/                      # Adapter test harnesses
│       └── test/java/com/nexuscore/test/
│           └── architecture/
│               └── ModuleBoundaryTest.java   # ArchUnit rules
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml                    # Dev environment (PG, MySQL, MSSQL, Oracle)
│
└── docs/
    ├── architecture-decision-records/
    └── adapter-development-guide.md
```

---

## 7. DATABASE STRATEGY

### Two Distinct Database Concerns

**Platform database** (PostgreSQL with pgvector extension) -- stores NexusCore's own operational data:

| Schema | Tables | Purpose |
|---|---|---|
| `nexus_core` | `adapter_registrations`, `adapter_configs` | Which adapters are installed, their configuration |
| `nexus_core` | `target_databases`, `target_credentials_ref` | Registered target DB connection metadata (credentials stored in Vault, not DB) |
| `nexus_core` | `audit_events` | Immutable audit trail of every operation |
| `nexus_core` | `workflow_definitions`, `workflow_executions`, `workflow_steps` | Orchestration state |
| `nexus_core` | `api_keys`, `rbac_roles`, `rbac_permissions` | Authentication and authorization |
| `nexus_vectors` | `embeddings` | pgvector table with vector column, metadata JSONB, source reference |
| `nexus_vectors` | `embedding_sources` | Knowledge artifacts that have been embedded |

**Target databases** (any supported RDBMS) -- the databases NexusCore connects to on behalf of users. These are NOT owned by NexusCore. The adapter layer creates ephemeral connections to them.

### Dynamic Connection Pool Management

```java
public class ConnectionPoolManager {

    private final ConcurrentHashMap<String, HikariDataSource> pools = new ConcurrentHashMap<>();
    private final CredentialResolver credentialResolver;
    private final MeterRegistry meterRegistry;

    public DataSource getOrCreatePool(TargetDatabase target) {
        return pools.computeIfAbsent(target.connectionKey(), key -> {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl(target.jdbcUrl());

            DatabaseCredential cred = credentialResolver.resolve(target.credentialRef());
            config.setUsername(cred.username());
            config.setPassword(cred.password());

            config.setMaximumPoolSize(target.maxPoolSize().orElse(5));
            config.setMinimumIdle(1);
            config.setIdleTimeout(Duration.ofMinutes(10).toMillis());
            config.setMaxLifetime(Duration.ofMinutes(30).toMillis());
            config.setConnectionTimeout(Duration.ofSeconds(5).toMillis());
            config.setPoolName("nexus-" + target.connectionKey());

            config.setMetricRegistry(meterRegistry);

            return new HikariDataSource(config);
        });
    }

    @Scheduled(fixedDelay = 60_000)
    public void evictIdlePools() {
        pools.entrySet().removeIf(entry -> {
            HikariDataSource ds = entry.getValue();
            if (ds.getHikariPoolMXBean().getIdleConnections() == ds.getHikariPoolMXBean().getTotalConnections()
                    && ds.getHikariPoolMXBean().getTotalConnections() > 0) {
                ds.close();
                return true;
            }
            return false;
        });
    }
}
```

### Credential Handling (NEVER stored in the platform DB)

```
User registers a target database
    → NexusCore stores: JDBC URL, database type, credential REFERENCE (e.g., "vault:secret/data/prod-db-1")
    → NexusCore does NOT store the actual password
    → At connection time: CredentialResolver calls Vault/KV to fetch current credentials
    → Vault can rotate the password independently; NexusCore always fetches the latest
```

### Tenant/Client Isolation (when you productize)

Two strategies, choose based on scale:

| Strategy | How | When |
|---|---|---|
| Schema-per-tenant | Each tenant gets their own schema in the platform DB (`tenant_acme.audit_events`) | Up to ~100 tenants, simplifies operations |
| Database-per-tenant | Each tenant gets their own PostgreSQL database | 100+ tenants, stronger isolation, more complex ops |

For Phase 1-2, use a `tenant_id` column on every table (shared schema). This is the simplest starting point and works until you have a real multi-tenancy requirement.

### pgvector Schema Design

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE nexus_vectors.embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type     VARCHAR(50) NOT NULL,      -- 'support_ticket', 'runbook', 'resolution', 'kb_article'
    source_id       VARCHAR(255) NOT NULL,
    content_hash    VARCHAR(64) NOT NULL,       -- SHA-256 of original content (dedup)
    embedding       vector(1536) NOT NULL,      -- OpenAI ada-002 dimension; parameterize this
    metadata        JSONB NOT NULL DEFAULT '{}',
    tenant_id       VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_vector ON nexus_vectors.embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

CREATE INDEX idx_embeddings_source ON nexus_vectors.embeddings (source_type, source_id);
CREATE INDEX idx_embeddings_tenant ON nexus_vectors.embeddings (tenant_id);
CREATE INDEX idx_embeddings_metadata ON nexus_vectors.embeddings USING gin (metadata);
```

---

## 8. SECURITY DESIGN

### TLS / mTLS Communication Model

```
┌─────────────────┐         mTLS          ┌──────────────────┐
│   NexusCore      │◄────────────────────►│  External System  │
│   (has its own   │   Both sides present  │  (enterprise      │
│    X.509 cert)   │   certificates        │   client)         │
└─────────────────┘                        └──────────────────┘

┌─────────────────┐      Server TLS       ┌──────────────────┐
│   API Consumer   │─────────────────────►│  NexusCore API    │
│   (JWT bearer)   │   NexusCore presents  │  (validates JWT   │
│                  │   its server cert     │   + RBAC check)   │
└─────────────────┘                        └──────────────────┘
```

**Internal (API consumers to NexusCore):** Standard server-side TLS + JWT bearer tokens. RBAC enforced on every endpoint.

**External (NexusCore to external systems):** Mutual TLS. NexusCore presents its own X.509 certificate; the external system presents theirs. Both sides validate.

### Machine Identity Strategy

Each NexusCore instance has a unique machine identity:

```java
public record MachineIdentity(
    String instanceId,          // UUID generated at first boot, persisted
    String hostname,            // OS hostname
    String certificateFingerprint,  // SHA-256 of the instance's X.509 cert
    Instant certExpiresAt,
    String environmentTag       // "prod", "staging", "dev"
) {}
```

This identity is attached to every outbound request and every audit log entry. External systems can whitelist NexusCore instances by certificate fingerprint.

### Authentication Model (layered)

```
Request arrives
    → TLS termination (certificate validation)
    → Authentication (JWT validation OR API key lookup OR mTLS client cert)
    → Authorization (RBAC: does this principal have permission for this adapter + operation?)
    → Audit (log the attempt regardless of outcome)
    → Execute
```

### RBAC Design

```java
public record Permission(
    String resource,    // "database:postgresql", "os:command", "external:*", "vector:search"
    String action       // "read", "write", "execute", "admin"
) {}

public record Role(
    String name,        // "dba-readonly", "ops-admin", "api-consumer"
    Set<Permission> permissions
) {}
```

Fine-grained: you can grant a principal read-only access to PostgreSQL adapters but no access to OS commands. This is critical for enterprise customers who have strict separation of duties.

### Secrets Management

```java
public interface SecretsProvider {
    String resolveSecret(String secretRef);
    Optional<Instant> secretExpiresAt(String secretRef);
    void rotateSecret(String secretRef);  // trigger rotation if supported
}

// Implementations:
// VaultSecretsProvider     → HashiCorp Vault (preferred)
// AzureKvSecretsProvider   → Azure Key Vault
// EnvSecretsProvider       → Environment variables (dev/test only)
// EncryptedFileSecretsProvider → AES-encrypted file on disk (air-gapped environments)
```

### Audit Trail

Every state-changing operation produces an immutable audit event:

```java
public record AuditEvent(
    UUID eventId,
    Instant timestamp,
    MachineIdentity source,
    String principalId,
    String action,              // "database.query.execute", "os.command.execute"
    String resource,            // "postgresql:prod-db-1", "os:server-alpha"
    String outcome,             // "SUCCESS", "DENIED", "ERROR"
    Duration executionTime,
    Map<String, String> context // sanitized metadata (NO query content, NO credentials)
) {}
```

Audit events are written to the platform database AND optionally streamed to an external SIEM (Splunk, ELK) via the event bus.

### Zero-Trust Improvements (Phase 2+)

- Short-lived JWT tokens (15-minute expiry, refresh token rotation)
- Certificate pinning for external endpoints
- Network policy enforcement (only allow egress to registered endpoints)
- Encrypted at-rest for the platform database (PostgreSQL TDE or volume-level encryption)
- Regular credential rotation via Vault's dynamic secrets engine

---

## 9. API / COMMUNICATION DESIGN

### REST API (primary interface)

```
/api/v1/database
    POST   /query              Execute a query against a target database
    POST   /mutation           Execute a mutation (INSERT/UPDATE/DELETE)
    GET    /targets            List registered target databases
    POST   /targets            Register a new target database
    GET    /targets/{id}/health  Health check a target
    GET    /targets/{id}/schema  Introspect schema

/api/v1/os
    POST   /command            Execute an OS command
    GET    /processes          List running processes
    GET    /services           List services and health
    GET    /metrics            System metrics

/api/v1/external
    POST   /send               Send a secure request to an external endpoint
    GET    /endpoints          List registered external endpoints
    POST   /endpoints          Register an external endpoint
    GET    /endpoints/{id}/health  Health probe

/api/v1/vector
    POST   /embed              Generate and store embeddings for content
    POST   /search             Semantic similarity search
    DELETE /embeddings         Delete embeddings by metadata filter

/api/v1/admin
    GET    /adapters           List registered adapters and capabilities
    GET    /health             Aggregated health check
    GET    /audit              Query audit trail
```

### Request / Response Contract

```java
// Uniform request envelope
public record NexusRequest<T>(
    String requestId,           // Client-provided correlation ID
    String tenantId,            // Tenant context (if multi-tenant)
    T payload,
    Map<String, String> metadata
) {}

// Uniform response envelope
public record NexusResponse<T>(
    String requestId,
    String status,              // "SUCCESS", "ERROR", "PARTIAL"
    T data,
    ErrorDetail error,          // null on success
    ResponseMetadata metadata   // execution time, adapter used, etc.
) {}
```

### Sync vs Async

| Operation | Mode | Why |
|---|---|---|
| Database query (small) | Synchronous | Low latency, simple |
| Database query (large/streaming) | Async (WebSocket or SSE) | Avoid HTTP timeout, stream results |
| OS command (fast) | Synchronous | Sub-second commands are fine sync |
| OS command (long-running) | Async (poll or WebSocket) | Commands can take minutes |
| External communication | Synchronous with timeout + retry | Resilience4j handles retry |
| Vector search | Synchronous | pgvector queries are fast (<100ms) |
| Embedding generation | Async (fire-and-forget with callback) | Embedding API calls are slow (1-5s) |

### Internal Event Bus

```java
// Spring's ApplicationEventPublisher -- lightweight, in-process
public class QueryExecutedEvent extends ApplicationEvent {
    private final AuditEvent auditEvent;
    private final QueryMetrics queryMetrics;
    // ...
}

// Listeners are decoupled subscribers
@Component
public class AuditEventListener {
    @Async
    @EventListener
    public void onQueryExecuted(QueryExecutedEvent event) {
        auditLogWriter.write(event.getAuditEvent());
    }
}

@Component
public class MetricsEventListener {
    @EventListener
    public void onQueryExecuted(QueryExecutedEvent event) {
        meterRegistry.timer("nexus.query.duration",
            "database_type", event.getDatabaseType().name())
            .record(event.getExecutionTime());
    }
}
```

---

## 10. AI / VECTOR STRATEGY

### Architecture: Pluggable Embedding Pipeline

```
Content arrives (support ticket, runbook, resolution log, KB article)
    → ContentPreprocessor (clean, chunk, extract key segments)
    → EmbeddingProvider.embed(chunks)  ← pluggable: OpenAI, Ollama, Cohere, local model
    → PgVectorStore.storeBatch(embeddings)
    → Metadata indexed (source type, tenant, tags, timestamp)
```

### Semantic Retrieval Flow

```
User query: "How do I fix connection timeout on prod PostgreSQL?"
    → EmbeddingProvider.embed(query)
    → PgVectorStore.searchSimilar(queryVector, topK=10, filter={source_type: ["resolution", "runbook"]})
    → RerankingStep (optional: cross-encoder reranking for precision)
    → Return top 5 results with similarity scores and source references
```

### RAG Pipeline (Phase 3+)

```
User asks a support question
    → Semantic retrieval (as above) returns relevant context chunks
    → ContextAssembler builds a prompt:
        "Given the following relevant documentation and past resolutions:
         [chunk 1] [chunk 2] [chunk 3]
         Answer the user's question: {question}"
    → LLM generates an answer grounded in real organizational knowledge
    → Answer is presented with source citations
    → User feedback (helpful/not helpful) improves retrieval quality over time
```

### Support Workflow Augmentation

| Use Case | How NexusCore + AI Helps |
|---|---|
| Incident triage | Semantic search finds similar past incidents and their resolutions |
| Runbook discovery | Embed all runbooks; when an alert fires, auto-retrieve the relevant runbook |
| Knowledge capture | Embed resolution notes from resolved tickets; build organizational memory |
| Proactive alerts | Detect anomalous patterns via embeddings of monitoring data |
| New employee onboarding | AI-powered Q&A over operational knowledge base |

### Premium Feature Path

This is where the money is:

1. **Basic tier (free/included):** Keyword search over operational data, manual runbook lookup
2. **Pro tier:** Semantic search over embedded knowledge, automatic similar-incident retrieval
3. **Enterprise tier:** Full RAG with LLM-generated answers, custom model fine-tuning on organizational data, compliance-grade audit trail on AI usage

### Embedding Provider SPI

```java
public interface EmbeddingProvider {
    float[] embed(String text);
    List<float[]> embedBatch(List<String> texts);
    int dimensions();               // 1536 for ada-002, 384 for all-MiniLM, etc.
    String modelIdentifier();
    EmbeddingProviderCapabilities capabilities();
}
```

This allows swapping between OpenAI (high quality, paid), Ollama/local models (free, private), or custom fine-tuned models without touching any other code.

---

## 11. IMPLEMENTATION ROADMAP

### Phase 1: MVP (8-12 weeks)

**Goal:** A working platform that can connect to PostgreSQL, execute OS commands, and serve a REST API over TLS.

| Week | Deliverable |
|---|---|
| 1-2 | Gradle multi-module project setup, dependency management, CI pipeline, Docker Compose dev env |
| 3-4 | `nexuscore-core`: all port interfaces, domain model records, value objects, adapter registry |
| 5-6 | `nexuscore-adapter-db-postgresql`: full implementation with ConnectionPoolManager, Testcontainers tests |
| 7-8 | `nexuscore-adapter-os`: command execution, process listing, service health (sandboxed) |
| 9 | `nexuscore-security`: TLS configuration, JWT authentication, basic RBAC |
| 10 | `nexuscore-api`: REST controllers, DTOs, validation, error handling |
| 11 | `nexuscore-platform`: Spring Boot assembly, configuration, health endpoints |
| 12 | Integration tests, Docker image, deployment documentation |

**Exit criteria:** Can register a PostgreSQL target, execute parameterized queries, run OS commands, all over TLS with JWT auth and audit logging.

### Phase 2: Production Hardening (6-8 weeks)

| Deliverable | Detail |
|---|---|
| MySQL adapter | Full `DatabasePort` implementation |
| SQL Server adapter | Full `DatabasePort` implementation |
| Oracle adapter | Full `DatabasePort` implementation (requires Oracle JDBC driver licensing attention) |
| Circuit breakers | Resilience4j on every adapter |
| Observability | Micrometer metrics, OpenTelemetry tracing, Grafana dashboards |
| Audit logging | Immutable audit trail with query/search API |
| Connection pool lifecycle | Eviction, monitoring, per-target pool limits |
| Security hardening | mTLS for external comms, secret rotation, RBAC refinement |
| Load testing | Gatling or k6 tests against all adapters |

### Phase 3: AI Enhancements (6-8 weeks)

| Deliverable | Detail |
|---|---|
| pgvector integration | Schema, HNSW index, `PgVectorStore` implementation |
| Embedding pipeline | `EmbeddingProvider` SPI + OpenAI and Ollama implementations |
| Semantic search API | REST endpoint for similarity search with metadata filtering |
| Content ingestion | Ingest support tickets, runbooks, resolution logs |
| Basic RAG | Context assembly + LLM call for AI-generated answers |
| Feedback loop | User feedback collection to measure retrieval quality |

### Phase 4: Productization (8-12 weeks)

| Deliverable | Detail |
|---|---|
| Multi-tenancy | Tenant isolation (schema-per-tenant or column-based) |
| Workflow engine | Define and execute multi-step support workflows |
| Adapter marketplace | SDK for third-party adapter development, adapter versioning |
| Admin UI | Web dashboard for adapter management, audit log viewing, health monitoring |
| Billing integration | Usage metering per adapter call, tier enforcement |
| SOC 2 prep | Compliance documentation, penetration testing, access reviews |

---

## 12. RISKS / BAD DECISIONS TO AVOID

### Architecture-Killing Mistakes

| Risk | Severity | Mitigation |
|---|---|---|
| **Premature microservices** | Critical | Start as a modular monolith. Extract services only when you have a clear scaling/team boundary need. Premature decomposition multiplies operational complexity 10x for a small team. |
| **Leaky adapter abstractions** | Critical | If your `DatabasePort` interface has PostgreSQL-specific methods, the abstraction is broken. Use capabilities introspection. Database-specific features go behind `capabilities().supportedFeatures()`. |
| **Connection pool exhaustion** | High | Unbounded pool creation per target will OOM the JVM. Enforce a global pool count limit and evict idle pools aggressively. |
| **SQL injection via adapter layer** | Critical | The adapter layer executes queries against external databases. ALL queries MUST be parameterized. Add a `QuerySanitizer` that rejects any query with string interpolation patterns. |
| **Embedding dimension lock-in** | High | If you hardcode 1536 dimensions (OpenAI ada-002), switching models requires re-embedding everything. Store dimension metadata and support multiple embedding models per source. |
| **Over-abstracting too early** | High | Do not build the workflow engine, billing system, or multi-tenancy in Phase 1. Build the core adapter layer first. You will learn what you actually need from using it. |
| **Testing against mocks only** | High | DB adapter tests MUST run against real databases via Testcontainers. Mock-only tests give false confidence and miss dialect-specific edge cases. |
| **Monolithic configuration** | Medium | One giant `application.yml` becomes unmaintainable. Use Spring profiles + per-adapter config files. |
| **Ignoring Oracle JDBC licensing** | Medium | Oracle's JDBC driver has specific redistribution rules. Use the Maven Central-published `ojdbc11` and document the licensing requirement. |
| **Building a UI too early** | Medium | The API layer is the product. A premature UI constrains the API design. Build the API first, consume it with curl/Postman, then build the UI. |

### Subtle Mistakes That Compound

- **Not versioning the REST API from day one** -- retrofitting `/api/v1/` is painful. Start with it.
- **Not using ArchUnit from day one** -- module boundary violations creep in silently. ArchUnit catches them at build time.
- **Storing credentials in the platform database** -- even encrypted, this is a liability. Use Vault or an external secrets provider from the start.
- **Not auditing "read" operations** -- compliance regimes (SOC 2, HIPAA) require audit trails on reads, not just writes.
- **Ignoring time zones** -- all timestamps MUST be `TIMESTAMPTZ` / `OffsetDateTime`. Mixing naive and aware timestamps is a data corruption vector.

---

## 13. STARTER CODE PLAN

Build in this order. Each step depends on the previous one being complete.

| Order | What | Why First |
|---|---|---|
| 1 | `settings.gradle.kts` + `build.gradle.kts` + version catalog | Everything else depends on the build system |
| 2 | `nexuscore-core`: port interfaces + domain records | The core defines the contracts everyone else implements |
| 3 | `nexuscore-adapter-db-common`: `ConnectionPoolManager`, `CredentialResolver`, `ResultSetMapper` | Shared DB infrastructure before any specific adapter |
| 4 | `nexuscore-adapter-db-postgresql`: `PostgreSQLAdapter` | First adapter proves the port interface design works |
| 5 | `nexuscore-test`: Testcontainers setup + adapter test harness | Validate the PostgreSQL adapter against a real database |
| 6 | `nexuscore-security`: `TlsContextFactory`, `JwtTokenProvider`, `AuditLogWriter` | Security must be in place before exposing an API |
| 7 | `nexuscore-api`: REST controllers for database operations | First externally-consumable surface |
| 8 | `nexuscore-platform`: Spring Boot app, configuration, health | Assembly that ties everything together |
| 9 | `nexuscore-adapter-os`: `CommandExecutor`, `ProcessMonitor` | Second adapter type validates the architecture generalizes |
| 10 | `nexuscore-observability`: Micrometer metrics, structured logging | Operational visibility for the running system |
| 11 | Docker image + `docker-compose.yml` | Deployable artifact |

After this sequence, you have a working platform that can:

- Connect to PostgreSQL targets and execute queries
- Execute sandboxed OS commands
- Serve a REST API over TLS with JWT auth
- Audit every operation
- Report metrics and health

Everything else (MySQL/MSSQL/Oracle adapters, vector layer, workflow engine, multi-tenancy) builds incrementally on top of this foundation.

---

## 14. "THINK LIKE A CTO" SECTION

### Make This a Business Asset, Not Just Code

**1. Open Core Model**

Release the core adapter framework as open source. Keep the enterprise features proprietary:

- Open: core port interfaces, PostgreSQL/MySQL adapters, basic OS adapter, REST API
- Proprietary: Oracle adapter, AI/RAG features, multi-tenancy, workflow engine, admin UI, enterprise SSO

This builds community, attracts contributors who build new adapters, and creates a funnel for enterprise sales. MuleSoft, Elastic, and Confluent all used this model to build billion-dollar businesses.

**2. The Adapter Marketplace is the Moat**

Every adapter someone builds for NexusCore increases the platform's value for everyone. This is a classic network effect. Invest early in:

- An Adapter SDK with clear documentation
- A certification process for third-party adapters
- A distribution mechanism (Maven repository or marketplace portal)

**3. AI Features Are the Margin Play**

The adapter layer is infrastructure (necessary but commoditizable). The AI/RAG layer is where you charge a premium:

- Embedding generation costs money (OpenAI API calls)
- Semantic search over organizational knowledge is high-value
- RAG-generated answers save hours of investigation time
- Fine-tuned models on customer data are defensible and sticky

Price the AI tier at 3-5x the base platform cost.

**4. Compliance Sells Itself**

SOC 2 Type II compliance for NexusCore means every customer's auditor stops asking questions about how their teams access databases and run OS commands. The audit trail alone justifies the platform cost for regulated industries (finance, healthcare, government).

Get SOC 2 early. It's a sales accelerator, not just a checkbox.

**5. Developer Experience is Distribution**

If a developer can `docker run nexuscore` and have a working instance with PostgreSQL adapter in 60 seconds, they will try it. If it takes 30 minutes of configuration, they won't.

Invest in:

- A `docker-compose.yml` that starts everything
- First-run wizards that auto-detect local databases
- Excellent API documentation (OpenAPI spec, Swagger UI)
- A CLI tool for common operations

**6. Think About What You're Really Selling**

You're not selling a database connector or an OS command runner. You're selling **operational confidence**: the assurance that every database query, every OS command, every external communication is authenticated, authorized, audited, and observable. That's a story CFOs and CISOs buy, not just engineers.

**7. Revenue Projection Framework**

```
Year 1: Internal use only → saves 0.5-1 FTE equivalent → $75K-150K cost avoidance
Year 2: First 3-5 enterprise customers → $50K-100K/year each → $150K-500K ARR
Year 3: Adapter marketplace + AI tier → 20-50 customers → $1M-5M ARR
Year 4: Managed SaaS launch → volume play → $5M-15M ARR trajectory
```

These are conservative numbers based on comparable infrastructure tooling sales. The key is to get to $500K ARR on the enterprise license model before investing in SaaS infrastructure.

---

## Summary of Key Decisions

| Decision | Choice | Key Tradeoff |
|---|---|---|
| Architecture | Hexagonal modular monolith | Simplicity now vs. extractability later |
| Java version | 21 LTS | Virtual threads justify the version choice alone |
| Framework | Spring Boot 3.3+ | Ecosystem maturity over startup speed |
| Build | Gradle multi-module | Complexity of Kotlin DSL vs. Maven's XML hell |
| Platform DB | PostgreSQL + pgvector | Single DB for both relational and vector needs |
| Adapter DB access | JdbcTemplate | SQL control across dialects vs. JPA convenience |
| Secrets | Vault via SPI | Operational dependency vs. credential security |
| Auth | JWT + mTLS | Token management complexity vs. zero-trust posture |
| Testing | Testcontainers | Slower CI vs. real integration confidence |
| Deployment | Docker → K8s | Start with Compose, graduate to K8s when needed |
| Business model | Open core | Community building vs. revenue protection |

---

## 15. LLM INTEGRATION ARCHITECTURE

NexusCore interfaces with LLMs in three fundamentally different patterns, each serving a different purpose and carrying different cost/risk profiles.

### The Three LLM Integration Patterns

```
┌──────────────────────────────────────────────────────────────────────┐
│                     LLM INTEGRATION PATTERNS                         │
│                                                                      │
│  PATTERN 1            PATTERN 2              PATTERN 3               │
│  Embedding            RAG Completion         Agentic Tool Use        │
│  (one-way)            (retrieve → generate)  (LLM drives NexusCore)  │
│                                                                      │
│  Content → LLM API    Query → Vector Search  User question           │
│         → float[]          → Context chunks       → LLM decides      │
│         → pgvector         → LLM + context             what to do    │
│                            → Grounded answer       → Calls adapters  │
│                                                    → Synthesizes     │
│                                                      diagnosis       │
│                                                                      │
│  Cost: $0.0001/embed  Cost: $0.01-0.05/call  Cost: $0.05-0.50/call  │
│  Latency: 100-500ms   Latency: 1-5s          Latency: 5-30s         │
│  Risk: Low            Risk: Medium            Risk: High (needs      │
│                       (hallucination)          guardrails)            │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 1: Embedding Generation (Data Ingestion Path)

The simplest pattern. NexusCore sends text to an LLM embedding API and stores the resulting vector in pgvector. No generation, no conversation, no risk of hallucination.

```
Support ticket resolved
    → ContentPreprocessor.chunk(resolution_notes)
    → EmbeddingProvider.embedBatch(chunks)     ← calls OpenAI /embeddings or local Ollama
    → PgVectorStore.storeBatch(embeddings)
    → Done. Knowledge is now searchable.
```

This runs asynchronously. When a knowledge artifact arrives (ticket closed, runbook updated, resolution logged), a background job embeds it. No user is waiting on this.

### Pattern 2: RAG Completion (Knowledge-Grounded Answers)

The core AI use case. A user asks a question, NexusCore retrieves relevant context from pgvector, and an LLM generates an answer grounded in organizational knowledge.

```
User: "How do we handle connection pool exhaustion on the Oracle adapter?"

    ① Embed the question
       → EmbeddingProvider.embed(question) → float[1536]

    ② Retrieve similar content from pgvector
       → PgVectorStore.searchSimilar(queryVector, topK=10,
             filter={source_type: ["runbook", "resolution", "kb_article"]})
       → Returns 10 chunks with similarity scores

    ③ Rerank (optional, improves precision)
       → Cross-encoder reranker scores each chunk against the original question
       → Keep top 5

    ④ Assemble the prompt
       → ContextAssembler builds:
         "You are a support engineer for NexusCore. Answer the question
          using ONLY the following context. Cite sources.

          Context:
          [Source: Runbook RB-042, score 0.94] When Oracle connection pools exhaust...
          [Source: Incident INC-1847, score 0.91] On March 3, we resolved pool exhaustion by...
          [Source: KB Article KB-112, score 0.87] Oracle adapter default pool size is 5...

          Question: How do we handle connection pool exhaustion on the Oracle adapter?"

    ⑤ Call the LLM
       → LlmPort.complete(assembledPrompt)
       → LLM generates a grounded answer with source citations

    ⑥ Return to user with citations
       → "Based on Runbook RB-042 and the resolution from INC-1847: Increase
          maxPoolSize to 10 in the adapter config, and check for leaked connections
          using the diagnostic query in KB-112..."
```

### Pattern 3: Agentic Tool Use (The Killer Feature)

NexusCore's adapter architecture creates a unique advantage: the LLM doesn't just answer questions from stored knowledge -- it **drives NexusCore's adapters as tools** to investigate live infrastructure in real time.

```
User: "Why is the production PostgreSQL slow right now?"

    ① NexusCore sends the question to the LLM WITH tool definitions:

       Available tools:
       ├── query_database(target, sql, params)     ← DatabasePort
       ├── introspect_schema(target, schema)        ← DatabasePort
       ├── execute_os_command(host, command)         ← OsPlatformPort
       ├── check_service_health(host, service)       ← OsPlatformPort
       ├── search_knowledge(query, topK)             ← VectorPort
       └── get_system_metrics(host)                  ← OsPlatformPort

    ② LLM decides on a diagnostic plan (multi-step reasoning):

       Step 1: query_database("prod-pg-1",
                 "SELECT count(*), state FROM pg_stat_activity GROUP BY state")
       → Result: {active: 47, idle: 3, idle_in_transaction: 12}

       Step 2: query_database("prod-pg-1",
                 "SELECT pid, query, now()-query_start AS duration
                  FROM pg_stat_activity WHERE state='active'
                  ORDER BY duration DESC LIMIT 5")
       → Result: [3 queries running >30 seconds, all hitting orders table]

       Step 3: get_system_metrics("prod-host-1")
       → Result: {cpu: 78%, disk_io_util: 94%, memory: 62%}

       Step 4: search_knowledge("PostgreSQL slow query orders table high disk IO")
       → Result: [Past incident INC-2103: resolved by adding index on orders.created_at]

    ③ LLM synthesizes a diagnosis:

       "Production PostgreSQL has 47 active connections with 12 idle-in-transaction
        (possible connection leak). Three queries on the orders table are running >30s,
        driving disk I/O to 94%. This matches incident INC-2103 from February, which
        was resolved by adding an index on orders.created_at. Recommended actions:
        1. Kill the 12 idle-in-transaction sessions
        2. Add the missing index
        3. Investigate the connection leak in the application pool config"
```

### LLM Port Interface

```java
public interface LlmPort {

    CompletionResult complete(CompletionRequest request);

    CompletionResult completeWithTools(CompletionRequest request, List<ToolDefinition> tools);

    StreamedCompletionResult streamComplete(CompletionRequest request);

    LlmCapabilities capabilities();
}

public record CompletionRequest(
    String systemPrompt,
    List<Message> messages,         // conversation history
    CompletionParameters params,    // temperature, maxTokens, topP
    String modelOverride            // null = use default model
) {}

public record CompletionResult(
    String content,
    TokenUsage usage,               // prompt tokens, completion tokens, total cost
    String modelUsed,
    String finishReason
) {}

public record LlmCapabilities(
    String provider,                // "openai", "anthropic", "ollama"
    List<String> availableModels,
    boolean supportsStreaming,
    boolean supportsFunctionCalling,
    int maxContextWindow,
    boolean isLocal                 // true for Ollama -- data never leaves the network
) {}
```

### LLM Provider Architecture (Provider-Agnostic)

Same adapter pattern as everything else:

```
LlmPort (interface in nexuscore-core)
    │
    ├── OpenAiLlmAdapter          → GPT-4o, GPT-4-turbo (best function calling)
    ├── AnthropicLlmAdapter       → Claude (strong reasoning, large context)
    ├── AzureOpenAiLlmAdapter     → Azure-hosted OpenAI (enterprise compliance)
    ├── OllamaLlmAdapter          → Local models (data never leaves your network)
    └── BedrockLlmAdapter         → AWS Bedrock (multi-model, enterprise billing)
```

**Technology choice: Spring AI** -- the emerging standard for LLM integration in the Spring ecosystem. Provides chat model abstraction, function calling support, direct pgvector integration, RAG pipeline helpers, streaming support, and Micrometer observability for token usage and cost tracking.

### Agentic Loop: LLM Agent Controller

```java
public class LlmAgentController {

    private final LlmPort llmPort;
    private final ToolRegistry toolRegistry;
    private final RbacEnforcer rbacEnforcer;
    private final AuditSink auditSink;

    public AgentResponse executeAgentLoop(
            String userQuestion,
            String principalId,
            int maxIterations) {

        List<Message> conversation = new ArrayList<>();
        conversation.add(Message.system(AGENT_SYSTEM_PROMPT));
        conversation.add(Message.user(userQuestion));

        for (int i = 0; i < maxIterations; i++) {

            CompletionResult result = llmPort.completeWithTools(
                new CompletionRequest(conversation),
                toolRegistry.getToolDefinitions());

            if (result.hasToolCalls()) {
                for (ToolCall call : result.toolCalls()) {
                    rbacEnforcer.enforce(principalId, call.toolName(), call.arguments());
                    ToolResult toolResult = toolRegistry.execute(call);
                    auditSink.record(AuditEvent.toolCall(principalId, call, toolResult));
                    conversation.add(Message.toolResult(call.id(), toolResult));
                }
            } else {
                return new AgentResponse(result.content(), conversation);
            }
        }

        return AgentResponse.maxIterationsReached(conversation);
    }
}
```

### Tool Registry: Adapters Become LLM Tools

```java
public class NexusCoreToolRegistry implements ToolRegistry {

    private final AdapterRouter adapterRouter;
    private final OsPlatformPort osPort;
    private final VectorPort vectorPort;

    @Override
    public List<ToolDefinition> getToolDefinitions() {
        return List.of(
            ToolDefinition.builder()
                .name("query_database")
                .description("Execute a read-only SQL query against a registered target database")
                .parameter("target_id", "string", "The registered target database ID")
                .parameter("sql", "string", "Parameterized SQL query")
                .build(),

            ToolDefinition.builder()
                .name("get_system_metrics")
                .description("Get CPU, memory, disk, and network metrics for a managed host")
                .parameter("host_id", "string", "The managed host identifier")
                .build(),

            ToolDefinition.builder()
                .name("search_knowledge")
                .description("Semantic search over past incidents, runbooks, and KB articles")
                .parameter("query", "string", "Natural language search query")
                .parameter("top_k", "integer", "Number of results to return (default 5)")
                .build()
        );
    }

    @Override
    public ToolResult execute(ToolCall call) {
        return switch (call.toolName()) {
            case "query_database" -> {
                var target = resolveTarget(call.argument("target_id"));
                var query = ParameterizedQuery.readOnly(call.argument("sql"));
                yield ToolResult.from(adapterRouter.route(target, query));
            }
            case "get_system_metrics" -> {
                yield ToolResult.from(osPort.collectMetrics());
            }
            case "search_knowledge" -> {
                var embedding = embeddingProvider.embed(call.argument("query"));
                var topK = call.argumentOrDefault("top_k", 5);
                yield ToolResult.from(vectorPort.searchSimilar(embedding, topK, SearchFilter.none()));
            }
            default -> throw new UnknownToolException(call.toolName());
        };
    }
}
```

### Security Guardrails (Non-Negotiable for Agentic Use)

| Guardrail | Implementation |
|---|---|
| **RBAC on every tool call** | The LLM agent inherits the calling user's permissions. If the user can't run OS commands, the LLM can't either. |
| **Read-only by default** | Agent tool calls default to read-only operations. Write/mutate requires explicit escalation and approval. |
| **Query sanitization** | LLM-generated SQL passes through `QuerySanitizer` before execution. No DDL, no `DROP`, no `TRUNCATE` unless explicitly allowed. |
| **Token budget enforcement** | Each agent invocation has a max token budget. Prevents runaway loops that burn through API credits. |
| **Max iteration cap** | The agent loop has a hard cap (e.g., 10 tool calls). Prevents infinite reasoning loops. |
| **Human-in-the-loop for mutations** | If the LLM wants to execute a write operation, NexusCore pauses and asks for human approval before proceeding. |
| **Audit trail on every tool call** | Every tool invocation the LLM makes is logged: what tool, what arguments, what result, how long, how many tokens consumed. |
| **Data sanitization on output** | LLM responses are passed through the same confidential data masking as all other outputs. |

### Cost Control Architecture

```java
public record TokenBudget(
    int maxPromptTokens,
    int maxCompletionTokens,
    int maxToolCallIterations,
    BigDecimal maxCostPerRequest     // hard dollar cap
) {}

public record UsageAccumulator(
    AtomicInteger totalPromptTokens,
    AtomicInteger totalCompletionTokens,
    AtomicReference<BigDecimal> totalCost
) {
    public void enforce(TokenBudget budget) {
        if (totalCost.get().compareTo(budget.maxCostPerRequest()) > 0) {
            throw new BudgetExceededException(
                "Request exceeded cost budget: $%s > $%s"
                    .formatted(totalCost.get(), budget.maxCostPerRequest()));
        }
    }
}
```

Per-tenant, per-user, and per-request budgets. Metered and reportable for billing.

### LLM Module Structure

```
nexuscore-llm/
    └── src/main/java/com/nexuscore/llm/
        ├── port/
        │   └── LlmPort.java              ← Core interface
        ├── provider/
        │   ├── OpenAiLlmAdapter.java
        │   ├── AnthropicLlmAdapter.java
        │   ├── AzureOpenAiLlmAdapter.java
        │   └── OllamaLlmAdapter.java
        ├── agent/
        │   ├── LlmAgentController.java    ← The agentic loop
        │   ├── ToolRegistry.java          ← Maps adapters to LLM tools
        │   └── NexusCoreToolRegistry.java
        ├── cost/
        │   ├── TokenBudget.java
        │   └── UsageAccumulator.java
        ├── prompt/
        │   ├── PromptTemplateRegistry.java
        │   └── PromptVersionManager.java
        └── LlmAutoConfiguration.java
```

### Strategic Advantage

Every other AI-powered operations tool (PagerDuty Copilot, Datadog AI, etc.) has to build integrations to databases, OS tools, and external systems from scratch. NexusCore already has all of those as adapters. The LLM layer sits on top and **reuses the entire adapter surface as its tool palette**.

Every new adapter added to NexusCore (CockroachDB, Redis, Kafka) automatically becomes a new tool the AI agent can use. The platform gets smarter with every integration, without writing any new AI code.

This is the clearest premium pricing boundary: the adapter layer is the free/base tier, the AI agent layer is the enterprise tier at 3-5x the price.
