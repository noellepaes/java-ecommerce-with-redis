# Ecommerce Modular Monolith

A **modular monolith** project built with **Java 17** (a single Maven/Spring Boot application), designed to evolve into a microservices architecture in the future.

**Repository highlight:** uses **Redis** for real-time collaborative recommendations with ephemeral view history, without persisting these associations in PostgreSQL.

## 🔴 Redis in the Project

The `recommendation/` module uses **Redis 7** as a view-tracking store. Every product click or view feeds a **bidirectional graph** in memory:

| Redis Key                             | Type | Content                             |
| ------------------------------------- | ---- | ----------------------------------- |
| `ecommerce:views:user:{customerId}`   | SET  | Product IDs viewed by the customer  |
| `ecommerce:views:product:{productId}` | SET  | Customer IDs who viewed the product |

**Why Redis?**

* **Ephemeral data** — browsing history with configurable TTL (`app.recommendation.customer-history-ttl`, default `30d`)
* **Fast reads and writes** for collaborative recommendations ("users who viewed X also viewed Y")
* **No JPA entity** — PostgreSQL remains responsible for transactional data while Redis handles behavioral signals

High-level flow:

1. `POST /api/recommendations/customers/{id}/views` records the view in both Redis SETs and refreshes the TTL
2. `GET /api/recommendations/customers/{id}` generates recommendations from the Redis graph
3. If there is insufficient behavioral data, the API fills the response with available catalog products from PostgreSQL

### Redis Insight — Key Visualization

Connect to Redis Insight at `localhost:6379` and filter by `ecommerce:views:*` to inspect the SETs in real time.

After starting the application in DEV mode, the seed process automatically generates sample views. The logs display customer and product IDs that can be used for testing.

<img width="1266" height="789" alt="Image" src="https://github.com/user-attachments/assets/5cfb12f4-a62a-4d5d-b71b-d1821cfb0457" />

> **To include the screenshot:** save it as `docs/images/redis-insight.png` (or update the path above).

## 🧱 Structure (Single Project, Package-Based Modules)

The application runs as a **single executable application**, while Bounded Contexts are organized into separate packages under `src/main/java`:

```text
src/main/java/com/ecommerce/
 ├── auth/
 ├── product/
 ├── customer/
 ├── order/
 ├── payment/
 ├── recommendation/   ← Redis (views + recommendations)
 └── shared/
```

### DDD Structure Within Each Module

Each context follows the structure below:

```text
module/
 ├── domain/
 │    ├── model/
 │    ├── service/
 │    ├── repository/
 │    └── exception/
 ├── application/
 │    ├── usecase/
 │    └── dto/
 ├── infrastructure/
 │    └── repository/
 └── presentation/
      ├── *Controller.java
      └── *Request.java
```

## 🗄️ Database

### Strategy: PostgreSQL with Separate Schemas

* A single PostgreSQL database with one schema per context:

  * `product_schema`
  * `customer_schema`
  * `order_schema`
  * `payment_schema`
  * `auth` (user tables)

### Flyway as the Single Source of Truth

Migrations are located under `src/main/resources/db/migration/v1/`:

* `V1__01_create_schemas.sql`
* `V2__create_customer_tables.sql`
* `V3__create_product_tables.sql`
* `V4__create_order_tables.sql`
* `V5__create_payment_tables.sql`
* `V6__create_auth_tables.sql`

## 🚀 Running the Project

### 1. Start PostgreSQL and Redis with Docker

```bash
docker-compose up -d
```

This starts:

* **PostgreSQL** at `localhost:5432`
* **Redis** at `localhost:6379`
* **Application** at `localhost:8080` (using the `docker` profile)

To start only the infrastructure (PostgreSQL + Redis) and run the application locally:

```bash
docker-compose up -d postgres redis
```

### 2. Build the Project

```bash
mvn clean install
```

### 3. Run the Application

```bash
mvn spring-boot:run
```

The application will be available at:

```text
http://localhost:8080
```

Swagger UI:

```text
http://localhost:8080/swagger-ui.html
```

## 📋 API Endpoints

### Recommendations (Redis)

* `POST /api/recommendations/customers/{customerId}/views` — Register a product view/click (stored in Redis)
* `GET /api/recommendations/customers/{customerId}` — Collaborative recommendations based on browsing history

### Product

* `POST /api/products` — Create product
* `GET /api/products/{id}` — Get product by ID
* `GET /api/products` — List products
* `PUT /api/products/{id}` — Update product
* `POST /api/products/{id}/decrease-stock` — Decrease stock

### Customer

* `POST /api/customers` — Create customer
* `GET /api/customers/{id}` — Get customer by ID
* `GET /api/customers` — List customers
* `PUT /api/customers/{id}` — Update customer
* `DELETE /api/customers/{id}` — Deactivate customer

### Order

* `POST /api/orders` — Create order
* `GET /api/orders/{id}` — Get order by ID
* `GET /api/orders/customer/{customerId}` — List customer orders
* `POST /api/orders/{id}/items` — Add item to order
* `POST /api/orders/{id}/pay` — Pay order
* `POST /api/orders/{id}/cancel` — Cancel order

### Payment

* `POST /api/payments` — Create payment
* `GET /api/payments/{id}` — Get payment by ID
* `GET /api/payments/order/{orderId}` — List payments for an order

## 🔄 Module Communication

### Current State (Modular Monolith)

* **`shared/` module:** Common classes (`BaseEntity`, `DomainEvent`, `BusinessException`)
* **UUID references:** Modules reference each other through UUIDs
* **Synchronous communication:** Services can call other services directly
* **Redis in `recommendation/`:** View history stored outside PostgreSQL

### Future State (Microservices)

* **Domain events:** `OrderCreatedEvent`, `PaymentApprovedEvent`, etc.
* **Message broker:** RabbitMQ/Kafka for asynchronous communication
* **API Gateway:** For synchronous communication between services

## 🎯 Implemented Business Rules

### Order (Aggregate Root)

* ✅ Can only transition from `PENDING` → `PAID`
* ✅ Can never transition from `CANCELLED` → `PAID`
* ✅ Paid orders cannot be canceled

### Product

* ✅ Stock validation before decrementing inventory
* ✅ Product must be active and in stock to be available

### Payment

* ✅ Only `PENDING` payments can be approved

### Recommendation (Redis)

* ✅ Views stored in bidirectional Redis SETs with TTL
* ✅ Recommendations prioritize customer co-viewing behavior
* ✅ Catalog fallback when Redis lacks sufficient signals

## 🛠️ Technologies

* Java 17
* Spring Boot 3.2.0
* Spring Data JPA
* Spring Data Redis
* Redis 7
* PostgreSQL 15
* Flyway (database migrations)
* Lombok
* Maven

## 📝 Next Steps

1. ✅ Modular monolith structure
2. ✅ Separate schemas
3. ✅ Flyway as the single source of truth
4. ✅ Redis-based recommendations
5. ⏳ Grafana + Prometheus (observability)
6. ⏳ Implement domain events
7. ⏳ Migrate to WebFlux (non-blocking)
8. ⏳ Extract services into microservices

## 📚 Documentation

* `docs/ARCHITECTURE.md` — Detailed architecture explanation
* `docs/MODULE_COMMUNICATION.md` — How modules communicate
* `docs/QUICK_START.md` — Quick testing guide
