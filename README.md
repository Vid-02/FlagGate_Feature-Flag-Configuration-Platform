# FlagGate

FlagGate is a backend **Feature Flag and Configuration Platform** designed to
support low-latency runtime flag evaluation and reliable configuration
propagation across distributed services.

The system models how modern backend platforms manage feature rollouts,
kill switches, and dynamic configuration updates while preserving availability,
determinism, and predictable performance on latency-sensitive request paths.

---

## Problem Overview

Large-scale backend systems require mechanisms to safely control behavior at
runtime without redeploying services. Common requirements include:

- Enabling or disabling features dynamically
- Gradually rolling out changes to subsets of traffic
- Instantly rolling back faulty or risky functionality
- Maintaining consistent configuration state across multiple services

A feature flag platform must therefore be:

- **Fast** — evaluated synchronously on the request path
- **Highly available** — resilient to partial system failures
- **Deterministic** — stable results for the same inputs
- **Operationally safe** — reliable propagation and convergence of changes

FlagGate addresses these requirements through a deliberately scoped but
architecturally robust backend design.

---

## System Architecture

FlagGate is structured around a clear separation between **configuration
management workflows** and **runtime flag evaluation**, a pattern commonly used
in large-scale distributed systems to isolate latency-sensitive paths.

### Management Service (Write Path)

- Creates and updates feature flag definitions
- Persists flag state in Apache Cassandra as the durable source of truth
- Publishes configuration change events to Apache Kafka

### Evaluation Service (Read Path)

- Evaluates feature flags for incoming requests
- Uses an in-memory cache to ensure low-latency access
- Falls back to Cassandra on cache misses
- Subscribes to Kafka events to keep cached state synchronized

This separation ensures that runtime evaluation remains fast, predictable,
and isolated from configuration update workflows.

---

## Core Concepts

### Feature Flags

Each feature flag is defined by:

- A unique key
- An enabled or disabled state
- Optional targeting rules
- Percentage-based rollout configuration
- A monotonically increasing version

Versioning allows services to apply updates idempotently and safely handle
out-of-order or replayed events.

---

### Deterministic Rollouts

Percentage rollouts are computed using deterministic hashing so that:

- The same entity consistently receives the same flag result
- Rollouts remain stable across requests and service restarts
- Gradual exposure can be performed without nondeterministic behavior

---

### Event-Driven Propagation

Configuration changes are propagated using Kafka to ensure that:

- Services converge on updated configuration without polling
- Updates are replayable and resilient to temporary consumer failures
- Local state remains consistent through version-based reconciliation

---

## Technology Stack

| Technology | Purpose |
|-----------|---------|
| Java | Core backend implementation |
| Spring Boot | REST services and application framework |
| Apache Cassandra | Highly available configuration storage |
| Apache Kafka | Event-driven configuration propagation |
| In-memory Cache | Low-latency runtime evaluation |
| Docker Compose | Local development and integration testing |
| Gradle (Multi-module) | Build and dependency management |

---

## Repo Structure

.
├── docs/
│   └── ARCHITECTURE.md            # Detailed system design, data flows, and tradeoffs
│
├── infra/
│   ├── docker-compose.yml         # Local Cassandra + Kafka infrastructure
│   └── cassandra/
│       └── schema.cql             # Cassandra schema for flag definitions
│
├── services/
│   ├── management-service/        # Control-plane service for flag creation and updates
│   │   ├── src/
│   │   │   └── main/
│   │   │       ├── java/com/flaggate/management/
│   │   │       │   ├── controller/        # REST APIs for managing flags
│   │   │       │   ├── service/           # Business logic for flag updates
│   │   │       │   ├── repository/        # Cassandra persistence layer
│   │   │       │   ├── messaging/         # Kafka publisher for change events
│   │   │       │   └── model/              # Flag domain models
│   │   │       └── resources/
│   │   │           └── application.yml
│   │   └── build.gradle
│   │
│   └── evaluation-service/        # Data-plane service for runtime flag evaluation
│       ├── src/
│       │   └── main/
│       │       ├── java/com/flaggate/evaluation/
│       │       │   ├── controller/        # Flag evaluation API
│       │       │   ├── service/           # Evaluation orchestration
│       │       │   ├── cache/             # In-memory flag cache
│       │       │   ├── rollout/           # Deterministic rollout logic
│       │       │   ├── repository/        # Cassandra read access
│       │       │   ├── messaging/         # Kafka consumer for updates
│       │       │   └── model/              # Evaluation request/response models
│       │       └── resources/
│       │           └── application.yml
│       └── build.gradle
│
├── scripts/
│   └── demo.sh                     # End-to-end demo script (flag update → evaluation)
│
├── tools/
│   └── loadtest/
│       └── k6/
│           └── evaluate_batch.js   # Load testing for evaluation latency
│
├── gradle/
│   └── wrapper/                    # Gradle wrapper binaries
│
├── build.gradle                    # Root Gradle configuration
├── settings.gradle                 # Multi-module project definition
├── gradlew                         # Gradle wrapper (Unix)
├── gradlew.bat                     # Gradle wrapper (Windows)
└── README.md

---

## Data Flow Summary

### Flag Update Flow

1. A feature flag is created or updated via the Management Service.
2. The updated definition is written to Cassandra.
3. A configuration change event is published to Kafka.
4. Subscribed services consume the event and update local state.

---

### Flag Evaluation Flow

1. A request arrives at the Evaluation Service.
2. The flag definition is resolved from the local in-memory cache.
3. Cassandra is queried only on cache misses.
4. Rollout and targeting rules are evaluated deterministically.
5. The final flag value is returned to the caller.

---

## Consistency and Reliability Model

- Cassandra serves as the durable source of truth for configuration state.
- Kafka provides reliable, replayable propagation of configuration changes.
- Consumers apply updates idempotently using version checks.
- Cached state converges toward the latest configuration over time.

This model favors availability and predictable latency on the read path while
maintaining correctness through versioning and event replay.

---

## Performance Considerations

- Runtime flag evaluation is optimized for latency-sensitive request paths.
- In-memory caching minimizes repeated database access.
- Batched evaluation reduces per-request overhead.
- Cassandra access patterns are restricted to predictable point reads.

Latency and cache efficiency can be validated using the load-testing utilities
included in the repository.

---

## Documentation

- **ARCHITECTURE.md** — detailed design decisions, data flows, and tradeoffs
- Inline code documentation within each service

---

## Design Philosophy

FlagGate emphasizes clear architectural boundaries, deterministic behavior,
and explicit engineering tradeoffs. The system is designed around strict
separation between control-plane configuration workflows and data-plane
runtime evaluation to preserve performance and operational safety.

Concerns such as authentication, administrative interfaces, and tenant
isolation are architecturally decoupled from the core evaluation and
propagation paths. This design ensures that such concerns can be layered
independently without compromising evaluation latency, consistency
guarantees, or runtime reliability.

By focusing on correctness, availability, and explainability at the core,
FlagGate reflects how large-scale backend platforms evolve safely over time.

---

## Author

Developed by **Vidhi Babariya**  
As a focused exploration of backend systems, event-driven architectures,
and scalable service design.

