# NexusCore Execution Plan (Granular, Test-First)

## Delivery Model

- Work in independent slices with strict scope.
- For each slice: define contracts -> write unit tests -> implement -> run unit tests -> add integration tests -> run full verification.
- Keep each slice releasable and observable.

## Independent Task Streams

1. Build system and module boundaries
2. Core domain contracts (ports, value objects, errors)
3. Router/orchestration core logic
4. Security foundation (authn/authz/audit interfaces)
5. Database adapter common utilities
6. PostgreSQL adapter
7. OS adapter
8. External secure communication adapter
9. Vector and embedding layer
10. API surface and request/response contracts
11. Observability and health
12. Integration and end-to-end test harness

## Immediate Iteration Plan (Now)

### Iteration 1

- Create Gradle multi-module scaffold
- Implement `nexuscore-core` contracts and adapter routing service
- Add unit tests for routing behavior and capability validation
- Run unit test suite and fix failures

### Iteration 2

- Add `nexuscore-test` and Testcontainers baseline
- Add PostgreSQL adapter integration tests
- Run integration tests

### Iteration 3

- Add OS adapter and tests
- Add external communication adapter skeleton and tests

## Quality Gates Per Iteration

- Unit tests pass
- Integration tests for touched modules pass
- Build compiles cleanly
- Lints for changed files are clean
- No cross-module dependency violations

## Branch/Release Discipline

- Keep changes small and reviewable
- One concern per commit
- No feature merge without passing tests
