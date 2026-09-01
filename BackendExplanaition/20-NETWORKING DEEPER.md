# LEVEL 20 — Networking, Go Much Deeper

## The Full Path, As a Map for This Document

```
Browser
   │   1. "what IP is yourapp.com?"
   ▼
  DNS                    ─────────────▶  PART A
   │   2. got an IP address
   ▼
  TCP                    ─────────────▶  PART B
   │   3. reliable connection established
   ▼
  TLS                    ─────────────▶  PART C
   │   4. connection now encrypted
   ▼
  HTTP                   ─────────────▶  PART D
   │   5. the actual request/response
   ▼
Load Balancer  ──▶  Reverse Proxy  ──▶  Application     (covered in Levels 14 & 17)
```

Every layer below solves a different problem: DNS answers "where is it," TCP answers "how do I
reliably deliver bytes," TLS answers "how do I keep this private," HTTP answers "what am I
actually asking for."

---

# PART A — DNS

## Domain

The human-readable name (`yourapp.com`) you register through a registrar. It doesn't host
anything by itself — it's just a name you can point at whatever infrastructure you want.

## IP Address

The actual numeric address machines use to route traffic (see Part B for IPv4/IPv6 detail).

## DNS Record Types

DNS isn't just "name → IP" — it's a general-purpose key-value system for domain-related lookups,
with different record **types** for different kinds of answers.

### A Record

Maps a domain name directly to an **IPv4** address. The most fundamental record type.

```
yourapp.com.   A   203.0.113.42
```

### AAAA Record

Same idea as A, but maps to an **IPv6** address (the extra "A"s are a nod to IPv6 addresses being
4x longer than IPv4).

```
yourapp.com.   AAAA   2001:0db8::ff00:0042:8329
```

### CNAME Record

An **alias** — points one name to another name (which then gets resolved itself), rather than
directly to an IP. Useful for pointing subdomains at services without hardcoding IPs that might
change.

```
www.yourapp.com.   CNAME   yourapp.com.
blog.yourapp.com.  CNAME   yourapp.ghost.io.
```

```
www.yourapp.com  ──CNAME──▶  yourapp.com  ──A──▶  203.0.113.42
(a CNAME lookup requires ONE MORE hop than a direct A record lookup)
```

**Rule:** a CNAME can't coexist with other records at the same name (e.g., you can't have both a
CNAME and an MX record on the exact same subdomain) — this is why the root domain itself
(`yourapp.com` with no subdomain) usually can't be a CNAME, and needs an "ALIAS"/"ANAME" record
(a provider-specific workaround) or a plain A record instead.

### MX Record

Specifies which mail servers handle email for the domain, with a priority value (lower = tried
first).

```
yourapp.com.   MX   10   mail1.yourapp.com.
yourapp.com.   MX   20   mail2.yourapp.com.   (backup, tried only if mail1 fails)
```

### TXT Record

Free-form text attached to a domain — used for arbitrary verification/metadata, most commonly
domain ownership verification and email anti-spoofing (SPF/DKIM/DMARC records).

```
yourapp.com.   TXT   "v=spf1 include:_spf.google.com ~all"     (SPF — who's allowed to send email as you)
yourapp.com.   TXT   "google-site-verification=abc123..."       (proving you own this domain to Google)
```

### NS Record

Specifies which **name servers** are authoritative for answering DNS queries about this domain —
these are the servers that actually hold all the other records (A, MX, TXT, etc.) for the domain.

```
yourapp.com.   NS   ns1.awsdns-01.com.
yourapp.com.   NS   ns2.awsdns-02.org.
```

## DNS Resolution — The Full Lookup Process

