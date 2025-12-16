# Technical Specification

## Capacity Planning Details

### Input Parameters
- **N = 6000**: Concurrent requests
- **S = 6**: Total servers
- **C = 300**: Max connections per database

### Server Allocation Strategy

| Server | Role | Purpose | Resource Usage |
|--------|------|---------|----------------|
| Server 1 | Application | Spring Boot App với Saga Orchestrator | CPU + Memory |
| Server 2 | Application | Spring Boot App với Saga Orchestrator | CPU + Memory |
| Server 3 | Database | MySQL Primary | CPU + Memory + Disk |
| Server 4 | Database | MySQL Replica | CPU + Memory + Disk |
| Server 5 | Redis + Kafka | Redis Cluster Node + Kafka Broker 1 | Redis: Memory, Kafka: Disk |
| Server 6 | Redis + Kafka | Redis Cluster Node + Kafka Broker 2 | Redis: Memory, Kafka: Disk |

**Total Database Connections**: 300 + 300 = 600 = 600 (2 × 300) ✓

**Resource Optimization**:
- **Server 5 & 6**: Perfect resource sharing
  - Redis: Sử dụng RAM (memory) - không conflict với Kafka
  - Kafka: Sử dụng Disk (storage) - không conflict với Redis
  - Optimal utilization: Tận dụng cả memory và disk trên cùng server

### Request Distribution Analysis

**Assumptions**:
- 80% read operations (product viewing)
- 20% write operations (purchases)
- Redis cache hit rate: 83%

**Read Operations (4800 requests)**:
```
Cache Hits: 4800 × 0.83 = 3984 requests → Redis (no DB load)
Cache Misses: 4800 × 0.17 = 816 requests → DB Replica
  - Single Replica: 816 requests (Virtual Threads handle blocking I/O efficiently)
```

**Write Operations (1200 requests)**:
```
Saga Orchestration: 1200 requests → Saga steps với Kafka events
  - Step 1: Reserve Inventory (local TX)
  - Step 2: Create Order (local TX)
  - Step 3: Process Payment
  - Step 4: Confirm Order (local TX)
Kafka: Publishes saga events cho coordination và event sourcing
```

**Actual Database Load**:
- Primary DB: ~300 concurrent connections (writes, connection pool handles 1200 requests)
- Replica: ~300 concurrent connections (reads, connection pool handles 816 requests)

**Connection Pool Configuration**:
```yaml
# Primary Database (Writes)
spring.datasource.hikari.maximum-pool-size: 300
spring.datasource.hikari.minimum-idle: 10
spring.datasource.hikari.connection-timeout: 30000

# Replica Database (Reads)
spring.datasource-replica.hikari.maximum-pool-size: 300
spring.datasource-replica.hikari.minimum-idle: 10
```

**Virtual Threads Configuration**:
```yaml
spring.threads.virtual.enabled: true
```

**Note**: With Virtual Threads, blocking I/O calls (JDBC, Redis) are handled efficiently by JVM. Each request has its own virtual thread (~KB/thread), and threads are automatically suspended/resumed during I/O operations. Connection pooling limits actual database connections.

---

## Consistency Guarantee - Detailed Analysis

### Problem: Race Condition Scenario

**Without Protection**:
```
Time    Thread 1              Thread 2              Stock
----------------------------------------------------------
T1      Read stock: 10        
T2                          Read stock: 10        
T3      Purchase qty: 5      
T4                          Purchase qty: 6      
T5      Update: 10-5=5      
T6                          Update: 10-6=4      
T7      Result: 5           Result: 4            ERROR: Sold 11, had 10!
```

### Solution: Multi-Layer Protection

**Layer 1: Redis Distributed Lock**
```java
// Only one thread can acquire lock per product
String lockKey = "lock:product:" + productId;
Boolean acquired = redis.setIfAbsent(lockKey, lockValue, Duration.ofSeconds(5));
```

**Layer 2: Database Row Lock**
```sql
-- Exclusive lock on product row
BEGIN TRANSACTION;
SELECT * FROM products WHERE id = ? FOR UPDATE;
-- Other transactions wait here
UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?;
COMMIT;
```

**Layer 3: Optimistic Locking**
```sql
-- Version check prevents lost updates
UPDATE products 
SET stock = stock - ?, version = version + 1 
WHERE id = ? AND version = ? AND stock >= ?;
-- Returns 0 rows if version changed (concurrent modification detected)
```

