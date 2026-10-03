# Ledger Platform Roadmap

This repository tracks the architecture, delivery plan, and technical decisions for the Ledger Platform.

The platform is being built as three independent applications that work together through explicit API contracts:

- `ledger-ios` — iOS client
- `ledger-bff` — Backend for Frontend for mobile clients
- `ledger-core` — Core financial backend

The goal is to build the platform through small vertical flows while keeping payment consistency, failure handling, and observability explicit.

## Platform Architecture

```text
Ledger iOS
    |
    v
Ledger BFF
    |
    v
LedgerCore
    |
    +-------------------+
    |                   |
    v                   v
PostgreSQL        Payment Provider
                  (simulated)
```

### Ledger iOS

The iOS application owns the user experience and client-side payment intent.

Main responsibilities:

- payment creation UI
- payment intent identity
- unknown-result recovery
- payment status lookup
- account balance and statement screens
- authentication state
- client-side correlation metadata

### Ledger BFF

The BFF adapts the platform APIs to the mobile client.

Main responsibilities:

- mobile-specific API contracts
- authentication and session handling
- response composition
- error mapping
- API version compatibility
- correlation ID propagation

Financial rules do not belong in the BFF.

### LedgerCore

LedgerCore owns financial rules and consistency.

Main responsibilities:

- payment lifecycle
- payment persistence
- idempotency
- transaction boundaries
- ledger operations
- account balance
- account statement
- external provider integration
- reconciliation
- observability

## Delivery Strategy

The platform is developed through vertical flows instead of building each application in isolation.

Example:

```text
iOS
 |
 | POST /payments
 | Idempotency-Key
 v
BFF
 |
 v
Core
 |
 v
PostgreSQL
```

A flow is expanded only after its basic contract works end to end. Reliability concerns such as retries, race conditions, reconciliation, and tracing are then added to the same flow.

## Roadmap

### Phase 1 — Core Foundation

- [x] Java 21
- [x] Spring Boot
- [x] Gradle
- [x] Docker
- [x] PostgreSQL
- [x] Flyway

### Phase 2 — Payment Domain

- [x] Payment lifecycle
- [x] Domain invariants
- [x] Payment factory method
- [x] Payment state transitions
- [x] Unit tests

### Phase 3 — Payment Persistence

- [ ] Create payments table
- [ ] Add Flyway migration
- [ ] Create JPA entity
- [ ] Add persistence mapper
- [ ] Define repository port
- [ ] Implement repository adapter
- [ ] Add integration tests

### Phase 4 — Create Payment Flow

- [ ] Implement Create Payment use case
- [ ] Define transaction boundary
- [ ] Add request validation
- [ ] Implement `POST /payments`
- [ ] Define response contract

### Phase 5 — Idempotency

- [ ] Add idempotency key support
- [ ] Compare the complete request fingerprint
- [ ] Add database uniqueness guarantees
- [ ] Add reservation ownership
- [ ] Add reservation expiration
- [ ] Store successful and declined results
- [ ] Define retention policy
- [ ] Handle safe retries
- [ ] Add concurrent request tests
- [ ] Add race condition tests

### Phase 6 — Payment Attempt Recovery

- [ ] Implement `GET /payment-attempts/{key}`
- [ ] Return processing attempts
- [ ] Return successful attempts
- [ ] Return declined attempts
- [ ] Distinguish unknown and expired keys
- [ ] Support replay metadata

### Phase 7 — BFF Payment Integration

- [ ] Create BFF project
- [ ] Define mobile payment contract
- [ ] Forward idempotency key
- [ ] Integrate payment creation with LedgerCore
- [ ] Integrate payment attempt lookup
- [ ] Add error mapping
- [ ] Propagate correlation ID

### Phase 8 — iOS Payment Integration

- [ ] Create Ledger iOS project structure
- [ ] Integrate the payment reliability lab
- [ ] Preserve `PaymentIntent` across unknown results
- [ ] Implement create payment flow
- [ ] Implement payment recovery by status lookup
- [ ] Handle succeeded, declined, processing, not found, and expired states
- [ ] Add payment receipt UI
- [ ] Add contract and UI tests

