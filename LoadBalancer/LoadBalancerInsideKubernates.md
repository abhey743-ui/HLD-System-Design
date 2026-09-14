# How Load Balancing Actually Works Inside Kubernetes — The Simple Story

> This is a plain-language rewrite. No dense tables, no wall of YAML. Just the journey a request takes, step by step, told like a story — with the real Kubernetes terms attached at each step so you can still connect it to what you read/hear elsewhere.

---

## The One-Line Version (read this first)

**When a user hits your app's URL, the request doesn't go straight to a Pod. It passes through a chain of "doormen" — DNS, an external Load Balancer, an Ingress Controller, a Service, and finally kube-proxy — and only the last doorman actually knows which Pod to hand it to.**

Everything below is just zooming into each doorman, one at a time.

---

## The Analogy We'll Use

Imagine your app is a big company with a support hotline. A customer calls, and the call has to travel through several layers before it reaches an actual support agent sitting at a desk:

1. **Phonebook** → looks up the company's number
2. **Call center switchboard** (outside the building) → answers first, decides which office building to forward the call to
3. **Building reception desk** → reads the call, decides which department it's for
4. **Department dispatcher** → knows which agents are currently free
5. **The agent** → actually picks up and helps the customer

Kubernetes load balancing is exactly this, with different names. Let's walk through it in order.

---

## Step 0 — Before Any Request: Why Scaling Matters Here

Quick grounding, because this changes *why* load balancing exists at all:

- **Vertical scaling** = giving one agent (one Pod, or one server) a bigger desk — more CPU/RAM. There's still only **one** agent. If they're overwhelmed, there's no one else to hand the call to.
- **Horizontal scaling** = hiring **more agents** (more Pods, or more machines). Now there are many identical agents who could all answer the same kind of call.

Load balancing only becomes an interesting problem once you've scaled **horizontally** — because now something has to *decide which one of the many identical agents* gets each call. That "something" is the whole subject of this file.

In Kubernetes, horizontal scaling happens two ways:
- **Horizontal Pod Autoscaler (HPA)** — adds more Pods (more agents) when the app is busy.
- **Cluster Autoscaler** — adds more Nodes/machines (more office floors) when there's no room left for new Pods.

Keep this in the back of your mind: the number of Pods behind your app is **always changing**. That's the entire reason a load balancer needs to exist — it's the only thing that always knows the current, live list of who's available.

---

## Step 1 — DNS: "What's the phone number?"

The user types `shop.example.com` into their browser. Before anything else happens, DNS is asked: *"what IP address does this name point to?"*

DNS hands back the IP address of your **external Load Balancer**. That's it — DNS's job ends here. It's purely the phonebook lookup. No load balancing decision happens yet.

---

## Step 2 — The External Load Balancer: "Which building do I send this to?"

Now the browser connects to that IP address. This IP belongs to a **Load Balancer that lives outside your Kubernetes cluster** — usually a cloud provider's Load Balancer (AWS NLB/ALB, GCP LB, Azure LB), or a hardware one if you're on your own datacenter.

This is the **first real doorman**. Its job is simple:
- It sits in front of your **Kubernetes Nodes** (the machines that make up your cluster).
- It just needs to get the request onto *one of the Nodes* — it doesn't know or care about Pods at all. Pods are an internal Kubernetes concept; this outside Load Balancer has no idea they even exist.

In Kubernetes, this is what a **Service of type `LoadBalancer`** creates for you. When you write:

```yaml
spec:
  type: LoadBalancer
```

...Kubernetes quietly talks to your cloud provider's API behind the scenes and says *"please create a real load balancer and point it at my Nodes."* That's the magic — you write one small YAML file, and an actual cloud load balancer appears.

**So where does the load balancer "sit" exactly?** Right at the edge, *outside* the cluster, facing the internet. Everything from here on is happening *inside* the cluster.

---

## Step 3 — Reaching the Cluster: The Building's Front Door

The external Load Balancer forwards the request to a specific **port on a Node**. This is called a **NodePort** — think of it as "the building's front door has a specific door number."

