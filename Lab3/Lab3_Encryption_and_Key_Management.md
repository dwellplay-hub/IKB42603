# Lab 3 — Encryption and Key Management

Student: MUHAMMAD DANISH ISYRAQ

Course: IKB42603 Cloud Computing Security Essentials

Topic: Data Protection, At-Rest & In-Transit Encryption, Envelope Encryption, KMS & Cryptographic Erasure

## Lab Summary / Objective

This lab demonstrates core data protection techniques: symmetric (AES-256) and asymmetric (RSA-2048) encryption, digital signatures, TLS for encryption-in-transit, Key Management Service (KMS) operations including Customer Master Key (CMK) creation, envelope encryption (data key generation and local encryption), per-tenant key management, cryptographic erasure, and integrity / tamper-evidence via hash chains.

## Evidence Mapping Table

| Screenshot | Task | Description |
|---|---:|---|
| ![Task 1](Evidence/Task%201.png) | Session A — Task 1 (AES Match) | Symmetric AES-256 encrypt/decrypt and plaintext/ciphertext verification |
| ![Task 2](Evidence/Task%202.png) | Session A — Task 2 (RSA Verify) | RSA-2048 keypair, encrypt/decrypt, signature generation and verification |
| ![Task 3](Evidence/Task%203.png) | Session A — Task 3 (TLS curl) | Self-signed TLS certificate and successful `curl -k https://` test to NGINX HTTPS container |
| ![Task 4](Evidence/Task%204.png) | Session B — Task 4 (KMS Keys/Envelope) | KMS CMK creation and direct KMS encrypt of a short string |
| ![Task 5.1](Evidence/Task%205.1.png) | Session B — Task 5 (Envelope encryption) | `generate-data-key` output and local AES encryption of `record.txt` |
| ![Task 5.2](Evidence/Task%205.2.png) | Session B — Task 5 (Envelope encryption) | Plaintext data key destruction / zeroization / not persisted |
| ![Task 5.3](Evidence/Task%205.3.png) | Session B — Task 5 (Envelope encryption) | Encrypted record (`record.enc`) and wrapped data key (`record.key.blob`) stored |
| ![Task 6](Evidence/Task%206.png) | Session B — Task 6 (Per-Tenant Keys & Erasure) | Tenant B key created, Tenant A key scheduled for deletion/disabled, KMS decrypt failure proof |
| ![Task 7](Evidence/Task%207.png) | Session B — Task 7 (Hash Chain) | SHA-256 hash comparison of clean and modified files and hash-chained log output |
| ![Verification Command](Evidence/Verification%20Command.png) | Verification | `openssl dgst -sha256 -verify public.pem -signature record.sig record.txt` command output |

---

## Session A — Encryption Fundamentals

### Task 1 — Symmetric Encryption (AES-256)

Objective: Demonstrate AES-256 encryption at rest, decrypt locally, and verify plaintext/ciphertext differences.

Commands (example):

```bash
# Generate a 256-bit key
openssl rand -out key.bin 32

# Encrypt with AES-256-CBC
openssl enc -aes-256-cbc -salt -in record.txt -out record.enc -pass file:./key.bin

# Decrypt
openssl enc -d -aes-256-cbc -in record.enc -out record.dec -pass file:./key.bin

# Verify integrity: plaintext should match decrypted output
diff record.txt record.dec

# Verify ciphertext != plaintext
hexdump -C record.txt | head
hexdump -C record.enc | head
```

Result: `record.dec` matches `record.txt`; ciphertext is non-readable binary. See evidence: ![Task 1](Evidence/Task%201.png)

---

### Task 2 — Asymmetric Encryption & Digital Signatures (RSA 2048)

Objective: Generate RSA-2048 keypair, demonstrate public-key encryption and private-key decryption, generate SHA-256 signature and verify with public key.

Commands (example):

```bash
# Generate RSA-2048 private key
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048

# Extract public key
openssl rsa -pubout -in private.pem -out public.pem

# Encrypt with public key
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa.enc

# Decrypt with private key
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa.enc -out record.rsa.dec

# Create SHA-256 signature
openssl dgst -sha256 -sign private.pem -out record.sig record.txt

# Verify signature (explicit verification command required by lab)
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

Verification: The `openssl dgst -sha256 -verify ...` command returns `Verified OK` when the signature matches. See evidence: ![Task 2](Evidence/Task%202.png)

---

### Task 3 — Encryption in Transit (TLS self-signed cert, NGINX HTTPS container)

Objective: Create a self-signed certificate, configure an HTTPS server (NGINX in a container), and verify encrypted transport with `curl`.

Commands (example):

```bash
# Generate self-signed cert for localhost
openssl req -x509 -newkey rsa:2048 -sha256 -nodes -keyout key.pem -out cert.pem -days 365 -subj "/CN=localhost"