```
Browser needs the IP for "yourapp.com"
          │
          ▼
┌───────────────────┐
│  Browser/OS cache      │  "have I looked this up recently?" → if yes, done, skip everything below
└─────────┬──────────┘
          │  cache miss
          ▼
┌───────────────────┐
│  Recursive Resolver     │  (usually your ISP's or a public one like 8.8.8.8) —
│  (does the legwork        │   does the multi-step lookup below ON YOUR BEHALF
│   on your behalf)          │
└─────────┬──────────┘
          │
          ▼
┌───────────────────┐
│  Root Nameserver        │  "I don't know yourapp.com specifically, but I know who
│                          │   handles .com — ask them" (13 root server clusters worldwide)
└─────────┬──────────┘
          ▼
┌───────────────────┐
│  TLD Nameserver          │  (handles .com specifically) "I don't have the records,
│  (.com registry)          │   but here are yourapp.com's NS records — ask them"
└─────────┬──────────┘
          ▼
┌───────────────────┐
│  Authoritative Nameserver│  the actual NS records for yourapp.com — HAS the real
│  (for yourapp.com)        │   answer: "A record → 203.0.113.42"
└─────────┬──────────┘
          │
          ▼
      Answer flows back up through the resolver to the browser
      (and gets CACHED at multiple levels along the way, per each record's TTL)
```

This whole chain typically completes in milliseconds and is heavily cached at every level — most
real-world lookups short-circuit at the browser/OS cache or the recursive resolver's cache
without ever reaching the root/TLD servers.

## TTL (Time To Live)

How long a DNS answer can be cached before it must be looked up again — set per record, in
seconds.

```
yourapp.com.   A   203.0.113.42   TTL=3600    (cache this for 1 hour)
```

```
Low TTL (e.g. 60s):    changes propagate fast, but MORE lookups hit your DNS provider
                       (useful right before/during a planned IP change or failover)
High TTL (e.g. 86400s): fewer lookups, less DNS provider load, but changes take
                       longer to be seen everywhere
```

## DNS Propagation

When you change a DNS record, the change doesn't take effect everywhere instantly — every
resolver/cache that had the OLD value cached will keep serving it until that cached entry's TTL
expires. This is why "DNS propagation" can take anywhere from minutes to (rarely) 48 hours,
depending on the TTL that was set before the change and how widely cached the old value was.

```
t=0     You change the A record from IP_OLD to IP_NEW at your DNS provider
t=0     Resolver A (already cached, TTL not yet expired) → still returns IP_OLD
t=0     Resolver B (never queried this before) → queries fresh, gets IP_NEW immediately
t=+1hr  Resolver A's cached TTL expires → next query re-fetches, now returns IP_NEW too
```

**Practical implication:** if you're planning a migration, lower the TTL well *before* the
change (e.g., a day ahead) so that by the time you actually change the record, the old, longer
TTL has already expired everywhere and the new low TTL lets the change propagate fast.

---

# PART B — TCP/IP

## TCP vs UDP

Two transport-layer protocols, with opposite trade-offs.

```
TCP (Transmission Control Protocol):        UDP (User Datagram Protocol):
- Connection-oriented (handshake first)      - Connectionless (just send it)
- Guarantees delivery & ORDER                - No delivery guarantee, no ordering
- Automatic retransmission of lost packets    - Lost packets are just... lost
- Slower (overhead of guarantees)              - Faster (no overhead)
- Use: web (HTTP), email, file transfer         - Use: video calls, live streaming, gaming,
                                                   DNS queries — where a LATE packet is
                                                   worse than a LOST one (you don't want an
                                                   old video frame re-sent 2 seconds later)
```

```
TCP:  [pkt1]──▶ ✓ ack  [pkt2]──▶ ✗ LOST, resend  [pkt2 again]──▶ ✓ ack  [pkt3]──▶ ✓ ack
      (guaranteed, in order, but a lost packet stalls everything behind it)

UDP:  [pkt1]──▶  [pkt2]──▶ ✗ LOST, nobody cares  [pkt3]──▶
      (fire and forget — the receiving app decides if/how to handle gaps)
```

## Three-Way Handshake (establishing a TCP connection)

