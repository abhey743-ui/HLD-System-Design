# Service Discovery in Distributed Systems

## 1. The Problem It Solves

Imagine you have a microservices architecture:

```
Order Service  →  needs to call →  Payment Service
Order Service  →  needs to call →  Inventory Service
```

In a simple world, `Order Service` would just call `http://192.168.1.15:8080` — a hardcoded IP and port for `Payment Service`. This works fine... until:

- `Payment Service` **auto-scales** and now runs on 10 instances instead of 1
- One instance **crashes** and gets replaced with a new IP
- You **deploy a new version** and instances get recreated with new IPs
- Instances run in **containers/Kubernetes pods**, which get new IPs constantly

Hardcoded IPs break immediately in this environment. You need a way for services to **find each other dynamically**, without hardcoding addresses.

That's exactly what **Service Discovery** solves.

```
Order Service ──▶ "Where is Payment Service right now?" ──▶ Service Registry
                                                                     │
                                                        returns: 10.0.0.5:8080
                                                                  10.0.0.9:8080
                                                                  10.0.0.3:8080
```

---

## 2. What is Service Discovery?

**Service Discovery** is the mechanism by which services in a distributed system **automatically detect and locate each other** — usually via a central component called a **Service Registry**.

Two core things happen:

1. **Registration** — when a service instance starts up, it registers itself (its IP, port, health status) with the registry
2. **Discovery** — when a service wants to call another service, it asks the registry "where are the healthy instances of X?" and gets back a list of addresses

---

## 3. Core Components

### 3.1 Service Registry
A **database of available service instances** and their network locations (IP + port). This is the "phonebook" of the system.

Popular implementations: **Consul, Eureka (Netflix), Zookeeper, etcd, Kubernetes' built-in DNS-based registry**

### 3.2 Service Provider (Registration)
Every service instance, on startup, **registers itself** with the registry (its address + metadata like version, health endpoint).

On shutdown (or crash), it should be **deregistered** — usually via:
- Explicit deregistration (graceful shutdown)
- **Heartbeats/TTL** — the registry expects periodic "I'm alive" pings; if none arrive within a timeout, the instance is automatically removed

### 3.3 Service Consumer (Discovery)
The calling service **queries the registry** to get a list of healthy instances of the service it wants to talk to, then picks one (often using a load balancing algorithm — round robin, least connections, etc.).

---

## 4. Two Main Patterns of Service Discovery

This is the most important interview distinction — there are **two fundamentally different approaches**.

### 4.1 Client-Side Discovery

**How it works:**
1. Service instance registers itself with the registry
2. The **client (calling service) directly queries the registry**
3. The client itself picks an instance (using its own load-balancing logic) and calls it directly

```
                   ┌───────────────┐
     1. register   │   Service      │
   ┌───────────────│   Registry     │◀──────────┐
   │                └───────┬───────┘            │
   │                        │ 2. query          register
   │                        ▼                     │
Payment Service      Order Service          Payment Service
  (instance A)       (client picks           (instance B)
                       instance directly)
                        │
                        └──────────▶ 3. direct call to chosen instance
```

**Pros:**
- No extra network hop — client calls the service directly (lower latency)
- Client has full control over load-balancing strategy

**Cons:**
- Discovery logic (registry client, load balancing) must be implemented in **every service/language** used — tight coupling between client and registry
- Harder to maintain across polyglot microservices (Java, Python, Go all need their own registry client library)

**Real-world example:** **Netflix Eureka + Ribbon** — classic client-side discovery combo used heavily before service meshes became popular.

---

### 4.2 Server-Side Discovery

**How it works:**
1. Service instance registers itself with the registry
2. The client sends the request to a **Load Balancer / Router** (not directly to the registry)
3. The **Load Balancer queries the registry** on the client's behalf, picks a healthy instance, and forwards the request

```
                   ┌───────────────┐
     1. register   │   Service      │
   ┌───────────────│   Registry     │
   │                └───────┬───────┘
   │                        │ 2. LB queries registry
   │                        ▼
Payment Service      ┌─────────────┐
  (instance A)   ◀───│Load Balancer│◀─── 3. request ─── Order Service (client)
                      └─────────────┘         (client only knows the LB address)
```

**Pros:**
- Client is **simple** — it just calls one fixed address (the load balancer); no discovery logic needed in client code
- Language-agnostic — works the same regardless of what the client service is written in
- Centralized control (routing rules, retries, circuit breaking can live in one place)

