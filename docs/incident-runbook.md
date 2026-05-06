# Incident Runbook: Java Memory Issues

[< Back to Main Guide](../java-memory-k8s-guide.md)

---

## How to Use This Runbook

This is a **copy-paste-ready** triage guide for when you're paged at 2am. Follow the steps in order. Each section has exact commands — replace `<pod>`, `<namespace>`, and `<app>` with your values.

```
NAMESPACE=my-namespace
APP=my-spring-app
```

---

## Runbook 1: Pod OOMKilled

**Alert:** `ContainerOOMRisk` or `PodRestarting` fired.

### Step 1: Confirm the OOMKill (30 seconds)

```bash
# Check pod status
kubectl get pods -l app=$APP -n $NAMESPACE

# Look for OOMKilled in termination reason
kubectl get pods -l app=$APP -n $NAMESPACE -o jsonpath='{range .items[*]}{.metadata.name} restarts={.status.containerStatuses[0].restartCount} reason={.status.containerStatuses[0].lastState.terminated.reason}{"\n"}{end}'

# Check events
kubectl get events -n $NAMESPACE --field-selector reason=OOMKilling --sort-by='.lastTimestamp' | tail -5
```

**If OOMKilled = Yes → Continue. If No → Jump to Runbook 2.**

### Step 2: Get Previous Logs (1 minute)

```bash
# Get logs from the crashed container
POD=$(kubectl get pods -l app=$APP -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n $NAMESPACE --previous | tail -100

# Look for:
# - "java.lang.OutOfMemoryError: Java heap space"     → Heap exhaustion
# - "java.lang.OutOfMemoryError: Metaspace"           → Metaspace exhaustion
# - "java.lang.OutOfMemoryError: Direct buffer"       → NIO buffer exhaustion
# - No Java error but "Killed"                        → Container RSS exceeded limit
```

### Step 3: Immediate Mitigation

**Option A: If single pod and service is down:**
```bash
# Restart the deployment (pods will come back with same config)
kubectl rollout restart deployment/$APP -n $NAMESPACE
kubectl rollout status deployment/$APP -n $NAMESPACE --timeout=120s
```

**Option B: If repeated OOMKills (crash loop):**
```bash
# Temporarily increase memory limit (emergency — revert after investigation)
kubectl patch deployment $APP -n $NAMESPACE --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "12Gi"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "12Gi"}
]'

# Watch for successful restart
kubectl rollout status deployment/$APP -n $NAMESPACE --timeout=180s
```

**Option C: If bad deployment caused it:**
```bash
# Rollback to previous version
kubectl rollout undo deployment/$APP -n $NAMESPACE
kubectl rollout status deployment/$APP -n $NAMESPACE --timeout=180s
```

### Step 4: Capture Diagnostics (Once Stable)

```bash
POD=$(kubectl get pods -l app=$APP -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')

# Current heap state
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.heap_info

# Native memory breakdown (if NMT enabled)
kubectl exec $POD -n $NAMESPACE -- jcmd 1 VM.native_memory summary

# Top classes in heap
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.class_histogram | head -30

# Thread count
kubectl exec $POD -n $NAMESPACE -- jcmd 1 Thread.print | grep "^\"" | wc -l

# If heap dump needed (WARNING: may fill disk, takes time):
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.heap_dump /tmp/heapdump.hprof
kubectl cp $NAMESPACE/$POD:/tmp/heapdump.hprof ./heapdump-$(date +%Y%m%d-%H%M).hprof
```

### Step 5: Determine Root Cause

| Finding | Root Cause | Next Action |
|:--------|:-----------|:------------|
| Heap at 95%+, specific class dominates histogram | Memory leak or cache issue | Heap dump analysis with Eclipse MAT |
| Heap fine (<70%), but RSS at limit | Non-heap growth (threads, native) | NMT analysis, check thread count |
| Metaspace at MaxMetaspaceSize | Classloader leak | Check for hot-reload in prod, dynamic proxy creation |
| Direct buffers at max | NIO buffer leak | Check Netty/file channel usage |
| Happened after deployment | New code introduced leak | Rollback, compare with previous version |
| Happened during traffic spike | Under-provisioned for peak | Increase memory or scale horizontally |

---

## Runbook 2: High Latency / GC Pauses

**Alert:** `JavaGCHighPauseTime` or `HighLatency` fired.

### Step 1: Confirm GC is the Cause (30 seconds)

```bash
POD=$(kubectl get pods -l app=$APP -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')

# Check GC status
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.heap_info

# Check recent GC activity
kubectl exec $POD -n $NAMESPACE -- tail -30 /tmp/gc.log 2>/dev/null

# Alternatively via actuator:
kubectl port-forward $POD 8080:8080 -n $NAMESPACE &
curl -s http://localhost:8080/actuator/metrics/jvm.gc.pause | jq
```

### Step 2: Is It GC or CPU Throttling?

```bash
# Check CPU throttling (Prometheus query or):
kubectl exec $POD -n $NAMESPACE -- cat /sys/fs/cgroup/cpu.stat 2>/dev/null || \
kubectl exec $POD -n $NAMESPACE -- cat /sys/fs/cgroup/cpu/cpu.stat 2>/dev/null
# Look for: "nr_throttled" — if high, it's CPU throttling, not GC

# Check current CPU usage
kubectl top pod $POD -n $NAMESPACE --containers
```

