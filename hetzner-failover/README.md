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

#### Example with `podSelector`

In `pod` mode your  `keepalived` pods are running on the same machine as the target.

```helm install hetzner-failover-ip --name hetzner-failover-ip \
--set floatingip1= YOUR_FLOATING_IP \
--set hetznertoken=YOUR_HETZNER_TOKEN \
--set namespace=YOURNAMESPACE \
--set target=YOUR_TARGET_SERVICE \
--set serviceType=pod
```
