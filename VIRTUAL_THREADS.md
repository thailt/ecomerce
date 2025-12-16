# Virtual Threads vs Reactive Programming (Mono/WebFlux)

## Tại sao hiện tại sử dụng Mono/WebFlux?

### Lý do ban đầu (trước Java 21)

1. **Thread Pool Limitation**:
   - Traditional threads: 1 thread = 1 request
   - Với 6000 concurrent requests cần ~6000 threads
   - OS threads tốn nhiều memory (~1MB/thread stack)
   - Context switching overhead lớn

2. **Blocking I/O Problem**:
   - Blocking database calls giữ thread trong thời gian chờ
   - Thread không thể xử lý request khác khi đang chờ DB response
   - Cần nhiều threads để handle concurrent requests

3. **Reactive Solution**:
   - Non-blocking I/O: Thread không bị block khi chờ I/O
   - 1 thread có thể handle nhiều requests (request multiplexing)
   - Chỉ cần vài threads (thường = số CPU cores) để handle hàng nghìn requests
   - Memory efficient: Không cần stack cho mỗi request

### Kiến trúc hiện tại với WebFlux

```
Request → WebFlux (Event Loop) → Non-blocking I/O
                                    ↓
                            R2DBC (Reactive DB Driver)
                                    ↓
                            Redis Reactive Client
                                    ↓
                            Response (Mono/Flux)
```

**Ưu điểm**:
- ✅ Hiệu quả với I/O-bound operations
- ✅ Memory efficient (ít threads)
- ✅ Request multiplexing tự động

**Nhược điểm**:
- ❌ Learning curve cao (reactive programming)
- ❌ Code phức tạp hơn (Mono/Flux chains)
- ❌ Debugging khó hơn
- ❌ Phải dùng reactive libraries (R2DBC, ReactiveRedisTemplate)

---

## Virtual Threads (Java 21) - Giải pháp mới

### Virtual Threads là gì?

- **Lightweight threads**: Được quản lý bởi JVM, không phải OS
- **Memory efficient**: Chỉ tốn vài KB/thread (thay vì 1MB)
- **Có thể tạo hàng triệu virtual threads**
- **Blocking I/O được xử lý hiệu quả**: JVM tự động suspend/resume

### So sánh

| Aspect | Traditional Threads | Reactive (WebFlux) | Virtual Threads |
|--------|---------------------|-------------------|-----------------|
| Threads/Request | 1:1 | Nhiều requests/thread | 1:1 (nhưng lightweight) |
| Memory/Thread | ~1MB | ~1MB (cho event loop) | ~KB |
| Max Concurrent | ~1000-10000 | Hàng triệu | Hàng triệu |
| Code Complexity | Đơn giản | Phức tạp (Mono/Flux) | Đơn giản |
| Blocking I/O | Vấn đề | Không vấn đề | Không vấn đề |
| Libraries | JDBC, blocking Redis | R2DBC, reactive Redis | JDBC, blocking Redis |

---

## Có thể chuyển sang Virtual Threads không?

### ✅ CÓ THỂ - Và có nhiều lợi ích!

### Lợi ích khi chuyển sang Virtual Threads:

1. **Code đơn giản hơn**:
   ```java
   // Reactive (hiện tại)
   public Mono<PurchaseResult> purchaseProduct(Long productId, Integer quantity) {
       return inventoryLockService.tryLock(productId, Duration.ofSeconds(5))
           .flatMap(locked -> {
               if (!locked) {
                   return Mono.just(PurchaseResult.failed("..."));
               }
               return checkInventory(productId, quantity)
                   .flatMap(available -> {
                       // ... nested chains
                   });
           });
   }
   
   // Virtual Threads (đơn giản hơn)
   public PurchaseResult purchaseProduct(Long productId, Integer quantity) {
       if (!inventoryLockService.tryLock(productId, Duration.ofSeconds(5))) {
           return PurchaseResult.failed("...");
       }
       try {
           int available = checkInventory(productId, quantity);
           if (available < quantity) {
               return PurchaseResult.failed("Insufficient stock");
           }
           return processPurchase(productId, quantity);
       } finally {
           inventoryLockService.releaseLock(productId);
       }
   }
   ```

2. **Debugging dễ hơn**: Stack traces rõ ràng, không có reactive chains

3. **Sử dụng blocking libraries**: JDBC, blocking Redis client (phổ biến hơn)

4. **Performance tương đương**: Virtual threads xử lý blocking I/O hiệu quả như non-blocking

---

## Kiến trúc với Virtual Threads

### Technology Stack mới:

