# System Design: Microservices Decomposition from Monolith

## Overview

When breaking down a monolithic system into microservices, the fundamental question isn't "how many services should we have?" but rather "what's the right way to divide responsibilities?" The answer depends on multiple factors: business domain structure, team organization, data dependencies, and technical constraints. This guide covers the key patterns, considerations, and when to apply each approach.

---

## Core Principles

Before exploring decomposition patterns, understand these foundational principles:

### 1. **Single Responsibility Principle (SRP)**
Each microservice should have one reason to change and should implement a small set of strongly related functions. This ensures cohesion and reduces the blast radius of changes.

### 2. **Loose Coupling, High Cohesion**
Services should be independent with minimal inter-service dependencies. High cohesion means related functionality stays together.

### 3. **Two-Pizza Team Rule**
Each microservice should be small enough to be owned and developed by a small team (typically 6-10 people). This promotes autonomy and ownership.

### 4. **Independent Deployability**
A key microservices benefit: each service should be deployable independently without requiring coordinated releases with other services.

### 5. **Data Ownership**
Each service should own its data and expose it only through well-defined APIs. Shared databases create tight coupling and are a major anti-pattern.

---

## Data-Driven Design Approach

Data-driven decomposition starts with understanding how data flows through your system. This is crucial because data boundaries often reveal natural service boundaries.

### Why Data Matters

- **Identifies natural boundaries**: Services that don't share data can scale independently
- **Reveals dependencies**: Cross-service data access indicates tight coupling
- **Enables decentralization**: Each service manages its own database (Database Per Service pattern)
- **Impacts consistency strategies**: Shared data requires complex distributed transaction handling (Saga pattern, CQRS, etc.)

### Database Per Service Pattern

This is the foundational pattern for microservices data management:

**Concept**: Each microservice has its own database (physical or logical separation).

**Benefits**:
- Schema changes in one service don't affect others
- Services can choose the best database technology for their needs (PostgreSQL for transactions, MongoDB for documents, Cassandra for high throughput)
- True independence and independent scaling
- Fault isolation—database failures in one service don't cascade

