# Load Balancers in HLD — Complete Guide

## 1. What is a Load Balancer?

A **Load Balancer (LB)** is a component that sits between clients and a group of backend servers. Its job is simple: **distribute incoming traffic across multiple servers** so that:

- No single server gets overloaded
- The system stays fast and responsive
- If one server dies, traffic is redirected to healthy ones (fault tolerance)
- You can scale horizontally by just adding more servers

```
                     ┌───────────┐
Client ───────────▶ │    LB      │
                     └─────┬─────┘
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         Server 1     Server 2     Server 3
```

Without a load balancer, all requests would hit one server, and that server becomes a **single point of failure** and a **bottleneck**.

---

## 2. Layer 4 vs Layer 7 Load Balancing (quick context)

Before the algorithms, know this — it comes up a lot in interviews:

| Type | Works at | Decision based on | Speed | Example |
|---|---|---|---|---|
| **L4 (Transport Layer)** | TCP/UDP | IP address + Port | Very fast | AWS NLB, LVS |
| **L7 (Application Layer)** | HTTP/HTTPS | URL, headers, cookies, content | Slower (more processing) | AWS ALB, NGINX, HAProxy |

L4 just forwards packets without reading the actual content. L7 can look inside the request (e.g., route `/api/images` to image servers, `/api/video` to video servers).

---

## 3. Types of Load Balancing Algorithms

Load balancing algorithms fall into two broad categories:

- **Static algorithms** — don't consider real-time server state (traffic split by a fixed rule)
- **Dynamic algorithms** — consider real-time server load, response time, connections, etc.

Let's go through each one.

---

### 3.1 Round Robin

**Idea:** Requests are sent to servers **one after another, in a fixed cyclic order**.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A   (cycle repeats)
Request 5 → Server B
```

**Type:** Static

**Best when:**
- All servers have roughly the **same specs/capacity**
- Requests are roughly equal in processing cost

**Weaknesses:**
- Doesn't consider that some servers might already be busy or slower
- A slow server still gets the same share of traffic as a fast one — can overload it

**Real-world use:** DNS round robin, simple NGINX setups.

---

### 3.2 Weighted Round Robin (WRR)

**Idea:** Same as round robin, but each server is given a **weight** based on its capacity (CPU, RAM, etc.). Servers with higher weight get more requests.

```
Server A → weight 5
Server B → weight 3
Server C → weight 2

Out of every 10 requests:
A gets 5, B gets 3, C gets 2
```

**Type:** Static (weights are usually set manually, not auto-adjusted)

**Best when:**
- Servers have **different capacities** (e.g., one server is a bigger EC2 instance than others)

**Weaknesses:**
- Weights are usually fixed manually — doesn't adapt to sudden real-time load spikes
- Needs manual tuning as infrastructure changes

**Real-world use:** Common in HAProxy, NGINX when server pool is heterogeneous.

---

### 3.3 Least Connections

**Idea:** Send the new request to whichever server currently has the **fewest active connections**.

```
Server A → 10 active connections
Server B → 4 active connections   ← next request goes here
Server C → 7 active connections
```

**Type:** Dynamic

**Best when:**
- Requests take **varying amounts of time** to process (e.g., some API calls are heavy, some are light)
- Long-lived connections exist (WebSockets, streaming)

**Weaknesses:**
- Doesn't know if a server is just "connected" vs actually "under heavy CPU load" — connection count isn't always equal to real load

**Real-world use:** HAProxy, NGINX (`least_conn` directive), common for stateful/long connections.

---

### 3.4 Weighted Least Connections

**Idea:** Combines **Least Connections + Weights**. Servers with higher capacity (weight) are allowed proportionally more active connections before being deprioritized.

```
effective_load = active_connections / weight
```

The server with the **lowest effective_load** gets the next request.

**Type:** Dynamic + Static hybrid

**Best when:**
- Servers have different capacities **and** requests vary in duration

**Real-world use:** HAProxy, large heterogeneous clusters.

---

### 3.5 IP Hash (Source IP Hashing)

**Idea:** Compute a hash of the client's IP address, and use that hash to consistently map the client to the **same server** every time.

```
server = hash(client_IP) % number_of_servers
```

**Type:** Static (deterministic)

**Best when:**
- You need **session persistence / sticky sessions** — e.g., a shopping cart stored in server memory, and the same user must keep hitting the same server
- No shared/centralized session store (like Redis) exists

**Weaknesses:**
- **Uneven distribution** if client IPs aren't diverse (e.g., many users behind one corporate NAT/proxy IP all hit the same server)
- If a server goes down, `% number_of_servers` changes for everyone — massive redistribution (this is the exact problem **Consistent Hashing** solves)

**Real-world use:** NGINX `ip_hash`, scenarios needing sticky sessions without external session storage.

---

### 3.6 Least Response Time

**Idea:** Send the request to the server with the **lowest response time** (and often combined with fewest active connections).

```
score = response_time × active_connections
```
Server with the lowest score wins.

**Type:** Dynamic

**Best when:**
- You want the **fastest possible experience** — commonly used for latency-sensitive services
- Server performance varies over time (not just by connection count)

**Weaknesses:**
- Needs continuous health/latency monitoring — more overhead to implement
- Response time can be noisy/fluctuate

**Real-world use:** AWS ELB (least outstanding requests + latency), CDNs.

---

### 3.7 Random

**Idea:** Just pick a server **at random** for each request.

**Type:** Static

**Best when:**
- Simple systems, doesn't need precision
- Surprisingly, with a large number of requests, random distribution tends to even out reasonably well (law of large numbers)

**Weaknesses:**
- No consideration of load at all
- Not reliable for small numbers of requests/servers

**Real-world use:** Rarely used alone; sometimes combined as "power of two random choices" (pick 2 random servers, send to the less loaded one — used internally by some modern systems like gRPC load balancing).

---

### 3.8 Consistent Hashing

**Idea:** An improvement over plain IP hashing. Servers and keys (e.g., client IDs) are placed on a **hash ring (0 to 2^32-1)**. A request is routed to the **next server clockwise** on the ring from its hashed key.

```
        Server A
       /         \
  Server D       Server B
       \         /
        Server C

