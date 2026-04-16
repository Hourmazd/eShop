# eShop Reference Application — Architecture

> **"AdventureWorks"** · .NET 10 · .NET Aspire 13 · Microservices · Event-Driven

---

## Table of Contents

1. [Business Requirements](#1-business-requirements)
2. [Solution-Level Architecture](#2-solution-level-architecture)
3. [Project-Level Architecture](#3-project-level-architecture)
4. [Data Model](#4-data-model)
5. [Deployment Model](#5-deployment-model)
6. [Security](#6-security)
7. [High Availability & Resiliency](#7-high-availability--resiliency)

---

## 1. Business Requirements

### 1.1 Functional Requirements

| # | Requirement | Implemented By |
|---|-------------|----------------|
| BR-01 | Customers can browse a product catalog with filtering, pagination, and search | Catalog.API + WebApp |
| BR-02 | Customers can perform AI-powered semantic product search | Catalog.API (pgvector + OpenAI/Ollama) |
| BR-03 | Customers can add products to a persistent shopping basket | Basket.API (Redis) |
| BR-04 | Customers can place orders and track their status end-to-end | Ordering.API + OrderProcessor |
| BR-05 | Order stock is validated before fulfilment; insufficient stock cancels the order | Catalog.API ↔ Ordering.API via event bus |
| BR-06 | Payment is processed asynchronously after stock is confirmed | PaymentProcessor |
| BR-07 | Customers receive real-time order status updates in the UI | WebApp subscribes to integration events |
| BR-08 | Third parties can subscribe to business events via webhooks | Webhooks.API + WebhookClient |
| BR-09 | The application is accessible from a web browser and native mobile/desktop | WebApp + ClientApp (MAUI) + HybridApp |
| BR-10 | Staff can manage catalog stock levels and product data | Catalog.API admin endpoints |

### 1.2 Non-Functional Requirements

| # | Requirement | Target |
|---|-------------|--------|
| NFR-01 | Service independence — each domain can be deployed and scaled separately | Microservices, per-service DB |
| NFR-02 | Eventual consistency — partial failures must not corrupt global state | Outbox pattern, Saga coordination |
| NFR-03 | Idempotent operations — retried requests must not create duplicate orders | IdentifiedCommand pattern |
| NFR-04 | Observability — all services emit correlated traces, metrics, and logs | OpenTelemetry (OTLP) |
| NFR-05 | Security — all APIs protected with industry-standard authentication | OAuth 2.0 / OpenID Connect |
| NFR-06 | Developer ergonomics — single-command local startup of the entire system | .NET Aspire AppHost |
| NFR-07 | Extensibility — AI provider can be swapped without service changes | Pluggable AI (OpenAI / Ollama / Azure OpenAI) |

---

## 2. Solution-Level Architecture

### 2.1 Architectural Style

eShop is a **microservices, event-driven** application orchestrated by **.NET Aspire**. Each bounded context owns its own process, database, and deployment unit. Services never share databases; all cross-service data flow is mediated by integration events over a message broker (RabbitMQ) or over typed HTTP/gRPC clients.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Clients                                       │
│  ┌──────────────┐  ┌───────────────────┐  ┌──────────────────────┐  │
│  │  WebApp      │  │  HybridApp        │  │  ClientApp           │  │
│  │  (Blazor     │  │  (MAUI + Blazor)  │  │  (MAUI iOS/Android/  │  │
│  │   Server)    │  │                   │  │   Windows/macOS)     │  │
│  └──────┬───────┘  └────────┬──────────┘  └─────────┬────────────┘  │
└─────────┼───────────────────┼─────────────────────────┼─────────────┘
          │ HTTPS             │ HTTPS                   │ HTTPS (via BFF)
          │                   │                         │
┌─────────▼───────────────────▼─────────────────────────▼─────────────┐
│                        API Surface                                    │
│  ┌────────────────────┐    ┌───────────────────────────────────────┐ │
│  │  Identity.API      │    │  YARP Mobile BFF                      │ │
│  │  (Duende IS4 /     │    │  (Reverse proxy for ClientApp /       │ │
│  │   OpenID Connect)  │    │   HybridApp)                          │ │
│  └────────────────────┘    └───────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────────────────────────────────────────────────────────────┐
│                      Backend Microservices                                   │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  Basket.API  │  │  Catalog.API │  │  Ordering.API│  │  Webhooks.API  │  │
│  │  (gRPC+HTTP) │  │  (HTTP REST) │  │  (HTTP REST) │  │  (HTTP REST)   │  │
│  │  Redis       │  │  PostgreSQL  │  │  PostgreSQL  │  │  PostgreSQL    │  │
│  │              │  │  + pgvector  │  │  + outbox    │  │                │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘  │
│         │                 │                  │                   │           │
│  ┌──────▼─────────────────▼──────────────────▼───────────────────▼────────┐ │
│  │                  RabbitMQ Event Bus  ("eshop_event_bus")                │ │
│  └──────┬─────────────────┬──────────────────┬───────────────────┬────────┘ │
│         │                 │                  │                   │           │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐           │           │
│  │ OrderProcessor│  │PaymentProcessor│ │  (Catalog    │           │           │
│  │ (Worker)     │  │ (Worker)     │  │   stock sub) │           │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │           │
└──────────────────────────────────────────────────────────────────────────────┘
          │
┌─────────▼────────────────────────────────────────────────────────────┐
│                       Infrastructure                                   │
│  ┌───────────────┐  ┌───────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  PostgreSQL   │  │  Redis    │  │  RabbitMQ    │  │  Aspire   │  │
│  │  (per-service │  │  (Basket  │  │  (Event Bus) │  │  Dashboard│  │
│  │   databases)  │  │   state)  │  │              │  │  + OTLP   │  │
│  └───────────────┘  └───────────┘  └──────────────┘  └───────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Integration Event Flow (Order Saga)

The order lifecycle is coordinated by a **choreography-based Saga** over RabbitMQ. No central orchestrator exists; each service reacts to events and publishes follow-on events.

```
WebApp                Ordering.API         Basket.API       Catalog.API      PaymentProcessor
   │                      │                    │                │                  │
   │── CreateOrder ───────►│                    │                │                  │
   │                      │── OrderStarted ────►│                │                  │
   │                      │   (clear basket)    │                │                  │
   │                      │── AwaitingValidation ───────────────►│                  │
   │                      │                                      │── StockConfirmed ►│
   │                      │◄─────────────────────────────────────│                  │
   │                      │── StockConfirmed ──────────────────────────────────────►│
   │                      │                                                          │
   │                      │◄──────────────────────────── PaymentSucceeded ───────────│
   │                      │── Paid / Shipped ────────────────────────────────────────│
   │◄─ status update ─────│
```

### 2.3 Shared Libraries

| Library | Purpose |
|---------|---------|
| **eShop.ServiceDefaults** | Extension methods wired once: OpenTelemetry, health checks, service discovery, standard HTTP resilience handler |
| **EventBus** | Abstractions: `IEventBus`, `IntegrationEvent` base, `IIntegrationEventHandler<T>` |
| **EventBusRabbitMQ** | Concrete RabbitMQ implementation with OTLP context propagation |
| **IntegrationEventLogEF** | Outbox pattern: `IntegrationEventLogService` persists events in the same EF transaction, then publishes |
| **Ordering.Domain** | DDD value objects, aggregates, domain events (shared by Ordering.API and Ordering.Infrastructure) |
| **Ordering.Infrastructure** | EF Core `OrderingContext`, repository implementations, migrations |
| **WebAppComponents** | Shared Razor components used by WebApp and HybridApp |

---

## 3. Project-Level Architecture

### 3.1 Identity.API

**Role:** Centralised authentication and authorisation authority.

**Technology:** ASP.NET Core · Duende IdentityServer 4 · ASP.NET Identity · PostgreSQL

**Responsibilities:**
- Issues JWT Bearer tokens and OpenID Connect ID tokens for all services and clients
- Stores users in `ApplicationUser` (ASP.NET Identity) backed by `identitydb` on PostgreSQL
- Hosts the OIDC discovery endpoint (`/.well-known/openid-configuration`)
- Seed default test users on startup via `UsersSeed`

**Key Interfaces:**

| Endpoint | Protocol | Purpose |
|----------|----------|---------|
| `/connect/authorize` | HTTPS | OAuth 2.0 authorization code flow |
| `/connect/token` | HTTPS | Token exchange |
| `/connect/userinfo` | HTTPS | User claims |
| `/.well-known/openid-configuration` | HTTPS | Discovery |

**Registered Clients:** `webapp`, `maui`, `webhooksclient`, `orderingswaggerui`, `catalogswaggerui`, `basketswaggerui`

**Scopes:** `openid`, `profile`, `orders`, `basket`, `webhooks`

---

### 3.2 Basket.API

**Role:** Manages per-user shopping baskets.

**Technology:** ASP.NET Core · gRPC · Redis · RabbitMQ (subscriber)

**Responsibilities:**
- Exposes a gRPC service (`GetBasket`, `UpdateBasket`, `DeleteBasket`) consumed by WebApp and ClientApp
- Stores basket data in Redis as JSON keyed by `{userId}`
- Subscribes to `OrderStartedIntegrationEvent` → clears the basket after a successful order
- Subscribes to `ProductPriceChangedIntegrationEvent` → updates prices in open baskets

**Key Patterns:**
- Redis as ephemeral store (baskets are recreated if lost — acceptable for cart data)
- gRPC HTTP/2 for efficient binary serialization

**gRPC Service:**
```
service Basket {
    rpc GetBasket(GetBasketRequest) → CustomerBasketResponse
    rpc UpdateBasket(UpdateBasketRequest) → CustomerBasketResponse
    rpc DeleteBasket(DeleteBasketRequest) → DeleteBasketResponse
}
```

---

### 3.3 Catalog.API

**Role:** Product catalog with AI-powered search.

**Technology:** ASP.NET Core Minimal API · Entity Framework Core · PostgreSQL + pgvector · OpenAI / Ollama

**Responsibilities:**
- CRUD for catalog items, brands, and types with pagination
- Manages product stock levels (`RemoveStock`, `AddStock`)
- Generates and stores vector embeddings for each catalog item
- Semantic search endpoint: `/api/catalog/items/withsemanticrelevance/{text}`
- Publishes `ProductPriceChangedIntegrationEvent` when item price changes
- Subscribes to `OrderStatusChangedToAwaitingValidationIntegrationEvent` → validates and reserves stock
- Subscribes to `OrderStatusChangedToCancelledIntegrationEvent` → returns reserved stock

**API Versioning:** v1.0 and v2.0 (v2 adds improved semantic search and embedding refresh)

**AI Integration (pluggable):**
- Azure OpenAI embeddings (`text-embedding-3-small`)
- OpenAI embeddings
- Ollama embeddings (local, no cloud dependency)

---

### 3.4 Ordering.API

**Role:** Order management implementing DDD + CQRS.

**Technology:** ASP.NET Core · MediatR · FluentValidation · Entity Framework Core · PostgreSQL · RabbitMQ

**Responsibilities:**
- Accepts order creation requests from WebApp / ClientApp
- Enforces idempotency via `IdentifiedCommand<TCommand>` — duplicate requests with the same `RequestId` are no-ops
- Coordinates the order saga by publishing and subscribing to integration events
- Exposes order query endpoints for authenticated users

**Layered Structure:**

```
Ordering.API            (HTTP endpoints, DI wiring, Saga event handlers)
    └── Ordering.Infrastructure  (EF Core, repositories, outbox)
            └── Ordering.Domain  (Aggregates, value objects, domain events)
```

**CQRS Commands:**

| Command | Effect |
|---------|--------|
| `CreateOrderCommand` | Creates Order aggregate, raises `OrderStartedDomainEvent` |
| `CancelOrderCommand` | Transitions order to Cancelled |
| `SetAwaitingValidationOrderStatusCommand` | Moves order to AwaitingValidation |
| `SetStockConfirmedOrderStatusCommand` | Moves order to StockConfirmed |
| `SetStockRejectedOrderStatusCommand` | Cancels order with rejection reason |
| `SetPaidOrderStatusCommand` | Moves order to Paid |
| `ShipOrderCommand` | Moves order to Shipped |

**MediatR Pipeline Behaviors:**

```
Request → LoggingBehavior → ValidatorBehavior → TransactionBehavior → Handler
```

**Order State Machine:**

```
Submitted → AwaitingValidation → StockConfirmed → Paid → Shipped
                                        │
                                        └──────────────────► Cancelled
```

---

### 3.5 OrderProcessor (Worker Service)

**Role:** Background worker coordinating time-sensitive order state transitions.

**Technology:** .NET Worker Service · MediatR · RabbitMQ subscriber

**Responsibilities:**
- Subscribes to `GracePeriodConfirmedIntegrationEvent` → dispatches `SetAwaitingValidationOrderStatusCommand`
- Provides a configurable grace period before stock validation begins (allows order cancellations)
- Runs as a long-lived hosted service, decoupled from the HTTP request pipeline

---

### 3.6 PaymentProcessor

**Role:** Simulated payment gateway.

**Technology:** ASP.NET Core Worker / minimal API · RabbitMQ subscriber

**Responsibilities:**
- Subscribes to `OrderStatusChangedToStockConfirmedIntegrationEvent`
- Simulates payment processing (always succeeds in reference implementation)
- Publishes `OrderPaymentSucceededIntegrationEvent` or `OrderPaymentFailedIntegrationEvent`

---

### 3.7 Webhooks.API

**Role:** Delivers integration events to external subscriber URLs.

**Technology:** ASP.NET Core · Entity Framework Core · PostgreSQL · RabbitMQ

**Responsibilities:**
- Manages webhook subscriptions via a REST API (`/api/v1/webhooks`)
- Subscribes to `ProductPriceChangedIntegrationEvent` and order status events from RabbitMQ
- On event receipt, retrieves matching subscriptions from `webhooksdb` and delivers via HTTP POST
- Tracks delivery status per webhook entry (Pending / Delivered / Failed)

---

### 3.8 WebApp

**Role:** Primary web client for end users.

**Technology:** ASP.NET Core · Blazor Server (Razor Components) · Interactive Server Rendering · gRPC client · OIDC

**Responsibilities:**
- Hosts the storefront: browse catalog, view product detail, manage basket, checkout, order history
- Authenticates users via OIDC code flow against Identity.API
- Propagates bearer tokens to downstream services via `HttpClientAuthorizationDelegatingHandler`
- Subscribes to integration events to push real-time order updates to UI (SignalR-backed Blazor)
- Proxies catalog product images via `/product-images/{id}`

**HTTP Clients:**

| Client | Destination | Protocol |
|--------|-------------|----------|
| `Basket.BasketClient` | basket-api | gRPC (HTTP/2) |
| `CatalogService` | catalog-api | HTTPS (REST v2) |
| `OrderingService` | ordering-api | HTTPS (REST v1) |

---

### 3.9 ClientApp (MAUI)

**Role:** Native cross-platform mobile and desktop app.

**Platforms:** iOS 15+, Android 21+, Windows 10, macOS 15+

**Technology:** .NET MAUI · MVVM (CommunityToolkit.Mvvm) · IdentityModel.OidcClient · gRPC client

**Responsibilities:**
- Provides a native shopping experience on iOS/Android/Windows/macOS
- Authenticates via OIDC through Mobile BFF (YARP)
- Shares service clients with WebApp where possible (typed HTTP clients)

---

### 3.10 HybridApp

**Role:** MAUI app embedding Blazor Razor components for cross-platform web-native UI.

**Technology:** .NET MAUI · BlazorWebView · WebAppComponents (shared Razor class library)

**Responsibilities:**
- Renders the same Razor component tree as WebApp inside a native WebView shell
- Shares authentication and navigation with ClientApp

---

### 3.11 WebhookClient

**Role:** Management UI for webhook subscriptions.

**Technology:** ASP.NET Core · Blazor Server · OIDC

**Responsibilities:**
- Lets users subscribe/unsubscribe external URLs to eShop events
- Displays webhook delivery history and status
- Authenticates users via OIDC (scope: `webhooks`)

---

### 3.12 eShop.AppHost

**Role:** .NET Aspire orchestration host for local development.

**Technology:** .NET Aspire 13 AppHost

**Responsibilities:**
- Declares all services, infrastructure containers, and their dependency graph
- Injects connection strings, service discovery endpoints, and OTLP configuration automatically
- Starts PostgreSQL (pgvector image), Redis, RabbitMQ as persistent containers
- Registers optional AI resources (Azure OpenAI / Ollama)
- Exposes the Aspire Dashboard for traces, logs, metrics, and resource health

**Declared Resources:**

```
postgres (ankane/pgvector)
  ├── catalogdb
  ├── identitydb
  ├── orderingdb
  └── webhooksdb

redis
rabbitmq ("eventbus")

Identity.API    → identitydb
Basket.API      → redis, eventbus
Catalog.API     → catalogdb, eventbus [, openai|ollama]
Ordering.API    → orderingdb, eventbus
OrderProcessor  → orderingdb, eventbus, waits-for Ordering.API
PaymentProcessor→ eventbus
Webhooks.API    → webhooksdb, eventbus
WebApp          → basket-api (gRPC), catalog-api, ordering-api, identity-api [, ollama]
WebhookClient   → identity-api, webhooks-api
mobile-bff      → catalog-api, ordering-api, identity-api, basket-api
```

---

### 3.13 eShop.ServiceDefaults

**Role:** Shared configuration applied to every service by calling `builder.AddServiceDefaults()`.

**What it wires:**
- OpenTelemetry — traces (ASP.NET Core, gRPC, HttpClient), metrics (runtime, ASP.NET Core, HttpClient), logs
- OTLP exporter (when `OTEL_EXPORTER_OTLP_ENDPOINT` set)
- Health check endpoints: `/health` (readiness) and `/alive` (liveness)
- Service discovery (`Microsoft.Extensions.ServiceDiscovery`)
- Default HTTP client resilience: `AddStandardResilienceHandler()` (circuit breaker + retry + timeout)
- Default API versioning and OpenAPI document generation

---

## 4. Data Model

### 4.1 Database Topology

Each service owns exactly one logical database on a shared PostgreSQL instance (easily separated into distinct servers in production).

| Service | Database | Schema | Persistence Technology |
|---------|----------|--------|----------------------|
| Catalog.API | `catalogdb` | public | PostgreSQL + pgvector |
| Identity.API | `identitydb` | public | PostgreSQL |
| Ordering.API | `orderingdb` | `ordering` | PostgreSQL |
| Webhooks.API | `webhooksdb` | public | PostgreSQL |
| Basket.API | — | — | Redis (key-value) |

### 4.2 Catalog Domain

```
CatalogBrand          CatalogType
──────────            ──────────
Id (int, PK)          Id (int, PK)
Brand (string)        Type (string)
      │                     │
      └─────────┬───────────┘
                │
           CatalogItem
           ──────────────────────────────────────────
           Id              (int, PK)
           Name            (string)
           Description     (string)
           Price           (decimal)
           PictureFileName (string)
           CatalogTypeId   (int, FK → CatalogType)
           CatalogBrandId  (int, FK → CatalogBrand)
           AvailableStock  (int)
           RestockThreshold(int)
           MaxStockThreshold(int)
           OnReorder       (bool)
           Embedding       (vector, pgvector)  ← AI embedding for semantic search

IntegrationEventLog (outbox)
──────────────────────────────
EventId       (Guid, PK)
EventTypeName (string)
State         (enum: NotPublished / InProgress / Published / PublishedFailed)
TimesSent     (int)
CreationTime  (DateTime)
Content       (string, JSON serialized event)
TransactionId (string)
```

### 4.3 Ordering Domain

```
CardType
─────────
Id   (int, PK)
Name (string)

Buyer ─────────────────────────────────────────
Id             (int, PK)
Name           (string)
Email          (string)
│
└─► PaymentMethod (1-to-many)
    ──────────────────────────
    Id         (int, PK)
    Alias      (string)
    CardNumber (string, last 4 digits)
    Expiration (string)
    CardHolder (string)
    CardTypeId (int, FK → CardType)
    BuyerId    (int, FK → Buyer)

Order ─────────────────────────────────────────────────────────
Id             (int, PK)
OrderDate      (DateTime)
BuyerId        (int, FK → Buyer)
OrderStatus    (enum: Submitted / AwaitingValidation / StockConfirmed / Paid / Shipped / Cancelled)
Description    (string)
IsDraft        (bool)
PaymentId      (int)
Address        (value object: Street, City, State, Country, ZipCode)
│
└─► OrderItem (1-to-many)
    ─────────────────────────
    Id              (int, PK)
    OrderId         (int, FK → Order)
    ProductId       (int)
    ProductName     (string)
    PictureUrl      (string)
    UnitPrice       (decimal)
    Discount        (decimal)
    Units           (int)

ClientRequest (idempotency log)
────────────────────────────────
Id     (Guid, PK)       ← RequestId from client
Name   (string)         ← Command type name
Time   (int)            ← Unix timestamp

IntegrationEventLog     ← same outbox schema as Catalog
```

### 4.4 Identity Domain

Standard ASP.NET Identity tables (`AspNetUsers`, `AspNetRoles`, `AspNetUserRoles`, `AspNetUserClaims`) plus Duende IdentityServer operational tables (clients, resources, persisted grants — kept in-memory for the reference implementation).

### 4.5 Webhooks Domain

```
WebhookSubscription
────────────────────────────
Id          (int, PK)
Type        (string)       ← event type name
Url         (string)       ← delivery endpoint
Token       (string)       ← HMAC signing secret
Date        (DateTime)
UserId      (string)       ← owner

WebhookData
────────────────────────────
Id           (int, PK)
Date         (DateTime)
DestUrl      (string)
ResponseCode (int)
RequestSent  (string, JSON)
Response     (string)
Payload      (string)
SenderId     (int, FK → WebhookSubscription)
```

### 4.6 Basket (Redis)

```
Key:   basket/{userId}
Value: JSON CustomerBasket
       {
         "buyerId": "...",
         "items": [
           {
             "id": "...",
             "productId": 1,
             "productName": "...",
             "unitPrice": 12.50,
             "oldUnitPrice": 0,
             "quantity": 2,
             "pictureUrl": "..."
           }
         ]
       }
```

---

## 5. Deployment Model

### 5.1 Local Development (Aspire)

.NET Aspire AppHost runs all services as local processes (not containers) while running infrastructure (PostgreSQL, Redis, RabbitMQ) as Docker containers. A single `dotnet run --project src/eShop.AppHost` command starts the entire system.

```
Developer Machine
├── dotnet process: Identity.API      :5243
├── dotnet process: Basket.API        :5222
├── dotnet process: Catalog.API       :5301
├── dotnet process: Ordering.API      :5102
├── dotnet process: OrderProcessor
├── dotnet process: PaymentProcessor
├── dotnet process: Webhooks.API      :5034
├── dotnet process: WebApp            :5173
├── dotnet process: WebhookClient     :5063
├── dotnet process: mobile-bff        :5209
│
├── Docker: postgres (ankane/pgvector) :5432
├── Docker: redis                      :6379
├── Docker: rabbitmq                   :5672 / :15672
│
└── Aspire Dashboard                   :18888
```

Service discovery is DNS-based; Aspire injects `https+http://basket-api` style connection strings that resolve to the locally running process.

### 5.2 Cloud Deployment (Azure Developer CLI)

The repository ships with `azure.yaml` and Bicep/azd configuration, enabling `azd up` one-command deployment to Azure:

```
Azure Subscription
├── Azure Container Apps Environment
│   ├── identity-api      (Container App)
│   ├── basket-api         (Container App)
│   ├── catalog-api        (Container App)
│   ├── ordering-api       (Container App)
│   ├── order-processor    (Container App, min replicas = 1)
│   ├── payment-processor  (Container App, min replicas = 1)
│   ├── webhooks-api       (Container App)
│   ├── webapp             (Container App, external ingress)
│   └── mobile-bff         (Container App, external ingress)
│
├── Azure Database for PostgreSQL Flexible Server
│   ├── catalogdb
│   ├── identitydb
│   ├── orderingdb
│   └── webhooksdb
│
├── Azure Cache for Redis
│
├── Azure Service Bus (replaces RabbitMQ in cloud)
│   └── Topic: eshop_event_bus
│
├── Azure Container Registry (image storage)
│
└── Azure Monitor / Application Insights (OTLP receiver)
```

### 5.3 Kubernetes (Optional)

The project structure is compatible with Helm/kubectl deployment. Each service is independently containerisable via multi-stage Dockerfiles. Key considerations:

- Secrets (connection strings, OAuth secrets) → Kubernetes Secrets / Azure Key Vault
- Service discovery → Kubernetes DNS (`basket-api.default.svc.cluster.local`)
- Ingress → NGINX / Azure Application Gateway in front of WebApp and mobile-bff
- Horizontal Pod Autoscaler → scale Catalog.API and WebApp based on CPU/request rate

### 5.4 Container Image Strategy

All services use AOT-compatible .NET builds (`PublishAot` flag available). Multi-stage Dockerfile pattern:

```
Stage 1 (sdk): restore → build → publish
Stage 2 (runtime): copy published output, set ENTRYPOINT
```

Images are pushed to Azure Container Registry and referenced by Container Apps.

---

## 6. Security

### 6.1 Authentication Architecture

eShop uses a **centralised authentication** model via Identity.API (Duende IdentityServer 4):

```
Client (WebApp/MAUI)
    │
    │  1. OIDC Authorization Code Flow (PKCE)
    ▼
Identity.API  (issues JWT Bearer tokens + ID tokens)
    │
    │  2. JWT Bearer token in Authorization header
    ▼
Backend APIs (validate token against Identity.API JWKS)
```

**Token lifetime:** Access tokens — 1 hour. Cookie session — 2 hours.

**Discovery:** All APIs resolve `jwks_uri` automatically from `Identity.Url/.well-known/openid-configuration`.

### 6.2 API Authorization

Every backend API is protected with JWT Bearer authentication:

```csharp
// Each service configures its own audience
services.AddAuthentication().AddJwtBearer(options => {
    options.Authority  = identityUrl;   // Identity.API
    options.Audience   = "basket";      // service-specific audience
    options.RequireHttpsMetadata = !isDevelopment;
});
```

Endpoints use standard `[Authorize]` attributes. The Catalog API has a mix of public (read) and protected (write/admin) endpoints.

### 6.3 Token Propagation

WebApp uses `HttpClientAuthorizationDelegatingHandler` to forward the authenticated user's access token to downstream APIs automatically. This avoids storing tokens in JavaScript and keeps the full token flow server-side (BFF security pattern).

### 6.4 MAUI / Mobile Security

ClientApp uses `IdentityModel.OidcClient` with the system browser for the OIDC flow (satisfies app store requirements). Tokens are stored in the platform secure store. All traffic routes through the YARP Mobile BFF, which acts as a token relay.

### 6.5 Webhook Security

- Each webhook subscription has a per-subscriber HMAC signing `Token`
- Delivery payloads are signed so the receiver can verify authenticity
- Subscription creation validates the grant URL before accepting

### 6.6 Secret Management

| Environment | Secrets managed by |
|-------------|-------------------|
| Local dev | Aspire secrets / user-secrets |
| Azure (azd) | Azure Key Vault, referenced by Container Apps |
| CI/CD | GitHub Actions encrypted secrets |

### 6.7 Network Security

- In production, only WebApp and mobile-bff have external ingress; all other services are internal-only
- Services communicate over private VNet / Container Apps internal networking
- RabbitMQ and PostgreSQL are not exposed outside the private network
- HTTPS enforced on all external endpoints (TLS termination at ingress)

---

## 7. High Availability & Resiliency

### 7.1 Resilience Patterns Summary

| Pattern | Where applied | Implementation |
|---------|--------------|----------------|
| Retry with backoff | All HTTP clients | `AddStandardResilienceHandler()` (Polly) |
| Circuit breaker | All HTTP clients | `AddStandardResilienceHandler()` (Polly) |
| Timeout | All HTTP clients | `AddStandardResilienceHandler()` (Polly) |
| Outbox (at-least-once delivery) | Catalog, Ordering | `IntegrationEventLogEF` |
| Idempotency (exactly-once semantics) | Ordering.API | `IdentifiedCommand<T>` + `ClientRequest` table |
| Execution strategy (DB transient faults) | All EF DbContexts | `CreateExecutionStrategy()` |
| Persistent messages | RabbitMQ | `DeliveryMode.Persistent` |
| Health checks | All services | `/health` (readiness) + `/alive` (liveness) |
| Graceful degradation | AI search | Falls back to keyword search if embedding unavailable |

### 7.2 HTTP Client Resilience

`AddStandardResilienceHandler()` adds a Polly pipeline to every `HttpClient`:

```
Request
  │
  ├─ Rate limiter          (protect downstream)
  ├─ Total timeout         (overall request limit)
  ├─ Retry (exponential backoff + jitter, max 3 attempts)
  ├─ Circuit breaker       (50% failure → open for 30s)
  └─ Attempt timeout       (per-attempt limit)
```

### 7.3 Outbox Pattern (Guaranteed Event Delivery)

The outbox prevents lost events when a service crashes between writing to its database and publishing to RabbitMQ:

```
Within a single DB transaction:
  1. Apply domain changes (e.g. update order status)
  2. Write IntegrationEventLogEntry (State = NotPublished)

After transaction commits:
  3. Retrieve NotPublished entries for this transaction
  4. Publish each entry to RabbitMQ
  5. Update entry State = Published (or PublishedFailed)
```

If the process crashes at step 4, a recovery path (or restart) re-publishes on startup. If RabbitMQ is unavailable, `PublishedFailed` entries can be retried.

### 7.4 Idempotent Order Creation

Client requests include a `RequestId` (UUID). `IdentifiedCommandHandler<T>` checks the `ClientRequests` table:
- If `RequestId` exists → return cached result, discard duplicate command
- If new → execute command, persist `RequestId` atomically

This makes order creation safe under network retries from the WebApp and ClientApp.

### 7.5 Database Resiliency

Entity Framework Core `CreateExecutionStrategy` wraps all critical operations (command handling, event publishing) in a retry loop that handles transient PostgreSQL errors (connection loss, timeout, deadlock):

```csharp
var strategy = _dbContext.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    using var tx = await _dbContext.BeginTransactionAsync();
    // ... do work ...
    await tx.CommitAsync();
    // publish events
});
```

### 7.6 Message Broker Resiliency

- **Persistent delivery:** All published messages have `DeliveryMode.Persistent`; RabbitMQ writes to disk before acknowledging
- **Consumer acknowledgement:** Messages are ack'd only after the handler completes successfully
- **Retry on publish:** Polly `ResiliencePipeline` retries RabbitMQ publish up to `RetryCount` (default: 10) times with backoff
- **Channel recovery:** RabbitMQ client reconnects automatically on connection drops

### 7.7 Health Checks

Each service exposes two health endpoints (wired by `eShop.ServiceDefaults`):

| Endpoint | Tag filter | Purpose | Used by |
|----------|-----------|---------|---------|
| `/alive` | `live` | Is the process alive? | Kubernetes liveness probe |
| `/health` | all | Are all dependencies ready? | Kubernetes readiness probe, Aspire |

Checks included: database connectivity, Redis connectivity, RabbitMQ connectivity.

### 7.8 Observability for Resiliency

Distributed tracing (OpenTelemetry) propagates `TraceId` across service boundaries, through RabbitMQ messages (via message headers), and into the Aspire Dashboard. This enables correlation of failures across the entire saga even when they span multiple services and async events.

All resilience events (retry, circuit-breaker state change) emit telemetry via Polly's built-in OpenTelemetry instrumentation, enabling dashboards that show retry rates and circuit-breaker trips in real time.

---

*This document was generated from the eShop source code as of April 2026 (branch: main, commit: b81ad95).*
