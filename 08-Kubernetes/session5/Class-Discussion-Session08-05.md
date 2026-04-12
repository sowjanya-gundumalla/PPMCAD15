# Class Discussion - Session 8-05
# Request Flows, Service Discovery & Deployment Patterns

---

## VM-Based Deployment vs Kubernetes

### VM-Based Deployment

Consider a classic 2-VM setup:

```
VM 1: Frontend (Angular + Nginx)
VM 2: Backend (Java)
```

**Request flow when an end user opens the app in their browser:**

```
Browser
  → DNS (Route53) resolves domain to Frontend VM's public IP
  → Frontend VM (Nginx serves the Angular app)
  → Angular app calls Backend VM directly via its IP address
  → Backend VM
```

The frontend knows the backend's IP address - it's hardcoded or configured at deploy time. If the backend VM changes, you update the config manually.

---

### Container-Based Deployment in Kubernetes

In Kubernetes, you no longer deal with VM IPs directly. The routing stack looks like this:

**Option A - via LoadBalancer + Ingress (recommended for production):**
```
Browser
  → DNS (Route53) → ALB (AWS Load Balancer)
  → ALB routes to Frontend Pod via target group
  → Frontend Pod calls Backend Pod via Service name (DNS)
  → Backend Pod
```

> Note: You do **not** configure Pod IPs directly as ALB target groups in the standard setup - Pod IPs are ephemeral and change every restart. Instead, the ALB targets **node IPs + NodePort**, or directly **Pod IPs** when using `target-type: ip` (which is what AWS LBC with Ingress does under the hood).

**Option B - via NodePort:**
```
Browser
  → DNS (Route53) → Worker Node IP : NodePort
  → kube-proxy forwards to the correct Pod
  → Pod
```

---

## Service Discovery in Kubernetes

Kubernetes provides **native service discovery** - microservices find each other by name, not IP address.

### Real-world example: E-commerce app

Suppose you deploy an e-commerce application to Kubernetes with these microservices:

```
checkout    order    payments    ui    api
```

Each runs in its own pod(s), with a Service in front of it.

The `checkout` microservice has business logic for checkout, and exposes a REST API. When it needs to call the `order` service internally, it doesn't use an IP - it uses the service name:

```
http://order
```

Kubernetes DNS (CoreDNS) resolves `order` to the ClusterIP of the `order` Service automatically.

Externally, the checkout flow might be reached at:
```
abc.com/checkout/  →  Ingress (ALB)  →  checkout-service  →  checkout pod
                                              ↓ internal call
                                         http://order  →  order pod
```

---

## DNS & FQDN in Kubernetes

Every Service in Kubernetes gets a DNS entry automatically via CoreDNS.

| Format | Example |
|---|---|
| Short name (same namespace) | `order` |
| With namespace | `order.default` |
| Full FQDN | `order.default.svc.cluster.local` |

```
FQDN format:  <service-name>.<namespace-name>.svc.cluster.local
```

From within the same namespace, short names work. Across namespaces, you need the full FQDN.

---

## Designing a 5-Microservice Deployment

**Requirement:**
- All 5 microservices talk to each other internally
- Only 1 needs to be exposed externally for end users

**How to deploy this in Kubernetes:**

```
                        [ Internet ]
                             |
                          Ingress
                       (1 ALB, 1 DNS)
                             |
                      ┌──────────────┐
                      │   ui-service │  ← only this is exposed externally
                      └──────┬───────┘
                             │ internal calls via Service names
              ┌──────────────┼──────────────┐
              ↓              ↓              ↓
         api-service   checkout-service   order-service
              ↓
        payments-service
```

| Service | Type | Accessible from |
|---|---|---|
| `ui-service` | Exposed via Ingress | Internet (end users) |
| `api-service` | ClusterIP | Inside cluster only |
| `checkout-service` | ClusterIP | Inside cluster only |
| `order-service` | ClusterIP | Inside cluster only |
| `payments-service` | ClusterIP | Inside cluster only |

All internal communication uses ClusterIP Services and Kubernetes DNS - no IPs, no manual config.

---

## NodePort - How It Actually Works

When you use `type: NodePort`, Kubernetes opens the same port on **every worker node** in the cluster.

**Example - 2-node cluster:**

```
Worker Node 1:  23.34.65.34
Worker Node 2:  23.34.65.35

NodePort assigned: 30080
```

Both nodes listen on that port:

```
http://23.34.65.34:30080   →  kube-proxy  →  Pod (could be on either node)
http://23.34.65.35:30080   →  kube-proxy  →  Pod (could be on either node)
```

Either node IP works - kube-proxy handles forwarding to the correct pod regardless of which node receives the request.

**In practice** you'd put a DNS record pointing to one of the node IPs, or sit a classic load balancer (L4) in front of both node IPs on port 30080 - that's essentially what `type: LoadBalancer` automates for you.