```yaml
Application Layer:
  - Spring Boot 3.x (Java 21)
  - Spring MVC (thay vì WebFlux)
  - Virtual Threads enabled
  - JDBC (thay vì R2DBC)
  - Blocking Redis Client (thay vì ReactiveRedisTemplate)
```

### Configuration:

```yaml
spring:
  threads:
    virtual:
      enabled: true  # Enable Virtual Threads
  
  datasource:
    url: jdbc:mysql://db-primary:3306/ecommerce
    hikari:
      maximum-pool-size: 300
      minimum-idle: 10
  
  data:
    redis:
      host: redis-1
      port: 6379
      lettuce:
        pool:
          max-active: 200
```

### Code Example với Virtual Threads:

```java
@Service
public class PurchaseService {
    
    private final InventoryLockService inventoryLockService;
    private final JdbcTemplate jdbcTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    
    @Async("virtualThreadExecutor")
    public CompletableFuture<PurchaseResult> purchaseProduct(
            Long productId, Integer quantity) {
        
        // Blocking calls - nhưng Virtual Threads xử lý hiệu quả
        if (!inventoryLockService.tryLock(productId, Duration.ofSeconds(5))) {
            return CompletableFuture.completedFuture(
                PurchaseResult.failed("Product is being processed"));
        }
        
        try {
            // Blocking Redis call
            String stockStr = redisTemplate.opsForValue()
                .get("product:" + productId + ":stock");
            int stock = Integer.parseInt(stockStr);
            
            if (stock < quantity) {
                return CompletableFuture.completedFuture(
                    PurchaseResult.failed("Insufficient stock"));
            }
            
            // Blocking JDBC call
            jdbcTemplate.update(
                "UPDATE products SET stock = stock - ?, version = version + 1 " +
                "WHERE id = ? AND stock >= ? AND version = ?",
                quantity, productId, quantity, currentVersion);
            
            // Blocking Redis update
            redisTemplate.opsForValue().decrement(
                "product:" + productId + ":stock", quantity);
            
            return CompletableFuture.completedFuture(
                PurchaseResult.success());
                
        } finally {
            inventoryLockService.releaseLock(productId);
        }
    }
}
```

---

## Performance Comparison

### Với 6000 concurrent requests:

**Reactive (WebFlux)**:
- Threads: ~16-32 (event loop threads)
- Memory: ~32MB (thread stacks)
- Request multiplexing: Tự động
- Code complexity: Cao

**Virtual Threads**:
- Threads: ~6000 virtual threads
- Memory: ~60MB (100KB × 6000 threads)
- Request multiplexing: Không cần (mỗi request có thread riêng)
- Code complexity: Thấp

**Kết luận**: Performance tương đương, nhưng Virtual Threads đơn giản hơn!

---

## Migration Strategy

### Bước 1: Enable Virtual Threads

```java
@Configuration
public class VirtualThreadConfig {
    
    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}
```

### Bước 2: Thay đổi dependencies

```xml
<!-- Thay đổi từ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-mysql</artifactId>
</dependency>

<!-- Thành -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

### Bước 3: Refactor code

- Thay `Mono<T>` → `T` hoặc `CompletableFuture<T>`
- Thay `ReactiveRedisTemplate` → `RedisTemplate`
- Thay `R2DBC` → `JDBC` hoặc `JdbcTemplate`
- Loại bỏ `.flatMap()`, `.map()`, `.then()` chains

---

## Kết luận

### Nên chuyển sang Virtual Threads nếu:

✅ Muốn code đơn giản hơn, dễ maintain
✅ Team không quen với reactive programming
✅ Muốn sử dụng blocking libraries phổ biến (JDBC, blocking Redis)
✅ Java 21+ available

### Giữ WebFlux nếu:

✅ Đã có codebase reactive lớn
✅ Team đã quen với reactive programming
✅ Cần tối ưu memory ở mức cực đại (ít threads hơn)
✅ Đang dùng Java < 21

### Recommendation cho hệ thống này:

**CHUYỂN SANG VIRTUAL THREADS** vì:
1. Java 21 đã available
2. Code sẽ đơn giản hơn nhiều
3. Performance tương đương
4. Dễ maintain và debug hơn
5. Connection pooling vẫn hoạt động tốt với Virtual Threads

---

## Updated Architecture với Virtual Threads

```
Request → Spring MVC → Virtual Thread Pool
                        ↓
                  Blocking I/O (JDBC, Redis)
                        ↓
                  JVM suspends thread
                        ↓
                  I/O completes
                        ↓
                  JVM resumes thread
                        ↓
                  Response
```

**Key Points**:
- Mỗi request có 1 virtual thread riêng
- Blocking I/O không block OS threads
- JVM tự động quản lý thread lifecycle
- Code đơn giản như traditional threading
- Performance như reactive programming