Here's a detail that trips people up: it doesn't matter *which* Node the Load Balancer happens to pick. Every Node in the cluster is wired up (by `kube-proxy`, which we'll get to) to know how to forward the request onward — even if the Pod that ends up handling it lives on a completely different machine. So "landing on the right building" doesn't mean "landing on the right floor" yet. That's the next step.

---

## Step 4 — Ingress Controller: The Reception Desk (only for web traffic)

If your app only needs "send everything to one place," you could stop at Step 3. But most real apps have multiple services — `/checkout` should go to one app, `/catalog` to another. Plain load balancers don't understand URLs paths — they just move packets around (this is called **Layer 4 / L4**).

To route based on the URL path or hostname (**Layer 7 / L7**), Kubernetes uses an **Ingress**. Think of it as the receptionist who actually reads the request and says *"ah, checkout stuff goes to the 3rd floor, catalog stuff goes to the 5th floor."*

Important detail: an Ingress is just a **list of rules** (a YAML file). Rules alone don't do anything — you also need an **Ingress Controller**, which is actual running software (commonly Nginx, Traefik, or a cloud-native one) that reads those rules and does the actual routing. This controller runs as its own Pods, sitting right behind the external Load Balancer from Step 2.

So the honest picture is: *the first thing inside your cluster that touches the request is usually another Pod* (the Ingress Controller) — it's doormen all the way down.

---

## Step 5 — The Service: The Department Dispatcher

Whether the request came through an Ingress or went straight to a NodePort, it eventually arrives at a **Service**. This is the core Kubernetes load-balancing object, and it's worth understanding well.

A Service is **not a real running thing** — it's a stable, fixed address (a virtual IP + a DNS name like `checkout-service`) that sits in front of a **group of Pods that can change at any moment**. You define which Pods belong to it using a `selector`, e.g. "all Pods labeled `app: checkout`."

Why does this matter? Because Pods are **disposable** — they crash, get replaced, get scaled up/down, and get new IP addresses every time. You (and your other microservices) should never talk to a Pod IP directly, because it might not exist five seconds from now. The Service is the one thing that never changes, no matter how much scaling happens underneath it.

---

## Step 6 — kube-proxy: The Actual Decision-Maker

Here's the part that answers "how does it *actually* pick a Pod?"

`kube-proxy` runs on **every single Node** in your cluster. Its whole job is: *make traffic sent to a Service's fixed address actually land on one real, healthy Pod.* It does this by programming rules directly into the Node's operating system (not by running its own separate proxy server that traffic has to pass through). Two ways it can do this:

- **iptables mode (older, still common):** picks a Pod basically **at random** (weighted evenly). Simple, works fine at small-to-medium scale.
- **IPVS mode (used in bigger clusters):** lets you actually choose an algorithm — round robin, least connections, hashing based on source IP for "sticky sessions," and a few others. This is the same load-balancing-algorithm menu people study in general networking, just running inside the kernel.

This is the moment the "decision" actually happens. Everything before this step (DNS, external LB, Ingress) was just about getting the request to the right neighborhood. **kube-proxy is the one that picks the actual Pod.**

---

## Step 7 — Health Checks: "Don't call an agent who stepped away from their desk"

A Service doesn't blindly load-balance across every Pod matching its label — only the ones that are actually ready to work. Here's how that's tracked:

- Every Pod has a **readiness probe** — a small, repeated health check (e.g., "hit `/health` every few seconds and expect a 200 OK").
- Kubernetes keeps a live, auto-updated list called an **EndpointSlice** — literally "the current list of Pod IPs that are healthy and allowed to receive traffic right now."
- The moment a Pod fails its readiness check, it's **instantly removed** from that list. `kube-proxy` immediately stops sending it traffic. Importantly: **the Pod itself isn't killed** — it might just be busy starting up or briefly struggling, and it can rejoin the list the moment it passes its check again.
- If a Pod crashes completely, Kubernetes' Deployment controller notices and creates a brand-new replacement Pod (with a new IP). Once *that* Pod passes its readiness check, it gets added to the list too.

This is the exact mechanism that makes horizontal scaling "just work": the moment HPA creates 10 new Pods to handle traffic, those Pods automatically join the EndpointSlice as soon as they're healthy, and kube-proxy immediately starts sending them traffic — **with zero manual reconfiguration of any load balancer.**

---

## Step 8 — The Pod Finally Handles It

The chosen, healthy Pod receives the request on its container port, processes it, and sends the response back the exact same way it came in — Pod → kube-proxy's rules → Node → external Load Balancer → DNS-resolved connection → back to the user's browser.

---

## Putting the Whole Journey Together

```
User types shop.example.com
        │
        ▼
   DNS lookup                     "What's the number?"
        │
        ▼
External Load Balancer            "Which building?" (cloud LB, outside the cluster)
        │
        ▼
NodePort on some Node              "Front door of the building"
        │
        ▼
Ingress Controller (optional)      "Reception — which department based on the URL?"
        │
        ▼
   Kubernetes Service              "Fixed address for a changing group of Pods"
        │
        ▼
      kube-proxy                   "Actually picks ONE healthy Pod (the real decision)"
        │
        ▼ (only Pods in the current EndpointSlice are eligible — this is the health-check gate)
        ▼
        Pod                        "The agent who actually does the work"
```

---

## Quick Answers to the Exact Questions You Asked

- **"Where does the load balancer sit, exactly?"** — There are actually two of them, at different points: one **outside** the cluster (the cloud/external Load Balancer, facing the internet) and one **conceptually inside** the cluster (the Service + kube-proxy combo, which load-balances across Pods). People often mean the second one when they say "Kubernetes does load balancing."
- **"How does the request get from the LB into the k8s environment?"** — Via a **NodePort**: the external LB forwards to a specific port that's open on every Node, and Kubernetes' internal networking (kube-proxy + the CNI plugin) takes it from there — even routing it across to a different machine if needed.
- **"How is the decision made about which Pod?"** — `kube-proxy`, using either simple random selection (iptables mode) or a real algorithm like round robin/least connections (IPVS mode) — but only ever choosing from the pool of Pods that are currently marked healthy.
- **"How does the health check work?"** — Readiness probes run continuously against each Pod. Failing one instantly (and temporarily) pulls that Pod out of rotation via the EndpointSlice, without killing it.
- **"How does this connect to scaling?"** — Horizontal scaling (more Pods, more Nodes) is exactly what makes this whole chain necessary in the first place, and it's designed so that new capacity is picked up automatically the moment it's healthy — no manual load balancer updates required.

---

## Step 9 — The Next Layer: Service Mesh (load balancing *between your own services*)

Everything in Steps 1-8 solves one problem: **getting an outside user's request into the cluster and onto a healthy Pod.** But once your app is actually running, that Pod usually needs to call *other* Pods — Checkout calls Inventory, Inventory calls Payments, Payments calls Notifications, and so on, hundreds of times a second. That's a completely different traffic pattern (**inside** the cluster, service-to-service), and kube-proxy's simple "pick a random healthy Pod" starts to feel too basic for it. This is where a **service mesh** (Istio, Linkerd, Consul Connect) comes in.

### Back to the analogy: everyone gets a personal assistant

Picture every single agent in our support-hotline company now getting their **own personal assistant** sitting right next to them. Here's the trick: the agent never dials another department directly anymore — every outgoing and incoming call physically passes through their assistant first, automatically, without the agent even noticing. In Kubernetes terms, this assistant is a tiny proxy container (Istio uses one called **Envoy**) that gets automatically injected into every Pod, right alongside your actual app container. This is called the **sidecar pattern**.

Because *every* call — in and out — now passes through an assistant, the assistants can quietly handle a bunch of things the agents themselves never have to think about:

- **Smarter load balancing, per call.** Instead of kube-proxy's simple random pick, the assistant can use real strategies — round robin, "send it to whoever currently has the fewest calls in progress" (least connections), or even "always send this specific caller to the same agent" (consistent hashing, useful for keeping a user's session on the same backend Pod).