**If CPU throttled → increase CPU limit or remove it. Not a memory issue.**

### Step 3: Assess GC Pressure

```bash
# Via Prometheus (if available):
# rate(jvm_gc_pause_seconds_sum{pod="$POD"}[5m])     → seconds of GC per second
# jvm_gc_pause_seconds_max{pod="$POD"}               → longest recent pause

# Via actuator:
curl -s http://localhost:8080/actuator/metrics/jvm.gc.pause | jq '.measurements'
# Look at COUNT (frequency) and TOTAL_TIME (seconds spent in GC)
```

### Step 4: Immediate Mitigation

**If heap is nearly full (>90%) and GC can't reclaim:**
```bash
# This is likely a memory leak or sustained pressure
# Scale up to distribute load
kubectl scale deployment/$APP -n $NAMESPACE --replicas=5

# Or temporarily increase memory
kubectl patch deployment $APP -n $NAMESPACE --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "12Gi"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "12Gi"}
]'
```

**If heap has room but pauses are long:**
```bash
# Consider switching GC (requires restart)
# Add to JAVA_TOOL_OPTIONS: -XX:+UseZGC
kubectl set env deployment/$APP -n $NAMESPACE \
  JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=75.0 -XX:+UseZGC -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError -XX:+ExitOnOutOfMemoryError"
```

---

## Runbook 3: Pods Failing to Start

**Alert:** Pods in `CrashLoopBackOff` or `Pending` state.

### Step 1: Identify the Problem

```bash
# Check pod status
kubectl get pods -l app=$APP -n $NAMESPACE
kubectl describe pod $POD -n $NAMESPACE | tail -30

# Common states:
# Pending       → Not enough node resources to schedule
# OOMKilled     → Container exceeded limit during startup
# CrashLoopBackOff → App is crashing (check logs)
# ContainerCreating → Image pull or volume mount issue
```

### Step 2: If Pending (Resource Constraint)

```bash
# Check why it can't be scheduled
kubectl describe pod $POD -n $NAMESPACE | grep -A 10 "Events"
# Look for: "Insufficient memory" or "didn't match Pod's node affinity"

# Check node capacity
kubectl top nodes
kubectl describe nodes | grep -A 10 "Allocated resources"

# Check namespace quota
kubectl describe resourcequota -n $NAMESPACE
```

**Fix:** Either reduce pod size, add nodes, or increase namespace quota.

### Step 3: If OOMKilled During Startup

```bash
# JVM is allocating -Xms immediately and exceeding limit
kubectl logs $POD -n $NAMESPACE --previous | head -20

# Verify the math:
# -XX:MaxRAMPercentage=75.0 with limits.memory=8Gi → heap=6Gi
# JVM will try to reserve 6Gi immediately at startup
# If container limit is only 6Gi, there's no room for non-heap → instant OOMKill
```

**Fix:** Container limit must be > heap size + non-heap overhead (use heap/0.75).

### Step 4: If CrashLoopBackOff (App Error)

```bash
# Get logs
kubectl logs $POD -n $NAMESPACE --previous

# If no previous logs (crashed too fast):
kubectl describe pod $POD -n $NAMESPACE | grep -A 5 "Last State"

# Check if startupProbe is killing it:
kubectl describe pod $POD -n $NAMESPACE | grep -A 5 "startup"
# If startup takes 90s but probe times out at 60s → increase failureThreshold
```

---

## Runbook 4: Gradual Memory Growth (Suspected Leak)

**Alert:** `NonHeapMemoryGrowing` or heap trending upward over days.

### Step 1: Confirm the Trend

```bash
# Check heap over time (via Prometheus):
# increase(jvm_memory_used_bytes{area="heap", pod=~".*$APP.*"}[24h])

# Spot check current state:
POD=$(kubectl get pods -l app=$APP -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.heap_info
```

### Step 2: Baseline and Diff

```bash
# Take baseline
kubectl exec $POD -n $NAMESPACE -- jcmd 1 VM.native_memory baseline
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.class_histogram > /tmp/baseline-classes.txt

echo "Baseline taken at $(date). Wait 2-4 hours, then run diff."

# After 2-4 hours:
kubectl exec $POD -n $NAMESPACE -- jcmd 1 VM.native_memory summary.diff
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.class_histogram > /tmp/current-classes.txt
diff /tmp/baseline-classes.txt /tmp/current-classes.txt | head -40
```

### Step 3: Heap Dump for Analysis

```bash
# Force GC first (to only capture non-reclaimable objects)
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.run
sleep 5

# Take heap dump
kubectl exec $POD -n $NAMESPACE -- jcmd 1 GC.heap_dump /tmp/heapdump.hprof

# Copy locally for Eclipse MAT analysis
kubectl cp $NAMESPACE/$POD:/tmp/heapdump.hprof ./leak-investigation-$(date +%Y%m%d).hprof
echo "Analyze with: Eclipse MAT → Leak Suspects Report"
```

