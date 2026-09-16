# Day 03 - Running Pods: Imperative vs Declarative

## What I Did Today

Today was about getting hands-on with pods: how to run them, get inside them, generate manifests without writing them by hand, and safely edit live resources. I also picked up a key workflow habit that matters when working in teams: always keep your YAML files in sync with whatever you apply to the cluster.

---

## Two Approaches to Running a Pod

Kubernetes gives you two ways to create a pod, and understanding the difference is fundamental to how you work day-to-day.

### Imperative

You fire a command directly. No YAML, no file, just an instruction.

```bash
kubectl run <pod-name> --image=nginx:latest
```

Fast for quick tasks or debugging. But it leaves no record; the intent only lives inside the cluster state, not in version-controlled files. Not suitable for team workflows where repeatability matters.

### Declarative

You define the desired state in a manifest file and let Kubernetes reconcile to it.

```bash
# Create or update a resource from a manifest
kubectl apply -f <manifest-file-path>

# Or use create for first-time creation only
kubectl create -f <manifest-file-path>
```

The manifest (`nginx-pod.yaml`) I used today:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-1
  labels:
    app: nginx-app
    type: frontend-server
spec:
  containers:
    - name: nginx-pod-1
      image: nginx:latest
      ports:
        - containerPort: 80
          protocol: TCP
```

This is the production-grade approach. The file lives in source control, it is reviewable, and anyone on the team can recreate the exact same resource.

---



## Getting Into a Pod

Just like `docker exec` lets you shell into a container, `kubectl exec` does the same for pods.

```bash
kubectl exec -it <pod-name> -- sh
```

`-it` gives you an interactive terminal. `--` separates kubectl flags from the shell command being run inside the container. Useful for inspecting the running environment, checking configs, or debugging a misbehaving application.

---



## Generating a Manifest Without Writing It

Writing YAML by hand is error-prone, especially when you are still learning the schema. Kubernetes has a built-in shortcut: `--dry-run=client` combined with `-o yaml`.

```bash
kubectl run <pod-name> --image=nginx:latest --dry-run=client -o yaml > nginx-new-pod.yml
```

`--dry-run=client` means the command is evaluated locally without hitting the API server, nothing is actually created. `-o yaml` prints what the resource would look like as a YAML manifest. Redirecting to a file gives you a ready-to-use starting point.

The output looks like this (from the generated file):

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx-app
    tier: frontend
  name: nginx-pod-2
spec:
  containers:
  - image: nginx:latest
    name: nginx-pod-2
    resources: {}
    ports:
      - containerPort: 80
        protocol: TCP
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Notice how it fills in fields like `dnsPolicy` and `restartPolicy` that you would have had to look up manually. This is a practical time-saver, especially when bootstrapping new resource definitions.

---



## Editing a Live Pod with `kubectl edit`

`kubectl edit` opens the live resource definition in your default editor. Any valid changes you save are applied to the cluster immediately, no `apply` command needed.

```bash
kubectl edit <resource-type> <resource-name>
```



### The Catch: Not All Fields Are Editable

Kubernetes does not allow you to modify most pod spec fields on a running pod. The following fields can be changed on a live pod:


| Field                          | Why It Is Allowed                                                             |
| ------------------------------ | ----------------------------------------------------------------------------- |
| `metadata.labels`              | Labels are just metadata, changing them does not affect the running container |
| `spec.activeDeadlineSeconds`   | Controls how long the pod is allowed to run                                   |
| `spec.initContainers[*].image` | Image updates for init containers                                             |
| `spec.containers[*].image`     | Image updates for app containers                                              |


Try to edit anything else like `spec.containers[*].ports` or `spec.restartPolicy` and Kubernetes will reject it with an error. The changes are discarded and the pod stays as-is.

This is by design. Pods are mostly immutable once running. For changes that go beyond these fields, the correct path is to delete the pod and recreate it with the updated manifest.

---



## Two Ways to Edit a Manifest and When to Use Each



### Option 1: Edit the file, then apply

Open the YAML file in any editor, make your changes, save the file, then run:

```bash
kubectl apply -f nginx-pod.yaml
```

This is the recommended approach in team settings. The YAML file stays as the source of truth, and the change is tracked in version control.

### Option 2: `kubectl edit` live

Edit the resource directly against the cluster. Changes take effect immediately without running `apply`.

```bash
kubectl edit pod nginx-pod
```

Quick and convenient for exploration or urgent fixes. But if you do this without also updating the corresponding YAML file, your file and the cluster state are now out of sync. Anyone who later runs `kubectl apply -f` with the old file will overwrite your live change.

**The rule I am taking away from today:** `kubectl edit` is fine for local experimentation. The moment a change needs to survive, be shared, or be reviewed, it goes into the YAML file first.

---

