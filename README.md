# Food Delivery Backend

[![Java](https://img.shields.io/badge/Java-21-orange.svg?style=flat-square&logo=openjdk)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.3-brightgreen.svg?style=flat-square&logo=springboot)](https://spring.io/projects/spring-boot)
[![Spring Security](https://img.shields.io/badge/Spring%20Security-6-green.svg?style=flat-square&logo=springsecurity)](https://spring.io/projects/spring-security)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-blue.svg?style=flat-square&logo=mysql)](https://www.mysql.com/)
[![Redis](https://img.shields.io/badge/Redis-7.0%20Alpine-red.svg?style=flat-square&logo=redis)](https://redis.io/)
[![Maven](https://img.shields.io/badge/Maven-3.9+-C71A36.svg?style=flat-square&logo=apachemaven)](https://maven.apache.org/)
[![JUnit 5](https://img.shields.io/badge/JUnit-5-25A162.svg?style=flat-square&logo=junit5)](https://junit.org/junit5/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg?style=flat-square&logo=docker)](https://www.docker.com/)
[![Build Status](https://img.shields.io/badge/Tests-156%20Passed-success.svg?style=flat-square)](#25-testing--quality-assurance)

A portfolio-grade, production-oriented **Food Delivery Backend** engineered as a robust **Modular Monolith** using **Java 21**, **Spring Boot 3**, **Spring Security (Stateless JWT)**, **Spring Data JPA / Hibernate**, **MySQL 8**, and **Redis 7**. The system provides enterprise-grade ordering, cart lifecycle management, single-restaurant checkout integrity with authoritative pricing snapshots, race-condition-free delivery claiming via atomic database conditional updates, and parameter-aware catalog caching with zero cache drift.

---

## Table of Contents

- [📊 Visual Flow Diagrams & Architecture (SYSTEM_FLOWS.md)](SYSTEM_FLOWS.md)
- [1. Project Overview](#1-project-overview)
- [2. What This System Solves](#2-what-this-system-solves)
- [3. Complete System Flow](#3-complete-system-flow)
- [4. User Roles & Capabilities](#4-user-roles--capabilities)
- [5. Customer Flow](#5-customer-flow)
- [6. Restaurant Owner Flow](#6-restaurant-owner-flow)
- [7. Delivery Partner Flow & Concurrency](#7-delivery-partner-flow--concurrency)
- [8. Order Lifecycle & State Machine](#8-order-lifecycle--state-machine)
- [9. Cart & Checkout Invariants](#9-cart--checkout-invariants)
- [10. Authentication & Security Model](#10-authentication--security-model)
- [11. System Architecture](#11-system-architecture)
- [12. Modular Monolith Design](#12-modular-monolith-design)
- [13. Database Architecture & ER Diagram](#13-database-architecture--er-diagram)
- [14. OrderItem Historical Snapshot Pattern](#14-orderitem-historical-snapshot-pattern)
- [15. Redis Caching Architecture](#15-redis-caching-architecture)
- [16. Cache Invalidation Matrix](#16-cache-invalidation-matrix)
- [17. API Reference](#17-api-reference)
- [18. Complete API Flow](#18-complete-api-flow)
- [19. Error Handling & Exception Architecture](#19-error-handling--exception-architecture)
- [20. Pagination, Sorting & Filtering](#20-pagination-sorting--filtering)
- [21. Validation Model](#21-validation-model)
- [22. Project Structure](#22-project-structure)
- [23. Technology Stack](#23-technology-stack)
- [24. Non-Functional Engineering Design](#24-non-functional-engineering-design)
- [25. Testing & Quality Assurance](#25-testing--quality-assurance)
- [26. Live Verification](#26-live-verification)
- [27. Getting Started & Setup](#27-getting-started--setup)
- [28. Business Rules & Domain Invariants](#28-business-rules--domain-invariants)
- [29. Architectural Decision Records (ADR)](#29-architectural-decision-records-adr)
- [30. Possible Future Enhancements](#30-possible-future-enhancements)

---

## 1. Project Overview

The backend orchestrates the complete multi-actor food ordering and fulfillment lifecycle across three distinct user personas:

- **Customers**: Register/login, browse active restaurants with dynamic search and pagination, inspect categorized menus, manage persistent carts (enforcing single-restaurant rules), execute transactional checkout with live price verification, cancel placed orders, and track real-time delivery status.
- **Restaurant Owners**: Register restaurants, update operational statuses, manage menu item catalogs (names, descriptions, categories, pricing, availability), view restaurant-specific incoming orders, transition orders through preparation stages (`ACCEPTED` $\rightarrow$ `PREPARING` $\rightarrow$ `READY_FOR_PICKUP`), and inspect driver fulfillment.
- **Delivery Partners**: Discover unassigned orders ready for pickup, atomically claim delivery orders with database-level concurrency protection, execute order pickup (`OUT_FOR_DELIVERY`), mark deliveries complete (`DELIVERED`), and inspect their assigned delivery histories.

```mermaid
flowchart TB
    Platform[Food Delivery Platform]

    Platform --> Customers[Customers]
    Platform --> Owners[Restaurant Owners]
    Platform --> Drivers[Delivery Partners]

    Customers --> C1[Browse Restaurants & Menus]
    Customers --> C2[Persistent Cart Management]
    Customers --> C3[Transactional Checkout]
    Customers --> C4[Order Tracking & Cancellation]

    Owners --> O1[Restaurant Profile & Operations]
    Owners --> O2[Menu Catalog & Availability]
    Owners --> O3[Order Acceptance & Kitchen Prep]
    Owners --> O4[Dispatch Tracking]

    Drivers --> D1[Discover Available Deliveries]
    Drivers --> D2[Atomic Concurrency-Safe Claim]
    Drivers --> D3[Pickup & Out for Delivery]
    Drivers --> D4[Delivery Fulfillment]
```

---

## 2. What This System Solves

Modern on-demand food delivery platforms require strict data consistency, financial integrity, and race-condition resistance:

1. **Price Integrity**: Menus change frequently. A customer cannot checkout with outdated cart prices. The system re-reads authoritative MySQL prices at the instant of checkout.
2. **Single-Restaurant Isolation**: Cart items from different restaurants cannot be mixed. The cart enforces single-restaurant integrity and blocks silent cart clearing.
3. **Historical Record Preservation**: Modifying or deleting menu items after an order is placed must never corrupt historical financial receipts or customer order records.
4. **Race-Condition-Free Dispatch**: When multiple delivery partners attempt to claim the same order simultaneously, database-level conditional updates guarantee that exactly one driver succeeds while all others receive `409 Conflict`.
5. **High-Throughput Read Caching**: Menu and restaurant browsing reads are cached in Redis with normalized, parameter-aware cache keys and automated cache eviction upon mutations.

```mermaid
flowchart LR
    Customer([Customer]) --> Browse[Browse Catalog]
    Browse --> Cart[Cart Isolation]
    Cart --> Checkout[Authoritative Checkout]
    Checkout --> Order[(MySQL Order Created)]
    Order --> Owner([Restaurant Owner])
    Owner --> Kitchen[Kitchen Prep]
    Kitchen --> Dispatch[Ready for Pickup]
    Dispatch --> Driver([Delivery Partner])
    Driver --> Deliver[Delivered & Tracked]
```

---

## 3. Complete System Flow

The complete end-to-end lifecycle from customer registration to final delivery fulfillment:

```mermaid
flowchart TD
    subgraph Customer Journey
        C_Auth[Register / Login] --> C_Browse[Browse Restaurants]
        C_Browse --> C_Menu[Inspect Restaurant Menu]
        C_Menu --> C_Cart[Add Items to Persistent Cart]
        C_Cart --> C_Checkout[Execute Checkout]
        C_Checkout --> C_OrderCreated[Order Created: PLACED]
    end

    subgraph Restaurant Kitchen Workflow
        C_OrderCreated --> O_Inspect[Owner Inspects Order]
        O_Inspect --> O_Accept[Owner Accepts: ACCEPTED]
        O_Accept --> O_Prep[Kitchen Prepares: PREPARING]
        O_Prep --> O_Ready[Mark Ready: READY_FOR_PICKUP]
    end

    subgraph Dispatch & Delivery Fulfillment
        O_Ready --> D_Discover[Driver Discovers in Available Queue]
        D_Discover --> D_Claim[Driver Executes Atomic Claim]
        D_Claim --> D_Pickup[Driver Marks Picked Up: OUT_FOR_DELIVERY]
        D_Pickup --> D_Deliver[Driver Completes Delivery: DELIVERED]
    end

    subgraph Order Tracking
        D_Pickup -.-> Track_Live[Customer & Owner Track Live Status]
        D_Deliver -.-> Track_Done[Customer Order Completed]
    end
```

---

## 4. User Roles & Capabilities

| Capability | Customer (`ROLE_CUSTOMER`) | Restaurant Owner (`ROLE_RESTAURANT_OWNER`) | Delivery Partner (`ROLE_DELIVERY_PARTNER`) | Public / Anonymous |
|---|:---:|:---:|:---:|:---:|
| User Registration & Login | Yes | Yes (Created by Admin/Seeding) | Yes (Created by Admin/Seeding) | Yes |
| Browse Restaurants & Menus | Yes | Yes | Yes | Yes |
| Manage Shopping Cart | Yes | No (`403 Forbidden`) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| Execute Order Checkout | Yes | No (`403 Forbidden`) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| View Customer Orders | Yes (Own orders only) | No (`403 Forbidden`) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| Cancel Order (`PLACED`) | Yes | No (`403 Forbidden`) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| Manage Restaurant Profile | No (`403 Forbidden`) | Yes (Own restaurant only) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| Manage Menu Items | No (`403 Forbidden`) | Yes (Own restaurant only) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| Progress Kitchen Status | No (`403 Forbidden`) | Yes (Own restaurant only) | No (`403 Forbidden`) | No (`401 Unauthorized`) |
| Discover Available Deliveries | No (`403 Forbidden`) | No (`403 Forbidden`) | Yes | No (`401 Unauthorized`) |
| Claim Delivery Order | No (`403 Forbidden`) | No (`403 Forbidden`) | Yes (Atomic conditional claim) | No (`401 Unauthorized`) |
| Pickup & Complete Delivery | No (`403 Forbidden`) | No (`403 Forbidden`) | Yes (Assigned orders only) | No (`401 Unauthorized`) |
| Track Delivery Status | Yes (Own orders only) | Yes (Own restaurant only) | Yes (Assigned orders only) | No (`401 Unauthorized`) |

---

## 5. Customer Flow

The customer flow guarantees server-side pricing integrity, single-restaurant cart constraints, and isolated order management.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant API as REST Controller
    participant Auth as Security / JWT
    participant Service as Order & Cart Service
    participant Redis as Redis 7 Cache
    participant MySQL as MySQL 8 (Source of Truth)

    Customer->>API: POST /api/auth/login
    API->>Auth: Validate credentials
    Auth-->>Customer: Return JWT Access Token

    Customer->>API: GET /api/restaurants?page=0&size=10
    API->>Service: getRestaurants()
    Service->>Redis: Check cache 'restaurants'
    alt Cache Hit
        Redis-->>Service: Return cached DTOs
    else Cache Miss
        Service->>MySQL: SELECT restaurants WHERE is_open = true
        MySQL-->>Service: Restaurant entities
        Service->>Redis: Store in cache with 5m TTL
    end
    Service-->>Customer: Return PagedResponse<RestaurantResponse>

    Customer->>API: POST /api/cart/items (menuItemId=101, qty=2)
    API->>Service: addItemToCart()
    Service->>MySQL: Verify MenuItem & Single-Restaurant Rule
    Service->>MySQL: Persist CartItem in MySQL
    Service-->>Customer: Return CartResponse

    Customer->>API: POST /api/orders/checkout
    API->>Service: checkout()
    Note over Service,MySQL: @Transactional boundary begins
    Service->>MySQL: Fetch Customer Cart
    Service->>MySQL: Re-read authoritative prices & availability from MenuItems
    Service->>MySQL: Compute Subtotal, Tax (5%), Delivery Fee (Rs 50), Total
    Service->>MySQL: Persist Order + OrderItems snapshots
    Service->>MySQL: Clear Customer Cart
    Note over Service,MySQL: Commit Transaction
    Service-->>Customer: Return OrderResponse (Status: PLACED)
```

---

## 6. Restaurant Owner Flow

Restaurant owners manage catalog operations and advance order states through the kitchen pipeline with strict resource ownership.

```mermaid
flowchart TD
    Owner([Restaurant Owner]) --> Login[Login via JWT]
    Login --> Ops{Owner Operations}

    Ops --> RestMgmt[Create / Update Restaurant Profile]
    Ops --> MenuMgmt[Add / Update / Delete Menu Items]
    Ops --> MenuToggle[Toggle Item Availability]
    Ops --> Orders[Fetch Incoming Restaurant Orders]

    RestMgmt --> EvictRest[Evict 'restaurants' & 'restaurantDetails' Cache]
    MenuMgmt --> EvictMenu[Evict 'restaurantMenus' & 'menuItems' Cache]
    MenuToggle --> EvictMenu

    Orders --> OrderAction{Order Action}
    OrderAction --> Accept[PATCH status: ACCEPTED]
    OrderAction --> Prep[PATCH status: PREPARING]
    OrderAction --> Ready[PATCH status: READY_FOR_PICKUP]
    OrderAction --> Cancel[PATCH status: CANCELLED]

    Ready --> Queue[(MySQL: Available for Delivery Claim)]
```

---

## 7. Delivery Partner Flow & Concurrency

When an order reaches `READY_FOR_PICKUP`, it appears in the unassigned delivery queue. Delivery partners can claim orders atomically.

### Concurrency Race Condition Resolution

```mermaid
sequenceDiagram
    autonumber
    actor DriverA as Delivery Partner A
    actor DriverB as Delivery Partner B
    participant API as DeliveryController
    participant Service as DeliveryService
    participant DB as MySQL 8

    Note over DB: Order #500: Status = READY_FOR_PICKUP, delivery_partner_id = NULL

    par Concurrent Claim Request
        DriverA->>API: POST /api/deliveries/500/claim
        API->>Service: claimDelivery(orderId=500, user=DriverA)
        Service->>DB: UPDATE orders SET delivery_partner_id = DriverA_ID, updated_at = NOW()<br/>WHERE id = 500 AND status = 'READY_FOR_PICKUP' AND delivery_partner_id IS NULL
    and
        DriverB->>API: POST /api/deliveries/500/claim
        API->>Service: claimDelivery(orderId=500, user=DriverB)
        Service->>DB: UPDATE orders SET delivery_partner_id = DriverB_ID, updated_at = NOW()<br/>WHERE id = 500 AND status = 'READY_FOR_PICKUP' AND delivery_partner_id IS NULL
    end

    alt Driver A Acquires Row Lock First
        DB-->>Service: 1 Row Affected (Driver A)
        Service-->>DriverA: 200 OK — DeliverySummaryResponse (Assigned to Driver A)
        DB-->>Service: 0 Rows Affected (Driver B)
        Service->>DB: SELECT * FROM orders WHERE id = 500
        DB-->>Service: Order #500 (Already assigned to Driver A)
        Service-->>DriverB: 409 Conflict — ORDER_ALREADY_CLAIMED
    end
```

### Delivery Lifecycle State Progression

```mermaid
flowchart LR
    Ready[READY_FOR_PICKUP] -->|POST /api/deliveries/{id}/claim| Claimed[Assigned to Driver]
    Claimed -->|PATCH /api/deliveries/{id}/pickup| Out[OUT_FOR_DELIVERY]
    Out -->|PATCH /api/deliveries/{id}/deliver| Done[DELIVERED]
```

---

## 8. Order Lifecycle & State Machine

The order lifecycle is governed by a centralized transition validator ([OrderStatusTransitionValidator.java](src/main/java/com/example/fooddelivery/enums/OrderStatusTransitionValidator.java)). Arbitrary client status mutations are strictly rejected.

```mermaid
stateDiagram-v2
    [*] --> PLACED : Customer Checkout

    PLACED --> ACCEPTED : Restaurant Owner Accepts
    PLACED --> CANCELLED : Customer or Owner Cancels

    ACCEPTED --> PREPARING : Restaurant Kitchen Begins Prep
    ACCEPTED --> CANCELLED : Restaurant Owner Cancels

    PREPARING --> READY_FOR_PICKUP : Restaurant Kitchen Completes Prep
    PREPARING --> CANCELLED : Restaurant Owner Cancels

    READY_FOR_PICKUP --> OUT_FOR_DELIVERY : Assigned Driver Picks Up
    OUT_FOR_DELIVERY --> DELIVERED : Assigned Driver Delivers

    DELIVERED --> [*]
    CANCELLED --> [*]
```

### State Machine Transition Rules Matrix

| Current Status | Target Status | Allowed Actor | Validation Rule |
|---|---|---|---|
| `PLACED` | `ACCEPTED` | Restaurant Owner | Owner must own the restaurant. |
| `PLACED` | `CANCELLED` | Customer / Owner | Customer can cancel only in `PLACED`. |
| `ACCEPTED` | `PREPARING` | Restaurant Owner | Owner advances kitchen workflow. |
| `ACCEPTED` | `CANCELLED` | Restaurant Owner | Owner cancellation before food prep. |
| `PREPARING` | `READY_FOR_PICKUP` | Restaurant Owner | Food is packaged and ready for dispatch. |
| `PREPARING` | `CANCELLED` | Restaurant Owner | Kitchen emergency cancellation. |
| `READY_FOR_PICKUP` | `OUT_FOR_DELIVERY` | Assigned Delivery Partner | Only the assigned driver can pickup. |
| `OUT_FOR_DELIVERY` | `DELIVERED` | Assigned Delivery Partner | Only the assigned driver can mark delivered. |
| `DELIVERED` | *Any* | *None* | Terminal state. No transitions allowed. |
| `CANCELLED` | *Any* | *None* | Terminal state. No transitions allowed. |

---

## 9. Cart & Checkout Invariants

```mermaid
flowchart TD
    Start([Customer Adds Item to Cart]) --> CheckCart{Does Cart Exist?}
    CheckCart -->|No| CreateCart[Create Persistent Cart for Customer]
    CheckCart -->|Yes| CheckRest{Matches Current Cart Restaurant?}

    CreateCart --> SetRest[Associate Restaurant]
    CheckRest -->|Yes| AddItem[Add / Increment Cart Item]
    CheckRest -->|No| Conflict[409 Conflict: CART_RESTAURANT_CONFLICT]

    AddItem --> ViewCart[GET /api/cart: Displays Subtotal & Item Breakdown]
    ViewCart --> Checkout[POST /api/orders/checkout]

    subgraph Transactional Checkout Boundary
        Checkout --> ValidateOpen{Restaurant Open?}
        ValidateOpen -->|No| ErrClosed[400 Bad Request: Restaurant Closed]
        ValidateOpen -->|Yes| FetchLivePrice[Re-read Live MenuItem Prices from DB]
        FetchLivePrice --> ValidateAvail{Items Available?}
        ValidateAvail -->|No| ErrAvail[400 Bad Request: Item Unavailable]
        ValidateAvail -->|Yes| CalcMath[Calculate Live Subtotal, Tax 5%, Fee Rs 50, Total]
        CalcMath --> SaveOrder[Create Order + Immutable OrderItem Snapshots]
        SaveOrder --> ClearCart[Clear Cart Items & Reset Restaurant]
    end
```

### Key Domain Invariants:
1. **Server-Side Pricing**: The client cannot submit subtotal, tax, fees, or total. All calculations occur server-side.
2. **Authoritative Price Snapshot**: Cart item prices are snapshot guides; checkout **re-reads live database prices** from `menu_items` table.
3. **Single-Restaurant Constraint**: A cart cannot contain items from multiple restaurants simultaneously.
4. **No Silent Cart Clears**: Adding an item from another restaurant returns `409 Conflict` rather than silently clearing the cart.

---

## 10. Authentication & Security Model

The system implements a stateless **Spring Security 6** architecture using **JWT (HMAC-SHA256)** and **BCrypt** password encryption.

```mermaid
flowchart TD
    Request[HTTP Request] --> JwtFilter[JwtAuthenticationFilter]
    JwtFilter --> HasHeader{Authorization: Bearer JWT?}

    HasHeader -->|No / Invalid| Anon[Anonymous Authentication]
    HasHeader -->|Valid| Extract[Extract Subject & Roles]
    Extract --> Context[Populate SecurityContext with CustomUserDetails]

    Anon --> SecRule{Endpoint Rule}
    Context --> SecRule

    SecRule -->|Public Route| AllowPublic[Permit Request]
    SecRule -->|Protected & Missing Auth| Ret401[401 Unauthorized: UNAUTHORIZED]
    SecRule -->|Protected & Role Match| CheckOwnership{Service Ownership Check}
    SecRule -->|Protected & Role Mismatch| Ret403[403 Forbidden: FORBIDDEN]

    CheckOwnership -->|Owner / Resource Valid| Execute[Execute Service & Commit MySQL]
    CheckOwnership -->|Resource Mismatch| Ret403
```

---

## 11. System Architecture

```mermaid
flowchart TB
    subgraph Client Layer
        Web[Web Browser / React / Next.js]
        Mobile[Mobile Application / iOS / Android]
        Postman[Postman / Integration Test Scripts]
    end

    subgraph Security & Ingress Layer
        Security[Spring Security 6 Filter Chain]
        JWT[JwtAuthenticationFilter + BCrypt]
    end

    subgraph Application Layer - Modular Monolith
        AuthController[AuthController]
        RestaurantController[RestaurantController]
        MenuItemController[MenuItemController]
        CartController[CartController]
        OrderController[OrderController & OwnerOrderController]
        DeliveryController[DeliveryController]

        AuthService[AuthService]
        RestaurantService[RestaurantService]
        MenuItemService[MenuItemService]
        CartService[CartService]
        OrderService[OrderService]
        DeliveryService[DeliveryService]
    end

    subgraph Caching Layer
        RedisCacheManager[Spring Cache / RedisCacheManager]
        Redis[('Redis 7 (Alpine)<br/>Catalog Caching Layer')]
    end

    subgraph Persistence Layer
        JPA[Spring Data JPA / Hibernate 6]
        Hikari[HikariCP Connection Pool]
        MySQL[('MySQL 8.0 (InnoDB)<br/>Authoritative Source of Truth')]
    end

    Web --> Security
    Mobile --> Security
    Postman --> Security

    Security --> JWT
    JWT --> AuthController
    JWT --> RestaurantController
    JWT --> MenuItemController
    JWT --> CartController
    JWT --> OrderController
    JWT --> DeliveryController

    AuthController --> AuthService
    RestaurantController --> RestaurantService
    MenuItemController --> MenuItemService
    CartController --> CartService
    OrderController --> OrderService
    DeliveryController --> DeliveryService

    RestaurantService -. Cached Reads .-> RedisCacheManager
    MenuItemService -. Cached Reads .-> RedisCacheManager
    RedisCacheManager <--> Redis

    AuthService --> JPA
    RestaurantService --> JPA
    MenuItemService --> JPA
    CartService --> JPA
    OrderService --> JPA
    DeliveryService --> JPA

    JPA --> Hikari --> MySQL
```

---

## 12. Modular Monolith Design

The application is structured as a **Modular Monolith**. Each domain package encapsulates its own Controllers, Services, DTOs, Mappers, and Repositories while sharing the transactional MySQL database.

```mermaid
flowchart TB
    App[Food Delivery Application Core]

    subgraph Domains
        AuthMod[Auth & Security Module]
        RestMod[Restaurant Management Module]
        MenuMod[Menu Catalog Module]
        CartMod[Cart Management Module]
        OrderMod[Order & Checkout Module]
        DelivMod[Delivery & Dispatch Module]
    end

    subgraph Cross-Cutting Infrastructure
        CacheInfra[Redis Caching & Invalidation Infrastructure]
        ExInfra[Global Exception & Error Handling Infrastructure]
        SecInfra[JWT & Security Context Infrastructure]
    end

    App --> AuthMod
    App --> RestMod
    App --> MenuMod
    App --> CartMod
    App --> OrderMod
    App --> DelivMod

    AuthMod --> SecInfra
    RestMod --> CacheInfra
    MenuMod --> CacheInfra
    CartMod --> ExInfra
    OrderMod --> ExInfra
    DelivMod --> ExInfra
```

---

## 13. Database Architecture & ER Diagram

```mermaid
erDiagram
    USERS {
        bigint id PK
        varchar name
        varchar email UK
        varchar password
        varchar phone
        enum role "CUSTOMER, RESTAURANT_OWNER, DELIVERY_PARTNER"
        timestamp created_at
        timestamp updated_at
    }

    RESTAURANTS {
        bigint id PK
        varchar name
        varchar description
        varchar address
        varchar phone
        boolean is_open
        bigint owner_id FK
        timestamp created_at
        timestamp updated_at
    }

    MENU_ITEMS {
        bigint id PK
        bigint restaurant_id FK
        varchar name
        varchar description
        decimal price
        varchar category
        varchar image_url
        boolean is_available
        timestamp created_at
        timestamp updated_at
    }

    CARTS {
        bigint id PK
        bigint customer_id FK "UK"
        bigint restaurant_id FK "Nullable"
        timestamp created_at
        timestamp updated_at
    }

    CART_ITEMS {
        bigint id PK
        bigint cart_id FK
        bigint menu_item_id FK
        int quantity
        decimal price
        timestamp created_at
        timestamp updated_at
    }

    ORDERS {
        bigint id PK
        varchar order_number UK
        bigint customer_id FK
        bigint restaurant_id FK
        bigint delivery_partner_id FK "Nullable"
        enum status "PLACED, ACCEPTED, PREPARING, READY_FOR_PICKUP, OUT_FOR_DELIVERY, DELIVERED, CANCELLED"
        varchar delivery_address
        varchar special_instructions
        decimal subtotal_amount
        decimal tax_amount
        decimal delivery_fee
        decimal total_amount
        varchar idempotency_key UK "Nullable"
        timestamp created_at
        timestamp updated_at
    }

    ORDER_ITEMS {
        bigint id PK
        bigint order_id FK
        bigint menu_item_id FK "Nullable (ON DELETE SET NULL)"
        varchar item_name
        decimal price
        int quantity
        decimal subtotal
        timestamp created_at
        timestamp updated_at
    }

    USERS ||--o{ RESTAURANTS : "owns"
    USERS ||--o| CARTS : "owns"
    USERS ||--o{ ORDERS : "places (as Customer)"
    USERS ||--o{ ORDERS : "delivers (as Delivery Partner)"
    RESTAURANTS ||--o{ MENU_ITEMS : "contains"
    RESTAURANTS ||--o{ CARTS : "referenced by"
    RESTAURANTS ||--o{ ORDERS : "receives"
    CARTS ||--o{ CART_ITEMS : "contains"
    MENU_ITEMS ||--o{ CART_ITEMS : "selected in"
    ORDERS ||--o{ ORDER_ITEMS : "contains"
    MENU_ITEMS ||--o{ ORDER_ITEMS : "referenced by"
```

---

## 14. OrderItem Historical Snapshot Pattern

When an order is created, the system decouples future menu changes from past order records.

```mermaid
flowchart LR
    subgraph Live Catalog
        MenuItem["MenuItem (ID: 42)<br/>Name: 'Paneer Butter Masala'<br/>Price: Rs 300.00"]
    end

    subgraph Checkout Snapshot
        Checkout["Checkout Execution<br/>Quantity: 2"]
        OrderItem["OrderItem<br/>menu_item_id: 42<br/>item_name: 'Paneer Butter Masala'<br/>price: Rs 300.00<br/>subtotal: Rs 600.00"]
    end

    subgraph Future Mutation
        Update["Owner updates price to Rs 350.00<br/>or Deletes MenuItem"]
    end

    MenuItem --> Checkout
    Checkout --> OrderItem
    Update -.-> MenuItem
    Update -. Does NOT affect .-> OrderItem
```

---

## 15. Redis Caching Architecture

Redis 7 is utilized strictly as a high-performance **read-heavy catalog cache**. Transactional data (users, carts, orders, delivery claims) resides exclusively in MySQL.

```mermaid
flowchart TD
    ClientReq[Client Catalog Request] --> CacheCheck{Redis Cache Lookup}

    CacheCheck -->|Cache HIT| ReturnCached[Return Cached DTO JSON]
    CacheCheck -->|Cache MISS| QueryMySQL[Query MySQL Source of Truth]

    QueryMySQL --> PopulateCache[Serialize DTO to Redis with TTL]
    PopulateCache --> ReturnFresh[Return Fresh DTO to Client]

    subgraph Cache Fault-Tolerance
        RedisDown[Redis Outage / Connection Error] -.-> Fallback[CacheErrorHandler Intercepts & Falls Back to MySQL]
    end
```

### Cache Configuration & Key Normalization

| Cache Name | Entity / Response Cached | TTL | Normalized Key Structure | Example Key |
|---|---|---|---|---|
| `restaurants` | `PagedResponse<RestaurantResponse>` | 5 min | `list:s={search}:o={isOpen}:p={page}:sz={size}:sort={sort}` | `list:s=pizza:o=true:p=0:sz=10:sort=id: ASC` |
| `restaurantDetails` | `RestaurantResponse` | 10 min | `#id` | `1` |
| `restaurantMenus` | `PagedResponse<MenuItemResponse>` | 5 min | `menu:r={restId}:c={cat}:a={avail}:s={search}:p={p}:sz={sz}:sort={sort}` | `menu:r=1:c=main:a=true:s=:p=0:sz=20:sort=id: ASC` |
| `menuItems` | `MenuItemResponse` | 10 min | `#id` | `101` |

---

## 16. Cache Invalidation Matrix

To prevent stale data, all catalog mutations automatically evict affected cache entries via `@CacheEvict`.

```mermaid
flowchart LR
    Mutation[Catalog Mutation: Create / Update / Delete / Toggle Status] --> MySQL[(MySQL Write Committed)]
    MySQL --> Evict[@CacheEvict Triggered]
    Evict --> EvictDetails[Evict Specific Key #id]
    Evict --> EvictLists[Evict All List Cache Entries]
```

| Operation | Triggering Endpoint | Evicted Cache Names | Invalidation Mode |
|---|---|---|---|
| Create Restaurant | `POST /api/restaurants` | `restaurants` | All entries (`allEntries = true`) |
| Update Restaurant | `PUT /api/restaurants/{id}` | `restaurants`, `restaurantDetails` | `#id` and all list entries |
| Toggle Restaurant Status | `PATCH /api/restaurants/{id}/status` | `restaurants`, `restaurantDetails`, `restaurantMenus` | `#id` and all list entries |
| Delete Restaurant | `DELETE /api/restaurants/{id}` | `restaurants`, `restaurantDetails`, `restaurantMenus` | `#id` and all list entries |
| Create Menu Item | `POST /api/restaurants/{id}/menu` | `restaurantMenus` | All entries (`allEntries = true`) |
| Update Menu Item | `PUT /api/menu-items/{id}` | `menuItems`, `restaurantMenus` | `#id` and all list entries |
| Toggle Item Availability | `PATCH /api/menu-items/{id}/status` | `menuItems`, `restaurantMenus` | `#id` and all list entries |
| Delete Menu Item | `DELETE /api/menu-items/{id}` | `menuItems`, `restaurantMenus` | `#id` and all list entries |

---

## 17. API Reference

### Authentication API (`/api/auth`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | Public | Register a new customer account |
| `POST` | `/api/auth/login` | Public | Authenticate and receive JWT access token |

### Restaurant Management API (`/api/restaurants`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/restaurants` | Public | List open/filtered restaurants (paginated, cached) |
| `GET` | `/api/restaurants/{id}` | Public | Get restaurant details by ID (cached) |
| `POST` | `/api/restaurants` | `RESTAURANT_OWNER` | Register a new restaurant |
| `PUT` | `/api/restaurants/{id}` | `RESTAURANT_OWNER` | Update restaurant details (ownership verified) |
| `PATCH` | `/api/restaurants/{id}/status` | `RESTAURANT_OWNER` | Toggle restaurant open/closed status |
| `DELETE` | `/api/restaurants/{id}` | `RESTAURANT_OWNER` | Delete restaurant |

### Menu Management API (`/api/restaurants/{id}/menu` & `/api/menu-items`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/restaurants/{id}/menu` | Public | Get restaurant menu items (paginated, filtered, cached) |
| `GET` | `/api/menu-items/{id}` | Public | Get menu item detail by ID (cached) |
| `POST` | `/api/restaurants/{id}/menu` | `RESTAURANT_OWNER` | Add a new menu item to restaurant |
| `PUT` | `/api/menu-items/{id}` | `RESTAURANT_OWNER` | Update menu item details |
| `PATCH` | `/api/menu-items/{id}/status` | `RESTAURANT_OWNER` | Toggle menu item availability |
| `DELETE` | `/api/menu-items/{id}` | `RESTAURANT_OWNER` | Delete menu item |

### Cart Management API (`/api/cart`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/cart` | `CUSTOMER` | Retrieve authenticated customer's cart |
| `POST` | `/api/cart/items` | `CUSTOMER` | Add item or increment quantity |
| `PUT` | `/api/cart/items/{itemId}` | `CUSTOMER` | Update cart item quantity (0 removes item) |
| `DELETE` | `/api/cart/items/{itemId}` | `CUSTOMER` | Remove single item from cart |
| `DELETE` | `/api/cart` | `CUSTOMER` | Clear all items and reset restaurant association |

### Order & Checkout API (`/api/orders`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/orders/checkout` | `CUSTOMER` | Execute transactional checkout from cart |
| `GET` | `/api/orders/my` | `CUSTOMER` | List authenticated customer's order history |
| `GET` | `/api/orders/{id}` | `CUSTOMER` | Get customer order details by ID |
| `PATCH` | `/api/orders/{id}/cancel` | `CUSTOMER` | Cancel order (only if `PLACED`) |
| `GET` | `/api/orders/{id}/delivery` | `CUSTOMER` | Track delivery status for customer order |

### Owner Order Management API (`/api/owner/orders`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/owner/orders` | `RESTAURANT_OWNER` | List orders for owner's restaurant (filter by status) |
| `PATCH` | `/api/owner/orders/{id}/status` | `RESTAURANT_OWNER` | Update status (`ACCEPTED`, `PREPARING`, `READY_FOR_PICKUP`, `CANCELLED`) |
| `GET` | `/api/owner/orders/{id}/delivery` | `RESTAURANT_OWNER` | Track delivery status for owner's restaurant order |

### Delivery & Dispatch API (`/api/deliveries` & `/api/delivery/orders`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/deliveries/available` | `DELIVERY_PARTNER` | List unassigned orders in `READY_FOR_PICKUP` status |
| `POST` | `/api/deliveries/{id}/claim` | `DELIVERY_PARTNER` | Atomically claim order for authenticated driver |
| `GET` | `/api/deliveries/my` | `DELIVERY_PARTNER` | List orders assigned to authenticated driver |
| `PATCH` | `/api/deliveries/{id}/pickup` | `DELIVERY_PARTNER` | Transition order to `OUT_FOR_DELIVERY` |
| `PATCH` | `/api/deliveries/{id}/deliver` | `DELIVERY_PARTNER` | Transition order to `DELIVERED` |
| `GET` | `/api/delivery/orders/available` | `DELIVERY_PARTNER` | Phase 6 backward-compatible available orders |
| `POST` | `/api/delivery/orders/{id}/claim` | `DELIVERY_PARTNER` | Phase 6 backward-compatible order claim |
| `GET` | `/api/delivery/orders/my` | `DELIVERY_PARTNER` | Phase 6 backward-compatible my deliveries |
| `PATCH` | `/api/delivery/orders/{id}/status` | `DELIVERY_PARTNER` | Phase 6 backward-compatible status transition |

### System & Health API (`/api/health` & `/actuator`)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/health` | Public | System health status & timestamp |
| `GET` | `/actuator/health` | Public | Spring Boot Actuator liveness/readiness probe |
| `GET` | `/actuator/info` | Public | Application build information |

---

## 18. Complete API Flow

```mermaid
flowchart LR
    Client[Client App] --> Ingress[Spring Security Filter Chain]

    Ingress --> AuthAPI["/api/auth/*"]
    Ingress --> RestAPI["/api/restaurants/*"]
    Ingress --> MenuAPI["/api/menu-items/*"]
    Ingress --> CartAPI["/api/cart/*"]
    Ingress --> OrderAPI["/api/orders/*"]
    Ingress --> OwnerAPI["/api/owner/orders/*"]
    Ingress --> DelivAPI["/api/deliveries/*"]

    AuthAPI --> S_Auth[AuthService]
    RestAPI --> S_Rest[RestaurantService]
    MenuAPI --> S_Menu[MenuItemService]
    CartAPI --> S_Cart[CartService]
    OrderAPI --> S_Order[OrderService]
    OwnerAPI --> S_Order
    DelivAPI --> S_Deliv[DeliveryService]

    S_Rest -. Reads .-> Redis[(Redis 7)]
    S_Menu -. Reads .-> Redis

    S_Auth --> MySQL[(MySQL 8)]
    S_Rest --> MySQL
    S_Menu --> MySQL
    S_Cart --> MySQL
    S_Order --> MySQL
    S_Deliv --> MySQL
```

---

## 19. Error Handling & Exception Architecture

All exceptions are intercepted by [GlobalExceptionHandler.java](src/main/java/com/example/fooddelivery/exception/GlobalExceptionHandler.java) and converted into structured `ErrorResponse` DTOs.

```mermaid
flowchart LR
    Request --> Controller
    Controller --> Service
    Service --> ExceptionThrown[Exception Thrown]
    ExceptionThrown --> Handler[GlobalExceptionHandler]
    Handler --> ErrorPayload[Structured ErrorResponse JSON]
    ErrorPayload --> ClientResponse[Client Receives Consistent JSON]
```

### Standard Error Response Schema

```json
{
  "status": 409,
  "error": "CART_RESTAURANT_CONFLICT",
  "message": "Cart contains items from another restaurant. Clear cart to add items from this restaurant.",
  "path": "/api/cart/items",
  "timestamp": "2026-09-14T10:55:00Z",
  "errors": null
}
```

### Handled HTTP Status Codes:
- `400 Bad Request`: `MethodArgumentNotValidException` (DTO validation failures), `IllegalArgumentException`, `IllegalOrderStateTransitionException`.
- `401 Unauthorized`: `BadCredentialsException`, `UsernameNotFoundException`, `AuthenticationException`.
- `403 Forbidden`: `AccessDeniedException` (Role or Resource Ownership violations).
- `404 Not Found`: `ResourceNotFoundException`.
- `409 Conflict`: `UserAlreadyExistsException`, `CartConflictException`, `OrderConflictException` (delivery race conditions), `DataIntegrityViolationException`.
- `500 Internal Server Error`: Generic unhandled exceptions.

---

## 20. Pagination, Sorting & Filtering

Endpoints returning collections support Spring Data `Pageable` parameters with safe defaults:
- `page`: 0-indexed page number (default: `0`).
- `size`: Page size (default: `20`, maximum constrained).
- `sort`: Dynamic sort field and direction (e.g., `sort=name,asc` or `sort=createdAt,desc`).

```mermaid
flowchart LR
    Query["GET /api/restaurants?search=pizza&isOpen=true&page=0&size=10&sort=name,asc"]
    Query --> Controller[RestaurantController]
    Controller --> Spec[JPA Specification Builder]
    Spec --> DB[(MySQL Index Scan)]
    DB --> PagedResult["PagedResponse<RestaurantResponse>"]
```

---

## 21. Validation Model

Validation is enforced across three distinct layers to prevent corrupted state:

```text
HTTP Request Body
       ↓
Layer 1: Bean Validation (Jakarta @NotNull, @NotBlank, @Min, @Email, @Size)
       ↓
Layer 2: Service-Level Business Logic (Ownership checks, State Transitions, Restaurant Open status)
       ↓
Layer 3: Database Constraints (Foreign Keys, Unique Indexes, NOT NULL columns)
```

---

## 22. Project Structure

```text
src/
├── main/
│   ├── java/com/example/fooddelivery/
│   │   ├── config/              # Redis, Security, CacheKeyUtils, App Config
│   │   ├── controller/          # REST API Controllers
│   │   ├── dto/
│   │   │   ├── request/         # Request Records (Validation Annotations)
│   │   │   └── response/        # Response Records (DTO / Entity Separation)
│   │   ├── entity/              # JPA Entities (User, Restaurant, MenuItem, Cart, Order)
│   │   ├── enums/               # OrderStatus, Role, OrderStatusTransitionValidator
│   │   ├── exception/           # Custom Domain Exceptions & GlobalExceptionHandler
│   │   ├── mapper/              # Object Mappers (Entity -> Response DTOs)
│   │   ├── repository/          # Spring Data JPA Repositories (Custom JPQL & Native Queries)
│   │   ├── security/            # JWT Token Provider, Auth Filter, CustomUserDetails
│   │   └── service/             # Business Logic Interfaces & Service Implementations
│   └── resources/
│       ├── application.yml      # Base configuration
│       ├── application-dev.yml  # Development profile
│       └── application-test.yml # Test profile (H2 in-memory DB)
└── test/
    └── java/com/example/fooddelivery/
        ├── config/              # Redis & Cache Key tests
        ├── controller/          # MockMvc Integration Tests
        ├── repository/          # JPA Specification & Repository tests
        ├── security/            # JWT & Security tests
        └── service/             # Service Unit & Concurrency Tests
```

---

## 23. Technology Stack

| Component | Technology | Version | Purpose |
|---|---|---|---|
| Language | Java | 21 (LTS) | Core programming language (Records, Pattern Matching) |
| Framework | Spring Boot | 3.4.3 | Application core, Web MVC, Dependency Injection |
| Security | Spring Security | 6.x | Stateless JWT authentication, role authorization |
| Database | MySQL | 8.0 | Authoritative transactional relational database |
| In-Memory Cache | Redis | 7.0 Alpine | Read-heavy catalog cache (Jackson JSON serialized) |
| ORM | Hibernate / JPA | 6.x | Relational persistence mapping |
| Connection Pool | HikariCP | Default | Production high-performance JDBC connection pool |
| Build Tool | Maven | 3.9+ | Build lifecycle & dependency management |
| Testing | JUnit 5 & Mockito | 5.x / 5.x | Unit and mock testing |
| Integration Testing | Spring MockMvc & H2 | Latest | Controller HTTP testing & transactional testing |
| Containerization | Docker & Compose | Latest | Orchestration for MySQL 8 and Redis 7 |

---

## 24. Non-Functional Engineering Design

- **Security**: Stateless JWT with HMAC-SHA256, BCrypt password hashing (strength 10), method-level `@PreAuthorize`, and service-layer resource ownership checks.
- **Transactional Consistency**: `@Transactional` boundaries enforce atomic cart checkout and price snapshot creation. MySQL is the single authoritative source of truth.
- **Concurrency Safety**: Atomic conditional SQL updates protect delivery order claims without distributed locking overhead.
- **Catalog Performance**: Redis caching reduces repeat database load for high-traffic restaurant and menu browsing.
- **Zero Cache Drift**: Synchronous cache evictions ensure catalog updates are immediately reflected in read APIs.
- **Resilience**: Redis outages gracefully degrade to direct database queries without causing 500 errors.

---

## 25. Testing & Quality Assurance

```mermaid
flowchart TB
    Live["Live Verification Scripts (verify_phase7.ps1, verify_phase8.ps1)"]
    Concurrency["Concurrency Integration Tests (DeliveryConcurrencyTest)"]
    Controller["MockMvc Controller Tests (Role & Endpoint Security)"]
    Service["Service Unit Tests (Business Logic & State Machine)"]
    Repository["Repository Tests (JPA Queries & Specifications)"]

    Live --> Concurrency
    Concurrency --> Controller
    Controller --> Service
    Service --> Repository
```

### Test Suite Execution Summary

```powershell
.\mvnw.cmd test
```

```text
[INFO] Results:
[INFO] 
[INFO] Tests run: 156, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

---

## 26. Live Verification

Automated PowerShell verification scripts validate the backend against live Docker instances of MySQL 8 and Redis 7:

- [verify_phase7.ps1](verify_phase7.ps1): Validates Redis connectivity, cache hits, key segregation, price mutation invalidation, availability toggle invalidation, and checkout price snapshot integrity.
- [verify_phase8.ps1](verify_phase8.ps1): Validates driver discovery, atomic claims, concurrent claim rejection (409 Conflict), kitchen-to-dispatch lifecycle progression, customer/owner delivery tracking isolation, and Phase 6 legacy backward compatibility.

---

## 27. Getting Started & Setup

### Prerequisites
- **Java 21 JDK** installed and configured (`JAVA_HOME`).
- **Docker Desktop** installed and running.
- **Git** and **Maven** (or use included `./mvnw`).

### 1. Clone the Repository
```bash
git clone https://github.com/your-username/food-delivery-backend.git
cd food-delivery-backend
```

### 2. Start MySQL & Redis Containers
```bash
docker compose up -d
```

### 3. Configure Environment Variables
Copy `.env.example` to `.env` or set environment variables:
```properties
DB_HOST=localhost
DB_PORT=3306
DB_NAME=food_delivery_db
DB_USERNAME=food_user
DB_PASSWORD=food_password
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
JWT_SECRET=your_base64_encoded_256_bit_secure_jwt_secret_key_here
JWT_EXPIRATION_MS=86400000
SERVER_PORT=8080
```

### 4. Build and Run the Application
**Linux / macOS:**
```bash
./mvnw spring-boot:run
```
**Windows:**
```powershell
.\mvnw.cmd spring-boot:run
```

### 5. Run the Full Test Suite
```powershell
.\mvnw.cmd test
```

---

## 28. Business Rules & Domain Invariants

```text
Authentication & Authorization
├── Public registration assigns ROLE_CUSTOMER exclusively.
├── Role identity cannot be client-supplied in registration payloads.
└── JWT token must be supplied in 'Authorization: Bearer <token>' header.

Restaurant & Menu Management
├── Only authenticated RESTAURANT_OWNER users can create restaurants.
├── Owners can only modify restaurants and menu items that they own.
└── Closed restaurants cannot accept new checkout orders.

Cart & Pricing
├── Each customer has exactly one persistent cart.
├── Carts enforce single-restaurant integrity (mixing restaurants returns 409 Conflict).
├── Cart prices are informational snapshots; checkout re-reads live MenuItem prices.
└── Successful checkout clears the cart and resets restaurant association.

Order Lifecycle & State Transitions
├── PLACED -> ACCEPTED -> PREPARING -> READY_FOR_PICKUP -> OUT_FOR_DELIVERY -> DELIVERED.
├── Customers can only cancel orders in 'PLACED' status.
├── Owners can cancel orders in 'PLACED', 'ACCEPTED', or 'PREPARING' status.
└── DELIVERED and CANCELLED are immutable terminal states.

Delivery & Dispatch
├── Only DELIVERY_PARTNER users can claim and fulfill deliveries.
├── Order claims are atomic (conditional SQL prevents race conditions).
└── Delivery partners can only pickup and deliver orders assigned to them.
```

---

## 29. Architectural Decision Records (ADR)

### ADR 1: Modular Monolith vs. Microservices
- **Decision**: Build as a Modular Monolith.
- **Rationale**: Eliminates distributed transaction complexities (2PC / Sagas) for cart checkout and order placement while maintaining clean domain boundaries, unified transactions, and fast local communication.

### ADR 2: MySQL as Authoritative Source of Truth
- **Decision**: MySQL 8 (InnoDB) is the sole transactional source of truth.
- **Rationale**: Ensures ACID guarantees for financial calculations, order snapshots, and atomic delivery claims.

### ADR 3: Redis Exclusively as a Read Cache
- **Decision**: Redis is strictly a catalog cache, not a transactional database.
- **Rationale**: Prevents data loss or inconsistency during cache eviction or Redis restarts. Financial data and carts remain safely in MySQL.

### ADR 4: Database Conditional Update for Delivery Claims
- **Decision**: Use `UPDATE orders SET delivery_partner_id = :id WHERE id = :orderId AND status = 'READY_FOR_PICKUP' AND delivery_partner_id IS NULL`.
- **Rationale**: Guarantees race-condition safety at the database engine level with zero distributed lock overhead.

---

## 30. Possible Future Enhancements

The following items represent potential future expansion points:
- **Payment Gateway Integration**: Webhook-driven payment processing (Stripe / Razorpay) with two-phase capture.
- **Real-Time WebSockets**: Live order tracking updates for customer and restaurant web dashboards.
- **Push Notifications**: Automated SMS / Email notifications via Firebase or AWS SNS upon order state changes.
- **Driver Geolocation**: Geospatial querying for nearest available delivery partners using PostGIS / Redis GEO.
- **Analytics & Reporting**: Daily/monthly restaurant sales summaries and driver payout metrics.

---

## License

This project is licensed under the MIT License.
