# The Complete Journey of an API Call Through the OSI Layers

A backend developer's walkthrough of what actually happens between clicking "Send" and receiving a response — mapped to every OSI layer, plus what happens inside the backend framework.

---

## Example Request

```
GET https://api.example.com/users/1 HTTP/1.1
Host: api.example.com
Authorization: Bearer xxxxx
```

---

## Step 1 — Application Layer (Layer 7)

The client application builds the HTTP request. Nothing is sent over the network yet.

```
Browser / React / Flutter / Postman
        │
        ▼
HTTP Request
GET /users/1
Host: api.example.com
Authorization: Bearer token
```

## Step 2 — Presentation Layer (Layer 6)

If the connection uses HTTPS, TLS encrypts the data.

```
HTTP
 │
 ▼
TLS Encrypts Data
```

What happens here:
- TLS handshake (only on first connection)
- Certificate verification
- Session key generation
- Encryption

Result: **Encrypted HTTP Data**

## Step 3 — Session Layer (Layer 5)

Manages the ongoing communication session.

```
Client
  │
Create Session
  │
Maintain Session
```

Examples: TLS session, HTTP Keep-Alive, WebSocket session.

## Step 4 — Transport Layer (Layer 4)

TCP establishes a reliable connection before any data is sent.

**TCP Three-Way Handshake**

```
Client                     Server

SYN --------------------->

       <------------------ SYN + ACK

ACK --------------------->

Connection established.
```

The HTTP request is then broken into TCP segments:

```
+----------------------+
| Source Port          |
| Destination Port     |
| Sequence Number      |
| ACK Number           |
| Flags                |
| HTTP Data            |
+----------------------+
```

Example: Source Port `54891` → Destination Port `443`

## Step 5 — Network Layer (Layer 3)

IP routing wraps the TCP segment into an IP packet.

```
TCP Segment
     │
     ▼
IP Packet

+-----------------------+
| Source IP             |
| Destination IP        |
| TTL                   |
| Protocol = TCP        |
| TCP Segment           |
+-----------------------+
```

Example: Source IP `192.168.1.10` → Destination IP `142.250.xxx.xxx`

If the destination isn't on the local network:

```
Packet → Router → ISP → Internet → Destination Router → Server
```

Routers make forwarding decisions based on their routing tables.

## Step 6 — Data Link Layer (Layer 2)

IP doesn't know the destination MAC address, so ARP resolves it first:

```
"Who has 192.168.1.1?"
Router replies: "My MAC is AA:BB:CC:11:22:33"
```

Ethernet then builds a frame:

```
+-----------------------+
| Destination MAC       |
| Source MAC            |
| IP Packet             |
| CRC                   |
+-----------------------+
```

A switch forwards the frame based on the destination MAC address.

## Step 7 — Physical Layer (Layer 1)

The frame becomes electrical, optical, or radio signals.

```
101010101010100101010
        │
        ▼
Cable → WiFi → Fiber
```

Bits travel across the physical medium.

---

## What Happens on the Internet

The packet passes through many devices on its way to the server:

```
Client
  ↓
Home Router
  ↓
ISP
  ↓
Regional Router
  ↓
Internet Backbone
  ↓
Cloud Provider
  ↓
Load Balancer
  ↓
Firewall
  ↓
Web Server
```

At every router along the way:

```
Receive Packet → Read Destination IP → Find Next Hop → Forward Packet
```

**Key insight**: the IP address stays the same end-to-end (aside from NAT), but the Layer 2 MAC addresses change on every single network hop.

---

## Server Receives the Request (Decapsulation)

The server processes the packet in reverse order — this is called **decapsulation**:

```
Bits → Frame → Packet → TCP Segment → TLS → HTTP
```

---

## Your Backend Framework Takes Over

Once HTTP reaches the server, it flows through the web server and application stack:

```
HTTP Request
  ↓
Nginx
  ↓
Gunicorn / Tomcat / Node
  ↓
Framework
  ↓
Router
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Database
```

### Example: `GET /users/1`

```
GET /users/1
  ↓
UserController
  ↓
UserService
  ↓
UserRepository
  ↓
MySQL
  ↓
SELECT * FROM users WHERE id=1
```

### Database Response

```
MySQL → Result → Repository → Service → Controller → JSON
```

Response body:

```json
{
  "id": 1,
  "name": "Mizan"
}
```

---

## Response Travels Back (Encapsulation)

The response goes back down through the same layers, each adding its own header:

```
Application → Presentation → Session → Transport → Network → Data Link → Physical
```

## Client Receives the Response (Decapsulation, again)

```
Bits → Frame → Packet → TCP Segment → TLS Decryption → HTTP → Browser / Flutter → JSON → UI Updates
```

---

## Complete End-to-End Flow

```
                    CLIENT                                  SERVER

Flutter/Postman
      │
      ▼
Create HTTP Request
      │
      ▼
TLS Encrypt (HTTPS)
      │
      ▼
TCP Segment (Port 443)
      │
      ▼
IP Packet (Destination IP)
      │
      ▼
Ethernet Frame (Destination MAC)
      │
      ▼
Electrical/WiFi Signals
      │
      ▼
──────────── Internet (Routers, Switches, ISP) ────────────
      │
      ▼
Server NIC Receives Bits
      │
      ▼
Frame → Packet → Segment
      │
      ▼
TLS Decrypt
      │
      ▼
HTTP Request
      │
      ▼
Nginx / Apache
      │
      ▼
Backend Framework (Django/Spring Boot)
      │
      ▼
Controller
      │
      ▼
Service
      │
      ▼
Repository
      │
      ▼
Database
      │
      ▼
JSON Response
      │
      ▼
Same Layers Back
      │
      ▼
Flutter/Postman Displays Response
```

---

## Recommended Learning Order for Backend Developers

To build a complete mental model of what happens after an API call, study these topics in order:

1. **HTTP/HTTPS** — methods, headers, status codes
2. **DNS** — how a domain becomes an IP address
3. **TCP** — three-way handshake, reliability, retransmission
4. **TLS/SSL** — certificate validation and encryption
5. **IP routing** — how packets reach the server
6. **Ethernet, ARP, and MAC addresses**
7. **Web servers** — Nginx / Apache
8. **Reverse proxies and load balancers**
9. **Your backend framework's request lifecycle** — Spring Boot, Django, Express, etc.
10. **Database communication and response generation**

Mastering these gives you a complete mental model of everything that happens from the moment an API is called until the response is displayed.
