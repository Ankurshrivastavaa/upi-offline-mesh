# UPI Offline Mesh

> **Mesh-Routed Deferred Payment Settlement**

A Java 17 + Spring Boot simulation of an offline payment system where payment packets propagate through a device mesh and are later uploaded by bridge devices for secure, idempotent settlement.

**Live Demo:** https://upi-offline-mesh-nvgy.onrender.com

**GitHub:** https://github.com/Ankurshrivastavaa/upi-offline-mesh

---

## 📌 Project Overview

In areas with unreliable or unavailable internet connectivity, a payment may not be immediately deliverable to a backend server.

This project demonstrates a **mesh-routed deferred settlement model**:

1. A payment is created on an offline device.
2. The payment is encrypted into a packet.
3. The packet propagates between simulated devices.
4. Bridge devices eventually obtain the packet.
5. A bridge uploads the packet to the backend.
6. The backend validates, decrypts, deduplicates, and settles the payment.

The project focuses on the backend engineering problems involved in securely processing the same payment packet when it may arrive through multiple paths.

> **Important:** This is a software simulation. It does not integrate with real UPI, NPCI, banks, Bluetooth hardware, or real payment infrastructure.

---

## 🎯 Engineering Problems Solved

The project focuses on four main backend problems:

### 1. Secure Payment Transmission

Payment information must remain protected while packets travel through untrusted intermediary devices.

### 2. Duplicate Delivery

The same packet may reach the backend through multiple bridge devices.

The backend must prevent the same payment from being settled more than once.

### 3. Packet Tampering

An intermediary should not be able to modify the encrypted payment data without detection.

### 4. Replay / Stale Packets

Old packets should not remain valid indefinitely.

---

# 🏗️ Architecture

