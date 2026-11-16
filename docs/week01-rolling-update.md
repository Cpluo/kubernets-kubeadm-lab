# Deployment Rolling Update and Rollback Lab

## 1. Objective

Verify that a Kubernetes Deployment can gradually replace application Pods with a new container image and roll back to a previous working version.

## 2. Environment

- Kubernetes: v1.37.2
- Cluster: 1 control plane and 2 worker nodes
- Container runtime: containerd 2.2.1
- CNI: Calico
- Deployment: `nginx-demo`
- Replicas: 2

## 3. Initial State

The Deployment initially used:

```text
nginx:alpine
```

The initial state was recorded as Revision 1:

```bash
kubectl annotate deployment nginx-demo \
  kubernetes.io/change-cause="Initial deployment using nginx:alpine" \
  --overwrite
```

## 4. Rolling Update

The container image was updated:

```bash
kubectl set image deployment/nginx-demo \
  nginx=nginx:1.27-alpine
```

The change cause was recorded:

```bash
kubectl annotate deployment nginx-demo \
  kubernetes.io/change-cause="Update nginx image to 1.27-alpine" \
  --overwrite
```

The rollout was verified:

```bash
kubectl rollout status deployment/nginx-demo
```

Result:

```text
deployment "nginx-demo" successfully rolled out
```

Both new Pods used `nginx:1.27-alpine` and reached the `Running` state:

| Node | Image | Status |
|---|---|---|
| w1 | nginx:1.27-alpine | Running |
| w2 | nginx:1.27-alpine | Running |

## 5. Revision History

The rollout history showed:

```text
REVISION  CHANGE-CAUSE
1         Initial deployment using nginx:alpine
2         Update nginx image to 1.27-alpine
```

The history was checked with:

```bash
kubectl rollout history deployment/nginx-demo
```

## 6. Rollback

The Deployment was rolled back:

```bash
kubectl rollout undo deployment/nginx-demo
```

The rollback status was verified:

```bash
kubectl rollout status deployment/nginx-demo
```

Result:

```text
deployment "nginx-demo" successfully rolled out
```

Both replacement Pods returned to the original image:

| Node | Image | Status |
|---|---|---|
| w1 | nginx:alpine | Running |
| w2 | nginx:alpine | Running |

## 7. Revision After Rollback

After rollback, the history showed:

```text
REVISION  CHANGE-CAUSE
2         Update nginx image to 1.27-alpine
3         Initial deployment using nginx:alpine
```

Rollback reapplied the old Pod template as a new current revision. It did not simply move the active revision number back to Revision 1.

## 8. How Rolling Update Worked

```text
The Deployment image was changed
↓
A new ReplicaSet was created
↓
New Pods using nginx:1.27-alpine were started
↓
Old Pods were gradually terminated
↓
The Deployment completed without removing all replicas at once
```

## 9. How Rollback Worked

```text
The previous revision was selected
↓
The old Pod template was applied again
↓
Replacement Pods using nginx:alpine were created
↓
Pods using nginx:1.27-alpine were terminated
↓
The previous application version became the current state
```

## 10. Important Commands

```bash
kubectl set image deployment/nginx-demo nginx=nginx:1.27-alpine
kubectl rollout status deployment/nginx-demo
kubectl rollout history deployment/nginx-demo
kubectl rollout history deployment/nginx-demo --revision=1
kubectl rollout undo deployment/nginx-demo
```

## 11. What I Learned

- A rolling update gradually replaces old Pods with new Pods.
- A Deployment creates a new ReplicaSet when the Pod template changes.
- `kubectl rollout status` verifies whether an update completed.
- `kubectl rollout history` displays Deployment revisions.
- Rollback reapplies an older Pod template as a new revision.
- Pod names and Pod IPs can change during updates and rollbacks.
- The Deployment continues maintaining the declared replica count.
