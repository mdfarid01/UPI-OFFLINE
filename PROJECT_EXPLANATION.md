# Project Explanation For Resume, Viva, And College Presentation

Project name:

```text
UPI Offline Mesh
```

Resume-friendly one-liner:

```text
Built a Java Spring Boot system that simulates offline UPI payments routed through a Bluetooth-style mesh network, with encrypted packets, duplicate prevention, replay protection, and transactional settlement.
```

---

## 1. What Problem Does This Project Solve?

Normal UPI needs internet.

If a user is in a basement, remote village, railway station, disaster area, crowded event, or any place with poor connectivity, normal UPI payments may fail because the sender phone cannot directly contact the bank/UPI backend.

This project explores a possible idea:

```text
Can a phone create a payment offline and pass it through nearby phones until one phone gets internet?
```

The answer shown by this project is:

```text
Yes, as a deferred-settlement demo.
```

That means the payment is not settled instantly. It is created offline, carried securely, and settled later when a bridge device reaches the backend.

---

## 2. Why Is This Project Needed?

This project is useful because it demonstrates real backend engineering problems:

- Offline-first system design
- Secure message transfer through untrusted devices
- Idempotency
- Duplicate request handling
- Replay protection
- Transactional debit/credit settlement
- Java Spring Boot REST APIs
- Database-backed ledger
- Dashboard-based simulation

Even though the Bluetooth mesh is simulated, the backend concepts are real and important in payment systems.

---

## 3. Simple Real-Life Example

Imagine Alice and Bob are in a basement with no internet.

Alice wants to send Bob INR 500.

Normal UPI:

```text
Alice phone -> Internet -> UPI backend -> Bank
```

But internet is unavailable.

In this project:

```text
Alice phone creates encrypted payment
Nearby phones carry it
One phone gets internet
That phone uploads it to backend
Backend verifies and settles
```

The nearby phones are only messengers. They cannot read the amount, sender, receiver, or PIN hash.

---

## 4. High-Level Architecture

```text
Offline Sender
     |
     | creates encrypted payment
     v
Mesh Packet
     |
     | gossiped through nearby phones
     v
Bridge Device With Internet
     |
     | POST /api/bridge/ingest
     v
Spring Boot Backend
     |
     | decrypt, validate, deduplicate
     v
Settlement Service
     |
     | debit sender, credit receiver
     v
Transaction Ledger
```

---

## 5. Main Components

### A. Dashboard UI

File:

```text
src/main/resources/templates/dashboard.html
```

Purpose:

- Lets user compose payment
- Shows virtual mesh devices
- Shows packets moving
- Shows account balances
- Shows transaction ledger
- Shows activity log

It is not a separate frontend framework. It is a Thymeleaf HTML page served by Spring Boot.

### B. REST Controller

File:

```text
src/main/java/com/demo/upimesh/controller/ApiController.java
```

Purpose:

- Exposes APIs for dashboard and mesh simulation
- Provides account and transaction data
- Accepts bridge packet ingestion

Important endpoints:

```text
GET  /api/accounts
GET  /api/transactions
GET  /api/mesh/state
POST /api/demo/send
POST /api/mesh/gossip
POST /api/mesh/flush
POST /api/bridge/ingest
```

### C. Demo Service

File:

```text
src/main/java/com/demo/upimesh/service/DemoService.java
```

Purpose:

- Seeds demo accounts
- Simulates sender phone
- Creates encrypted payment packets

Seed accounts:

```text
alice@demo  INR 5000
bob@demo    INR 1000
carol@demo  INR 2500
dave@demo   INR 500
```

### D. Mesh Simulator

Files:

```text
MeshSimulatorService.java
VirtualDevice.java
```

Purpose:

- Simulates nearby phones
- Stores packets in virtual devices
- Runs gossip rounds
- Finds bridge devices with internet

Default virtual devices:

```text
phone-alice       offline
phone-stranger1   offline
phone-stranger2   offline
phone-stranger3   offline
phone-bridge      internet available
```

### E. Cryptography Service

File:

```text
src/main/java/com/demo/upimesh/crypto/HybridCryptoService.java
```

Purpose:

- Encrypt payment instructions
- Decrypt packets on backend
- Hash ciphertext for idempotency

Uses:

```text
RSA-OAEP + AES-256-GCM
```

### F. Idempotency Service

File:

```text
src/main/java/com/demo/upimesh/service/IdempotencyService.java
```

Purpose:

- Prevents same payment from settling multiple times
- Uses `ConcurrentHashMap.putIfAbsent`

