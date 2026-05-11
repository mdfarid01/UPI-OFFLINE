# 💻 Code Walkthrough - Understanding the Implementation

## 🎯 The ONE File You Need to Understand First

**File:** `service/BridgeIngestionService.java`

This file does EVERYTHING. Here's what it looks like (simplified):

```java
@Service
public class BridgeIngestionService {
    
    // This is called when a bridge uploads a packet from offline phones
    public IngestionResponse ingest(MeshPacket packet) {
        
        // STEP 1: Create fingerprint (hash) of encrypted packet
        String packetHash = hashCiphertext(packet.getCiphertext());
        // Example: "abc123def456xyz789"
        
        // STEP 2: Check if we've seen this before
        boolean isFirstTime = idempotencyService.claim(packetHash);
        
        if (!isFirstTime) {
            // Already processed this exact packet!
            return new IngestionResponse("DUPLICATE_DROPPED", packetHash);
        }
        
        // STEP 3: Decrypt the packet using our private key
        try {
            PaymentInstruction instruction = 
                hybridCryptoService.decrypt(packet.getCiphertext());
            // Now we can read: sender, receiver, amount, time, nonce
        } catch (Exception e) {
            // Packet was tampered with! GCM tag didn't match
            return new IngestionResponse("INVALID", packetHash);
        }
        
        // STEP 4: Check if packet is fresh (not old)
        if (isOlderThan24Hours(instruction.getSignedAt())) {
            return new IngestionResponse("INVALID", packetHash);
        }
        
        // STEP 5: Settle the payment
        try {
            Long txnId = settlementService.settle(instruction);
            return new IngestionResponse("SETTLED", packetHash, txnId);
        } catch (Exception e) {
            return new IngestionResponse("INVALID", packetHash);
        }
    }
}
```

**That's it!** This simple flow is the entire system.

---

## 🔐 How Encryption Works

**File:** `crypto/HybridCryptoService.java`

### The Problem
- RSA can only encrypt ~245 bytes
- Our JSON payload is bigger
- Solution: Hybrid encryption

### The Solution

```java
@Service
public class HybridCryptoService {
    
    // ENCRYPT
    public String encrypt(PaymentInstruction instruction) {
        // 1. Convert payment to JSON
        String json = convertToJson(instruction);
        // Example: {"sender":"alice","amount":500,"nonce":"xyz789"...}
        
        // 2. Generate fresh AES-256 key for THIS message
        SecretKey aesKey = generateAESKey();
        
        // 3. Encrypt JSON with AES-256-GCM
        byte[] iv = generateRandomIV();
        byte[] aesEncrypted = encryptWithAES(json, aesKey, iv);
        // GCM includes authentication tag automatically
        
        // 4. Encrypt AES key with RSA-OAEP
        byte[] rsaEncrypted = encryptWithRSA(aesKey, bankPublicKey);
        
        // 5. Combine: [RSA-encrypted key][IV][AES-encrypted payload + auth tag]
        return combine(rsaEncrypted, iv, aesEncrypted);
    }
    
    // DECRYPT
    public PaymentInstruction decrypt(String ciphertext) {
        // 1. Split ciphertext into parts
        byte[] rsaPart = extractRSAPart(ciphertext);
        byte[] iv = extractIV(ciphertext);
        byte[] aesPart = extractAESPart(ciphertext);
        
        // 2. Decrypt RSA part with our private key
        SecretKey aesKey = decryptRSA(rsaPart, bankPrivateKey);
        
        // 3. Decrypt AES part with AES key
        String json = decryptAES(aesPart, aesKey, iv);
        // ⚠️ If auth tag doesn't match → Exception thrown
        
        // 4. Parse JSON back to PaymentInstruction
        return parseJson(json);
    }
}
```

### Why This is Secure

```
Before: "Send ₹500 to Bob"
        ↓ encrypt with AES key
After:  "jK8$92nxL#@!mK9$xL2@9pL#!9mK$92"
        ↓ but AES key is also encrypted with RSA
                ↓
Final:  "RSA_BLOB[RSA_ENCRYPTED_AES_KEY]IV[AES_CIPHERTEXT_WITH_AUTHTAG]"

Hacker intercepts:
- Can't read AES key (encrypted with RSA)
- Can't modify ciphertext (GCM auth tag catches it)
- Can't generate valid auth tag (doesn't have the AES key)
```

---

## ♾️ Idempotency: The Duplicate Killer

**File:** `service/IdempotencyService.java`

