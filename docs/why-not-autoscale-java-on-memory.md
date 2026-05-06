# Why You Should Not Autoscale Java on Memory

[< Back to Main Guide](../java-memory-k8s-guide.md)

---

## The Short Answer

The JVM allocates its maximum heap at startup and **never gives it back to the OS**. From Kubernetes' perspective, memory usage is effectively constant regardless of whether the application is idle or under heavy load. Memory-based autoscaling is meaningless for Java.

---

## How the JVM Uses Memory (vs What K8s Sees)

### A Typical Non-Java Application (Node.js, Go, Python)

```mermaid
graph LR
    subgraph Memory["Memory Over Time"]
        direction TB
        Low["Idle: 200MB RSS"]
        Med["Normal: 500MB RSS"]
        High["Peak: 1.2GB RSS"]
    end

    Low -->|"Traffic increases"| Med
    Med -->|"Traffic spike"| High
    High -->|"Traffic drops"| Low

    style Low fill:#4CAF50,color:#fff
    style High fill:#F44336,color:#fff
```

Memory tracks load. HPA on memory makes sense — scale up when memory rises, scale down when it falls.

### A Java Application

```mermaid
graph LR
    subgraph Memory["Memory Over Time"]
        direction TB
        Start["Startup: JVM reserves -Xmx<br/>RSS jumps to ~6GB"]
        Idle["Idle: RSS stays at ~6GB<br/>(heap allocated, mostly empty)"]
        Load["Under load: RSS stays at ~6GB<br/>(heap fills and GCs, same allocation)"]
        Peak["Peak traffic: RSS stays at ~6GB<br/>(GC works harder, RSS unchanged)"]
    end

    Start --> Idle
    Idle --> Load
    Load --> Peak

    style Start fill:#FF9800,color:#fff
    style Idle fill:#FF9800,color:#fff
    style Load fill:#FF9800,color:#fff
    style Peak fill:#FF9800,color:#fff
```

Memory is **flat**. The JVM requested its full heap at startup. It doesn't grow or shrink with traffic. HPA on memory sees a flatline.

---

## What Actually Happens with Memory-Based HPA

### Scenario 1: HPA Never Triggers

```mermaid
sequenceDiagram
    participant HPA
    participant Pods
    participant Users

    Note over Pods: 3 pods, each using ~75% of memory limit<br/>(normal for Java — heap is 75% of container)

    HPA->>Pods: Check memory utilization
    Note over HPA: Memory: 75% (target: 80%)
    HPA->>HPA: No scaling needed

    Users->>Pods: Traffic doubles
    Note over Pods: GC works harder<br/>CPU increases<br/>Latency increases<br/>But RSS stays at 75%

    HPA->>Pods: Check memory utilization
    Note over HPA: Memory: 75% (target: 80%)
    HPA->>HPA: Still no scaling needed

    Note over Users: Users experiencing slow responses<br/>but HPA doesn't react
```

**Result:** The application is overloaded, but memory hasn't changed, so HPA does nothing. Users suffer.

### Scenario 2: HPA Triggers Immediately and Scales to Max

```yaml
# If you set a low memory target:
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50    # Target 50% memory usage
```

```mermaid
sequenceDiagram
    participant HPA
    participant Pods

    Note over Pods: Pod starts, JVM allocates heap
    Note over Pods: RSS immediately at 75% of limit

    HPA->>Pods: Check memory: 75%
    Note over HPA: 75% > 50% target → SCALE UP
    HPA->>Pods: Scale 3 → 6 pods

    Note over Pods: New pods start, also at 75%
    HPA->>Pods: Check memory: still 75%
    Note over HPA: Still > 50% target → SCALE UP
    HPA->>Pods: Scale 6 → 12 pods

    Note over HPA: Continues until maxReplicas
    HPA->>Pods: Scale to 15 (max)

    Note over Pods: 15 pods all at 75% memory<br/>Cluster resources wasted<br/>DB connection pool exhausted
```

**Result:** HPA scales to maximum immediately on startup, wasting cluster resources and potentially exhausting database connections. Memory never drops below 75% because that's how the JVM works.

### Scenario 3: HPA Oscillation

If memory happens to hover near the target threshold (due to GC cycles briefly freeing heap):

