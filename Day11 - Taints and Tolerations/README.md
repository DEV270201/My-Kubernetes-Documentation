# Day 11 - Taints and Tolerations

## What I Did Today

Today I learned about taints and tolerations: how you can restrict which pods are allowed to run on which nodes, and how pods can opt in to tainted nodes by declaring a toleration. I ran a series of experiments to understand the rules around how Kubernetes evaluates taints and tolerations when scheduling pods.

---

## What Are Taints

A taint is applied to a node. It tells the scheduler: "do not place pods here unless the pod explicitly says it is okay with this taint."

The format is:

```
key=value:Effect
```

For example:

```bash
kubectl taint nodes second-kub-cluster-worker2 tier=backend:NoSchedule
```

This places a taint with key `tier`, value `backend`, and effect `NoSchedule` on the node.

### Taint Effects


| Effect             | Behavior                                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `NoSchedule`       | New pods will not be scheduled on this node unless they tolerate the taint. Existing running pods are not affected. |
| `PreferNoSchedule` | The scheduler tries to avoid placing pods on this node but will if there is no other option.                        |
| `NoExecute`        | New pods are not scheduled, and existing pods that do not tolerate the taint are evicted.                           |


---



## What Are Tolerations

A toleration is added to a pod spec. It tells the scheduler: "this pod is okay with a particular taint and can be scheduled on nodes that have it."

A toleration does not guarantee the pod will go to a tainted node. It just removes the restriction. The scheduler still picks the best fit from all eligible nodes.

---



## Experiment 1: Taint Both Workers, Pod Gets Stuck

I applied `tier=backend:NoSchedule` to both worker nodes:

```bash
kubectl taint nodes second-kub-cluster-worker2 tier=backend:NoSchedule
kubectl taint nodes second-kub-cluster-worker tier=backend:NoSchedule
```

At this point, the cluster had taints on every node that a regular pod would care about:

- `second-kub-cluster-control-plane` — `node-role.kubernetes.io/control-plane:NoSchedule` (default)
- `second-kub-cluster-worker` — `tier=backend:NoSchedule`
- `second-kub-cluster-worker2` — `tier=backend:NoSchedule`

One pod named `nginx-pod-3`, which was already running on `second-kub-cluster-worker2`, was not affected. `NoSchedule` only blocks new pods. It does not evict pods that are already running.

Then I created a plain pod with no tolerations:

```bash
kubectl run nginx-frontend --image=nginx:latest --dry-run=client -o yaml > nginx-frontend.yml
kubectl apply -f nginx-frontend.yml
```

It got stuck in `Pending`:

```
NAME             READY   STATUS    RESTARTS   AGE
nginx-frontend   0/1     Pending   0          10s
```

Describing the pod showed the reason:

```
Events:
  Warning  FailedScheduling  default-scheduler  0/3 nodes are available: 
  3 node(s) had untolerated taint(s).
```

All three nodes had taints the pod did not tolerate. No node was eligible.

---



## Experiment 2: Adding a Toleration for the Taint

I updated `nginx-frontend.yml` to add a toleration for `tier=backend:NoSchedule`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-frontend
  name: nginx-frontend
spec:
  containers:
  - image: nginx:latest
    name: nginx-frontend
    resources: {}
  tolerations:
  - key: tier
    operator: Equal
    value: backend
    effect: NoSchedule
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```



### Gotcha: `Equal` Not `Equals`

My first attempt used `operator: Equals`, which Kubernetes rejected:

```
The Pod "nginx-frontend" is invalid: spec.tolerations[0].operator: 
Unsupported value: "Equals": supported values: "Equal", "Exists"
```

The correct value is `Equal` without the trailing `s`. Easy to miss.

After fixing it, the pod was scheduled successfully:

```
NAME             READY   STATUS    NODE
nginx-frontend   1/1     Running   second-kub-cluster-worker
```

---



## Experiment 3: Multiple Taints — All Must Be Tolerated

I added a second taint to both workers:

```bash
kubectl taint nodes second-kub-cluster-worker app=nginx:NoSchedule
kubectl taint nodes second-kub-cluster-worker2 app=nginx:NoSchedule
```

Now each worker had two taints:

```
Taints:
  tier=backend:NoSchedule
  app=nginx:NoSchedule