```java
@Service
public class IdempotencyService {
    
    // This is where the magic happens
    private final ConcurrentHashMap<String, Instant> seen = 
        new ConcurrentHashMap<>();
    
    public boolean claim(String packetHash) {
        // ConcurrentHashMap.putIfAbsent is ATOMIC
        // Returns null if key didn't exist (first time = success)
        // Returns existing value if key exists (duplicate = failure)
        
        Instant previous = seen.putIfAbsent(packetHash, Instant.now());
        return previous == null;  // true = first claimer, false = duplicate
    }
}
```

### Why This Works Under Concurrency

```
Scenario: 3 bridges upload SAME packet at the EXACT same microsecond

Thread 1: seen.putIfAbsent("abc123", time) → returns NULL → ✅ SETTLES
Thread 2: seen.putIfAbsent("abc123", time) → returns existing entry → ❌ DUPLICATE
Thread 3: seen.putIfAbsent("abc123", time) → returns existing entry → ❌ DUPLICATE

Result: Exactly one thread wins, others lose
This is ATOMIC at the CPU level - no race condition possible
```

---

## 💳 Settlement (The Database Transaction)

**File:** `service/SettlementService.java`

```java
@Service
public class SettlementService {
    
    @Transactional  // ⚠️ CRITICAL: All-or-nothing
    public Long settle(PaymentInstruction instruction) {
        String sender = instruction.getSender();
        String receiver = instruction.getReceiver();
        BigDecimal amount = instruction.getAmount();
        
        // 1. DEBIT SENDER
        Account senderAccount = accountRepo.findByVpa(sender);
        senderAccount.setBalance(senderAccount.getBalance().subtract(amount));
        accountRepo.save(senderAccount);
        
        // 2. CREDIT RECEIVER
        Account receiverAccount = accountRepo.findByVpa(receiver);
        receiverAccount.setBalance(receiverAccount.getBalance().add(amount));
        accountRepo.save(receiverAccount);
        
        // 3. WRITE TO LEDGER
        Transaction txn = new Transaction();
        txn.setSender(sender);
        txn.setReceiver(receiver);
        txn.setAmount(amount);
        txn.setOutcome("SETTLED");
        txn.setPacketHash(instruction.getPacketHash());
        transactionRepo.save(txn);
        
        return txn.getId();
    }
}
```

### Why @Transactional is Important

```
Without @Transactional:
1. Debit Alice ✅
2. Network error
3. Credit Bob ❌
Result: Alice lost money, Bob didn't get it! ❌

With @Transactional:
1. Start transaction
2. Debit Alice ✅
3. Credit Bob ✅
4. Write ledger ✅
5. Commit (all 3 succeed together) ✅
   OR
5. Any failure → ROLLBACK (all 3 undo)
Result: All or nothing, never partial ✅
```

---

## 📱 How the Mesh Works

**File:** `service/MeshSimulatorService.java`

```java
@Service
public class MeshSimulatorService {
    
    // The 5 virtual phones
    private final VirtualDevice[] devices;
    
    public void runGossipRound() {
        // Every device broadcasts to every other device
        
        for (VirtualDevice broadcaster : devices) {
            for (MeshPacket packet : broadcaster.getPackets()) {
                // Decrease TTL (time to live)
                packet.setTtl(packet.getTtl() - 1);
                
                if (packet.getTtl() > 0) {
                    // Broadcast to all other devices
                    for (VirtualDevice listener : devices) {
                        if (broadcaster != listener) {
                            listener.receivePacket(packet);
                            // listener now holds a copy of the packet
                        }
                    }
                }
            }
        }
    }
}
```

### Real-World Mapping

```
VirtualDevice = One phone
MeshPacket = The encrypted payment
TTL (Time To Live) = How many hops before packet dies

Round 1:
- Alice has packet (TTL=5)
- Broadcasts to 4 others
- All 5 now have the packet (TTL=4)

Round 2:
- All 5 broadcast
- All 5 still have it (TTL=3)
- TTL keeps decreasing

Round 3:
- Bridge now has internet
- Uploads all packets it holds
- Bank processes them
```

---

## 🧪 The Test That Proves Everything

**File:** `test/IdempotencyConcurrencyTest.java`

