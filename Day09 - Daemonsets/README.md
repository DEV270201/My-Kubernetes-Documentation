# Day 09 - DaemonSets

## What I Did Today

Today I learned about DaemonSets: what they are, how they behave differently from Deployments and ReplicaSets, and an interesting observation about how taints on the control plane affect scheduling. I also ran the same DaemonSet across two different cluster configurations to see the difference firsthand.

---

## What is a DaemonSet

A DaemonSet ensures that one instance of a pod runs on every node in the cluster. When a new node is added to the cluster, the DaemonSet automatically schedules a pod on it. When a node is removed, the pod is cleaned up.

Unlike a Deployment where you specify a replica count, with a DaemonSet you do not control how many pods run. The cluster decides that based on the number of eligible nodes.

Common real-world use cases:

- Log collectors that need to run on every node (Fluentd, Filebeat)
- Node monitoring agents (Prometheus node-exporter, Datadog agent)
- Networking plugins that need a presence on every node

---

## The Manifest

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    app: nginx-app-ds
spec:
  selector:
    matchLabels:
      app: nginx-app-ds
  template:
    metadata:
      labels:
        app: nginx-app-ds
    spec:
      containers:
      - name: nginx-ds-container
        image: nginx:latest
        ports:
          - containerPort: 80
```

Notice there is no `replicas` field. That is intentional and by design.

---

## Experiment 1: DaemonSet on a 3-Node Cluster

My cluster had three nodes: one control plane and two workers.

```
NAME                               ROLES           
second-kub-cluster-control-plane   control-plane   
second-kub-cluster-worker          <none>          
second-kub-cluster-worker2         <none>          
```

After applying the DaemonSet:

```
NAME       DESIRED   CURRENT   READY   
nginx-ds   2         2         2       
```

The DESIRED count was 2, not 3. The DaemonSet skipped the control plane node entirely.

### Why the Control Plane Was Skipped

The control plane node has a taint applied to it by default:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

Taints tell the scheduler to avoid placing pods on a node unless the pod explicitly tolerates that taint. Since the DaemonSet manifest has no toleration for this taint, the scheduler skipped the control plane and only scheduled on the two worker nodes.

This is intentional. The control plane runs critical cluster components (API server, scheduler, etcd) and is generally kept free of application workloads.

### Self-Healing Behavior

I deleted one of the DaemonSet pods to see what would happen:

```bash
kubectl delete pod nginx-ds-2tpsw
```

Immediately after:

```
NAME       DESIRED   CURRENT   READY   
nginx-ds   2         2         1       
```

The DaemonSet detected the pod was gone and spun up a replacement on the same node within seconds, restoring `READY` back to 2.

---

## Experiment 2: DaemonSet on a Single-Node Cluster

I created a fresh single-node cluster to compare behavior:

```bash
kind create cluster --name my-single-node-cluster
kubectl config use-context kind-my-single-node-cluster
```

This cluster had only one node, the control plane:

```
NAME                                   ROLES           
my-single-node-cluster-control-plane   control-plane   
```

After applying the same DaemonSet:

```
NAME       DESIRED   CURRENT   READY   
nginx-ds   1         1         1       
```

This time the DaemonSet scheduled on the control plane node. In a single-node KinD cluster, the control plane node also acts as a worker node, so KinD removes or relaxes the default `NoSchedule` taint. The DaemonSet pod was scheduled on it because there was no other node to run workloads.

---

## Useful Commands

```bash
# Short form for daemonsets
kubectl get ds

# Explain the DaemonSet resource schema
kubectl explain daemonsets

# Check which clusters exist
kubectl config get-clusters
```

---

## Key Takeaways

- A DaemonSet runs exactly one pod per eligible node. You do not set a replica count.
- By default, the control plane node is skipped due to its `NoSchedule` taint. You can override this by adding a toleration to the DaemonSet spec.
- You can restrict which nodes a DaemonSet targets using node selectors or node affinity.
- DaemonSets are self-healing just like ReplicaSets. Delete a pod and it gets replaced on the same node immediately.
- On a single-node KinD cluster, the control plane acts as a worker node, so the DaemonSet schedules there since it is the only available node.

---
