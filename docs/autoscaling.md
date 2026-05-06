# Autoscaling

[< Back to Main Guide](../java-memory-k8s-guide.md)

---

## Autoscaling Strategy for Java

```mermaid
flowchart TD
    subgraph Scaling["Scaling Approaches"]
        HPA["Horizontal Pod Autoscaler<br/>(add more pods)"]
        VPA["Vertical Pod Autoscaler<br/>(resize existing pods)"]
        KEDA["KEDA<br/>(event-driven scaling)"]
    end

    HPA -->|"Best for"| HPAUse["Stateless web apps<br/>Scale on CPU or request rate"]
    VPA -->|"Best for"| VPAUse["Right-sizing recommendations<br/>NOT auto-resize for Java"]
    KEDA -->|"Best for"| KEDAUse["Queue-based workloads<br/>Custom metric triggers"]

    style HPA fill:#4CAF50,color:#fff
    style VPA fill:#FF9800,color:#fff
    style KEDA fill:#2196F3,color:#fff
```

---

## Critical Rule: Do NOT Autoscale Java on Memory

```mermaid
graph LR
    subgraph JVM["JVM Memory Over Time"]
        direction TB
        Start["Startup: allocates -Xmx immediately"]
        Steady["Steady state: heap appears ~75% full always"]
        GC["GC cycles: sawtooth within allocated heap"]
    end

    JVM -->|"Memory appears constant"| Problem["HPA sees static memory<br/>Never triggers OR always triggers"]

    style Problem fill:#F44336,color:#fff
```

**Why:** The JVM requests its full heap at startup via `-Xms == -Xmx`. From Kubernetes' perspective, memory usage is effectively constant regardless of traffic. Memory-based HPA is meaningless for Java.

**Scale on:**
- CPU utilization
- HTTP request rate
- Request latency (p99)
- Queue depth (for async workers)
- Custom business metrics

---

## Horizontal Pod Autoscaler (HPA)

### Basic CPU-Based HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-spring-app-hpa
  namespace: my-namespace
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

### Advanced: Multi-Metric HPA with Behavior Controls

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-spring-app-hpa
  namespace: my-namespace
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-spring-app
  minReplicas: 3
  maxReplicas: 15

  # Scale-up and scale-down behavior
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60        # Wait 60s before scaling up
      policies:
        - type: Pods
          value: 2                           # Add max 2 pods per period
          periodSeconds: 60
        - type: Percent
          value: 50                          # Or 50% of current, whichever is less
          periodSeconds: 60
      selectPolicy: Min                      # Use the more conservative policy

    scaleDown:
      stabilizationWindowSeconds: 300        # Wait 5min before scaling down
      policies:
        - type: Pods
          value: 1                           # Remove max 1 pod per period
          periodSeconds: 120
      selectPolicy: Min

  metrics:
    # CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Custom metric: request rate per pod (requires Prometheus Adapter)
    - type: Pods
      pods:
        metric:
          name: http_server_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"

    # Custom metric: p99 latency (scale up when latency degrades)
    - type: Pods
      pods:
        metric:
          name: http_server_requests_p99_seconds
        target:
          type: AverageValue
          averageValue: "0.5"   # Scale up when p99 > 500ms
```

### Why Conservative Scale-Down for Java

```mermaid
sequenceDiagram
    participant HPA
    participant Deploy as Deployment
    participant NewPod as New Pod
    participant Traffic

    Note over HPA: CPU > 70% for 60s
    HPA->>Deploy: Scale 3 → 5 pods
    Deploy->>NewPod: Start new pods
    Note over NewPod: JVM startup: 30-90s<br/>JIT warmup: 2-5min
    NewPod-->>Traffic: Slow responses during warmup

    Note over HPA: CPU drops below 70%
    Note over HPA: Wait 300s (stabilization)
    HPA->>Deploy: Scale 5 → 4 pods
    Note over HPA: Wait 120s (period)
    HPA->>Deploy: Scale 4 → 3 pods
```

Java apps are **expensive to start** (30-90s startup, 2-5 min JIT warmup). Aggressive scale-down causes:
- Wasted startup cost if traffic rebounds
- Brief latency spike as surviving pods absorb traffic before the removed pod fully drains

---

## Prometheus Adapter (Required for Custom Metrics)

To use custom metrics in HPA, install the Prometheus Adapter:

```yaml
# prometheus-adapter-config.yaml
rules:
  - seriesQuery: 'http_server_requests_seconds_count{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_seconds_count$"
      as: "${1}_per_second"
    metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'

  - seriesQuery: 'http_server_requests_seconds_bucket{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_seconds_bucket$"
      as: "${1}_p99_seconds"
    metricsQuery: 'histogram_quantile(0.99, rate(<<.Series>>{<<.LabelMatchers>>}[2m]))'
