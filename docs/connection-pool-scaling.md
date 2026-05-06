# Connection Pool Scaling

[< Back to Main Guide](../java-memory-k8s-guide.md)

---

## The Hidden Scaling Ceiling

When HPA scales your pods from 3 to 15, each pod opens its own database connection pool. **The database becomes the bottleneck**, not memory.

```mermaid
flowchart TD
    subgraph Pods["Pods (HPA: 3 → 15)"]
        P1["Pod 1<br/>Pool: 20 connections"]
        P2["Pod 2<br/>Pool: 20 connections"]
        P3["Pod 3<br/>Pool: 20 connections"]
        Dots["..."]
        P15["Pod 15<br/>Pool: 20 connections"]
    end

    subgraph DB["PostgreSQL"]
        MaxConn["max_connections: 100"]
        Active["Active: 60 (pods)<br/>Reserved: 10 (superuser)<br/>Available: 30"]
    end

    P1 --> DB
    P2 --> DB
    P3 --> DB
    P15 --> DB

    DB -->|"15 pods × 20 = 300<br/>but DB only allows 100"| Fail["Connection refused!<br/>Pods start failing health checks<br/>→ Cascade failure"]

    style Fail fill:#F44336,color:#fff
    style MaxConn fill:#FF9800,color:#fff
```

---

## The Math

```
Total connections = pods × pool_size_per_pod

Current:  3 pods × 20 connections = 60 total
Scaled:  15 pods × 20 connections = 300 total

PostgreSQL default max_connections: 100
RDS default max_connections: varies by instance size (e.g., db.r5.large = 1000)
```

### Memory Impact of Connections

Connections consume memory on **both sides**:

| Side | Memory Per Connection | Notes |
|:-----|:---------------------|:------|
| **Application (HikariCP)** | ~10-50 KB | Connection object + buffers |
| **PostgreSQL** | ~5-10 MB | `work_mem` + sort buffers per connection |
| **MySQL** | ~1-5 MB | Thread stack + buffers |
| **Connection Pooler (PgBouncer)** | ~2 KB | Lightweight multiplexing |

For the database:
```
15 pods × 20 connections × 10MB (PostgreSQL) = 3 GB of DB memory just for connections
```

---

## Right-Sizing the Pool

### HikariCP Configuration

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # Per pod — not 20!
      minimum-idle: 2              # Keep 2 warm connections
      idle-timeout: 300000         # 5 min idle before closing
      max-lifetime: 1800000        # 30 min max connection age
      connection-timeout: 5000     # 5s wait for connection before error
      leak-detection-threshold: 30000  # Warn if connection held > 30s
      pool-name: "HikariPool-main"
```

### Sizing Formula

```
pool_size_per_pod = max_db_connections / max_pod_replicas - safety_margin

Example:
  DB max_connections:     200
  Reserved (superuser):    10
  Reserved (migrations):    5
  Available for app:      185
  Max HPA replicas:        15
  Safety margin:           10%

  pool_size_per_pod = (185 / 15) × 0.9 ≈ 11

  → Set maximum-pool-size: 10
```

### How Many Connections Do You Actually Need?

```mermaid
graph TD
    Requests["Incoming Requests/sec"] --> Threads["Tomcat Threads<br/>(handling requests)"]
    Threads --> Duration["Average DB Query Duration"]

    Threads -->|"Connections needed =<br/>concurrent_db_queries =<br/>threads_doing_DB_work × avg_query_time"| Pool["Required Pool Size"]

    Example["Example:<br/>50 concurrent requests<br/>60% touch DB<br/>avg query 50ms"] --> Calc["30 × 0.05s = 1.5<br/>concurrent connections needed<br/><br/>But: add headroom for spikes<br/>→ Pool size: 10"]

    style Calc fill:#4CAF50,color:#fff
```

**The key insight:** Pool size should be based on **concurrent database queries**, not request volume. Most requests spend very little time holding a connection.

Reference: [HikariCP Pool Sizing Guide](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

Formula from HikariCP author:
```
connections = ((core_count * 2) + effective_spindle_count)
```
For SSD-backed databases: `connections ≈ core_count * 2 + 1`

A web app with 4 CPU cores: **pool size = 9-10** is often optimal.

---

## Connection Pooler (PgBouncer / ProxySQL)

For high-scale deployments, put a connection pooler between your app and the database:

```mermaid
flowchart LR
    subgraph Pods["15 Pods"]
        P1["Pod 1: 10 conns"]
        P2["Pod 2: 10 conns"]
        P15["Pod 15: 10 conns"]
    end

    subgraph Pooler["PgBouncer"]
        Client["Client connections: 150"]
        Server["Server connections: 30"]
        Mode["Mode: transaction"]
    end

    subgraph DB["PostgreSQL"]
        Max["max_connections: 50<br/>(plenty of headroom)"]
    end

    P1 --> Pooler
    P2 --> Pooler
    P15 --> Pooler
    Pooler --> DB

    style Pooler fill:#4CAF50,color:#fff
