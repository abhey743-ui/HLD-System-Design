# Setting Up Load Balancers — Manual vs Cloud-Managed

> Part 3 of 5. Builds on `01-load-balancers-fundamentals.md`.

Once you've decided you need a load balancer, you have two broad paths: **run the software yourself** (manual/self-managed) or **rent it as a service** (cloud-managed). Most real companies actually use *both*, at different layers of their stack — this file shows exactly how each works and how they're configured.

---

## 1. Manual / Self-Hosted Load Balancers

You install and run load balancing **software** yourself, on your own VM/container, and you're responsible for its uptime, scaling, patching, and redundancy.

### 1.1 Common tools

| Tool | Layer | Notes |
|---|---|---|
| **Nginx** | L7 (also does L4 `stream` mode) | Extremely common, doubles as a web server/reverse proxy |
| **HAProxy** | L4 + L7 | Purpose-built for load balancing, very high performance, battle-tested |
| **Envoy** | L4 + L7 | Modern, built for microservices/service mesh, rich observability, used inside Istio |
| **Traefik** | L7 | Cloud-native, auto-discovers backends (great with Docker/Kubernetes) |
| **IPVS (Linux kernel)** | L4 | Extremely fast, kernel-level; this is what Kubernetes uses internally (see file 4) |
| **Keepalived** | — | Not a LB itself — provides **VRRP-based failover** so you can run 2 LBs in active/passive mode without a single point of failure |

### 1.2 Example: a minimal HAProxy config (L7, round robin + health checks)

```
frontend http_front
    bind *:80
    default_backend app_servers

backend app_servers
    balance roundrobin
    option httpchk GET /health
    server app1 10.0.0.11:8080 check
    server app2 10.0.0.12:8080 check
    server app3 10.0.0.13:8080 check
```

- `balance roundrobin` → the algorithm.
- `option httpchk GET /health` → HAProxy polls `/health` on each server; if it fails, that server is pulled from rotation automatically.
- Swapping `balance roundrobin` for `balance leastconn` switches the algorithm to least-connections — this one line is the entire "algorithm decision" in practice.

### 1.3 Example: a minimal Nginx config (L7, weighted round robin)

```
upstream app_servers {
    server 10.0.0.11:8080 weight=3;
    server 10.0.0.12:8080 weight=1;
    server 10.0.0.13:8080 weight=1;
}

server {
    listen 80;
    location / {
        proxy_pass http://app_servers;
    }
}
```

Here `app1` gets 3x the traffic of the other two — this is **weighted round robin** in action, useful when one server has more CPU/RAM than the others, or during a **canary rollout** where you slowly increase the weight of the new version.

### 1.4 Pros and Cons of Manual Setup

**Pros:**
- Full control over every setting, algorithm, timeout, and header rule.
- No per-request cloud billing — just the cost of the VM/container running it.
- Portable — same config works on any cloud or on-prem.

**Cons:**
- **You own the redundancy problem** — the LB itself becomes a single point of failure unless you run at least 2 (often via Keepalived + a floating/virtual IP, or DNS with health checks).
- **You own scaling it** — if the LB itself gets overwhelmed, you have to scale *it* too (harder than scaling stateless app servers).
- **You own patching, monitoring, upgrades, SSL cert renewal**, unless automated separately.

---

## 2. Cloud-Managed Load Balancers

The cloud provider runs the actual load balancing infrastructure (often on their own massively redundant, globally distributed hardware/software) — you just describe the desired configuration via API/console/IaC, and they handle uptime, scaling, and redundancy of the LB itself.

### 2.1 AWS

| Service | Layer | Use case |
|---|---|---|
| **ALB** (Application Load Balancer) | L7 | HTTP/HTTPS, path/host-based routing, WebSockets, ideal for microservices & Kubernetes Ingress |
| **NLB** (Network Load Balancer) | L4 | Extreme throughput, static IPs, TCP/UDP, ideal for gaming/streaming/low-latency needs |
| **Gateway Load Balancer** | L3/L4 | Routes traffic through third-party virtual appliances (firewalls, intrusion detection) |
| **Global Accelerator** | Network/Anycast | Gives you 2 static anycast IPs that route globally to the nearest healthy AWS region |

Configuration model: you create **Target Groups** (the pool of backend instances/IPs/Lambda functions/containers), attach **health checks** to the target group, then create **Listeners** (e.g., port 443 → forward to Target Group X, or path `/api/*` → Target Group Y).

### 2.2 GCP (Google Cloud)

| Service | Layer | Scope |
|---|---|---|
| **Global external HTTP(S) Load Balancer** | L7 | Anycast IP, single global entry point, routes to the closest healthy region automatically |
| **Regional external HTTP(S) LB** | L7 | Single region |
| **Internal HTTP(S) LB / Internal TCP/UDP LB** | L4/L7 | Service-to-service traffic inside a VPC, doesn't touch the public internet |
| **Network Load Balancer (TCP/UDP)** | L4 | Regional, pass-through |

GCP's flagship feature here is that the **Global external HTTP(S) LB is a single anycast IP by default** — this is the Anycast concept from file 2 baked directly into the product.

### 2.3 Azure

| Service | Layer | Use case |
|---|---|---|
| **Azure Load Balancer** | L4 | Regional, TCP/UDP |
| **Application Gateway** | L7 | HTTP(S), path-based routing, WAF built-in |
| **Front Door** | L7, Global/Edge | Anycast-like global entry point + CDN + WAF |
| **Traffic Manager** | DNS-level (see file 2) | GeoDNS/priority/performance routing across regions |

### 2.4 Configuring a cloud LB — conceptually the same everywhere

Regardless of provider, you're always configuring the same four things:

1. **Frontend/Listener** — the public IP/port/protocol clients connect to.
2. **Backend pool / Target group** — the list of servers, instances, containers, or IPs that can serve the request.
3. **Routing rules** — for L7: path-based, host-based, header-based rules mapping to different backend pools.
4. **Health checks** — endpoint + interval + failure threshold that decides who's "in rotation."

### 2.5 Pros and Cons of Cloud-Managed

**Pros:**
- The LB itself is inherently highly available — no "who load-balances the load balancer" problem.
- Scales automatically to huge traffic spikes without you provisioning anything.
- Deep integration with the rest of the cloud (auto-registers new EC2 instances / Kubernetes pods, integrates with WAF, DDoS protection, certificate managers).
- Pay-as-you-go, no hardware/patching to manage.

**Cons:**
- **Cost** — bills per hour + per GB processed, which adds up at very high scale.
- **Less low-level control** — you can't tweak every internal algorithm parameter the way you can with raw HAProxy.
- **Vendor lock-in** — config/APIs are provider-specific (though Kubernetes abstracts a lot of this away — see file 4).

---

## 3. Which one do real companies actually use?

Almost always **a mix**, layered:

- **Global entry point:** Cloud-managed (Route53/Cloud DNS + Global Accelerator/Global LB) — because building your own global anycast network is basically impossible for a normal company.
- **Regional/cluster entry point:** Often cloud-managed L7/L4 LB (ALB/NLB, GCP LB) sitting in front of a Kubernetes cluster — because Kubernetes' cloud-controller-manager can provision these automatically (see file 4).
- **Inside the cluster / between microservices:** Usually **self-hosted software** — Nginx/Traefik Ingress Controllers, or Envoy via a service mesh like Istio — because this traffic is internal, extremely high-frequency, and benefits from being co-located and programmable.

This exact layering — global cloud LB → regional cloud LB → in-cluster software LB → service mesh sidecar LB — is precisely what the case study in `05-end-to-end-case-study.md` walks through end to end.
