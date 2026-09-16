# Day 08 - Multi-Container Pods and Init Containers

## What I Did Today

Today I experimented with running multiple containers inside a single pod, specifically focusing on init containers and how they gate the startup of the main container. I also ran into a few real errors along the way that made the behavior click.

---

## Init Containers

An init container is a special container that runs before the main container starts. Kubernetes runs all init containers to completion, in order, before it allows the main container to start. If any init container fails or keeps running, the main container never starts.

This is useful for:

- Waiting for a dependency (a service, a database) to be ready before the app starts
- Running setup scripts, seeding config, or pre-fetching data
- Any one-time work that must complete before the main process launches

---

## The `command` and `args` Fields

When defining a container in a pod spec, you can control what it runs using two fields:

- `command`: overrides the container image's default entrypoint
- `args`: the arguments passed to that command

```yaml
command: ["sh", "-c"]
args:
  - |
    until nslookup some-service; do
      echo "Waiting..."
      sleep 2
    done
```

Splitting it this way keeps the shell invocation (`sh -c`) in `command` and the actual script logic in `args`, which is cleaner when the script is more than one line.

---

## Experiment 1: One Init Container Waiting for a Service

### The Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: main-init
  name: main-init
spec:
  containers:
  - image: busybox:1.28
    name: main-container
    env:
    - name: CONTAINER_TYPE
      value: "Main"
    command: ["sh", "-c", "echo 'The app is running .....' && sleep 3600"]
  initContainers:
  - image: busybox:1.28
    name: init-container
    command: ["sh", "-c"]
    args:
    - |
      until nslookup main-service.default.svc.cluster.local; do
        echo "Waiting for main-service to be available..."
        sleep 2
      done
      echo "main-service is available"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

### What Happened

After applying this, the pod stayed stuck at `Init:0/1`:

```
NAME        READY   STATUS     RESTARTS   AGE
main-init   0/1     Init:0/1   0          25s
```

`Init:0/1` means zero out of one init containers have completed. The init container was running its `nslookup` loop, but `main-service` did not exist yet so every lookup failed.

### Debugging with `describe` and `logs`

To understand what was going on inside:

```bash
kubectl describe pod main-init
```

This showed the init container in `Running` state while the main container was in `Waiting` with reason `PodInitializing`. The events section confirmed the init container had started and was running.

```bash
# This fails because the main container has not started yet
kubectl logs main-init

# Specify the init container directly
kubectl logs main-init -c init-container
```

The init container logs showed the nslookup loop repeating every 2 seconds, confirming it was blocked on the missing service.

### Unblocking the Init Container

I created a deployment and exposed it as a service named exactly `main-service`:

```bash
kubectl create deployment main-init-deploy --image=nginx:latest --replicas=1
kubectl expose deployment main-init-deploy --port=80 --target-port=80 --name=main-service
```

The moment the service was created, the init container resolved the DNS lookup, printed `main-service is available`, exited cleanly, and the main container started immediately:

```
NAME        READY   STATUS    RESTARTS   AGE
main-init   1/1     Running   0          7m38s
```

This confirmed the core behavior: the main container is completely blocked until every init container exits with success.

---

## Experiment 2: Two Init Containers Running in Sequence

### The Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: main-init
  name: main-init
spec:
  containers:
  - image: busybox:1.28
    name: main-container
    env:
    - name: CONTAINER_TYPE
      value: "Main"
    command: ["sh", "-c", "echo 'The app is running .....' && sleep 3600"]
  initContainers:
  - image: busybox:1.28
    name: init-container
    command: ["sh", "-c"]
    args:
    - |
      until nslookup main-service.default.svc.cluster.local; do
        echo "Waiting for main-service to be available..."
        sleep 2
      done
      echo "main-service is available"
  - image: busybox:1.28
    name: init-container-2
    command: ["sh", "-c"]
    args:
    - |
      until nslookup db-service.default.svc.cluster.local; do
        echo "Waiting for db-service to be available..."
        sleep 2
      done
      echo "db-service is available"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

### Pod Immutability Gotcha

Before getting to experiment with two init containers, I hit this error when trying to update the running pod:

```
The Pod "main-init" is invalid: spec.initContainers: Forbidden: pod updates may not add or remove containers
```

You cannot add or remove containers from a running pod. This is the same immutability constraint from Day 03 but applied to init containers. The fix is to delete the pod and recreate it with the updated manifest.

### What Happened

With two init containers and `main-service` already running, the pod status showed:

```
NAME        READY   STATUS     RESTARTS   AGE
main-init   0/1     Init:1/2   0          3s
```

`Init:1/2` means the first init container completed and the second is now running. Init containers run strictly in order, one at a time. The second one was waiting for `db-service`, which did not exist yet.

After exposing the second service:

```bash
kubectl expose deployment main-init-deploy --port=80 --target-port=80 --name=db-service
```

The second init container resolved, and the pod moved to `1/1 Running`.

---

## Key Takeaways

- Init containers run sequentially and must all exit successfully before the main container starts.
- The pod status `Init:X/Y` tells you how many init containers have completed out of the total.
- Use `kubectl logs <pod-name> -c <container-name>` to read logs from a specific container inside a pod.
- Use `kubectl describe pod <pod-name>` to see container states and events when something is stuck.
- You cannot add or remove containers on a live pod. Delete and recreate it.

---
