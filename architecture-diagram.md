# Architecture Diagrams

## System Architecture Overview

```mermaid
graph TB
    subgraph ClientLayer["Client Layer"]
        U[Users/Browsers]
    end
    
    subgraph LoadBalancerLayer["Load Balancing Layer"]
        LB[Nginx Load Balancer<br/>Health Checks + SSL]
    end
    
    subgraph AppLayer["Application Layer - 2 Servers"]
        APP1[App Server 1<br/>Spring Boot 3.x<br/>Virtual Threads]
        APP2[App Server 2<br/>Spring Boot 3.x<br/>Virtual Threads]
    end
    
    subgraph CacheLayer["Cache and Message Broker Layer"]
        SERVER5[Server 5<br/>Redis Cluster Node 1<br/>+ Kafka Broker 1<br/>Memory + Disk]
        SERVER6[Server 6<br/>Redis Cluster Node 2<br/>+ Kafka Broker 2<br/>Memory + Disk]
    end
    
    subgraph DatabaseLayer["Database Layer - 2 Servers"]
        DB1[(MySQL Primary<br/>Writes + Critical Reads)]
        DB2[(MySQL Replica<br/>Read-Only)]
    end
    
    U --> LB
    LB --> APP1
    LB --> APP2
    
    APP1 --> SERVER5
    APP1 --> SERVER6
    APP2 --> SERVER5
    APP2 --> SERVER6
    
    APP1 --> DB1
    APP1 --> DB2
    APP2 --> DB1
    APP2 --> DB2
    
    DB1 -.->|Replication| DB2
    
    SERVER5 -.->|Redis Cluster| SERVER6
    SERVER5 -.->|Kafka Cluster| SERVER6
    SERVER5 -.->|Saga Events| APP1
    SERVER5 -.->|Saga Events| APP2
    SERVER6 -.->|Saga Events| APP1
    SERVER6 -.->|Saga Events| APP2
```

## Request Flow - Product Viewing

```mermaid
sequenceDiagram
    participant U as User
    participant LB as Load Balancer
    participant APP as App Server
    participant Redis as Redis Cache
    participant DB as MySQL Replica
    
    U->>LB: GET /products/123
    LB->>APP: Route to App Server
    APP->>Redis: Get product:123:details
    alt Cache Hit
        Redis-->>APP: Product Data (1ms)
        APP-->>LB: Response
        LB-->>U: Product Info (~5ms total)
    else Cache Miss
        Redis-->>APP: Cache Miss
        APP->>DB: SELECT * FROM products WHERE id=123
        DB-->>APP: Product Data (15ms)
        APP->>Redis: SET product:123:details (TTL: 300s)
        APP-->>LB: Response
        LB-->>U: Product Info (~20ms total)
    end
```

## Request Flow - Product Purchase (Saga Pattern với Redis Atomic Operations)

```mermaid
sequenceDiagram
    participant U as User
    participant LB as Load Balancer
    participant APP as Saga Orchestrator
    participant Redis as Redis (Lock + Atomic Ops)
    participant DB as MySQL Primary
    participant KAFKA as Kafka Broker
    
    U->>LB: POST /purchase {productId: 123, qty: 2}
    LB->>APP: Route to App Server
    
    APP->>Redis: SETNX lock:product:123 (30s TTL)
    alt Lock Acquired
        Redis-->>APP: Lock Acquired
        
        APP->>KAFKA: Publish SagaStartEvent
        KAFKA-->>APP: Event Published
        
        Note over APP,Redis: Step 1: Reserve Inventory (Redis Atomic)
        APP->>Redis: Lua Script: Reserve Inventory<br/>Check available + INCR reserved
        alt Sufficient Stock
            Redis-->>APP: Reserved Success<br/>Available: 8, Reserved: 2
            APP->>KAFKA: Publish InventoryReservedEvent
            KAFKA-->>APP: Event Published
            
            Note over APP,DB: Step 2: Create Order
            APP->>DB: BEGIN TX: INSERT INTO orders
            DB-->>APP: Order Created (id: 456)
            APP->>KAFKA: Publish OrderCreatedEvent
            KAFKA-->>APP: Event Published
            
            Note over APP: Step 3: Process Payment
            APP->>APP: Process Payment
            APP->>KAFKA: Publish PaymentProcessedEvent
            KAFKA-->>APP: Event Published
            
            Note over APP,Redis: Step 4: Confirm Order (Redis Atomic)
            APP->>DB: UPDATE order status = 'CONFIRMED'
            DB-->>APP: Confirmed
            APP->>Redis: Lua Script: Confirm Order<br/>DECR reserved + DECR stock
            Redis-->>APP: Stock: 8, Reserved: 0
            APP->>KAFKA: Publish OrderConfirmedEvent
            KAFKA-->>APP: Event Published
            
            APP->>Redis: DEL lock:product:123
            Redis-->>APP: Lock Released
            
            APP-->>LB: Purchase Success
            LB-->>U: Success Response (~100ms)
        else Insufficient Stock
            Redis-->>APP: Reserve Failed<br/>Available: 0
            APP->>Redis: DEL lock:product:123
            APP-->>LB: Insufficient Stock Error
            LB-->>U: Error Response (~10ms)
        end
    else Lock Not Acquired
        Redis-->>APP: Lock Failed
        APP-->>LB: Product Being Processed
        LB-->>U: Retry Response (~5ms)
    end
    
    Note over APP,KAFKA: If any step fails:<br/>Rollback reservation trong Redis<br/>DECR reserved (atomic operation)<br/>Compensation events published
```

