# Order Processing & Inventory Microservice
-> A production-style, cloud-native, event-driven microservice built with:
__________________________________________________________________
|    Technology	   |      Version      |         Purpose         |
__________________________________________________________________
|     Java	       |         17	       |        Language         |
__________________________________________________________________
|Spring-Boot       |	      3.2	       |        Framework        |
__________________________________________________________________
|Gradle	           |         8         |	     Build tool        |
__________________________________________________________________
|PostgreSQL	       |        15         |	     Persistence       | 
__________________________________________________________________
|Redpanda          |(Kafka-compatible) |	Latest Event streaming |
__________________________________________________________________
|Flyway	           |        9	         |   DB migrations         |
__________________________________________________________________
|springdoc-openapi |	     2.3	       |   Swagger UI            |
__________________________________________________________________
|Testcontainers	   |      1.19         |	Integration tests      |
__________________________________________________________________


---
Architecture
```
┌─────────────────────────────────────────────────────┐
│                   REST Clients                      │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP/JSON
┌──────────────────────▼──────────────────────────────┐
│   Controllers (OrderController, InventoryController)│
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│     Services (OrderService, InventoryService)       │
│  • business rules  • optimistic locking             │
│  • idempotency guards  • outbox writes              │
└────────────┬──────────────────────────┬─────────────┘
             │                          │
┌────────────▼──────────┐  ┌────────────▼────────────┐
│  JPA Repositories     │  │  OutboxRepository        │
│  (Order, Inventory)   │  │  (outbox\_events table)   │
└────────────┬──────────┘  └────────────┬─────────────┘
             │                          │
┌────────────▼──────────────────────────▼─────────────┐
│                    PostgreSQL                        │
└─────────────────────────────────────────────────────┘
                          │ (1-second poll)
             ┌────────────▼──────────────┐
             │   OutboxPublisher         │
             │   (@Scheduled poller)     │
             └────────────┬──────────────┘
                          │ Kafka produce
             ┌────────────▼──────────────┐
             │   Redpanda (Kafka)        │
             │   Topics:                 │
             │   • order.created         │
             │   • inventory.reserved    │
             │   • order.confirmed       │
             │   • order.cancelled       │
             └───────────────────────────┘
```
Transactional Outbox Pattern
Events are written into an `outbox\_events` table in the same DB transaction as the business state change. A background `@Scheduled` poller reads unpublished rows and forwards them to Kafka.
Why this matters: Without the outbox, a crash between "DB commit" and "Kafka publish" would silently drop events. The outbox eliminates this dual-write problem.
Tradeoff vs publish-after-commit:
✅ At-least-once delivery guaranteed, even across restarts
⚠️ Adds ~1 second of publish latency (configurable)
⚠️ Consumers must be idempotent (duplicate events are possible)
Concurrency & Idempotency
Optimistic locking (`@Version` on `InventoryItem`) prevents lost-update anomalies.
Pessimistic write lock (`SELECT FOR UPDATE`) serialises concurrent reservations on the same SKU row.
Idempotent endpoints: `POST /orders/{id}/reserve` and `POST /orders/{id}/confirm` return the current state unchanged if the transition has already occurred.
---
Quick Start
Prerequisites
Docker + Docker Compose v2
Run everything with one command
```bash
docker compose up --build
```
This starts:
PostgreSQL on port 5432
Redpanda (Kafka-compatible) on port 9092
Redpanda Console (UI) on port 8081
Order Service on port 8080 (waits for DB + Kafka health checks)
Flyway runs migrations automatically on startup.
---
API Reference
Swagger UI
Open in browser: http://localhost:8080/swagger-ui.html
Raw OpenAPI JSON: http://localhost:8080/api-docs
Actuator (Observability)
```bash
# Health check
curl http://localhost:8080/actuator/health

# Metrics
curl http://localhost:8080/actuator/metrics
```
---
Sample curl Commands
1. Create inventory (admin)
```bash
curl -s -X POST http://localhost:8080/inventory \\
  -H "Content-Type: application/json" \\
  -d '{"sku": "WIDGET-001", "availableQuantity": 100}' | jq
```
```bash
curl -s -X POST http://localhost:8080/inventory \\
  -H "Content-Type: application/json" \\
  -d '{"sku": "GADGET-XL", "availableQuantity": 50}' | jq
```
2. Get inventory for a SKU
```bash
curl -s http://localhost:8080/inventory/WIDGET-001 | jq
```
3. Create an order
```bash
ORDER=$(curl -s -X POST http://localhost:8080/orders \\
  -H "Content-Type: application/json" \\
  -d '{
    "lineItems": \[
      {"sku": "WIDGET-001", "quantity": 5},
      {"sku": "GADGET-XL",  "quantity": 2}
    ]
  }')
echo $ORDER | jq

# Extract ID for subsequent calls
ORDER\_ID=$(echo $ORDER | jq -r '.id')
echo "Order ID: $ORDER\_ID"
```
4. Get order details
```bash
curl -s http://localhost:8080/orders/$ORDER\_ID | jq
```
5. Reserve inventory
```bash
curl -s -X POST http://localhost:8080/orders/$ORDER\_ID/reserve | jq
```
6. Confirm order
```bash
curl -s -X POST http://localhost:8080/orders/$ORDER\_ID/confirm | jq
```
7. Cancel an order (try with a new order in CREATED or INVENTORY_RESERVED state)
```bash
# Create a new order to cancel
ORDER2=$(curl -s -X POST http://localhost:8080/orders \\
  -H "Content-Type: application/json" \\
  -d '{"lineItems": \[{"sku": "WIDGET-001", "quantity": 3}]}')
ORDER2\_ID=$(echo $ORDER2 | jq -r '.id')

# Reserve then cancel
curl -s -X POST http://localhost:8080/orders/$ORDER2\_ID/reserve | jq
curl -s -X POST http://localhost:8080/orders/$ORDER2\_ID/cancel | jq

# Inventory should be fully restored
curl -s http://localhost:8080/inventory/WIDGET-001 | jq
```
8. Test 409 – insufficient inventory
```bash
curl -s -X POST http://localhost:8080/orders \\
  -H "Content-Type: application/json" \\
  -d '{"lineItems": \[{"sku": "WIDGET-001", "quantity": 9999}]}' \\
  | jq '.id' -r \\
  | xargs -I{} curl -s -X POST http://localhost:8080/orders/{}/reserve | jq
```
9. Test 400 – validation error
```bash
curl -s -X POST http://localhost:8080/orders \\
  -H "Content-Type: application/json" \\
  -d '{"lineItems": \[{"sku": "WIDGET-001", "quantity": -1}]}' | jq
```
---
Inspecting Kafka Messages
Using Redpanda Console (GUI)
Open http://localhost:8081 → Topics → click any topic → Messages tab.
Using rpk CLI (inside the Redpanda container)
```bash
# Tail the order.created topic
docker exec order-redpanda rpk topic consume order.created --brokers=localhost:9092

# Tail all four topics simultaneously (in separate terminals)
docker exec order-redpanda rpk topic consume inventory.reserved --brokers=localhost:9092
docker exec order-redpanda rpk topic consume order.confirmed   --brokers=localhost:9092
docker exec order-redpanda rpk topic consume order.cancelled   --brokers=localhost:9092

# List all topics
docker exec order-redpanda rpk topic list --brokers=localhost:9092
```
---
Running Tests
Requires Docker running locally (Testcontainers spins up real containers):
```bash
./gradlew test
```
Test report: `build/reports/tests/test/index.html`
The integration test suite covers:
Happy path: create → reserve → confirm
Idempotent reserve (called twice, inventory deducted once)
Insufficient stock → 409
Cancel releases reserved inventory
Confirm without reserve → 409
404 for unknown order
---
Project Structure
```
order-service/
├── build.gradle                  # Gradle build (Java 17, Spring Boot 3.2)
├── settings.gradle
├── Dockerfile                    # Multi-stage build (builder + runtime)
├── docker-compose.yml            # Postgres + Redpanda + App
└── src/
    ├── main/
    │   ├── java/com/example/orderservice/
    │   │   ├── OrderServiceApplication.java      # Entry point
    │   │   ├── config/
    │   │   │   └── KafkaConfig.java              # Topic definitions
    │   │   ├── controller/
    │   │   │   ├── OrderController.java
    │   │   │   └── InventoryController.java
    │   │   ├── service/
    │   │   │   ├── OrderService.java             # Core business logic
    │   │   │   └── InventoryService.java
    │   │   ├── repository/
    │   │   │   ├── OrderRepository.java
    │   │   │   ├── InventoryRepository.java      # Pessimistic lock query
    │   │   │   └── OutboxRepository.java
    │   │   ├── domain/
    │   │   │   ├── model/
    │   │   │   │   ├── Order.java
    │   │   │   │   ├── OrderLineItem.java
    │   │   │   │   ├── OrderStatus.java
    │   │   │   │   ├── InventoryItem.java        # @Version optimistic locking
    │   │   │   │   └── OutboxEvent.java          # Transactional outbox record
    │   │   │   └── event/
    │   │   │       └── DomainEvents.java         # Java records for events
    │   │   ├── dto/
    │   │   │   ├── request/
    │   │   │   │   ├── CreateOrderRequest.java
    │   │   │   │   └── UpsertInventoryRequest.java
    │   │   │   └── response/
    │   │   │       └── Responses.java
    │   │   ├── kafka/
    │   │   │   └── OutboxPublisher.java          # @Scheduled outbox poller
    │   │   └── exception/
    │   │       ├── ResourceNotFoundException.java
    │   │       ├── OrderConflictException.java
    │   │       └── GlobalExceptionHandler.java
    │   └── resources/
    │       ├── application.yml
    │       └── db/migration/
    │           └── V1\_\_init\_schema.sql           # Flyway migration
    └── test/
        └── java/com/example/orderservice/
            └── OrderServiceIntegrationTest.java  # Testcontainers integration tests
```
