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

## Request Flow - Product Purchase (Saga Pattern)

```mermaid
sequenceDiagram
    participant U as User
    participant LB as Load Balancer
    participant APP as Saga Orchestrator
    participant Redis as Redis Cache
    participant DB as MySQL Primary
    participant KAFKA as Kafka Broker
    
    U->>LB: POST /purchase {productId: 123, qty: 2}
    LB->>APP: Route to App Server
    
    APP->>Redis: SETNX lock:product:123 (30s TTL)
    alt Lock Acquired
        Redis-->>APP: Lock Acquired
        
        APP->>Redis: GET product:123:stock
        Redis-->>APP: Stock: 10
        
        alt Sufficient Stock
            APP->>KAFKA: Publish SagaStartEvent
            KAFKA-->>APP: Event Published
            
            Note over APP,DB: Step 1: Reserve Inventory
            APP->>DB: BEGIN TX: UPDATE reserved_stock
            DB-->>APP: Reserved
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
            
            Note over APP,DB: Step 4: Confirm Order
            APP->>DB: BEGIN TX: UPDATE order status + finalize inventory
            DB-->>APP: Confirmed
            APP->>KAFKA: Publish OrderConfirmedEvent
            KAFKA-->>APP: Event Published
            
            APP->>Redis: DECR product:123:stock (by 2)
            Redis-->>APP: Updated Stock: 8
            
            APP->>Redis: DEL lock:product:123
            Redis-->>APP: Lock Released
            
            APP-->>LB: Purchase Success
            LB-->>U: Success Response (~100ms)
        else Insufficient Stock
            APP->>Redis: DEL lock:product:123
            APP-->>LB: Insufficient Stock Error
            LB-->>U: Error Response (~10ms)
        end
    else Lock Not Acquired
        Redis-->>APP: Lock Failed
        APP-->>LB: Product Being Processed
        LB-->>U: Retry Response (~5ms)
    end
    
    Note over APP,KAFKA: If any step fails:<br/>Compensation events published<br/>Rollback in reverse order
```

## Data Consistency Mechanism (Saga Pattern)

```mermaid
graph LR
    subgraph "Layer 1: Distributed Lock"
        L1[Redis SETNX Lock<br/>30 second TTL<br/>Prevents concurrent access]
    end
    
    subgraph "Layer 2: Saga Orchestration"
        L2[Kafka Events<br/>Saga coordination<br/>Event sourcing]
    end
    
    subgraph "Layer 3: Local Transactions"
        L3[Step 1: Reserve Inventory<br/>Step 2: Create Order<br/>Step 3: Process Payment<br/>Step 4: Confirm Order]
    end
    
    subgraph "Layer 4: Compensation"
        L4[Automatic Rollback<br/>Reverse order compensation<br/>Event-driven]
    end
    
    subgraph "Layer 5: Cache Consistency"
        L5[Atomic DECR in Redis<br/>After saga completes<br/>TTL refresh]
    end
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5
    
    style L1 fill:#ff9999
    style L2 fill:#99ccff
    style L3 fill:#99ff99
    style L4 fill:#ffcc99
    style L5 fill:#ff99cc
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

