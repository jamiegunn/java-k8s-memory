# Java + Kubernetes Operations Guide

A comprehensive, opinionated knowledge base for Java (Spring Boot) engineers deploying to Kubernetes — covering JVM tuning, container configuration, observability, infrastructure, and operational process.

**19 documents** | **Mermaid diagrams throughout** | **Copy-paste ready commands and manifests**

---

## Who This Is For

- Java / Spring Boot developers deploying to Kubernetes
- Platform engineers supporting JVM workloads
- SREs responsible for Java services in production
- Anyone who has been OOMKilled and wants to understand why

---

## Quick Start

```bash
# JVM flags (set via environment variable in K8s)
JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=75.0 -XX:+UseG1GC"
```

```yaml
# Container resources (requests == limits for Guaranteed QoS)
resources:
  requests:
    memory: "8Gi"
  limits:
    memory: "8Gi"
```

```
Container Limit = Desired Heap / 0.75
```

For the full picture, start with the [Main Guide](java-memory-k8s-guide.md).

---

## Topic Index

### JVM & Java

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 1 | JVM Memory Model | [jvm-memory-model.md](docs/jvm-memory-model.md) | Heap vs non-heap, generation lifecycle, the critical formula (`heap / 0.75 = container limit`), NMT verification |
| 2 | Container-Aware JVM Config | [container-configuration.md](docs/container-configuration.md) | `-XX:MaxRAMPercentage`, GC selection (G1/ZGC/Shenandoah), Java version container support, `JAVA_TOOL_OPTIONS` |
| 3 | JVM Warmup | [jvm-warmup.md](docs/jvm-warmup.md) | JIT compilation phases, warmup scripts, AppCDS, readiness gating, HPA oscillation from cold pods |
| 4 | Spring Boot Memory Traps | [spring-boot-memory-traps.md](docs/spring-boot-memory-traps.md) | Unbounded `@Cacheable`, JPA persistence context leaks, session memory, CGLIB proxies, `@Async` thread explosion, high-cardinality metrics |

### Containers & Docker

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 5 | Dockerfile Best Practices | [dockerfile-best-practices.md](docs/dockerfile-best-practices.md) | Multi-stage builds, base image selection, layer caching, why NOT to hardcode `-Xmx`, security scanning |

### Kubernetes

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 6 | Resource Configuration | [kubernetes-resources.md](docs/kubernetes-resources.md) | Requests vs limits, QoS classes, CPU throttling, probes (startup/liveness/readiness), PDB, `kubectl` commands |
| 7 | Autoscaling | [autoscaling.md](docs/autoscaling.md) | HPA, VPA, KEDA, behavior tuning, Prometheus Adapter, scale-down cooldowns for Java |
| 8 | Graceful Shutdown | [graceful-shutdown.md](docs/graceful-shutdown.md) | `preStop` hooks, Spring `server.shutdown=graceful`, SIGTERM race condition, timing budget, rolling update coordination |

### Observability

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 9 | Monitoring & Metrics | [monitoring-metrics.md](docs/monitoring-metrics.md) | Spring Actuator setup, Prometheus alert rules, Grafana panels, PromQL queries, JFR recording, `jcmd` diagnostics |
| 10 | Troubleshooting | [troubleshooting.md](docs/troubleshooting.md) | Decision trees for OOMKill, slow pods, startup failures, GC issues, CPU throttling |

### CI/CD & Deployment

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 11 | CI/CD with GitHub Actions | [cicd-github-actions.md](docs/cicd-github-actions.md) | Full pipeline (build, test, smoke test, deploy, validate, rollback), Helm values per environment, PR resource change detection |
| 12 | Capacity Planning | [capacity-planning.md](docs/capacity-planning.md) | Right-sizing process, k6 load testing, A/B config comparison, cost analysis, node capacity verification |

### Deep Dives

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 13 | Why Not Autoscale Java on Memory | [why-not-autoscale-java-on-memory.md](docs/why-not-autoscale-java-on-memory.md) | JVM memory lifecycle, 3 HPA failure scenarios, what to scale on instead (CPU, RPS, latency, queue depth) |
| 14 | Connection Pool Scaling | [connection-pool-scaling.md](docs/connection-pool-scaling.md) | HikariCP sizing formula, PgBouncer in K8s, cascade failure from HPA x pool size, Redis/HTTP pool sizing |
| 15 | Advanced Considerations | [advanced-considerations.md](docs/advanced-considerations.md) | Rancher specifics, virtual threads (Java 21), GraalVM native image, cgroup v1 vs v2, sidecar memory, node pressure |

### Supporting Infrastructure

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 16 | Operationally Ready Valkey | [valkey-production-guide.md](docs/valkey-production-guide.md) | Helm chart production values, replication mode, ACL auth, TLS, persistence (AOF+RDB), Prometheus alerts, backup/restore, Spring Boot integration |

### Process & Runbooks

