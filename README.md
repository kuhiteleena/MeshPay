
# MeshPay — Offline UPI Mesh Routing & Deferred Settlement Simulator

MeshPay is a Java 17 and Spring Boot project that demonstrates offline payment packet routing through a simulated Bluetooth-style mesh network.

Imagine being in a basement with no internet connectivity. You want to send your friend ₹500. Your phone encrypts the payment instruction and passes the packet through nearby devices. When a device with internet access becomes available, it uploads the packet to the backend for processing and settlement.

This repository contains the Spring Boot backend and a software simulator of the mesh network, allowing the complete flow to be demonstrated on a single laptop without real Bluetooth hardware.

> **Note:** MeshPay is an educational demonstration. It simulates payment processing and does not integrate with NPCI, a bank, or real UPI transactions.

## Table of Contents

- [What This Project Demonstrates](#what-this-project-demonstrates)
- [Technology Stack](#technology-stack)
- [Getting Started](#getting-started)
- [Demo Flow](#demo-flow)
- [Architecture](#architecture)
- [The Three Challenges and Their Solutions](#the-three-challenges-and-their-solutions)
- [Project Structure](#project-structure)
- [API Reference](#api-reference)
- [Tests](#tests)
- [Implementation Details and Limitations](#implementation-details-and-limitations)
- [License](#license)

## What This Project Demonstrates

The project demonstrates three challenges and their corresponding solutions:

1. **Untrusted intermediaries:** Hybrid RSA and AES-GCM encryption protects the payment payload while it travels through intermediate devices. Authenticated encryption detects ciphertext tampering.
2. **Duplicate transaction delivery:** Atomic idempotency based on the ciphertext hash prevents the same packet from being processed more than once in the demonstrated application.
3. **Replay attacks:** Timestamp freshness validation and unique nonces help mitigate repeated submission of previously captured payment instructions.

The dashboard lets you observe packet propagation, account balances, transaction history, and packet-processing outcomes.

## Technology Stack

| Component | Technology |
|---|---|
| Programming Language | Java 17 |
| Backend Framework | Spring Boot 3.3 |
| Build Tool | Maven Wrapper |
| Persistence | Spring Data JPA |
| Database | H2 In-Memory Database |
| Encryption | RSA-OAEP and AES-256-GCM |
| Hashing | SHA-256 |
| Idempotency | Java ConcurrentHashMap |
| Transaction Management | Spring `@Transactional` |
| Concurrency Protection | Atomic operations and optimistic locking |
| Dashboard | HTML and JavaScript |
| Testing | JUnit-based automated tests |

## Getting Started

### Prerequisites

- JDK 17 or newer
- Internet connectivity for the initial Maven dependency download

Maven does not need to be installed separately because the project includes the Maven Wrapper.

### Run on Windows

Open a terminal in the project directory and execute:

```powershell
.\mvnw.cmd spring-boot:run
```

### Run on macOS/Linux

```bash
./mvnw spring-boot:run
```

### Open the Dashboard

After the application starts, open:

http://localhost:8080

Use the dashboard to create a simulated payment, run mesh gossip rounds, upload packets through bridge devices, and inspect transaction results.

### Stop the Application

Press `Ctrl+C` in the terminal.

## Demo Flow

### Step 1: Compose a Payment

Choose the sender, receiver, amount, and PIN, then click **Inject into Mesh**.

The backend:

1. Creates a `PaymentInstruction`.
2. Generates a unique nonce and timestamp.
3. Encrypts the payment instruction.
4. Wraps the ciphertext in a `MeshPacket` with a TTL of 5.
5. Injects the packet into the simulated mesh through a virtual device.

### Step 2: Run Gossip Rounds

Click **Run Gossip Round** to simulate packet propagation between virtual devices.

Each device holding a packet broadcasts it to the other devices in the simulator. The TTL decreases as the packet travels through the network.

This is a software simulation, not actual Bluetooth communication.

### Step 3: Upload Through a Bridge

Click **Bridges Upload to Backend**.

The simulated bridge device represents a phone that has regained internet access. It uploads the packets it holds to:

`POST /api/bridge/ingest`

The backend then:

1. Calculates the SHA-256 hash of the ciphertext.
2. Attempts to claim the hash in the idempotency cache.
3. Rejects duplicate ciphertext hashes.
4. Decrypts the payment instruction.
5. Checks timestamp freshness.
6. Processes the settlement through the settlement service.

The dashboard displays the resulting account balances and transaction ledger.

### Step 4: Demonstrate Idempotency

The project includes a concurrency test that submits the same packet to the ingestion service from three threads simultaneously.

The expected test outcome is:

- One packet is settled.
- Two duplicate deliveries are dropped.
- The sender's balance is debited only once.

Run the dedicated test with:

```powershell
.\mvnw.cmd test "-Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce"
```

The default dashboard configuration seeds one bridge device. The concurrency test directly exercises simultaneous duplicate delivery.

## Architecture

```text
                 SENDER DEVICE
                       |
                       v
              Payment Instruction
                       |
                       v
              Hybrid Encryption
                RSA-OAEP + AES-GCM
                       |
                       v
                  Mesh Packet
                       |
                       v
              Simulated Mesh Network
                 /      |      \
                v       v       v
             Device   Device   Bridge
                                 |
                                 v
                          Internet Access
                                 |
                                 v
                      POST /api/bridge/ingest
                                 |
                                 v
                         SHA-256 Hashing
                                 |
                                 v
                       Idempotency Claim
                                 |
                                 v
                         Decrypt Payload
                                 |
                                 v
                       Freshness Validation
                                 |
                                 v
                       Settlement Service
                                 |
                                 v
                    Transactional Debit/Credit
                                 |
                                 v
                       Transaction Ledger
```

### Main Components

- **Mesh simulation:** Virtual devices and gossip rounds model packet forwarding.
- **Cryptography:** Encrypts and decrypts payment instructions.
- **Bridge ingestion:** Receives uploaded packets and coordinates packet processing.
- **Idempotency service:** Prevents repeated processing of the same ciphertext hash within the demonstrated application.
- **Settlement service:** Performs simulated debit and credit operations within a database transaction.
- **Persistence:** Stores simulated accounts and transactions in H2.
- **Dashboard and REST API:** Provide controls for demonstrating the payment flow.

## The Three Challenges and Their Solutions

### Challenge 1: Untrusted Intermediaries

**Problem**

Payment packets travel through intermediate devices. These devices should not be able to read or silently modify the encrypted payment payload.

**Solution: Hybrid Encryption Using RSA-OAEP and AES-256-GCM**

MeshPay uses hybrid encryption:

1. Generate a fresh AES-256 key for the packet.
2. Encrypt the payment JSON using AES-256-GCM.
3. Encrypt the AES key using the server's RSA public key with RSA-OAEP.
4. Combine the encrypted AES key, initialization vector, and encrypted payload into the ciphertext representation.

The server uses its RSA private key to recover the AES key and decrypt the payload.

AES-GCM authenticates the encrypted payload. If the ciphertext is modified, authentication fails during decryption.

**Implementation:** `HybridCryptoService.java`

### Challenge 2: Duplicate Transaction Delivery

**Problem**

Multiple bridge devices could upload copies of the same payment packet at nearly the same time. Processing every copy could debit the sender more than once.

**Solution: Atomic Idempotency Using the Ciphertext Hash**

The backend calculates the SHA-256 hash of the ciphertext and attempts to claim it using an atomic operation:

```java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;
```

The first successful claim proceeds through the ingestion pipeline. Subsequent claims for the same hash are rejected as duplicates.

The transaction table also has a unique constraint on `transactions.packet_hash` as an additional safeguard.

The current implementation uses `ConcurrentHashMap`. Redis with `SET NX EX` is described as a possible production replacement but is not implemented in this project.

**Implementation:** `IdempotencyService.java`

### Challenge 3: Replay Attacks

**Problem**

An attacker might submit a previously captured payment packet again at a later time.

**Solution: Timestamp Freshness Validation and Unique Nonces**

The encrypted payment instruction contains a timestamp and a unique nonce.

The backend rejects payment instructions whose `signedAt` timestamp falls outside the configured 24-hour freshness window. The timestamp is protected by the authenticated encrypted payload, so changing it without valid encryption causes authentication to fail.

The nonce distinguishes separately created payment instructions. Repeated delivery of the same packet is also handled by ciphertext-hash idempotency.

These mechanisms mitigate the replay scenario demonstrated by the project. They do not make the entire offline payment system immune to every replay or double-spending scenario.

**Implementation:** `BridgeIngestionService.java`

## Project Structure

```text
MeshPay/
├── .mvn/
│   └── wrapper/
├── src/
│   ├── main/
│   │   ├── java/com/demo/upimesh/
│   │   │   ├── UpiMeshApplication.java
│   │   │   ├── model/
│   │   │   │   ├── Account.java
│   │   │   │   ├── AccountRepository.java
│   │   │   │   ├── Transaction.java
│   │   │   │   ├── TransactionRepository.java
│   │   │   │   ├── MeshPacket.java
│   │   │   │   └── PaymentInstruction.java
│   │   │   ├── crypto/
│   │   │   │   ├── ServerKeyHolder.java
│   │   │   │   └── HybridCryptoService.java
│   │   │   ├── service/
│   │   │   │   ├── DemoService.java
│   │   │   │   ├── VirtualDevice.java
│   │   │   │   ├── MeshSimulatorService.java
│   │   │   │   ├── IdempotencyService.java
│   │   │   │   ├── SettlementService.java
│   │   │   │   └── BridgeIngestionService.java
│   │   │   ├── controller/
│   │   │   │   ├── ApiController.java
│   │   │   │   └── DashboardController.java
│   │   │   └── config/
│   │   │       └── AppConfig.java
│   │   └── resources/
│   │       ├── application.properties
│   │       └── templates/dashboard.html
│   └── test/
│       └── java/com/demo/upimesh/
│           └── IdempotencyConcurrencyTest.java
├── pom.xml
├── mvnw
├── mvnw.cmd
└── README.md
```

## API Reference

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Display the dashboard |
| GET | `/api/server-key` | Return the server's RSA public key |
| GET | `/api/accounts` | Retrieve simulated account balances |
| GET | `/api/transactions` | Retrieve the latest 20 transactions |
| GET | `/api/mesh/state` | Inspect virtual-device and mesh state |
| POST | `/api/demo/send` | Create and inject a simulated payment |
| POST | `/api/mesh/gossip` | Run a mesh gossip round |
| POST | `/api/mesh/flush` | Upload packets from bridge devices |
| POST | `/api/mesh/reset` | Reset the mesh and idempotency cache |
| POST | `/api/bridge/ingest` | Receive and process an uploaded mesh packet |
| GET | `/h2-console` | Access the development H2 console |

### Bridge Ingestion Request

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
```

Example request body:

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-RSA-and-AES-blob"
}
```

Example response:

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Possible outcomes include `SETTLED`, `DUPLICATE_DROPPED`, and `INVALID`.

The examples illustrate the data format. Actual values and outcomes depend on the request and application state.

### H2 Console

The development H2 console uses:

- JDBC URL: `jdbc:h2:mem:upimesh`
- Username: `sa`
- Password: empty

## Tests

Run the complete test suite:

```powershell
.\mvnw.cmd test
```

The documented tests cover:

### 1. Encryption/decryption round trip

`encryptDecryptRoundTrip` checks that a payment payload can be encrypted and subsequently decrypted.

### 2. Tampered ciphertext

`tamperedCiphertextIsRejected` modifies a ciphertext byte and checks that the ingestion service rejects the invalid packet rather than settling it.

### 3. Concurrent duplicate delivery

`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` submits the same packet through three concurrent threads and verifies the expected single settlement and duplicate rejection.

These tests cover the documented scenarios. They are not a complete security audit or proof of production readiness.

## Implementation Details and Limitations

MeshPay is a teaching and demonstration project.

| Current implementation | Production alternative described in the original design |
|---|---|
| H2 in-memory database | PostgreSQL or MySQL with replicas |
| `ConcurrentHashMap` idempotency cache | Redis using `SET NX EX` |
| RSA key pair generated at startup | Private-key management using an HSM, AWS KMS, or HashiCorp Vault |
| Server-side simulated payment creation | Payment creation running on an Android client |
| Software-simulated mesh | Real Bluetooth Low Energy GATT or Wi-Fi Direct communication |
| Simulated settlement service | Integration with an appropriate bank payment system |
| No bridge authentication on ingestion | Mutual TLS or signed bridge-node credentials |
| Seeded simulated accounts | Verified users and real payment authorization |
| Development H2 console | Disabled in production |
| No rate limiting | Bridge and sender rate limits |
| Console logging | Structured logging, monitoring, and alerts |

These production alternatives are not implemented features of MeshPay.

### Offline Double-Spending

The receiver cannot know whether the sender has sufficient funds when a payment packet is created offline.

A sender could create different payment instructions against the same balance. When the packets eventually reach the backend, one payment may be rejected because the available balance has already been used.

Packet-level idempotency prevents repeated processing of the same packet; it does not prevent a sender from creating different payments against the same funds.

### Simulated Device Networking

Actual Bluetooth communication between phones is not implemented. The software simulator does not reproduce all the practical constraints of background Bluetooth operation on Android or iOS.

### Privacy Considerations

Intermediate devices cannot read the encrypted payment payload, but carrying a packet can still expose metadata about the existence of a transaction.

### Production Readiness

The project does not implement the complete identity verification, payment authorization, infrastructure security, or regulatory integration required for a real financial service.

For these reasons, MeshPay is best described as a **mesh-routed deferred-settlement simulation**, rather than a complete real-time offline UPI payment system.

## License

The original repository does not specify a license. Check the repository's licensing terms before redistributing or reusing its code.

---

**MeshPay — Offline UPI Mesh Routing & Deferred Settlement Simulator**

A demonstration of encrypted packet routing, atomic idempotency, and replay mitigation in a simulated offline payment environment.
