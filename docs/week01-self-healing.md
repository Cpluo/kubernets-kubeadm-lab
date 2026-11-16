# Deployment Self-Healing Lab

## 1. Objective

Verify that Kubernetes automatically creates a replacement Pod when a Pod managed by a Deployment is deleted.

## 2. Environment

- Kubernetes: v1.37.2
- Cluster: 1 control plane and 2 worker nodes
- Container runtime: containerd 2.2.1
- CNI: Calico
- Application: Nginx
- Desired replicas: 2

## 3. Initial State

The `nginx-demo` Deployment had two available replicas.

```bash
kubectl get deployment nginx-demo
kubectl get pods -o wide
```

Initial Pods:

| Pod | Status | Pod IP | Node |
|---|---|---|---|
| nginx-demo-74b4649967-6hsrx | Running | 10.244.80.194 | w2 |
| nginx-demo-74b4649967-q2rb9 | Running | 10.244.190.66 | w1 |

## 4. Test Procedure

Store the name of one Nginx Pod in a shell variable:

```bash
POD_NAME=$(kubectl get pods \
  -l app=nginx-demo \
  -o jsonpath='{.items[0].metadata.name}')
```

Confirm the selected Pod:

```bash
echo "$POD_NAME"
```

Delete the selected Pod:

```bash
kubectl delete pod "$POD_NAME"
```

Watch the replacement process:

```bash
kubectl get pods -o wide -w
```

## 5. Result

The original Pod was deleted:

```text
nginx-demo-74b4649967-6hsrx
Pod IP: 10.244.80.194
Node: w2
```

Kubernetes automatically created a replacement Pod:

```text
nginx-demo-74b4649967-h796p
Pod IP: 10.244.80.196
Node: w2
```

The Deployment returned to the desired state:

```text
READY: 2/2
UP-TO-DATE: 2
AVAILABLE: 2
```

## 6. How Self-Healing Worked

```text
A managed Pod was deleted
↓
The ReplicaSet detected only one remaining replica
↓
The Deployment required two replicas
↓
A replacement Pod was created
↓
The scheduler assigned the Pod to a worker node
↓
The kubelet and containerd started the new container
↓
The Deployment returned to two available replicas
```

## 7. Troubleshooting Note

The first deletion attempt returned:

```text
Error from server (NotFound)
```

The cause was an extra character entered at the end of the Pod name.

Instead of manually typing a long resource name, the corrected command used the shell variable:

```bash
kubectl delete pod "$POD_NAME"
```

Useful alternatives include copying the exact resource name or using shell tab completion.

## 8. What I Learned

- A Deployment maintains a declared desired state.
- A ReplicaSet maintains the required number of Pods.
- Deleting a Pod does not delete its Deployment.
- A replacement Pod usually receives a new name and Pod IP.
- `kubectl get pods -o wide -w` is useful for observing recovery.
- Shell variables reduce mistakes when working with long resource names.