```

### PgBouncer in Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: my-namespace
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: pgbouncer
          image: edoburu/pgbouncer:latest
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_URL
              value: "postgres://user:pass@db-host:5432/mydb"
            - name: POOL_MODE
              value: "transaction"          # Multiplex at transaction boundary
            - name: DEFAULT_POOL_SIZE
              value: "30"                   # Server-side connections to DB
            - name: MAX_CLIENT_CONN
              value: "200"                  # Client-side connections from pods
            - name: MAX_DB_CONNECTIONS
              value: "50"                   # Hard cap on DB connections
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer
  namespace: my-namespace
spec:
  selector:
    app: pgbouncer
  ports:
    - port: 5432
      targetPort: 5432
```

### Pool Modes

| Mode | Behavior | Best For |
|:-----|:---------|:---------|
| **transaction** | Connection returned to pool after each transaction | Most web apps (recommended) |
| **session** | Connection held for entire client session | Apps using session-level features (temp tables, SET) |
| **statement** | Connection returned after each statement | Simple queries only (no multi-statement transactions) |

---

## Spring Boot Integration with Pooler

```yaml
# Point app at PgBouncer, not directly at the DB
spring:
  datasource:
    url: jdbc:postgresql://pgbouncer:5432/mydb    # PgBouncer service
    hikari:
      maximum-pool-size: 10
      # IMPORTANT: Disable connection test query when using PgBouncer in transaction mode
      # PgBouncer doesn't support DISCARD ALL or RESET ALL
      connection-test-query: ""
      # Some PgBouncer configurations require these:
      auto-commit: true
```

### Gotchas with PgBouncer + Spring

| Issue | Cause | Fix |
|:------|:------|:----|
| "prepared statement does not exist" | PgBouncer in transaction mode can't track prepared statements across connections | Use `prepareThreshold=0` in JDBC URL or PgBouncer with `max_prepared_statements` |
| SET commands lost | Transaction mode resets session on return | Avoid session-level SET; use transaction-level settings |
| LISTEN/NOTIFY broken | Requires persistent connection | Use session mode for specific service, or skip PgBouncer for that connection |
| Advisory locks don't work | Lock held on server connection, which changes | Use session mode or avoid advisory locks |

JDBC URL with PgBouncer compatibility:
```
jdbc:postgresql://pgbouncer:5432/mydb?prepareThreshold=0&preparedStatementCacheQueries=0
```

---

## Monitoring Connection Pools

### HikariCP Metrics (via Micrometer)

```yaml
# These are exposed automatically when using Spring Boot Actuator + Micrometer
# Available at /actuator/prometheus
```

| Metric | What It Tells You | Alert When |
|:-------|:------------------|:-----------|
| `hikaricp_connections_active` | Currently in-use connections | Approaching pool max |
| `hikaricp_connections_idle` | Available connections | Dropping to 0 |
| `hikaricp_connections_pending` | Threads waiting for a connection | > 0 for extended periods |
| `hikaricp_connections_timeout_total` | Connection acquisition timeouts | Any increase |
| `hikaricp_connections_max` | Pool max size | Should match config |
| `hikaricp_connections_usage_seconds` | How long connections are held | Increasing = slow queries or leaks |

### Prometheus Alert Rules

```yaml
groups:
  - name: connection-pool-alerts
    rules:
      - alert: ConnectionPoolExhausted
        expr: |
          hikaricp_connections_pending{application="my-spring-app"} > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Threads waiting for DB connections in {{ $labels.pod }}"

      - alert: ConnectionPoolNearMax
        expr: |
          hikaricp_connections_active{application="my-spring-app"}
          / hikaricp_connections_max{application="my-spring-app"}
          > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connection pool at >90% capacity in {{ $labels.pod }}"

      - alert: ConnectionTimeouts
        expr: |
          increase(hikaricp_connections_timeout_total{application="my-spring-app"}[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Connection acquisition timeouts occurring in {{ $labels.pod }}"

      - alert: SlowConnectionUsage
        expr: |
          hikaricp_connections_usage_seconds_max{application="my-spring-app"} > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connections held >30s — possible leak in {{ $labels.pod }}"
```