```mermaid
sequenceDiagram
    participant HPA
    participant Pods

    Note over Pods: GC runs → RSS briefly dips
    HPA->>Pods: Memory: 68% → scale down
    HPA->>Pods: Remove 1 pod

    Note over Pods: Surviving pods serve more → GC pressure increases
    HPA->>Pods: Memory: 82% → scale up
    HPA->>Pods: Add 2 pods

    Note over Pods: JVM startup on new pods → more memory
    HPA->>Pods: Memory: 71% → scale down
    
    Note over HPA: Endless oscillation<br/>Pods constantly starting and stopping<br/>Service stability destroyed
```

**Result:** Constant churn. Pods starting and stopping, never reaching steady state. Each restart loses JIT warmup, making performance worse.

---

## The Root Cause: JVM Memory Lifecycle

```mermaid
flowchart TD
    subgraph JVMLifecycle["JVM Memory Lifecycle"]
        Startup["1. JVM starts<br/>Requests -Xms from OS<br/>(e.g., 6GB)"]
        Allocate["2. Objects created on heap<br/>Heap usage grows"]
        GC["3. GC runs<br/>Reclaims dead objects<br/>Heap usage drops"]
        Retain["4. But RSS stays high<br/>JVM does NOT return<br/>memory to OS"]
        Loop["5. Repeat 2-4 forever<br/>RSS is constant"]
    end

    Startup --> Allocate --> GC --> Retain --> Loop
    Loop --> Allocate

    style Retain fill:#F44336,color:#fff
```

**Key behavior:** When the GC frees heap memory, it remains **committed** — the JVM keeps it for future allocations. The OS (and therefore Kubernetes) still sees it as used.

This is by design:
- Re-requesting memory from the OS is expensive
- The JVM assumes it will need the memory again
- `-Xms == -Xmx` (recommended for containers) means full allocation at startup

### Can the JVM Return Memory?

Some GCs can return memory to the OS, but it's unreliable for autoscaling:

| GC | Can Return Memory? | Flag | Practical for HPA? |
|:---|:-------------------|:-----|:-------------------|
| G1GC | Yes (Java 12+) | `-XX:G1PeriodicGCInterval=N` | No — too slow, unpredictable |
| ZGC | Yes (Java 13+) | `-XX:ZUncommitDelay=N` | No — aggressive uncommit hurts performance |
| Shenandoah | Yes | `-XX:ShenandoahUncommitDelay=N` | No — same issues |
| SerialGC | No | — | — |
| ParallelGC | No | — | — |

Even with uncommit enabled:
- There's a delay (default 5 minutes for ZGC)
- The JVM only uncommits if heap usage is **very** low
- Under any real load, the heap stays committed
- The uncommit/recommit cycle adds latency
- **It defeats the purpose of `-Xms == -Xmx`**, which is the recommended container configuration

---

## What You Should Scale On Instead

```mermaid
flowchart TD
    subgraph Good["Good HPA Metrics for Java"]
        CPU["CPU Utilization<br/>Target: 70%"]
        RPS["Request Rate<br/>(custom metric)"]
        Latency["p99 Latency<br/>(custom metric)"]
        Queue["Queue Depth<br/>(KEDA / custom)"]
    end

    subgraph Bad["Bad HPA Metrics for Java"]
        Memory["Memory Utilization"]
    end

    CPU -->|"Correlates with load"| Works["Scales with traffic ✓"]
    RPS -->|"Directly measures load"| Works
    Latency -->|"Measures user impact"| Works
    Queue -->|"Measures backpressure"| Works
    Memory -->|"Flat regardless of load"| Broken["Does not work ✗"]

    style Good fill:#E8F5E9
    style Bad fill:#FFEBEE
    style Works fill:#4CAF50,color:#fff
    style Broken fill:#F44336,color:#fff
```

### Why CPU Works

Unlike memory, CPU usage **does** correlate with traffic for Java:
- More requests → more computation → higher CPU
- GC uses more CPU under heap pressure
- JIT compilation uses CPU during warmup
- CPU drops when traffic drops

```yaml
# Recommended: CPU-based HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-spring-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-spring-app
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Why Request Rate Works

Custom metric that directly measures demand:

```yaml
# Scale when RPS per pod exceeds threshold
metrics:
  - type: Pods
    pods:
      metric:
        name: http_server_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

### Why Latency Works

Scale up when user experience degrades:

```yaml
# Scale when p99 exceeds SLO
metrics:
  - type: Pods
    pods:
      metric:
        name: http_server_requests_p99_seconds
      target:
        type: AverageValue
        averageValue: "0.5"    # Scale up when p99 > 500ms
```

