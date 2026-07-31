# Load Balancing — Interview Notes (SDE-1 / SDE-2 / Intern)

---

## 1. Load Balancer — The Basics

### What is it?
A **load balancer (LB)** sits between clients and a pool of backend servers. It receives incoming requests and distributes them across servers so that no single server gets overwhelmed.

Think of it as a **traffic cop** — it never does the actual work, it just decides *who* does the work.

### Simple Diagram
```
                          ┌───────────┐
                          │  Server 1 │
                       ┌─▶│  (up)     │
                       │  └───────────┘
 Clients ──▶ ┌────────────────┐       ┌───────────┐
   many      │  Load Balancer │──────▶│  Server 2 │
  requests   │ (traffic cop)  │       │  (up)     │
             └────────────────┘       └───────────┘
                       │
                       │  ┌───────────┐
                       └─▶│  Server 3 │
                          │  (down) ✗ │  <- LB stops sending here
                          └───────────┘
```

### Real-life analogy
A busy restaurant kitchen: instead of one chef cooking every order, the head chef (LB) assigns each new order to whichever chef (server) is free. If a chef goes on a break/collapses (server down), orders stop going to them.

### Why do we need it? (Problems without LB)
| Problem | What happens |
|---|---|
| Single Point of Failure | One server down → whole app down |
| Overloaded servers | Traffic grows, one server can't keep up → slow/crash |
| Limited scalability | Can't just "add a server" — nothing routes traffic to it |

### How it works (step by step)
1. **Receives request** from client (instead of client hitting a server directly).
2. **Checks server health** (via heartbeats / health checks).
3. **Picks a server** using a load-balancing algorithm.
4. **Forwards request**, and if a server fails mid-way, **reroutes** to a healthy one.

### Real-world examples
- **NGINX / HAProxy** — software LBs, often in front of microservices.
- **AWS ELB / ALB / NLB** — managed cloud load balancers.
- **F5 BIG-IP** — hardware/enterprise LB appliances.

