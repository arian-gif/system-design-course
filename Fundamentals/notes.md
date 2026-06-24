# Fundamentals — Notes

> System Design Course · Unit 1: Fundamentals
> Started: 2026-06-23

---

## Table of Contents

1. [The Single-Server Setup](#1-the-single-server-setup)
2. [Databases](#2-databases)
3. [Vertical vs. Horizontal Scaling](#3-vertical-vs-horizontal-scaling)
4. [Single Point of Failure (SPOF)](#4-single-point-of-failure-spof)

---

## 1. The Single-Server Setup

The simplest possible architecture: **one server** handles everything (web app, business logic, and data).

### How a request reaches the server

1. **User enters a URL** in the browser (e.g. `myapp.com`).
2. The browser asks a **DNS provider** to resolve that domain name.
3. The **DNS provider returns the server's IP address** (DNS = the "phonebook" that maps human-readable domain names → machine IP addresses).
4. The browser sends the request directly to that **IP address**.
5. The **server responds with the data** (HTML, JSON, etc.).

### Flow

```
User → enters URL → DNS provider → returns IP address
                                         │
User's browser ──── request to IP ──────▶ Server
User's browser ◀──── response (data) ──── Server
```

**Key idea:** DNS is *not* part of your server — it's a separate, usually third-party service whose only job is the domain → IP translation. Once the browser has the IP, it talks to the server directly.

### Client ↔ Server via API

- The **client sends a request** to the server through an **API**.
- The **server sends back the data**.

```
Client ──── API request ────▶ Server
Client ◀──── data response ──── Server
```

### The limitation

A single server works fine at small scale, but it **falls short when user demand grows**. One machine can only handle so many concurrent requests / so much load before it becomes a bottleneck.

> This limitation is what motivates the rest of system design — scaling beyond a single server (load balancing, multiple servers, separating the database, caching, etc.).

---

## 2. Databases

In the single-server setup, that one machine holds three things:

- **Backend code** (your application logic)
- **Cache** (fast, temporary storage)
- **Database (DB)** (persistent storage)

As you scale, the DB is often **moved onto its own server** and accessed over the network — and it can be exposed/accessed through an **API layer** rather than the app talking to it directly.

### Two broad types of database

| | **Relational (SQL)** | **Non-Relational (NoSQL)** |
|---|---|---|
| Structure | **Structured** — fixed schema | **Flexible** — schema can vary |
| Model | Tables (rows + columns), like a spreadsheet | Documents, columns, key-value, graphs |
| Query language | SQL | Varies by type |
| Best when | Relationships & consistency matter | Flexibility & scale matter |

---

### Relational Databases (SQL)

Data lives in **tables** of rows and columns — conceptually similar to an Excel sheet. Examples: PostgreSQL, MySQL.

**Advantages:**
- Support **complex JOIN operations** (combining related tables).
- Strong **data consistency and integrity**.

**Transactions & ACID** — a transaction is a unit of work that may span multiple steps. SQL transactions follow the **ACID** guarantees:

- **A — Atomicity:** the whole transaction succeeds, or it completely fails (no half-done state).
- **C — Consistency:** the database always moves from one valid state to another.
- **I — Isolation:** concurrent transactions don't interfere with each other.
- **D — Durability:** once committed, data survives crashes/power loss.

---

### Non-Relational Databases (NoSQL)

Prioritize **flexibility** and scale. Main sub-types:

1. **Document stores** — store data as documents (typically JSON-like). e.g. **MongoDB**.
2. **Wide-column stores** — data organized by rows with *dynamic* columns (columns can differ per row). e.g. **Cassandra**, **HBase**, **Bigtable**, Cosmos DB (wide-column mode). See deep-dive below.
3. **Key-value stores** — simple `key → value` lookups, very fast. e.g. **Redis** (in-memory), **DynamoDB** (disk-backed). See deep-dive below.
4. **Graph databases** — store data as **nodes** (entities) and **edges** (relationships between them). The relationships are first-class data, not something you reconstruct with JOINs. e.g. **Neo4j**. See deep-dive below.

---

#### Deep-dive: Wide-column stores & dynamic columns

Each row has a **row key** (unique ID) and a set of **columns** — but unlike a relational table, **each row can have a different set of columns**. Columns aren't fixed by a schema; they're created per-row at write time. That's what **"dynamic columns"** means.

- **Relational:** schema is fixed — every row has the *same* columns (empty ones stored as `NULL`).
- **Wide-column:** each row stores *only* the columns it actually has — different rows can have totally different columns.

```
Row "user:101" → name=Alice  email=a@x.com  city=NYC
Row "user:102" → name=Bob    phone=555-1234
Row "user:103" → name=Carol  email=c@x.com  age=30  premium=true
```

**Why use them:**
- Built for **massive scale** and **write-heavy** workloads (spread across many machines).
- Great for **sparse data** — many possible fields, but each record only uses a few, with no wasted `NULL`s and no schema migration to add a field.

**Trade-off:** weak at JOINs / complex queries — you model the data around the queries you'll run.

> Mental model: a wide-column row is like a **flexible hash map per row** (`column name → value`), not a rigid spreadsheet row.

---

#### Deep-dive: Graph databases

Store data as a **graph**: **nodes** (the things/entities) connected by **edges** (the relationships between them). Both nodes and edges can carry properties.

```
(Alice) ──friends_with──▶ (Bob)
   │                        │
 likes                   works_at
   ▼                        ▼
(Pizza)                 (Acme Inc)
```

The key idea: **relationships are stored directly** as part of the data. In a relational DB you'd reconstruct "who is friends with whom" using JOINs across tables; in a graph DB you just **traverse the edges**, which is far faster for deeply connected data.

**Good for:**
- Social networks (friends-of-friends), recommendation engines, fraud detection, knowledge graphs.
- Any query that's really about **connections** ("shortest path", "how are these two related").

Example: **Neo4j**.

---

#### Deep-dive: Key-value stores (and how they differ from MongoDB)

A **key-value store** maps a **key → value** and is the simplest, fastest DB type. e.g. **Redis**, **DynamoDB**, **Memcached**.

- Only two real operations: **`get(key)`** and **`put(key, value)`**.
- The value is **opaque** — the DB treats it as a blob and can't look inside it.
- **In-memory** ones (like **Redis**) live in RAM → extremely fast, commonly used as a **cache**. (Not all key-value stores are in-RAM — **DynamoDB** is disk-backed.)

**Why isn't MongoDB a key-value store?** Mongo is a **document store**. Both use keys, but the difference is the *value*:

| | **Key-Value** (Redis) | **Document store** (MongoDB) |
|---|---|---|
| The value is... | an **opaque blob** | a **structured document** (JSON/BSON) |
| DB understands the value? | ❌ no | ✅ yes |
| Query *by contents* of the value? | ❌ only by key | ✅ by key **or** any field inside |
| Example query | `get("user:101")` | "all users where `city = NYC` and `age > 30`" |

> Key point: **every document store is "key → something," but a key-value store keeps the value dumb/opaque.** Mongo's advantage is that it can *see into* and query the value; a key-value store cannot.

---

### Advantages of NoSQL

- **Data can live together in one place** — because there's no rigid schema, you can store related data **denormalized** (all in one document/record) instead of split across many tables. This avoids expensive JOINs.
- **Low latency** — simpler data model + denormalized reads = fast lookups.
- **Handles big data / scales out** — designed to **scale horizontally** (spread across many machines), so it copes with very large volumes and high traffic.

> Trade-off reminder: you gain flexibility, speed, and scale, but generally **give up JOINs and strong ACID guarantees** (NoSQL often favors *eventual* consistency). You design the data around the queries you'll run.

---

## 3. Vertical vs. Horizontal Scaling

Two ways to handle more load than a single server can take.

### Vertical scaling ("scale up")

Make the **single server more powerful** — add more CPU, RAM, faster disks. Same one machine, just beefier.

- ✅ Simple — no code changes, no extra coordination.
- ❌ **Hard ceiling** — there's a physical/cost limit to how good one machine can get.
- ❌ **Single point of failure** — if that one server goes down, **everything goes down**.

### Horizontal scaling ("scale out")

Run **multiple servers** that share the load.

- ✅ **Far better for large scale** — add more machines as demand grows (effectively no ceiling).
- ✅ **Resilient** — if one server goes down, the others keep serving traffic (no single point of failure).
- ❌ More complex — you now need something to distribute requests across the servers.

### How to implement horizontal scaling: the Load Balancer

A **load balancer** sits in front of the servers and **decides which server each incoming request goes to**, spreading traffic across them.

```
                 ┌──────────▶ Server 1
Clients ──▶ Load │──────────▶ Server 2
            Balancer─────────▶ Server 3
```

- Distributes traffic so no single server is overwhelmed.
- If a server goes down, the load balancer **stops routing traffic to it** and uses the healthy ones.

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| How | bigger single server | more servers |
| Ceiling | limited by one machine | scales much further |
| Failure | single point of failure | survives a server going down |
| Complexity | simple | needs a load balancer |

### Load Balancing Strategies / Algorithms

How the load balancer decides *which* server gets each request. (Course covers **7** — listed so far below.)

1. **Round Robin** — hand requests out in order, cycling through servers: request 1 → server 1, request 2 → server 2, request 3 → server 3, then back to server 1. Simple and even *if* all requests cost about the same.

2. **Least Connections** — send the request to the server with the **fewest active connections**. Better when requests take **variable amounts of time** (not fixed per user), so a server stuck on slow requests doesn't keep getting more.

3. **Least Response Time** — route to the server with the **lowest (fastest) response time**, i.e. the most responsive/least-loaded server gets the next request. Factors in actual server performance, not just connection count.

4. **IP Hash** — hash the client's IP address to pick a server, so **all requests from one IP always go to the same server**. Useful for **session stickiness** (keeping a user tied to one server).

5. **Weighted** — assign **weights** to servers (e.g. a more powerful server gets a higher weight) and combine with Round Robin or Least Connections. Higher-weighted servers receive a proportionally larger share of traffic.

6. **Geographic (Geo-based)** — route the request to the server **closest to the client geographically** (e.g. US users → US server). Lowers latency and can help with data-residency rules.

7. **Consistent Hashing** — place servers around a **hash ring (circle)**. A hash function maps each request onto a point on the ring, and it's served by the **next server clockwise** on the circle. Its real value: when a server is added/removed, only a **small slice of keys** get remapped (instead of reshuffling everything), so it's heavily used for **caching and sharding**.

> Quick guide: **Round Robin** = equal & simple · **Least Connections / Least Response Time** = adapt to load · **IP Hash** = stickiness · **Weighted** = uneven server sizes · **Geographic** = lowest latency by location · **Consistent Hashing** = stable mapping when servers change (caches/shards).

### Which is used most?

- **Round Robin** (and **Weighted Round Robin**) is the most common **default** — simple and works well when servers are similar.
- **Least Connections / Least Response Time** are widely used when request durations vary a lot.
- **Consistent Hashing** dominates for **caches and sharded data stores** (not general web traffic).
- **Geographic** is standard for global/CDN-style traffic.

> There's no single "best" — production systems pick per use case (often **Weighted Round Robin** or **Least Connections** for app servers, **Consistent Hashing** for caches).

### Health Checks

The load balancer regularly **pings each server** to confirm it's alive and responding (a *health check*). Based on the result:

- **Healthy** server → keeps receiving traffic.
- **Unhealthy** server (failed health check) → the load balancer **stops routing requests to it** until it passes again.

This is what makes horizontal scaling **fault-tolerant** — a dead or struggling server is automatically taken out of rotation, so users aren't sent to a broken machine.

### Hardware vs. Software Load Balancers

Load balancers come in two forms:

- **Hardware load balancer** — a **dedicated physical appliance** built specifically for load balancing (e.g. F5, Citrix NetScaler). Very high performance and reliability, but **expensive** and **less flexible** (you're tied to the device/vendor and physical capacity).
- **Software load balancer** — load balancing run as **software** on standard/commodity servers or in the cloud (e.g. **NGINX**, **HAProxy**, AWS ELB). **Cheaper, flexible, and easy to scale** — the common choice for most modern systems.

| | Hardware | Software |
|---|---|---|
| Form | dedicated physical appliance | runs on normal servers / cloud |
| Cost | high | low |
| Flexibility / scaling | limited, vendor-bound | flexible, easy to scale |
| Performance | very high | high (good enough for most) |

---

## 4. Single Point of Failure (SPOF)

A **SPOF** is any single component that, if it fails, brings **the entire system down**. Examples:

- The **database** can be a SPOF.
- The **load balancer** itself can be a SPOF (ironic — the thing protecting your servers can become the weak link).

**Why SPOFs are bad — they hurt on three fronts:**
- **Availability** — one failure takes everything down.
- **Scalability** — that single component becomes the bottleneck for the whole system.
- **Security** — a single chokepoint is a single, high-value target to attack.

### How to eliminate a SPOF (load balancer example)

1. **Redundancy** — run **more than one load balancer** instead of relying on a single one.
2. **Health checks & monitoring** — continuously monitor the health of the load balancers themselves.
3. **Self-healing / failover** — if the active load balancer goes down, the system **automatically fails over to the backup** one.

> **DB as a SPOF** is handled separately — covered later (replication, etc.) in another section.
