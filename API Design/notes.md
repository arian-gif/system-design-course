# API Design — Notes

> System Design Course · Unit 2: API Design
> Started: 2026-06-23

---

## Table of Contents

1. [What is an API?](#1-what-is-an-api)
2. [API Styles: REST, GraphQL, gRPC](#2-api-styles-rest-graphql-grpc)
3. [Key API Design Principles](#3-key-api-design-principles)
4. [API Protocols — When to Use Each](#4-api-protocols--when-to-use-each)
5. [The API Design Process](#5-the-api-design-process)
6. [The API Lifecycle](#6-the-api-lifecycle)
7. [Application Layer Protocols (Deep-dive)](#7-application-layer-protocols-deep-dive)
8. [Transport Layer: TCP vs UDP](#8-transport-layer-tcp-vs-udp)
9. [Query Parameters: Filtering, Sorting, Pagination](#9-query-parameters-filtering-sorting-pagination)
10. [Designing GraphQL APIs](#10-designing-graphql-apis)
11. [Authentication & Authorization](#11-authentication--authorization)

---

## 1. What is an API?

**API = Application Programming Interface** — the way **two pieces of software communicate with each other**.

In our setup, the **client and server communicate via an API**. The API defines the "contract" between them:

- **What requests** can be sent
- **How to make** those requests (format, parameters, etc.)
- **What response** to expect back

```
Client ──── request (per API contract) ────▶ Server
Client ◀──── response (per API contract) ──── Server
```

### Key properties

- **Abstraction** — the API exposes **what** functionality is available but **hides the implementation** (*how* it works internally). The caller doesn't need to know how the server does its job.
- **Service boundaries** — APIs **define clear interfaces between system components**, drawing the line where one service ends and another begins.

> Mental model: an API is a **contract + menu**. It tells you what you can order and how to order it, without showing you what happens in the kitchen.

---

## 2. API Styles: REST, GraphQL, gRPC

Three common styles for building APIs, each suited to different needs.

### REST

- Uses **HTTP** as its protocol, with **standardized HTTP methods**: `GET` (read), `POST` (create), `PUT` (replace), `PATCH` (partial update), `DELETE`, etc.
- Simple, universal, cacheable — the default for most **web and mobile apps**.

### GraphQL

- A **query language** for APIs: the **client asks for exactly the data it needs** — no more, no less.
- Three operation types:
  - **Query** — read data
  - **Mutation** — write/change data
  - **Subscription** — real-time updates (server pushes changes)
- **Minimizes round trips**: where REST might need **3 requests** to gather data from several endpoints, GraphQL can fetch it all in **1**.
- Great for **complex UIs** that pull together lots of related data.

### gRPC

- **High-performance** RPC framework that uses **Protocol Buffers (protobuf)** instead of JSON — a compact **binary** format that's faster and smaller.
- Supports **streaming**, including **bidirectional streaming** (both sides send a continuous stream of messages).
- Best for **high-performance, internal service-to-service** communication → very common between **microservices**.

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol/format | HTTP + JSON | HTTP + query language | HTTP/2 + Protocol Buffers (binary) |
| Strength | simple, universal, cacheable | fetch exactly what you need, fewer round trips | speed + streaming |
| Best for | web & mobile apps | complex UIs | microservices / internal high-performance |

---

#### Deep-dive: Microservices, and what gRPC actually does

**Monolith vs. microservices**

- **Monolith** — all features (users, payments, search, notifications) live in **one big codebase/program**. One deploy, scaled as a single unit. (This is the single-server app from Fundamentals.)
- **Microservices** — that one app is split into **many small, independent services**, each owning **one job**, with its own codebase (often its own DB), deployed and **scaled independently**. They **talk to each other over the network via APIs**.

```
     MONOLITH                      MICROSERVICES
 ┌──────────────┐         ┌───────┐ ┌────────┐ ┌────────┐
 │ users        │         │ Users │ │Payments│ │ Search │
 │ payments     │   →     └───────┘ └────────┘ └────────┘
 │ search       │         ┌──────────────┐
 │ notifications│         │ Notifications│
 └──────────────┘         └──────────────┘
```

Because microservices constantly call each other over the network, they generate **huge volumes of internal API calls** — which is the problem gRPC is built for.

**gRPC = Remote Procedure Call (RPC)**

Lets one service **call a function that lives on another service, over the network, as if it were a local function**:

```
user = userService.GetUser(id=42)   // looks like a normal function call,
                                     // but userService runs on another machine
```

Each "call" = a method defined in the API's schema (e.g. `GetUser`, `CreatePayment`). You invoke it like a function; gRPC handles sending the request across the network and returning the response. The available calls and their input/output shapes are defined up front in a **`.proto` schema** that both sides agree on.

Two things make gRPC fast and ideal for microservices:

1. **Protocol Buffers (binary), not JSON.** JSON is text — readable but bulky and slow to parse (`{"id":42,"name":"Alice"}`). Protobuf encodes the same data as compact **binary** → smaller and faster. Trade-off: not human-readable, and both sides must share the schema.
2. **Streaming, incl. bidirectional.** REST is one request → one response. gRPC can hold a **continuous stream** open, with **both sides** sending messages over one connection (live feeds, chat, telemetry).

**Why not use gRPC everywhere?** Browsers/phones can't easily speak it and it's not human-readable, so it's poor for public clients.

> Mental model: **REST/GraphQL = how the outside world (browsers, phones) talks to your system. gRPC = how your system's internal services talk to each other.**

---

## 3. Key API Design Principles

Four principles of a good API:

1. **Consistency**
2. **Simplicity**
3. **Security**
4. **Performance**

> **Guiding rule:** the best API is one developers can use **without reading the documentation** — it behaves the way they'd guess.

### 1. Consistency

- **Consistent naming** across endpoints (e.g. always `userId`, not `user_id` in one place and `uid` in another).
- **Consistent patterns** — similar things work in similar ways, so once you learn one endpoint you can predict the rest.

### 2. Simplicity

- **Focus on the use case** — design around what developers actually need to do.
- **Intuitive design** — obvious enough to guess.

### 3. Security

- **Authentication & authorization** — *authentication* = who you are; *authorization* = what you're allowed to do.
- **Input validation** — never trust client input; validate it to block bad/malicious data.
- **Rate limiting** — cap how many requests a client can make in a time window (prevents abuse/overload).

### 4. Performance

- **Caching** — serve repeated requests from a fast store instead of recomputing/refetching.
- **Pagination** — return large result sets in **small chunks ("pages")** instead of all at once (see note below).
- **Minimize payloads** — send back **as little data as needed**, nothing extra.
- **Reduce round trips** — let clients get what they need in **fewer requests**.
- **Don't duplicate endpoints** — avoid creating two different endpoints that return the same data.

> **What is pagination?** Imagine an endpoint that could return 1,000,000 results. Sending them all in one response is slow and heavy. Instead you return them in **pages** — e.g. 20 at a time — and the client requests the next page as needed (`?page=2` or `?limit=20&offset=40`). Faster responses, less memory, less bandwidth. (We'll see this again in feeds/lists.)

---

## 4. API Protocols — When to Use Each

Which underlying protocol fits which job:

- **HTTP** → **RESTful APIs.** Standard request/response. Uses **HTTP status codes** (200, 404, 500, etc.) and maps cleanly to **CRUD** operations (Create/Read/Update/Delete via POST/GET/PUT-PATCH/DELETE). The default for most web/mobile APIs.
- **WebSockets** → **real-time APIs.** Keeps a **persistent, two-way connection** open so the server can push data instantly (chat, live notifications, multiplayer, live feeds) — instead of the client repeatedly polling.
- **GraphQL** → runs **over HTTP**, used when clients need to **fetch exactly the data they want** in one request (complex UIs).
- **gRPC** → best for **microservices** (internal, high-performance service-to-service calls).

| Protocol | Use it for | Key trait |
|---|---|---|
| **HTTP / REST** | web & mobile CRUD APIs | status codes, request/response |
| **WebSockets** | real-time apps (chat, live updates) | persistent two-way connection |
| **GraphQL (over HTTP)** | complex UIs needing precise data | client queries exactly what it needs |
| **gRPC** | microservice-to-microservice | high performance, binary (protobuf) |

> **CRUD ↔ HTTP methods:** Create → `POST`, Read → `GET`, Update → `PUT`/`PATCH`, Delete → `DELETE`.

> **Why WebSockets for real-time?** Plain HTTP is request/response — the client must *ask* before it gets anything. For live data that won't work (you'd have to poll constantly). A WebSocket stays open so the **server can push** the moment something changes.

---

## 5. The API Design Process

**Steps to design an API:**

1. **Identify use cases & user stories** — what does the API need to let people do?
2. **Define scope & boundaries** — build *only* for the features you actually want to support (don't over-build).
3. **Determine performance requirements** — expected load, latency, throughput.
4. **Identify security constraints** — auth, data sensitivity, compliance.

**Three approaches to designing it:**

| Approach | Start from… | Idea |
|---|---|---|
| **Top-down** | high-level requirements & workflows | start from what users need, then define the endpoints |
| **Bottom-up** | existing data models & capabilities | start from the data/systems you already have |
| **Contract-first** | the API contract itself | define the full API contract **before** writing any implementation |

> **Contract-first** is popular for teams: agree on the contract up front so frontend and backend can build in parallel against the same agreed interface.

---

## 6. The API Lifecycle

The stages an API goes through from birth to retirement:

1. **Design** — define requirements, the contract, and expected outcomes.
2. **Development** — build the API and test it locally.
3. **Deployment & Monitoring** — ship it and watch it in production (track health, errors, usage).
4. **Maintenance** — keep it working and make it **easy for future developers to maintain**.
5. **Deprecation & Retirement** — when a newer/better API replaces it, the old one is **deprecated** (marked for removal, users warned) and eventually **retired** (shut off).

```
Design → Development → Deployment/Monitoring → Maintenance → Deprecation → Retirement
```

> **Deprecation vs retirement:** *deprecated* = still works but discouraged (with a sunset date and migration path); *retired* = actually removed. Good APIs give consumers plenty of warning before retirement so they can migrate.

---

## 7. Application Layer Protocols (Deep-dive)

When choosing a protocol, it must meet your **latency** and **throughput** requirements.

Common **application-layer** protocols:

- **HTTP / HTTPS**
- **WebSockets**
- **AMQP** — Advanced Message Queuing Protocol
- **gRPC** — Google Remote Procedure Call

### HTTP / HTTPS

A REST API request travels over HTTP and includes:
- a **host** and **headers** (e.g. an `Authorization` header for credentials),
- and the response comes back with **status codes** plus useful headers like **`Cache-Control`**.

**HTTPS = HTTP + TLS/SSL** (an encryption layer on top of HTTP).

**Benefits of HTTPS:**
- **Data encryption** — traffic is unreadable to eavesdroppers.
- **Data integrity** — data can't be altered in transit undetected.
- **Authentication** — confirms you're talking to the real server.

**Risks of *not* using HTTPS:**
- **Man-in-the-middle (MITM) attacks** — attacker secretly intercepts the connection.
- **Data tampering** — data modified in transit.
- **Information theft** — credentials/personal data stolen.
- **Loss of user trust.**

### WebSockets

1. Client and server perform a **handshake** to open the connection.
2. The connection stays open: client and server can **send messages freely**, and the **server can push** data to the client.

**Why not just use HTTP for this?** For real-time, HTTP means constant polling, which causes:
- too much **latency**,
- **wasted bandwidth** (repeated requests that mostly return "nothing new"),
- **server resource overhead** (handling all those polling requests).

### AMQP (Advanced Message Queuing Protocol)

A **messaging** protocol built around a **message broker** and **queues**:

1. A **producer** sends a message to the **message broker**.
2. The broker places it in a **queue**.
3. When a **consumer** is free, it **pulls** the message from the queue.

```
Producer ──▶ Message Broker ──▶ [ Queue ] ──▶ Consumer (pulls when ready)
```

This **decouples** producer and consumer — they don't have to be available at the same time, and work can buffer in the queue during spikes. (Connects to message queues in the Big Data unit.)

### gRPC

Uses **Protocol Buffers** (compact binary format) for high-performance, service-to-service communication. (See the microservices deep-dive in §2.)

| Protocol | Style | Best for |
|---|---|---|
| **HTTP/HTTPS** | request/response | standard REST web/mobile APIs |
| **WebSockets** | persistent, bidirectional | real-time push (chat, live feeds) |
| **AMQP** | async messaging via broker/queue | decoupled background work, buffering spikes |
| **gRPC** | binary RPC | internal microservice calls |

---

## 8. Transport Layer: TCP vs UDP

The **transport layer** handles **how data actually moves from one computer to another**. The application protocols from §7 (HTTP, WebSockets, etc.) run *on top of* these.

Two main protocols, and they're a classic trade-off:

- **TCP** — **reliable but slower**
- **UDP** — **fast but unreliable**

### TCP — Transmission Control Protocol

Built for **reliability**:

- **Guaranteed delivery** — every packet is accounted for.
- **Connection-based** — establishes a connection first via a **3-way handshake** before any data is sent.
- **Ordered packets** — data arrives in the same order it was sent.
- **Error checking** — detects corrupted/missing data.
- **Retransmission** — data is sent in **chunks (packets)**; if a chunk is lost or missing, **TCP resends it**.

The cost of all this bookkeeping is **more overhead → slower**.

#### The 3-Way Handshake

How TCP opens a connection before sending data:

1. **SYN** — the **client** sends a *synchronize* request to the server ("let's connect").
2. **SYN-ACK** — the **server** *acknowledges* and sends back its own *synchronize* ("got it, let's connect").
3. **ACK** — the **client** *acknowledges* the server's reply → **connection established**.

```
Client                         Server
  │ ──────── SYN ───────────▶  │   (1) client: let's sync
  │ ◀────── SYN-ACK ────────── │   (2) server: ack + sync
  │ ──────── ACK ───────────▶  │   (3) client: ack → connected
  │ ===== connection open ==== │
```

> "SYN" = synchronize, "ACK" = acknowledge. Both sides confirm they can send *and* receive before any real data flows — that mutual confirmation is what makes TCP reliable (and is the overhead UDP skips).

### UDP — User Datagram Protocol

Built for **speed**:

- **Not guaranteed** — packets may be lost and that's accepted.
- **No connection, no tracking** — it just fires data off ("fire and forget").
- **Faster, less overhead** — none of TCP's handshake/ordering/retransmission work.

### TCP vs UDP

| | **TCP** | **UDP** |
|---|---|---|
| Reliability | guaranteed delivery | no guarantee (packets can drop) |
| Connection | yes (3-way handshake) | none |
| Ordering | packets arrive in order | no ordering |
| Error handling | checks + resends lost packets | none |
| Speed | slower (more overhead) | faster (less overhead) |
| Use when… | data **must** be correct & complete | speed matters more than perfection |

> **Rule of thumb:** use **TCP** when every byte matters — web pages (HTTP/HTTPS), APIs, file transfers, email. Use **UDP** when speed beats perfection and a few lost packets are OK — **live video/voice calls, game streaming, live broadcasts** (a dropped frame is better than a laggy one).

---

## 9. Query Parameters: Filtering, Sorting, Pagination

APIs use **query parameters** (the `?key=value` part of a URL) to let clients shape the data they get back, instead of always returning everything.

- **Filtering** — return only records matching a condition.
  `GET /products?category=shoes&color=black`
- **Sorting** — order the results by a field.
  `GET /products?sort=price` (or `sort=-price` for descending)
- **Pagination** — return results in small chunks ("pages") instead of all at once, using a **limit** (page size) and a way to pick the page.
  `GET /products?limit=20&page=2` (or `?limit=20&offset=40`)

```
GET /products?category=shoes&sort=price&limit=20&page=2
              └── filter ──┘ └─ sort ─┘ └─ pagination ─┘
```

These often **combine in one request** (filter → sort → paginate).

> **Why this matters (ties to §3 performance):** filtering + pagination keep payloads small and responses fast — you don't send 1,000,000 rows when the client needs 20. `limit` caps how much comes back per request; the page/offset picks *which* slice.

---

## 10. Designing GraphQL APIs

(Builds on the GraphQL intro in §2.)

**The problem GraphQL solves:** the client gets **exactly what it wants in one request** — no over-fetching (extra fields it doesn't need) and no under-fetching (having to make several REST calls to gather everything).

### Define a schema

GraphQL is **schema-first**: you declare your data **types** and the operations available on them. The schema is the contract.

```graphql
# A data type
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!     # can pull related data in the same query
}

# Read operations
type Query {
  user(id: ID!): User        # fetch one specific user + whatever fields you ask for
  users: [User!]!            # fetch all users
}

# Write operations
type Mutation {
  createUser(name: String!, email: String!): User
}
```

- **`type User { ... }`** — defines the shape of the data (its fields).
- **`type Query`** — the entry points for **reading** data. The client picks *which fields* it wants back.
- **`type Mutation`** — the entry points for **writing/changing** data (add, update, delete).
- (**`type Subscription`** — real-time updates, from §2.)

### Client asks for exactly what it needs

```graphql
query {
  user(id: "42") {
    name
    email          # ← only these fields come back, nothing else
  }
}
```

> Contrast with REST: REST gives you a **fixed** response shape per endpoint (often too much or too little), and related data may need multiple endpoints. GraphQL lets the **client** decide the exact fields and pull related data (like a user's posts) in **one** request.

> ⚠️ Some overlap with §2 — the new piece here is the **schema definition** (`type User`, `Query`, `Mutation`) as the design artifact.

### Error handling & safety in GraphQL

- **Return errors clearly** — send back an **error object (JSON)** that includes a **status code** so the client knows what went wrong (rather than failing silently).
- **Avoid deeply nested queries** — because clients control the shape, they *could* request data nested many levels deep (e.g. user → posts → comments → author → posts → …), which can be very expensive.
- **Implement query depth limits** — cap how deep a query is allowed to go, so a single malicious/accidental query can't overload the server.

> **Why this is a GraphQL-specific concern:** in REST the server controls the response shape, so cost is bounded. In GraphQL the **client** controls it — that flexibility is the strength *and* the risk. Depth limits (and complexity limits / rate limiting) are how you keep that power from becoming a denial-of-service hole.

---

## 11. Authentication & Authorization

### Authentication (AuthN) — *who are you?*

Before the system lets a user access the server, it first needs to **verify the user's identity**. Authentication is the concept of confirming that a user (whether via a **mobile app**, a **third-party app**, etc.) is **who they claim to be**.

- It's the **first gate**: prove your identity before you get in.
- Applies to third-party/clients too — making sure they are genuinely who they say they are.

> Recall from §3 (security): **authentication = who you are**, **authorization = what you're allowed to do**. Authentication always comes first.

> ⚠️ **Authentication methods ≠ authorization frameworks.** This section lists *authentication* methods (how you prove identity). Authorization frameworks (like OAuth — controlling *what* you can access) are a separate thing covered next.

### Authentication methods

**1. Basic Authentication**
- Sends **Base64-encoded `username:password`** with the request; server checks if the credentials are valid or invalid.
- ❌ **Weak** — Base64 is **encoding, not encryption**, so it's fully **reversible** (anyone can decode it back to the password).
- Only somewhat safe over **HTTPS** (encrypts the transport), but still considered a **bad** approach.

**2. Digest Authentication**
- Sends a **hashed** form of the username/password instead of plain Base64.
- Still has weaknesses and is **rarely used** today.

**3. API Key Authentication**
- The client is issued an **API key** that grants access to resources; it sends the key with each request.
- ❌ Risk: if the **API key leaks**, you're cooked — whoever has it can act as you.
- ❌ The key info is **stored in a DB**, so anyone with DB access could look it up.

**4. Traditional Web (Session-based) Authentication**
- User logs in with credentials → the server **creates a session** for them.
- The session is typically stored in **Redis** (fast, with **built-in expiration**).
- This is **stateful** — the server must **store the session state** (who's logged in).
- ✅ Good for **web apps**, but ❌ **doesn't scale well for distributed systems or APIs** (see why below).

> **Pattern:** the first three send credentials/keys on *every* request; the session approach logs in *once* and then tracks state server-side. The scaling problem with sessions is what pushes APIs toward **stateless tokens** (e.g. JWT) — likely covered next.

### Token-Based Authentication (Bearer & JWT)

Solves the session/scaling problem by making auth **stateless**.

**The flow:**
1. Client sends its **credentials** to an **auth server**.
2. The auth server verifies them and returns a **JWT token**.
3. The client then sends that **token with every request** to the API (no need to re-send the password).

```
Client ──credentials──▶ Auth Server ──issues──▶ JWT
Client ──request + JWT (Bearer token)──▶ API  ──▶ verifies token, allows/denies
```

**Bearer token** — "whoever **bears** (holds) the token has access." It's sent in the header:
`Authorization: Bearer <token>`. (So if someone steals it, they get in — hence HTTPS + expiry matter.)

**JWT (JSON Web Token)** — a token that **is** a JSON object containing the user's info (claims), and it's **cryptographically signed**. Three parts:
- **Header** — token type + signing algorithm
- **Payload** — the claims (user id, roles, expiry, etc.)
- **Signature** — signed with the server's secret key; proves the token wasn't tampered with

**Why JWT is stateless (no DB needed):** the user's info lives **inside the token**, and the signature lets the server **verify it independently** — so the server doesn't store anything or look anyone up.

> **The JWT trade-off:** because nothing is stored server-side, a JWT **can't easily be revoked** before it expires. (Some systems add a Redis **token blocklist** to fix this.) One-liner: *JWT trades revocability for statelessness.*

#### Access Tokens vs Refresh Tokens

To limit the damage if a token is stolen, JWT setups use **two** tokens:

- **Access token** — **short-lived** (e.g. minutes). Sent with each API request to prove access. If stolen, it expires fast.
- **Refresh token** — **long-lived** (e.g. days/weeks). Its *only* job is to **request a new access token** when the old one expires — without making the user log in again.

**Storage:** keep the refresh token in an **`HttpOnly` cookie** (JavaScript can't read it → protects against XSS theft).

```
Access token expires
      │
Client ──refresh token──▶ Auth Server ──▶ issues a new access token
      │
Client keeps calling the API with the new access token
```

> **Why two tokens?** Short-lived access tokens keep the blast radius small (a leaked one dies quickly), while the long-lived refresh token saves the user from logging in every few minutes. You get **security + good UX** at once.