### Step 4: Common Leak Patterns to Look For

In Eclipse MAT:
- **Leak Suspects Report** → identifies objects retaining the most memory
- **Dominator Tree** → shows which objects "own" the most heap
- **Path to GC Roots** → shows why objects can't be collected

| Retained By | Likely Cause |
|:------------|:-------------|
| `ConcurrentHashMap` in a service bean | Unbounded cache, no eviction |
| `ArrayList` in an event listener | Events stored but never consumed |
| `SessionImpl` (Hibernate) | Long-running transaction holding entity references |
| `byte[]` in Tomcat | Response buffers not released |
| Spring ApplicationContext references | Classloader leak (DevTools, dynamic reload) |

---

## Runbook 5: Emergency Memory Increase

**Situation:** Production is down or degraded. Need more memory NOW.

### Immediate (< 5 minutes)

```bash
# 1. Increase memory limit
kubectl patch deployment $APP -n $NAMESPACE --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "12Gi"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "12Gi"}
]'

# 2. Wait for rollout
kubectl rollout status deployment/$APP -n $NAMESPACE --timeout=300s

# 3. Verify pods healthy
kubectl get pods -l app=$APP -n $NAMESPACE
kubectl top pods -l app=$APP -n $NAMESPACE --containers

# 4. Verify JVM sees new limit
POD=$(kubectl get pods -l app=$APP -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n $NAMESPACE -- jcmd 1 VM.flags | grep MaxHeapSize
# Should be 75% of 12Gi ≈ 9Gi (if using MaxRAMPercentage)
```

### Post-Incident (Same Day)

```bash
# 1. Document what happened
echo "Incident: OOMKill at $(date). Temporary increase to 12Gi."

# 2. Create a ticket/issue to:
#    - Investigate root cause
#    - Right-size permanently
#    - Revert temporary increase OR make it permanent with justification
```

---

## Escalation Matrix

| Situation | Escalate To | Why |
|:----------|:------------|:----|
| OOMKill, restart fixes it | On-call (info only) | May recur, investigate next business day |
| Repeated OOMKills, restart doesn't help | Senior on-call + team lead | Service is degraded |
| Memory increase blocked by quota | Platform/infra team | Need quota/node increase |
| Suspected memory leak in code | Application dev team | Requires code analysis |
| All pods across multiple services OOMKilled | Incident commander | Cluster-wide issue (node pressure) |
| Can't identify root cause after 30 min | Senior SRE + dev lead | Need heap dump analysis expertise |

---

## Post-Incident Template

```markdown
## Incident Report: Java Memory Issue

**Date:** YYYY-MM-DD HH:MM
**Duration:** X minutes
**Impact:** [Describe user impact — errors, latency, downtime]
**Severity:** P1/P2/P3

### Timeline
- HH:MM — Alert fired: [alert name]
- HH:MM — On-call acknowledged
- HH:MM — [First action taken]
- HH:MM — [Mitigation applied]
- HH:MM — Service recovered

### Root Cause
[What caused the memory issue — leak, under-provisioned, traffic spike, etc.]

### Mitigation
[What was done to restore service]

### Evidence
- Grafana dashboard link: [URL]
- Heap dump location: [path]
- Relevant metrics: [screenshots or PromQL queries]

### Action Items
- [ ] [Permanent fix — e.g., fix leak, resize, add monitoring]
- [ ] [Preventive — e.g., add alert, load test, update runbook]
- [ ] [Process — e.g., review change management, update documentation]
```

---

## Quick Reference Card

Print this and keep at your desk:

```
┌─────────────────────────────────────────────────────────────┐
│ JAVA MEMORY INCIDENT QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Check pod status:                                           │
│   kubectl get pods -l app=$APP -n $NS                       │
│                                                             │
│ Check OOMKill reason:                                       │
│   kubectl describe pod $POD -n $NS | grep -A5 "Last State" │
│                                                             │
│ Get crash logs:                                             │
│   kubectl logs $POD -n $NS --previous | tail -100           │
│                                                             │
│ Rollback:                                                   │
│   kubectl rollout undo deployment/$APP -n $NS               │
│                                                             │
│ Emergency memory increase:                                  │
│   kubectl patch deployment $APP -n $NS --type='json' \      │
│     -p='[{"op":"replace",                                   │
│     "path":"/spec/template/spec/containers/0/               │
│     resources/limits/memory","value":"12Gi"}]'              │
│                                                             │
│ Check heap:                                                 │
│   kubectl exec $POD -n $NS -- jcmd 1 GC.heap_info          │
│                                                             │
│ Thread dump:                                                │
│   kubectl exec $POD -n $NS -- jcmd 1 Thread.print           │
│                                                             │
│ Heap dump:                                                  │
│   kubectl exec $POD -n $NS -- jcmd 1 GC.heap_dump          │
│     /tmp/heapdump.hprof                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Next Steps

- [Troubleshooting](troubleshooting.md) — Detailed decision trees
- [Monitoring & Metrics](monitoring-metrics.md) — Setting up alerts to prevent incidents
- [Change Management Checklist](change-management-checklist.md) — Making permanent fixes safely