**Challenges**:
- Distributed transactions become complex (can't rely on ACID guarantees across databases)
- Cross-service queries require API composition or eventual consistency
- Data consistency becomes eventually consistent rather than immediate
- Requires event-driven architecture for data synchronization

**Implementation Options**:
1. **Shared database server, separate schemas**: Cost-effective but less isolated
2. **Shared database server, separate tables**: Even less isolation
3. **Completely separate database servers**: Maximum isolation but higher operational complexity

### Example: E-Commerce Platform

Imagine an e-commerce monolith with one database:

```
Monolith Database:
├── customers table
├── products table
├── orders table
├── inventory table
└── payments table
```

Data-driven decomposition identifies these natural boundaries:

```
Customer Service Database:
├── customers table
└── customer_profiles table

Product Service Database:
├── products table
└── product_reviews table

Order Service Database:
├── orders table
└── order_items table

Inventory Service Database:
├── inventory table
└── stock_reservations table

Payment Service Database:
├── payments table
└── payment_history table
```

Now, when Order Service needs to check inventory, it calls the Inventory Service API rather than accessing the database directly.

---

## Primary Decomposition Patterns

### Pattern 1: Decompose by Business Capability

**What it is**: Identify what the business does to generate value and create a service for each capability.

**When to use**:
- You have a clear understanding of business units
- Business capabilities are relatively stable (unlikely to change frequently)
- You have subject matter experts (SMEs) for each capability
- Your organization structure roughly aligns with business capabilities

**Advantages**:
- Stable architecture—business capabilities don't change as often as technical requirements
- Teams organized around delivering business value, not technical features
- Natural alignment with Conway's Law (team structure mirrors system architecture)
- Services are loosely coupled by design

**Disadvantages**:
- Requires deep business understanding to identify capabilities correctly
- Large services initially (less fine-grained decomposition)
- Misidentified capabilities are expensive to refactor later

**How to identify business capabilities**:
1. Ask: "What does this business need to do to generate value?"
2. Look at organizational structure—different departments often represent capabilities
3. Examine business processes and workflows
4. Create a business capability map showing all capabilities and their relationships

**Example: Insurance Company**

Business capabilities:
- **Sales**: Quote generation, policy issuance
- **Customer Management**: Profile management, preferences
- **Claims Processing**: Claim submission, validation, payment
- **Underwriting**: Risk assessment, policy approval
- **Billing**: Invoice generation, payment processing
- **Compliance**: Regulatory reporting, audit trails

Each becomes a microservice.

**Data considerations**:
- `Customer` data might be shared conceptually but owned by Customer Management Service
- Order Service (or Policy Service) references customers through API calls
- Claims Service maintains its own view of customer information for claims context

---

### Pattern 2: Decompose by Subdomain (Domain-Driven Design - DDD)

**What it is**: Use Domain-Driven Design's subdomain concept to identify service boundaries. Subdomains are parts of the business domain that have different characteristics and business value.

**DDD Subdomain Types**:
1. **Core Subdomains**: Key business differentiators, competitive advantages (e.g., "recommendation engine" for Netflix)
2. **Supporting Subdomains**: Important but not differentiators (e.g., "billing" for most SaaS)
3. **Generic Subdomains**: Standard business functions available off-the-shelf (e.g., "authentication", "logging")

**Key DDD Concepts**:
- **Bounded Context**: A clear boundary within which a domain model applies. Each bounded context is a microservice.
- **Ubiquitous Language**: Team-specific domain language used within a bounded context
- **Entity**: Domain object with identity (e.g., "Order")
- **Aggregate**: Collection of entities treated as a single unit (e.g., "Order" aggregate contains Order + OrderItems)

**When to use**:
- Your domain has well-understood business rules and complex interactions
- You're dealing with legacy systems where domain boundaries are already somewhat established
- Your team includes domain experts who can define bounded contexts clearly
- You need a more fine-grained decomposition than business capabilities provide

**Advantages**:
- Enables fine-grained service design through careful domain analysis
- Clearer boundaries around domain logic and data
- More naturally aligns with how domain experts think about the problem
- Supports multiple "contexts" for the same business concept (e.g., "Customer" in Marketing vs. "Customer" in Support has different attributes)

**Disadvantages**:
- Requires domain expertise and Event Storming workshops
- Can result in many small services (increased operational complexity)
- Identifying subdomains takes time and collaboration
- May not align perfectly with organizational structure

**How to identify subdomains**:

1. **Event Storming Workshop**: Bring together developers, business analysts, product owners, and domain experts
   - Identify domain events (things that happened)
   - Group related events into clusters
   - Identify commands that trigger events
   - Find aggregates that maintain consistency
   - Draw boundaries around cohesive groups = subdomains

2. **Domain Analysis**: Examine existing systems
   - What are the distinct areas of expertise?
   - What changes together?
   - What has different scaling or availability needs?

**Example: E-Commerce with Subdomains**

```
Core Subdomains (competitive advantages):
├── Recommendation Engine
└── Pricing Engine

Supporting Subdomains:
├── Inventory Management
├── Order Fulfillment
└── Customer Service

Generic Subdomains:
├── Authentication
├── Notifications
└── Payment Processing (often outsourced)
```

**Business Capability vs. Subdomain**

| Aspect | Business Capability | Subdomain |
|--------|-------------------|-----------|
| Focus | What the business does | How the business does it (domain logic) |
| Granularity | Coarser (higher level) | Finer (can decompose further) |
| Change driver | Business model changes | Domain knowledge, business rules |
| Scope | Usually stable | Can be refined over time |
| Use case | Clear organizational structure | Complex domains with multiple experts |

---

### Pattern 3: Decompose by Transactions

**What it is**: Group services based on business transactions that must complete together atomically.

**When to use**:
- Response time is critical (can't afford asynchronous processing)
- Your domain has well-defined transactions that shouldn't span many services
- Your system has high throughput requirements for specific transactions
- You want to minimize inter-service communication for critical paths

**Advantages**:
- Minimizes network calls for critical transactions
- Reduces latency for time-sensitive operations
- Reduces need for distributed transaction handling (Saga pattern)
- Clearer transaction boundaries

**Disadvantages**:
- Can result in large services with multiple concerns
- May create poorly designed services by forcing unrelated things together
- Can become another form of distributed monolith if overdone
- Makes scaling individual functions difficult

**How to apply**:
1. Map out your critical business transactions
2. Identify all services that must participate in each transaction
3. Group services involved in the same transactions
4. Accept that some services will be larger

**Example: Insurance Claims**

A claim request transaction involves:
1. Validate customer exists and policy is active (Customer Service)
2. Create claim record (Claims Service)
3. Assign to adjuster (Claims Management Service)
4. Update policy status (Policy Service)

If these transactions are critical and response-time sensitive, you might group Customer + Policy together and Claims + Claims Management together, rather than splitting them further.

**Important caveat**: Use this pattern cautiously. It can quickly lead to a distributed monolith if not balanced with other patterns.

---

### Pattern 4: Service Per Team

**What it is**: Align microservices with team ownership rather than business/domain boundaries.

**When to use**:
- You have a geographically distributed organization
- You want to maximize team autonomy and independence
- You have clear team boundaries already in place
- Deployment cadence and team coordination are more constrained than domain logic

**Advantages**:
- Teams can work independently with minimal coordination
- Clear ownership and accountability
- Different teams can use different technologies (behind stable APIs)
- Enables rapid parallel development
- Aligns with inverse Conway's Law (design organization to get desired system architecture)

**Disadvantages**:
- Doesn't necessarily align with domain boundaries (can lead to poor design)
- Large organizations might have many teams, resulting in too many services
- Difficult to coordinate when domain logic spans multiple teams
- May require extensive API versioning and compatibility management

**Example**:

An organization has:
- Frontend Team → Frontend Service (or multiple)
- Backend Team → Backend Service (or multiple)
- Data Team → Data Processing Service
- ML Team → ML/Recommendation Service
- DevOps Team → Infrastructure/Platform Services

Each team owns their service(s) independently.

**Conway's Law Connection**: This pattern deliberately applies inverse Conway's Law: you design your organization structure to achieve your desired architecture. If you want loosely coupled, independently deployable services, structure teams that way.

---

### Pattern 5: Strangler Fig Pattern

**What it is**: Incrementally replace parts of a monolith with microservices without a big-bang rewrite. New microservices "strangle" (gradually replace) the old system.

**When to use**:
- You have a large, active monolithic system you can't afford to rewrite
- You need to maintain business continuity during migration
- You want to minimize migration risk
- You can't afford extended freeze periods

**Advantages**:
- Low-risk migration—can validate each extracted service independently
- Enables rapid delivery of new capabilities during migration
- Business remains operational throughout
- Can pause or adjust strategy mid-migration

**Disadvantages**:
- Takes longer than big-bang rewrites
- Requires maintaining both old and new system in parallel
- Needs a facade/proxy layer to route traffic appropriately
- Can create temporary complexity

**How it works**:

1. **Transform**: Create a facade (proxy/API gateway) between clients and the monolith
2. **Coexist**: Route new traffic to microservices, old traffic still goes to monolith
3. **Eliminate**: Gradually replace monolith functions with microservices until monolith is empty and can be decommissioned

**Implementation steps**:

1. Identify a small, self-contained capability to extract first
2. Build a new microservice implementing that capability
3. Create anti-corruption layer (ACL) to translate between old and new models if needed
4. Implement facade to route requests to either old monolith or new service
5. Test both paths in parallel (new service alongside old)
6. Once verified, route all traffic through facade to new service
7. Remove the old implementation from monolith
8. Repeat for next capability

**Example**:

```
Initial state:
Clients → Monolith (handles orders, payments, shipping, customers)

After strangler fig extraction of payments:
Clients → Facade → 
  ├── Payment Service (new microservice)
  └── Monolith (orders, shipping, customers)
  
Facade determines: is this a payment request? → Route to Payment Service
Otherwise: → Route to Monolith

Later (extract shipping):
Clients → Facade → 
  ├── Payment Service (microservice)
  ├── Shipping Service (microservice)
  └── Monolith (orders, customers)

Eventually:
Clients → API Gateway → 
  ├── Order Service
  ├── Payment Service
  ├── Shipping Service
  └── Customer Service
  
(Monolith fully replaced and decommissioned)
```

---

### Pattern 6: Branch by Abstraction

**What it is**: Create an abstraction layer that allows old and new implementations to coexist, then gradually route traffic to the new implementation.

**When to use**:
- You want to decompose gradually while maintaining stable production code
- You need to do A/B testing of new implementations
- You want to minimize long-lived branches in source control
- You prefer trunk-based development

**Advantages**:
- Enables continuous integration while making large changes
- Allows testing new implementation against real production workloads
- Supports gradual rollout and rollback
- Reduces merge conflicts compared to long-lived feature branches

**Disadvantages**:
- Adds temporary code complexity (the abstraction layer)
- Requires feature flags/toggles infrastructure
- Both implementations must be maintained during transition
- Can be harder to reason about than simple replacements

**How it works**:

1. Create an adapter interface
2. Implement both old and new functionality behind the adapter
3. Use feature flags to determine which implementation to use
4. Test both implementations in production with controlled traffic
5. Gradually shift traffic to new implementation
6. Once fully migrated, remove old implementation and adapter

**Example**:

```java
// Abstraction layer
interface NotificationService {
    void sendNotification(String userId, String message);
}

// Old implementation (from monolith)
class LegacyNotificationService implements NotificationService {
    public void sendNotification(String userId, String message) {
        // Old monolith code
    }
}

// New implementation (microservice)
class NotificationMicroservice implements NotificationService {
    public void sendNotification(String userId, String message) {
        // Call to notification microservice
    }
}

// Adapter decides which to use
class NotificationAdapter implements NotificationService {
    public void sendNotification(String userId, String message) {
        if (featureFlags.isEnabled("use_notification_service")) {
            notificationService.sendNotification(userId, message);
        } else {
            legacyService.sendNotification(userId, message);
        }
    }
}
```

---

## Organizational Factors: Conway's Law

**Conway's Law**: "Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure."

### Implications for Microservices:

1. **Team Structure Shapes Architecture**: If you have three teams, you'll naturally end up with three large services, regardless of domain boundaries.

2. **Communication Overhead**: When teams must communicate to make a change, your services will have corresponding tight coupling.

3. **Distributed Teams → Distributed Architecture**: Geographically separated teams tend to create loosely coupled services naturally.

4. **Inverse Conway Maneuver**: Deliberately reshape your organization to achieve your desired architecture:
   - Want loosely coupled services? → Create small, autonomous teams
   - Want tight integration? → Co-locate teams
   - Want service independence? → Minimize dependencies between teams

### Practical Application:

```
Desired Architecture:
Payment Service ↔ Order Service ↔ Inventory Service
(loosely coupled, independent teams)

Organize teams accordingly:
Team A owns Payment Service
Team B owns Order Service
Team C owns Inventory Service
(with minimal cross-team dependencies)

NOT: All three teams reporting to one manager who enforces dependencies
```

---

## Decision Framework: Choosing Your Decomposition Strategy

### Step 1: Understand Your Constraints

Ask these questions:

1. **Team Structure**: How is your organization structured? Are there natural team boundaries?
2. **Business Stability**: How stable is your business domain? Will it change frequently?
3. **Expertise**: Do you have domain experts who understand the business deeply?
4. **Timeline**: Do you have time for careful planning, or do you need speed?
5. **Risk Tolerance**: Can you afford multi-month migrations, or do you need continuous delivery?
6. **Scale**: Are you scaling by throughput, by team, or by geographic distribution?
7. **Existing Knowledge**: Do you already know your system well (domain boundaries clear)?

### Step 2: Select Primary Pattern

| Pattern | Best For | Team Size | Migration Speed | Domain Clarity Needed |
|---------|----------|-----------|-----------------|----------------------|
| Business Capability | Clear business structure | Medium | Moderate | Moderate |
| Subdomain (DDD) | Complex domains | Any | Slow | High |
| Transaction | Performance-critical systems | Small | Moderate | Moderate |
| Service Per Team | Large, distributed org | Large | Fast | Low |
| Strangler Fig | Existing monoliths | Any | Slow | Low |
| Branch by Abstraction | Risk-averse deployments | Any | Moderate | Low |

### Step 3: Apply Data-Driven Constraints

1. **Data dependency analysis**: Which services must frequently access the same data?
   - If high dependency → Consider grouping or event-driven sync
   - If low dependency → Confirm good decomposition

2. **Transaction analysis**: What operations must complete atomically?
   - Single service? → Good decomposition
   - Cross-service? → Either group services or use Saga pattern

3. **Consistency requirements**: Can you tolerate eventual consistency?
   - No → Need tighter coupling or two-phase commit (expensive)
   - Yes → Enables true microservices decomposition

### Step 4: Validate with Events

Apply Event Storming:
1. Identify domain events
2. Group related events
3. Identify commands that cause events
4. Group by aggregate
5. Draw service boundaries around aggregates

This validates your chosen pattern against domain reality.

---

## When to Use Each Pattern: Real-World Examples

### Example 1: Startup E-Commerce Platform

**Context**: Early-stage startup, 15 people, all co-located, no complex domain yet

**Choose**: Business Capability

**Reasoning**:
- Small team size → clear communication, can coordinate easily
- Business model still evolving → need stability, not premature optimization
- Clear business functions: Product Catalog, Shopping Cart, Orders, Payments, Shipping
- Can evolve to DDD later if domain becomes complex

**Result**:
```
Services:
- Catalog Service (products, categories)
- Cart & Orders Service (together initially)
- Payment Service (payments, invoicing)
- Shipping Service (fulfillment, tracking)
```

### Example 2: Mature SaaS Platform (e.g., Project Management Tool)

**Context**: 200 engineers, multiple offices, domain well-understood

**Choose**: Combination of Subdomain + Service Per Team

**Reasoning**:
- Large org → must use Service Per Team to avoid chaos
- Mature domain → DDD analysis reveals core/supporting subdomains
- Different expertise areas: Core recommendation engine (small team), User management (shared), Notifications (generic)
- Multiple geographic teams → need independent deployability

**Result**:
```
Teams:
- Core Team → Recommendation & Priority Engine Service
- User Team → User Management & Authentication Service
- Integration Team → Notification & Messaging Service
- Platform Team → Infrastructure & Common Services
```

### Example 3: Legacy Monolith (e.g., 10-year-old Insurance System)

**Context**: Large monolith, 100+ engineers, business is operational/stable

**Choose**: Strangler Fig + Decompose by Business Capability

**Reasoning**:
- Too large to rewrite → need strangler fig for safety
- Business capabilities are well-known (underwriting, claims, billing, etc.)
- Can't afford downtime → need to run both systems in parallel
- Migration will take years → phased approach essential

**Result**:
```
Phase 1 (Year 1): Extract Claims Service
Phase 2 (Year 2): Extract Underwriting Service
Phase 3 (Year 3): Extract Billing Service
(Continue until monolith is empty)

Use strangler fig facade to route traffic.
```

### Example 4: High-Throughput Real-Time System (e.g., Stock Trading)

**Context**: Sub-millisecond response requirements, complex interdependencies

**Choose**: Decompose by Transaction + Strangler Fig

**Reasoning**:
- Performance critical → can't afford async patterns everywhere
- Transactions have clear boundaries (order placement, settlement, etc.)
- Legacy system needs gradual replacement → strangler fig
- Must minimize inter-service hops → co-locate frequently-called services

**Result**:
```
Group related operations:
- Quote Service (fast path, minimal dependencies)
- Order Service (handles order entry + settlement together due to tight coupling requirement)
- Risk Management Service (checks exposure, approves/rejects)
- Settlement Service (atomically updates portfolio)

Use strangler fig to migrate one transaction at a time.
```

---

## Critical Anti-Patterns to Avoid

### 1. Distributed Monolith

**What is it**: Services that appear independent but are actually tightly coupled. Changes to one require changes to others; they must be deployed together.

**Why it happens**:
- Services share a database
- Excessive synchronous dependencies
- Overlapping responsibilities
- Inappropriate transaction grouping

**How to avoid**:
- Use Database Per Service religiously
- Prefer asynchronous communication
- Enforce clear service boundaries in code reviews
- Use DDD bounded contexts

**Red flags**:
- "We need to deploy all services together"
- Multiple services modifying the same database table
- Chain of synchronous calls: Service A → B → C → D
- Cross-service schema dependencies

### 2. Nano-Services (Too Many Services)

**What is it**: Overly fine-grained decomposition where each microservice does almost nothing.

**Why it happens**:
- Misunderstanding microservices as "one class per service"
- Over-application of DDD without aggregates
- Pressure to have "many" services

**Impact**:
- Operational complexity explodes
- Distributed debugging becomes nightmare
- Performance suffers from network latency
- Deployment and monitoring overhead

**How to avoid**:
- Remember the "two-pizza team" rule
- Each service should handle meaningful business functionality
- Start coarser, refine over time if needed
- Use aggregates as service boundaries

### 3. Premature Extraction

**What is it**: Extracting services before you understand domain boundaries clearly.

**Why it happens**:
- Pressure to adopt microservices quickly
- Skipping Event Storming and domain analysis
- Following a template instead of understanding domain

**Impact**:
- Wrong boundaries require expensive refactoring
- Services don't align with team structure
- Data flows awkwardly across services
- Cross-cutting concerns end up everywhere

**How to avoid**:
- Invest time in domain understanding first
- Run Event Storming sessions with stakeholders
- Start with strangler fig on existing system
- Migrate one domain at a time deliberately

### 4. Shared Databases

**What is it**: Multiple services directly accessing and modifying the same database.

**Why it happens**:
- "It's easier than managing multiple databases"
- Temporary decision that becomes permanent
- Existing infrastructure requires it

**Impact**:
- True micros are impossible (tight coupling at database level)
- Schema changes block all teams
- Cross-service transactions require 2PC (expensive)
- Can't scale services independently

**How to avoid**:
- Enforce Database Per Service from day one
- Accept additional complexity as cost of microservices benefits
- Use event-driven architecture for data sync
- Create view databases/caches if read performance is needed

---

## Implementation Roadmap: From Monolith to Microservices

### Phase 1: Analysis (Weeks 1-4)

1. **Domain Analysis & Event Storming** (1-2 weeks)
   - Bring together product, business, and engineering
   - Identify domain events and commands
   - Draw bounded contexts and subdomains

2. **Current System Understanding** (1 week)
   - Map data flows
   - Identify data dependencies
   - Document transaction flows

3. **Team Alignment** (1 week)
   - Decide on decomposition strategy
   - Align team structure with desired architecture (inverse Conway)
   - Get stakeholder buy-in

### Phase 2: Foundation (Weeks 5-12)

1. **Infrastructure Setup**
   - Container orchestration (Kubernetes, Docker)
   - Service registry/discovery
   - API Gateway/reverse proxy
   - Centralized logging, metrics, tracing

2. **Development Patterns**
   - Establish API design standards
   - Implement inter-service communication patterns (REST, gRPC, messaging)
   - Create deployment pipeline
   - Establish monitoring/alerting

3. **First Service** (proof of concept)
   - Extract simplest, least coupled service
   - Validate infrastructure and patterns
   - Document learnings

### Phase 3: Gradual Migration (Ongoing)

1. **Apply chosen pattern** (Strangler Fig, Branch by Abstraction, etc.)
2. **Extract services incrementally**
   - Validate boundaries against domain knowledge
   - Ensure each service is independently deployable
   - Establish data ownership and sync patterns
   - Get experience before scaling up

3. **Evolve based on learnings**
   - Adjust service boundaries if needed
   - Optimize communication patterns
   - Refactor as domain understanding deepens

---

## Metrics for Good Decomposition

How do you know your decomposition is working?

### Service-Level Metrics

1. **Deployment Independence**: Can you deploy Service A without touching Service B?
   - Good: Yes, almost never need coordinated releases
   - Bad: "We need to release all services together"

2. **Team Autonomy**: Can teams develop independently?
   - Good: Teams rarely block each other
   - Bad: Team A waiting for Team B for API changes

3. **Data Coupling**: Do services need direct database access to each other?
   - Good: All data access through APIs/events
   - Bad: Services querying each other's databases

4. **Change Blast Radius**: When you modify Service A, how many other services need changes?
   - Good: 0-1 other services (rare)
   - Bad: Changes ripple through 5+ services

### System-Level Metrics

1. **Deployment Frequency**: How often can you deploy each service?
   - Good: Multiple times per day per service
   - Bad: All services deploy together once per week

2. **Time to Recovery**: If Service A fails, how long until it's healthy again?
   - Good: Seconds to minutes, doesn't affect other services
   - Bad: Service A failure cascades to Services B, C, D

3. **API Latency**: Inter-service response times
   - Good: <100ms for most calls
   - Bad: >500ms indicating chatty communication

4. **Test Independence**: Can you test Service A without Service B running?
   - Good: Yes, mock Service B easily
   - Bad: Tests require whole system running

---

## Conclusion: Key Takeaways

1. **There's no magic number**: Good microservices decomposition isn't about hitting a target number of services.

2. **Start with domain understanding**: Use Event Storming, DDD, and domain analysis to find natural boundaries.

3. **Data is your guide**: Database-level dependencies reveal service coupling and guide decomposition decisions.

4. **Align organization to architecture**: Use Conway's Law deliberately (Inverse Conway Maneuver) to structure teams for independence.

5. **Pick the right migration pattern**: Strangler Fig, Branch by Abstraction, and Greenfield all have their place depending on context.

6. **Validate with metrics**: Monitor deployment frequency, team autonomy, and change impact to confirm good decomposition.

7. **Avoid anti-patterns**: Distributed monoliths, nano-services, shared databases, and premature extraction trap teams.

8. **Iterate and evolve**: Your first decomposition won't be perfect—expect to refactor as you learn.