| # | Topic | Document | Key Content |
|:-:|:------|:---------|:------------|
| 17 | Incident Runbook | [incident-runbook.md](docs/incident-runbook.md) | 5 copy-paste runbooks (OOMKill, GC pauses, startup failure, memory leak, emergency increase), escalation matrix, post-incident template |
| 18 | Change Management Checklist | [change-management-checklist.md](docs/change-management-checklist.md) | 5-phase process (validate, plan, test, deploy, monitor), pre/post-deploy verification commands, rollback procedure |
| 19 | Developer Interview Questionnaire | [developer-interview-questionnaire.md](docs/developer-interview-questionnaire.md) | 8-section discovery template, red flag identification, sizing recommendation template, decision framework flowchart |

---

## Key Principles

1. **[Never hardcode `-Xmx` in the Dockerfile](docs/dockerfile-best-practices.md)** — use [`MaxRAMPercentage`](docs/container-configuration.md) and control memory at the K8s layer
2. **[Set `requests.memory == limits.memory`](docs/kubernetes-resources.md)** — gives Guaranteed QoS (last to be evicted)
3. **[Do not autoscale on memory](docs/why-not-autoscale-java-on-memory.md)** — the JVM doesn't release heap; scale on CPU or request rate
4. **[Prove the need before increasing](docs/capacity-planning.md)** — use [metrics](docs/monitoring-metrics.md) to validate heap pressure
5. **[Account for non-heap](docs/jvm-memory-model.md)** — the JVM uses 25-30% more memory than the heap alone
6. **Container limit = heap / 0.75** — the [formula](docs/jvm-memory-model.md#the-critical-formula) that prevents OOMKills

---

## By Situation

Not sure where to start? Find your scenario:

| I need to... | Start here |
|:-------------|:-----------|
| Understand why my pod is OOMKilled | [Troubleshooting](docs/troubleshooting.md) → [Incident Runbook](docs/incident-runbook.md) |
| Increase Java heap from 3GB to 6GB | [JVM Memory Model](docs/jvm-memory-model.md) → [Change Management Checklist](docs/change-management-checklist.md) |
| Set up monitoring for a Java app | [Monitoring & Metrics](docs/monitoring-metrics.md) |
| Configure autoscaling correctly | [Why Not Memory](docs/why-not-autoscale-java-on-memory.md) → [Autoscaling](docs/autoscaling.md) |
| Deploy Valkey (Redis) for caching/sessions | [Valkey Production Guide](docs/valkey-production-guide.md) |
| Interview a dev about their app's memory needs | [Developer Interview Questionnaire](docs/developer-interview-questionnaire.md) |
| Build a CI/CD pipeline with memory validation | [CI/CD with GitHub Actions](docs/cicd-github-actions.md) |
| Fix slow performance after deployment | [JVM Warmup](docs/jvm-warmup.md) → [Troubleshooting](docs/troubleshooting.md) |
| Right-size a new Java service for K8s | [JVM Memory Model](docs/jvm-memory-model.md) → [Container Config](docs/container-configuration.md) → [K8s Resources](docs/kubernetes-resources.md) |
| Investigate a suspected memory leak | [Spring Boot Memory Traps](docs/spring-boot-memory-traps.md) → [Incident Runbook](docs/incident-runbook.md) |
| Scale without breaking database connections | [Connection Pool Scaling](docs/connection-pool-scaling.md) |
| Prepare for a production deployment | [Change Management Checklist](docs/change-management-checklist.md) |

---

## Prerequisites

- Java 17+ (ideally 21 LTS)
- Spring Boot 3.x
- Kubernetes 1.25+
- Helm or Kustomize for manifest management
- Prometheus + Grafana for observability
- GitHub Actions for CI/CD (adaptable to other platforms)

## Diagrams

All documents use [Mermaid](https://mermaid.js.org/) for diagrams. These render natively on GitHub and in most modern Markdown viewers.

## Contributing

- Test YAML manifests against a real cluster before committing
- Validate Mermaid diagrams render correctly on GitHub
- Keep `kubectl` commands up to date with current API versions
- When adding new documents, update both [java-memory-k8s-guide.md](java-memory-k8s-guide.md) TOC and this README

---

## References

### Official Documentation

- [Oracle JVM Tuning Guide (Java 21)](https://docs.oracle.com/en/java/javase/21/gctuning/)
- [Kubernetes — Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes — Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Spring Boot Actuator Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Micrometer Prometheus Registry](https://micrometer.io/docs/registry/prometheus)

### Helm Charts & Infrastructure

- [Valkey Helm Chart (valkey-io/valkey-helm)](https://github.com/valkey-io/valkey-helm)
- [Valkey Documentation](https://valkey.io/docs/)

### Tools

- [Eclipse Temurin JDK (Adoptium)](https://adoptium.net/)
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)
- [HikariCP](https://github.com/brettwooldridge/HikariCP)
- [k6 Load Testing](https://k6.io/)
- [Eclipse MAT (Memory Analyzer)](https://eclipse.dev/mat/)
- [Redis Exporter for Prometheus](https://github.com/oliver006/redis_exporter)

### Related Reading

- [HikariCP — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Google SRE — Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Kubernetes — Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)

---

## License

Internal reference documentation. Adapt freely for your organization.