### Why Queue Depth Works (Event-Driven)

For Kafka consumers or message processors:

```yaml
# KEDA: scale on queue backlog
triggers:
  - type: kafka
    metadata:
      topic: my-topic
      consumerGroup: my-group
      lagThreshold: "100"    # Scale when consumer lag > 100
```

---

## Multi-Metric Strategy

Use multiple metrics together for best results:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-spring-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-spring-app
  minReplicas: 3
  maxReplicas: 15
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
  metrics:
    # Primary: CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Secondary: request rate
    - type: Pods
      pods:
        metric:
          name: http_server_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

HPA evaluates all metrics and uses whichever recommends the **highest** replica count. This means:
- CPU spike → scales up (even if RPS is low — maybe GC pressure)
- RPS spike → scales up (even if CPU hasn't caught up yet)
- Both low → scales down

---

## "But What About Memory Leaks?"

A common argument for memory-based scaling: "If we scale on memory, we'll automatically add pods when there's a memory leak."

**This doesn't work because:**

1. Memory leaks are **bugs** — autoscaling masks them while the problem compounds
2. New pods also start leaking → eventually all pods are full → cluster exhausted
3. The leak is still there, just running on more pods
4. You've now consumed 5× the cluster resources while the leak continues

```mermaid
flowchart TD
    Leak["Memory leak detected"] --> Scale{Scale on memory?}
    
    Scale -->|"Yes (Bad)"| More["More pods created<br/>Each one also leaks"]
    More --> MoreMore["Even more pods<br/>Cluster memory exhausted"]
    MoreMore --> Crash["Cluster-wide impact<br/>All services affected"]
    
    Scale -->|"No (Good)"| Alert["Alert fires:<br/>heap > 85% sustained"]
    Alert --> Investigate["Team investigates<br/>Finds leak"]
    Investigate --> Fix["Fix deployed<br/>Problem solved"]
    
    style Crash fill:#F44336,color:#fff
    style Fix fill:#4CAF50,color:#fff
```

**The right approach:** Monitor heap percentage with alerts. When heap is sustained > 85%, alert the team to investigate. Fix the leak — don't throw more hardware at it.

---

## When Memory Monitoring IS Useful (Just Not for HPA)

Memory metrics are critical — just not as an autoscaling trigger:

| Use Case | Metric | Purpose |
|:---------|:-------|:--------|
| **Alerting** | `jvm_memory_used_bytes / jvm_memory_max_bytes > 0.85` | Detect heap pressure before OOM |
| **Alerting** | `container_memory_working_set_bytes / limit > 0.90` | Detect container OOMKill risk |
| **VPA recommendations** | VPA `updateMode: "Off"` | Right-size requests/limits over time |
| **Capacity planning** | Historical RSS trends | Plan node pool sizing |
| **Leak detection** | `deriv(jvm_memory_used_bytes[1h]) > 0` | Detect monotonic memory growth |
| **Post-deploy validation** | RSS vs limit gap | Verify new config has headroom |

---

## Summary

```mermaid
flowchart TD
    Question{"Should I autoscale<br/>Java on memory?"}
    Question -->|"No"| Why["JVM memory is constant.<br/>It doesn't reflect load."]
    Why --> Instead["Scale on: CPU, request rate,<br/>latency, or queue depth"]
    Instead --> Monitor["Monitor memory for:<br/>alerting, leak detection,<br/>capacity planning"]
    
    style Question fill:#F44336,color:#fff
    style Instead fill:#4CAF50,color:#fff
```

| Metric | Reflects Load? | Scales with Traffic? | Good for HPA? |
|:-------|:--------------|:--------------------|:---------------|
| **CPU** | Yes | Yes | **Yes** |
| **Request rate** | Yes | Yes | **Yes** |
| **Latency (p99)** | Yes | Inversely | **Yes** |
| **Queue depth** | Yes | Yes | **Yes** |
| **Memory (RSS)** | No | No | **No** |
| **Memory (heap %)** | Partially (GC sawtooth) | Weakly | **No** |

---

## Next Steps

- [Autoscaling](autoscaling.md) — Full HPA/VPA/KEDA configuration
- [Monitoring & Metrics](monitoring-metrics.md) — How to use memory metrics for alerting (not scaling)
- [JVM Memory Model](jvm-memory-model.md) — Why the JVM behaves this way
- [Capacity Planning](capacity-planning.md) — Right-sizing without autoscaling on memory