# (Example) Launch nginx container with cert mounted and port 8443 mapped
docker run --name nginx-https -d -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/ssl/certs/cert.pem:ro \
  -v $(pwd)/key.pem:/etc/ssl/private/key.pem:ro nginx

# Test via curl (allowing self-signed cert)
curl -k https://localhost:8443/
```

Result: TLS handshake completes; HTTP response served over HTTPS. See evidence: ![Task 3](Evidence/Task%203.png)

---

## Session B — Key Management, Envelope Encryption & Erasure

### Task 4 — KMS Master Key creation (CMK) and direct KMS encryption

Objective: Create a CMK in the local KMS (e.g., LocalStack) and use it to encrypt a short secret string.

Commands (example using LocalStack endpoint):

```bash
# Create CMK
CMK_ID=$(aws --endpoint-url=http://localhost:4566 kms create-key --description "Lab3 CMK for MUHAMMAD DANISH ISYRAQ" --query KeyMetadata.KeyId --output text)

# Encrypt a short string
aws --endpoint-url=http://localhost:4566 kms encrypt --key-id $CMK_ID --plaintext "my-secret-data" --query CiphertextBlob --output text | base64 --decode > secret.blob

# List keys (verification)
aws --endpoint-url=http://localhost:4566 kms list-keys
```

Result: CMK is created and KMS returns a ciphertext blob. See evidence: ![Task 4](Evidence/Task%204.png)

---

### Task 5 — Envelope Encryption (generate-data-key, local AES encryption, destroy plaintext data key)

Objective: Use KMS to generate a data key, use the plaintext data key to encrypt a large record locally with AES, then securely discard the plaintext key and store only the encrypted data key.

Commands (example):

```bash
# Generate a data key (returns Plaintext and CiphertextBlob)
aws --endpoint-url=http://localhost:4566 kms generate-data-key --key-id $CMK_ID --key-spec AES_256 \
  --query '{Plaintext:Plaintext,CiphertextBlob:CiphertextBlob}' --output json > data_key.json

# Extract plaintext key (base64) and ciphertext blob
cat data_key.json | jq -r .Plaintext > data_key_plain.b64
cat data_key.json | jq -r .CiphertextBlob > data_key_enc.b64

# Decode plaintext key to binary and use to encrypt record.txt locally
base64 -d data_key_plain.b64 > data_key_plain.bin
openssl enc -aes-256-cbc -salt -in record.txt -out record.enc -pass file:./data_key_plain.bin

# Securely destroy plaintext copy
shred -u data_key_plain.bin || srm data_key_plain.bin || rm -f data_key_plain.bin

# Keep only: record.enc and data_key_enc.b64 (wrapped data key)
```

Notes: The plaintext data key is ephemeral and must not be stored. See evidence for generate-data-key and local encryption: ![Task 5.1](Evidence/Task%205.1.png) ![Task 5.2](Evidence/Task%205.2.png) ![Task 5.3](Evidence/Task%205.3.png)

---

### Task 6 — Per-Tenant Keys & Cryptographic Erasure

Objective: Demonstrate per-tenant CMKs, disable/schedule deletion on a tenant key (Tenant A), and show that KMS decrypt fails after key disablement (proof of cryptographic erasure for that tenant's data).

Commands (example):

```bash
# Create Tenant A and Tenant B CMKs
TENANT_A_KEY=$(aws --endpoint-url=http://localhost:4566 kms create-key --description "TenantA Key" --query KeyMetadata.KeyId --output text)
TENANT_B_KEY=$(aws --endpoint-url=http://localhost:4566 kms create-key --description "TenantB Key" --query KeyMetadata.KeyId --output text)

# Encrypt a small secret under Tenant A key
aws --endpoint-url=http://localhost:4566 kms encrypt --key-id $TENANT_A_KEY --plaintext "tenantA-secret" --query CiphertextBlob --output text | base64 --decode > tenantA.secret.blob

# Schedule deletion or disable Tenant A key (simulate erasure)
aws --endpoint-url=http://localhost:4566 kms schedule-key-deletion --key-id $TENANT_A_KEY --pending-window-in-days 7
# OR disable immediately
aws --endpoint-url=http://localhost:4566 kms disable-key --key-id $TENANT_A_KEY

# Attempt to decrypt with disabled/scheduled-for-deletion key (expected to fail)
aws --endpoint-url=http://localhost:4566 kms decrypt --ciphertext-blob fileb://tenantA.secret.blob || true
```

Result: After key disablement/scheduling deletion, KMS rejects decrypt operations for ciphertext wrapped by the disabled key. See evidence: ![Task 6](Evidence/Task%206.png)

---

### Task 7 — Integrity & Tamper-Evidence (SHA-256 hash chain)

Objective: Compute SHA-256 of a file, modify the file, demonstrate hash mismatch, and show a simple hash chain for a log sequence.

Commands (example):

```bash
# Compute initial hash
openssl dgst -sha256 -binary record.txt | xxd -p > record.sha256

# Modify the file (simulate tampering)
echo "tamper" >> record.txt

# Compute new hash and compare
openssl dgst -sha256 -binary record.txt | xxd -p > record.modified.sha256
diff record.sha256 record.modified.sha256 || echo "Hashes differ: tamper detected"

# Hash chain example (append previous hash into next entry calculation)
echo "entry1: $(openssl dgst -sha256 -hex entry1.txt)" > chain.log
PREV_HASH=$(awk '{print $2}' chain.log)
cat entry2.txt | (printf "%s" "$PREV_HASH"; cat) | openssl dgst -sha256 -hex >> chain.log
```

Evidence: SHA-256 mismatch and chained log output shown in ![Task 7](Evidence/Task%207.png)

---

## Verification Commands (explicitly required)

Run these to inspect keys and verify signatures:

```bash
aws --endpoint-url=http://localhost:4566 kms list-keys

openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

See screenshot of the `openssl dgst -sha256 -verify` output: ![Verification Command](Evidence/Verification%20Command.png)

---

## Short-Answer Questions (Q1–Q5)

Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

* Symmetric Encryption (e.g., AES-256):Uses a single shared key for both encryption and decryption. It is computationally fast and highly efficient. Key distribution is challenging because the secret key must be securely transmitted to all communicating parties without interception. Typical use includes bulk data encryption at rest, such as file systems, disk volumes, and databases.
* Asymmetric Encryption (e.g., RSA): Uses a mathematically linked key pair consisting of a public key (for encryption/verification) and a private key (for decryption/signing). It is significantly slower and computationally heavier than symmetric encryption. Key distribution is simple because the public key can be shared openly across untrusted networks. Typical use includes TLS handshakes, key exchange mechanisms, and digital signatures.

Q2. Why is key management described as the weakest link, not the algorithm?

* Modern cryptographic algorithms (such as AES-256 and RSA-2048) are mathematically sound and practically impossible to break through brute-force attacks. System compromise almost never occurs from breaking the math, but rather from vulnerabilities in the key lifecycle.

Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.

* Hardware Security Modules (HSMs) provide strict security boundaries but suffer from network latency and payload size constraints when encrypting large files directly. Envelope encryption limits HSM operations to protecting and wrapping the small master key and data keys, allowing large payloads to be encrypted locally at full wire speed while maintaining hardware-grade root security.

Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?

* In shared multi-tenant cloud storage, physical media is abstracted into virtualized blocks that are replicated, cached, and snapshotted across distributed infrastructure where tenants have no direct access to physical sectors to perform zero-overwrites (dd or disk shredding). Cryptographic erasure achieves provable deletion by permanently destroying or revoking the KMS master key that decrypts the wrapped data key. Without the root key, all associated ciphertext across every physical storage block and backup replica instantly becomes permanently undecipherable random noise, rendering the data unrecoverable.

Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)?

* A hash chain builds a cryptographically linked sequence where each log entry’s hash is computed over both its own log text and the cryptographic hash of the preceding record.

---

## Security Best-Practices Checklist

- [x] Use strong, modern algorithms (AES-256 for symmetric, RSA-2048 or stronger for asymmetric).
- [x] Keep plaintext data keys ephemeral; never persist unwrapped keys.
- [x] Use envelope encryption to limit HSM/KMS operations to small keys.
- [x] Separate tenant keys (per-tenant CMKs) to enable targeted cryptographic erasure.
- [x] Use TLS for in-transit protection; prefer validated public certs in production.
- [x] Maintain integrity checks (SHA-256 hash chains) for tamper-evidence.

---

## Conclusion

This lab demonstrates practical, applied data protection: symmetric encryption for efficient at-rest protection, asymmetric crypto for secure key exchange and signatures, TLS to protect data-in-transit, and KMS-backed envelope encryption to combine strong root-of-trust with high-performance local encryption. Per-tenant keys and cryptographic erasure provide scalable ways to meet deletion guarantees in cloud platforms. Hash chains and signature verification provide tamper-evidence and non-repudiation for sensitive logs and records. Together, these techniques form a layered defense for protecting confidentiality, integrity, and availability of tenant data.