```
Client                                          Server
   │                                                │
   │──────────────── SYN (seq=x) ─────────────────▶│   "I want to connect, my starting sequence is x"
   │                                                │
   │◀────────── SYN-ACK (seq=y, ack=x+1) ──────────│   "OK, acknowledged, my starting sequence is y"
   │                                                │
   │──────────────── ACK (ack=y+1) ────────────────▶│   "acknowledged, connection established"
   │                                                │
   │◀═══════════ connection is now OPEN ═══════════▶│
```

**Why three steps, not just one?** Both sides need to prove they can both SEND and RECEIVE —
a single SYN only proves the client can send; the SYN-ACK proves the server received it AND can
send back; the final ACK proves the client received THAT — after this, both directions are
verified as working.

## Four-Way Termination (closing a TCP connection)

```
Client                                          Server
   │──────────────── FIN ──────────────────────▶│   "I'm done sending"
   │◀─────────────── ACK ───────────────────────│   "acknowledged"
   │◀─────────────── FIN ───────────────────────│   "I'm done sending too"
   │──────────────── ACK ───────────────────────▶│   "acknowledged, fully closed"
```

**Why four steps instead of the three used to open?** Closing is independent per direction — one
side finishing doesn't mean the other side is also done sending, so each direction gets its own
FIN/ACK pair (which is why it's sometimes just 3 packets in practice, if the server bundles its
ACK and FIN together).

## Connection

A connection is the **stateful, established communication channel** between two endpoints,
uniquely identified by the 4-tuple: `(source IP, source port, destination IP, destination port)`.
This is why a server can handle thousands of simultaneous client connections on one listening
port — each client has a different source IP/port, making each connection's tuple unique.

## Port

A numbered endpoint (0–65535) identifying which application on a machine should receive traffic
(covered in Level 12).

## Socket

The programming interface / endpoint object representing one side of a connection — the actual
thing your code reads from and writes to (covered in Level 12).

## NAT (Network Address Translation)

