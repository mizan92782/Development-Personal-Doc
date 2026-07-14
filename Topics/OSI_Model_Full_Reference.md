# OSI Model — Full Reference (Top → Bottom)

A complete, memorizable reference for the 7-layer OSI Model, aimed at backend developers.

---

## Layer 7 — Application

| Field | Details |
|---|---|
| Purpose | User interacts with network services |
| Protocols | HTTP, HTTPS, FTP, SFTP, SMTP, POP3, IMAP, DNS, DHCP, SSH, Telnet, SNMP |
| Data Unit | Data |
| Devices | Proxy Server, Gateway, Web Server |
| Examples | Browser, Email, REST API, Mobile App |

## Layer 6 — Presentation

| Field | Details |
|---|---|
| Purpose | Translation, Encryption, Compression |
| Protocols | SSL/TLS, JPEG, PNG, GIF, MPEG, ASCII, Unicode |
| Data Unit | Data |
| Devices | SSL Offloader, Encryption Gateway |
| Examples | JSON, XML, UTF-8, Base64, AES Encryption |

## Layer 5 — Session

| Field | Details |
|---|---|
| Purpose | Create, Maintain, Terminate Sessions |
| Protocols | NetBIOS, RPC, PPTP, SMB Session |
| Data Unit | Data |
| Devices | Gateway |
| Examples | Login Session, Database Session |

## Layer 4 — Transport

| Field | Details |
|---|---|
| Purpose | Reliable communication, Error recovery |
| Protocols | TCP, UDP, SCTP |
| Data Unit | Segment (TCP), Datagram (UDP) |
| Devices | Firewall, Load Balancer |
| Port Numbers | 80, 443, 22, 3306, 5432, etc. |

## Layer 3 — Network

| Field | Details |
|---|---|
| Purpose | Logical Addressing & Routing |
| Protocols | IPv4, IPv6, ICMP, IPSec, OSPF, RIP, BGP |
| Data Unit | Packet |
| Address | IP Address |
| Devices | Router, Layer-3 Switch |

## Layer 2 — Data Link

| Field | Details |
|---|---|
| Purpose | MAC Addressing, Error Detection |
| Protocols | Ethernet (802.3), Wi-Fi (802.11), PPP, ARP, VLAN (802.1Q), STP |
| Data Unit | Frame |
| Address | MAC Address |
| Devices | Switch, Bridge, Wireless AP |

## Layer 1 — Physical

| Field | Details |
|---|---|
| Purpose | Transfer Bits |
| Protocols/Standards | Ethernet Cable, Fiber, USB, Bluetooth, DSL |
| Data Unit | Bits |
| Devices | Hub, Repeater, Cable, Connector |
| Media | UTP, Fiber Optic, Coaxial Cable |

---

## Data Encapsulation (Sender Side)

```
Application  → Data
      │
      ▼
Presentation → Data
      │
      ▼
Session      → Data
      │
      ▼
Transport    → Segment
      │
      ▼
Network      → Packet
      │
      ▼
Data Link    → Frame
      │
      ▼
Physical     → Bits
```

The receiver reverses this process — **de-encapsulation** — stripping each layer's header as the data moves back up from Bits to Application Data.

---

## Device Mapping

| Layer | Typical Device |
|---|---|
| Application | Proxy Server |
| Presentation | SSL/TLS Gateway |
| Session | Gateway |
| Transport | Firewall, Load Balancer |
| Network | Router, Layer-3 Switch |
| Data Link | Switch, Bridge, Access Point |
| Physical | Hub, Repeater, Cable |

---

## Address Used Per Layer

| Layer | Address |
|---|---|
| 7–5 | No network address |
| 4 | Port Number |
| 3 | IP Address |
| 2 | MAC Address |
| 1 | Electrical/Optical Signals |

---

## PDU (Protocol Data Unit) Summary

| Layer | PDU Name |
|---|---|
| 7 — Application | Data |
| 6 — Presentation | Data |
| 5 — Session | Data |
| 4 — Transport | Segment (TCP) / Datagram (UDP) |
| 3 — Network | Packet |
| 2 — Data Link | Frame |
| 1 — Physical | Bits |

---

## Backend Developer View — Request Path

```
Client
   │
HTTP/HTTPS
   │
TLS
   │
TCP (Port 443)
   │
IP
   │
Ethernet
   │
Cable/WiFi
   │
Internet
   │
Server
```

### Example: `GET /api/users`

When this request is made, it travels through the layers as:

| Layer | Protocol Used |
|---|---|
| Application | HTTP |
| Presentation | TLS |
| Session | Session Management |
| Transport | TCP |
| Network | IP |
| Data Link | Ethernet |
| Physical | Cable/WiFi |

The server processes the request and sends the response back through the **same layers in reverse**.

---

## Easy Mnemonics

**Layer 7 → Layer 1 (top to bottom):**

> **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

- A = Application
- P = Presentation
- S = Session
- T = Transport
- N = Network
- D = Data Link
- P = Physical

**Layer 1 → Layer 7 (bottom to top):**

> **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

- P = Physical
- D = Data Link
- N = Network
- T = Transport
- S = Session
- P = Presentation
- A = Application
