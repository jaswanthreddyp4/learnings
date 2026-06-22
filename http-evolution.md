# HTTP: From 1.0 to 1.1

---

## Table of Contents

1. [HTTP Fundamentals](#http-fundamentals)
2. [HTTP Headers](#http-headers)
3. [HTTP Methods](#http-methods)
4. [Idempotency](#idempotency)
5. [CORS](#cors)
6. [Status Codes](#status-codes)
7. [HTTP Caching](#http-caching)
8. [Content Negotiation](#content-negotiation)
9. [HTTP 1.0 — How It Worked](#http-10--how-it-worked)
10. [Problems with HTTP 1.0](#problems-with-http-10)
11. [HTTP 1.1 — The Evolution](#http-11--the-evolution)

---

## HTTP Fundamentals

HTTP (HyperText Transfer Protocol) is a **simple, lightweight, stateless** protocol designed for communication between a client and a server. It gained massive traction as the web evolved.

HTTP is built on top of **TCP** — TCP operates at the transport layer, establishes connections, and handles reliability. HTTP operates at the **application layer** on top of it.

---

## HTTP Headers

Headers carry metadata about the request/response. They are kept **separate from the request body** so that network components can read routing and control information without having to unpack the full payload.

### Request Headers

| Header | Purpose |
|---|---|
| `User-Agent` | Info about the client (browser, Postman, mobile app) |
| `Authorization` | Bearer tokens sent from client to server to authenticate |
| `Cookie` | Session cookies sent with the request |
| `Accept` | Content type the client expects (e.g. `application/json`, `text/xml`) |

### General Headers

Present in both requests and responses:

| Header | Purpose |
|---|---|
| `Date` | Timestamp of the message |
| `Cache-Control` | Caching directives (e.g. `max-age=60`, `no-cache`) |
| `Connection` | Connection status (`keep-alive` / `close`) |

### Representation Headers

| Header | Purpose |
|---|---|
| `Content-Type` | Format of the body (e.g. `application/json`, `text/html`) |
| `Content-Length` | Size of the response body in bytes |
| `Content-Encoding` | Encoding applied to the body (e.g. `gzip`) |
| `ETag` | Tag value used for caching/validation |

### Security Headers

| Header | Purpose |
|---|---|
| `HSTS` (Strict-Transport-Security) | Forces HTTPS — content must be transferred over a secured connection |
| `CSP` (Content-Security-Policy) | Prevents cross-site information leaks from JS/CSS |

---

## HTTP Methods

HTTP methods define the **action** the client is requesting the server to perform.

| Method | Purpose | Has Body? |
|---|---|---|
| `GET` | Fetch data from the server. Must not modify server data. | Optional |
| `POST` | Create / add new data on the server. | Yes |
| `PATCH` | Partially update existing data (selective replacement). | Yes |
| `PUT` | Fully replace an existing resource with the request body. | Yes |
| `DELETE` | Remove data from the server. | Optional |

> **Rule of thumb:** Use `PATCH` by default unless you need to replace an entire resource, in which case use `PUT`.

---

## Idempotency

An operation is **idempotent** if calling it multiple times produces the same result as calling it once.

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | ✅ Yes | Reading the same resource repeatedly returns the same data |
| `PUT` | ✅ Yes | Replacing a resource with the same data has no additional effect |
| `DELETE` | ✅ Yes | Deleting an already-deleted resource is a no-op |
| `POST` | ❌ No | Each call creates a new resource |
| `PATCH` | ❌ No | Appending/incrementing data produces different results on each call |

---

## CORS

**Cross-Origin Resource Sharing (CORS)** is a browser security mechanism that controls whether a web page can make requests to a different domain than the one that served it.

### 1. Simple Request

A basic cross-origin GET request:

```http
GET /api/products/123 HTTP/1.1
Host: api.anotherdomain.com
Origin: https://example.com
Accept: application/json
```

**Response (allowed):**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Access-Control-Allow-Origin: https://example.com

{ "id": 123, "name": "Example Product", "price": 29.99 }
```

The browser checks the `Access-Control-Allow-Origin` header. If it matches the requesting origin (or is `*`), the browser lets the JavaScript code read the response. If it's missing, the browser **blocks** the response — even though the server returned 200.

---

### 2. Pre-flight Request

A **pre-flight** request is automatically triggered by the browser before the actual request when:

1. The method is `DELETE` or `PUT`
2. The request includes non-simple headers (e.g. `Authorization`, `X-Custom-Header`)

The pre-flight uses the `OPTIONS` method to ask the server: *"Will you accept my actual request?"*

**Pre-flight request:**

```http
OPTIONS /api/resource HTTP/1.1
Host: api.anotherdomain.com
Origin: https://example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization
```

**Pre-flight response:**

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: PUT, DELETE
Access-Control-Allow-Headers: Authorization
Access-Control-Max-Age: 86400
```

`204 No Content` means the server responded successfully but has no body to return. Once the browser sees the allowed origin and method, it proceeds with the actual request.

---

## Status Codes

Status codes are 3-digit numbers that tell the client the outcome of a request — **without needing to parse the response body**.

### 1xx — Informational

| Code | Meaning |
|---|---|
| `100 Continue` | For large requests: client sends headers first, server responds with 100 before client sends the body |

### 2xx — Success

| Code | Meaning |
|---|---|
| `200 OK` | Request succeeded |
| `201 Created` | Request succeeded and a new resource was created (typically `POST`/`PUT`) |
| `204 No Content` | Request succeeded but no body in the response (e.g. `DELETE`, CORS pre-flight) |

### 3xx — Redirection

| Code | Meaning |
|---|---|
| `301 Moved Permanently` | Resource has moved to a new URL permanently — future requests should use the new URL |
| `302 Found` | Resource is temporarily at a different URL — client should still use the original URL |
| `304 Not Modified` | Resource hasn't changed — client can use its cached copy |

### 4xx — Client Errors

| Code | Meaning |
|---|---|
| `400 Bad Request` | Client sent invalid/malformed data |
| `401 Unauthorized` | Invalid or missing credentials |
| `403 Forbidden` | Authenticated, but not permitted to perform this action |
| `404 Not Found` | Resource not found (incorrect URL) |
| `405 Method Not Allowed` | The HTTP method is not supported on this endpoint |
| `429 Too Many Requests` | Client exceeded the server's rate limit |

### 5xx — Server Errors

| Code | Meaning |
|---|---|
| `500 Internal Server Error` | Unexpected error in server-side logic |
| `501 Not Implemented` | Server doesn't support the requested feature |
| `502 Bad Gateway` | Proxy/load balancer received an invalid response from upstream |
| `503 Service Unavailable` | Server is down (maintenance/overload) |
| `504 Gateway Timeout` | Proxy/load balancer timed out waiting for upstream server |

---

## HTTP Caching

Caching stores copies of responses so they can be **reused**, saving server compute and reducing latency.

### How it works

The server includes cache directives in its response:
- `Cache-Control: max-age=60` — the response is valid for 60 seconds
- `ETag: "v1"` — a unique fingerprint of the resource

### Browser Caching — Three Scenarios

**Case 1: Request within TTL**
> Browser checks the cached URL. TTL hasn't expired (e.g. only 30s have passed out of 60s). Browser **serves from cache** — no network request made.

**Case 2: Request after TTL expires**
> Cache is stale, so browser sends the request with `If-None-Match: "v1"`. Server checks — resource hasn't changed — responds with `304 Not Modified`. **No data transferred**, browser serves from cache.

**Case 3: Resource has been updated**
> Browser sends `If-None-Match: "v1"`. Server's current ETag is `"v2"` — mismatch. Server responds with `200 OK`, new data, a new ETag, and a fresh `max-age`. Cache is updated.

### Cache Invalidation

- `GET` requests are **cache-friendly** (meant for cached fetches)
- `POST` and `PUT` requests are **cache-breaking** — they signal that cached data is now stale

---

## Content Negotiation

The mechanism by which client and server **agree on the format** of data being exchanged.

The client expresses its preferences via request headers:

```http
Accept: application/json
Accept-Language: en
Accept-Encoding: gzip
```

The server then responds in a matching format and declares it via `Content-Type`.

---

## HTTP 1.0 — How It Worked

HTTP 1.0 was simple by design: **one request → one response → connection closed**. Every HTTP exchange followed this lifecycle:

### Request/Response Lifecycle

```
1. Client opens a TCP socket to the server
2. TCP 3-way handshake (SYN → SYN-ACK → ACK)
3. Client sends:  GET /index.html HTTP/1.0
4. Server sends:  HTML response
5. Server calls close() on the socket
6. TCP 4-way teardown (FIN → ACK → FIN → ACK)
7. Connection is gone
```

### TCP 4-Way Teardown (Closing the Connection)

Once the server called `close()`, TCP performed a graceful shutdown:

| Step | Packet | Meaning |
|---|---|---|
| 1 | `FIN` (server → client) | "I have no more data to send" |
| 2 | `ACK` (client → server) | "Acknowledged, I know you're done" |
| 3 | `FIN` (client → server) | "I'm ready to close my side too" |
| 4 | `ACK` (server → client) | "Confirmed, goodbye" |

> **The connection is now dead.** Both the HTTP session and the TCP socket are gone.

---

## Problems with HTTP 1.0

### 1. New TCP connection per request

Every image, stylesheet, script, and icon on a page required:
- A full **TCP 3-way handshake** to open
- A full **TCP 4-way teardown** to close

For a page with 30 assets, that's 30 × (3 + 4) = **210 TCP packets** just for connection management, before any real data moves.

### 2. `TIME_WAIT` — The Port Exhaustion Problem

When a TCP connection closes, it doesn't vanish immediately. The socket enters a `TIME_WAIT` state for approximately **2 minutes** (OS-dependent). This is by design — it prevents stale packets from an old connection from being confused with a new one.

On a busy server, loading a single modern webpage (with dozens of assets) would leave dozens of sockets stuck in `TIME_WAIT`. Scale that to thousands of users and the server would **run out of available ports**, even though it wasn't doing any real work.

### 3. TCP Slow Start — Every Single Time

TCP has a built-in mechanism called **congestion control** (slow start). When a new connection opens, the Congestion Window starts at its minimum and ramps up gradually. Since HTTP 1.0 opened a new connection for every request, this ramp-up happened **over and over again** — the connection was never warm enough to reach full speed.

### 4. TLS Re-handshake (for HTTPS)

For every new connection over HTTPS, the full **TLS handshake** had to be repeated — adding multiple round trips of latency before a single byte of content was transferred.

### 5. Statelessness forces connection teardown

> *"The stateless nature of HTTP 1.0 is the reason for TCP overhead."*

HTTP 1.0 was deliberately stateless — the server kept no memory of previous requests. Because the server had no reason to remember the client, it had no reason to keep the connection alive. Once the response was sent, the server closed the socket and freed that memory. Statelessness and connection-per-request were philosophically linked.

The client knew the response was complete **because the connection dropped**. There was no other signal.

> Analogy: You order a pizza, pay, and leave. Every time you return for a refill, you have to reintroduce yourself — the waiter has no memory of you.

---

## HTTP 1.1 — The Evolution

HTTP 1.1 was introduced to solve the TCP overhead problem. The core insight:

> **Keep the TCP connection alive across multiple request/response cycles.**

### The Shift: "Close by Default" → "Keep-Alive by Default"

In HTTP 1.1, connections stay open unless explicitly told to close.

```http
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

- The client asks to keep the connection open for 5 seconds or up to 100 requests.
- Most servers (e.g. Nginx) default to a `keep-alive_timeout` of **60–75 seconds**.
- If no new request arrives in that window, the server sends a `FIN` to close and free memory.

### The New Problem: How Does the Client Know the Response Is Over?

In HTTP 1.0, the client knew the response ended because **the connection dropped**. With persistent connections, a new signal was needed. HTTP 1.1 introduced two mechanisms:

#### Mechanism 1: `Content-Length`

The server declares the exact byte count of the response body upfront. The client reads exactly that many bytes and then knows the response is complete — the TCP connection stays open.

```http
HTTP/1.1 200 OK
Content-Length: 13
Connection: keep-alive

Hello, World!
```

- Client reads 13 bytes → response done
- TCP connection remains `ESTABLISHED`
- Browser can immediately send the next request (e.g. `GET /style.css`) on the same socket — **no new handshake**

#### Mechanism 2: Chunked Transfer Encoding

Used when the server doesn't know the final response size upfront (e.g. streaming, paginated results, large report generation).

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Connection: keep-alive

7
Hello, 
6
World!
0
```

Each chunk is prefixed with its **size in hexadecimal bytes**:
- `7` → read the next 7 bytes (`Hello, `)
- `6` → read the next 6 bytes (`World!`)
- `0` → **end of message** — TCP connection stays open, but this HTTP transaction is complete

### A Full HTTP 1.1 Exchange

**Request:**
```http
GET /hello.txt HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/plain
Connection: keep-alive
```

**Response:**
```http
HTTP/1.1 200 OK
Date: Mon, 04 May 2026 09:15:00 GMT
Server: Apache/2.4.41 (Ubuntu)
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Connection: keep-alive

Hello, World!
```

The `Host` header is now **mandatory** in HTTP 1.1 — it allows a single server to host multiple domains (virtual hosting).

### What HTTP 1.1 Fixed

| Problem in HTTP 1.0 | HTTP 1.1 Solution |
|---|---|
| New TCP connection per request | Persistent connections (`Connection: keep-alive`) |
| TCP slow start on every request | Reuses the same "warm" TCP connection |
| `TIME_WAIT` port exhaustion | Far fewer connections opened and closed |
| TLS re-handshake per request | TLS session reused on the same connection |
| No way to signal end of response without closing | `Content-Length` and `Transfer-Encoding: chunked` |
| One host per IP | Mandatory `Host` header enables virtual hosting |

---

> **Summary:** HTTP 1.0's stateless "one request, one connection" design caused severe TCP overhead at scale. HTTP 1.1's persistent connections eliminated the redundant handshakes, made TCP's congestion control actually useful, and introduced `Content-Length` / chunked encoding as clean ways to delimit responses on a shared connection.
