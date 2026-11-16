# Deployment Scaling Lab

## 1. Objective

Verify that a Kubernetes Deployment can scale the number of running Pods up and down while maintaining the declared desired state.

## 2. Environment

- Kubernetes: v1.37.2
- Cluster: 1 control plane and 2 worker nodes
- Container runtime: containerd 2.2.1
- CNI: Calico
- Application: Nginx
- Deployment: `nginx-demo`

## 3. Initial State

The Deployment initially maintained two Nginx Pods.

```bash
kubectl get deployment nginx-demo
kubectl get pods -o wide
```

Expected state:

```text
READY: 2/2
AVAILABLE: 2
```

## 4. Scale Up

The Deployment was scaled from two replicas to four:

```bash
kubectl scale deployment nginx-demo --replicas=4
```

The Pod creation process was observed with:

```bash
kubectl get pods -o wide -w
```

After scaling, all four Pods reached the `Running` state.

Pod distribution:

| Worker Node | Number of Pods |
|---|---:|
| w1 | 2 |
| w2 | 2 |

The scheduler distributed the Pods evenly in this experiment. However, default Kubernetes scheduling does not guarantee an equal distribution unless explicit scheduling rules are configured.

## 5. Scale Down

The Deployment was scaled back from four replicas to two:

```bash
kubectl scale deployment nginx-demo --replicas=2
```

Kubernetes terminated two unnecessary Pods and returned the Deployment to the desired state:

```text
READY: 2/2
UP-TO-DATE: 2
AVAILABLE: 2
```

Final Pod distribution:

| Worker Node | Number of Pods |
|---|---:|
| w1 | 1 |
| w2 | 1 |

## 6. How Scaling Worked

```text
The desired replica count was changed
↓
The Deployment updated the ReplicaSet
↓
The ReplicaSet compared the desired and actual Pod counts
↓
Scale up: new Pods were created
Scale down: unnecessary Pods were terminated
↓
The scheduler selected worker nodes for new Pods
↓
The kubelet and containerd started the containers
```

## 7. Important Commands

```bash
kubectl get deployment nginx-demo
kubectl get pods -o wide
kubectl scale deployment nginx-demo --replicas=4
kubectl get pods -o wide -w
kubectl scale deployment nginx-demo --replicas=2
```

## 8. What I Learned

- The `replicas` value represents the Deployment's desired Pod count.
- Scaling a Deployment does not require recreating the application.
- ReplicaSet creates or removes Pods to match the desired replica count.
- The scheduler determines where newly created Pods run.
- Default scheduling may distribute Pods across nodes, but equal distribution is not guaranteed.
- Scaling down removes unnecessary Pods while keeping the Deployment and Service available.