```

I created a new pod `nginx-frontend-2` that only tolerated `tier=backend:NoSchedule` and applied it:

```bash
kubectl apply -f nginx-frontend-2.yml
```

It got stuck in `Pending` again with the same `FailedScheduling` event, even though the pod tolerated one of the two taints.

This is the key rule: **a pod must tolerate all taints on a node to be scheduled there.** Tolerating some but not all is not enough. The node filters out any pod that fails to clear even one taint.

---



## Experiment 4: Wildcard Toleration — Tolerate Everything

Instead of listing out every taint individually, you can use a wildcard toleration that matches all taints on any node, regardless of key, value, or effect:

```yaml
tolerations:
- operator: Exists
```

No `key` is specified. `operator: Exists` with no key means: tolerate any taint that exists. This is the most permissive toleration possible.

After updating `nginx-frontend-2.yml` with this and reapplying:

```bash
kubectl apply -f nginx-frontend-2.yml
```

```
NAME               READY   STATUS    NODE
nginx-frontend-2   1/1     Running   second-kub-cluster-worker
```

The pod was scheduled despite the node having two taints it had not explicitly listed, because the wildcard covered all of them.

The describe output of the pod (nginx-frontend-2) confirmed:

```
Tolerations: op=Exists
```

---



## Experiment 5: NoExecute — Taints That Evict Running Pods

Before this experiment, here is what was running and what each pod had as tolerations:


| Pod                | Node    | Tolerations                            |
| ------------------ | ------- | -------------------------------------- |
| `nginx-pod-3`      | worker2 | none (only default system tolerations) |
| `nginx-frontend`   | worker  | `tier=backend:NoSchedule`              |
| `nginx-frontend-2` | worker  | wildcard `op=Exists`                   |


I then added `app=nginx:NoExecute` to both worker nodes:

```bash
kubectl taint nodes second-kub-cluster-worker2 app=nginx:NoExecute
kubectl taint nodes second-kub-cluster-worker app=nginx:NoExecute
```

The result was immediate:

```
NAME               READY   STATUS    RESTARTS   AGE
nginx-frontend-2   1/1     Running   0          15m
```

Two pods vanished. Only `nginx-frontend-2` survived. Here is what happened to each:

- `nginx-pod-3` had no toleration for `app=nginx` at all. The `NoExecute` taint evicted it instantly.
- `nginx-frontend` had a toleration for `tier=backend:NoSchedule` but nothing for `app=nginx:NoExecute`. Partial tolerations are not enough, so it was evicted too.
- `nginx-frontend-2` had the wildcard toleration `op=Exists`, which matches every taint including `NoExecute`. It was not evicted.

This is the critical difference between `NoSchedule` and `NoExecute`:

- `NoSchedule` only blocks future scheduling. Pods already on the node keep running undisturbed.
- `NoExecute` goes further: it also evicts any pod currently running on the node that does not tolerate the taint.

The `NoExecute` effect is what makes taints truly disruptive. You saw this contrast directly: in Experiment 1, applying `NoSchedule` left `nginx-pod-3` running on the tainted node without touching it. The moment you switched to `NoExecute`, it was gone within seconds.

---



## Experiment 6: The Effect in a Toleration Must Match the Effect on the Taint

After the previous experiment, both worker nodes had only one taint remaining: `app=nginx:NoExecute`.

I created a new pod `nginx-frontend-3` with a toleration that matched the key and value but specified the wrong effect:

```yaml
tolerations:
- key: app
  operator: Equal
  value: nginx
  effect: NoSchedule
```

The node has `app=nginx:NoExecute`. The toleration says `app=nginx:NoSchedule`. The key and value match but the effect does not. Kubernetes treated this as a miss:

```
Events:
  Warning  FailedScheduling  0/3 nodes are available: 3 node(s) had untolerated taint(s).