Lets many devices on a private network (e.g., your home network's 192.168.x.x devices) share ONE
public IP address when talking to the internet — the router rewrites each outgoing packet's
private source IP/port to its own public IP and a unique port, and reverses the mapping for
replies.

```
┌───────────── Private network (behind NAT) ─────────────┐
│  Laptop: 192.168.1.5     Phone: 192.168.1.6                │
└─────────────────┬────────────────────────────────────┘
                    │  NAT rewrites: 192.168.1.5:54321 → 203.0.113.9:60001
                    │                192.168.1.6:54322 → 203.0.113.9:60002
                    ▼
              Router (public IP: 203.0.113.9)
                    │
                    ▼
                Internet

From the internet's perspective, BOTH devices appear to be 203.0.113.9 —
the router remembers which port maps back to which internal device for replies.
```

**Why NAT exists:** IPv4 address exhaustion — there aren't enough public IPv4 addresses for every
device on Earth to have its own, so NAT lets entire networks share one public address.

## IPv4 vs IPv6

```
IPv4: 203.0.113.42                          32-bit address space  ≈ 4.3 billion addresses
                                             (already exhausted — this is WHY NAT is so widespread)

IPv6: 2001:0db8::ff00:0042:8329             128-bit address space ≈ 340 undecillion addresses
                                             (practically limitless — every device could have
                                              its own public address, no NAT needed)
```

Adoption has been gradual because IPv4 and IPv6 aren't directly interoperable — the internet is
currently in a long transition period where most systems support both ("dual stack").

---

# PART C — TLS

## SSL/TLS

TLS (Transport Layer Security) is the modern protocol that encrypts data in transit; "SSL" is its
older, now-deprecated predecessor — people still say "SSL certificate" colloquially, but the
actual protocol in use today is TLS.

## Certificates

A digital document proving a server's identity, containing its public key, the domain(s) it's
valid for, an expiration date, and a digital signature from a trusted Certificate Authority.

```
Certificate for yourapp.com
├── Public key: (used to encrypt data only the server can decrypt with its private key)
├── Valid for: yourapp.com, www.yourapp.com
├── Issued by: Let's Encrypt (a Certificate Authority)
├── Valid: Jan 1, 2026 – Apr 1, 2026
└── Signature: [cryptographic proof the CA vouches for this certificate]
```

## Certificate Authority (CA)

A trusted third party (Let's Encrypt, DigiCert, etc.) that verifies a domain's ownership before
issuing it a certificate, and cryptographically signs that certificate — browsers/OSes ship with
a built-in list of CAs they trust, which is the entire foundation of why HTTPS certificates work
without you personally verifying every server's identity yourself.

```
                     ┌───────────────┐
   You request a       │ Certificate      │  verifies you control yourapp.com
   cert for              │ Authority         │  (e.g., via a DNS TXT record or file challenge),
   yourapp.com           │ (e.g. Let's         │  then issues + signs a certificate
                        │ Encrypt)           │
                     └───────────────┘
                              │
                              ▼
                  Your server presents this signed cert to visitors
                              │
                              ▼
                  Browser checks: "is this signed by a CA I already trust?"
                  (browsers ship with a built-in list of trusted root CAs)
```

## Public/Private Keys

TLS uses **asymmetric cryptography**: two mathematically linked keys — anything encrypted with
the public key can only be decrypted with the matching private key, and vice versa for signing.

```
Public key:  shared with EVERYONE, safe to expose (it's literally in the certificate)
Private key: kept SECRET on the server, never transmitted anywhere

Encryption:  Client encrypts data using the server's PUBLIC key
             → only the server (holding the PRIVATE key) can decrypt it

Signing:     Server signs data using its PRIVATE key
             → anyone with the PUBLIC key can verify the signature is authentic
```

## TLS Handshake

Establishes an encrypted channel — happens right after the TCP three-way handshake, before any
actual HTTP data is exchanged.

```
Client                                              Server
   │──── ClientHello (supported TLS versions, ────▶│
   │      cipher suites, a random number)            │
   │                                                  │
   │◀─── ServerHello (chosen cipher suite,           │
   │      a random number, the SERVER CERTIFICATE) ──│
   │                                                  │
   │  [Client verifies the certificate against         │
   │   trusted CAs — checks domain match, expiry,        │
   │   and the CA's signature]                            │
   │                                                  │
   │──── Client generates a "pre-master secret",      │
   │      encrypts it with the server's PUBLIC key,   ─▶│
   │      sends it                                    │
   │                                                  │
   │  [Both sides now independently derive the         │
   │   SAME symmetric session key from the              │
   │   pre-master secret + both random numbers]           │
   │                                                  │
   │◀════ rest of the connection uses fast            │
   │       SYMMETRIC encryption with this session key ═▶│
```

**Why switch to symmetric encryption after the handshake, instead of using public/private keys
for everything?** Asymmetric crypto (public/private key) is computationally expensive; symmetric
crypto (same shared key both directions) is much faster. TLS uses the slow, secure asymmetric
method just once, to safely establish a shared secret, then uses that shared secret with fast
symmetric encryption for the actual bulk data transfer.

## HTTPS

Simply **HTTP running on top of a TLS-encrypted connection** — same HTTP semantics (methods,
headers, status codes) you already know, just wrapped in encryption so it can't be read or
tampered with in transit.

```
HTTP:   Browser ──── plaintext request/response ────▶ Server   (readable by anyone in the middle)
HTTPS:  Browser ══ TLS-encrypted request/response ══▶ Server   (unreadable to anyone but the two ends)
```

---

# PART D — HTTP, Deeper

## HTTP/1.1

The long-standing standard: text-based, one request per connection by default (though
`keep-alive` improved this — see below). Each request/response is a simple, readable text
exchange.

```
GET /users HTTP/1.1
Host: yourapp.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42

{"users": [...]}
```

**Limitation — Head-of-Line Blocking:** even with keep-alive, HTTP/1.1 typically processes
requests on a connection one at a time in order — a slow response blocks everything queued
behind it on that same connection. Browsers work around this by opening several parallel
connections per host (commonly 6).

## HTTP/2

Introduces **multiplexing** — many requests and responses can be in flight simultaneously over a
**single** TCP connection, each broken into small binary "frames" that get interleaved and
reassembled at the other end. Also adds header compression (HPACK) and server push.

```
HTTP/1.1 (needs multiple connections to parallelize):
Connection 1: [ Request A ─────▶ Response A ]
Connection 2: [ Request B ─▶ Response B ]
Connection 3: [ Request C ───▶ Response C ]

HTTP/2 (one connection, multiplexed):
Connection 1: [A-frame][B-frame][A-frame][C-frame][B-frame][A-frame]...
              all interleaved on ONE connection, reassembled independently
              — a slow response for A doesn't block B or C from completing
```

**Remaining limitation:** HTTP/2 solves head-of-line blocking at the HTTP layer, but it's still
running over ONE TCP connection — if that TCP connection experiences packet loss, TCP's own
in-order delivery guarantee stalls ALL the multiplexed streams until the lost packet is
retransmitted (this is "TCP-level head-of-line blocking," and it's what HTTP/3 fixes).

## HTTP/3

Runs over **QUIC** (built on UDP, not TCP) instead of TCP — each multiplexed stream has
*independent* loss recovery, so a lost packet only stalls the one stream it belongs to, not every
other in-flight request on the connection. Also has a faster, built-in TLS-integrated handshake
(fewer round trips to establish a fully secure connection than the traditional TCP+TLS handshake
combo).

```
HTTP/2 over TCP:  one lost packet ──▶ stalls ALL streams (TCP-level head-of-line blocking)
HTTP/3 over QUIC: one lost packet ──▶ stalls ONLY that one stream, others continue unaffected
```

## Keep-Alive

Reuses the same TCP connection for multiple sequential requests instead of opening a brand-new
TCP (and TLS) connection for every single request — avoiding the overhead of a fresh handshake
each time.

```
Without keep-alive:  Request 1 [full TCP+TLS handshake] → response → CLOSE connection
                       Request 2 [full TCP+TLS handshake AGAIN] → response → CLOSE
                       (expensive — handshakes aren't free)

With keep-alive:      Request 1 [TCP+TLS handshake ONCE] → response
                       Request 2 [reuse SAME connection] → response
                       Request 3 [reuse SAME connection] → response
                       ... connection closes after an idle timeout or explicit close
```

```
Connection: keep-alive     ← this header (HTTP/1.1 default, actually) signals this behavior
```

## Connection Pooling

The client-side practice of **reusing a pool of already-open connections** across many logical
requests, instead of opening/closing connections per request — this is what makes keep-alive
actually useful at the application level.

```python
import httpx

# Without pooling — a new connection (and its handshake cost) for every single call
for url in urls:
    httpx.get(url)   # each call may open a fresh connection

# With pooling — a shared client reuses open connections across all calls
with httpx.Client() as client:
    for url in urls:
        client.get(url)    # reuses an existing connection whenever possible, MUCH faster
```

**Why this matters for backend code specifically:** database drivers, HTTP clients calling other
services, and Redis clients all benefit enormously from connection pooling — creating a fresh
connection per request is one of the most common, easily-fixed sources of unnecessary latency in
backend systems.

## Chunked Transfer Encoding

Lets the server start sending a response **before it knows the total size** — useful for
streaming large or dynamically-generated content, since normally HTTP needs a `Content-Length`
header up front.

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
0\r\n
\r\n
```

```
Without chunking:  server must compute the ENTIRE response first, THEN send Content-Length + body
With chunking:     server sends pieces AS THEY BECOME AVAILABLE — e.g., streaming a live
                    log tail, a large generated report, or an LLM's token-by-token response
```

## Compression

Reduces the number of bytes actually sent over the wire — the client advertises what it supports,
the server picks one and compresses accordingly.

```
Request:   Accept-Encoding: gzip, br, deflate
Response:  Content-Encoding: gzip
           [gzip-compressed bytes — client decompresses before use]
```

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
```

**Why it matters:** a JSON API response or a JS bundle can often shrink by 70-90% with gzip/
brotli — directly reduces latency (less data to transfer) especially on slower connections.

## HTTP Caching

Lets clients (and intermediate caches/CDNs) reuse a previous response instead of re-fetching it,
governed by response headers the server sets.

```
Cache-Control: max-age=3600, public
```

```
Client requests /logo.png
   │
   ▼
Already have a cached copy, and max-age hasn't expired?
   │
   ├── YES ──▶ use the cached copy, NO network request at all — instant
   │
   └── NO  ──▶ fetch fresh from the server, cache the new response per its headers
```

```
Cache-Control: no-store           → never cache this at all (sensitive data)
Cache-Control: private            → only the browser can cache it, not shared/CDN caches
Cache-Control: public, max-age=86400 → anyone (including CDNs) can cache for 1 day
```

## ETag

An opaque identifier (usually a hash of the content) the server attaches to a response, letting
clients cheaply ask "has this actually changed since I last saw it?" without re-downloading the
full content just to find out.

```
First request:
   Response:  ETag: "a1b2c3d4"
              [full response body]

Later request (cached copy might be stale):
   Request:   If-None-Match: "a1b2c3d4"
   Response:  304 Not Modified          (empty body — content hasn't changed, use your cached copy!)
              — or —
   Response:  200 OK, ETag: "e5f6g7h8"  (content DID change, here's the new version)
```

## Conditional Requests

The general mechanism ETag (and `Last-Modified`) enable — a request that says "only give me the
full data if it's actually different from what I specify," using headers like `If-None-Match`
(paired with ETag) or `If-Modified-Since` (paired with `Last-Modified`).

```
┌────────┐  If-None-Match: "a1b2c3d4"  ┌────────┐
│ Client   │ ──────────────────────────▶│ Server   │
└────────┘                             └───┬────┘
                                             │  content hash still matches "a1b2c3d4"?
                              ┌──────────────┴──────────────┐
                        YES: unchanged                NO: changed
                              │                              │
                              ▼                              ▼
                    304 Not Modified                200 OK + new body + new ETag
                    (tiny response,                  (full response, since the
                     saves bandwidth AND               client's cached copy really
                     server processing)                 is out of date)
```

**Why this matters for backend design:** implementing ETags on expensive-to-generate or
rarely-changing resources (e.g., a user's profile that changes infrequently) can dramatically cut
bandwidth and even let you skip regenerating the response body entirely when nothing has changed
— you only need to check "did the underlying data change" (often just a version number or hash
comparison), which is usually much cheaper than reserializing the whole response.

---

# Full Path, Revisited With Everything Filled In

```
Browser
   │  DNS resolution: yourapp.com → 203.0.113.42 (via A record, possibly through a CNAME chain,
   │                   cached per each record's TTL)
   ▼
  TCP
   │  Three-way handshake: SYN → SYN-ACK → ACK — connection established,
   │  identified by (client IP:port, server IP:port)
   ▼
  TLS
   │  TLS handshake: server presents its CA-signed certificate, client verifies it,
   │  both derive a shared symmetric session key — connection is now encrypted (HTTPS)
   ▼
  HTTP
   │  HTTP/2 (or /3) request sent, possibly reusing a pooled/kept-alive connection,
   │  compressed, with conditional headers (If-None-Match) potentially avoiding
   │  a full response entirely via 304
   ▼
Load Balancer  →  Reverse Proxy  →  Application     (Levels 14 & 17)
```

**The one-sentence takeaway:** every layer in this stack exists to solve exactly one problem —
DNS finds the address, TCP guarantees reliable ordered delivery, TLS guarantees privacy and
authenticity, and HTTP defines the actual request/response semantics your application logic
finally gets to work with — and nearly every "weird" networking behavior you'll debug in
production traces back to one specific layer in this exact chain misbehaving.
