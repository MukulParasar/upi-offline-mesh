# UPI Offline Mesh

> **Mesh-routed deferred payment settlement over untrusted intermediaries** — a Spring Boot backend that demonstrates how UPI-style payments can survive zero-connectivity environments using Bluetooth-style gossip, hybrid encryption, and atomic idempotency.

**Live demo:** [upi-offline-mesh-production-5c2a.up.railway.app](https://upi-offline-mesh-production-5c2a.up.railway.app/) &nbsp;|&nbsp; **Author:** Mukul &nbsp;|&nbsp; **Stack:** Java 17 · Spring Boot 3.3 · H2 · AES-GCM · RSA-OAEP

---

## Why I built this

I got curious about how UPI Lite handles offline payments — specifically, what happens when there's no internet anywhere in the chain. Studied an existing open codebase (with the author's permission) to deeply understand the hard problems: untrusted relays, duplicate storms, and replay attacks. Every design decision in this system is documented and understood.

---

## What this demo proves

Three things work end to end:

1. **A payment travels through untrusted intermediaries unread and untampered.** Hybrid RSA-OAEP + AES-GCM encryption means relay phones see only opaque ciphertext. If they flip one bit, the GCM auth tag fails and the backend rejects the packet.
2. **The same payment settles exactly once, even under concurrent delivery.** Atomic `ConcurrentHashMap.putIfAbsent` on the ciphertext hash deduplicates before any decryption or DB work. The concurrency test proves this with three simultaneous threads.
3. **Replayed packets are rejected.** A `signedAt` timestamp inside the encrypted payload (which an attacker cannot change without breaking the GCM tag) expires packets after 24 hours.

---

## How to run it

**Prerequisites:** JDK 17+ on PATH. That's it — no database, no Redis, no Maven install.

```bash
# Windows
mvnw.cmd spring-boot:run

# Mac / Linux
./mvnw spring-boot:run
```

First run downloads Maven + dependencies (~90 MB, ~2 min). Subsequent starts take ~5 seconds.

Open **http://localhost:8080** once you see `Started UpiMeshApplication`.