```

The pod stayed in `Pending`. A toleration with a specific `effect` only matches taints with exactly that effect.

I deleted the pod, removed the `effect` field from the toleration entirely, and reapplied:

```yaml
tolerations:
- key: app
  operator: Equal
  value: nginx
```

No `effect` specified. This time the pod was scheduled on `second-kub-cluster-worker2` and ran successfully:

```
NAME               READY   STATUS    NODE
nginx-frontend-3   1/1     Running   second-kub-cluster-worker2
```

The describe output confirmed:

```
Tolerations: app=nginx
```

No effect listed — meaning it tolerates any effect for that key-value pair, whether `NoSchedule`, `NoExecute`, or `PreferNoSchedule`.

So there are two ways to write flexible tolerations:


| Approach                 | Syntax                   | What it matches            |
| ------------------------ | ------------------------ | -------------------------- |
| Omit `effect`            | `key: app, value: nginx` | Any effect for `app=nginx` |
| Omit `key` with `Exists` | `operator: Exists`       | Any taint on any node      |


The first is targeted (you still care about the key and value, just not the effect). The second is a full wildcard.

---



## Experiment 7: Scheduling a Pod on the Control Plane

This one was more of a "can I actually do it?" experiment. The control plane node has a default taint:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

That taint is exactly why no regular pods ever land on it. But by combining a wildcard toleration with a `nodeSelector`, you can force a pod there.

I updated `nginx-frontend-2.yml` with both:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx-frontend-2
  name: nginx-frontend-2
spec:
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
  - operator: Exists
  containers:
  - image: nginx:latest
    name: nginx-frontend-2
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Two things working together here:

- `nodeSelector` tells the scheduler to only consider nodes which satisfies pod's selectors. All other nodes are ruled out immediately.
- `operator: Exists` with no key is the wildcard toleration that clears every taint on the node, including the control plane's `NoSchedule` taint.

Without the toleration, the scheduler would reject the control plane node even with the nodeSelector. Without the nodeSelector, the wildcard toleration would just let the pod land on any node. Together they pin the pod to exactly one node and remove the only thing blocking it.

The pod came up running on the control plane:

```
NAME               READY   STATUS    NODE
nginx-frontend-2   1/1     Running   second-kub-cluster-control-plane
nginx-frontend-3   1/1     Running   second-kub-cluster-worker2
```

```bash
kubectl get pods -o wide
```

```
NAME               IP           NODE
nginx-frontend-2   10.244.0.5   second-kub-cluster-control-plane
nginx-frontend-3   10.244.2.7   second-kub-cluster-worker2
```

`10.244.0.x` is the control plane's pod CIDR, confirming it is genuinely running there. You would never do this in production, but it proves that the control plane taint is just a convention enforced through the standard taint/toleration mechanism, not a hard lock.

---



## Useful Commands

```bash
# Apply a taint to a node
kubectl taint nodes <node-name> key=value:Effect

# Remove a taint from a node (note the trailing -)
kubectl taint nodes <node-name> key=value:Effect-

# View taints on a node
kubectl describe node <node-name> | grep -A5 Taints

# View tolerations on a pod
kubectl describe pod <pod-name> | grep -A5 Tolerations
```

---



## Key Takeaways

- A taint on a node blocks pods that do not have a matching toleration.
- `NoSchedule` does not evict running pods. It only affects new scheduling decisions.
- `NoExecute` is stronger: it blocks new pods and immediately evicts running pods that do not tolerate the taint.
- A pod must tolerate every taint on a node to be scheduled there. Partial matches are not enough.
- `operator: Equal` with an `effect` specified requires all three to match: key, value, and effect.
- If you omit `effect` from a toleration, it matches any effect for that key-value pair.
- `operator: Exists` with no key is a full wildcard: it tolerates any taint on any node regardless of key, value, or effect.
- `nodeSelector` constrains which nodes the scheduler considers. Combined with a wildcard toleration, you can pin a pod to any node including the control plane.
- The control plane taint is a convention, not a hard lock. Tolerate it and the scheduler will happily place pods there.

---

