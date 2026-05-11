# How to Run & Test UPI Offline Mesh

## 1️⃣ RUN THE APPLICATION

### Mac/Linux:
```bash
./mvnw spring-boot:run
```

### Windows:
```cmd
mvnw.cmd spring-boot:run
```

**Expected Output:**
```
Started UpiMeshApplication in X.XXX seconds (process running for Y.YYY s)
```

**Open Dashboard:**
→ http://localhost:8080

**Stop:**
→ Press `Ctrl+C` in terminal

---

## 2️⃣ TEST THE APPLICATION

### Run All Tests:
```bash
./mvnw clean test
```

### Run Specific Test:
```bash
./mvnw test -Dtest=IdempotencyConcurrencyTest
```

### Run Specific Test Method:
```bash
./mvnw test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

---

## 3️⃣ AVAILABLE TESTS

1. **`encryptDecryptRoundTrip`**
   - Verifies hybrid RSA-OAEP + AES-256-GCM encryption/decryption works

2. **`tamperedCiphertextIsRejected`**
   - Flips a byte in ciphertext, ensures it's rejected as INVALID

3. **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`** ⭐ THE KEY TEST
   - 3 threads deliver same packet simultaneously
   - Asserts: exactly 1 SETTLED, 2 DUPLICATE_DROPPED
   - Proves idempotency works under concurrency

---

## 4️⃣ DASHBOARD FLOW (After Starting Server)

Visit http://localhost:8080 and follow these buttons:

### Step 1: Inject Payment into Mesh
Click **"📤 Inject into Mesh"**
- Choose sender, receiver, amount
- Creates encrypted MeshPacket, hands to virtual device "phone-alice"

### Step 2: Run Gossip Rounds  
Click **"🔄 Run Gossip Round"** (2-3 times)
- Packet hops device-to-device via simulated Bluetooth
- Every device broadcasts to every other device
- TTL decrements per hop

### Step 3: Bridge Uploads to Backend
Click **"📡 Bridges Upload to Backend"**
- Simulates bridge node getting 4G and uploading
- Triggers full pipeline: hash → claim → decrypt → settle
- Watch Account Balances table update ✅

### Step 4: Demo Idempotency (Optional)
- Modify code to seed multiple bridges
- Click Gossip multiple times
- Click "Flush Bridges" → see DUPLICATE_DROPPED in response

---

## 5️⃣ API ENDPOINTS (For Integration Testing)

### Get Accounts & Balances:
```bash
curl http://localhost:8080/api/accounts
```

### Get Transaction Ledger:
```bash
curl http://localhost:8080/api/transactions
```

### Get Server Public Key:
```bash
curl http://localhost:8080/api/server-key
```

### Get Mesh State (All Virtual Devices):
```bash
curl http://localhost:8080/api/mesh/state
```

### Ingest Packet (Production Endpoint):
```bash
curl -X POST http://localhost:8080/api/bridge/ingest \
  -H "Content-Type: application/json" \
  -H "X-Bridge-Node-Id: phone-bridge-42" \
  -H "X-Hop-Count: 3" \
  -d '{
    "packetId": "550e8400-e29b-41d4-a716-446655440000",
    "ttl": 2,
    "createdAt": 1730000000000,
    "ciphertext": "base64-encoded-blob"
  }'
```

---

## 6️⃣ DATABASE (H2 Console)

### Access:
→ http://localhost:8080/h2-console

### Login:
- **JDBC URL:** `jdbc:h2:mem:upimesh`
- **Username:** `sa`
- **Password:** (leave blank)

### Useful Queries:
```sql
SELECT * FROM account;                     -- Check balances
SELECT * FROM transaction;                 -- Check settled txns
SELECT * FROM transaction WHERE outcome='INVALID';  -- Check failed txns
```

---

## 7️⃣ TROUBLESHOOTING

| Problem | Solution |
|---------|----------|
| `java: command not found` | Install JDK 25 or Java 17+ |
| Port 8080 in use | Change `server.port` in `application.properties` |
| Slow first run | Maven downloading dependencies (~80MB). Wait 2-3 min |
| Tests fail intermittently | Concurrency test is timing-sensitive; run 3x |

---

## 8️⃣ PROJECT STRUCTURE

```
src/main/
├── java/com/demo/upimesh/
│   ├── UpiMeshApplication.java          ← Main Spring Boot entry point
│   ├── model/
│   │   ├── Account.java                 ← JPA entity
│   │   ├── Transaction.java             ← Settled TX ledger
│   │   ├── MeshPacket.java              ← Wire format
│   │   └── PaymentInstruction.java      ← Decrypted payload
│   ├── crypto/
│   │   ├── HybridCryptoService.java     ← RSA + AES encryption
│   │   └── ServerKeyHolder.java         ← Key generation
│   ├── service/
│   │   ├── BridgeIngestionService.java  ← THE critical pipeline
│   │   ├── IdempotencyService.java      ← Duplicate detection
│   │   ├── SettlementService.java       ← Debit/credit/ledger
│   │   ├── MeshSimulatorService.java    ← Gossip simulation
│   │   └── DemoService.java             ← Test data seeding
│   └── controller/
│       ├── ApiController.java           ← REST endpoints
│       └── DashboardController.java     ← Serves UI
└── resources/
    ├── application.properties           ← Config
    └── templates/dashboard.html         ← Interactive UI

src/test/
└── java/com/demo/upimesh/
    └── IdempotencyConcurrencyTest.java ← 3 critical tests
```

---

## 9️⃣ KEY BEHAVIOR TO OBSERVE

### ✅ Encryption Works
- Run test: `encryptDecryptRoundTrip` passes

### ✅ Tampering Detected
- Run test: `tamperedCiphertextIsRejected` passes
- Try modifying ciphertext → INVALID response

### ✅ Idempotency Works
- Run test: `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` passes
- Same payment from 3 bridges → 1 settles, 2 dropped

### ✅ Freshness Check Works
- Old packets (>24h) are rejected even if properly encrypted

---