## Data Consistency Mechanism (Saga Pattern với Redis Atomic + DB Optimistic Locking)

```mermaid
graph TB
    subgraph "Layer 1: Distributed Lock"
        L1[Redis SETNX Lock<br/>30 second TTL<br/>Prevents concurrent access]
    end
    
    subgraph "Layer 2: Saga Orchestration"
        L2[Kafka Events<br/>Saga coordination<br/>Event sourcing]
    end
    
    subgraph "Layer 3: Redis Atomic Operations"
        L3[Step 1: Reserve Inventory<br/>Lua Script: Check available + INCR reserved<br/>Atomic operation trong Redis]
    end
    
    subgraph "Layer 4: Database Optimistic Locking"
        L4["Step 2: Create Order<br/>INSERT với version tracking<br/>Step 4: Confirm Order<br/>UPDATE với version check<br/>WHERE id AND version match"]
    end
    
    subgraph "Layer 5: Compensation"
        L5["Automatic Rollback<br/>Redis: DECR reserved (atomic)<br/>DB: UPDATE với version check<br/>Event-driven compensation"]
    end
    
    subgraph "Layer 6: Multi-Layer Consistency"
        L6["Redis: Atomic INCR/DECR<br/>Lua scripts đảm bảo atomicity<br/>DB: Optimistic locking với version<br/>Detect concurrent modifications"]
    end
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5
    L5 --> L6
    
    style L1 fill:#ff9999
    style L2 fill:#99ccff
    style L3 fill:#99ff99
    style L4 fill:#ffcc99
    style L5 fill:#ff99cc
    style L6 fill:#cc99ff
```

## Capacity Distribution

```mermaid
pie title Request Distribution (N=6000)
    "Redis Cache Hits" : 4000
    "DB Replica Reads" : 800
    "Direct DB Writes" : 1200
```

```mermaid
pie title Database Connection Usage (C=300 per DB)
    "Primary DB Connections" : 300
    "Replica Connections" : 300
```

## High Availability - Failover Scenarios

```mermaid
graph TB
    subgraph "Normal Operation"
        N1[All 2 App Servers Active]
        N2[Primary DB + 1 Replica]
        N3[Redis Cluster - 2 nodes]
        N4[Kafka Cluster - 2 brokers]
    end
    
    subgraph "App Server Failure"
        A1[App Server 1 Down]
        A2[Load Balancer Detects Failure]
        A3[Routes to App Server 2]
        A4["System Continues at 50% Capacity<br/>Saga state in Kafka"]
    end
    
    subgraph "Primary DB Failure"
        D1[Primary DB Down]
        D2[Replica Promoted to Primary]
        D3[System Continues - No Read Replica]
    end
    
    subgraph "Redis+Kafka Server Failure"
        RK1[Server 5 Down - Redis+Kafka]
        RK2[Redis Cluster continues with Server 6]
        RK3[Kafka Cluster continues with Broker 2]
        RK4["System Continues Normally<br/>Perfect Resource Sharing"]
    end
    
    N1 --> A1
    N2 --> D1
    N3 --> RK1
    N4 --> RK1
```

## Redis Distributed Lock - Concurrent Purchase Prevention

