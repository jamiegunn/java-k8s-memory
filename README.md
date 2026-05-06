# Java + Kubernetes Memory Guide

A comprehensive, opinionated guide for Java (Spring Boot) engineers who need to right-size, deploy, monitor, and scale JVM applications in Kubernetes — particularly in Rancher-managed clusters.

## Who This Is For

- Java / Spring Boot developers deploying to Kubernetes
- Platform engineers supporting JVM workloads
- SREs responsible for Java services in production
- Anyone who has been OOMKilled and wants to understand why

## The Problem This Solves

Increasing Java heap from 3GB to 6GB sounds simple — just change `-Xmx`, right? In reality, it touches the JVM memory model, container configuration, Kubernetes scheduling, autoscaling behavior, CI/CD pipelines, monitoring thresholds, and incident response. This guide covers the full lifecycle.

## Quick Start

If you just need the answer:

```bash
# JVM flags (set via environment variable in K8s)
JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=75.0 -XX:+UseG1GC"

# Container resources (requests == limits for Guaranteed QoS)
resources:
  requests:
    memory: "8Gi"
  limits:
    memory: "8Gi"

# The formula
Container Limit = Desired Heap / 0.75
```

For the full picture, start with the [main guide](java-memory-k8s-guide.md).

## Guide Structure

### Core Concepts

| Document | Description |
|:---------|:------------|
| [Main Guide](java-memory-k8s-guide.md) | Overview, navigation, and key principles |
| [JVM Memory Model](docs/jvm-memory-model.md) | Heap, non-heap, and why the JVM uses more memory than `-Xmx` |
| [Container-Aware JVM Config](docs/container-configuration.md) | Flags, GC selection, percentage-based sizing |
| [Dockerfile Best Practices](docs/dockerfile-best-practices.md) | Multi-stage builds, base images, security |

### Kubernetes & Deployment

| Document | Description |
|:---------|:------------|
| [Kubernetes Resources](docs/kubernetes-resources.md) | Requests, limits, QoS classes, probes |
| [Autoscaling](docs/autoscaling.md) | HPA, VPA, KEDA — and why NOT to scale Java on memory |
| [CI/CD with GitHub Actions](docs/cicd-github-actions.md) | Build pipeline, memory smoke tests, rollback |
| [Capacity Planning](docs/capacity-planning.md) | Load testing, right-sizing, cost analysis |

### Production Operations

| Document | Description |
|:---------|:------------|
| [Monitoring & Metrics](docs/monitoring-metrics.md) | Actuator, Prometheus, Grafana, alert rules |
| [Graceful Shutdown](docs/graceful-shutdown.md) | Connection draining, `preStop` hooks, SIGTERM handling |
| [JVM Warmup](docs/jvm-warmup.md) | JIT compilation, cold-start latency, warmup strategies |
| [Connection Pool Scaling](docs/connection-pool-scaling.md) | HikariCP sizing, PgBouncer, DB as a scaling ceiling |
| [Spring Boot Memory Traps](docs/spring-boot-memory-traps.md) | Caches, JPA contexts, thread pools, metaspace bloat |

### Troubleshooting & Process

| Document | Description |
|:---------|:------------|
| [Troubleshooting](docs/troubleshooting.md) | Decision trees for OOMKill, slow pods, startup failures |
| [Incident Runbook](docs/incident-runbook.md) | Copy-paste triage commands for 2am pages |
| [Advanced Considerations](docs/advanced-considerations.md) | Rancher specifics, virtual threads, GraalVM, cgroups |
| [Change Management Checklist](docs/change-management-checklist.md) | Step-by-step process with verification commands |

## Key Principles

1. **Never hardcode `-Xmx` in the Dockerfile** — use `MaxRAMPercentage` and control memory at the K8s layer
2. **Set `requests.memory == limits.memory`** — gives Guaranteed QoS (last to be evicted)
3. **Do not autoscale on memory** — the JVM doesn't release heap; scale on CPU or request rate
4. **Prove the need before increasing** — use metrics to validate heap pressure
5. **Account for non-heap** — the JVM uses 25-30% more memory than the heap alone
6. **Container limit = heap / 0.75** — the formula that prevents OOMKills

## Prerequisites

- Java 17+ (ideally 21 LTS)
- Spring Boot 3.x
- Kubernetes 1.25+
- Helm or Kustomize for manifest management
- Prometheus + Grafana for observability
- GitHub Actions for CI/CD (adaptable to other platforms)

## Diagrams

All documents use [Mermaid](https://mermaid.js.org/) for diagrams. These render natively on GitHub and in most modern Markdown viewers. If they appear as code blocks, use a Mermaid-compatible viewer or browser extension.

## Contributing

- Test YAML manifests against a real cluster before committing
- Validate Mermaid diagrams render correctly on GitHub
- Keep `kubectl` commands up to date with current API versions
- When adding new documents, update both [java-memory-k8s-guide.md](java-memory-k8s-guide.md) TOC and this README

## License

Internal reference documentation. Adapt freely for your organization.