### Key characteristics (good to say in interviews)
- Traffic distribution
- High availability (fails over to healthy servers)
- Horizontal scalability (easy to add/remove servers)
- Health monitoring (active + passive)
- SSL/TLS termination (LB decrypts HTTPS so backend servers don't have to)

### ⚠️ Watch out: LB is also a single point of failure
If the LB itself dies, everything behind it becomes unreachable — that's why in production you run **multiple LBs** (active-passive or active-active), often behind a DNS layer or virtual IP (VIP) with a heartbeat protocol like VRRP.

---

## 2. Types of Load Balancer

### A) By Deployment (the "what runs it" axis)

| Type | What it is | Example | Pros | Cons |
|---|---|---|---|---|
| **Hardware LB** | Dedicated physical appliance | F5 BIG-IP, Citrix ADC | Very high performance, reliable | Expensive, inflexible, needs physical setup |
| **Software LB** | App running on a normal server/VM | NGINX, HAProxy | Cheap, flexible, easy to deploy | Limited by underlying server's hardware |
| **Virtual/Cloud LB** | Managed service in the cloud | AWS ELB, Kemp VLM | Auto-scaling, no infra to manage | Vendor lock-in, usage-based cost |

**Common real pattern:** Hardware LB at the network edge (entry point) → Software LB internally between microservices → Cloud LB for elastic scaling during traffic spikes.

### B) By OSI Layer (the "how smart is the routing" axis — **most asked in interviews**)

```
        L7 Load Balancer (Application Layer)
        "I can read your HTTP request"
                │
   routes /api/* → API servers
   routes /images/* → static server
   routes based on cookie → same user, same server
                │
        L4 Load Balancer (Transport Layer)
        "I only see IP + Port, not content"
                │
   routes TCP/UDP packets by IP:Port only
   very fast, doesn't open/read the packet
```

| | Layer 4 (Network/Transport) | Layer 7 (Application) |
|---|---|---|
| Sees | IP address, TCP/UDP port | HTTP headers, URL, cookies, body |
| Speed | Very fast (no content inspection) | Slower (has to parse request) |
| Routing intelligence | Dumb — just forwards by IP/port | Smart — can route by URL path, header, etc. |
| Example | HAProxy in TCP mode, NLB | NGINX, AWS ALB |
| Use case | High-throughput, low-latency (gaming, video) | Microservices routing, A/B testing, canary deploys |

### C) GSLB — Global Server Load Balancer
Routes traffic across **geographically distributed data centers**, not just servers in one rack.
- Example: Cloudflare Load Balancing, AWS Route 53 latency-based routing.
- A user in India gets routed to the Mumbai data center; a user in the US gets routed to Virginia.
- Real life: Netflix routes you to the nearest edge server so your show doesn't buffer.

### 💡 Classic interview question
**"L4 vs L7 — when would you pick one over the other?"**
> Pick **L4** when you need raw speed and don't care about request content (e.g., a game server, a database proxy). Pick **L7** when you need smart routing — like sending `/checkout` to a beefier server, or doing SSL termination, or sticky sessions based on cookies.

---

## 3. Load Balancing Algorithms

```
                Load Balancing Algorithms
                     │
        ┌────────────┴────────────┐
     STATIC                    DYNAMIC
  (fixed rules,             (real-time server
   no real-time data)         conditions)
        │                          │
  ┌─────┼─────┐          ┌─────────┼──────────┐
Round  Weighted  Source  Least   Least Response  Resource
Robin  R.Robin   IP Hash Conn.   Time             Based
```

### Static Algorithms (predefined rules)

| Algorithm | How it works | Real-life analogy | Pros | Cons |
|---|---|---|---|---|
| **Round Robin** | Server 1 → 2 → 3 → 1 → 2 ... in a loop | Dealing cards one-by-one to each player | Dead simple | Ignores actual load — a slow server still gets equal share |
| **Weighted Round Robin** | Same as RR, but stronger servers get more turns (weight) | Giving the strongest waiter more tables | Accounts for server capacity | Still doesn't adapt to real-time load |
| **Source IP Hash** | `hash(client IP) % servers` → same client always → same server | Always being served by "your regular" waiter | Session persistence, no shared session store needed | Uneven load if some IPs send way more traffic (e.g., NAT'd office network) |

### Dynamic Algorithms (real-time decisions)

| Algorithm | How it works | Real-life analogy | Pros | Cons |
|---|---|---|---|---|
| **Least Connections** | Send new request to server with fewest **active** connections | Joining the shortest checkout line | Adapts to real load | Needs to track connections live — more overhead |
| **Least Response Time** | Picks server with lowest recent latency | Joining the fastest-moving line, not just the shortest | Optimizes for user-perceived speed | Needs historical monitoring data |
| **Resource-Based** | Checks actual CPU/RAM/bandwidth on each server (via an agent) | Manager sends new customers to whichever employee isn't sweating | Best accuracy | Most complex, monitoring overhead |

### Quick decision cheat-sheet (great to say out loud in interviews)
- Small app, similar servers → **Round Robin**
- Servers have different capacities → **Weighted Round Robin**
- Need same user to hit same server (shopping cart, WebSocket) → **Source IP Hash** (or sticky sessions at L7)
- Requests vary wildly in duration → **Least Connections**
- Latency is critical (e.g. gaming, trading) → **Least Response Time**
- Cloud/containers with fluctuating resource usage → **Resource-Based**

---

## 4. Concurrency vs Parallelism

### The one-line difference
> **Concurrency** = dealing with many things at once (structure).
> **Parallelism** = doing many things at once (execution).

### Analogy
- **Concurrency**: One cashier quickly switching between 3 customers — swipe card for customer A, while bagging for customer B, while printing receipt for C. Only **one** thing physically happens at a time, but progress overlaps.
- **Parallelism**: Three cashiers, three registers, three customers — all happening **literally simultaneously**.

### Diagram
```
CONCURRENCY (1 core, context switching)
Core:  [Task1][Task2][Task3][Task1][Task2][Task3]...
        (interleaved, gives illusion of "at once")

PARALLELISM (multi-core, true simultaneity)
Core1: [ Task1 ][ Task1 ][ Task1 ]
Core2: [ Task2 ][ Task2 ][ Task2 ]
Core3: [ Task3 ][ Task3 ][ Task3 ]
        (all running at the exact same time)
```

### Table
| | Concurrency | Parallelism |
|---|---|---|
| Cores needed | 1 is enough | needs 2+ |
| Mechanism | Context switching / interleaving | Simultaneous execution |
| Goal | Responsiveness (don't block) | Throughput (do more, faster) |
| Example | Node.js handling 1000s of HTTP requests on one thread via event loop | Splitting an image into 4 tiles, processing each on a separate CPU core |
| Debugging | Hard (non-deterministic interleavings) | Hard, but somewhat simpler than concurrency bugs |

### Why this shows up in a "load balancing" interview
Because **a single backend server handles concurrent requests** (via threads/event loop), while a **load balancer + multiple servers = parallelism** at the system level. Interviewers like to check you understand both layers.

---

## 5. Stateless vs Stateful Load Balancing

### Stateless
The LB treats every request independently — no memory of past requests.

```
Request 1 (user A) → Server 2
Request 2 (user A) → Server 1   <- different server, doesn't matter!
Request 3 (user A) → Server 3
```
- **Analogy**: Ordering fast food — any cashier can take your next order, they don't need to remember your last visit.
- **Used for**: REST APIs, microservices, CDNs, search engines — anything where each request carries all info it needs (e.g., a JWT token in the header).

### Stateful
The LB remembers which server is handling a client's *session* and keeps sending them there ("sticky sessions").

```
Request 1 (user A) → Server 2  (session created)
Request 2 (user A) → Server 2  (same server, remembers session)
Request 3 (user A) → Server 2
```
- **Analogy**: Your regular doctor — you keep going back to the *same* doctor who has your history, instead of a new one every time.
- **Used for**: Shopping carts, banking sessions, multiplayer game servers, live chat/collaboration tools.

### Table
| | Stateless | Stateful |
|---|---|---|
| Session tracking | None | Maintained (sticky sessions) |
| Scalability | Very easy — add servers freely | Harder — session data ties client to a server |
| Fault tolerance | High — any server can serve any request | Risk of session loss if that server dies |
| Complexity | Simple | Complex (needs session replication/shared store) |
| Example | Public REST API | Online banking, e-commerce cart |

### 💡 Common follow-up interview question
**"How do you make a 'stateful-feeling' app scale like stateless?"**
> Move session state out of the server into a shared store — Redis, Memcached, or a database — so *any* server can serve *any* request but still access the same session data. This is called making the **servers themselves stateless** even though the *application* has state.

---

## 6. Load Balancing vs Failover

### The one-line difference
> **Load Balancing** = "let's share the work so nobody's overwhelmed" (performance).
> **Failover** = "if the main one dies, switch to backup" (availability).

### Diagram
```
LOAD BALANCING                     FAILOVER
   Client                             Client
     │                                  │
     ▼                                  ▼
 [Load Balancer]                  [Primary Server] ✗ (down)
   │    │    │                          │
   ▼    ▼    ▼                          ▼ (detected failure, switch)
  S1   S2   S3                   [Standby Server] (takes over)
 (all active,                    (idle until needed)
  sharing load)
```

### Real-life examples
- **Load Balancing**: Flash sale on an e-commerce site — thousands of users hit the site at once, traffic is spread across many servers so none crash.
- **Failover**: Online banking — if the primary database server crashes, a standby takes over instantly so transactions aren't interrupted.

### Table
| | Load Balancing | Failover |
|---|---|---|
| Goal | Optimize performance / distribute traffic | Maintain availability during failure |
| Normal operation | All servers active, sharing load | Standby is usually idle |
| Improves speed? | Yes | No — it's a safety net, not a performance tool |
| Cost | Higher infra (all nodes doing real work) | Backup nodes may sit unused (wasted cost) |
| Complexity | More setup/monitoring | Generally simpler |

### 💡 Interview gotcha
They're **not mutually exclusive** — most production systems use **both**: a load balancer distributes traffic across active servers, *and* failover logic exists in case the load balancer itself (or a whole region) goes down. Mention this — it shows systems-level thinking.

---

## 7. Consistent Hashing

### The problem it solves
With plain hashing, `server = hash(key) % N`. If you add/remove a server, **N changes**, and almost **every key remaps** to a different server. In a cache, that's a disaster — everything becomes a cache miss at once.

### The idea
Put both **servers** and **keys** on a **hash ring** (0 to 2³²-1, imagined as a circle). A key is assigned to the **first server found going clockwise** from its position.

### Diagram
```
                     0/360°
                       │
        Node_A ●───────┼───────
              /         │       \
            /           │         \
    key1 ● /            │           \
          /              │             ● Node_C
         /               │            /
        │                │           /
         \               │          /
    key2 ●\              │        /
            \            │      /
              \          │    /
               ●─────────┼──/
              Node_B     │

key1 → clockwise → hits Node_A first
key2 → clockwise → hits Node_B first
```
- If **Node_A fails**, only the keys between the *previous* node and Node_A get remapped (to the next node clockwise) — everything else is untouched.
- Compare to `% N` hashing, where a single node change reshuffles **almost everything**.

### Real-life analogy
Think of a circular parking arrangement with valet stations. Each car (key) just walks clockwise to the nearest open valet station (server). If one valet station closes, only the cars that were headed there need to walk a bit further to the next one — everyone else's assignment is unaffected.

### Virtual Nodes (important add-on, often asked)
Problem: with few real servers, they might land unevenly on the ring, causing hot spots.
Fix: each physical server is represented by **many virtual nodes** scattered around the ring (e.g., Node_A_0, Node_A_1, ... Node_A_99), smoothing out the distribution.

### Where it's used in real systems
- **Amazon DynamoDB**, **Cassandra** — partitioning data across nodes.
- **Redis Cluster / Memcached** — distributed caching.
- **CDNs** — mapping content to edge servers.

### Advantages / Disadvantages
| Advantages | Disadvantages |
|---|---|
| Minimal remapping on scale up/down | Needs a good hash function |
| Great for caches & distributed DBs | Extra complexity vs plain `% N` |
| Improves fault tolerance | Virtual nodes add bookkeeping overhead |

### 💡 Classic interview question
**"Why not just use `hash(key) % N`?"**
> Because when N changes (server added/removed), *nearly all* keys remap — causing a "rehashing storm" (mass cache misses / massive data movement). Consistent hashing bounds the disruption to roughly `K/N` keys (K = total keys, N = servers), not all of them.

---

## 8. Rapid-Fire Interview Q&A (for SDE-1 / SDE-2 / Intern)

**Q1. What is a load balancer and why do we need it?**
> A component that distributes incoming traffic across multiple servers to avoid overload, improve availability, and enable horizontal scaling. Without it: single point of failure, overloaded servers, poor scalability.

**Q2. Difference between L4 and L7 load balancing?**
> L4 routes based on IP/port only (fast, "dumb"); L7 routes based on HTTP content — URL, headers, cookies (slower, "smart", enables things like path-based routing and SSL termination).

**Q3. How does a load balancer detect a server is down?**
> Active health checks (periodic pings/HTTP checks) + passive health checks (watching real traffic for errors/timeouts). If checks fail repeatedly, the server is marked unhealthy and removed from rotation; it's added back once it passes checks again.

**Q4. Round Robin vs Least Connections — when would each fail you?**
> Round Robin fails when requests have very different costs (a slow request piles up on one server while others finish fast and idle). Least Connections handles that better since it reacts to actual load, but costs more to compute/track.

**Q5. What's a "sticky session" and what's the trade-off?**
> LB routes the same client consistently to the same server (via cookie or IP hash) so session state doesn't need to be shared. Trade-off: less flexible load distribution, and if that server dies, you can lose the session unless it's replicated elsewhere.

**Q6. How would you make a stateful app horizontally scalable?**
> Externalize session state to a shared store (Redis/Memcached/DB) so the app servers themselves become stateless, and any server can serve any request.

**Q7. What is consistent hashing and why use it over `mod N`?**
> It maps servers and keys onto a hash ring so only a fraction of keys move when a node is added/removed, unlike `%N` hashing where nearly everything remaps. Used in distributed caches (Redis Cluster) and databases (DynamoDB, Cassandra).

**Q8. What are virtual nodes in consistent hashing and why do we need them?**
> Multiple virtual points on the ring per physical server, to avoid uneven distribution when there are few real servers — prevents hot spots.

**Q9. Load balancing vs failover — how are they different?**
> Load balancing distributes traffic across *active* servers to optimize performance; failover switches to a *standby* server only when the primary fails, focused on availability, not performance.

**Q10. Can the load balancer itself become a bottleneck or single point of failure?**
> Yes. Mitigated by running multiple LBs (active-active or active-passive) often coordinated via DNS, a virtual IP + heartbeat protocol (e.g., VRRP/keepalived), or a cloud-managed LB service that's inherently redundant.

**Q11. Concurrency vs Parallelism — give a one-liner.**
> Concurrency is about *managing* multiple tasks (can happen on 1 core via context switching); parallelism is about *executing* multiple tasks at the same time (needs multiple cores).

**Q12. Design question you might get: "Design a URL shortener / rate limiter / chat app — where does load balancing fit?"**
> Mention: L7 LB at the edge for smart routing + SSL termination → app servers are stateless (session in Redis) → consistent hashing used if there's a sharded cache/DB layer behind it → health checks + auto-scaling group behind the LB → failover across availability zones/regions for high availability.

---

## 9. One-Page Summary Table (for last-minute revision)

| Concept | Core Idea | Real Example |
|---|---|---|
| Load Balancer | Distributes traffic across servers | NGINX, AWS ELB |
| Hardware/Software/Cloud LB | *Where* it runs | F5 / NGINX / AWS ALB |
| L4 vs L7 | *What* it looks at (IP:port vs HTTP content) | HAProxy TCP mode vs NGINX HTTP |
| GSLB | Routes across data centers/regions | Cloudflare, Route 53 |
| Round Robin | Rotate through servers evenly | Simple apps |
| Weighted RR | Rotate, but bigger servers get more | Mixed-capacity fleet |
| Source IP Hash | Same client → same server | Sessions without shared store |
| Least Connections | Send to least busy server | Variable-length requests |
| Least Response Time | Send to fastest server | Latency-sensitive apps |
| Resource-Based | Send by real CPU/RAM/bandwidth | Cloud/container environments |
| Concurrency | Managing many tasks (can be 1 core) | Node.js event loop |
| Parallelism | Executing many tasks simultaneously | Multi-core image processing |
| Stateless LB | No session memory | REST APIs, CDNs |
| Stateful LB | Sticky sessions | Shopping carts, banking |
| Failover | Switch to backup on failure | Standby DB server |
| Consistent Hashing | Minimal remap on scale change | Redis Cluster, DynamoDB |
| Virtual Nodes | Even out the hash ring | Used inside consistent hashing |

---

*Tip for interviews: always pair a definition with a trade-off ("X is good for ___ but costs ___") — that's what separates a rote answer from a strong systems-thinking answer at SDE-1/2 level.*