```bash
# Run tests
mvnw.cmd test

# Run the headline concurrency test specifically
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

---

## The demo flow

The dashboard walks through four steps:

| Step | Action | What happens internally |
|---|---|---|
| 1 | **Inject into Mesh** | Server builds `PaymentInstruction` with nonce + timestamp, hybrid-encrypts it, wraps in `MeshPacket` (TTL 5), hands to `phone-alice` |
| 2 | **Run Gossip Round** | Every device broadcasts its packets to every other device in range; TTL decrements per hop |
| 3 | **Bridges Upload** | `phone-bridge` (the only device with `hasInternet=true`) POSTs all held packets to `/api/bridge/ingest` in parallel |
| 4 | **Reset** | Clears mesh state and idempotency cache for a clean run |

Watch **Account Balances** and **Transaction Ledger** update live after Step 3.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ encrypt with server's RSA public key                     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND (this repo)                     │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── atomic putIfAbsent (≈ Redis    │
│       │                                  SETNX). Duplicates rejected    │
│       │                                  here, before any work.         │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       RSA-OAEP unwraps AES key; AES-GCM decrypts payload        │
│       │       AND verifies auth tag — one flipped bit = exception       │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within last 24 h                         │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger row   │
│       @Version on Account = optimistic locking (defense in depth)       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The three hard problems

### 1 — Untrusted intermediaries

A stranger's phone carries your transaction. They can't read it or change it because:

- **Hybrid RSA-OAEP + AES-GCM.** RSA can only encrypt ~245 bytes (2048-bit key), so the actual payload uses AES-256-GCM (fast, authenticated). Only the AES key is RSA-encrypted.
- **Wire format:** `[256 B RSA-wrapped AES key][12 B IV][AES-GCM ciphertext + 16 B auth tag]`
- **GCM auth tag:** any bit flip anywhere in the ciphertext throws an exception on decrypt. The server can't be tricked into processing tampered data.

This is the same scheme TLS uses. See `HybridCryptoService.java`.

### 2 — The duplicate storm

Three bridges upload the same packet within milliseconds. Naively processing all three debits the sender three times.

**Fix: atomic compare-and-set on the ciphertext hash.**

```java
// IdempotencyService.java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null; // true = first claimer only
```

`ConcurrentHashMap.putIfAbsent` is atomic — exactly one thread wins, the rest short-circuit as `DUPLICATE_DROPPED` before any decryption or DB work.

**Why hash the ciphertext, not the `packetId`?** `packetId` is in the unencrypted outer wrapper — a malicious relay can rewrite it, making two copies of the same payment appear different. The ciphertext is authenticated by GCM; two legitimate deliveries of the same packet are byte-identical.

Defense-in-depth: `transactions.packet_hash` also has a unique DB index. If the cache layer ever fails, the database rejects the second write.

In production: `ConcurrentHashMap` → `Redis SET key NX EX 86400`. Same semantics, distributed.

### 3 — Replay attacks

**Fix: two layers inside the encrypted payload** (which an attacker cannot modify without breaking the GCM tag):

- **`signedAt`** — epoch millis. Server rejects packets older than 24 hours.
- **`nonce`** — UUID per payment. Two legitimate ₹100 payments from Alice to Bob have different nonces → different ciphertexts → different hashes → both settle correctly. A replay of one specific packet is byte-identical → idempotency cache catches it.

---

## File-by-file walkthrough

```
upi-offline-mesh/
├── pom.xml                                  Maven build — Spring Boot 3.3, Java 17
├── mvnw, mvnw.cmd                           Maven wrapper (no install needed)
└── src/main/
    ├── resources/
    │   ├── application.properties           H2 in-memory DB, port 8080, TTLs
    │   └── templates/dashboard.html         Interactive demo UI
    └── java/com/demo/upimesh/
        ├── UpiMeshApplication.java          Entry point
        ├── model/
        │   ├── Account.java                 JPA entity; @Version = optimistic lock
        │   ├── AccountRepository.java       Spring Data JPA
        │   ├── Transaction.java             Ledger; unique index on packetHash
        │   ├── TransactionRepository.java   Spring Data JPA
        │   ├── MeshPacket.java              Wire format (outer fields readable, ciphertext opaque)
        │   └── PaymentInstruction.java      Decrypted payload
        ├── crypto/
        │   ├── ServerKeyHolder.java         Generates RSA-2048 keypair on startup
        │   └── HybridCryptoService.java     RSA-OAEP + AES-256-GCM + ciphertext hash
        ├── service/
        │   ├── DemoService.java             Seeds accounts; simulates sender phone
        │   ├── VirtualDevice.java           One simulated phone
        │   ├── MeshSimulatorService.java    Gossip protocol
        │   ├── IdempotencyService.java      ConcurrentHashMap ≈ Redis SETNX
        │   ├── SettlementService.java       @Transactional debit + credit + ledger
        │   └── BridgeIngestionService.java  THE pipeline: hash → claim → decrypt → freshness → settle
        ├── controller/
        │   ├── ApiController.java           All REST endpoints
        │   └── DashboardController.java     Serves dashboard at /
        └── config/
            └── AppConfig.java               @EnableScheduling for cache eviction

src/test/java/com/demo/upimesh/
└── IdempotencyConcurrencyTest.java          3-bridges-at-once + tamper test
```

---

## API reference

| Method | Path | Description |
|---|---|---|
| GET | `/` | Dashboard HTML |
| GET | `/api/server-key` | RSA public key (base64) |
| GET | `/api/accounts` | All accounts and balances |
| GET | `/api/transactions` | Last 20 transactions |
| GET | `/api/mesh/state` | State of every virtual device |
| POST | `/api/demo/send` | Simulate sender — encrypt + inject packet |
| POST | `/api/mesh/gossip` | One gossip round |
| POST | `/api/mesh/flush` | Bridges upload to backend (parallel) |
| POST | `/api/mesh/reset` | Clear mesh + idempotency cache |
| POST | `/api/bridge/ingest` | **Production endpoint.** Real bridges POST here. |
| GET | `/h2-console` | In-memory DB browser |

H2 login: URL `jdbc:h2:mem:upimesh` · user `sa` · no password.

**`POST /api/bridge/ingest` — request:**
```http
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-RSA-and-AES-blob"
}
```

**Response:**
```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

