# Kubernetes and Load Balancing — Deep Dive

> Part 4 of 5. This is the core technical piece — how everything from files 1-3 actually gets implemented inside Kubernetes.

---

## 1. Quick Kubernetes Architecture Refresher

Before load balancing makes sense, you need the map:

- **Cluster** — the whole Kubernetes system: a set of machines (nodes) working together.
- **Node** — a physical or virtual machine (this is your "horizontal scaling of machines" — more nodes = more capacity).
- **Pod** — the smallest deployable unit; wraps one or more containers. Pods are **ephemeral** — they get created, destroyed, and rescheduled constantly (deploys, crashes, autoscaling, node failures).
- **Deployment** — describes how many replica Pods of your app should exist. **This is "horizontal scaling of containers"** — scaling a Deployment from 3 to 30 replicas.
- **kubelet** — the agent on every node that actually runs the containers.
- **kube-proxy** — a networking component on every node responsible for making Services reachable and load-balancing traffic to Pods (the heart of this file).
- **Control Plane** — API server, scheduler, controller-manager, etcd — the "brain" deciding what should run where.
- **CNI (Container Network Interface)** plugin — gives every Pod its own IP address and makes Pod-to-Pod networking work across nodes (e.g., Calico, Cilium, Flannel).

Because Pods are ephemeral and constantly get new IPs, **you can never point a client directly at a Pod IP**. This is exactly the "dynamic service discovery" problem from your handout — and Kubernetes' answer is the **Service** object.

---

## 2. Kubernetes Service Types (this is where load balancing configuration actually lives)

A **Service** is a stable abstraction — a fixed name + IP — that sits in front of a *changing* set of Pods, and load-balances across them.

### 2.1 ClusterIP (default)
- Gets a stable **virtual IP**, reachable only **inside the cluster**.
- Used for internal service-to-service communication (microservice A calling microservice B).
- This is the Service type used for almost everything *except* the one entry point at the edge of the cluster.

### 2.2 NodePort
- Opens a specific port (30000-32767 range) on **every node** in the cluster. Hitting `<any-node-IP>:<nodePort>` routes you (via kube-proxy) to the Service, which then load-balances to a Pod.
- Rarely used directly in production — usually just the plumbing underneath a cloud LoadBalancer Service (see next) or used for quick testing/on-prem setups.

### 2.3 LoadBalancer
- Everything NodePort does, **plus**: if you're running on a cloud provider, the **cloud-controller-manager** automatically provisions an actual **cloud load balancer** (an AWS NLB/ALB, GCP LB, Azure LB) and points it at your nodes' NodePort.
- This is the magic bridge between "cloud-managed LB" (file 3) and "Kubernetes-internal LB" (this file): you write one YAML object, and Kubernetes talks to the cloud API on your behalf to spin up a real external load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: checkout-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"   # example AWS-specific annotation
spec:
  type: LoadBalancer
  selector:
    app: checkout
  ports:
    - port: 443
      targetPort: 8443
```

- `selector: app: checkout` is the crucial line — the Service load-balances across **every Pod carrying the label `app: checkout`**, however many there currently are.

### 2.4 ExternalName
- Not really load balancing — just a DNS-level CNAME alias to an external service (e.g., pointing to a managed database endpoint outside the cluster). Mentioned for completeness.

### 2.5 Ingress (the real L7 layer inside Kubernetes)

A Service (even type LoadBalancer) is fundamentally **L4** — it doesn't understand HTTP paths or hostnames. For **L7 routing** (path-based, host-based — exactly the concepts from file 1), Kubernetes uses a separate object: **Ingress**.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
spec:
  rules:
    - host: api.shop.com
      http:
        paths:
          - path: /checkout
            pathType: Prefix
            backend:
              service:
                name: checkout-service
                port:
                  number: 80
          - path: /catalog
            pathType: Prefix
            backend:
              service:
                name: catalog-service
                port:
                  number: 80
```

