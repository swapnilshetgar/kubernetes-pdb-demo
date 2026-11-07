# Kubernetes Version Upgrade with a PodDisruptionBudget (PDB)

## Current cluster and version

To upgrade the Kubernetes version:

```text
kops get cluster
--------------------------

reyaz.k8s.local aws     ap-south-1a,ap-south-1b
```

```sh
kops get cluster -o yaml
```

```sh
kops validate cluster
```

```text
Validating cluster reyaz.k8s.local

INSTANCE GROUPS
NAME                            ROLE            MACHINETYPE     MIN     MAX     SUBNETS
control-plane-ap-south-1a       ControlPlane    m7i-flex.large  1       1       ap-south-1a
nodes-ap-south-1a               Node            m7i-flex.large  1       1       ap-south-1a
nodes-ap-south-1b               Node            m7i-flex.large  1       1       ap-south-1b

NODE STATUS
NAME                    ROLE            READY
i-05155e27550668ac1     control-plane   True
i-05d1e26c4a7b9f75e     node            True
i-05e596f69ff9b58fa     node            True

Your cluster reyaz.k8s.local is ready
```

```sh
kubectl version
```

```text
Client Version: v1.37.2
Kustomize Version: v5.8.1
Server Version: v1.32.13
Warning: version difference between client (1.37) and server (1.32) exceeds the supported minor version skew of +/-1
```

```sh
kubectl get nodes
```

```text
NAME                  STATUS   ROLES           AGE    VERSION
i-05155e27550668ac1   Ready    control-plane   107m   v1.32.13
i-05d1e26c4a7b9f75e   Ready    node            104m   v1.32.13
i-05e596f69ff9b58fa   Ready    node            104m   v1.32.13
```

## The role of a PDB during an upgrade

The following example shows the workload distribution:

```text
Worker-1
 ├── Pod-A
 └── Pod-B

Worker-2
 └── Pod-C

minAvailable: 2
```

Now, suppose Worker-1 needs to be upgraded. Kubernetes/kOps will drain the node:

```text
Worker-1
    ↓
kubectl drain
    ↓
PDB check
    ↓
Are at least 2 Pods available?
    ↓
YES
    ↓
Pod eviction
```

The Deployment can schedule a replacement Pod on another available node.

## Demo flow

```text
Create Deployment
      ↓
Create 3 Pods
      ↓
Create PDB
      ↓
Spread Pods across nodes
      ↓
Node maintenance / Kubernetes version upgrade
      ↓
Drain node
      ↓
PDB eviction protection
      ↓
Deployment creates a replacement Pod
      ↓
Upgrade node
```

## Create a Deployment

Create the file:

```sh
vi nginx-deployment.yaml
```

Add this manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx-demo

  template:
    metadata:
      labels:
        app: nginx-demo

    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Apply it and check the Deployment:

```sh
kubectl apply -f nginx-deployment.yaml
kubectl get deployment
```

```text
NAME         READY   UP-TO-DATE   AVAILABLE
nginx-demo   3/3     3            3
```

View the Pods:

```sh
kubectl get pods -o wide
```

```text
NAME                         READY   STATUS    NODE
nginx-demo-xxxxx-aaa         1/1     Running   worker-1
nginx-demo-xxxxx-bbb         1/1     Running   worker-2
nginx-demo-xxxxx-ccc         1/1     Running   worker-2
```

A problem can occur here: all three Pods might be placed on the same worker. For the PDB demo, we want the Pods spread across both worker nodes.

## Create a PDB

Create the file:

```sh
vi nginx-pdb.yaml
```

Add this manifest:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb

spec:
  minAvailable: 2

  selector:
    matchLabels:
      app: nginx-demo
```

Apply it and inspect the PDB:

```sh
kubectl apply -f nginx-pdb.yaml
kubectl get pdb
```

```text
NAME        MIN AVAILABLE   ALLOWED DISRUPTIONS
nginx-pdb   2               1
```

View the PDB details:

```sh
kubectl describe pdb nginx-pdb
```

```text
Name:           nginx-pdb
Namespace:      default
Min available:  2
Selector:       app=nginx-demo
Status:
    Allowed disruptions:  1
    Current:              3
    Desired:              2
    Total:                3
