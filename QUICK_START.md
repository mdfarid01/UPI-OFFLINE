# 🚀 UPI Offline Mesh — Quick Start Guide

## ✅ Current Status
- **Java**: Upgraded from 17 → **25** (latest LTS)
- **Spring Boot**: Upgraded from 3.3.5 → **3.5.0**
- **Tests**: **3/3 PASSING** ✓ with Java 25

---

## 📖 Test Results (Just Ran)

```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 ✅
Total time: 5.881 seconds

✓ encryptDecryptRoundTrip — Hybrid encryption works
✓ tamperedCiphertextIsRejected — Tampering detected  
✓ singlePacketDeliveredByThreeBridgesSettlesExactlyOnce — Idempotency under 3 concurrent threads ⭐
```

### What These Tests Prove:

| Test | What it validates |
|------|-------------------|
| **`encryptDecryptRoundTrip`** | RSA-OAEP + AES-256-GCM encrypt/decrypt is symmetric |
| **`tamperedCiphertextIsRejected`** | GCM auth tag detects any bit flip → INVALID response |
| **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`** | 3 threads, 1 packet, simultaneous delivery → exactly 1 SETTLED, 2 DUPLICATE_DROPPED |

---

## 🏃 How to RUN

### Start the Server (Terminal 1)
```bash
./mvnw spring-boot:run
```

**Expected:** Server starts in ~5-10 seconds

### Open Dashboard (Browser)
```
http://localhost:8080
```

### Stop Server
```
Ctrl+C
```

---

## 🧪 How to TEST

### Run All Tests
```bash
./mvnw clean test
```

### Run ONE Specific Test
```bash
./mvnw test -Dtest=IdempotencyConcurrencyTest
```

### Run ONE Specific Test Method
```bash
./mvnw test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

---

## 🎮 Interactive Dashboard (After Starting Server)

### Step 1: Inject Payment
- Choose sender, receiver, amount
- Click **"📤 Inject into Mesh"**
- Packet created and handed to virtual device

### Step 2: Run Gossip Rounds
- Click **"🔄 Run Gossip Round"** (2-3 times)
- Packet hops through simulated Bluetooth mesh
- Watch TTL decrease

### Step 3: Upload via Bridge
- Click **"📡 Bridges Upload to Backend"**
- Simulates bridge node getting 4G
- Triggers: hash → claim → decrypt → settle pipeline
- **Watch Account Balances update** ✅

### Step 4: View Results
- **Transaction Ledger** shows settled TX
- **Account Balances** shows updated amounts
- **H2 Console** (http://localhost:8080/h2-console) for raw DB query

---

## 🔗 Useful URLs (After Starting Server)

| URL | Purpose |
|-----|---------|
| http://localhost:8080 | Dashboard UI |
| http://localhost:8080/api/accounts | View all accounts & balances |
| http://localhost:8080/api/transactions | View transaction ledger |
| http://localhost:8080/api/mesh/state | View virtual device state |
| http://localhost:8080/h2-console | Browse database (JDBC: `jdbc:h2:mem:upimesh`, user: `sa`) |

---

## 📁 Key Files to Understand

```
src/main/java/com/demo/upimesh/

Cryptography (How packets are protected):
├── crypto/HybridCryptoService.java     ← RSA-OAEP + AES-256-GCM
├── crypto/ServerKeyHolder.java         ← RSA keypair generation

The Critical Pipeline (How payments settle):
├── service/BridgeIngestionService.java ← hash → claim → decrypt → settle
├── service/IdempotencyService.java     ← Atomic deduplication (ConcurrentHashMap)
├── service/SettlementService.java      ← @Transactional debit/credit

Mesh Simulation:
├── service/MeshSimulatorService.java   ← Gossip protocol
├── service/VirtualDevice.java          ← One simulated phone

REST API & UI:
├── controller/ApiController.java       ← 8 endpoints
├── controller/DashboardController.java ← Serves UI
└── model/                              ← JPA entities (Account, Transaction, MeshPacket, PaymentInstruction)

Tests:
src/test/java/com/demo/upimesh/
└── IdempotencyConcurrencyTest.java    ← 3 critical tests
```

---

## 🎯 What to Try Next

### 1️⃣ Start the server and play with the dashboard
```bash
./mvnw spring-boot:run
# → http://localhost:8080
```

### 2️⃣ Run the tests and watch the logs
```bash
./mvnw test
```
**Look for logs showing:**
- `Server RSA keypair generated` (crypto layer)
- `DUPLICATE packet ... from bridge` (idempotency layer)
- `SETTLED ₹100.00 from alice@demo to bob@demo` (settlement layer)

### 3️⃣ Query the database
- Open http://localhost:8080/h2-console
- Run: `SELECT * FROM account;`
- Run: `SELECT * FROM transaction;`

### 4️⃣ Call the API directly
```bash
# Get server public key
curl http://localhost:8080/api/server-key

# Get accounts
curl http://localhost:8080/api/accounts

# Get all transactions
curl http://localhost:8080/api/transactions
```

### 5️⃣ Read the code
- Start with `service/BridgeIngestionService.java` (the core pipeline)
- Then `crypto/HybridCryptoService.java` (how encryption works)
- Then `IdempotencyConcurrencyTest.java` (the proof of idempotency)

---

## 🐛 Troubleshooting

| Problem | Fix |
|---------|-----|
| `Port 8080 already in use` | Change `server.port` in `application.properties` |
| Tests hang | Give it 30 seconds (Spring context startup) |
| Dashboard won't load | Make sure server is running (`mvnw spring-boot:run`) |
| H2 console won't connect | Use JDBC URL: `jdbc:h2:mem:upimesh`, user: `sa`, no password |
| `java: command not found` | Install JDK 25 or Java 17+ |

---

## ✨ Key Takeaways

✅ **Encryption works** — Hybrid RSA-OAEP + AES-256-GCM  
✅ **Idempotency works** — Atomic ConcurrentHashMap deduplication  
✅ **Tampering detected** — GCM auth tag prevents fake packets  
✅ **Concurrency safe** — 3 threads, 1 packet → exactly 1 settles  
✅ **Java 25 compatible** — All tests pass with latest LTS

---

## 📚 Architecture Summary

```
Offline Phone (encrypted)
        ↓ Bluetooth mesh (hops)
Untrusted Intermediaries
        ↓ One walks outside
Bridge Node (has 4G)
        ↓ HTTPS POST
Spring Boot Backend (this project)
        1. Hash ciphertext (dedup key)
        2. Claim in cache (atomic)
        3. Decrypt with RSA
        4. Check freshness
        5. Settle: debit/credit/ledger
        ↓
Result: SETTLED or DUPLICATE_DROPPED
```

---

Enjoy exploring! 🎉

