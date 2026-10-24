# Day 12 - Horizontal Pod Autoscaling

## What I Did Today

Today I learned about resource requests and limits for containers, how the Metrics Server feeds data into Kubernetes, and how the HorizontalPodAutoscaler uses that data to automatically scale a Deployment up and down. I ran a live experiment where I created real CPU load and watched the HPA scale from 2 pods to 3 pods, then back down to 2 after the load was gone.

---

## Resource Requests and Limits

Every container can declare two resource properties: `requests` and `limits`.

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

**Requests** is the minimum the container needs to run. The scheduler uses this when deciding which node to place the pod on. A node must have at least this much free capacity.

**Limits** is the ceiling. The container is not allowed to exceed this.

### What happens when a container exceeds its limits

| Resource | Behavior when limit is exceeded |
| -------- | ------------------------------- |
| CPU | Container is **throttled** by the kernel. It gets less frequent CPU time slices, making the application slow and laggy. It keeps running. |
| Memory | Container is **killed** by the OOM Killer. The kernel terminates it to protect the node. The pod restarts based on its restart policy. |

Setting requests and limits is considered a best practice in Kubernetes. Without requests, the scheduler has no idea how much capacity a pod needs and can overcommit a node. Without limits, a misbehaving container can starve everything else on the node.

Requests and limits can be specified at the pod level or the container level. The recommendation is at the **container level** since it gives finer-grained control when a pod has multiple containers.

---

## Scaling Options

### Horizontal Scaling (more/fewer pods)

| Method | How |
| ------ | --- |
| Manual | Edit the manifest and update `replicas`, or run `kubectl scale deploy <name> --replicas=<n>` |
| Automatic | Use HPA (HorizontalPodAutoscaler) |

### Vertical Scaling (bigger/smaller containers)

| Method | How |
| ------ | --- |
| Manual | Edit `requests` and `limits` in the manifest |
| Automatic | Use VPA (VerticalPodAutoscaler) — requires a pod restart to take effect |

---

## Metrics Server

Every node runs a **kubelet**, and kubelet includes an embedded component called **cAdvisor**. cAdvisor collects CPU and memory metrics from every running container on that node and exposes them via an API.

The **Metrics Server** is a cluster-level component that scrapes cAdvisor from all nodes, aggregates the data, and serves it through the Kubernetes API. The HPA controller relies on the Metrics Server to get current resource utilization.

I deployed the Metrics Server using the official manifest:

```bash
kubectl apply -f components.yaml
```

Without the Metrics Server running, the HPA cannot get any metrics and the `TARGETS` column shows `<unknown>`:

```
NAME           REFERENCE                 TARGETS              MINPODS   MAXPODS   REPLICAS
nginx-deploy   Deployment/nginx-deploy   cpu: <unknown>/50m   2         5         1
```

Once the Metrics Server is running, it populates to real values:

```
NAME           REFERENCE                 TARGETS        MINPODS   MAXPODS   REPLICAS
nginx-deploy   Deployment/nginx-deploy   cpu: 0%/50%    2         5         2
```

---

## HorizontalPodAutoscaler

The HPA is a Kubernetes object with its own controller. The controller runs a **control loop every 15 seconds**, querying the Metrics Server for current utilization. Based on the result it calculates how many replicas should be running, then updates the Deployment.

HPA works with Deployments (preferred) and StatefulSets.

### Creating an HPA

You can create one imperatively:

```bash
kubectl autoscale deployment nginx-deploy --min=2 --max=5 --cpu=50%
```

Or declaratively via a manifest. The `--cpu` flag accepts either a percentage (`50%`) or a raw quantity (`500m`). These are different:

- `--cpu=50%` → target average CPU **utilization** as a percentage of the container's request
- `--cpu=500m` → target average CPU **value** in millicores (absolute, not relative to requests)