`outcome` is one of `SETTLED` · `DUPLICATE_DROPPED` · `INVALID`.

---

## Tests

```bash
mvnw.cmd test
```

| Test | What it checks |
|---|---|
| `encryptDecryptRoundTrip` | Hybrid encryption is symmetric |
| `tamperedCiphertextIsRejected` | One flipped byte → `INVALID`, no settlement |
| `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` | Three concurrent threads, one packet → exactly 1 `SETTLED`, 2 `DUPLICATE_DROPPED`, sender debited once |

---

## What's NOT real (production swap table)

| Demo | Production |
|---|---|
| H2 in-memory DB | PostgreSQL / MySQL with replicas |
| `ConcurrentHashMap` idempotency | Redis `SET key NX EX 86400` |
| RSA keypair regenerated on startup | Private key in HSM (AWS KMS / HashiCorp Vault) |
| Server-side `DemoService.createPacket()` | Same logic on Android (Kotlin port) |
| `MeshSimulatorService` | Real BLE GATT or Wi-Fi Direct |
| Single settlement service | Integration with NPCI / bank core |
| No auth on `/api/bridge/ingest` | Mutual TLS or signed bridge-node certificates |
| In-memory seeded accounts | Real KYC'd users, real VPAs, real PIN verification |
| H2 console exposed | Disabled |
| No rate limiting | Per-bridge-node rate limit + per-sender velocity check |
| Console logs | Structured logs to SIEM, alerts on `INVALID` spikes |

The cryptography and idempotency code is essentially production-shaped. The infrastructure around it is what changes.

---

## Honest limitations of the concept

These are not implementation bugs — they're inherent to "no internet anywhere in the chain":

1. **No offline proof of funds.** When the sender shows "₹500 sent," it's an IOU. If the account is empty when the packet reaches the backend, the settlement is `REJECTED` and the receiver has no recourse. Real UPI Lite solves this with a pre-funded hardware-backed wallet that gives cryptographic proof of balance offline.

2. **Offline double-spend is possible.** A sender with ₹500 can inject a packet to Bob in basement A, walk to basement B, inject another to Carol. Whichever reaches the backend first wins; the other is `REJECTED`. Same root cause as #1.

3. **Real Bluetooth is hard.** Background BLE on Android is throttled since Android 8; iOS peripheral mode is restricted. Reliable connections between strangers' phones without active apps is genuinely difficult. This demo skips the problem entirely by simulating the mesh in software.

4. **Privacy / metadata.** A stranger can't read your transaction, but they know they're carrying one. In production this would require regulatory disclosure and clarity on device-seizure liability.

Calling this **"mesh-routed deferred settlement"** rather than "real-time offline UPI" is the more accurate and defensible pitch.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `java: command not found` | Install JDK 17+. Windows: `winget install EclipseAdoptium.Temurin.17.JDK` |
| Port 8080 in use | Change `server.port` in `application.properties` |
| First run hangs | Downloading Maven + deps (~90 MB). Wait 2–3 min; subsequent starts are ~5 s |
| `mvnw.cmd not recognized` on PowerShell | Prefix with `.\` → `.\mvnw.cmd spring-boot:run` |
| Concurrency test flakes | Timing-sensitive. Run 3× — consistent failure means file an issue with the output |

---

## Acknowledgements
Original codebase used with the author's permission. 
Studied, documented, and extended with UI improvements by me.

---

*MIT License.*
