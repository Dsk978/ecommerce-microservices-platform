# E-Commerce Microservices Platform

 
> Java 21 · Spring Boot 3.2 · Kafka · React.js · MongoDB · Redis · Keycloak · Docker · Kubernetes

---

## What is this project?

A production-style **E-Commerce platform** built as 4 independent microservices —
each service owns its own database, communicates asynchronously via Kafka, and is
secured with Keycloak OAuth2/JWT. The entire platform is containerised with Docker
and orchestrated via Kubernetes with auto-scaling (HPA).

This project demonstrates:
- True microservice isolation — each service has its own DB, no shared tables
- Async event-driven communication via Kafka — no tight coupling between services
- Production-grade security via Keycloak OAuth2/JWT
- Resilience patterns via Resilience4j circuit breaker
- Full observability via Prometheus and Grafana
- Kubernetes orchestration with Horizontal Pod Autoscaler

---

## What is Microservices Architecture?

In a monolith, all features (products, orders, inventory, notifications) live in
one codebase and one database. If one part fails, everything fails.

In microservices, each feature is an **independent service** with its own codebase,
its own database, and its own deployment. They talk to each other through APIs or
events.
```mermaid
flowchart LR
    subgraph Monolith
        M[Single App\nProducts + Orders\n+ Inventory + Notifications]
    end

    subgraph Microservices
        P[Product Service]
        O[Order Service]
        I[Inventory Service]
        N[Notification Service]
    end

    Monolith -->|breaks into| Microservices
```

---

## High Level Design
```mermaid
flowchart TD
    USER([React.js Frontend\nCatalogue · Cart · Orders])
    GW[API Gateway\nRouting · Auth filter]
    PS[Product Service\nMongoDB]
    OS[Order Service\nMySQL]
    IS[Inventory Service\nMySQL]
    NS[Notification Service\nEmail · SMS]
    KF[Kafka\nAsync Event Streaming]
    KC[Keycloak\nOAuth2 · JWT]
    RD[(Redis Cache)]
    PROM[Prometheus + Grafana\nObservability]

    USER -->|REST via Axios| GW
    GW -->|validate JWT| KC
    KC -->|token valid| GW
    GW --> PS
    GW --> OS
    GW --> IS
    OS -->|OrderPlaced event| KF
    IS -->|StockUpdated event| KF
    KF -->|consume events| NS
    PS <-->|cache reads| RD
    PS & OS & IS & NS -->|metrics| PROM
```

---

## The 4 Microservices
```mermaid
flowchart LR
    subgraph Product Service
        P1[REST API\nCRUD products]
        P2[(MongoDB\nFlexible schema)]
        P3[(Redis\nCache layer)]
        P1 --- P2
        P1 --- P3
    end

    subgraph Order Service
        O1[REST API\nPlace orders]
        O2[(MySQL\nOrder records)]
        O1 --- O2
    end

    subgraph Inventory Service
        I1[REST API\nStock levels]
        I2[(MySQL\nStock records)]
        I1 --- I2
    end

    subgraph Notification Service
        N1[Kafka Consumer]
        N2[Email / SMS]
        N1 --- N2
    end

    O1 -->|check stock| I1
    O1 -->|OrderPlaced event| K[Kafka]
    I1 -->|StockUpdated event| K
    K --> N1
```

---

## Order Placement — Step by Step Flow
```mermaid
sequenceDiagram
    participant U as React Frontend
    participant G as API Gateway
    participant KC as Keycloak
    participant O as Order Service
    participant I as Inventory Service
    participant K as Kafka
    participant N as Notification Service

    U->>G: POST /orders (Bearer JWT)
    G->>KC: Validate token
    KC-->>G: Token valid
    G->>O: Forward order request
    O->>I: GET /inventory/check (is stock available?)
    I-->>O: Yes, stock available
    O->>O: Save order to MySQL
    O->>K: Publish OrderPlaced event
    K->>N: Consume OrderPlaced event
    N-->>U: Send confirmation email
    O-->>U: 201 Order Created
```

---

## Security Flow — Keycloak OAuth2 / JWT

Instead of each microservice managing its own login logic, Keycloak acts as a
central Identity Provider. Every service trusts the same JWT token.
```mermaid
flowchart LR
    U[User / Frontend]
    KC[Keycloak\nAuth Server]
    GW[API Gateway]
    SVC[Microservices]

    U -->|1 - Login with credentials| KC
    KC -->|2 - Returns JWT access token| U
    U -->|3 - API request + Bearer token| GW
    GW -->|4 - Validate token signature| KC
    KC -->|5 - Valid| GW
    GW -->|6 - Forward to service| SVC
    SVC -->|7 - Response| U
```

**Why Keycloak?**
- Zero auth code inside microservices — they just validate the JWT
- Role-based access control managed in one place
- Industry-standard OAuth2/OpenID Connect
- Supports SSO across all services

---

## Resilience Pattern — Circuit Breaker

When Order Service calls Inventory Service, what happens if Inventory is down?
Without a circuit breaker, Order Service hangs and eventually crashes too
(cascade failure). Resilience4j prevents this.
```mermaid
flowchart LR
    OS[Order Service]
    CB{Resilience4j\nCircuit Breaker}
    IS[Inventory Service]
    FB[Fallback Response\nDefault stock value]

    OS --> CB
    CB -->|CLOSED\nnormal traffic| IS
    CB -->|OPEN\ntoo many failures| FB
    FB -->|safe default| OS
```

