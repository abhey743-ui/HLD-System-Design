# DNS and Global Traffic Management

> Part 2 of 5. See `01-load-balancers-fundamentals.md` for base concepts.

Before a request ever reaches *any* load balancer, it has to answer one question: **what IP address does `yourapp.com` even point to?** That's DNS's job — and DNS turns out to be the *very first* (and crudest) layer of load balancing in almost every real system.

---

## 1. What is DNS?

**DNS (Domain Name System)** is the internet's phonebook. Humans use names (`google.com`), computers use IP addresses (`142.250....`). DNS translates one into the other.

## 2. How DNS Resolution Actually Works (step by step)

When you type `yourapp.com` into a browser:

1. **Browser cache check** — has this been resolved recently? If yes, use the cached IP.
2. **OS cache check** — same idea, at the operating system level.
3. **Recursive resolver** — if not cached, your device asks a **recursive DNS resolver** (usually run by your ISP, or a public one like `8.8.8.8` Google DNS, or `1.1.1.1` Cloudflare).
4. **Root nameserver** — the recursive resolver asks a **root server**, which doesn't know the answer but knows *who to ask next* — it points to the `.com` TLD servers.
5. **TLD nameserver** — the `.com` **Top-Level-Domain server** doesn't know the IP either, but knows which **authoritative nameserver** is responsible for `yourapp.com`.
6. **Authoritative nameserver** — this is the *actual source of truth* for your domain (often your DNS provider — Route53, Cloudflare, Google Cloud DNS). It returns the real IP address(es).
7. The recursive resolver **caches** this answer for the record's TTL (Time To Live) and returns it to your browser.
8. Your browser now connects directly to that IP — which, in a real production system, is the address of a **load balancer**, not an individual app server.

```
Browser → OS cache → Recursive Resolver → Root server → .com TLD server
                                                              │
                                                              ▼
                                              Authoritative NS (Route53/CloudDNS)
                                                              │
                                                              ▼
                                                     Returns IP of the LB
```

## 3. Key DNS Record Types (the ones that matter for load balancing)

| Record | Purpose |
|---|---|
| **A** | Maps a domain to an IPv4 address |
| **AAAA** | Maps a domain to an IPv6 address |
| **CNAME** | Maps a domain to *another domain name* (can't be used at the root/apex domain in most cases) |
| **ALIAS / ANAME** | Like CNAME but usable at the root domain — commonly used to point `yourapp.com` straight at a cloud LB's DNS name (e.g., an AWS ALB) |
| **NS** | Delegates a subdomain to a different set of nameservers |

## 4. DNS Round Robin — the "poor man's load balancer"

You can actually put **multiple A records** under one domain name:

```
yourapp.com  A  203.0.113.10
yourapp.com  A  203.0.113.11
yourapp.com  A  203.0.113.12
```

DNS resolvers will hand these out in rotation (or randomized order), spreading clients across multiple IPs. This *sounds* like load balancing, but it's a weak version of it:

**Problems:**
- **No health checks** (by default) — if `203.0.113.11` is dead, DNS keeps handing it out anyway until someone manually removes the record.
- **Caching/TTL issues** — resolvers and clients cache the answer. If you remove a bad server, clients who already cached its IP keep hitting it until the TTL expires (this can be minutes to hours).
- **No awareness of real load** — it just rotates blindly, no concept of "least busy server."
- **Uneven distribution** — because of caching at multiple layers (ISP resolvers serving many users), traffic doesn't spread evenly at all.

Because of this, DNS Round Robin is usually only used as one *ingredient* of a bigger system — pointing to a small number of **regional load balancer entry points**, not to individual app servers.

## 5. GeoDNS / Latency-based Routing

A smarter DNS setup answers differently *depending on where the query is coming from*:

- A user in Mumbai querying `yourapp.com` gets the IP of the **Asia-Pacific region's load balancer**.
- A user in London gets the IP of the **Europe region's load balancer**.

This is called **GeoDNS** (routing by geographic location of the resolver) or **latency-based routing** (routing to whichever region historically responds fastest to that user's network). Cloud providers offer this as a managed feature:
- **AWS Route53** — Geolocation routing policy, Latency-based routing policy.
- **GCP Cloud DNS** — combined with Cloud Load Balancing's Global anycast IP (see below).
- **Azure Traffic Manager** — Performance routing, Geographic routing.

## 6. GSLB — Global Server Load Balancing

**GSLB** is the general term for "load balancing across entire data centers / regions," and it's usually built from a combination of:

1. **DNS-based routing** (GeoDNS/latency-based, as above), **plus**
2. **Continuous health checks of entire regions** — if an entire region/data center goes down, GSLB stops sending traffic there and reroutes everyone to the next-best region, **plus**
3. Sometimes **Anycast IP routing** (see below) instead of DNS tricks entirely.

This is the top-most layer of load balancing in a global system — deciding *which continent/region* handles a request, before any server-level load balancing even begins.

## 7. Anycast Routing (the more modern alternative to GeoDNS)

Instead of handing out *different IPs* to different users (GeoDNS), **Anycast** advertises the **exact same IP address** from multiple physical locations around the world using BGP (Border Gateway Protocol) at the network routing layer. The internet's own routing infrastructure automatically sends each user's traffic to the *nearest* location advertising that IP.

- Used by: Cloudflare, Google Cloud's Global HTTP(S) Load Balancer, AWS Global Accelerator, most large CDNs.
- **Advantage over GeoDNS:** no DNS caching/TTL delay — if a location goes down, BGP reroutes almost instantly, and it works at the network layer so it's extremely fast and resilient.
- **How it feels to an engineer:** you get *one single IP address* for your entire global service, and the network itself figures out routing. This is why AWS Global Accelerator / GCP Global LB can offer a single global anycast IP that "just works" everywhere.

## 8. CDN's relationship to Load Balancing

A **CDN (Content Delivery Network)** — Cloudflare, Akamai, CloudFront, Fastly — is, at its core, a massively distributed anycast + caching layer:
- Static assets (images, JS, CSS, video) are cached at **edge locations** close to users, so most requests never even reach your origin servers.
- Dynamic/API requests that *can't* be cached get proxied onward to your actual regional load balancers.

So in a real modern stack, the very first "load balancer" a request meets is often the CDN's edge network, and only uncacheable traffic continues on to your GSLB → regional LB → application servers.

## 9. Putting it together — the full "before it even reaches your servers" journey

```
User types yourapp.com
        │
        ▼
   DNS Resolution (GeoDNS / latency-based / Anycast)
        │
        ▼
   Nearest CDN Edge / Global Anycast entry point
        │  (static assets served from cache here — request ends)
        │  (dynamic requests continue)
        ▼
   Regional Load Balancer (this is where file 1's L4/L7 concepts apply)
        │
        ▼
   Backend servers / Kubernetes cluster (file 4)
```

This is exactly the layered flow the case study in `05-end-to-end-case-study.md` walks through with a real example company.