I first ran `--cpu=50` (no `%`), which Kubernetes interpreted as `50m` (50 millicores, absolute). That is why the target showed as `50m` instead of `50%`. Deleting and recreating with `--cpu=50%` fixed it.



### Scale-down stabilization

The HPA has a built-in cooldown for scale-down (default: 5 minutes). After load drops, it does not immediately remove pods. It waits to see if the utilization stays low before scaling back in. This prevents flapping when load is bursty.

You can see this in the `describe` output:

```
Conditions:
  AbleToScale  True  ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
```

This is why the pod count stayed at 3 for several minutes even after CPU dropped to 0%.

---

## Experiment: Live Load Test

### Setup

I deployed an nginx Deployment with resource requests defined and a ClusterIP Service in front of it using `nginx-hpa.yml`:

```yaml
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "200m"
```

Then created the HPA:

```bash
kubectl autoscale deployment nginx-deploy --min=2 --max=5 --cpu=50%
```

### Observing the HPA enforce the minimum

Initially the Deployment had 1 replica. The HPA immediately scaled it up to 2 to satisfy `--min=2`:

```
Events:
  Normal  SuccessfulRescale  New size: 2; reason: Current number of replicas below Spec.MinReplicas
```

### Generating load

I used a busybox pod running a wget loop hitting the nginx service continuously:

```bash
kubectl run -it --rm load-generator-1 --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://nginx-svc; done;"
```

### Scale-out

As load built up, CPU crossed the 50% threshold:

```
nginx-deploy   Deployment/nginx-deploy   cpu: 56%/50%   2 (min)         5 (max)        2 (desired)
```

The HPA triggered a scale-out after a few polling cycles:

```
Events:
  Normal  SuccessfulRescale  New size: 3; reason: cpu resource utilization (percentage of request) above target
```

Pods went from 2 to 3. With load spread across 3 pods, CPU dropped back below target:

```
nginx-deploy   Deployment/nginx-deploy   cpu: 36%/50%   2         5         3
```

### Scale-in after removing load

Once the load-generator pod was removed, CPU dropped to 0%:

```
nginx-deploy   Deployment/nginx-deploy   cpu: 0%/50%   2         5         3
```

The HPA held at 3 replicas for several minutes due to scale-down stabilization. Eventually it scaled back to 2:

```
Events:
  Normal  SuccessfulRescale  New size: 2; reason: All metrics below target
```

---

## Useful Commands

```bash
# Deploy Metrics Server
kubectl apply -f components.yaml

# Create HPA imperatively (percentage-based)
kubectl autoscale deployment <name> --min=<n> --max=<n> --cpu=<n>%

# View all HPAs
kubectl get hpa

# Inspect HPA details, conditions, and events
kubectl describe hpa <name>

# Generate load against a ClusterIP service from inside the cluster
kubectl run -it --rm load-generator-1 --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://<service-name>; done;"

# View cluster events
kubectl events
```

---

## Key Takeaways

- Container `requests` tell the scheduler how much capacity a pod needs. Always set them or the scheduler is flying blind.
- CPU limit breaches throttle the container. Memory limit breaches kill it (OOM Killer).
- The Metrics Server is a prerequisite for HPA. Without it, HPA cannot retrieve utilization data and the targets show `<unknown>`.
- cAdvisor inside kubelet is what actually collects per-container metrics. The Metrics Server aggregates them cluster-wide.
- HPA controller runs every 15 seconds. It queries the Metrics Server and recalculates the desired replica count using current vs. target utilization.
- `--cpu=50%` is relative to the container's CPU request. `--cpu=500m` is an absolute millicore value. They behave differently.
- HPA has a scale-down stabilization window (default 5 minutes) to prevent thrashing when load fluctuates. Scale-out has no such delay.
- The HPA always enforces `minReplicas`. Even with 0% CPU usage it will not drop below the minimum.
- A pod must have CPU `requests` defined for CPU-based HPA to work correctly. The percentage target is relative to that request.

---