Client hash lands between D and A → goes to Server A
```

**Type:** Static (deterministic), but resilient to change

**Best when:**
- You need sticky sessions/caching **and** servers frequently scale up/down (auto-scaling groups)
- Distributed caches (e.g., Memcached, DynamoDB, Cassandra ring topology)

**Why it's better than IP Hash:**
- When a server is added/removed, only a **small fraction of keys** get remapped — not the entire mapping. This avoids the "cache stampede" problem of plain IP hashing.

**Real-world use:** CDNs, distributed caches, Cassandra/DynamoDB partitioning, some API gateways.

---

### 3.9 URL Hash / Path-Based Routing (Layer 7 only)

**Idea:** Route based on the **URL path or content** of the request, not just IP.

```
/images/*  → Image servers
/video/*   → Video servers
/api/*     → API servers
```

**Type:** Static/content-aware

**Best when:**
- Microservices architecture where different services handle different endpoints
- Useful for caching — same URL always hits the same cache server

**Real-world use:** API Gateways, NGINX/HAProxy reverse proxy rules, Kubernetes Ingress.

---

## 4. Quick Comparison Table

| Algorithm | Type | Considers server load? | Sticky sessions? | Best use case |
|---|---|---|---|---|
| Round Robin | Static | ❌ | ❌ | Equal-capacity servers, simple traffic |
| Weighted Round Robin | Static | ❌ (manual weights) | ❌ | Servers with different capacities |
| Least Connections | Dynamic | ✅ | ❌ | Long-lived / variable-duration requests |
| Weighted Least Connections | Dynamic | ✅ | ❌ | Mixed capacity + variable requests |
| IP Hash | Static | ❌ | ✅ | Simple sticky sessions |
| Least Response Time | Dynamic | ✅ | ❌ | Latency-sensitive apps |
| Random | Static | ❌ | ❌ | Simple, low-stakes systems |
| Consistent Hashing | Static (resilient) | ❌ | ✅ | Caching layers, auto-scaling clusters |
| URL/Path Hash | Static | ❌ | Depends | Microservices, content-based routing |

---

## 5. How to Choose the Right Algorithm (Interview Tip)

Ask yourself:

1. **Are all servers equal in capacity?**
   → Yes: Round Robin. No: Weighted Round Robin.

2. **Do requests take wildly different amounts of time to process?**
   → Yes: Least Connections (or Weighted Least Connections).

3. **Do I need the same client to always hit the same server (session data stored locally, or caching)?**
   → Yes: IP Hash or Consistent Hashing (prefer Consistent Hashing if servers scale up/down often).

4. **Is latency the top priority?**
   → Least Response Time.

5. **Is this a microservices setup where different routes go to different services?**
   → URL/Path-based routing (L7).

In real systems (like AWS ELB/ALB, NGINX, HAProxy), **multiple strategies are often combined** — e.g., Least Connections + health checks + weighted capacity — rather than relying on one pure algorithm.

---

## 6. Bonus: Health Checks (Why LBs Need Them)

No matter which algorithm you use, a load balancer constantly runs **health checks** (e.g., ping `/health` every few seconds). If a server fails the check, it's temporarily removed from the pool — so no algorithm ever sends traffic to a dead server.

```
LB → GET /health → Server
     ← 200 OK  (healthy, keep in rotation)
     ← timeout/5xx (unhealthy, remove temporarily)
```

---

## 7. One-Line Summary of Each

- **Round Robin** — take turns, in order.
- **Weighted Round Robin** — take turns, but stronger servers get more turns.
- **Least Connections** — go to whoever's least busy right now.
- **Weighted Least Connections** — least busy, adjusted for capacity.
- **IP Hash** — same client always goes to the same server.
- **Least Response Time** — go to whoever replies fastest.
- **Random** — pick anyone, no logic.
- **Consistent Hashing** — like IP hash, but survives servers being added/removed gracefully.
- **URL/Path Hash** — route based on what the request is asking for, not who's asking.
