# Encryption & Decryption: Interview-Ready Notes

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Symmetric Encryption](#symmetric-encryption)
3. [Asymmetric Encryption](#asymmetric-encryption)
4. [Comparison & Use Cases](#comparison--use-cases)
5. [Real-World Applications](#real-world-applications)

---

## Fundamentals

### What is Encryption & Decryption?

**Encryption** is the process of converting plaintext (readable data) into ciphertext (scrambled/unreadable data) using a cryptographic algorithm and a key.

**Decryption** is the reverse process—converting ciphertext back into plaintext using the appropriate key.

### Key Concepts

- **Plaintext**: Original readable message
- **Ciphertext**: Encrypted, unreadable data
- **Key**: Secret value used to encrypt/decrypt data
- **Algorithm**: Mathematical process for encryption/decryption
- **Substitution**: Replacing plaintext characters with other characters
- **Permutation**: Rearranging the order of data bits

---

## Symmetric Encryption

Symmetric encryption uses **one shared key** for both encryption and decryption. Both parties must possess the same secret key.

### How It Works (Conceptually)

1. Sender has plaintext and a secret key
2. Algorithm encrypts plaintext using the key → produces ciphertext
3. Ciphertext is transmitted
4. Receiver uses the same secret key to decrypt → gets original plaintext

### Characteristics

- **Speed**: Very fast, efficient
- **Key Management**: Challenging (must securely share the same key)
- **Key Size**: Smaller keys (128-256 bits typical)
- **Use Case**: Large amounts of data encryption, bulk data transfer
- **Security Level**: Provides confidentiality but NOT authentication

### Two Main Types

#### 1. Block Ciphers

**Definition**: Encrypts data in fixed-size blocks (usually 128 bits)

**How They Work**:
- Divide plaintext into fixed blocks
- Encrypt each block with the key
- Each block produces a block of ciphertext
- Data is held in memory until a complete block is ready

**Common Block Ciphers**:

##### AES (Advanced Encryption Standard) - **Most Important**
- **Key Sizes**: 128, 192, 256 bits
- **Block Size**: 128 bits (fixed)
- **Rounds**: 10 (for 128-bit key), 12 (for 192-bit key), 14 (for 256-bit key)
- **Structure**: Substitution-Permutation Network (NOT Feistel)
- **Operations Per Round**:
  - **SubBytes**: Replace each byte using a substitution table
  - **ShiftRows**: Shift bytes in each row cyclically left
  - **MixColumns**: Mix data in columns using matrix multiplication (skipped in final round)
  - **AddRoundKey**: XOR with round key
- **Status**: NIST standard (2001), trusted worldwide
- **Use**: HTTPS/SSL-TLS, VPNs, file encryption, government use
- **Why It's Best**: Fast, secure, hardware-accelerated (AES-NI), standardized, highly analyzed

##### DES (Data Encryption Standard) - **Deprecated**
- **Key Size**: 56 bits (very short by modern standards)
- **Block Size**: 64 bits
- **Rounds**: 16 Feistel rounds
- **Status**: NO LONGER SECURE (since 1990s)
- **Why Deprecated**: 56-bit key can be brute-forced, vulnerable to cryptanalysis
- **Still Used**: Legacy banking systems and EMV chip cards (not recommended for new apps)

##### 3DES (Triple DES) - **Deprecated**
- **How It Works**: Applies DES three times to the same data
  - Encrypt with key 1 → Decrypt with key 2 → Encrypt with key 3
  - This "triple" approach makes it harder to crack than single DES
- **Key Sizes**: 112 bits (2-key mode) or 168 bits (3-key mode)
- **Effective Security**: About 112 bits (due to Meet-in-the-Middle attack)
- **Status**: NIST deprecated it; officially disallowed after 2023
- **Why Phased Out**: Slower than AES, effective key strength not as strong, vulnerable to certain attacks
- **Still Used**: EMV, legacy payment systems, but being replaced by AES

##### Blowfish - **Limited Use**
- **Key Sizes**: 32-448 bits (variable)
- **Block Size**: 64 bits (TOO SMALL for modern security)
- **Rounds**: 16 Feistel rounds
- **Invented**: 1993 as DES replacement
- **Status**: Secure but outdated
- **Problems**: 64-bit block leads to birthday attacks, not suitable for large data
- **Use**: Some legacy password managers, older VPN systems, NOT recommended for new projects
- **Modern Alternative**: Twofish (successor)

##### Twofish - **Rarely Used**
- **Key Sizes**: 128, 192, 256 bits
- **Block Size**: 128 bits
- **Rounds**: 16 Feistel rounds
- **Structure**: Feistel network with key-dependent S-boxes
- **Security**: Strong, no practical attacks known
- **Status**: Was AES competition finalist but not selected
- **Why Not Popular**: AES won the competition; Twofish is available but AES is preferred
- **Use**: OpenPGP optional cipher, niche applications

##### IDEA (International Data Encryption Algorithm)
- **Key Size**: 128 bits
- **Block Size**: 64 bits
- **Rounds**: 8 rounds of substitution and permutation
- **Structure**: Different from Feistel (uses modular multiplication and addition)
- **Security**: Secure, no practical attacks found
- **Status**: Older algorithm, not widely adopted
- **Use**: Some legacy VPNs, email encryption
- **Why Limited Use**: 64-bit block size limitation, replaced by AES

##### CAST5 (CAST-128)
- **Key Sizes**: 40-128 bits (variable in 8-bit increments)
- **Block Size**: 64 bits
- **Rounds**: 16 Feistel rounds
- **Structure**: Uses larger S-boxes (8×32 instead of DES 6×4)
- **Use**: Default in GPG/PGP, some secure email, OpenPGP
- **Security**: Considered secure despite age
- **Limitation**: 64-bit blocks, older algorithm

---

#### 2. Stream Ciphers

**Definition**: Encrypts data **bit-by-bit or byte-by-byte** as it flows, one element at a time

**How They Work**:
- Generate a keystream (pseudo-random sequence) from the secret key
- XOR each plaintext bit/byte with corresponding keystream bit/byte
- No need to wait for complete blocks

**Advantages**: 
- Can encrypt variable-length messages
- Real-time encryption possible
- No padding needed

**Disadvantages**:
- Must NEVER reuse the same key with the same keystream
- If keystream repeats, attacker can recover plaintext

**Common Stream Ciphers**:

##### RC4 (Rivest Cipher 4) - **DEPRECATED**
- **Key Sizes**: Variable length
- **Byte-oriented**: Operates on one byte at a time
- **History**: Created 1987, was a trade secret, leaked 1994
- **Famous Use**: WEP (WiFi), SSL/TLS (initially)
- **Status**: COMPLETELY BROKEN - RFC 7465 prohibits use in TLS
- **Vulnerabilities**:
  - Weak key scheduling
  - Statistical biases in keystream
  - Fluhrer-Mantin-Shamir attack breaks WEP
  - Second output byte biased toward zero
- **Modern Status**: DO NOT USE for any new application

##### RC5 & RC6
- **RC5**: Invented 1994, uses variable block and word sizes, relatively simple
- **RC6**: Enhanced version, finalist in AES competition but lost to Rijndael (AES)
- **Status**: RC5/RC6 rarely used; AES is standardized alternative
- **Use**: Limited to legacy systems

##### ChaCha20 - **Modern & Recommended**
- **Key Size**: 256 bits
- **Nonce Size**: 64 bits (unique per message)
- **Design**: Created 2008 by Daniel Bernstein, based on earlier Salsa20
- **Status**: Modern, secure, increasingly adopted
- **Usage**:
  - Google Chrome (TLS cipher suite since 2014)
  - WhatsApp (billions of messages daily)
  - WireGuard VPN protocol
  - TLS 1.3 recommended cipher
- **Advantages**:
  - Fast on CPUs without AES hardware support
  - Resistant to timing attacks
  - No known practical attacks
  - Excellent for mobile and IoT devices
- **Often Combined**: ChaCha20-Poly1305 (with authentication)

##### Salsa20
- **History**: Created 2005 by Daniel Bernstein
- **Status**: Secure alternative to AES, basis for ChaCha20
- **Use**: Some VPNs and applications requiring stream cipher

---

### Block Cipher Modes of Operation

When using block ciphers, you need a "mode" that specifies how to handle multiple blocks:

- **ECB (Electronic Codebook)**: Simple but insecure (same plaintext block = same ciphertext)
- **CBC (Cipher Block Chaining)**: Common, secure, each block depends on previous
- **CTR (Counter Mode)**: Turns block cipher into stream cipher
- **GCM (Galois/Counter Mode)**: Provides both encryption and authentication

---

## Asymmetric Encryption

Asymmetric encryption uses **two different keys**: a public key (for encryption) and a private key (for decryption).

### How It Works (Conceptually)

1. Each person generates a key pair: public key + private key
2. Sender encrypts data using **recipient's public key**
3. Only the recipient (with their private key) can decrypt
4. Public key can be shared openly; private key must be kept secret

### Key Insight

The mathematical relationship is such that:
- What the public key encrypts, ONLY the private key can decrypt
- What the private key signs, the public key can verify
- Cannot derive private key from public key (computationally hard)

### Characteristics

- **Speed**: Much slower than symmetric (100-1000x)
- **Key Management**: Simpler (public key can be shared freely)
- **Key Size**: Larger keys (1024-4096 bits typical, ECC uses smaller)
- **Use Case**: Key exchange, digital signatures, small data, authentication
- **Security**: Provides confidentiality, authentication, non-repudiation

### Main Algorithms

#### RSA (Rivest-Shamir-Adleman) - **Most Common**

**Mathematical Basis**: Difficulty of factoring large numbers

**Key Generation**:
1. Choose two large prime numbers: p and q
2. Calculate n = p × q (part of both public and private key)
3. Calculate φ(n) = (p-1) × (q-1) using Euler's totient function
4. Choose encryption exponent e (typically 65537) such that gcd(e, φ(n)) = 1
5. Calculate decryption exponent d such that (d × e) ≡ 1 mod φ(n)
   - This is the modular multiplicative inverse
6. **Public Key**: (e, n)
7. **Private Key**: (d, n)

**Encryption Process**:
- Ciphertext = Plaintext^e mod n

**Decryption Process**:
- Plaintext = Ciphertext^d mod n

**Key Sizes**: 
- 1024 bits (weak), 2048 bits (current standard), 4096 bits (very secure)

**Applications**:
- HTTPS/SSL-TLS (secure websites)
- Digital signatures
- Email encryption (PGP/GPG)
- Certificate authorities
- Cryptocurrency

**Advantages**:
- Widely standardized and trusted
- Works for both encryption and signatures
- Mathematically proven security

**Disadvantages**:
- Slow (can't encrypt large data directly)
- Requires large key sizes for equivalent security vs ECC
- Vulnerable to quantum computers (Shor's algorithm can break it)

**How Long to Break**: With current computing, 2048-bit RSA is safe. Quantum computers would break it quickly (hence post-quantum cryptography research).

---

#### ECC (Elliptic Curve Cryptography) - **Modern & Efficient**

**Mathematical Basis**: Difficulty of elliptic curve discrete logarithm problem

**Key Concept**: Instead of large integers, uses points on an elliptic curve

**How It Works**:
1. Define an elliptic curve (mathematical equation) and a generator point G
2. Private key: a random number d
3. Public key: a point Q on the curve calculated as Q = d × G (scalar multiplication)
4. Security: Very hard to find d knowing d × G

**Key Sizes**:
- 256-bit ECC ≈ 3072-bit RSA in security strength
- Much smaller keys = faster, less storage

**Variants**:
- **ECDH**: Elliptic Curve Diffie-Hellman (key exchange)
- **ECDSA**: Elliptic Curve Digital Signature Algorithm (signing)

**Advantages**:
- Smaller key sizes with same security
- Faster computation than RSA
- Lower power consumption (good for mobile/IoT)
- Better performance on modern CPUs

**Applications**:
- Bitcoin and blockchain (uses ECDSA with Curve25519)
- TLS 1.3 (modern HTTPS)
- Secure messaging apps
- IoT devices
- Mobile security

**Popular Curves**:
- **NIST curves** (P-256, P-384, P-521): Standardized but some concerns about potential backdoors
- **Curve25519** (X25519, Ed25519): Modern, considered more secure, used in WireGuard
- **Curve448**: Even larger, highest security level

**Why It's Gaining Adoption**: Post-quantum concerns + efficiency makes ECC the future

---

#### Diffie-Hellman (DH) - **Key Exchange Only**

**Purpose**: Two parties establish a shared secret over an insecure channel

**How It Works**:
1. Alice and Bob agree on two numbers: large prime p and generator g
2. Alice picks random secret number a, calculates A = g^a mod p, sends A to Bob
3. Bob picks random secret number b, calculates B = g^b mod p, sends B to Alice
4. Alice calculates: g^(a×b) mod p = B^a mod p
5. Bob calculates: g^(a×b) mod p = A^b mod p
6. Both now have the same shared secret without ever transmitting it!

**Security**: Based on discrete logarithm problem (hard to calculate a from A)

**Limitations**:
- Vulnerable to man-in-the-middle attacks without authentication
- NOT used for encryption directly, only key exchange

**Modern Variant**: ECDH (Elliptic Curve Diffie-Hellman) - same idea but with elliptic curves, smaller keys

**Use**: Establishing symmetric keys, TLS handshake, secure protocols

---

#### ElGamal - **Key Exchange & Signatures**

**Based On**: Diffie-Hellman and discrete logarithm problem

**How It Works**:
- Similar to Diffie-Hellman but extended for encryption
- Less commonly used than RSA or ECC today

**Status**: Rarely used; DSA and ECDSA preferred for signatures; RSA/ECC for encryption

---

#### DSA (Digital Signature Algorithm) - **Signatures Only**

**Purpose**: Create and verify digital signatures (NOT for encryption)

**Key Generation**:
1. Choose large prime p and q (where q divides p-1)
2. Calculate generator g in finite field modulo p
3. Choose random private key x
4. Calculate public key y = g^x mod p

**Signing Process**:
1. Hash message using SHA-256 to get digest
2. Choose random k
3. Calculate r = (g^k mod p) mod q
4. Calculate s = (k^-1 × (message_digest + x×r)) mod q
5. Signature = (r, s)

**Verification Process**:
1. Calculate w = s^-1 mod q
2. Calculate u1 = (message_digest × w) mod q
3. Calculate u2 = (r × w) mod q
4. Calculate v = ((g^u1 × y^u2) mod p) mod q
5. If v = r, signature is valid

**Key Features**:
- Unique signature each time (due to random k)
- Cannot forge without private key
- Based on discrete logarithm problem

**Modern Variant**: ECDSA (Elliptic Curve DSA) - same concept but with elliptic curves

**Status**: Being superseded by ECDSA and EdDSA; still NIST approved for verification only (not generation in FIPS 186-5)

---

#### EdDSA (Edwards-curve Digital Signature Algorithm) - **Modern**

**Basis**: Elliptic curves (Ed25519, Ed448)

**Advantages**:
- No need for random number generation (deterministic)
- Resistant to side-channel attacks
- Fast
- Modern standard

**Use**: SSH keys, cryptocurrency, TLS 1.3

---

---

## Comparison & Use Cases

### Symmetric vs Asymmetric

| Aspect | Symmetric | Asymmetric |
|--------|-----------|-----------|
| **Keys Used** | One shared key | Public + Private key pair |
| **Speed** | Very fast (1000x faster) | Slow (high computational overhead) |
| **Key Management** | Hard (must securely share same key) | Easy (public key can be shared openly) |
| **Key Size for Same Security** | 256 bits | 2048-4096 bits (RSA) or 256 bits (ECC) |
| **Authentication** | No (only confidentiality) | Yes (can verify sender) |
| **Non-Repudiation** | No | Yes (sender can't deny signing) |
| **Use Case** | Bulk data, large files, high-speed | Key exchange, signatures, small data |
| **Examples** | AES, ChaCha20, DES, 3DES | RSA, ECC, DSA, Diffie-Hellman |
| **Encryption** | Yes | Limited (usually for key exchange) |
| **Digital Signatures** | No | Yes (primary use) |

### Hybrid Approach (Used in Real World)

Most modern systems use **both**:

**Example: HTTPS/TLS**
1. Client and server use **RSA/ECC (asymmetric)** to securely exchange a symmetric key
2. Then use that symmetric key with **AES (symmetric)** to encrypt actual data transfer
3. Benefits: Security of asymmetric + speed of symmetric

**Why**: Asymmetric is slow; symmetric is fast. Exchange keys with asymmetric, transmit data with symmetric.

---

## When to Use What

### Use Symmetric Encryption When:
- You need **speed** (bulk data, streaming)
- You can securely share the key in advance
- Encrypting files, database, backups
- No need for authentication
- **Examples**: File encryption, VPN data transfer, database encryption

### Use Asymmetric Encryption When:
- You need **authentication** or **non-repudiation**
- Parties haven't met before (can't share secret key)
- Need digital signatures
- Public key infrastructure is available
- **Examples**: HTTPS handshake, email signing, blockchain, digital certificates

### Use ECC Instead of RSA When:
- You have mobile/IoT devices (limited power, storage)
- You need faster operations
- You want smaller key sizes
- You're building modern systems (post-quantum concerns)

### Use AES When:
- You need symmetric encryption (it's the standard)
- Government/enterprise requirement
- NIST approval required
- Speed with security

### Use ChaCha20 When:
- You don't have AES hardware support (ARM processors, older CPUs)
- Building VPNs or modern protocols
- Mobile applications
- Real-time encryption needed

---

## Real-World Applications

### Banking & Finance
- **ATM/Payment Cards**: 3DES (legacy, transitioning to AES)
- **Wire Transfers**: AES encryption
- **Public Keys**: RSA for certificate authorities

### Web Security (HTTPS)
- **Handshake**: RSA or ECC key exchange
- **Data Transfer**: AES encryption
- **Modern**: TLS 1.3 uses ECC and ChaCha20/AES

### Email & Messaging
- **Encryption**: AES
- **Signatures**: RSA, DSA, or ECDSA
- **Key Exchange**: Diffie-Hellman or ECDH
- **WhatsApp/Signal**: ChaCha20-Poly1305 or AES-GCM

### Blockchain & Cryptocurrency
- **Bitcoin**: ECDSA with Curve25519
- **Ethereum**: ECC
- **Signatures**: EdDSA variants

### VPNs
- **OpenVPN**: AES (traditional)
- **WireGuard**: ChaCha20-Poly1305 (modern)
- **Key Exchange**: Diffie-Hellman or ECDH

### Government & Military
- **Standard**: AES (NSA approved)
- **Top Secret**: AES-256

### SSH & Remote Access
- **Key Generation**: RSA (older) or Ed25519 (modern)
- **Signatures**: ECDSA or EdDSA
- **Session Encryption**: AES

---

## Algorithms NOT Used Much (Why)

| Algorithm | Why It's Not Used |
|-----------|------------------|
| **DES** | 56-bit key too short, cracked easily |
| **3DES** | Slow, deprecated by NIST, replaced by AES |
| **RC4** | Multiple vulnerabilities, RFC 7465 prohibits in TLS |
| **Blowfish** | 64-bit block size too small, use Twofish or AES instead |
| **IDEA** | 64-bit blocks, older, replaced by AES |
| **Twofish** | AES won competition; not standardized |
| **MD5/SHA-1** | Cryptographically broken, use SHA-256 |
| **Diffie-Hellman** | Vulnerable to man-in-the-middle without authentication; use ECDH |
| **DSA** | Slower than ECDSA; being phased out (FIPS 186-5) |

---

## Key Takeaways for Interviews

### Quick Answers to Common Questions

**Q: AES vs RSA?**
A: AES is symmetric (fast, bulk data); RSA is asymmetric (slow, key exchange, signatures). Use both together.

**Q: How does encryption work?**
A: Plaintext + Key → Algorithm → Ciphertext. Reverse with correct key.

**Q: Why not use RSA for all data?**
A: It's too slow (100-1000x slower than AES). Use RSA to exchange a symmetric key, then use that for bulk encryption.

**Q: Is my data safe with AES?**
A: Yes. AES-256 is considered unbreakable with current technology.

**Q: What about quantum computers?**
A: They could break RSA and ECC. ECC is more resistant than RSA. Post-quantum algorithms being researched.

**Q: When should I use ECC?**
A: When efficiency matters (mobile, IoT), or modern systems. It gives same security as RSA with smaller keys.

**Q: How many bits should a key be?**
A: Symmetric: 128-256 bits (256 for high security). Asymmetric (RSA): 2048-4096 bits. ECC: 256+ bits.

**Q: Why use digital signatures?**
A: Prove message came from you (authentication) and wasn't modified (integrity). Non-repudiation.

**Q: DES or 3DES in 2024?**
A: Never. Both deprecated. Use AES only.

**Q: Is HTTPS safe?**
A: Yes. Modern TLS 1.3 uses ECC + ChaCha20 or AES-GCM. Very secure.

---

## Quick Reference: Algorithm Recommendations

### Current Best Practice (2024)

| Purpose | Algorithm | Key Size |
|---------|-----------|----------|
| **Data Encryption** | AES | 256 bits |
| **Stream (No HW Support)** | ChaCha20 | 256 bits |
| **Key Exchange** | ECDH (Curve25519) | 256 bits |
| **Digital Signatures** | Ed25519 or ECDSA | 256 bits |
| **Certificate Authority** | RSA or ECDSA | 2048 bits (RSA) or 256 bits (ECC) |
| **Legacy Support** | AES + RSA | 256 bits (AES), 2048 bits (RSA) |

---

## Interview Checklist

Before your interview, ensure you can:

✅ Explain symmetric vs asymmetric encryption clearly  
✅ Describe how AES works (key expansion, rounds, operations)  
✅ Explain RSA key generation, encryption, decryption  
✅ Discuss when to use each encryption type  
✅ Know which algorithms are deprecated and why  
✅ Understand digital signatures and their purpose  
✅ Explain how TLS/HTTPS uses both symmetric and asymmetric  
✅ Discuss key sizes and security levels  
✅ Know the limitations of current algorithms (quantum threats)  
✅ Give real-world examples for each algorithm

---

**Good luck with your interview!**