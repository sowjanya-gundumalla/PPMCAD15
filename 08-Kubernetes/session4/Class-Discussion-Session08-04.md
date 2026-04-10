# Class Discussion - Session 8-04
## Kubernetes Foundations - Multi-Container Pods

---

## Pods & Microservices - The Right Deployment Model

### One Microservice Per Pod

Suppose you have a microservice-based application with **5 microservices**:

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│    UI    │  │   API    │  │  Login   │  │ Backend  │  │  Notif.  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
  Pod 1          Pod 2          Pod 3         Pod 4          Pod 5
```

All 5 are Node.js based apps that need to be deployed into Kubernetes.

**The rule:** No 2 microservices should be deployed within a single pod.

Each microservice gets its own pod - that's the baseline. If you need fault isolation, independent scaling, or separate resource limits, this separation is what enables all of it.

---

## Replicas - Scaling a Single Microservice

If the API pod starts getting overloaded, you scale it horizontally using **replicas**:

```
                    ┌──────────┐
                    │   API    │  Pod 1
                    └──────────┘
kubectl scale →     ┌──────────┐
replicas=3          │   API    │  Pod 2
                    └──────────┘
                    ┌──────────┐
                    │   API    │  Pod 3
                    └──────────┘
```

Each of the other 4 microservices can be independently scaled the same way.

---

## So... When Do You Need Multiple Containers in a Pod?

The question then becomes - if 1 microservice = 1 pod, **why does Kubernetes even allow multiple containers per pod?**

Answer: for **sidecar containers** - supporting containers that assist the main container.

### The Sidecar Pattern

```
┌──────────────────────────────────────────────┐
│                  API Pod                     │
│                                              │
│   ┌─────────────────┐   ┌─────────────────┐  │
│   │   Main: API     │   │  Sidecar: Log   │  │
│   │   Container     │   │  Shipper        │  │
│   │                 │   │                 │  │
│   │  writes logs    │-> │  reads logs     │  │
│   │  to shared disk │   │  ships to ELK/  │  │
│   └─────────────────┘   │  Splunk etc.    │  │
│                         └─────────────────┘  │
│              shared volume (disk)            │
└──────────────────────────────────────────────┘
```

Key properties of a sidecar:
- Runs **alongside** the main container inside the same pod
- Shares the **same disk (volume)** as the main container
- Has **no separate lifecycle** - starts and stops with the pod

---

## Real-World Sidecar Examples

### 1. Log Shipper Sidecar

The most common sidecar pattern in production.

**The problem it solves:**
Your API container writes logs to a file on disk. But containers are ephemeral - when the pod dies, the logs die with it. You need the logs shipped to an external source (Elasticsearch, Splunk, CloudWatch) in real time.

**How it works:**
- Main container (API) writes logs to `/var/log/app/`
- Log shipper sidecar (e.g., Fluent Bit, Filebeat) mounts the **same volume**
- Sidecar continuously tails the log files and ships them externally
- The API container doesn't need to know anything about the logging infrastructure

```yaml
# Conceptual structure
volumes:
  - name: log-volume        # shared between both containers

containers:
  - name: api               # main container writes here
    volumeMounts:
      - mountPath: /var/log/app

  - name: log-shipper       # sidecar reads from same path
    volumeMounts:
      - mountPath: /var/log/app
```

### 2. Health Check Sidecar

A sidecar that monitors the main container and reports its health status externally - useful when the main app doesn't expose a `/health` endpoint itself, or when you want richer health metadata than a simple HTTP probe can provide.

---