This is similar to Redis `SETNX` in production.

### G. Bridge Ingestion Service

File:

```text
src/main/java/com/demo/upimesh/service/BridgeIngestionService.java
```

Purpose:

This is the most important backend pipeline.

It does:

```text
receive packet
hash ciphertext
claim hash
decrypt
check freshness
settle or reject
return result
```

### H. Settlement Service

File:

```text
src/main/java/com/demo/upimesh/service/SettlementService.java
```

Purpose:

- Checks payment rules
- Debits sender
- Credits receiver
- Writes transaction ledger

Uses:

```text
@Transactional
```

This ensures debit and credit happen together.

---

## 6. Detailed Flow

### Step 1: User Clicks Inject

Dashboard sends:

```http
POST /api/demo/send
```

Backend creates:

```text
PaymentInstruction
```

Fields:

```text
senderVpa
receiverVpa
amount
pinHash
nonce
signedAt
```

Then it encrypts it and wraps it into:

```text
MeshPacket
```

Fields:

```text
packetId
ttl
createdAt
ciphertext
```

### Step 2: User Clicks Gossip

Dashboard sends:

```http
POST /api/mesh/gossip
```

Mesh simulator copies packets between virtual devices.

This represents Bluetooth-style packet sharing.

### Step 3: User Clicks Bridge

Dashboard sends:

```http
POST /api/mesh/flush
```

The bridge phone uploads packets to:

```http
POST /api/bridge/ingest
```

### Step 4: Backend Processes Packet

Backend performs:

```text
SHA-256(ciphertext)
```

Then:

```java
seen.putIfAbsent(packetHash, now)
```

If the hash already exists:

```text
DUPLICATE_DROPPED
```

If it is new:

```text
Decrypt -> validate -> settle
```

### Step 5: Settlement

Valid payment:

```text
Alice balance decreases
Bob balance increases
Transaction row status = SETTLED
```

Invalid payment:

```text
Balances unchanged
Transaction row status = REJECTED
```

---

## 7. Why Encryption Is Needed

The payment is carried by stranger phones.

Without encryption:

- Stranger could read sender
- Stranger could read receiver
- Stranger could read amount
- Stranger could modify payment

With encryption:

- Stranger sees only ciphertext
- Backend private key is required to decrypt
- Any tampering breaks AES-GCM authentication

This is why the project uses hybrid encryption.

---

## 8. Why Hybrid Encryption Is Used

RSA alone is not good for large data.

AES is fast but needs a shared secret key.

So the project combines both:

1. Generate random AES key.
2. Encrypt payment JSON with AES-GCM.
3. Encrypt AES key with RSA public key.
4. Send both together.
5. Backend uses RSA private key to recover AES key.
6. Backend uses AES key to decrypt payment.

This pattern is used in many secure systems.

---

## 9. Why Idempotency Is Needed

In a mesh network, the same packet can reach the backend many times.

Example:

```text
Bridge A uploads packet
Bridge B uploads same packet
Bridge C uploads same packet
```

If backend processes all of them:

```text
Alice sends INR 500
Alice gets debited INR 1500
```

That is a serious bug.

So the backend uses ciphertext hash as an idempotency key.

Only the first upload is processed.

All duplicate uploads are dropped.

---

## 10. Why Hash The Ciphertext?

The project hashes:

```text
ciphertext
```

Not:

```text
packetId
```

Reason:

- Packet ID is outside encryption.
- A malicious relay could modify packet ID.
- Ciphertext represents the real payment payload.
- Same packet has same ciphertext.
- Modified ciphertext fails decryption.

So ciphertext hash is a strong idempotency key.

---

## 11. Why `@Transactional` Is Needed

Settlement has two balance changes:

```text
debit sender
credit receiver
```

If one succeeds and the other fails, the ledger becomes corrupt.

`@Transactional` ensures:

```text
Either both happen, or neither happens.
```

This is very important in payment systems.

---

## 12. Business Rules

The settlement layer checks:

| Rule | Result |
|---|---|
| Amount <= 0 | Reject |
| Unknown sender | Reject |
| Unknown receiver | Reject |
| Sender equals receiver | Reject |
| Insufficient balance | Reject |
| Valid payment | Settle |

Same-person transfer example:

```text
alice@demo -> alice@demo
```

Result:

```text
REJECTED
Balance unchanged
```

---

## 13. What Is H2 And Why Used?

H2 is a lightweight Java database.

This project uses H2 in memory:

```properties
spring.datasource.url=jdbc:h2:mem:upimesh
```

Meaning:

