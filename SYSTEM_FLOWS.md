# Food Delivery Backend — System Flow Diagrams & Architecture

This document contains comprehensive visual diagrams and ASCII flowcharts for all user personas, domain lifecycles, concurrency guarantees, and system architecture.

---

## Table of Contents

- [1. Complete System Flow (All Users Combined)](#1-complete-system-flow-all-users-combined)
- [2. User Roles & Capabilities](#2-user-roles--capabilities)
- [3. Customer Journey Flow](#3-customer-journey-flow)
- [4. Restaurant Owner Flow](#4-restaurant-owner-flow)
- [5. Delivery Partner Flow & Concurrency Race](#5-delivery-partner-flow--concurrency-race)
- [6. Order Lifecycle State Machine](#6-order-lifecycle-state-machine)
- [7. Cart & Checkout Invariants Flow](#7-cart--checkout-invariants-flow)
- [8. Security & Authorization Model](#8-security--authorization-model)
- [9. Layered System Architecture](#9-layered-system-architecture)
- [10. Database Architecture & ER Diagram](#10-database-architecture--er-diagram)
- [11. Redis Caching & Invalidation Architecture](#11-redis-caching--invalidation-architecture)

---

## 1. Complete System Flow (All Users Combined)

### Visual ASCII Flowchart

```text
========================================================================================================
                                      COMPLETE SYSTEM FLOW
========================================================================================================

 [ CUSTOMER ]                    [ RESTAURANT OWNER ]                 [ DELIVERY PARTNER ]
      │                                    │                                    │
      ▼                                    │                                    │
  1. Register / Login                      │                                    │
      │                                    │                                    │
      ▼                                    │                                    │
  2. Browse Restaurants & Menus            │                                    │
     (Redis Cache Layer)                   │                                    │
      │                                    │                                    │
      ▼                                    │                                    │
  3. Add Items to Cart                     │                                    │
     (Single-Restaurant Rule)              │                                    │
      │                                    │                                    │
      ▼                                    │                                    │
  4. Checkout                              │                                    │
     (Re-reads Live DB Prices)             │                                    │
      │                                    │                                    │
      ▼                                    ▼                                    │
  ┌────────────────────────────────────────────────────────┐                    │
  │ Order Status: [ PLACED ]                               │                    │
  └────────────────────────────────────────────────────────┘                    │
                                           │                                    │
                                           ▼                                    │
                               5. Owner Accepts Order                           │
                               ┌───────────────────────────┐                    │
                               │ Order Status: [ ACCEPTED ]│                    │
                               └───────────────────────────┘                    │
                                           │                                    │
                                           ▼                                    │
                               6. Kitchen Begins Prep                           │
                               ┌────────────────────────────┐                   │
                               │ Order Status: [ PREPARING ]│                   │
                               └────────────────────────────┘                   │
                                           │                                    │
                                           ▼                                    │
                               7. Food Packaged & Ready                         │
                               ┌──────────────────────────────────┐             │
                               │ Order Status: [ READY_FOR_PICKUP ] ──┐         │
                               └──────────────────────────────────┘   │         │
                                                                      │         ▼
                                                          8. Discovers in Available Queue
                                                                      │
                                                                      ▼
                                                          9. Atomic Claim (MySQL Lock)
                                                                      │
                                                                      ▼
                                                         10. Pick Up Order
                                                             ┌──────────────────────────────────┐
                                                             │ Order Status: [ OUT_FOR_DELIVERY]│
                                                             └──────────────────────────────────┘
                                                                      │
                                                                      ▼
                                                         11. Mark Delivered
                                                             ┌─────────────────────────────┐
  12. Customer Receives Food <───────────────────────────────│ Order Status: [ DELIVERED ] │
      & Order Completed                                      └─────────────────────────────┘
========================================================================================================
```

### Mermaid Diagram

```mermaid
flowchart TD
    subgraph Customer Journey
        C_Auth[1. Register / Login] --> C_Browse[2. Browse Restaurants & Search]
        C_Browse --> C_Menu[3. View Categorized Menu]
        C_Menu --> C_Cart[4. Add Items to Persistent Cart]
        C_Cart --> C_Checkout[5. Execute Transactional Checkout]
        C_Checkout --> C_OrderCreated[6. Order Created: PLACED]
    end

    subgraph Restaurant Kitchen Workflow
        C_OrderCreated --> O_Inspect[7. Owner Views Incoming Order]
        O_Inspect --> O_Accept[8. Owner Accepts: ACCEPTED]
        O_Accept --> O_Prep[9. Kitchen Prepares: PREPARING]
        O_Prep --> O_Ready[10. Mark Ready: READY_FOR_PICKUP]
    end

    subgraph Dispatch & Delivery Fulfillment
        O_Ready --> D_Discover[11. Driver Discovers Available Order]
        D_Discover --> D_Claim[12. Driver Claims Order Atomically]
        D_Claim --> D_Pickup[13. Driver Marks Picked Up: OUT_FOR_DELIVERY]
        D_Pickup --> D_Deliver[14. Driver Completes Delivery: DELIVERED]
    end

    subgraph Live Tracking
        D_Pickup -.-> Track_Live[15. Customer & Owner Track Live Status]
        D_Deliver -.-> Track_Done[16. Order Delivered & Finished]
    end
```

---

## 2. User Roles & Capabilities

### Visual ASCII Role Map

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        FOOD DELIVERY PLATFORM                          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
   [ CUSTOMER ]             [ RESTAURANT OWNER ]      [ DELIVERY PARTNER ]
   • Register / Login       • Manage Restaurant       • Find Available Orders
   • Browse Catalog         • Manage Menu & Prices    • Atomic Claim
   • Manage Cart            • Toggle Availability     • Pick Up Orders
   • Checkout               • Accept / Prepare Order  • Mark Delivered
   • Cancel Placed Order    • Dispatch Order          • View Delivery History
   • Track Delivery         • Track Dispatch          • Track Status
```

### Mermaid Diagram

```mermaid
flowchart TB
    Platform[Food Delivery Platform]

    Platform --> Customers[Customer Role]
    Platform --> Owners[Restaurant Owner Role]
    Platform --> Drivers[Delivery Partner Role]

    Customers --> C1[Browse Restaurants & Menus]
    Customers --> C2[Manage Shopping Cart]
    Customers --> C3[Execute Checkout]
    Customers --> C4[Cancel Placed Orders]
    Customers --> C5[Track Delivery Status]

    Owners --> O1[Manage Restaurant Profile]
    Owners --> O2[Manage Menu Items & Prices]
    Owners --> O3[Toggle Item Availability]
    Owners --> O4[Manage Incoming Orders]
    Owners --> O5[Progress Kitchen Status]

    Drivers --> D1[Discover Available Deliveries]
    Drivers --> D2[Atomic Concurrency-Safe Claim]
    Drivers --> D3[Pickup: OUT_FOR_DELIVERY]
    Drivers --> D4[Complete: DELIVERED]
    Drivers --> D5[View Assigned Delivery History]
```

---

## 3. Customer Journey Flow

### Visual ASCII Sequence

```text
CUSTOMER                    API / CONTROLLER              REDIS CACHE                MYSQL DATABASE
   │                              │                            │                           │
   │── 1. POST /api/auth/login ──>│                            │                           │
   │<─ 2. JWT Access Token ───────│                            │                           │
   │                              │                            │                           │
   │── 3. GET /api/restaurants ──>│                            │                           │
   │                              │── 4. Check Cache ─────────>│                           │
   │                              │<─ 5. Cache Miss ───────────│                           │
   │                              │── 6. Query Open Restaurants ──────────────────────────>│
   │                              │<─ 7. Restaurant Records ───────────────────────────────│
   │                              │── 8. Cache DTOs (5m TTL) ─>│                           │
   │<─ 9. Restaurant List ────────│                            │                           │
   │                              │                            │                           │
   │── 10. POST /api/cart/items ─>│ (menuItemId=101, qty=2)    │                           │
   │                              │── 11. Validate Single Restaurant & Save ──────────────>│
   │<─ 12. CartResponse ──────────│                            │                           │
   │                              │                            │                           │
   │── 13. POST /checkout ───────>│                            │                           │
   │                              │── 14. [TRANSACTION BEGIN] ────────────────────────────>│
   │                              │── 15. Re-read Live MenuItem Prices ───────────────────>│
   │                              │── 16. Compute Subtotal + Tax + Delivery Fee ──────────>│
   │                              │── 17. Insert Order & OrderItem Snapshots ─────────────>│
   │                              │── 18. Clear Customer Cart ────────────────────────────>│
   │                              │── 19. [TRANSACTION COMMIT] ───────────────────────────>│
   │<─ 20. Order (Status: PLACED)─│                            │                           │
```

### Mermaid Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant API as REST Controller
    participant Auth as Spring Security / JWT
    participant Service as Order & Cart Service
    participant Redis as Redis 7 Cache
    participant MySQL as MySQL 8 (Source of Truth)

    Customer->>API: POST /api/auth/login
    API->>Auth: Validate email & password
    Auth-->>Customer: Return JWT Access Token

    Customer->>API: GET /api/restaurants?page=0&size=10
    API->>Service: getRestaurants()
    Service->>Redis: Check cache 'restaurants'
    alt Cache Hit
        Redis-->>Service: Return cached DTOs (fast response)
    else Cache Miss
        Service->>MySQL: SELECT * FROM restaurants WHERE is_open = true
        MySQL-->>Service: Restaurant entities
        Service->>Redis: Store in cache (5 min TTL)
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

## 4. Restaurant Owner Flow

### Visual ASCII Flowchart

```text
[ RESTAURANT OWNER ]
         │
         ├──► 1. Manage Restaurant Profile ──► Evict 'restaurants' & 'restaurantDetails' Cache
         │
         ├──► 2. Manage Menu Items & Prices ─► Evict 'restaurantMenus' & 'menuItems' Cache
         │
         ├──► 3. Toggle Availability ────────► Evict 'restaurantMenus' Cache
         │
         └──► 4. Kitchen Order Management:
                     │
                     ▼
                 [ PLACED ]
                     │
                     ▼ (PATCH /api/owner/orders/{id}/status -> ACCEPTED)
                 [ ACCEPTED ]
                     │
                     ▼ (PATCH /api/owner/orders/{id}/status -> PREPARING)
                 [ PREPARING ]
                     │
                     ▼ (PATCH /api/owner/orders/{id}/status -> READY_FOR_PICKUP)
             [ READY_FOR_PICKUP ] ──► (Appears in Driver Available Deliveries Queue)
```

### Mermaid Diagram

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

    Ready --> Queue[(MySQL: Ready for Delivery Claim)]
```

---

## 5. Delivery Partner Flow & Concurrency Race

### Visual ASCII Race Condition Resolution

```text
Order #500: [ READY_FOR_PICKUP ] (delivery_partner_id = NULL)

  DRIVER A (Requests Claim)                       DRIVER B (Requests Claim)
         │                                               │
         ▼                                               ▼
POST /api/deliveries/500/claim                  POST /api/deliveries/500/claim
         │                                               │
         ▼                                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ATOMIC MYSQL DATABASE UPDATE                          │
│                                                                             │
│  UPDATE orders                                                              │
│  SET delivery_partner_id = :driverId, updated_at = NOW()                    │
│  WHERE id = 500 AND status = 'READY_FOR_PICKUP'                             │
│    AND delivery_partner_id IS NULL;                                         │
└─────────────────────────────────────────────────────────────────────────────┘
         │                                               │
         ▼                                               ▼
  [ 1 Row Affected ]                              [ 0 Rows Affected ]
         │                                               │
         ▼                                               ▼
  HTTP 200 OK                                     HTTP 409 CONFLICT
  (Driver A Claims Order)                         (ORDER_ALREADY_CLAIMED)
         │
         ▼ (PATCH /api/deliveries/500/pickup)
  [ OUT_FOR_DELIVERY ]
         │
         ▼ (PATCH /api/deliveries/500/deliver)
  [ DELIVERED ]
```

### Mermaid Diagram

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

---

## 6. Order Lifecycle State Machine

### Visual ASCII State Diagram

```text
                     ┌──────────────────┐
                     │     PLACED       │
                     └────────┬─────────┘
                              │
            ┌─────────────────┴─────────────────┐
            │                                   │
            ▼ (Owner Accepts)                   ▼ (Customer/Owner Cancels)
     ┌──────────────┐                   ┌───────────────┐
     │   ACCEPTED   │                   │   CANCELLED   │◄─── [Terminal State]
     └──────┬───────┘                   └───────────────┘
            │                                   ▲
            │ (Owner Prepares)                  │ (Owner Cancels)
            ▼                                   │
     ┌──────────────┐                           │
     │  PREPARING   ├───────────────────────────┘
     └──────┬───────┘
            │ (Food Ready)
            ▼
 ┌──────────────────────┐
 │   READY_FOR_PICKUP   │
 └──────────┬───────────┘
            │ (Assigned Driver Picks Up)
            ▼
 ┌──────────────────────┐
 │   OUT_FOR_DELIVERY   │
 └──────────┬───────────┘
            │ (Assigned Driver Delivers)
            ▼
 ┌──────────────────────┐
 │      DELIVERED       │◄─── [Terminal State]
 └──────────────────────┘
```

### Mermaid Diagram

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

---

## 7. Cart & Checkout Invariants Flow

### Visual ASCII Diagram

```text
[ Customer Adds Item ]
         │
         ▼
  Does Cart Exist? ──No──► [ Create Customer Cart ] ──► [ Associate Restaurant ]
         │ Yes
         ▼
  Matches Cart Restaurant? ──No──► [ 409 CONFLICT: CART_RESTAURANT_CONFLICT ]
         │ Yes
         ▼
  [ Add / Increment CartItem ]
         │
         ▼
  [ Customer Clicks Checkout ]
         │
  ┌──────▼──────────────────────────────────────────────────────────────────┐
  │                   TRANSACTIONAL CHECKOUT BOUNDARY                       │
  │                                                                         │
  │  1. Check Restaurant is Open (Reject if closed)                         │
  │  2. Re-read live prices from menu_items table                           │
  │  3. Check Item availability (Reject if unavailable)                     │
  │  4. Compute Subtotal + Tax (5%) + Delivery Fee (Rs 50) + Total          │
  │  5. Persist Order and immutable OrderItem snapshots                     │
  │  6. Clear Customer Cart and reset restaurant                            │
  └─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
  [ Order Created: Status PLACED ]
```

### Mermaid Diagram

```mermaid
flowchart TD
    Start([Customer Adds Item to Cart]) --> CheckCart{Does Cart Exist?}
    CheckCart -->|No| CreateCart[Create Persistent Cart for Customer]
    CheckCart -->|Yes| CheckRest{Matches Current Cart Restaurant?}

    CreateCart --> SetRest[Associate Restaurant]
    CheckRest -->|Yes| AddItem[Add / Increment Cart Item]
    CheckRest -->|No| Conflict[409 Conflict: CART_RESTAURANT_CONFLICT]

    AddItem --> ViewCart[GET /api/cart: Displays Subtotal & Items]
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

---

## 8. Security & Authorization Model

### Visual ASCII Diagram

```text
[ HTTP Request ]
       │
       ▼
[ JwtAuthenticationFilter ]
       │
       ├──► Has Valid Bearer JWT? ──No──► [ Anonymous Principal ]
       │                                         │
       └──► Yes ──► [ Extract Roles & Subject ]  │
                         │                       │
                         ▼                       ▼
            [ Check Route Security Rules (SecurityConfig) ]
                         │
        ┌────────────────┼────────────────┐
        │ Public Route   │ Protected      │ Role Mismatch
        ▼                ▼                ▼
    [ 200 OK ]    [ Check Roles ]    [ 403 FORBIDDEN ]
                         │
                         ▼ Role Matched
           [ Service Resource Ownership Check ]
                         │
        ┌────────────────┴────────────────┐
        │ Valid Ownership                 │ Ownership Mismatch
        ▼                                 ▼
   [ Execute Business Logic ]        [ 403 FORBIDDEN ]
```

### Mermaid Diagram

```mermaid
flowchart TD
    Request[Incoming HTTP Request] --> JwtFilter[JwtAuthenticationFilter]
    JwtFilter --> HasHeader{Authorization: Bearer JWT?}

    HasHeader -->|No / Invalid| Anon[Anonymous Authentication]
    HasHeader -->|Valid| Extract[Extract Subject & Roles]
    Extract --> Context[Populate SecurityContext with CustomUserDetails]

    Anon --> SecRule{Endpoint Rule}
    Context --> SecRule

    SecRule -->|Public Route| AllowPublic[Permit Request (200 OK)]
    SecRule -->|Protected & Missing Auth| Ret401[401 Unauthorized]
    SecRule -->|Protected & Role Match| CheckOwnership{Service Ownership Check}
    SecRule -->|Protected & Role Mismatch| Ret403[403 Forbidden]

    CheckOwnership -->|Owner / Resource Valid| Execute[Execute Service & Commit MySQL]
    CheckOwnership -->|Resource Mismatch| Ret403
```

---

## 9. Layered System Architecture

### Visual ASCII Diagram

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                               │
│            Web Browser / Mobile Application / Postman Scripts           │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ HTTP / REST
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYER (Spring Security 6)                   │
│          Stateless JWT Filter  •  BCrypt Password Encoder  •  CORS      │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  CONTROLLER LAYER (REST Controllers)                    │
│  AuthController  •  RestaurantController  •  MenuItemController         │
│  CartController  •  OrderController       •  DeliveryController         │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             SERVICE LAYER                               │
│  AuthService     •  RestaurantService     •  MenuItemService            │
│  CartService     •  OrderService          •  DeliveryService            │
└───────────────────┬─────────────────────────────────┬───────────────────┘
                    │                                 │
           (Cached Catalog Reads)         (Transactional Operations)
                    │                                 │
                    ▼                                 ▼
┌──────────────────────────────────────┐  ┌───────────────────────────────┐
│             REDIS 7 CACHE            │  │        SPRING DATA JPA        │
│  • restaurants                       │  │      Hibernate ORM 6.x        │
│  • restaurantDetails                 │  └───────────────┬───────────────┘
│  • restaurantMenus                   │                  │
│  • menuItems                         │                  ▼
│  (JSON Serialized, Normalized Keys)  │  ┌───────────────────────────────┐
└──────────────────────────────────────┘  │        MYSQL 8 DATABASE       │
                                          │   (InnoDB Source of Truth)    │
                                          └───────────────────────────────┘
```

### Mermaid Diagram

```mermaid
flowchart TB
    subgraph Client Layer
        Web[Web Browser / React / Next.js]
        Mobile[Mobile Application / iOS / Android]
        Postman[Postman / Integration Test Scripts]
    end

    subgraph Security Layer
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

## 10. Database Architecture & ER Diagram

### Visual ASCII ER Diagram

```text
┌─────────────────────────────────────────┐
│                 USERS                   │
├─────────────────────────────────────────┤
│ PK  id               bigint             │
│ UK  email            varchar(100)       │
│     name             varchar(100)       │
│     password         varchar(255)       │
│     phone            varchar(20)        │
│     role             enum               │
└────┬─────────────────┬──────────────────┘
     │ 1:N (owns)      │ 1:1 (customer)
     ▼                 ▼
┌──────────────────┐ ┌─────────────────────────────────────────┐
│   RESTAURANTS    │ │                  CARTS                  │
├──────────────────┤ ├─────────────────────────────────────────┤
│ PK  id           │ │ PK  id               bigint             │
│ FK  owner_id     │ │ FK  customer_id (UK) bigint             │
│     name         │ │ FK  restaurant_id    bigint (nullable)  │
│     address      │ └────┬────────────────────────────────────┘
│     is_open      │      │ 1:N (contains)
└────┬─────────────┘      ▼
     │ 1:N           ┌─────────────────────────────────────────┐
     ▼               │               CART_ITEMS                │
┌──────────────────┐ ├─────────────────────────────────────────┤
│    MENU_ITEMS    │ │ PK  id               bigint             │
├──────────────────┤ │ FK  cart_id          bigint             │
│ PK  id           │ │ FK  menu_item_id     bigint             │
│ FK  restaurant_id│ │     quantity         int                │
│     name         │ │     price (snapshot) decimal(10,2)      │
│     price        │ └─────────────────────────────────────────┘
│     is_available │
└──────────────────┘

     ┌─────────────────────────────────────────────────────────┐
     │                         ORDERS                          │
     ├─────────────────────────────────────────────────────────┤
     │ PK  id                  bigint                          │
     │ UK  order_number        varchar(64)                     │
     │ FK  customer_id         bigint                          │
     │ FK  restaurant_id       bigint                          │
     │ FK  delivery_partner_id bigint (nullable)               │
     │     status              enum (PLACED...DELIVERED)       │
     │     subtotal_amount     decimal(10,2)                   │
     │     tax_amount          decimal(10,2)                   │
     │     delivery_fee        decimal(10,2)                   │
     │     total_amount        decimal(10,2)                   │
     │ UK  idempotency_key     varchar(128) (nullable)         │
     └────┬────────────────────────────────────────────────────┘
          │ 1:N (contains)
          ▼
     ┌─────────────────────────────────────────────────────────┐
     │                      ORDER_ITEMS                        │
     │             (Immutable Historical Snapshots)            │
     ├─────────────────────────────────────────────────────────┤
     │ PK  id                  bigint                          │
     │ FK  order_id            bigint                          │
     │ FK  menu_item_id        bigint (nullable, ON DELETE SET)│
     │     item_name           varchar(120)                    │
     │     price               decimal(10,2)                   │
     │     quantity            int                             │
     │     subtotal            decimal(10,2)                   │
     └─────────────────────────────────────────────────────────┘
```

### Mermaid Diagram

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

## 11. Redis Caching & Invalidation Architecture

### Visual ASCII Cache & Eviction Flow

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CATALOG READ FLOW (Cache-Aside)                   │
└─────────────────────────────────────────────────────────────────────────────┘

  Client Request (GET /api/restaurants or /api/restaurants/{id}/menu)
       │
       ▼
  [ Service Layer (CacheKeyUtils) ]
       │
       ├──► Check Redis Cache (Key: list:s={search}:o={isOpen}:p={page}:sz={size}:sort={sort})
       │         │
       │         ├──► [ CACHE HIT ] ──► Return Cached JSON DTO (Fast Response ~2ms)
       │         │
       │         └──► [ CACHE MISS ]
       │                   │
       │                   ▼
       │              Query MySQL Source of Truth
       │                   │
       │                   ▼
       │              Serialize DTO to Redis (TTL: 5m for lists, 10m for details)
       │                   │
       │                   ▼
       │              Return DTO to Client
       │
       └──► (Redis Outage) ──► [ CacheErrorHandler Fallback ] ──► Direct MySQL Query (No 500)


┌─────────────────────────────────────────────────────────────────────────────┐
│                           CATALOG MUTATION & EVICTION FLOW                  │
└─────────────────────────────────────────────────────────────────────────────┘

  Restaurant / Menu Modification (POST, PUT, PATCH, DELETE)
       │
       ▼
  1. Commit Mutation to MySQL Source of Truth
       │
       ▼
  2. Trigger @CacheEvict Annotations
       │
       ├──► Evict Specific Item Detail: 'restaurantDetails' or 'menuItems' (#id)
       │
       └──► Evict All Catalog Lists: 'restaurants' or 'restaurantMenus' (allEntries=true)
       │
       ▼
  3. Subsequent Client Reads Fetch Fresh Authoritative Data from MySQL
```

### Mermaid Diagram

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