**Cons:**
- Extra network hop through the load balancer (slightly higher latency)
- The load balancer itself becomes a critical piece of infrastructure to manage and scale

**Real-world example:** **AWS ELB/ALB, Kubernetes Services (kube-proxy), NGINX** — the client just calls a Kubernetes Service name, and Kubernetes handles routing to actual pod IPs behind the scenes.

---

## 5. Client-Side vs Server-Side — Comparison Table

| Aspect | Client-Side Discovery | Server-Side Discovery |
|---|---|---|
| Who queries the registry | The client itself | A load balancer/router |
| Extra network hop | No | Yes |
| Client complexity | High (needs registry client + LB logic) | Low (just calls one address) |
| Language independence | Poor (needs a client library per language) | Good (language-agnostic) |
| Example tech | Netflix Eureka + Ribbon | Kubernetes Services, AWS ELB |
| Best for | Homogeneous tech stack, latency-critical systems | Polyglot microservices, simpler client code |

---

## 6. How Does the Registry Stay Accurate? (Health Checking)

A registry is only useful if it reflects the **current, real** state of the system. Two common strategies:

### 6.1 Self-Registration
The service instance itself is responsible for registering and sending periodic heartbeats to the registry.
> *Simple, but couples the service to the registry's API.*

### 6.2 Third-Party Registration (Sidecar Pattern)
A separate process (like a **sidecar** in Kubernetes, or a **registrar** service) monitors the service and registers/deregisters it — the service itself doesn't know about the registry at all.
> *Decouples business logic from infrastructure concerns — very common in service meshes like Istio.*

### Health Check Mechanisms
- **Heartbeat/TTL** — instance pings registry every N seconds; if missed, it's marked unhealthy/removed
- **Active health checks** — registry (or LB) periodically calls a `/health` endpoint on each instance

---

## 7. DNS-Based Service Discovery

A simpler, widely-used approach: use **DNS** as the registry.

- Each service gets a DNS name (e.g., `payment-service.internal`)
- DNS resolves to one or more IPs of healthy instances
- Kubernetes uses this internally — every Service gets a DNS entry, and `kube-dns`/`CoreDNS` resolves it to the right pod IPs

**Pros:** Simple, built into existing infrastructure, no extra client library needed
**Cons:** DNS caching (TTL) can cause **stale results** — a dead instance might still be returned briefly until the DNS cache expires

---

## 8. Service Mesh (Modern Evolution)

In modern cloud-native systems, service discovery is often handled by a **Service Mesh** (e.g., **Istio, Linkerd**), where a **sidecar proxy** (like Envoy) runs alongside every service instance and transparently handles:

- Service discovery
- Load balancing
- Retries & circuit breaking
- Encryption (mTLS) between services
- Observability (metrics, tracing)

This removes discovery logic from application code entirely — it becomes purely an infrastructure concern.

```
Order Service ──▶ Envoy Sidecar ──▶ Envoy Sidecar ──▶ Payment Service
                  (discovers, load
                   balances, retries)
```

---

## 9. Popular Tools at a Glance

| Tool | Type | Notes |
|---|---|---|
| **Consul** (HashiCorp) | Service Registry + Health Checks | Supports multi-datacenter, key-value store too |
| **Eureka** (Netflix) | Client-side registry | Popular in Spring Cloud ecosystem |
| **Zookeeper** | Coordination service (used for discovery too) | Strong consistency, used by Kafka, Hadoop |
| **etcd** | Distributed key-value store | Used internally by Kubernetes itself |
| **Kubernetes DNS/Services** | Server-side, DNS-based | Built-in, no extra setup needed |
| **Istio/Linkerd** | Service Mesh | Sidecar-based, handles discovery + much more |

---

## 10. When to Mention This in an HLD Interview

Bring up Service Discovery when:
- You're designing a system with **multiple microservices** that call each other
- The system needs to **auto-scale** (new instances appearing/disappearing)
- You're running on **Kubernetes/containers** (mention it's handled natively via K8s Services + DNS)
- You want to show depth beyond just "services talk to each other" — mention **how** they find each other and **handle failures gracefully** (health checks, deregistration)

---

## 11. One-Line Summary

**Service Discovery** = the phonebook system that lets services find each other's current network location automatically, instead of relying on hardcoded IPs that break the moment something scales, restarts, or crashes.

- **Client-side discovery** → client asks the registry directly, then calls the instance
- **Server-side discovery** → client calls a load balancer, which asks the registry and forwards the request
- **Service Mesh** → discovery + load balancing + security all handled transparently by sidecar proxies