- No database installation needed
- Database starts with the app
- Data is temporary
- Data disappears when app stops
- Perfect for demos and college projects

For production, use PostgreSQL or MySQL.

---

## 14. What Makes This A Good Java Spring Boot Project?

This is not just CRUD.

It includes:

- REST API design
- Service-layer architecture
- JPA entities and repositories
- Transaction management
- Cryptography
- Concurrent duplicate handling
- In-memory cache with scheduled cleanup
- Dashboard UI
- Deployment support
- Real-world payment-system thinking

That makes it strong as a major Java backend project.

---

## 15. Resume Points

You can write:

```text
UPI Offline Mesh - Java Spring Boot
```

Bullet points:

```text
- Built a Spring Boot backend simulating offline UPI payments over a Bluetooth-style mesh network.
- Implemented hybrid RSA-OAEP + AES-256-GCM encryption to protect payment packets through untrusted relay devices.
- Designed idempotent bridge ingestion using SHA-256 ciphertext hashes and atomic duplicate detection.
- Implemented transactional settlement with Spring Data JPA, H2, optimistic locking, and ledger records.
- Built an interactive Thymeleaf dashboard to visualize mesh devices, packet propagation, balances, and transaction status.
- Added rejection handling for stale packets, insufficient funds, invalid transfers, and self-transfers.
- Containerized the application with Docker for cloud deployment.
```

---

## 16. Interview/Viva Explanation In 60 Seconds

Say this:

```text
My project is UPI Offline Mesh, a Java Spring Boot demo for offline UPI-style payments.

The problem is that normal UPI requires internet, so payments fail in low-connectivity areas. In my system, the sender creates an encrypted payment packet offline. Nearby phones relay this packet like a Bluetooth mesh. These phones cannot read or modify the payment because it is encrypted using RSA-OAEP and AES-GCM.

When any bridge phone gets internet, it uploads the packet to the Spring Boot backend. The backend first hashes the ciphertext and claims that hash atomically to prevent duplicate settlement. Then it decrypts the packet, checks replay freshness, validates business rules, and performs settlement inside a database transaction.

The project has REST APIs, JPA entities, an H2 database, a transaction ledger, idempotency handling, and an interactive dashboard showing mesh devices, packet flow, balances, and transaction results.
```

---

## 17. Questions Faculty May Ask

### Q: Is this real UPI?

Answer:

```text
No. It is a backend simulation of offline UPI-style deferred settlement. It demonstrates the architecture and safety mechanisms but does not integrate with NPCI or real banks.
```

### Q: Why do you need encryption?

Answer:

```text
Because the payment packet is carried by untrusted phones. Encryption ensures they cannot read or modify the payment.
```

### Q: What if the same packet is uploaded multiple times?

Answer:

```text
The backend hashes the ciphertext and atomically claims it. Only the first upload settles. Others return DUPLICATE_DROPPED.
```

### Q: Why not use packet ID for duplicate detection?

Answer:

```text
Packet ID is outside encryption and can be changed by intermediaries. Ciphertext hash is safer because it identifies the encrypted payment payload itself.
```

### Q: What happens if sender has insufficient balance?

Answer:

```text
The backend decrypts the packet, checks balance, records a REJECTED transaction, and does not change balances.
```

### Q: What happens if Alice sends money to herself?

Answer:

```text
The UI blocks it, and the backend also rejects it. Balances remain unchanged.
```

### Q: Why use H2?

Answer:

```text
H2 is used because this is a demo. It requires no setup and runs in memory. For production, I would replace it with PostgreSQL or MySQL.
```

### Q: Is Bluetooth implemented?

Answer:

```text
No. The Bluetooth mesh is simulated using virtual devices. The backend logic is the focus of the project.
```

### Q: What would you improve for production?

Answer:

```text
I would add real mobile BLE transport, PostgreSQL, Redis for distributed idempotency, proper authentication, HSM/KMS for private keys, rate limiting, monitoring, and integration with a real payment switch.
```

---

## 18. System Design Keywords To Mention

Use these words in resume/interview:

```text
Spring Boot
REST API
JPA
H2
Transactional settlement
Hybrid encryption
AES-GCM
RSA-OAEP
Idempotency
Replay protection
Optimistic locking
Mesh simulation
Deferred settlement
Docker deployment
```

---

## 19. Honest Limitations

Say this clearly if asked:

```text
This project does not guarantee real offline payment finality. It demonstrates deferred settlement. A real offline payment system would need secure hardware, pre-funded offline balance, or bank-approved offline wallets.
```

This honesty makes the project stronger, not weaker.

