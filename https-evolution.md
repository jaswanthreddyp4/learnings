# HTTPS: The Secure Web — From SSL to TLS 1.3

---

## Table of Contents

1. [What is HTTPS?](#what-is-https)
2. [Why HTTP Was Broken for Security](#why-http-was-broken-for-security)
3. [The Three Security Goals of HTTPS](#the-three-security-goals-of-https)
4. [SSL → TLS: The Evolution](#ssl--tls-the-evolution)
5. [How the TLS Handshake Works](#how-the-tls-handshake-works)
6. [Certificates and Certificate Authorities (CA)](#certificates-and-certificate-authorities-ca)
7. [TLS 1.2 vs TLS 1.3 — Key Differences](#tls-12-vs-tls-13--key-differences)
8. [HTTPS + HTTP/1.1 Combined Benefits](#https--http11-combined-benefits)
9. [HSTS — Forcing HTTPS at the Browser Level](#hsts--forcing-https-at-the-browser-level)
10. [Certificate Transparency](#certificate-transparency)
11. [Summary Table](#summary-table)

---

## What is HTTPS?

**HTTPS** (HyperText Transfer Protocol **Secure**) is not a new protocol — it is **HTTP running on top of TLS** (Transport Layer Security).

```
Application Layer  →  HTTP
Security Layer     →  TLS (Transport Layer Security)
Transport Layer    →  TCP
Network Layer      →  IP
```

The `S` in HTTPS does not mean a different protocol. It means the HTTP messages that were previously sent in **plaintext over TCP** are now **encrypted, authenticated, and integrity-checked** by TLS before being handed to TCP.

> **Key distinction:** HTTP defines *what* is communicated. TLS defines *how safely* it is communicated.

---

## Why HTTP Was Broken for Security

Plain HTTP has three fundamental vulnerabilities:

### 1. Eavesdropping (No Confidentiality)

Every HTTP request and response travels as **raw plaintext** over the network. Anyone on the same Wi-Fi, your ISP, or any router in between can read:
- Your session cookies
- Your passwords in form submissions
- Your private messages
- Your bank account numbers

```
Client  ──plaintext──▶  Router  ──plaintext──▶  ISP  ──plaintext──▶  Server
                           ↑
                    Attacker reads everything here
```

### 2. Tampering (No Integrity)

An attacker positioned between client and server (Man-in-the-Middle) can **modify** the content of requests and responses in transit. They could:
- Inject ads into a webpage
- Replace a software download with malware
- Alter financial transaction amounts

### 3. Impersonation (No Authentication)

HTTP has no way to verify that the server you're talking to is **actually who it claims to be**. An attacker can set up a server that looks like `linkedin.com` and your browser has no way to know the difference.

---

## The Three Security Goals of HTTPS

TLS solves all three problems:

| Goal | HTTP | HTTPS (TLS) |
|---|---|---|
| **Confidentiality** | ❌ Plaintext | ✅ Encrypted with symmetric key |
| **Integrity** | ❌ Tamperable in transit | ✅ MAC (Message Authentication Code) on every record |
| **Authentication** | ❌ No identity verification | ✅ Server identity verified by Certificate Authority |

---

## SSL → TLS: The Evolution

HTTPS started with **SSL** (Secure Sockets Layer), designed by Netscape in 1994. It evolved over decades into modern TLS.

| Version | Year | Status | Notes |
|---|---|---|---|
| **SSL 1.0** | Never released | ❌ Deprecated | Had critical security flaws, never deployed |
| **SSL 2.0** | 1995 | ❌ Deprecated | Multiple attacks found (DROWN attack) |
| **SSL 3.0** | 1996 | ❌ Deprecated | Broken by POODLE attack (2014) |
| **TLS 1.0** | 1999 | ❌ Deprecated | SSL 3.0 redesign; vulnerable to BEAST, POODLE |
| **TLS 1.1** | 2006 | ❌ Deprecated | Patched some TLS 1.0 issues; still weak |
| **TLS 1.2** | 2008 | ✅ Current | Strong; supports AEAD ciphers; widely deployed |
| **TLS 1.3** | 2018 | ✅ Current (recommended) | Faster, simpler, more secure; legacy crypto removed |

> **As of 2020**, TLS 1.0 and TLS 1.1 were officially deprecated by all major browsers (Chrome, Firefox, Safari, Edge).

### Why the SSL→TLS rename?

When IETF took over the protocol from Netscape, they renamed it TLS to reflect the standards body's ownership — not because the protocol changed dramatically. TLS 1.0 is essentially SSL 3.1 under the hood.

---

## How the TLS Handshake Works

The TLS handshake happens **before any HTTP data is exchanged**. Its job: negotiate encryption parameters and exchange keys.

### TLS 1.2 Handshake (2 Round Trips)

```
Client                                         Server
  |                                               |
  |──── ClientHello ──────────────────────────▶  |
  |     (supported cipher suites, TLS version,   |
  |      random bytes "client_random")            |
  |                                               |
  |  ◀── ServerHello ─────────────────────────   |
  |     (chosen cipher suite, TLS version,        |
  |      random bytes "server_random")            |
  |                                               |
  |  ◀── Certificate ─────────────────────────   |
  |     (server's public key + CA signature)      |
  |                                               |
  |  ◀── ServerHelloDone ─────────────────────   |
  |                                               |
  |  [Client verifies certificate with CA]        |
  |                                               |
  |──── ClientKeyExchange ────────────────────▶  |
  |     (pre-master secret encrypted with         |
  |      server's public key)                     |
  |                                               |
  |──── ChangeCipherSpec ─────────────────────▶  |
  |──── Finished ─────────────────────────────▶  |
  |                                               |
  |  ◀── ChangeCipherSpec ─────────────────────  |
  |  ◀── Finished ─────────────────────────────  |
  |                                               |
  |  ══ TLS Tunnel Established ══════════════    |
  |──── GET /feed/ HTTP/1.1 ──────────────────▶  |  ← encrypted
```

**Key derivation:** Both sides independently compute the same **session key** from:
- `client_random` + `server_random` + `pre-master secret` → **master secret** → **session keys**

This means the private key is only used once (to decrypt the pre-master secret). All actual data encryption uses the symmetric session key.

---

### TLS 1.3 Handshake (1 Round Trip — 0-RTT possible)

TLS 1.3 dramatically simplified the handshake:

```
Client                                         Server
  |                                               |
  |──── ClientHello ──────────────────────────▶  |
  |     (supported cipher suites, key_share,      |
  |      client_random)                           |
  |                                               |
  |  ◀── ServerHello + Certificate ─────────────  |
  |  ◀── CertificateVerify + Finished ──────────  |
  |                                               |
  |──── Finished ─────────────────────────────▶  |
  |                                               |
  |  ══ TLS Tunnel Established ══════════════    |
  |──── GET /feed/ HTTP/1.1 ──────────────────▶  |  ← encrypted
```

**The key improvement:** In TLS 1.3, the client **already sends its key share** in the first `ClientHello` (it guesses the key exchange algorithm the server will pick). This saves an entire round trip.

### 0-RTT (Zero Round Trip Time Resumption)

For clients that have connected before, TLS 1.3 supports **0-RTT resumption** using a Pre-Shared Key (PSK) from the previous session. The client can send encrypted HTTP data **on the very first packet** — before the handshake even completes.

> **Tradeoff:** 0-RTT has a replay attack risk — an attacker who captures the first packet can replay it. It should only be used for idempotent requests (e.g. `GET`).

---

## Certificates and Certificate Authorities (CA)

### What Is a Certificate?

A TLS certificate is a digitally signed document that contains:
- The domain name it is valid for (e.g. `*.linkedin.com`)
- The server's **public key**
- The issuing **Certificate Authority (CA)**
- The **expiration date**
- A **digital signature** from the CA

### The Chain of Trust

You trust a website because your OS/browser ships with a pre-installed list of **trusted Root CAs** (e.g. DigiCert, Let's Encrypt, Sectigo). When a server presents its certificate, your browser validates:

```
Root CA (trusted by OS/browser)
   └── Intermediate CA (signed by Root CA)
         └── Server Certificate (signed by Intermediate CA)
```

1. Browser receives the server's certificate
2. It checks the CA's signature on the certificate using the CA's public key
3. It walks up the chain until it reaches a trusted Root CA
4. If the chain is valid and the domain matches → ✅ Trusted
5. If any step fails → ❌ "Your connection is not private"

### Certificate Types

| Type | Validates | Padlock | Use Case |
|---|---|---|---|
| **DV** (Domain Validated) | Domain ownership only | ✅ | Most websites, blogs, APIs |
| **OV** (Organisation Validated) | Domain + company identity | ✅ | Business websites |
| **EV** (Extended Validation) | Full legal entity vetting | ✅ (was green bar) | Banks, high-security sites |
| **Wildcard** | `*.example.com` (all subdomains) | ✅ | CDNs, large platforms |
| **SAN** (Subject Alternative Name) | Multiple domains in one cert | ✅ | e.g. `linkedin.com` + `www.linkedin.com` |

---

## TLS 1.2 vs TLS 1.3 — Key Differences

| Feature | TLS 1.2 | TLS 1.3 |
|---|---|---|
| **Handshake round trips** | 2 RTT | 1 RTT (0-RTT possible) |
| **Key exchange** | RSA or ECDHE | ECDHE only (RSA removed) |
| **Cipher suites** | 37 options (many weak) | 5 curated, all AEAD |
| **Forward secrecy** | Optional | ✅ Mandatory (ECDHE always) |
| **Encryption start** | After handshake | Certificate encrypted too |
| **Deprecated features removed** | No | ✅ RSA key exchange, SHA-1, RC4, 3DES, MD5 |
| **Session resumption** | Session IDs / Tickets | PSK-based, more secure |

### Forward Secrecy — Why It Matters

**Forward secrecy** (or Perfect Forward Secrecy — PFS) means that even if an attacker records all your encrypted traffic **today**, and then steals the server's private key **in the future**, they **still cannot decrypt** the old recorded traffic.

- **TLS 1.2 without PFS (RSA key exchange):** The same private key decrypts the pre-master secret. Steal the key later → decrypt everything retroactively.
- **TLS 1.2/1.3 with ECDHE:** A fresh ephemeral key pair is generated for every session. The server's long-term private key is never used for key exchange — only for authentication. Old sessions cannot be decrypted.

---

## HTTPS + HTTP/1.1 Combined Benefits

Connecting back to what you learned in `http-evolution.md`:

HTTP 1.1 introduced **persistent connections** (`Connection: keep-alive`). HTTPS benefits from this massively:

| Scenario | HTTP 1.0 + HTTPS | HTTP 1.1 + HTTPS |
|---|---|---|
| TCP handshake per asset | ✅ Yes (expensive) | ✅ Once per connection |
| TLS handshake per asset | ✅ Yes (2 RTT each) | ✅ Once per connection |
| TCP slow start per asset | ✅ Yes | ✅ Connection stays "warm" |
| Total overhead for 30 assets | 30 × (TCP + TLS) handshakes | 1 × (TCP + TLS) handshake |

> **This is why HTTP 1.1's persistent connections were doubly important** — they didn't just save TCP handshakes, they saved the expensive TLS handshake overhead too.

### Session Resumption (TLS Session Tickets)

In HTTP 1.1 + HTTPS, if a connection does close and the client reconnects:

1. The server issues a **session ticket** — an encrypted blob of the session parameters
2. The client stores it
3. On reconnect, the client presents the ticket → server decrypts it → session resumes **without a full TLS handshake**

This dramatically reduces re-connection cost.

---

## HSTS — Forcing HTTPS at the Browser Level

Even with HTTPS available, a user could type `http://linkedin.com` and get an insecure connection (even briefly, before a redirect). **HSTS (HTTP Strict Transport Security)** eliminates this.

### How It Works

When the server responds over HTTPS, it includes:

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- `max-age=31536000` — browser remembers to use HTTPS for this domain for 1 year
- `includeSubDomains` — applies to all subdomains too
- `preload` — domain is hardcoded into browsers' HSTS preload list (ships with Chrome/Firefox)

After the first HTTPS visit, the browser **refuses to make HTTP connections** to that domain — it upgrades them to HTTPS **before sending any packet**. Not even a redirect is needed.

### HSTS Preload List

Major sites like `linkedin.com`, `google.com`, and `github.com` are on the HSTS preload list. This means:

> Even on your **very first visit** (before ever receiving an HSTS header), your browser already knows to use HTTPS — because it was compiled into the browser binary.

---

## Certificate Transparency

**CT (Certificate Transparency)** is a public audit log of every TLS certificate ever issued. Browsers like Chrome **require** that certificates be logged in CT logs, and verify the SCT (Signed Certificate Timestamp) to prove it.

This prevents:
- A rogue CA from issuing a fraudulent certificate for `linkedin.com` without LinkedIn knowing
- Domain owners can monitor CT logs and be alerted if an unexpected certificate is issued for their domain

---

## Summary Table

| Feature | Plain HTTP | HTTPS (TLS 1.2) | HTTPS (TLS 1.3) |
|---|---|---|---|
| **Confidentiality** | ❌ | ✅ AES-GCM | ✅ AES-GCM / ChaCha20 |
| **Integrity** | ❌ | ✅ HMAC | ✅ AEAD (built-in) |
| **Authentication** | ❌ | ✅ Certificate + CA | ✅ Certificate + CA |
| **Forward Secrecy** | ❌ | Optional (ECDHE) | ✅ Mandatory |
| **Handshake RTTs** | 0 (no TLS) | 2 RTT | 1 RTT (0-RTT possible) |
| **HSTS support** | ❌ | ✅ | ✅ |
| **Certificate Transparency** | ❌ | ✅ | ✅ |
| **Legacy crypto removed** | — | No | ✅ |

---

> **Summary:** HTTPS = HTTP + TLS. It layered three critical security properties — confidentiality, integrity, and authentication — on top of HTTP without changing the semantics of the protocol itself. TLS evolved from SSL through a series of security audits and attacks, each generation removing weak cryptography and streamlining the handshake. TLS 1.3 in 2018 was a clean-break redesign: 1-RTT handshake, mandatory forward secrecy, and all legacy cipher suites removed.