---

## Scaling Scenarios

### Scenario: HPA Scales Up, DB Can't Handle It

```mermaid
sequenceDiagram
    participant HPA
    participant Pods
    participant DB

    Note over Pods: 3 pods × 10 conn = 30
    HPA->>Pods: Scale to 10 pods
    Pods->>DB: 10 × 10 = 100 new connections requested
    DB-->>Pods: max_connections reached!
    Pods->>Pods: Connection timeout → health check fails
    Note over Pods: Pods marked unhealthy → restarted
    Note over Pods: CASCADING FAILURE
```

**Prevention:**
1. Set `maximum-pool-size` conservatively
2. Use `connection-timeout` to fail fast (5s) rather than hang
3. Set HPA `maxReplicas` based on DB capacity, not just CPU
4. Use a connection pooler for high-scale environments

### Scenario: Pod Crash Leaks Connections

When a pod is killed (OOMKill, node failure), its DB connections may not be closed cleanly:

```bash
# Check for leaked connections on PostgreSQL:
SELECT count(*) as idle_connections,
       client_addr,
       state
FROM pg_stat_activity
WHERE datname = 'mydb'
  AND state = 'idle'
  AND state_change < now() - interval '10 minutes'
GROUP BY client_addr, state;
```

**Prevention:**
```yaml
spring:
  datasource:
    hikari:
      max-lifetime: 1800000    # Recycle connections every 30min
      # This ensures leaked connections eventually time out on DB side too
```

On PostgreSQL:
```sql
-- Kill connections idle for more than 30 minutes
ALTER SYSTEM SET idle_in_transaction_session_timeout = '30min';
```

---

## Redis / External Service Pools

The same pattern applies to all connection pools — Redis, HTTP clients, gRPC channels:

### Redis (Lettuce — Spring Boot Default)

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 16         # Max connections per pod
          max-idle: 8
          min-idle: 2
          max-wait: 5000ms       # Fail fast
```

Total Redis connections: `15 pods × 16 = 240` — verify Redis `maxclients` (default 10000, usually fine).

### HTTP Client Pools (RestTemplate / WebClient)

```java
@Bean
public RestTemplate restTemplate() {
    var factory = new HttpComponentsClientHttpRequestFactory();
    var connManager = PoolingHttpClientConnectionManager.custom()
        .setMaxConnTotal(50)         // Total connections across all hosts
        .setMaxConnPerRoute(20)      // Per downstream service
        .build();
    factory.setHttpClient(HttpClients.custom()
        .setConnectionManager(connManager)
        .build());
    return new RestTemplate(factory);
}
```

---

## Decision Guide

```mermaid
flowchart TD
    Start{How many pods<br/>at max scale?} -->|"< 5 pods"| Direct["Direct DB connection<br/>Pool: 10-20 per pod"]
    Start -->|"5-20 pods"| Consider["Consider PgBouncer<br/>Pool: 5-10 per pod"]
    Start -->|"20+ pods"| Required["PgBouncer required<br/>Pool: 3-5 per pod"]

    Direct --> Monitor["Monitor:<br/>hikaricp_connections_pending"]
    Consider --> Monitor
    Required --> Monitor

    Monitor --> Alert{Pending > 0<br/>sustained?}
    Alert -->|"Yes"| Fix["Either:<br/>1. Increase pool size<br/>2. Add PgBouncer<br/>3. Optimize queries<br/>4. Limit HPA max"]

    style Required fill:#F44336,color:#fff
    style Fix fill:#FF9800,color:#fff
```

---

## Checklist

- [ ] `maximum-pool-size` calculated based on: `DB_max_connections / max_HPA_replicas`
- [ ] `connection-timeout` set to fail fast (5s) rather than queue indefinitely
- [ ] `max-lifetime` set to recycle connections (prevents stale/leaked connections)
- [ ] `leak-detection-threshold` enabled for non-production environments
- [ ] HPA `maxReplicas` accounts for DB connection capacity
- [ ] DB monitoring shows connection count vs max
- [ ] PgBouncer considered if max pods > 10
- [ ] Redis/HTTP client pools also bounded

---

## Next Steps

- [Autoscaling](autoscaling.md) — Setting maxReplicas based on connection limits
- [Monitoring & Metrics](monitoring-metrics.md) — Connection pool metrics
- [Spring Boot Memory Traps](spring-boot-memory-traps.md) — Other Spring-specific issues