### Phase 9 — Ledger

- [ ] Create ledger entries
- [ ] Implement append-only model
- [ ] Define payment and ledger transaction boundary
- [ ] Calculate account balance
- [ ] Build account statement

### Phase 10 — External Provider

- [ ] Add provider simulation
- [ ] Implement payment authorization
- [ ] Handle provider timeouts
- [ ] Handle provider errors
- [ ] Handle lost responses
- [ ] Add retry strategy

### Phase 11 — Reconciliation

- [ ] Add intermediate payment states
- [ ] Add provider status lookup
- [ ] Implement reconciliation job
- [ ] Handle internal and provider status mismatches

### Phase 12 — Observability

- [ ] Add correlation ID
- [ ] Add structured logs
- [ ] Add application metrics
- [ ] Add distributed tracing
- [ ] Add health checks

### Phase 13 — Auth and Session

- [ ] Add authentication
- [ ] Add authorization
- [ ] Define session strategy
- [ ] Integrate authentication across iOS, BFF, and Core

### Phase 14 — Reliability and Delivery

- [ ] Add Testcontainers
- [ ] Add CI pipelines
- [ ] Build Docker images
- [ ] Add OpenAPI documentation
- [ ] Add Architecture Decision Records
- [ ] Document failure scenarios
- [ ] Add operational documentation

## Payment Reliability Contract

The first vertical flow is based on an explicit payment contract.

The client creates a payment intent with an idempotency key:

```http
POST /payments
Idempotency-Key: <key>
```

If the client receives an unknown result, it does not assume that the payment failed. It checks the same intent before deciding whether another command is safe:

```http
GET /payment-attempts/{key}
```

The expected recovery states are:

| State | Client behavior |
| --- | --- |
| succeeded | Show the existing payment result |
| declined | Finish the intent as declined |
| processing | Keep the same intent and check again later |
| not found | Retry only when the contract guarantees that no financial effect exists |
| expired | Do not retry automatically |

This contract will be shared across the iOS lab, BFF, and LedgerCore implementations.

## Testing Strategy

The platform uses different test levels for different guarantees.

### Unit Tests

Used for isolated domain rules and state transitions.

### Integration Tests

Used for:

- PostgreSQL behavior
- Flyway migrations
- JPA mappings
- repository adapters
- transaction boundaries

### Contract Tests

The payment contract suite should run against both the simulated backend and the real LedgerCore implementation.

### Concurrency Tests

Critical payment flows will cover:

- duplicate idempotency keys
- simultaneous requests
- reservation ownership
- conflicting updates
- single financial effect guarantees

### Failure Tests

The platform will simulate failures such as:

- response lost after commit
- failure before debit
- server error after debit
- provider timeout
- delayed confirmation

## GitHub Project

The GitHub Project for the platform should use this repository as the architecture and roadmap reference.

Suggested fields:

### Status

- Backlog
- Ready
- In Progress
- In Review
- Done

### Component

- Core
- BFF
- iOS
- Infrastructure

### Area

- Payments
- Idempotency
- Ledger
- Reconciliation
- Auth
- Observability

### Priority

- P0
- P1
- P2
- P3

Example issue titles:

```text
[Core] Add payments database migration
[Core] Implement payment repository adapter
[Core] Add idempotency reservation

[BFF] Define mobile payment contract
[BFF] Propagate idempotency key

[iOS] Preserve PaymentIntent after unknown result
[iOS] Implement payment recovery flow
```

## Architecture Decision Records

Important technical decisions will be documented as ADRs.

Planned structure:

```text
docs/
└── adr/
    ├── 0001-use-postgresql.md
    ├── 0002-manage-schema-with-flyway.md
    ├── 0003-separate-domain-from-persistence.md
    └── 0004-idempotency-strategy.md
```

ADRs will be added when a decision changes architecture, consistency guarantees, data modeling, or operational behavior.