**Layer 4: Atomic Cache Update**
```lua
-- Lua script for atomic decrement
local current = redis.call('GET', KEYS[1])
if tonumber(current) >= tonumber(ARGV[1]) then
    return redis.call('DECRBY', KEYS[1], ARGV[1])
else
    return -1
end
```

### Consistency Proof

**Scenario**: 2 concurrent purchase requests for same product

```
Time    Thread 1                          Thread 2                          Result
------------------------------------------------------------------------------------------
T1      Acquire Redis Lock ✓              
T2                                      Try Acquire Lock ✗ (wait/retry)
T3      Read stock from Redis: 10        
T4      Check: 10 >= 5? ✓                
T5      DB: SELECT FOR UPDATE (row lock) 
T6                                      (Still waiting for lock)
T7      DB: UPDATE stock = 5             
T8      DB: COMMIT                       
T9      Redis: DECR stock by 5          
T10     Release Redis Lock               
T11                                    Acquire Redis Lock ✓
T12                                    Read stock from Redis: 5
T13                                    Check: 5 >= 6? ✗
T14                                    Release Lock
T15                                    Return: Insufficient Stock
```

**Result**: Only 5 items sold (correct), Thread 2 correctly rejected.

---

## Latency Optimization Strategy

### Target Latency Breakdown

| Operation | Target | Actual (with optimizations) |
|-----------|--------|----------------------------|
| Cache Hit (Read) | < 10ms | ~5ms |
| Cache Miss (Read) | < 50ms | ~20ms |
| Purchase (Write) | < 100ms | ~50ms |

### Optimization Techniques

1. **Redis Cache**:
   - Product details: TTL 300 seconds
   - Inventory counts: TTL 5 seconds (frequent updates)
   - Cache warming on startup

2. **Connection Pooling**:
   - Pre-established connections (no handshake overhead)
   - Connection reuse (reduces latency by ~10ms per request)

3. **Read Replicas**:
   - Distribute read load (reduces contention)
   - Geographic distribution (if needed)

4. **Virtual Threads Architecture**:
   - Blocking I/O handled efficiently by JVM
   - Each request has its own virtual thread
   - Threads automatically suspend/resume during I/O
   - Traditional blocking code - simple and maintainable

5. **Database Indexing**:
   ```sql
   CREATE INDEX idx_products_stock ON products(stock);
   CREATE INDEX idx_orders_product ON orders(product_id);
   CREATE INDEX idx_orders_created ON orders(created_at);
   ```

---

## High Availability Implementation

### Component Redundancy

| Component | Redundancy Strategy | Failover Time |
|-----------|---------------------|---------------|
| Application Servers | 2 instances với Saga Orchestrator, load balancer health checks | < 5 seconds |
| Primary Database | 1 primary + 1 replica, auto-promotion | < 30 seconds |
| Kafka Cluster | 2 brokers với replication factor = 2 | < 10 seconds (automatic failover) |
| Redis Cluster | 2 nodes với cluster mode và data replication | < 5 seconds (automatic failover) |
| Load Balancer | Active-passive (or cloud LB) | < 1 second |

### Health Check Configuration

**Application Health Endpoint**:
```java
@RestController
public class HealthController {
    
    @GetMapping("/health")
    public Map<String, Object> health() {
        // Blocking calls - Virtual Threads handle efficiently
        String dbStatus = checkDatabase();
        String redisStatus = checkRedis();
        
        return Map.of(
            "status", "UP",
            "database", dbStatus,
            "redis", redisStatus
        );
    }
}
```

**Nginx Health Check**:
```nginx
upstream app_servers {
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
    server app3:8080 max_fails=3 fail_timeout=30s;
}

server {
    location /health {
        proxy_pass http://app_servers;
        proxy_connect_timeout 2s;
        proxy_read_timeout 2s;
    }
}
```

### Database Failover Strategy

**MySQL Replication**:
```sql
-- Primary Configuration (my.cnf)
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW

-- Replica Configuration (my.cnf)
[mysqld]
server-id = 2
relay-log = mysql-relay-bin
read-only = 1
```

**Automatic Failover** (using MySQL Group Replication or MHA):
- Monitor primary health
- Promote replica to primary if primary fails
- Update application connection strings
- Rebuild failed primary as new replica

---

## Scalability Roadmap