- **Automatic retries.** If an assistant dials another department and the line is briefly busy or drops, it can quietly redial a couple of times on its own — the original agent never even knows there was a hiccup.

- **Circuit breaking.** If a whole department (say, Payments) is clearly struggling — timing out, erroring constantly — the assistants calling it will notice and temporarily **stop sending it new calls entirely**, giving it room to recover instead of piling on more load and making things worse. This is the exact same "circuit breaker" idea from general system design, just enforced automatically at the network layer.

- **ID checks on every call (mTLS).** Every assistant verifies the identity of the assistant on the other end before letting the call through — so services only ever talk to other verified services, automatically encrypted, without your actual app code having to do anything about security.

- **A paper trail for every call.** Since literally 100% of traffic flows through these assistants, they can log exactly how long every call took, who called whom, and how often it failed — giving you rich, automatic observability across your entire company without instrumenting every single agent by hand.

### Where this sits in the picture

This layer sits **one level deeper than everything before it**. Steps 1-8 got the request as far as "Pod A." Step 9 is what happens *after* Pod A decides it needs to call Pod B, Pod C, etc. to actually do its job:

```
        (Steps 1-8: request already arrived at Pod A)
                        │
                        ▼
        Pod A's App Container needs to call Pod B
                        │
                        ▼
        Pod A's own sidecar assistant (Envoy)   ← load balancing, retries,
                        │                          circuit breaking, ID check
                        ▼
        Pod B's sidecar assistant (Envoy)        ← receives, verifies, hands off
                        │
                        ▼
        Pod B's App Container actually handles it
```

Notice this is a **separate, additional decision point** from kube-proxy — it doesn't replace kube-proxy, it works on top of it. Your Service and EndpointSlice from Step 5-7 still exist and still track which Pods are healthy; the sidecar just makes a *smarter* choice among them and adds all the safety features above.

### The honest trade-off

A service mesh isn't free — every Pod now runs an extra container, every call has an extra tiny hop through a proxy, and it's genuinely one of the more complex pieces of infrastructure to operate. Teams generally only reach for it once they have enough microservices talking to each other that "who's calling whom, how reliably" has become hard to reason about on its own. If you want to go deeper into exactly how Istio itself works — the sidecar injection, mTLS setup, traffic-splitting for canary releases — that's worth its own dedicated, hands-on walkthrough rather than cramming it into this file.
