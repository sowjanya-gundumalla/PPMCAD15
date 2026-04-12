# Class Discussion - Session 8-06
# Ingress & Why It Exists

---

## The Problem with LoadBalancer Service Type at Scale

When you expose services externally using `type: LoadBalancer`, each service provisions its own cloud load balancer. This works fine for 1 service - but imagine you have **20–30 microservices** in your cluster and need to expose **5 of them** externally.

You now have 5 separate load balancers. Here's why that's a problem:

**Cost**
Each service creates a separate cloud load balancer. 5 external services = 5 separate LBs, each billed independently. At scale, this adds up fast.

**Operational overhead**
Instead of managing 1 load balancer with 1 TLS certificate, you're managing 5 - each with its own cert lifecycle, security group rules, health check config, and DNS entry.

**IP exhaustion**
Each load balancer consumes at least one public IP. In large clusters, this eats into your VPC's available IP space.

**Layer 4 only - no smart routing**
By default, the `LoadBalancer` service type provisions a Layer 4 (TCP/UDP) load balancer. It forwards traffic based on IP and port - that's it. There is no concept of hostnames or URL paths at this layer.

---

## Layer 4 vs Layer 7

Layer 7 is the **Application Layer** of the OSI model. A Layer 7 load balancer (like an ALB) can read and act on actual HTTP request data - things a Layer 4 LB is completely blind to:

| What L7 can see | Example |
|---|---|
| Hostname | `ak-solution.com` vs `api.ak-solution.com` |
| URL path | `/api`, `/stream`, `/foodorder` |
| Client IP | `203.0.113.42` |
| Client geography | Country, region |
| Cookies & headers | Session tokens, `Content-Type`, etc. |

This is exactly what enables the two routing strategies that Ingress is built on.

---

## The Rule

> **Whenever you need to expose more than 1 microservice externally - use Ingress, not LoadBalancer.**

A single Ingress creates one ALB. All routing decisions happen inside that one load balancer. 5 microservices to expose = 1 ALB, 1 TLS cert, 1 DNS entry.

---

## Path-Based Routing

The ALB inspects the **URL path** of incoming requests and routes to the matching service.

```
ak-solution.com/api         → ALB reads path → api-service        → api pods
ak-solution.com/stream      → ALB reads path → streaming-service   → streaming pods
ak-solution.com/foodorder   → ALB reads path → food-service        → food pods
```

All three routes are handled by a **single ALB** with a single DNS name.

---

## Host-Based Routing

The ALB inspects the **hostname** (the `Host` header in the HTTP request) and routes accordingly.

```
api.ak-solution.com     → ALB reads hostname → api-service         → api pods
stream.ak-solution.com  → ALB reads hostname → streaming-service   → streaming pods
food.ak-solution.com    → ALB reads hostname → food-service        → food pods
```

Same idea - one ALB, multiple services, routing by subdomain instead of path.

**Real-world example from AWS itself:**

```
https://ap-southeast-1.console.aws.amazon.com/  →  routed to Singapore
https://us-east-1.console.aws.amazon.com/       →  routed to North Virginia
```

AWS uses host-based routing on their own console - the subdomain tells the load balancer which regional backend to send you to.

---