# Architecture Diagrams

## System Architecture Overview

```mermaid
graph TB
    subgraph "Client Layer"
        U[Users/Browsers]
    end
    
    subgraph "Load Balancing Layer"
        LB[Nginx Load Balancer<br/>Health Checks + SSL]
    end
    
    subgraph "Application Layer - 2 Servers"
        APP1[App Server 1<br/>Spring Boot 3.x<br/>Virtual Threads]
        APP2[App Server 2<br/>Spring Boot 3.x<br/>Virtual Threads]
    end
    
    subgraph "Cache & Coordination Layer"
        REDIS1[Redis Node 1<br/>Master<br/>Cache + Locking + Pub/Sub]
        REDIS2[Redis Node 2<br/>Slave<br/>Cache + Locking + Pub/Sub]
    end
    
    subgraph "Database Layer - 2 Servers"
        DB1[(MySQL Primary<br/>Writes + Critical Reads)]
        DB2[(MySQL Replica<br/>Read-Only)]
    end
    
    U --> LB
    LB --> APP1
    LB --> APP2
    
    APP1 --> REDIS1
    APP1 --> REDIS2
    APP2 --> REDIS1
    APP2 --> REDIS2
    
    APP1 --> DB1
    APP1 --> DB2
    APP2 --> DB1
    APP2 --> DB2
    
    DB1 -.->|Replication| DB2
    
    REDIS1 -.->|Master-Slave<br/>Replication| REDIS2
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

## Request Flow - Product Purchase

```mermaid
sequenceDiagram
    participant U as User
    participant LB as Load Balancer
    participant APP as App Server
    participant Redis as Redis Cache
    participant DB as MySQL Primary
    
    U->>LB: POST /purchase {productId: 123, qty: 2}
    LB->>APP: Route to App Server
    
    APP->>Redis: SETNX lock:product:123 (5s TTL)
    alt Lock Acquired
        Redis-->>APP: Lock Acquired
        
        APP->>Redis: GET product:123:stock
        Redis-->>APP: Stock: 10
        
        alt Sufficient Stock
            APP->>DB: BEGIN TRANSACTION
            APP->>DB: SELECT * FROM products WHERE id=123 FOR UPDATE
            DB-->>APP: Product Row (locked)
            
            APP->>DB: UPDATE products SET stock=stock-2, version=version+1 WHERE id=123 AND stock>=2
            DB-->>APP: 1 row updated
            
            APP->>DB: INSERT INTO orders (product_id, quantity)
            DB-->>APP: Order Created
            
            APP->>DB: COMMIT
            DB-->>APP: Transaction Committed
            
            APP->>Redis: DECR product:123:stock (by 2)
            Redis-->>APP: Updated Stock: 8
            
            APP->>Redis: DEL lock:product:123
            Redis-->>APP: Lock Released
            
            APP->>Redis: PUBLISH order.created (Pub/Sub)
            Redis-->>APP: Event Published (non-blocking)
            
            APP-->>LB: Purchase Success
            LB-->>U: Success Response (~50ms)
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
```

## Data Consistency Mechanism

```mermaid
graph LR
    subgraph "Layer 1: Distributed Lock"
        L1[Redis SETNX Lock<br/>5 second TTL<br/>Prevents concurrent access]
    end
    
    subgraph "Layer 2: Database Transaction"
        L2[MySQL FOR UPDATE<br/>Row-level locking<br/>ACID transaction]
    end
    
    subgraph "Layer 3: Optimistic Locking"
        L3[Version Field Check<br/>WHERE version = ?<br/>Prevents lost updates]
    end
    
    subgraph "Layer 4: Cache Consistency"
        L4[Atomic DECR in Redis<br/>Cache invalidation<br/>TTL refresh]
    end
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    
    style L1 fill:#ff9999
    style L2 fill:#99ccff
    style L3 fill:#99ff99
    style L4 fill:#ffcc99
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
        N3[Redis Master-Slave (2 Nodes)]
    end
    
    subgraph "App Server Failure"
        A1[App Server 1 Down]
        A2[Load Balancer Detects Failure]
        A3[Routes to App Server 2]
        A4[System Continues at 50% Capacity]
    end
    
    subgraph "Primary DB Failure"
        D1[Primary DB Down]
        D2[Replica Promoted to Primary]
        D3[System Continues (No Read Replica)]
    end
    
    subgraph "Redis Master Failure"
        R1[Redis Master Down]
        R2[Slave Promoted to Master]
        R3[Data Replicated from Previous Master]
        R4[System Continues with Single Redis Node]
    end
    
    N1 --> A1
    N2 --> D1
    N3 --> R1
```