**Three states:**
- `CLOSED` — everything normal, requests pass through
- `OPEN` — too many failures detected, requests go to fallback immediately
- `HALF-OPEN` — testing if the service recovered, lets a few requests through

---

## Why Kafka over REST for inter-service communication?

| | Synchronous REST | Asynchronous Kafka |
|---|---|---|
| Coupling | Tight — Order waits for Notification | Loose — Order publishes, moves on |
| If Notification is down | Order request fails | Event stays in Kafka, consumed later |
| Scalability | Limited by slowest service | Each service scales independently |
| Use case | Inventory check (needs immediate answer) | Notification (fire and forget) |

In this project:
- Order → Inventory = **REST** (needs immediate stock confirmation)
- Order → Notification = **Kafka** (fire and forget, no need to wait)

---

## Tech Stack

| Technology | Version | Role in this project |
|---|---|---|
| **Java** | 21 | Core language — virtual threads, records |
| **Spring Boot** | 3.2 | Microservice framework for all 4 services |
| **Apache Kafka** | — | Async event streaming — OrderPlaced, StockUpdated events |
| **React.js** | — | Frontend — product catalogue, cart, order history via Axios |
| **MongoDB** | — | Product Service DB — flexible document store for product catalogue |
| **MySQL** | — | Order + Inventory Service — relational data, ACID transactions |
| **Redis** | — | Product catalogue cache — reduces MongoDB load on repeated reads |
| **Keycloak** | — | OAuth2/JWT auth server — central identity for all services |
| **Resilience4j** | — | Circuit breaker — prevents cascade failures between services |
| **Docker Compose** | — | Local multi-service orchestration — one command startup |
| **Kubernetes** | — | Production orchestration — HPA, ConfigMaps, Services |
| **Prometheus** | — | Scrapes metrics from all services every 15 seconds |
| **Grafana** | — | Live dashboards — request rate, latency, error rate per service |

---

## Why MongoDB for Products but MySQL for Orders?
```mermaid
flowchart LR
    subgraph MongoDB - Products
        M1[Flexible schema]
        M2[Product has varying\nattributes per category]
        M3[Shoes have size+color\nLaptops have RAM+CPU]
    end

    subgraph MySQL - Orders
        S1[Fixed schema]
        S2[Orders always have\nsame structure]
        S3[ACID transactions\ncritical for payments]
    end
```

- Products have different attributes per category — MongoDB's flexible documents
  handle this naturally. A shoe has size/color, a laptop has RAM/CPU.
- Orders always have the same structure and need ACID transaction guarantees —
  MySQL is the right tool.

---

## Kubernetes Setup
```mermaid
flowchart TD
    HPA[HPA - Horizontal Pod Autoscaler\nmin=2 max=10 · target CPU 70%]
    CM[ConfigMaps\nenv variables]
    SVC[Services\nload balancing]
    P1[Pod 1]
    P2[Pod 2]
    P3[Pod 3 - auto scaled]

    HPA --> P1
    HPA --> P2
    HPA --> P3
    CM --> P1
    CM --> P2
    CM --> P3
    SVC --> P1
    SVC --> P2
    SVC --> P3
```

HPA automatically adds more pods when CPU exceeds 70% and removes them when
load drops — zero manual intervention.

---

## Observability Stack

Each service exposes metrics at `/actuator/prometheus`. Prometheus scrapes all
services and Grafana visualises them in real time.

**Metrics tracked per service:**
- HTTP request rate and latency (p50, p95, p99)
- Error rate (4xx, 5xx)
- JVM memory and GC activity
- Kafka consumer lag for Notification Service
- Redis cache hit/miss ratio for Product Service

---

## Project Structure
```
ecommerce-microservices-platform/
├── product-service/          # Spring Boot · MongoDB · Redis cache
│   ├── src/
│   └── Dockerfile
├── order-service/            # Spring Boot · MySQL · Kafka producer
│   ├── src/
│   └── Dockerfile
├── inventory-service/        # Spring Boot · MySQL · REST API
│   ├── src/
│   └── Dockerfile
├── notification-service/     # Spring Boot · Kafka consumer · Email
│   ├── src/
│   └── Dockerfile
├── api-gateway/              # Spring Cloud Gateway · Keycloak filter
├── frontend/                 # React.js · Axios · Cart · Orders
├── kubernetes/               # HPA · ConfigMaps · Services per microservice
└── docker-compose.yml        # Full local stack — one command startup
```

---

## Key Results

- 4 independent microservices each with their own DB — true service isolation
- Kafka async pipeline — order processing never blocked by notification delays
- Resilience4j circuit breaker — inventory failures return graceful fallback
- Keycloak OAuth2 — zero auth code inside microservices
- Kubernetes HPA — auto-scales pods on CPU load, zero downtime
- Full observability — Prometheus metrics scraped every 15s, Grafana dashboards
- Redis cache — repeated product reads served without hitting MongoDB

---



**Sai Krishna Darla** — Java Backend Engineer  
[LinkedIn](https://www.linkedin.com/in/saikrishna-darla/) ·
[GitHub](https://github.com/Dsk978)
```
```
