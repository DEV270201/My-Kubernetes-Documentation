# Day 10 - Static Pods and Manual Scheduling

## What I Did Today

Today I went deeper into how Kubernetes actually boots itself up. The key question was: if the scheduler is what assigns pods to nodes, and the scheduler itself runs as a pod, then who scheduled the scheduler? That led me to static pods. I also ran an experiment that showed exactly what happens when the scheduler is gone, and how you can bypass it entirely by doing manual scheduling.

---

## The Chicken-and-Egg Problem

When Kubernetes starts, none of the control plane components are running yet. There is no API server to talk to, no scheduler to assign pods, and no controller manager managing anything. So how do they all come up?

The answer is that these components do not go through the normal scheduling pipeline at all. They are **static pods**.

---

## What Are Static Pods

Static pods are pods that kubelet manages directly by watching a specific directory on the host filesystem. On the control plane node, that directory is:

```
/etc/kubernetes/manifests/
```

kubelet watches this directory continuously. When it finds a YAML file there, it creates and runs that pod on its own, without talking to the API server or the scheduler. If the pod crashes, kubelet restarts it. If the YAML file is removed, kubelet stops the pod.

```
# ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

Each of these files is a pod spec. The scheduler, API server, etcd, and controller manager all start this way.

kubelet itself is not a pod. It runs as a systemd service on the control plane node:

```bash
systemctl list-units | grep kubelet
# kubelet.service    loaded active running  kubelet: The Kubernetes Node Agent
```

That is why it can run without needing to be scheduled by anything. kubelet is the foundation everything else sits on.

---

## Inspecting the Scheduler Static Pod

Running `kubectl describe` on the scheduler pod reveals the tell:

```bash
kubectl describe pod kube-scheduler-second-kub-cluster-control-plane -n kube-system
```

Key fields:

```
Annotations:  kubernetes.io/config.source: file
Controlled By: Node/second-kub-cluster-control-plane
```

`config.source: file` means kubelet picked it up from the manifests directory. `Controlled By: Node/...` means the node (kubelet) is responsible for it, not a Deployment or ReplicaSet.

---

## Experiment: What Happens When the Scheduler Is Gone

I removed the scheduler manifest by moving it out of the watched directory:

```bash
docker exec -it second-kub-cluster-control-plane /bin/sh
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp
```

After that, the scheduler pod disappeared from `kubectl get pods -n kube-system`.

Then I created a pod without specifying a node:

```bash
kubectl run nginx-pod-nonodename --image=nginx:latest
```

The pod was created but stayed stuck in `Pending`:

```
NAME                   READY   STATUS    RESTARTS   AGE
nginx-pod-nonodename   0/1     Pending   0          12s
```

Describing the pod confirmed why:

```
Node:    <none>
Events:  <none>
```

No events, no node assigned. Without the scheduler, nobody is looking at unassigned pods and deciding where they should run. The pod just sits there indefinitely.

---

## Manual Scheduling: Bypassing the Scheduler with nodeName

Normal pods have no `nodeName` in their spec. That is the scheduler's job: to pick a node and write the `nodeName` field. But you can set it yourself, which bypasses the scheduler entirely.

I generated a base manifest using `--dry-run`:

```bash
kubectl run nginx-pod-withnodename --image=nginx:latest --dry-run=client -o yaml > nginx-pod-withnodename.yml
```

Then I added `nodeName` to the spec manually:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-pod-withnodename
  name: nginx-pod-withnodename
spec:
  nodeName: second-kub-cluster-worker2
  containers:
  - image: nginx:latest
    name: nginx-pod-withnodename
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### Gotcha: nodeName Goes in spec, Not in containers

My first attempt put `nodeName` inside `spec.containers[0]`, which is wrong. Kubernetes rejected it:

```
Error from server (BadRequest): strict decoding error: unknown field "spec.containers[0].nodeName"
```

`nodeName` belongs at the `spec` level, not inside the container definition. Once I moved it to the right place, the apply worked.

```bash
kubectl apply -f nginx-pod-withnodename.yml
# pod/nginx-pod-withnodename created
```

Checking the pods:

```
NAME                     READY   STATUS    NODE
nginx-pod-nonodename     0/1     Pending   <none>
nginx-pod-withnodename   1/1     Running   second-kub-cluster-worker2
```

The manually scheduled pod ran immediately. The scheduler was still missing and the pending pod was still stuck, but that did not affect this one at all. kubelet on `second-kub-cluster-worker2` received the pod spec with a node assignment already set, so it just ran it.

---

## Restoring the Scheduler

After the experiment I moved the scheduler manifest back:

```bash
docker exec -it second-kub-cluster-control-plane /bin/sh
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
```

kubelet detected the file, started the scheduler pod again, and within seconds the pending pod got assigned a node and started running:

```
NAME                     READY   STATUS    NODE
nginx-pod-nonodename     1/1     Running   second-kub-cluster-worker2
nginx-pod-withnodename   1/1     Running   second-kub-cluster-worker2
```

---

## Useful Commands

```bash
# Inspect the manifests directory inside the control plane node
docker exec -it <control-plane-node-name> ls /etc/kubernetes/manifests/ (applicable if using KinD)

# Check that kubelet is running as a systemd service
docker exec -it <control-plane-node-name> systemctl list-units | grep kubelet (applicable if using KinD)

# Describe a static pod (note the annotations and Controlled By field)
kubectl describe pod <static-pod-name> -n kube-system

# Generate a base manifest for manual scheduling
kubectl run <pod-name> --image=<image> --dry-run=client -o yaml > pod.yml

# See which node a pod is running on
kubectl get pods -o wide
```

---

## Key Takeaways

- Static pods are managed directly by kubelet, not the scheduler. Their specs live in `/etc/kubernetes/manifests/` on the node.
- kubelet itself is a systemd service, not a pod. It is the lowest-level piece that everything else depends on.
- Control plane components (API server, scheduler, etcd, controller manager) all start as static pods. This solves the bootstrap problem.
- Pods without a `nodeName` depend on the scheduler to be assigned a node. If the scheduler is gone, they stay `Pending` indefinitely.
- Pods with `nodeName` set bypass the scheduler. kubelet on the target node picks them up and runs them directly.
- `nodeName` belongs in `spec`, not inside `spec.containers`. Putting it inside containers causes a strict decoding error.

---
