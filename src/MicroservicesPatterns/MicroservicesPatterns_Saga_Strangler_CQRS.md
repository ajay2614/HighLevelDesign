Monolithic Architecture

A single deployable application containing all components like UI, business logic, and database access.
All modules share the same codebase and single process.

Database relation:
Uses a single shared database for the whole application. All modules like User, Order, Product share same schema and tables. Transactions are ACID and fully consistent.

Pros:

Simple to develop, deploy, and debug
Strong consistency due to single DB
Easier for small applications

Cons:

Hard to scale specific modules
Any small change requires full redeploy
Database tightly coupled with app
Performance issues at large scale

Example:
E-commerce monolith app contains UserModule, ProductModule, and OrderModule using one MySQL database.

Microservices Architecture

System divided into multiple small services.
Each service handles a specific business domain and runs independently.
They communicate using REST APIs or message queues.

Database relation:
Each service has its own separate database (Database per Service pattern).
Services do not share schema or tables, ensuring loose coupling.
Each DB can use a suitable type (SQL or NoSQL).

Pros:

Independent deployment and scaling
Fault isolation between services
Polyglot persistence (different DBs for different services)

Cons:

Complex inter-service communication
Hard to maintain consistency across DBs
Cross-service joins require extra effort

Example:
UserService with PostgreSQL,
OrderService with MySQL,
PaymentService with MongoDB.

Saga Pattern

Used in microservices to handle distributed transactions.
Each local transaction updates its own DB and publishes an event to trigger the next step.
If a step fails, compensating transactions rollback the previous ones.

Database relation:
Each service maintains its own local database.
No global transaction or distributed lock is used.
Data consistency is achieved via events and compensating transactions, making it eventually consistent.

Flow Example:

OrderService creates order in order_db (status = pending)
Publishes event → PaymentService deducts amount in payment_db
Publishes event → InventoryService reserves items in inventory_db
If inventory fails, rollback triggers → PaymentService refunds → OrderService cancels order

Types:

Choreography: Each service listens to events and reacts
Orchestration: A central Saga orchestrator controls flow

Use case:
Maintaining data consistency across multiple microservices without using distributed transactions.

Strangler Fig Pattern

Used to migrate a monolithic system into microservices gradually.
New services are developed around the edges of the monolith, and old functionalities are replaced step by step until the monolith is completely removed.

Database relation:
Initially both monolith and new services may share or sync data.
New services later get their own databases as migration progresses.
Data migration and synchronization tools (like Debezium or Kafka) are often used.

Flow Example:

Initially: Monolith handles /orders using legacy_db
Add new OrderService → starts handling /orders/new using order_db
Gradually shift all order logic to new service
Finally retire legacy_db and monolith code

Use case:
When modernizing legacy monolithic systems safely and gradually without full rewrite.

CQRS (Command Query Responsibility Segregation)

Separates read and write operations into different models or services.
Commands are responsible for writes (create, update, delete).
Queries are responsible for reads (fetch, list, view).
Both use different data models and sometimes separate databases.

Database relation:
Write side uses normalized DB optimized for consistency.
Read side uses denormalized or replicated DB optimized for fast queries.
Data sync between them happens asynchronously using events.

Example:
In a banking system,

Command side updates the main account DB (PostgreSQL).
Query side reads from a denormalized or cached DB (Elasticsearch or Redis).
When balance updates, an event updates the read DB in background.

DTOs Example:
Command side receives input DTOs (CommandDTO) for write operations, e.g.
CreateAccountCommandDTO { userId, initialBalance }
Query side returns output DTOs (QueryDTO) for read operations, e.g.
AccountDetailsDTO { accountId, userName, balance, transactions }

Characteristics:

Commands and queries are strictly separated
Improves scalability and performance for read-heavy systems
Consistency between read and write models is eventual

Use case:
Used when read and write workloads differ or when system needs high read performance.

Summary Notes

Monolithic → one app, one DB, ACID consistent
Microservices → many services, DB per service, eventual consistency
Saga → distributed transaction handling with local DBs and compensations
Strangler Fig → safe incremental migration from monolith to microservices
CQRS → separates read and write models, different DBs and DTOs for input/output