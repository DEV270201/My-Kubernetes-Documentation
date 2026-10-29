# Day 13 - Health Probes

## What I Did Today

Today I learned about the three types of health probes in Kubernetes — **readiness**, **liveness**, and **startup** — and built an intuition for when each one applies. I ran two hands-on experiments: one with a multi-container pod behind a Service to see how a failing readiness probe affects traffic routing, and another with a dedicated liveness probe to see how Kubernetes reacts when an app stops responding (and what happens when the probe itself is misconfigured).

---

## The Three Probe Types

| Probe | Question it answers | Action on failure |
| ----- | -------------------- | ------------------ |
| **Readiness** | "Can this container currently serve traffic?" | Pod removed from the Service's Endpoints list. Container is **not** restarted. |
| **Liveness** | "Is this container's process still functioning, or is it stuck?" | Container is **killed and restarted**. |
| **Startup** | "Has this slow-booting app finished starting up yet?" | Liveness/readiness checks are held off until this passes once. If it never passes, the container is killed and restarted. |

### Readiness checks external dependencies, liveness checks the app itself

My own framing, confirmed: readiness should check things like "is the DB connection up," "has the app finished its bootstrap sequence," i.e. **external dependencies and point-in-time state**. Liveness should check whether **the app's own process/runtime is still functioning**, independent of its dependencies.

---

## Building Intuition for Liveness Probes

The confusing part is readiness is easy to reason about ("can I serve traffic right now?"), but liveness is vague — it depends entirely on the app. The mental model that made it click:

### The "Will a Restart Fix It?" Rule

Before putting a check inside a liveness probe, ask:

> **If this check fails and Kubernetes kills + restarts the container, will the problem actually be solved?**

- **DB is down** → restarting the app does NOT fix a dead database → **not a liveness check** (this is readiness).
- **Memory leak / OOM freeze** → restarting wipes the leaked memory → **good liveness check**.
- **Deadlocked threads** → restarting kills the deadlock and spins up fresh threads → **good liveness check**.

Liveness is a **dead man's switch**: it doesn't care if the app has everything it needs to work (fuel, clear tracks); it only cares if the engine itself is still alive and responding at all.

### What to actually check, by app pattern

| App pattern | What to probe | Why it works |
| ----------- | -------------- | ------------- |
| Web/REST APIs | A trivial `/healthz/live` endpoint that just returns `200 OK` | If the runtime is hung (infinite loop, thread exhaustion), it can't even respond to a basic HTTP request — the timeout itself is the proof of failure. |
| Background workers / queue consumers | An `exec` probe checking a heartbeat file's timestamp (e.g. `touch /tmp/heartbeat` inside the main loop, probe checks it's recent) | No HTTP port to probe, so you look for proof the main loop is still iterating. |
| Legacy / unmodifiable apps | `exec` probe checking the process is alive (e.g. `ps aux | grep <process>`) | Last resort — just confirms the process hasn't vanished outright. |

**Golden rule:** keep liveness probes dumb and cheap. Complex logic inside a liveness probe can throw its own errors and kill an otherwise healthy container.

---

## initialDelaySeconds vs. startupProbe

- `initialDelaySeconds` on a liveness/readiness probe is a **fixed, blind wait**. You have to set it to the worst-case boot time, so fast boots waste time waiting and the probe can't react to an early hang until the delay passes.
- A `startupProbe` **disables liveness/readiness entirely** until it passes once, then hands off. It polls frequently (`periodSeconds` × `failureThreshold` gives it a generous total window), so a fast app becomes ready immediately and a slow app still gets the time it needs — without guessing a fixed number.
- They are not usually combined. `startupProbe` replaces the need for a large `initialDelaySeconds`; keep `initialDelaySeconds` only for small, predictable waits (2-5s) on an app already known to be started.

---

## Multi-Container Pod Behavior (confirmed via experiment)

A few things I reasoned through and then verified:

- **Container crash/OOM/liveness failure → only that container restarts.** The pod is a logical wrapper; the kubelet repairs individual containers. Other containers in the pod are untouched and keep their state.
- **`restartPolicy` is a Pod-level field**, not per-container — even though the restart action itself happens at the container level. You cannot mix `Always` for one container and `Never` for another in the same pod (the one exception: a container inside `initContainers` with `restartPolicy: Always` is treated as a native sidecar, K8s 1.29+).
- **A failing readiness probe on ANY container marks the whole Pod NotReady**, even if every other container (including the main app) is perfectly healthy. This is because a Service routes to a **Pod IP**, not to an individual container — there's no such thing as "send traffic to container A but not container B" within the same pod. Kubernetes does **not** restart the container for a readiness failure — it just pulls the pod out of the Service's Endpoints list until it passes again.

---

## Experiment 1 — Readiness Probe in a Multi-Container Pod (`health-probe.yml`)

### Setup

- A `Deployment` with 2 replicas, each pod running two containers:
  - `web-server` (nginx:alpine) — serves `index.html` from a shared `emptyDir` volume.
  - `probe-helper` (busybox) — writes `index.html`, touches a `/data/healthy` file, then sleeps. Has a **readiness probe** (`exec: cat /data/healthy`).