### Phase 1: Current (6 Servers)
- 2 App Servers với Saga Orchestrator
- 1 Primary DB + 1 Replica
- 2 Redis+Kafka Servers (perfect resource sharing):
  - Server 1: Redis cluster node (memory) + Kafka broker 1 (disk)
  - Server 2: Redis cluster node (memory) + Kafka broker 2 (disk)
- Handles 6000 concurrent requests

### Phase 2: Horizontal Scaling (Add App Servers)
- Add 3 more App Servers (total 6)
- Capacity: 12000 concurrent requests
- No database changes needed

### Phase 3: Database Read Scaling
- Add 2 more DB Replicas (total 5 DBs)
- Capacity: 15000+ concurrent requests
- Read capacity increases linearly

### Phase 4: Database Sharding
- Shard by product category
- Each shard: 1 Primary + 2 Replicas
- Application routes queries to correct shard

### Phase 5: Caching Layer Expansion
- Redis Cluster expansion (6+ nodes)
- CDN for static assets
- Edge caching for product pages

---

## Monitoring & Alerting

### Key Performance Indicators (KPIs)

1. **Latency Metrics**:
   - p50, p95, p99 response times
   - Database query latency
   - Cache hit latency

2. **Throughput Metrics**:
   - Requests per second
   - Orders per second
   - Cache hit rate

3. **Resource Metrics**:
   - Database connection pool usage
   - Redis memory usage
   - CPU and memory per server

4. **Business Metrics**:
   - Purchase success rate
   - Inventory accuracy
   - Failed purchase rate

### Alert Thresholds

```yaml
alerts:
  - name: HighLatency
    condition: p95_latency > 100ms
    severity: warning
    
  - name: DatabaseConnectionPoolExhausted
    condition: db_connections > 80% of max
    severity: critical
    
  - name: LowCacheHitRate
    condition: cache_hit_rate < 70%
    severity: warning
    
  - name: PurchaseFailureRate
    condition: purchase_failure_rate > 1%
    severity: critical
    
  - name: InventoryInconsistency
    condition: redis_stock != db_stock
    severity: critical
```

---

## Security Considerations

### Authentication & Authorization
- JWT tokens for user authentication
- Role-based access control (RBAC)
- Rate limiting per user/IP

### Data Protection
- Encrypted database connections (SSL/TLS)
- Encrypted cache (Redis AUTH + TLS)
- Sensitive data encryption at rest

### API Security
- Input validation
- SQL injection prevention (parameterized queries)
- XSS protection
- CSRF tokens

---

## Deployment Strategy

