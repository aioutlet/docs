# AIOutlet Platform Architecture

> **Document Version**: 1.0  
> **Last Updated**: November 3, 2025  
> **Status**: Active

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Service Catalog](#service-catalog)
- [Technology Stack](#technology-stack)
- [Communication Patterns](#communication-patterns)
- [Data Architecture](#data-architecture)
- [Infrastructure Components](#infrastructure-components)
- [Security Architecture](#security-architecture)
- [Deployment Architecture](#deployment-architecture)

---

## Overview

AIOutlet is a cloud-native, microservices-based e-commerce platform built for scalability, resilience, and maintainability. The platform follows Domain-Driven Design (DDD) principles with event-driven architecture patterns.

### Architecture Principles

1. **Microservices Architecture**: Each service owns its domain and data
2. **Event-Driven**: Asynchronous communication via message broker (Dapr Pub/Sub)
3. **API-First**: RESTful APIs with standardized contracts
4. **Polyglot**: Choose the best technology for each service
5. **Cloud-Native**: Containerized, orchestrated with Kubernetes
6. **Observability**: Distributed tracing, structured logging, metrics
7. **Security-First**: JWT authentication, RBAC, secrets management

---

## System Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Container Orchestration (Kubernetes)                   │
└─────────────────────────────────────────────────────────────────────────────────┘

 ┌──────────────┐
 │ Client Apps  │                    ┌──────────────────────────────────────────────────────────┐
 │              │                    │                  Cloud Services                          │
 │ ┌──────────┐ │   ┌──────────┐    │                                                          │
 │ │ Mobile   │─┼──→│Mobile BFF│────┼→ ┌────────────────────┐                                  │
 │ │   App    │ │   │(Node.js) │    │  │   Auth Service     │                                  │
 │ └──────────┘ │   └──────────┘    │  │  ┌────────────┐   │                                  │
 │              │                    │  │  │  Auth API  │   │                                  │
 │ ┌──────────┐ │   ┌──────────┐    │  │  └─────┬──────┘   │                                  │
 │ │ Browser  │─┼──→│ Web App  │────┼→ │     MongoDB        │                                  │
 │ │          │ │   │ (React)  │    │  └────────────────────┘                                  │
 │ └──────────┘ │   └─────┬────┘    │                                                          │
 │              │         │          │  ┌────────────────────┐                                  │
 │ ┌──────────┐ │         │          │  │   User Service     │                                  │
 │ │ Admin    │─┼─────────┘          │  │  ┌────────────┐   │                                  │
 │ │   App    │ │   ┌──────────┐    │  │  │  User API  │   │                                  │
 │ └──────────┘ │   │ Web BFF  │────┼→ │  └─────┬──────┘   │                                  │
 └──────────────┘   │(Node.js) │    │  │     MongoDB        │                                  │
                    └──────────┘    │  └────────────────────┘                                  │
                                    │                                                          │
                                    │  ┌────────────────────┐                                  │
                                    │  │ Product Service    │                                  │
                                    │  │  ┌────────────┐   │                                  │
                                    │  │  │Product API │   │                                  │
                                    │  │  │  (Python)  │   │                                  │
                                    │  │  └─────┬──────┘   │                                  │
                                    │  │     MongoDB        │                                  │
                                    │  └──────────┬─────────┘                                  │
                                    │             │                                            │
                                    │  ┌──────────┴─────────┐         ┌────────────┐          │
                                    │  │   Cart Service     │         │ Event Bus  │          │
                                    │  │  ┌────────────┐   │◄────────│ (RabbitMQ) │          │
                                    │  │  │  Cart API  │   │         │   Dapr     │          │
                                    │  │  │    (Go)    │   │         │  Pub/Sub   │          │
                                    │  │  └────────────┘   │         └─────▲──────┘          │
                                    │  │       Redis        │               │                  │
                                    │  └──────────┬─────────┘               │                  │
                                    │             │                         │                  │
                                    │  ┌──────────┴─────────┐               │                  │
                                    │  │   Order Service    │               │                  │
                                    │  │  ┌────────────┐   │               │                  │
                                    │  │  │ Order API  │   │───────────────┘                  │
                                    │  │  │   (.NET)   │   │                                  │
                                    │  │  └────────────┘   │                                  │
                                    │  │    PostgreSQL      │                                  │
                                    │  └──────────┬─────────┘                                  │
                                    │             │                                            │
                                    │  ┌──────────┴─────────┐                                  │
                                    │  │  Payment Service   │                                  │
                                    │  │  ┌────────────────┐│                                  │
                                    │  │  │  Payment API   ││                                  │
                                    │  │  │    (.NET)      ││                                  │
                                    │  │  └────────────────┘│                                  │
                                    │  │    SQL Server      │                                  │
                                    │  └──────────┬─────────┘                                  │
                                    │             │                                            │
                                    │  ┌──────────┴─────────┐                                  │
                                    │  │ Inventory Service  │                                  │
                                    │  │  ┌────────────┐   │                                  │
                                    │  │  │Inventory API│   │                                  │
                                    │  │  │  (Python)   │   │                                  │
                                    │  │  └────────────┘   │                                  │
                                    │  │    PostgreSQL      │                                  │
                                    │  └──────────┬─────────┘                                  │
                                    │             │                                            │
                                    │  ┌──────────┴─────────┐                                  │
                                    │  │  Review Service    │                                  │
                                    │  │  ┌────────────┐   │                                  │
                                    │  │  │ Review API │   │                                  │
                                    │  │  │  (Node.js) │   │                                  │
                                    │  │  └────────────┘   │                                  │
                                    │  │     MongoDB        │                                  │
                                    │  └──────────┬─────────┘                                  │
                                    │             │                                            │
                                    │  ┌──────────┴─────────┐                                  │
                                    │  │ Notification       │         ┌──────────────┐         │
                                    │  │   Service          │─────────│Observability │         │
                                    │  │  (Node.js)         │         │  (Metrics,   │         │
                                    │  └────────────────────┘         │   Traces,    │         │
                                    │                                 │   Logs)      │         │
                                    │  ┌────────────────────┐         └──────────────┘         │
                                    │  │  Audit Service     │                                  │
                                    │  │   (Node.js)        │                                  │
                                    │  └────────────────────┘                                  │
                                    │                                                          │
                                    │  ┌────────────────────┐                                  │
                                    │  │  Admin Service     │                                  │
                                    │  │   (Node.js)        │                                  │
                                    │  └────────────────────┘                                  │
                                    │                                                          │
                                    │  ┌────────────────────┐                                  │
                                    │  │ Order Processor    │                                  │
                                    │  │   (Java/Spring)    │                                  │
                                    │  └────────────────────┘                                  │
                                    │                                                          │
                                    │  ┌────────────────────┐                                  │
                                    │  │ Message Broker     │                                  │
                                    │  │   Service (Go)     │                                  │
                                    │  └────────────────────┘                                  │
                                    └──────────────────────────────────────────────────────────┘
                                                 │
                    ┌────────────────────────────┴────────────────────────┐
                    │           Shared Infrastructure                     │
                    │  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐ │
                    │  │   Logging   │ │   Tracing   │ │   Metrics    │ │
                    │  │(ELK Stack)  │ │  (Jaeger)   │ │(Prometheus/  │ │
                    │  │             │ │             │ │  Grafana)    │ │
                    │  └─────────────┘ └─────────────┘ └──────────────┘ │
                    └─────────────────────────────────────────────────────┘
```

---

## Service Catalog

### Customer-Facing Services

| Service               | Technology         | Database   | Port | Purpose                                             |
| --------------------- | ------------------ | ---------- | ---- | --------------------------------------------------- |
| **Auth Service**      | Node.js/TypeScript | MongoDB    | 8000 | JWT authentication, user sessions, token management |
| **User Service**      | Node.js/TypeScript | MongoDB    | 8001 | User profiles, preferences, customer management     |
| **Product Service**   | Python/FastAPI     | MongoDB    | 8003 | Product catalog, variations, badges, search         |
| **Cart Service**      | Go                 | Redis      | 8004 | Shopping cart, cart persistence, cart operations    |
| **Order Service**     | .NET (C#)          | PostgreSQL | 8005 | Order management, order status, order history       |
| **Payment Service**   | .NET (C#)          | SQL Server | 8006 | Payment processing, refunds, payment methods        |
| **Inventory Service** | Python/FastAPI     | PostgreSQL | 8007 | Stock management, reservations, availability        |
| **Review Service**    | Node.js/TypeScript | MongoDB    | 8008 | Product reviews, ratings, review moderation         |

### Internal/Backend Services

| Service                    | Technology         | Database   | Port | Purpose                                   |
| -------------------------- | ------------------ | ---------- | ---- | ----------------------------------------- |
| **Admin Service**          | Node.js/TypeScript | MongoDB    | 8009 | Admin operations, dashboard, analytics    |
| **Audit Service**          | Node.js/TypeScript | MongoDB    | 8010 | Audit logs, event tracking, compliance    |
| **Notification Service**   | Node.js/TypeScript | MongoDB    | 8011 | Email, SMS, push notifications            |
| **Order Processor**        | Java/Spring Boot   | PostgreSQL | 8012 | Order fulfillment, async order processing |
| **Message Broker Service** | Go                 | N/A        | 8013 | Message routing, event orchestration      |

### Gateway/UI Services

| Service         | Technology         | Purpose                                         |
| --------------- | ------------------ | ----------------------------------------------- |
| **Web BFF**     | Node.js/TypeScript | Backend-for-Frontend for web UI                 |
| **API Gateway** | Kong/Nginx         | Request routing, rate limiting, SSL termination |

---

## Technology Stack

### Languages & Frameworks

- **Node.js/TypeScript**: Auth, User, Review, Notification, Admin, Audit Services
- **.NET (C#)**: Order Service, Payment Service
- **Python/FastAPI**: Product Service, Inventory Service
- **Java/Spring Boot**: Order Processor Service
- **Go**: Cart Service, Message Broker Service

### Databases

- **MongoDB**: Document storage for Auth, User, Product, Review, Admin, Audit Services
- **PostgreSQL**: Relational data for Order, Inventory, Order Processor Services
- **SQL Server**: Payment Service (PCI compliance requirements)
- **Redis**: Cache and session store for Cart Service

### Message Broker & Events

- **RabbitMQ**: Primary message broker
- **Dapr Pub/Sub**: Framework-agnostic event abstraction
  - Services publish/consume events without direct broker coupling
  - Supports multiple broker backends (RabbitMQ, Kafka, Azure Service Bus)
  - Automatic retry, dead-letter queues, at-least-once delivery

### Observability Stack

- **Distributed Tracing**: Jaeger or Zipkin
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Metrics**: Prometheus + Grafana
- **Health Checks**: Built-in `/health` and `/health/ready` endpoints

### Infrastructure

- **Container Runtime**: Docker
- **Orchestration**: Kubernetes
- **CI/CD**: GitHub Actions
- **Secrets Management**: Azure Key Vault / Kubernetes Secrets
- **API Gateway**: Kong or Nginx Ingress Controller

---

## Communication Patterns

### 1. Synchronous Communication (REST)

**Use Case**: Real-time queries, read operations

**Pattern**: Client → API Gateway → Service → Database

**Example**:

```
GET /api/products/507f1f77bcf86cd799439011
Web UI → API Gateway → Product Service → MongoDB → Response
```

**Services Using REST**:

- Product Service: CRUD operations
- User Service: Profile management
- Cart Service: Cart operations
- Order Service: Order queries

### 2. Asynchronous Communication (Events via Dapr)

**Use Case**: Event-driven workflows, eventual consistency

**Pattern**: Service A → Dapr Pub/Sub → Message Broker → Dapr Pub/Sub → Service B

**Example**:

```
Product Service publishes: product.created event
                ↓
        Dapr Pub/Sub (RabbitMQ)
                ↓
        ┌───────┴───────┬───────────┐
        ↓               ↓           ↓
  Audit Service  Notification  Search Service
                   Service
```

**Event Flow Examples**:

1. **Product Created**:

   - Product Service → `product.created` → Audit Service (logs), Notification Service (alerts)

2. **Order Placed**:

   - Order Service → `order.created` → Inventory Service (reserve stock), Payment Service (charge), Notification Service (email)

3. **Review Posted**:

   - Review Service → `review.created` → Product Service (update rating), Audit Service (log)

4. **Payment Completed**:
   - Payment Service → `payment.completed` → Order Service (update status), Notification Service (send receipt)

### 3. Service-to-Service REST (Direct)

**Use Case**: Immediate data validation

**Example**:

```
Order Service → (REST) → Product Service (validate product exists)
Order Service → (REST) → Inventory Service (check stock)
```

**Pattern**: Synchronous validation before order creation

---

## Data Architecture

### Data Ownership

Each service owns its data and schema:

| Service           | Database   | Data Ownership                           |
| ----------------- | ---------- | ---------------------------------------- |
| Product Service   | MongoDB    | Products, variations, badges, categories |
| User Service      | MongoDB    | Users, profiles, addresses               |
| Order Service     | PostgreSQL | Orders, order items, order status        |
| Inventory Service | PostgreSQL | Stock levels, reservations, warehouses   |
| Cart Service      | Redis      | Active shopping carts (TTL-based)        |
| Payment Service   | SQL Server | Payment transactions, refunds            |
| Review Service    | MongoDB    | Reviews, ratings, moderation status      |
| Audit Service     | MongoDB    | Audit logs, event history                |

### Data Consistency Models

1. **Strong Consistency**: Within service boundaries (ACID transactions)
2. **Eventual Consistency**: Cross-service via events (5-10 seconds SLA)

**Example: Product Review Aggregation**

```
Review Service (source of truth)
        ↓ (publishes review.created event)
Product Service (denormalized review aggregates)
        ↓ (eventually consistent within 5 seconds)
Customer sees updated rating
```

### Data Denormalization Strategy

**Product Service** maintains denormalized data for performance:

- Review aggregates (from Review Service)
- Inventory availability (from Inventory Service)
- Sales metrics (from Analytics Service)

**Trade-off**: Faster reads, eventual consistency (5-10s delay)

---

## Infrastructure Components

### Dapr (Distributed Application Runtime)

**Purpose**: Microservices building blocks

**Components Used**:

1. **Pub/Sub** (Primary Use Case):

   ```yaml
   # Product Service publishes event via Dapr
   POST http://localhost:3500/v1.0/publish/pubsub/product.created

   # Audit Service subscribes via Dapr
   GET http://localhost:3500/v1.0/subscribe/pubsub
   ```

2. **Service Invocation**:

   ```
   Order Service → Dapr → Product Service (validate product)
   ```

3. **State Management** (Cart Service):

   ```
   Dapr State Store → Redis (cart data with TTL)
   ```

4. **Secrets Management**:
   ```
   Dapr Secrets API → Azure Key Vault / Kubernetes Secrets
   ```

**Benefits**:

- Framework-agnostic (works with Node.js, .NET, Python, Go, Java)
- Broker-agnostic (switch from RabbitMQ to Kafka without code changes)
- Built-in retry, circuit breaker, distributed tracing

### Message Broker (RabbitMQ)

**Configuration**:

- **Exchanges**: Topic-based routing
- **Queues**: Durable, per-service consumer groups
- **Dead Letter Queue**: Failed messages after max retries

**Example Topics**:

- `product.events` (product.created, product.updated, product.deleted)
- `order.events` (order.created, order.updated, order.cancelled)
- `payment.events` (payment.completed, payment.failed)

### API Gateway

**Responsibilities**:

- Request routing to backend services
- Rate limiting (100 req/min per IP for public, 500 req/min for authenticated)
- JWT token validation
- SSL/TLS termination
- CORS handling
- Request/response transformation

---

## Security Architecture

### Authentication Flow

```
1. User Login
   Web UI → Auth Service → JWT Token (signed)

2. API Request with JWT
   Web UI → API Gateway (validates JWT) → Service → Response

3. Service-to-Service
   Service A → API Gateway (service token) → Service B
```

### JWT Token Structure

```json
{
  "sub": "user-123",
  "role": "customer", // or "admin"
  "iat": 1699000000,
  "exp": 1699086400
}
```

### Role-Based Access Control (RBAC)

- **Customer Role**: Read-only access to active products, orders, reviews
- **Admin Role**: Full CRUD, statistics, audit logs, system configuration

### Secrets Management

- **Development**: Environment variables in `.env` files
- **Production**: Azure Key Vault or Kubernetes Secrets
- **Dapr Integration**: Services fetch secrets via Dapr Secrets API

---

## Deployment Architecture

### Container Strategy

Each service:

- Runs in Docker container
- Exposed via Kubernetes Service
- Scaled horizontally via Kubernetes HPA (Horizontal Pod Autoscaler)

### Kubernetes Resources

**Per Service**:

```yaml
- Deployment (pods, replicas, health checks)
- Service (ClusterIP for internal, LoadBalancer for external)
- ConfigMap (non-sensitive config)
- Secret (sensitive config)
- HorizontalPodAutoscaler (auto-scaling rules)
```

### Scaling Strategy

| Service         | Min Replicas | Max Replicas | Trigger   |
| --------------- | ------------ | ------------ | --------- |
| Product Service | 2            | 10           | CPU > 70% |
| Order Service   | 2            | 8            | CPU > 70% |
| Cart Service    | 1            | 5            | CPU > 60% |
| Auth Service    | 2            | 6            | CPU > 70% |

### Health Checks

All services expose:

- `/health` - Liveness probe (is process alive?)
- `/health/ready` - Readiness probe (is service ready to accept traffic?)

---

## Cross-Cutting Concerns

### Distributed Tracing

**Implementation**: Correlation ID propagation

```
Request → Service A (generates correlation-id: abc-123)
       → Service B (propagates correlation-id: abc-123)
       → Service C (propagates correlation-id: abc-123)
```

**Header**: `X-Correlation-ID`

**Tracing Tools**: Jaeger, Zipkin

### Structured Logging

**Format**: JSON logs with standard fields

```json
{
  "timestamp": "2025-11-03T14:32:10.123Z",
  "level": "info",
  "service": "product-service",
  "correlationId": "abc-123",
  "userId": "user-456",
  "operation": "CreateProduct",
  "message": "Product created successfully"
}
```

### Monitoring & Alerting

**Metrics Exposed**: Prometheus format at `/metrics`

**Common Alerts**:

- Error rate > 5% for 5 minutes
- P95 latency > 500ms for 5 minutes
- Pod crash loop
- Database connection pool exhausted

---

## Service Dependencies

### Product Service Dependencies

**Publishes Events To**:

- Audit Service (audit logging)
- Notification Service (back-in-stock alerts)
- Order Service (product validation)

**Consumes Events From**:

- Review Service (review aggregates)
- Inventory Service (stock availability)
- Analytics Service (sales metrics)

**Direct REST Calls**: None (fully event-driven)

### Order Service Dependencies

**Publishes Events To**:

- Inventory Service (stock reservation)
- Payment Service (payment processing)
- Notification Service (order confirmation)
- Audit Service (audit logging)

**Consumes Events From**:

- Payment Service (payment status)
- Inventory Service (stock confirmation)

**Direct REST Calls**:

- Product Service (validate product exists)
- User Service (validate user exists)

---

## Future Enhancements

1. **API Versioning**: Support `/api/v1/` and `/api/v2/` endpoints
2. **GraphQL Gateway**: Unified query interface for web/mobile
3. **Event Sourcing**: Complete event history for orders
4. **CQRS**: Separate read/write models for Product Service
5. **Service Mesh**: Istio for advanced traffic management
6. **Real-Time Updates**: WebSocket support via SignalR/Socket.io
7. **Multi-Region**: Active-active deployment across regions

---

## Document Change History

| Version | Date       | Changes                       | Author  |
| ------- | ---------- | ----------------------------- | ------- |
| 1.0     | 2025-11-03 | Initial platform architecture | AI Team |

---

## References

- [Product Service PRD](../services/product-service/docs/PRD.md)
- [Product Service Architecture](../services/product-service/docs/ARCHITECTURE.md)
- [Dapr Documentation](https://docs.dapr.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