```

Verify custom metrics are available:
```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/my-namespace/pods/*/http_server_requests_per_second"
```

---

## Vertical Pod Autoscaler (VPA)

### Use VPA for Recommendations Only

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-spring-app-vpa
  namespace: my-namespace
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-spring-app
  updatePolicy:
    updateMode: "Off"    # IMPORTANT: recommendations only, no auto-resize
  resourcePolicy:
    containerPolicies:
      - containerName: my-spring-app
        minAllowed:
          memory: "4Gi"
          cpu: "1000m"
        maxAllowed:
          memory: "16Gi"
          cpu: "8000m"
        controlledResources: ["cpu", "memory"]
```

### Why NOT `updateMode: "Auto"` for Java

| Mode | Behavior | Java Impact |
|:-----|:---------|:------------|
| `Off` | Recommendations only | Safe — use to inform manual decisions |
| `Initial` | Set resources only on pod creation | Acceptable for new pods |
| `Auto` | **Restarts pods to resize** | Disruptive — kills running JVMs |

VPA `Auto` mode evicts and recreates pods to apply new resource values. For Java apps with 60-90s startup times, this causes:
- Service disruption during restarts
- Lost JIT-compiled code (warmup penalty)
- Potential cascade if multiple pods restart simultaneously

**Best practice:** Use `updateMode: "Off"`, review VPA recommendations periodically, and apply changes via Helm/Kustomize in planned deployments.

```bash
# Check VPA recommendations
kubectl describe vpa my-spring-app-vpa -n my-namespace
```

---

## KEDA (Event-Driven Autoscaling)

For workloads driven by queues or custom event sources:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-spring-app-keda
  namespace: my-namespace
spec:
  scaleTargetRef:
    name: my-spring-app
  minReplicaCount: 3
  maxReplicaCount: 15
  cooldownPeriod: 300             # 5min before scaling down
  pollingInterval: 15             # Check every 15s
  triggers:
    # Scale based on Prometheus query
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: http_requests_total
        query: >-
          sum(rate(http_server_requests_seconds_count{app="my-spring-app"}[2m]))
        threshold: "100"          # Scale up when total RPS > 100
        activationThreshold: "10" # Only activate scaler above 10 RPS

    # Scale based on RabbitMQ queue depth
    - type: rabbitmq
      metadata:
        host: amqp://guest:guest@rabbitmq.default:5672/
        queueName: my-queue
        queueLength: "50"         # Scale when queue > 50 messages
```

---

## HPA + VPA Interaction

```mermaid
flowchart TD
    Warning["WARNING: Never use HPA and VPA<br/>on the SAME metric dimension"]

    HPA_CPU["HPA scales on CPU"] --- VPA_MEM["VPA recommends on memory"]
    HPA_CPU -->|"OK"| Safe["Safe: different dimensions"]

    HPA_CPU2["HPA scales on CPU"] --- VPA_CPU["VPA adjusts CPU limits"]
    HPA_CPU2 -->|"CONFLICT"| Loop["Infinite scaling loop"]

    style Warning fill:#F44336,color:#fff
    style Safe fill:#4CAF50,color:#fff
    style Loop fill:#F44336,color:#fff
```

**Safe combinations:**
- HPA on CPU/request-rate + VPA on memory only (with `updateMode: "Off"`)
- HPA + no VPA (most common)
- VPA only (for non-horizontally-scalable workloads)

---

## Capacity Planning for Autoscaling

Before enabling autoscaling, verify your cluster can handle the maximum:

```bash
# Maximum memory needed: maxReplicas × limits.memory
# Example: 15 pods × 8Gi = 120Gi

# Check cluster capacity
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
ALLOC_MEM:.status.allocatable.memory,\
ALLOC_CPU:.status.allocatable.cpu

# Check current usage across nodes
kubectl top nodes

# Check if cluster autoscaler is configured (for node scaling)
kubectl get deployment cluster-autoscaler -n kube-system
```

### Cluster Autoscaler Integration

If your node pool supports autoscaling:

```mermaid
flowchart LR
    Traffic["Traffic Spike"] --> HPA["HPA: scale pods<br/>3 → 8"]
    HPA --> Pending["Pods: Pending<br/>(no node capacity)"]
    Pending --> CA["Cluster Autoscaler:<br/>add nodes"]
    CA --> Scheduled["Pods scheduled<br/>on new nodes"]

    style Traffic fill:#F44336,color:#fff
    style Scheduled fill:#4CAF50,color:#fff
```

---

## Monitoring Autoscaler Behavior

```bash
# HPA current state and events
kubectl describe hpa my-spring-app-hpa -n my-namespace

# Watch HPA in real time
kubectl get hpa my-spring-app-hpa -n my-namespace -w

# HPA events (shows scaling decisions)
kubectl get events -n my-namespace --field-selector involvedObject.name=my-spring-app-hpa

# Current replica count and target metrics
kubectl get hpa -n my-namespace -o wide
```

---

## Next Steps

- [Monitoring & Metrics](monitoring-metrics.md) — Setting up the observability that informs autoscaling
- [Capacity Planning](capacity-planning.md) — Load testing to determine correct scaling thresholds
- [Kubernetes Resources](kubernetes-resources.md) — Resource configuration details