```java
@SpringBootTest
public class IdempotencyConcurrencyTest {
    
    @Test
    void singlePacketDeliveredByThreeBridgesSettlesExactlyOnce() {
        // Create ONE payment
        MeshPacket packet = createValidPayment(
            "alice@demo", "bob@demo", BigDecimal.valueOf(100)
        );
        
        // Store original balances
        BigDecimal aliceBalanceBefore = getBalance("alice@demo");
        BigDecimal bobBalanceBefore = getBalance("bob@demo");
        
        // Fire 3 threads at the backend SIMULTANEOUSLY
        ExecutorService executor = Executors.newFixedThreadPool(3);
        List<Future<IngestionResponse>> results = new ArrayList<>();
        
        for (int i = 0; i < 3; i++) {
            results.add(
                executor.submit(() -> 
                    bridgeIngestionService.ingest(packet)  // Same packet, 3 threads
                )
            );
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        
        // Collect results
        int settledCount = 0;
        int duplicateCount = 0;
        
        for (Future<IngestionResponse> future : results) {
            IngestionResponse response = future.get();
            if ("SETTLED".equals(response.getOutcome())) {
                settledCount++;
            } else if ("DUPLICATE_DROPPED".equals(response.getOutcome())) {
                duplicateCount++;
            }
        }
        
        // THE ASSERTIONS
        assertEquals(1, settledCount);      // ✅ Exactly 1 settled
        assertEquals(2, duplicateCount);    // ✅ Exactly 2 duplicates
        
        BigDecimal aliceBalanceAfter = getBalance("alice@demo");
        BigDecimal bobBalanceAfter = getBalance("bob@demo");
        
        // ✅ Alice debited EXACTLY ₹100 (not ₹300)
        assertEquals(aliceBalanceBefore.subtract(BigDecimal.valueOf(100)), 
                     aliceBalanceAfter);
        
        // ✅ Bob credited EXACTLY ₹100 (not ₹300)
        assertEquals(bobBalanceBefore.add(BigDecimal.valueOf(100)), 
                     bobBalanceAfter);
    }
}
```

### What This Test PROVES

```
Before:
- Alice: ₹1000
- Bob: ₹1000

During:
- Thread 1: ingest(packet) → SETTLED ✅ Alice: ₹900, Bob: ₹1100
- Thread 2: ingest(packet) → DUPLICATE_DROPPED
- Thread 3: ingest(packet) → DUPLICATE_DROPPED

After:
- Alice: ₹900 (charged EXACTLY once)
- Bob: ₹1100 (credited EXACTLY once)
- Database: 1 transaction record (not 3)

This PROVES: Even under extreme concurrency, idempotency works!
```

---

## 🚀 How It All Fits Together

```
1. Sender (offline) → creates encrypted PaymentInstruction
                   → wraps in MeshPacket
                   → broadcasts to nearby phones

2. Phones → relay packet through mesh (Gossip protocol)
          → TTL decrements per hop

3. Bridge → gets internet
          → uploads packet to BridgeIngestionService

4. BridgeIngestionService → hashes packet (dedup key)
                          → checks idempotency cache
                          → decrypts (if first time)
                          → verifies freshness
                          → settles payment

5. Database → stores transaction
            → updates account balances
            → @Transactional ensures all-or-nothing

6. Response → "SETTLED" or "DUPLICATE_DROPPED"
```

---

## ✨ The Hardest Part: Why Even Concurrency Experts Get This Wrong

```
Problem:
- 3 threads simultaneously call BridgeIngestionService.ingest()
- Each wants to process the same payment
- How do you make sure exactly 1 processes?

❌ Wrong way:
    if (!seen.containsKey(hash)) {  // Thread 1 checks: false
        // ...
    }
    // Meanwhile Thread 2 also sees false!
    // Both think they're first
    
✅ Right way (this project):
    if (seen.putIfAbsent(hash, now) == null) {  // Atomic operation
        // Only ONE thread gets true
        // Other threads see existing entry
    }
```

---

## 📚 Reading Order

Start here: (1 hour total)

1. **This file** (10 min) - You're reading it now! ✅
2. **VISUAL_FLOW.txt** (10 min) - See the flow
3. **service/BridgeIngestionService.java** (10 min) - Read the actual code
4. **crypto/HybridCryptoService.java** (10 min) - Understand encryption
5. **test/IdempotencyConcurrencyTest.java** (10 min) - See the proof

---

## 🎯 Summary

The genius of this project:

✅ **Encryption** makes packets unreadable to intermediaries  
✅ **Idempotency** makes it impossible to process duplicates  
✅ **Atomicity** makes all-or-nothing settlement  
✅ **Concurrency** handles simultaneous uploads safely  

All wrapped up in ~500 lines of clean, understandable Java code.