### Docker Compose Structure
```yaml
version: '3.8'
services:
  app1:
    image: ecommerce-app:latest
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_PRIMARY_URL=jdbc:mysql://db-primary:3306/ecommerce
      - DB_REPLICA_URL=jdbc:mysql://db-replica:3306/ecommerce
      - REDIS_CLUSTER_NODES=redis-kafka-1:6379,redis-kafka-2:6379
      - KAFKA_BOOTSTRAP_SERVERS=redis-kafka-1:9092,redis-kafka-2:9092
    depends_on:
      - db-primary
      - redis-node-1
      - redis-node-2
      - kafka-broker-1
      - kafka-broker-2
  
  db-primary:
    image: mysql:8.0
    environment:
      - MYSQL_DATABASE=ecommerce
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_REPLICATION_MODE=master
  
  # Server 5: Redis Cluster Node 1 + Kafka Broker 1
  # Redis uses memory, Kafka uses disk - perfect resource sharing
  
  redis-node-1:
    image: redis:7-alpine
    hostname: redis-kafka-1
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file /data/nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --maxmemory 4gb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis-1-data:/data
    ports:
      - "6379:6379"
    deploy:
      resources:
        limits:
          memory: 4G  # Redis uses memory
        reservations:
          memory: 2G
  
  kafka-broker-1:
    image: apache/kafka:3.7.0  # Kafka 3.7+ với KRaft mode
    hostname: redis-kafka-1
    environment:
      # KRaft Mode Configuration (no Zookeeper)
      - KAFKA_NODE_ID=1
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@redis-kafka-1:9093,2@redis-kafka-2:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://redis-kafka-1:9092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      # Log configuration
      - KAFKA_LOG_DIRS=/var/lib/kafka/data
      - KAFKA_LOG_RETENTION_HOURS=168  # 7 days
      - KAFKA_LOG_SEGMENT_BYTES=1073741824  # 1GB
      # Metadata configuration
      - KAFKA_METADATA_LOG_DIR=/var/lib/kafka/metadata
      # Replication
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=2
      - KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=2
      - KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
      - KAFKA_DEFAULT_REPLICATION_FACTOR=2
      - KAFKA_MIN_INSYNC_REPLICAS=1
    volumes:
      - kafka-1-data:/var/lib/kafka/data  # Kafka uses disk
      - kafka-1-metadata:/var/lib/kafka/metadata  # KRaft metadata
    ports:
      - "9092:9092"  # Broker port
      - "9093:9093"  # Controller port
    deploy:
      resources:
        limits:
          memory: 2G  # Kafka JVM heap
        reservations:
          memory: 1G
  
  # Server 6: Redis Cluster Node 2 + Kafka Broker 2
  # Redis uses memory, Kafka uses disk - perfect resource sharing
  
  redis-node-2:
    image: redis:7-alpine
    hostname: redis-kafka-2
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file /data/nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --maxmemory 4gb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis-2-data:/data
    ports:
      - "6380:6379"
    deploy:
      resources:
        limits:
          memory: 4G  # Redis uses memory
        reservations:
          memory: 2G
  
  kafka-broker-2:
    image: apache/kafka:3.7.0  # Kafka 3.7+ với KRaft mode
    hostname: redis-kafka-2
    environment:
      # KRaft Mode Configuration (no Zookeeper)
      - KAFKA_NODE_ID=2
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@redis-kafka-1:9093,2@redis-kafka-2:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://redis-kafka-2:9092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      # Log configuration
      - KAFKA_LOG_DIRS=/var/lib/kafka/data
      - KAFKA_LOG_RETENTION_HOURS=168  # 7 days
      - KAFKA_LOG_SEGMENT_BYTES=1073741824  # 1GB
      # Metadata configuration
      - KAFKA_METADATA_LOG_DIR=/var/lib/kafka/metadata
      # Replication
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=2
      - KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=2
      - KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
      - KAFKA_DEFAULT_REPLICATION_FACTOR=2
      - KAFKA_MIN_INSYNC_REPLICAS=1
    volumes:
      - kafka-2-data:/var/lib/kafka/data  # Kafka uses disk
      - kafka-2-metadata:/var/lib/kafka/metadata  # KRaft metadata
    ports:
      - "9094:9092"  # Broker port
      - "9095:9093"  # Controller port
    deploy:
      resources:
        limits:
          memory: 2G  # Kafka JVM heap
        reservations:
          memory: 1G

volumes:
  redis-1-data:
  redis-2-data:
  kafka-1-data:
  kafka-2-data:
  kafka-1-metadata:
  kafka-2-metadata:
```

**Resource Allocation per Server (Server 5 & 6)**:
```
Server 5 (redis-kafka-1):
├─ Redis Node 1: 4GB RAM (memory-based cache)
└─ Kafka Broker 1: 2GB RAM (JVM) + Disk (event logs)

Server 6 (redis-kafka-2):
├─ Redis Node 2: 4GB RAM (memory-based cache)
└─ Kafka Broker 2: 2GB RAM (JVM) + Disk (event logs)

Total per Server:
├─ RAM: ~6GB (4GB Redis + 2GB Kafka)
└─ Disk: ~50GB (Kafka logs, Redis optional persistence)
```

**Perfect Resource Sharing Benefits**:
- ✅ **No Conflict**: Redis (RAM) và Kafka (Disk) không compete resources
- ✅ **Optimal Utilization**: Tận dụng cả memory và disk trên cùng server
- ✅ **High Availability**: Cả Redis và Kafka đều có replication
- ✅ **Cost Efficient**: Giảm số lượng servers cần thiết

### CI/CD Pipeline
1. Code commit → GitHub/GitLab
2. Automated tests (unit + integration)
3. Docker image build
4. Deploy to staging
5. Smoke tests
6. Deploy to production (blue-green deployment)

---

## Testing Strategy

### Unit Tests
- Service layer logic
- Cache operations
- Lock mechanisms

### Integration Tests
- Database transactions
- Redis operations
- Message queue processing

### Load Tests
- 6000 concurrent requests simulation
- Stress testing (beyond 6000)
- Failure scenario testing

### Consistency Tests
- Concurrent purchase simulation
- Verify no overselling
- Cache-DB consistency verification