```mermaid
sequenceDiagram
    participant UA as User A
    participant UB as User B
    participant APP1 as App Server 1
    participant APP2 as App Server 2
    participant Redis as Redis Lock
    participant DB as MySQL
    
    Note over UA,DB: Both users try to buy Product ID 123 (Stock: 1)
    
    UA->>APP1: POST /purchase (product:123, qty:1)
    UB->>APP2: POST /purchase (product:123, qty:1)
    
    APP1->>Redis: SETNX lock:product:123
    APP2->>Redis: SETNX lock:product:123
    
    Redis-->>APP1: ✅ SUCCESS (Lock acquired)
    Redis-->>APP2: ❌ FAILED (Lock exists)
    
    Note over APP1,Redis: User A proceeds with purchase
    APP1->>Redis: Lua Script: Reserve Inventory<br/>INCR reserved (atomic)
    Redis-->>APP1: Reserved Success
    APP1->>DB: Create order
    APP1->>Redis: Lua Script: Confirm Order<br/>DECR reserved + DECR stock (atomic)
    Redis-->>APP1: Stock: 0, Reserved: 0
    APP1->>Redis: Release lock
    
    Note over APP2: User B cannot proceed
    APP2-->>UB: Error: "Product is being processed"
    
    Note over UA,DB: Result: Only User A purchased successfully<br/>Stock = 0 (Redis), User B blocked by lock
```

## Concurrent Purchase: 10 Users, Stock = 9

```mermaid
sequenceDiagram
    participant U1 as User 1
    participant U2 as User 2-9
    participant U10 as User 10
    participant Redis as Redis Lock
    participant DB as MySQL
    
    Note over U1,DB: 10 Users cùng mua Product 123 (Stock = 9)
    
    par All users request simultaneously
        U1->>Redis: SETNX lock:product:123
        U2->>Redis: SETNX lock:product:123
        U10->>Redis: SETNX lock:product:123
    end
    
    Redis-->>U1: ✅ SUCCESS (Lock acquired)
    Redis-->>U2: ❌ FAILED (Lock exists)
    Redis-->>U10: ❌ FAILED (Lock exists)
    
    Note over U1,Redis: User 1 proceeds (Stock: 9 → 8)
    U1->>Redis: Lua Script: Reserve Inventory<br/>INCR reserved (atomic)
    Redis-->>U1: Reserved Success (Available: 8)
    U1->>DB: Create order
    U1->>Redis: Lua Script: Confirm Order<br/>DECR reserved + DECR stock (atomic)
    Redis-->>U1: Stock: 8, Reserved: 0
    U1->>Redis: Release lock
    
    Note over U2,Redis: User 2-9 retry sequentially
    loop Users 2-9
        U2->>Redis: SETNX lock:product:123
        Redis-->>U2: ✅ SUCCESS
        U2->>Redis: Lua Script: Reserve Inventory<br/>INCR reserved (atomic)
        Redis-->>U2: Reserved Success
        U2->>DB: Create order
        U2->>Redis: Lua Script: Confirm Order<br/>DECR reserved + DECR stock (atomic)
        U2->>Redis: Release lock
    end
    
    Note over U10: User 10 tries after all others
    U10->>Redis: SETNX lock:product:123
    Redis-->>U10: ✅ SUCCESS
    U10->>Redis: Lua Script: Reserve Inventory<br/>Check available stock
    Redis-->>U10: Reserve Failed (Available: 0)
    U10->>Redis: Release lock
    U10-->>U10: ❌ Purchase failed (Insufficient stock)
    
    Note over U1,DB: Result: Users 1-9 succeed<br/>User 10 fails (Stock = 0 in Redis)
```

## Distributed Lock vs Optimistic Locking Comparison

```mermaid
graph TB
    subgraph DistributedLock["Distributed Lock Approach"]
        DL1[10 Users request]
        DL2[Only 1 acquires lock]
        DL3[Sequential processing]
        DL4[9 succeed, 1 fails]
        DL5[Avg latency: ~900ms]
        
        DL1 --> DL2
        DL2 --> DL3
        DL3 --> DL4
        DL4 --> DL5
    end
    
    subgraph OptimisticLock["Optimistic Locking Approach"]
        OL1[10 Users request]
        OL2[All read data]
        OL3[Parallel processing]
        OL4[9 succeed after retries]
        OL5[Avg latency: ~50ms]
        OL6[High retry overhead]
        
        OL1 --> OL2
        OL2 --> OL3
        OL3 --> OL4
        OL4 --> OL5
        OL4 --> OL6
    end
    
    style DL5 fill:#99ff99
    style OL5 fill:#ffcc99
    style OL6 fill:#ff9999
```