- A `ClusterIP` Service (`experiment-svc`) selecting `app: test-app`, load balancing across both pods.
- A `load-generator` pod running `curl` in a loop against the Service to observe live traffic routing.

### What I observed

1. Both pods started `2/2 Running` and the Service had both pod IPs in its `Endpoints` list — `curl` responses alternated between both pod hostnames.
2. I `exec`'d into one pod's `probe-helper` container and `rm /data/healthy`. Within a few failed probe cycles (`period=2s`, `failure threshold=3`), that pod flipped to `1/2 Ready`.
3. `kubectl describe svc experiment-svc` showed its `Endpoints` list **immediately dropped that pod's IP** — traffic only reached the remaining healthy pod.
4. **The unhealthy container was never restarted** (`Restart Count: 0` the whole time) — only `Ready: False`. This is the key proof that readiness failure ≠ container restart, it's purely a traffic-routing decision.
5. Repeating this on the second pod as well caused the Service's `Endpoints:` to go completely empty — zero pods receiving traffic even though both app containers were still "Running" the whole time.
6. Restoring the file (`touch /data/healthy`) brought the pod back into `Endpoints` and traffic resumed.

### Takeaway

Readiness failures are invisible at the container/restart level — you have to check `kubectl describe svc` or `kubectl get endpoints` to see the actual effect, not `kubectl get pods` restart counts.

---

## Experiment 2 — Liveness Probe (`liveness-probe.yml`)

### Setup

A single pod running the official Kubernetes test image (`registry.k8s.io/e2e-test-images/agnhost:2.48`) with the `liveness` arg. This image is built to deliberately return `200` on `/healthz` for the first ~10 seconds, then `500` afterward — purpose-built for testing liveness behavior.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```

### What I observed (correct port)

- Pod ran fine initially (`Ready: True`, `Restart Count: 0`).
- After ~10s, `/healthz` started returning `500`. Once 3 consecutive failures (`#failure=3`) accumulated, kubelet logged `Liveness probe failed: HTTP probe failed with statuscode: 500` and then `Killing... Container liveness failed liveness probe, will be restarted`.
- The container restarted, worked fine again for ~10s, then failed again — restart count climbed (1, 2, 3...). This matches the image's intended behavior and is expected.

### What I observed (misconfigured port — the real lesson)

I then changed the probe's port from `8080` to `9000` (a port the app isn't actually listening on):

- The container went straight into a restart spiral: `Liveness probe failed: Get "http://<pod-ip>:9000/healthz": dial tcp <pod-ip>:9000: connect: connection refused`.
- Because the probe could never succeed even once, Kubernetes kept killing and restarting the container on every failure cycle, and it eventually hit **`CrashLoopBackOff`** with increasing back-off delays between restart attempts.
- **Why this happened:** the app was never actually broken — the probe itself was pointed at the wrong port. From Kubernetes' perspective there's no difference between "the app is genuinely frozen" and "my probe is misconfigured and can't reach the app" — both produce the identical symptom (probe failure → restart). This is an important debugging lesson: **a CrashLoopBackOff caused by a liveness probe doesn't always mean the app is broken — check the probe's target (port/path) against what the app is actually listening on before assuming the app itself is at fault.**

---

## Useful Commands

```bash
# Watch pod readiness/restart status live
kubectl get pods -o wide

# Check what pods a Service is actually routing to right now
kubectl describe svc <service-name>
kubectl get endpoints <service-name>

# Exec into a specific container inside a multi-container pod
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Tail recent events for a pod (probe failures, kills, restarts)
kubectl describe pod <pod-name>

# Cluster-wide event stream, useful for correlating probe failures across pods
kubectl events

# Generate continuous traffic against a ClusterIP service to observe load balancing live
kubectl run load-generator --image=curlimages/curl -i --tty --rm -- /bin/sh -c "while true; do curl -s http://<service-name>; sleep 1; done"
```

---

## Key Takeaways

- Readiness = "can I serve traffic right now" (checks external dependencies/state). Liveness = "is my own process still functioning" (independent of dependencies). Use the **"will a restart fix it?"** test to decide which bucket a check belongs in.
- Keep liveness probes dumb and cheap — an HTTP `200`, a heartbeat file timestamp, or a process check. Complex logic inside a liveness probe is a self-inflicted outage risk.
- `startupProbe` replaces a guessed `initialDelaySeconds` with a dynamic "wait exactly as long as needed" boot gate, then hands control to liveness/readiness.
- A container crash/OOM/liveness failure restarts **only that container** — other containers in the pod are untouched. `restartPolicy` is a **Pod-level** field even though restarts happen per-container.
- A readiness failure on **any** container marks the **whole Pod** `NotReady` and removes it from the Service's `Endpoints` — because Services route by Pod IP, not per-container. No restart happens; it's purely a traffic decision. Verify with `kubectl describe svc` / `kubectl get endpoints`, not just `kubectl get pods`.
- A `CrashLoopBackOff` driven by a failing liveness probe doesn't always mean the app is broken — a misconfigured probe (wrong port/path) produces the exact same symptom as a genuinely frozen app. Always check the probe's target against what the app is actually listening on first.

---