Events:                   <none>
```

This means:

- Total Pods = 3
- PDB minimum = 2
- 3 - 2 = 1

One Pod may be evicted. Two Pods cannot be evicted simultaneously.

## Simulate node maintenance

```sh
kubectl get nodes
```

```text
NAME                  STATUS   ROLES           AGE    VERSION
i-05155e27550668ac1   Ready    control-plane   171m   v1.32.13
i-05d1e26c4a7b9f75e   Ready    node            169m   v1.32.13
i-05e596f69ff9b58fa   Ready    node            169m   v1.32.13
```

```sh
kubectl get pods -l app=nginx-demo -o wide
```

```text
NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE                  NOMINATED NODE   READINESS GATES
nginx-demo-67b869d47f-psm84   1/1     Running   0          12m   100.114.28.146   i-05e596f69ff9b58fa   <none>           <none>
nginx-demo-67b869d47f-tv447   1/1     Running   0          12m   100.111.72.143   i-05d1e26c4a7b9f75e   <none>           <none>
nginx-demo-67b869d47f-zskvz   1/1     Running   0          12m   100.114.28.145   i-05e596f69ff9b58fa   <none>           <none>
```

Node `i-05d1e26c4a7b9f75e` has one Pod.

### Cordon the node

In Kubernetes, cordon marks a node as unschedulable. After that, Kubernetes will not schedule new Pods on the node, but Pods that are already running there will continue running.

```sh
kubectl cordon i-05d1e26c4a7b9f75e
kubectl get nodes
```

```text
NAME                  STATUS                     ROLES           AGE    VERSION
i-05155e27550668ac1   Ready                      control-plane   175m   v1.32.13
i-05d1e26c4a7b9f75e   Ready,SchedulingDisabled   node            172m   v1.32.13
i-05e596f69ff9b58fa   Ready                      node            172m   v1.32.13
```

### Drain the node

```sh
kubectl drain i-05d1e26c4a7b9f75e --ignore-daemonsets
```

This command demonstrates the PDB in practice:

```text
Node
 ↓
Pod eviction request
 ↓
PDB check
 ↓
At least 2 Pods?
 ↓
YES
 ↓
Evict Pod
```

Observe the Pods live.

In one terminal:

```sh
watch kubectl get pods -o wide
```

In another terminal:

```sh
kubectl drain i-05d1e26c4a7b9f75e --ignore-daemonsets
```

### If the drain reports a local storage error

You may see an error like this:

```text
error: unable to drain node "i-05d1e26c4a7b9f75e" due to error: cannot delete Pods with local storage (use --delete-emptydir-data to override): argocd/argocd-application-controller-0, argocd/argocd-dex-server-ffdbfd496-gh8hp, argocd/argocd-repo-server-c94b4fd6f-l4wkc, continuing command...
There are pending nodes to be drained:
 i-05d1e26c4a7b9f75e
cannot delete Pods with local storage (use --delete-emptydir-data to override): argocd/argocd-application-controller-0, argocd/argocd-dex-server-ffdbfd496-gh8hp, argocd/argocd-repo-server-c94b4fd6f-l4wkc
```

This means drain is trying to evict the Pods, but one or more Pods use `emptyDir` or local ephemeral storage. To avoid data loss, Kubernetes stops the drain by default.

If you want to empty the node for maintenance and losing this local data is acceptable, run:

```sh
kubectl drain i-05d1e26c4a7b9f75e \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Example Pod output after the replacement is scheduled:

```text
NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE                  NOMINATED NODE   READINESS GATES
nginx-demo-67b869d47f-ls6d9   1/1     Running   0          13s   100.114.28.147   i-05e596f69ff9b58fa   <none>           <none>
nginx-demo-67b869d47f-psm84   1/1     Running   0          46m   100.114.28.146   i-05e596f69ff9b58fa   <none>           <none>
nginx-demo-67b869d47f-zskvz   1/1     Running   0          46m   100.114.28.145   i-05e596f69ff9b58fa   <none>           <none>
```

You can observe this flow:

```text
Old Pod
   ↓
Terminating
   ↓
Evicted
   ↓
Deployment notices a replica is missing
   ↓
New Pod created
   ↓
New Pod scheduled on another worker
   ↓
Running
```

Example:

**BEFORE**

```text
Worker-1:
nginx-demo-aaa  Running

Worker-2:
nginx-demo-bbb  Running
nginx-demo-ccc  Running
```

**Drain**

```text
Worker-1:
nginx-demo-aaa  Terminating
```

**Deployment replacement**

```text
Worker-2:
nginx-demo-bbb  Running
nginx-demo-ccc  Running
nginx-demo-ddd  ContainerCreating
```

### What did the PDB protect?

You have three Pods and a PDB with `minAvailable: 2`. Kubernetes must ensure:

- ❌ All 3 Pods are not evicted at once.
- ❌ 2 Pods are not evicted at once.
- ✅ 1 Pod can be evicted.

This helps maintain application availability.

Check the Pods:

```sh
kubectl get pods -o wide
```

At this point, Worker-1 has no application Pods.

## Perform the Kubernetes version upgrade

Check the current kOps version, cluster, node versions, and configured Kubernetes version:

```sh
kops version
kops get cluster
kubectl get nodes -o wide
kops get cluster -o yaml | grep kubernetesVersion
```

Preview the cluster upgrade:

```sh
kops upgrade cluster
```

The last command displays a preview. Do not add `--yes` yet. kOps can determine the target/recommended Kubernetes version for the upgrade.

Choose the target Kubernetes version:

```sh
kops upgrade cluster
```

This shows the available/recommended upgrade. Suppose the output shows target version `v1.33.x`.

Preview the upgrade command:

```sh
kops upgrade cluster
```

To select a specific version:

```sh
kops upgrade cluster \
  --kubernetes-version=<TARGET_VERSION>
```

## After maintenance

When maintenance is complete, make the node schedulable again:

```sh
kubectl uncordon i-05d1e26c4a7b9f75e
kubectl get nodes
```

