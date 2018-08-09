# High available IPs with Kubernetes in Hetzner Clouds.

This helm chart manages the assignment of your floating IPs if the target machine or process dies.

#### No single point of failure anymore!

When your machine or process crashes, the remaining (selected) ones, elect one to be then the new master. The new master then assigns its ip though the Hetzner API as target. It assigns the floating ip to itself (the new edge machine).

#### downtime < 15s

__Less than 15 seconds__ before your backup gets the new master.

```


                             │
                             │
        PodSelector          │            NodeSelector
          example            │              example
┌─────────────────────────┐  │    ┌─────────────────────────┐
│                         │  │    │                         │
│                         │  │    │                         │
│  Hetzners Floating IP   │  │    │  Hetzners Floating IP   │
│                         │  │    │                         │
│                         │  │    │                         │
└─────────────────────────┘  │    └─────────────────────────┘
             │               │                 │
             ▼               │                 ▼
 ┌──────────────────────┐    │     ┌──────────────────────┐  ┌──────────────────────┐
 │App Node              │    │     │Edge Node             │  │Edge Node             │
 │  ┌────────────────┐  │    │     │ ┌──────────────────┐ │  │ ┌──────────────────┐ │
 │  │     my-app     │  │    │     │ │  Nginx-Ingress   │ │  │ │  Nginx-Ingress   │ │
 │  │                │  │    │     │ │    (example)     │ │  │ │    (example)     │ │
 │  │                │  │    │     │ │ Label: role=edge │ │  │ │ Label: role=edge │ │
 │  └────────────────┘  │    │     │ └──────────────────┘ │  │ └──────────────────┘ │
 │  ┌────────────────┐  │    │     │  ┌────────────────┐  │  │  ┌────────────────┐  │
 │  │  failover-vip  │  │    │     │  │  failover-vip  │  │  │  │  failover-vip  │  │
 │  │   keepalived   │  │    │     │  │   keepalived   │  │  │  │   keepalived   │  │
 │  │                │  │    │     │  │                │  │  │  │                │  │
 │  └────────────────┘  │    │     │  └────────────────┘  │  │  └────────────────┘  │
 │                      │          │                      │  │                      │
 └──────────────────────┘          └──────────────────────┘  └──────────────────────┘
                                               │
                                               │
                                               └───────────┐
                                                           │
                                                           │
                                                           ▼
                                               ┌──────────────────────┐
                                               │App Node              │
                                               │                      │
                                               │  ┌────────────────┐  │
                                               │  │     my-app     │  │
                                               │  │                │  │
                                               │  │                │  │
                                               │  └────────────────┘  │
                                               └──────────────────────┘
```


### Easy setup

Run one of these commands and you are "failsafe":

#### Example NodeSelector

In `node` mode your `keepalived` pods are running on machines defined by a `nodeAffinity`:

```
helm install hetzner-failover-ip --name hetzner-failover-ip \
--set floatingip1= YOUR_FLOATING_IP \
--set hetznertoken=YOUR_HETZNER_TOKEN \
--set namespace=YOURNAMESPACE \
--set target=YOUR_TARGET_SERVICE \
--set serviceType=node \
--set nodeSelectorKey=role \
--set nodeSelectorValue=edge
```

#### Example NodeSelector with multiple IPs and `nginx-ingress`

Install `nginx-ingress` with 2 IPs 1.2.3.4,5.6.7.8, replace with your actual Hetzner floating IPs.`nginx-ingress` will be started on every k8s `node`.
```
helm install stable/nginx-ingress --name ingress --namespace ingress \
--set rbac.create=true \
--set controller.kind=DaemonSet \
--set controller.service.type=ClusterIP \
--set controller.service.externalIPs='{1.2.3.4,5.6.7.8}' \
--set controller.stats.enabled=true \
--set controller.metrics.enabled=true
```

Install 2 `keepalived` with these IPs(one install per IP) and different namespace and VRID, bind it to `nginx-ingress` and `nodes` with label `role=worker` only
```
helm install hetzner-failover-ip --name floating-ip0 --namespace floating-ip0 \
--set floatingip1=1.2.3.4 \
--set vrid=50 \
--set hetznertoken=API_TOKEN \
--set namespace=ingress \
--set target=ingress-nginx-ingress-controller \
--set serviceType=node \
--set nodeSelectorKey=role \
--set nodeSelectorValue=worker \
--set replicaCount=2

helm install hetzner-failover-ip --name floating-ip1 --namespace floating-ip1 \
--set floatingip1=5.6.7.8 \
--set vrid=51 \
--set hetznertoken=API_TOKEN \
--set namespace=ingress \
--set target=ingress-nginx-ingress-controller \
--set serviceType=node \
--set nodeSelectorKey=role \
--set nodeSelectorValue=worker \
--set replicaCount=2
```
Label all non-master `nodes` as workers to `keepalived`
```
kubectl get nodes -o name -l '!node-role.kubernetes.io/master,!role'|xargs -n1 -I'{}' kubectl label '{}' role=worker
```

#### Example with `podSelector`

In `pod` mode your  `keepalived` pods are running on the same machine as the target.

```helm install hetzner-failover-ip --name hetzner-failover-ip \
--set floatingip1= YOUR_FLOATING_IP \
--set hetznertoken=YOUR_HETZNER_TOKEN \
--set namespace=YOURNAMESPACE \
--set target=YOUR_TARGET_SERVICE \
--set serviceType=pod
```
