# Technical Documentation: OrderFulfillmentEJB

## 1. Purpose
The `OrderFulfillmentEJB` is a stateless session bean responsible for managing the transition of a customer order from a "Payment Confirmed" state to the "In Fulfillment" or "Backordered" state. It orchestrates the complex workflow involving inventory reservation, shipping calculations, warehouse communication via asynchronous messaging, and customer notifications.

## 2. Data Flow
The process follows a sequential pipeline:
1. **Input**: Receives an `orderId` (Long).
2. **Persistence**: Fetches the `Order` entity and associated `OrderLineItem` collection from the `ordersPU` persistence unit.
3. **Inventory Loop**: Iterates through line items, calling the `InventoryService` to reserve stock.
4. **Shipping Calculation**: If stock is available, it sends order details to the `ShippingService` to retrieve costs and tracking info.
5. **Integration (JMS)**: Sends a `WarehousePickRequest` to the `jms/OrderQueue` for physical picking.
6. **Notification**: Triggers the `NotificationService` based on whether the order was fully or partially fulfilled.
7. **Output**: Returns a `FulfillmentResult` object indicating success/failure and the final order status.

## 3. Business Logic
### 3.1 Pre-conditions
*   The order must exist in the database.
*   The order status must be exactly `PAYMENT_CONFIRMED`. Any other status results in an `InvalidOrderStateException`.

### 3.2 Fulfillment Logic
*   **Full Fulfillment**: All requested quantities are reserved; order moves to `IN_FULFILLMENT`.
*   **Partial Fulfillment**: If a requested quantity isn't available, the system attempts to reserve whatever is left (`availableQty`). The item is marked `PARTIALLY_FULFILLED`, and the remainder is added to a backorder list.
*   **Full Backorder**: If no items in the order can be fulfilled, the order status is updated to `FULLY_BACKORDERED`.

### 3.3 Warehouse Integration
*   **Priority Routing**: The JMS message is tagged with a priority property (`HIGH` for "EXPRESS" shipping, `NORMAL` otherwise).

## 4. Dependencies
| Dependency | Type | Role |
| :--- | :--- | :--- |
| `EntityManager (em)` | JPA | CRUD operations on Order and LineItem entities. |
| `InventoryServiceLocal` | EJB | Reserves and releases stock; manages reservation tokens. |
| `ShippingServiceLocal` | EJB | Calculates shipping costs, delivery dates, and tracking numbers. |
| `NotificationServiceLocal` | EJB | Sends emails/alerts to customers regarding fulfillment status. |
| `ConnectionFactory` / `Queue` | JMS | Asynchronous communication with the Warehouse Management System (WMS). |
| `SessionContext (ctx)` | Resource | Manages container-managed transaction rollbacks. |

## 5. Error Handling & Transaction Management
### 5.1 Transaction Strategy
*   **Type**: Container Managed Transactions (CMT).
*   **Attribute**: `REQUIRED`. The method starts a new transaction or joins an existing one.
*   **Atomicity**: If a fatal error occurs, `ctx.setRollbackOnly()` is called to ensure the database state remains consistent.

### 5.2 Exception Handling
*   **`OrderNotFoundException` / `InvalidOrderStateException`**: Business validation errors.
*   **`InventoryException`**: Triggers a full transaction rollback and returns a failure result.
*   **`ShippingException`**: Triggers a **manual compensation logic** (`releaseAllReservations`) before rolling back the transaction. This is critical because inventory reservations may happen in a separate service/database that doesn't participate in the current JTA transaction.

## 6. Modernization Recommendations

### 6.1 Architectural Shift: From EJB to Microservices
*   **Decompose into Services**: This EJB is a "God Service" orchestrating four different domains (Order, Inventory, Shipping, Notification). These should be split into independent microservices.
*   **Saga Pattern**: The current implementation uses a distributed transaction (JTA) and manual compensation (`releaseAllReservations`). In a distributed environment, implement the **Saga Pattern (Orchestration-based)** to manage the sequence of reserve $\rightarrow$ ship $\rightarrow$ notify, with defined compensating transactions for each step.

### 6.2 Technology Stack Updates
*   **Jakarta EE / Spring Boot**: Replace `@Stateless` EJBs with Spring Boot `@Service` components.
*   **Messaging**: Replace legacy JMS with **Apache Kafka** or **RabbitMQ** for better scalability and event-driven architecture (e.g., emitting an `OrderFulfillmentStarted` event).
*   **Persistence**: Transition from JPA `em.merge()` to Spring Data JPA repositories.

### 6.3 Logic Improvements
*   **Parallel Processing**: The current loop processes inventory items sequentially. Using `CompletableFuture` or parallel streams could reduce latency when calling external services.
*   **State Machine**: Replace the `if/else` status checks with a formal **State Machine** (e.g., Spring StateMachine) to manage `OrderStatus` transitions more robustly.
*   **DTOs**: The method returns `FulfillmentResult` but modifies the `Order` entity directly. Transition to a strict DTO pattern to decouple the API from the database schema.
