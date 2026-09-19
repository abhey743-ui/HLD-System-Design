# Functional & Non-Functional Requirements in HLD

## 1. Why This Matters

Before you draw a single box in a system design diagram, you need to know **what the system must do** and **how well it must do it**. This is the very first step of any HLD (High Level Design) interview or real-world architecture — and interviewers judge you heavily on whether you ask the right questions here.

Requirements are split into two categories:

- **Functional Requirements (FR)** — *What* the system does
- **Non-Functional Requirements (NFR)** — *How well* the system does it

```
Functional Requirement:      "Users can upload a video"
Non-Functional Requirement:  "The upload must complete within 2 seconds for a 100MB file, 
                               even with 1 million concurrent users"
```

Both are equally important. A system that does the right things but is slow, insecure, or crashes under load is a failed system — just as much as one that doesn't do the right things at all.

---

## 2. Functional Requirements (FR)

### What they are

Functional requirements describe the **actual features and behaviors** of the system — the things a user (or another system) can directly interact with or observe.

They answer: **"What should the system do?"**

### Characteristics

- Describe specific behavior or function
- Usually written as user actions or system responses
- Testable with clear pass/fail criteria (e.g., "user can log in with correct password")
- Directly tied to business/product requirements

### Examples (using a URL Shortener like Bit.ly)

| # | Functional Requirement |
|---|---|
| 1 | User can submit a long URL and receive a shortened URL |
| 2 | User can click the shortened URL and get redirected to the original URL |
| 3 | User can set a custom alias for their short URL |
| 4 | User can set an expiration date for a short URL |
| 5 | System should track number of clicks per short URL |
| 6 | User can create an account and view their URL history |

### Examples (using an E-Commerce site)

- User can search for products
- User can add items to a cart
- User can place an order
- User can track order status
- Admin can add/remove products from inventory
- System sends order confirmation email

### How to identify FRs in an interview

Ask yourself:
- What actions can a user take?
- What actions can the system take automatically (e.g., send notification, expire a link)?
- What are the core use cases / user stories?

**Tip:** Functional requirements typically come straight from the **feature list** the interviewer or product spec gives you. If asked "design Twitter," FRs would include: post a tweet, follow a user, like a tweet, view a timeline, etc.

---

## 3. Non-Functional Requirements (NFR)

### What they are

Non-functional requirements describe the **quality attributes** and **operational characteristics** of the system — they don't add new features, but define the standard the features must meet.

They answer: **"How should the system behave?"**

### The Most Common NFRs (with explanations)

#### 3.1 Scalability
The system's ability to handle growth — more users, more data, more traffic — without degrading performance.
> *"System should support 10 million daily active users and scale horizontally."*

#### 3.2 Availability
The percentage of time the system is up and operational. Often expressed in "nines."

| Availability | Downtime per year |
|---|---|
| 99% ("two nines") | ~3.65 days |
| 99.9% ("three nines") | ~8.76 hours |
| 99.99% ("four nines") | ~52.6 minutes |
| 99.999% ("five nines") | ~5.26 minutes |

> *"System should be available 99.99% of the time."*

#### 3.3 Reliability
The system consistently produces correct results and doesn't lose data, even during failures.
> *"No uploaded file should ever be lost, even if a server crashes mid-upload."*

#### 3.4 Latency
The time taken to respond to a single request. Lower is better.
> *"API response time should be under 200ms for 95% of requests (P95 latency)."*

#### 3.5 Throughput
The number of requests/operations the system can handle per unit time.
> *"System must support 50,000 requests per second at peak."*

#### 3.6 Consistency
How up-to-date and synchronized data is across the system, especially in distributed systems (ties into the CAP theorem).
> *"All users should see the same tweet count within 1 second of a like being added" (eventual consistency)*
> vs
> *"Bank balance must be immediately consistent after a transaction" (strong consistency)*

#### 3.7 Durability
Once data is written/committed, it should never be lost — even in power failure or disk crash (usually achieved via replication/backups).
> *"Once a payment is confirmed, that record must survive server crashes."*

#### 3.8 Security
Protecting the system from unauthorized access, data breaches, and attacks.
> *"All user passwords must be hashed; all traffic must use HTTPS/TLS."*

#### 3.9 Fault Tolerance
The system should keep working (perhaps in a degraded state) even if some component fails.
> *"If one server in the cluster goes down, traffic should automatically reroute to healthy servers."*

#### 3.10 Maintainability
How easily the system can be updated, debugged, and extended by developers.
> *"Codebase should follow modular microservice boundaries to allow independent deployment."*

#### 3.11 Cost Efficiency
Achieving requirements without unnecessary infrastructure spend.
> *"Use auto-scaling to avoid over-provisioning during low-traffic hours."*

---

## 4. Quick Comparison Table

| Aspect | Functional Requirements | Non-Functional Requirements |
|---|---|---|
| Defines | What the system does | How well the system does it |
| Focus | Features, behavior | Quality, performance, constraints |
| Example | "User can reset password" | "Password reset email sent within 5 seconds" |
| Testable via | Feature tests / user acceptance tests | Load tests, benchmarks, monitoring |
| Comes from | Product requirements, user stories | Business SLAs, scale expectations, engineering standards |
| Interview signal | Shows you understood the problem | Shows you understand distributed systems trade-offs |

---

## 5. How to Gather These in an HLD Interview

A strong HLD answer always starts with clarifying both types before designing anything. A simple framework:

**Step 1 — Ask about Functional Requirements**
- "What are the core features I should focus on?"
- "Is [X] feature in scope, or can we assume it's out of scope?"

**Step 2 — Ask about Non-Functional Requirements (scale numbers matter a lot here)**
- "How many users are we expecting — daily active users, monthly active users?"
- "What's the read-to-write ratio?" (e.g., Twitter is read-heavy, most systems are)
- "What latency is acceptable?"
- "Do we need strong consistency or is eventual consistency okay?"
- "What's the expected data volume/growth over time?"

### Example: Designing "Instagram" (mini walkthrough)

**Functional Requirements:**
- Users can upload photos/videos
- Users can follow other users
- Users can view a feed of posts from people they follow
- Users can like/comment on posts

**Non-Functional Requirements:**
- Should support 500M daily active users
- Feed should load in under 200ms
- System should be highly available (99.99%)
- Should be eventually consistent (a like doesn't need to show up instantly for everyone)
- Should be able to store and serve petabytes of image/video data reliably (durability)

---

## 6. Common Mistakes to Avoid

- ❌ Jumping straight into architecture diagrams without listing FRs/NFRs first
- ❌ Treating NFRs as an afterthought — they often **drive the entire architecture** (e.g., strong consistency requirement rules out many NoSQL options)
- ❌ Not prioritizing — you can't optimize for everything at once (e.g., strong consistency often trades off against availability/latency — CAP theorem)
- ❌ Forgetting to ask about scale (number of users, requests/sec, data size) — this single-handedly changes your entire design

---

## 7. One-Line Summary

- **Functional Requirements** = the features — *what the user can do*
- **Non-Functional Requirements** = the guarantees — *how fast, how reliable, how available, how secure, how scalable*

Always gather both before designing. FRs shape your **API and features**. NFRs shape your **architecture, database choice, and infrastructure**.