An Ingress *object* is just a set of routing rules — it does nothing by itself. You need an **Ingress Controller** (actual running software) to read these rules and act on them: **Nginx Ingress Controller**, **Traefik**, **HAProxy Ingress**, or a cloud-native one (**AWS Load Balancer Controller**, **GCP Ingress-GCE**). The controller itself typically runs as Pods exposed via a Service of type LoadBalancer — so the *very first* thing that receives external traffic is actually an Nginx (or similar) Pod running *inside* your cluster, sitting behind a cloud LB.

---

## 3. kube-proxy — how Service load balancing is actually implemented

This is the piece your handout's algorithms (round robin, least connections, etc.) map onto concretely. `kube-proxy` runs on **every node** and is responsible for making a Service's virtual IP actually route to real Pod IPs. It has three historical modes:

### 3.1 userspace mode (legacy, rarely used today)
kube-proxy itself acts as an actual proxy process, round-robining connections between Pods. Slow — every packet passes through userspace. Deprecated in practice.

### 3.2 iptables mode (long-time default)
kube-proxy programs the Linux kernel's **iptables** rules so that traffic to a Service's virtual IP gets **randomly** redirected (via NAT rules with probability weighting) directly to one of the backing Pod IPs — entirely inside the kernel, no extra proxy process in the data path.
- **Algorithm:** essentially **random** selection (weighted equally by default), not true round robin — implemented as a chain of probabilistic iptables rules.
- **Downside at scale:** with thousands of Services/Pods, the iptables rule-set becomes enormous and slow to update.

### 3.3 IPVS mode (modern default for large clusters)
kube-proxy configures the kernel's **IPVS** (IP Virtual Server) subsystem — the same L4 load-balancing tech mentioned in file 1. IPVS uses efficient hash tables instead of a giant sequential rule chain, so it scales far better with huge numbers of Services.
- **Algorithms available (configurable!):** `rr` (round robin), `lc` (least connection), `dh` (destination hashing), `sh` (source hashing — session affinity), `sed` (shortest expected delay), `nq` (never queue). This is literally the load-balancing-algorithm menu from your handout, implemented at the kernel level.

### 3.4 So what actually happens on a request, mechanically?

```
Client request arrives at Node's network interface
        │
        ▼
kube-proxy's iptables/IPVS rules intercept traffic to the Service's virtual IP
        │
        ▼
One of the Service's healthy Pod IPs is chosen (per the configured algorithm)
        │
        ▼
Packet is forwarded (via the CNI network) — possibly to a Pod on a DIFFERENT node
        │
        ▼
Pod's container processes the request and responds
```

Notice: kube-proxy doesn't care which **node** a Pod lives on — Pod networking (via the CNI plugin) makes every Pod reachable from every node, so the "chosen Pod" could be local or on a completely different machine. This is what makes horizontal scaling of containers across many machines transparent.

---

## 4. Endpoints / EndpointSlices — how kube-proxy knows which Pods are healthy

A Service doesn't loadbalance to "all Pods matching this label" blindly — it loadbalances to Pods that are **Ready**, tracked via an **EndpointSlice** object that Kubernetes updates automatically:

