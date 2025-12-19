
# Spring-Redis-Cache-Strategy

## Overview

This project is a **multi-microservice demonstration** of advanced Redis caching strategies built with **Spring Boot 3.x**. It simulates a high-traffic e-commerce backend where frequent reads (product details, dynamic pricing, user sessions) heavily benefit from caching to reduce database load and improve latency.

The goal is to showcase real-world caching patterns:
- Cache-Aside
- Time-to-Live (TTL) for volatile data
- LRU eviction for memory management
- Cache warming
- Distributed cache invalidation via Redis Pub/Sub
- Graceful fallback to DB when Redis is unavailable
- Cache metrics and monitoring

## Real-World Scenario

Imagine an e-commerce platform like **Amazon** during Black Friday:
- Millions of users view product pages.
- Product details change infrequently.
- Prices can be dynamic (promotions, user-specific).
- User sessions must be fast but short-lived.

Direct DB hits would overwhelm the database. This project shows how Redis acts as a centralized, high-performance cache across multiple services.

## Microservices Involved

| Service             | Responsibility                                                                 | Port  |
|---------------------|--------------------------------------------------------------------------------|-------|
| **eureka-server**       | Service discovery using Netflix Eureka                                          | 8761  |
| **product-service**     | Manages product catalog (CRUD), caches product data with LRU policy            | 8081  |
| **pricing-service**     | Calculates dynamic prices, caches results with short TTL, calls product-service| 8082  |
| **user-service**        | Manages user sessions and preferences, caches session data with TTL            | 8083  |

All services share a **single Redis instance** as the distributed cache.

## Tech Stack

- Spring Boot 3.x
- Spring Data Redis (Lettuce client)
- Spring Cache Abstraction (`@Cacheable`, `@CacheEvict`, `@CachePut`)
- Spring Cloud Netflix Eureka (service discovery)
- Spring Cloud OpenFeign (inter-service calls)
- PostgreSQL (persistent storage)
- Redis (caching + Pub/Sub)
- Docker & Docker Compose
- Lombok, Micrometer (metrics), Actuator
- Maven (multi-module)

## Docker Containers

```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    command: redis-server --save 60 1 --loglevel warning
    ports:
      - "6379:6379"

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  product-service:
    build: ./product-service
    depends_on:
      - postgres
      - redis
      - eureka-server
    ports:
      - "8081:8081"

  pricing-service:
    build: ./pricing-service
    depends_on:
      - redis
      - eureka-server
      - product-service
    ports:
      - "8082:8082"

  user-service:
    build: ./user-service
    depends_on:
      - redis
      - eureka-server
    ports:
      - "8083:8083"
```

Run with: `docker-compose up --build`

## Caching Strategies Implemented

| Strategy               | Used In             | Details                                                                 |
|------------------------|---------------------|-------------------------------------------------------------------------|
| **Cache-Aside**        | All services        | Application checks Redis first → DB on miss → populate cache             |
| **TTL (Expiration)**   | Pricing & User      | Volatile data expires automatically (e.g., 1 min for prices, 30 min for sessions) |
| **LRU Eviction**       | Product Service     | Configured via Redis `maxmemory-policy allkeys-lru`                     |
| **Cache Warming**      | Product Service     | Pre-load top 100 products on startup                                    |
| **Cache Invalidation** | Product → Pricing   | On product update, publish event to Redis channel → Pricing evicts keys |
| **Fallback**           | All services        | If Redis down → direct DB query with warning log                        |

## Key Features

- Custom cache key generation (e.g., `product::123`)
- Redis Pub/Sub for distributed invalidation
- Cache metrics (hit/miss ratio) via Micrometer + Actuator
- Feign Client with fallback for inter-service resilience
- Structured logging and health checks
- Cache stampede protection via synchronized loading (optional extension)
- Configurable via `application.yml`

## Expected Endpoints

### Product Service (`http://localhost:8081`)

| Method | Endpoint                  | Description                              |
|--------|---------------------------|------------------------------------------|
| GET    | `/api/products/{id}`      | Get product (cached)                     |
| GET    | `/api/products`           | List products (partial caching)          |
| POST   | `/api/products`           | Create product                           |
| PUT    | `/api/products/{id}`      | Update product → triggers invalidation   |
| DELETE | `/api/products/{id}`      | Delete → evict cache                     |
| POST   | `/api/products/warm-cache`| Manually warm cache with popular products|

### Pricing Service (`http://localhost:8082`)

| Method | Endpoint                        | Description                              |
|--------|---------------------------------|------------------------------------------|
| POST   | `/api/prices/calculate`         | Calculate price (cached per product+user)|
| GET    | `/api/prices/actuator/metrics`  | View cache metrics                       |

### User Service (`http://localhost:8083`)

| Method | Endpoint                  | Description                              |
|--------|---------------------------|------------------------------------------|
| GET    | `/api/users/{id}/session` | Get session data (cached with TTL)       |
| PUT    | `/api/users/{id}/prefs`   | Update preferences → evict session cache |

### Actuator (All services)

- `/actuator/health`
- `/actuator/metrics`
- `/actuator/redis` (custom Redis health)
- `/actuator/prometheus` (for Grafana/Prometheus)

## Architecture Overview

```
Clients
   ↓
[Load Balancer / API Gateway (optional)]
   ↓
Eureka Server (Service Discovery)
   ↓
┌─────────────────┬──────────────────────┬─────────────────┐
│ product-service │ pricing-service      │ user-service    │
└─────────────────┴──────────────────────┴─────────────────┘
         ↓              ↓            ↓           ↓
      PostgreSQL     Redis (Cache + Pub/Sub)     Redis
```

**Cache Invalidation Flow**:
1. Product updated in `product-service`
2. Publish message to Redis channel `cache.invalidation.product`
3. `pricing-service` subscriber receives → evicts related price keys

## How to Run

1. Clone the repo
2. Ensure Docker is running
3. Run `docker-compose up --build`
4. Access Eureka dashboard: `http://localhost:8761`
5. Test endpoints with Postman/curl

## Testing Cache Behavior

1. Call `/api/products/1` → First call: slow (DB), subsequent: fast (cache hit)
2. Update product → Pricing cache invalidated
3. Stop Redis container → Services fallback to DB (no crash)

## Skills Demonstrated

- Advanced Redis integration with Spring Boot
- Spring Cache abstraction with custom configurations
- Distributed cache invalidation using Pub/Sub
- Service discovery and resilient inter-service communication
- Observability (metrics, health checks)
- Production-ready patterns: fallback, warming, eviction

## Future Extensions (Optional)

- Redis Cluster for high availability
- Distributed locking for cache warming
- Caffeine as L2 cache
- Integration with Prometheus + Grafana dashboards
