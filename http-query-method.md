# The HTTP QUERY Method — Explained Simply

> **Source:** [draft-ietf-httpbis-safe-method-w-body](https://httpwg.org/http-extensions/draft-ietf-httpbis-safe-method-w-body.html)  
> **Status:** Internet-Draft (Standards Track) — IETF HTTP Working Group, June 2026

---

## What is this?

This is a proposal to add a **brand-new HTTP method called `QUERY`** to the web.

Just like `GET`, `POST`, `PUT`, `DELETE` — `QUERY` would be an official HTTP method that every server and browser would understand.

---

## The Problem It Solves

When you want to **ask a server for data** (i.e., run a search or query), you currently have two bad options:

### Option 1: Use `GET`
```http
GET /contacts?name=John&age=30&city=NewYork&limit=50 HTTP/1.1
```
❌ **Problems:**
- All your query parameters go into the **URL**, which has length limits
- URLs get **logged** everywhere — servers, proxies, browser history — so sensitive search terms are exposed
- Complex queries (e.g. JSON, SQL) are very ugly to encode into a URL

### Option 2: Use `POST`
```http
POST /contacts HTTP/1.1
Content-Type: application/json

{ "name": "John", "age": 30, "city": "New York" }
```
❌ **Problems:**
- `POST` tells the internet "this request **changes something**"
- Caches and proxies won't cache it
- It can't be safely retried automatically
- It's misleading — you're just *searching*, not creating/changing anything

### The fix: `QUERY`

`QUERY` gives you the **best of both worlds**:
- ✅ Sends data in the **request body** (like POST) — no URL length limits
- ✅ Is **safe and idempotent** (like GET) — nothing changes, it can be retried and cached

---

## What Does a QUERY Request Look Like?

```http
QUERY /contacts HTTP/1.1
Host: example.org
Content-Type: application/x-www-form-urlencoded
Accept: application/json

select=surname,givenname,email&limit=10&match="email=*@example.*"
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  { "surname": "Smith",  "givenname": "John",    "email": "smith@example.org" },
  { "surname": "Jones",  "givenname": "Sally",   "email": "sally@example.com" },
  { "surname": "Dubois", "givenname": "Camille", "email": "camille@example.net" }
]
```

You can use **any format** in the body — form-encoded, JSON, SQL, JSONPath, XML, etc.

---

## Key Properties of QUERY

| Property | GET | POST | **QUERY** |
|---|:---:|:---:|:---:|
| Safe (doesn't change data) | ✅ | ❌ | ✅ |
| Idempotent (safe to retry) | ✅ | ❌ | ✅ |
| Can have a request body | ❌ | ✅ | ✅ |
| Responses can be cached | ✅ | ⚠️ limited | ✅ |

---

## Important Rules

### 1. You MUST include a `Content-Type`
The server needs to know what format your query is in.  
If you forget it → **400 Bad Request**.  
If the server doesn't support your format → **415 Unsupported Media Type**.

### 2. Caching works, but the body is part of the cache key
Unlike GET (where only the URL is the cache key), for QUERY the **URL + body + Content-Type** together form the cache key. Same query body = cache hit.

### 3. The server can give you a shortcut GET URL
After a QUERY, the server can respond with:

- **`Location` header** → a GET URL to *re-run the same query* anytime
- **`Content-Location` header** → a GET URL to *retrieve this exact result snapshot*

```http
HTTP/1.1 200 OK
Content-Type: application/json
Location: /contacts/stored-queries/42
Content-Location: /contacts/stored-results/17
```

### 4. Conditional requests are supported
You can use `If-Modified-Since` or `If-None-Match` — if nothing changed, you get back `304 Not Modified` (no wasted data transfer).

### 5. Redirects work like normal
- `301`, `302`, `307`, `308` → redirect, resend QUERY to new URL
- `303 See Other` → go GET the result from a different URL (no body returned)

---

## The New `Accept-Query` Header

Servers can advertise which query formats they support:

```http
HTTP/1.1 200 OK
Accept-Query: application/sql, application/jsonpath, application/x-www-form-urlencoded
```

This tells clients: *"You can send me QUERY requests using these formats."*

---

## A Real-World Example: JSONPath Query

```http
QUERY /errata.json HTTP/1.1
Host: example.org
Content-Type: application/jsonpath
Accept: application/json

$..[?@.errata_status_code=="Rejected" && @.submit_date>"2024"]["doc-id"]
```

This runs a JSONPath expression as a query against `/errata.json` — something impossible to express cleanly in a GET URL.

---

## Security & Privacy Notes

- 🔒 **Better privacy than GET** — query bodies are less likely to be logged than URL parameters
- ⚠️ **Don't put sensitive data in the `Location` URI** — if the server creates a GET-able URL from the query, it shouldn't embed sensitive query content in that URL
- 🌐 **CORS requires a preflight** — `QUERY` is not in the browser's CORS safe-list, so cross-origin QUERY requests will need an `OPTIONS` preflight first

---

## Why Not Just Use `SEARCH`?

WebDAV already has `SEARCH`, `REPORT`, and `PROPFIND` — but they're all XML-only and tied to WebDAV history. `QUERY` is:

- **Format-agnostic** — works with any `Content-Type`
- **Not tied to WebDAV** — a clean, general-purpose method for the modern web
- **Named after the URI query component** — makes the semantic intent obvious

---

## Current Status

| | |
|---|---|
| **Type** | Internet-Draft (not yet an RFC) |
| **Track** | Standards Track |
| **Revision** | draft-14 (June 2026) |
| **Working Group** | IETF HTTP Working Group |
| **Expires** | December 2026 |

This is still being reviewed and finalized. It has gone through 14 revision rounds, which shows strong community interest and active development.

---

## TL;DR

> **`QUERY` is a new HTTP method for sending search/query requests with a body, while still being safe and cacheable like GET.**  
> It fixes the ugly hack of using POST for queries, and the URL-length problem with GET.  
> Think of it as: **"GET with a body that doesn't change anything."**