- Every Pod has **readiness probes** (health checks — the Kubernetes equivalent of a load balancer's `/health` check from file 1 and 3).
- If a Pod fails its readiness probe, it's immediately removed from the Service's EndpointSlice — kube-proxy stops sending it traffic — **without ever removing the Pod itself** (it might still be starting up, or temporarily struggling, and can rejoin once healthy).
- If a Pod crashes entirely, the Deployment controller creates a replacement Pod (with a new IP), and it's added to the EndpointSlice once ready.

This is the direct, concrete implementation of the "health checks add/remove servers from rotation" concept from file 1.

---

## 5. Bare-metal Kubernetes — MetalLB

Cloud providers auto-provision a real LB for `type: LoadBalancer` Services. But what if you're running Kubernetes **on-premises / bare metal** (no cloud API to call)? There's no cloud-controller-manager to magically create an LB.

**MetalLB** solves this: it implements the `LoadBalancer` Service type on bare metal by either:
- **Layer 2 mode** — one node announces the Service's IP via ARP, and MetalLB handles failover if that node dies, or
- **BGP mode** — advertises the Service IP via BGP to your physical network routers (the same Anycast-style routing concept from file 2), letting real network hardware load-balance across nodes.

---

## 6. Service Mesh — even smarter L7 load balancing between microservices

Once you have **hundreds or thousands of microservices** calling each other constantly (exactly the scenario in file 5's case study), plain kube-proxy L4 load balancing starts to feel too simple. You want: retries, timeouts, circuit breakers (from your handout!), mutual TLS, fine-grained traffic splitting (canary/A-B testing), and rich observability — **per service-to-service call**, not just at the cluster edge.

This is what a **service mesh** (Istio, Linkerd, Consul Connect) provides, using the **sidecar pattern**:

- Every Pod gets a second container injected automatically: a tiny proxy (Istio uses **Envoy**).
- **All** network traffic in and out of the Pod is transparently routed through this sidecar proxy.
- The sidecar handles: load balancing (with real algorithms — round robin, least request, consistent hashing — configurable **per service**), automatic retries with backoff, **circuit breaking** (exactly the concept from your handout — Envoy has native circuit breaker support: max connections, max pending requests, outlier detection that ejects unhealthy instances), mutual TLS between services, and detailed metrics/tracing.

```
Pod A                                   Pod B
┌─────────────┐                    ┌─────────────┐
│  App        │                    │  App        │
│  Container  │                    │  Container  │
└─────┬───────┘                    └──────▲──────┘
      │ localhost                         │ localhost
      ▼                                   │
┌─────────────┐   mTLS + LB + retries  ┌──┴──────────┐
│ Envoy Sidecar├─────────────────────► │ Envoy Sidecar│
└─────────────┘                        └─────────────┘
```

This means the "least connections" or "circuit breaker" decisions from your handout, at the *microservice-to-microservice* level, are actually implemented here — one layer above kube-proxy's simpler L4 randomness.

---

## 7. Horizontal Scaling in Kubernetes — the direct link to "thousands of servers"

Your handout's horizontal scaling concept maps onto **two independent Kubernetes autoscalers**:

- **Horizontal Pod Autoscaler (HPA)** — scales the **number of Pod replicas** for a Deployment up/down based on CPU/memory/custom metrics (e.g., requests-per-second). This is *horizontal scaling of containers*.
- **Cluster Autoscaler** — scales the **number of Nodes (machines)** in the cluster up/down, adding new VMs when existing nodes can't fit more Pods, removing nodes when they're underused. This is *horizontal scaling of machines*.

Crucially: **every new Pod created by HPA is automatically picked up by the Service's EndpointSlice**, and therefore automatically starts receiving load-balanced traffic — no manual load balancer reconfiguration needed. Similarly, every new Node added by Cluster Autoscaler is automatically usable by the scheduler and reachable by kube-proxy's networking rules.

This tight, automatic feedback loop — **scale Pods/Nodes → Kubernetes auto-updates Services/EndpointSlices → load balancer immediately starts using new capacity** — is precisely why Kubernetes + cloud LBs are the default stack for companies running thousands of microservices across thousands of servers.

---

## 8. Summary table — the full "load balancer stack" inside and around Kubernetes

| Layer | Component | Operates at | Algorithm/mechanism |
|---|---|---|---|
| Global | GSLB / Anycast / GeoDNS | DNS / Network | Geography, latency, region health |
| Cloud edge | Cloud LB (ALB/NLB/GCP LB) | L4/L7 | Provider-specific, usually round robin/least outstanding requests |
| Cluster edge | Ingress Controller (Nginx/Traefik) | L7 | Path/host routing, round robin/least conn per upstream |
| Cluster-internal (Service) | kube-proxy (iptables/IPVS) | L4 | Random (iptables) or rr/lc/sh/dh (IPVS) |
| Service-to-service | Service mesh sidecar (Envoy) | L7 | Round robin, least request, consistent hash, + circuit breakers |

Every one of these is doing exactly the same conceptual job your handout describes — "pick a healthy backend using an algorithm" — just at a different altitude of the system. The next file, `05-end-to-end-case-study.md`, walks a single user request through **every single layer in this table**, in order, using a fictional global company.