```text
                    OFFLINE MESH
                         │
                         ▼
              ┌─────────────────────┐
              │   Sender Device     │
              │    phone-alice      │
              └──────────┬──────────┘
                         │
                    Encrypted Packet
                         │
                         ▼
              ┌─────────────────────┐
              │   Mesh Devices      │
              │                     │
              │ phone-bob           │
              │ phone-carol         │
              │ bridge-1            │
              │ bridge-2            │
              └──────────┬──────────┘
                         │
                    Gossip Forwarding
                         │
                         ▼
              ┌─────────────────────┐
              │   Bridge Devices    │
              └──────────┬──────────┘
                         │
                    Upload Packet
                         │
                         ▼
              ┌─────────────────────┐
              │   Spring Boot API   │
              │                     │
              │ Hash / Idempotency  │
              │ Freshness Check     │
              │ Decryption          │
              │ Settlement          │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │      H2 Ledger      │
              │                     │
              │ Sender Debit        │
              │ Receiver Credit     │
              │ Transaction Record  │
              └─────────────────────┘



              🔄 End-to-End Flow
Payment Created
      │
      ▼
Payment Instruction
      │
      ▼
Hybrid Encryption
(RSA-OAEP + AES-256-GCM)
      │
      ▼
Mesh Packet Created
      │
      ▼
Gossip Propagation
      │
      ▼
Multiple Bridges Receive Packet
      │
      ▼
Bridge Upload
      │
      ▼
SHA-256 Packet Hash
      │
      ▼
Idempotency Check
      │
      ▼
Freshness Validation
      │
      ▼
AES-GCM Decryption
      │
      ▼
Transactional Settlement
      │
      ▼
Sender Debited
Receiver Credited
Transaction Recorded
🔐 Security Architecture

The project uses hybrid encryption to protect payment packets.

Encryption Flow
Payment Data
     │
     ▼
Generate AES Key
     │
     ▼
AES-256-GCM Encryption
     │
     ▼
Encrypt AES Key
using RSA-OAEP
     │
     ▼
Encrypted Payment Packet
RSA-OAEP

RSA is used to securely protect the randomly generated AES key.

AES-256-GCM

AES-GCM encrypts the actual payment data and provides authenticated encryption.

This means that modification of the ciphertext can be detected during decryption.

SHA-256

The encrypted packet is hashed using SHA-256.

Encrypted Packet
       │
       ▼
     SHA-256
       │
       ▼
Packet Hash

The hash is used as an idempotency key for duplicate detection.

♻️ Idempotency & Concurrency

One of the main engineering challenges is handling the same packet arriving through multiple bridges.

For example:

             ┌── Bridge 1 ──┐
             │              │
Sender ──────┼── Bridge 2 ──┼──► Backend
             │              │
             └── Bridge 3 ──┘

All three bridges may upload the same encrypted packet.

The backend calculates a deterministic SHA-256 hash of the packet.

A concurrent idempotency check is then performed using:

putIfAbsent(packetHash, value)

ConcurrentHashMap.putIfAbsent() provides an atomic check-and-insert operation.

Conceptually:

if (processedPackets.putIfAbsent(packetHash, true) != null) {

    // Packet was already processed

    return;
}

// First processing attempt continues

This prevents concurrent duplicate requests from independently reaching the settlement logic.

The database also contains a unique constraint on the packet hash as an additional defense.

🛡️ Replay & Tamper Detection

The backend performs validation before settlement.

Freshness Validation

Packets contain a timestamp.

The backend checks whether the packet is still within the accepted freshness window.

Packet Timestamp
       │
       ▼
Freshness Check
       │
   ┌───┴────┐
   │        │
 Valid    Expired
   │        │
   ▼        ▼
Continue   Reject
Tamper Detection

AES-GCM authentication detects modifications to encrypted packet data.

If the ciphertext has been modified:

Modified Ciphertext
        │
        ▼
AES-GCM Decryption
        │
        ▼
Authentication Failure
        │
        ▼
Packet Rejected
💰 Transaction Settlement

After validation and successful decryption, the payment is settled transactionally.

The settlement process performs:

1. Find sender account
2. Find receiver account
3. Debit sender
4. Credit receiver
5. Create transaction record

The settlement service uses Spring's:

@Transactional

This ensures the settlement operations execute as one database transaction.

If the transaction fails, the database changes can be rolled back instead of leaving a partially completed settlement.

🔒 Optimistic Locking

Account entities use optimistic locking with:

@Version

This helps detect conflicting concurrent updates to account records.

Conceptually:

Transaction A ──► Account Version 1
                       │
Transaction B ──► Account Version 1
                       │
                       ▼
                 Concurrent Update
                       │
                       ▼
               Version Conflict

This adds another layer of protection when multiple settlement operations attempt to update the same account.

🧪 Testing

The project includes tests covering important security and concurrency scenarios.

Encryption / Decryption

Tests that encrypted payment data can successfully be decrypted back into the original data.

Tampered Ciphertext

Tests that modified encrypted data is rejected.

Duplicate Bridge Delivery

Tests that the same payment packet delivered by multiple bridge devices does not result in multiple settlements.

Example scenario:

Bridge 1 ──────┐
Bridge 2 ──────┼──► Same Packet
Bridge 3 ──────┘
                    │
                    ▼
              Idempotency Check
                    │
                    ▼
             Single Settlement
🧩 Demo Workflow

The application includes a simulated mesh environment.

A typical demonstration looks like:

phone-alice
     │
     │ Payment: ₹500
     ▼
Mesh Gossip
     │
     ├────► phone-bob
     │
     ├────► bridge-1
     │
     └────► bridge-2
              │
              ▼
        Backend Ingestion
              │
              ▼
        Security Checks
              │
              ▼
          Settlement

Example result:

SETTLED ₹500 from alice@demo to bob@demo

If the same packet is received again:

Duplicate packet detected
        │
        ▼
Settlement skipped
🛠️ Tech Stack
Category	Technology
Language	Java 17
Framework	Spring Boot 3.3
Build Tool	Maven
Web	Spring Web
Database	H2
ORM	Spring Data JPA / Hibernate
Encryption	RSA-OAEP + AES-256-GCM
Hashing	SHA-256
Concurrency	ConcurrentHashMap
Validation	Jakarta Validation
Testing	JUnit / Spring Boot Test
Containerization	Docker
Deployment	Render
📁 Project Structure
src/
├── main/
│   ├── java/
│   │   └── ...
│   │
│   └── resources/
│       ├── application.properties
│       └── templates/
│
└── test/
    └── java/

Main backend components include:

model/
├── Account.java
├── Transaction.java
├── MeshPacket.java
└── PaymentInstruction.java

crypto/
├── ServerKeyHolder.java
└── HybridCryptoService.java

service/
├── DemoService.java
├── VirtualDevice.java
├── MeshSimulatorService.java
├── IdempotencyService.java
├── SettlementService.java
└── BridgeIngestionService.java

controller/
├── ApiController.java
└── DashboardController.java

config/
└── AppConfig.java
🌐 REST API
Endpoint	Purpose
GET /	Dashboard
GET /api/server-key	Get server public key
GET /api/accounts	View demo accounts
GET /api/transactions	View transactions
GET /api/mesh/state	View mesh state
POST /api/demo/send	Create demo payment
POST /api/mesh/gossip	Run mesh gossip
POST /api/mesh/flush	Flush bridge packets
POST /api/mesh/reset	Reset demo state
POST /api/bridge/ingest	Ingest bridge packet
/h2-console	H2 database console
🚀 Run Locally
Requirements

Make sure you have:

Java 17+
Maven
Git

Clone the repository:

git clone https://github.com/Ankurshrivastavaa/upi-offline-mesh.git

Move into the project:

cd upi-offline-mesh

Run the application:

Windows
mvnw.cmd spring-boot:run
Linux / macOS
./mvnw spring-boot:run

The application will start on:

http://localhost:8080
🐳 Run with Docker

Build the Docker image:

docker build -t upi-offline-mesh .

Run the container:

docker run -p 8080:8080 upi-offline-mesh

Open:

http://localhost:8080
☁️ Deployment

The application is containerized using Docker and deployed on Render.

Deployment flow:

GitHub
   │
   ▼
Render
   │
   ▼
Docker Build
   │
   ▼
Java 17 Container
   │
   ▼
Spring Boot Application
Live Application

https://upi-offline-mesh-nvgy.onrender.com

📊 Engineering Highlights
Secure Data Handling
RSA-OAEP key protection
AES-256-GCM authenticated encryption
SHA-256 packet hashing
Freshness validation
Tamper detection
Concurrency
Atomic ConcurrentHashMap.putIfAbsent
Database uniqueness constraint
JPA optimistic locking
Transaction Integrity
Spring @Transactional
Atomic sender debit / receiver credit
Transaction ledger
Backend Architecture
Layered Spring Boot architecture
REST APIs
Service-oriented business logic
JPA persistence
Automated tests
Deployment
Dockerized application
Java 17 runtime
Maven build
Render deployment
⚠️ Current Limitations

This project is intentionally a backend simulation and has several limitations.

Simulated Mesh

The mesh is implemented in software.

It does not currently communicate using:

Bluetooth
BLE
Wi-Fi Direct
Nearby Connections
Database

The application currently uses an H2 in-memory database.

Restarting the application resets the database state.

Idempotency Storage

The in-memory ConcurrentHashMap is suitable for demonstrating concurrency and duplicate prevention within the running application.

A distributed production deployment would require shared durable state such as Redis or a database-backed idempotency mechanism.

Cryptographic Keys

The RSA keypair is generated when the application starts.

A production system would require secure key storage and key rotation.

Authentication

The demo does not implement production-grade user authentication, authorization, or rate limiting.

Real Payment Infrastructure

The project does not connect to:

UPI
NPCI
Banks
Payment gateways
Real payment accounts
🔮 Possible Production Evolution

If this architecture were extended toward a production system, potential improvements would include:

H2
 │
 ▼
PostgreSQL

ConcurrentHashMap
 │
 ▼
Redis / Distributed Idempotency Store

Simulated Mesh
 │
 ▼
BLE / Wi-Fi Direct / Nearby Networking

Generated Keys
 │
 ▼
Secure Key Management / KMS

Basic API
 │
 ▼
Authentication + Authorization
+ Rate Limiting
+ Monitoring
+ Audit Logging

These are future architectural directions rather than features currently implemented in the project.

🧠 Key Learnings

This project helped explore practical backend engineering concepts including:

Hybrid cryptography
AES-GCM authenticated encryption
RSA-OAEP
SHA-256 hashing
Idempotent API processing
Concurrent request handling
Optimistic locking
Database transactions
JPA entity relationships
REST API design
Spring Boot service architecture
Automated testing
Docker containerization
Cloud deployment

The main takeaway is that handling a payment request is not only about moving data from one API to another.

A reliable payment-processing backend also needs to consider:

Security
   +
Concurrency
   +
Idempotency
   +
Transaction Integrity
   +
Failure Handling
👨‍💻 Author
Ankur Shrivastava

B.Tech Computer Science with Business Systems

School of Information Technology, RGPV

Bhopal, Madhya Pradesh, India

Links

GitHub:
https://github.com/Ankurshrivastavaa

LinkedIn:
https://www.linkedin.com/in/ankur-shrivastava-65184724b/

⭐ Project

If you found the project interesting, feel free to explore the repository and the implementation.

GitHub:
https://github.com/Ankurshrivastavaa/upi-offline-mesh

Live Demo:
https://upi-offline-mesh-nvgy.onrender.com
