# Lab 3: Data Protection — Encryption & Key Management
Course: IKB42603 Cloud Computing Security Essentials

Topic: At-rest and in-transit encryption, envelope encryption and cryptographic erasure — OpenSSL & LocalStack KMS

Environment: OpenSSL, Docker (nginx for TLS) and LocalStack on localhost:4566

Name: Nurlisa Sofiya binti Mahadzir

## Lab Summary // Objective

This lab demonstrates data protection across confidentiality, integrity and key management using OpenSSL and LocalStack KMS:

- Symmetric encryption (AES-256) protects data at rest with a single shared key, and decryption is verified.
- Asymmetric encryption (RSA-2048) shows the public/private key roles and provides digital signatures for origin and integrity.
- TLS protects data in transit, with the difference between plaintext HTTP and encrypted HTTPS observed.
- A Key Management Service (KMS) manages master keys and implements envelope encryption, so only the master key needs hardware-grade protection.
- Per-tenant keys and cryptographic erasure make deleted data provably unrecoverable.
- Hashing verifies data integrity, and a hash-chained record is tamper-evident.

## Architecture Diagram

The following architecture models data protection across the at-rest, in-transit, and integrity dimensions, with LocalStack KMS managing the keys.

```text
                   Data Protection Architecture (Lab 3)
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
   Data at Rest               Data in Transit              Integrity
   AES-256 (Task 1)           TLS / HTTPS (Task 3)         SHA-256 hash chain
   RSA-2048 (Task 2)                  │                        (Task 7)
        │                        nginx :8443
        │                    (self-signed cert)
   LocalStack KMS (Tasks 4–6)
   CMK-A ──wraps──▶ data key (DEK)
   envelope encryption + cryptographic erasure
```

## Evidence Folder

All screenshots used for this report are stored in the `evidence lab 3` folder.

| **Evidence File** | **Purpose** |
|---|---|
| `dockerdesktop.png` & `setup docker.png` | Docker Desktop installed and configured for the first time — Docker was previously used only from the command line |
| `lab3task1a.png` – `lab3task1b.png` | AES-256 encryption of `record.txt` and the `MATCH: decryption successful` confirmation |
| `lab3task2.png` | RSA key pair generation, public-key encryption, and the `Verified OK` signature output |
| `lab3task3a.png` – `lab3task3d.png` | `curl -k https://localhost:8443/record.txt` output confirming the record is served over TLS |
| `lab3task4a.png` – `lab3task4b.png` | `kms create-key` output for the tenant-A CMK, including the captured `KeyId` |

## Overview

This lab is split into two sessions:

- **Session A (Week 5):** Encryption fundamentals by hand — symmetric (AES) and asymmetric (RSA) cryptography, plus TLS for data in transit (Tasks 1–3).
- **Session B (Week 6):** Key management at scale — LocalStack KMS, envelope encryption, per-tenant keys, cryptographic erasure and integrity hashing (Tasks 4–7).

Session A builds the cryptographic fundamentals by hand. Session B shows how a cloud KMS manages keys at scale and enables provable deletion.

**Security tip:** Encryption is only as strong as its key management. Watch carefully in Session B where the keys live — that is the real security control, not the algorithm.

---
## Session A (Week 5) — Encryption Fundamentals

## Environment Setup

```bash
# 1. Confirm OpenSSL is available (pre-installed on macOS/Linux; use Git Bash or WSL on Windows)
openssl version
```
**Why:** OpenSSL provides the `enc`, `genrsa`, `pkeyutl`, `req` and `dgst` commands used across both sessions.
**Result:** The OpenSSL version is printed, confirming the toolchain is ready.

**Docker Desktop setup:** Docker Desktop was installed and configured for the first time for this lab — before this, Docker was only used from the command line (e.g. `docker run` in Git Bash). Docker Desktop provides the container engine (via WSL2) that runs the nginx TLS server in Task 3 and LocalStack in Session B.

```bash
# 2. Confirm Docker is now available through Docker Desktop
docker --version
```
**Why:** Confirms the Docker Desktop engine is installed and reachable before any container is launched.
**Result:** The Docker version is printed.

```bash
# 3. Create the sensitive record that will be protected throughout the lab
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt
```
**Why:** The same realistic sensitive record is used for symmetric encryption (Task 1), asymmetric encryption (Task 2) and TLS serving (Task 3), making the confidentiality comparisons concrete.
**Result:** `record.txt` contains the sample patient record.

Evidence: <div align="left">
<img alt="Docker Desktop first-time setup" src="evidence lab 3/dockerdesktop.png">
<img alt="Docker Desktop setup confirmation" src="evidence lab 3/setup docker.png">
</div>

---

## Task 1 — Symmetric Encryption (Data at Rest)

```bash
# Encrypt with AES-256 (you will be prompted for a passphrase = the key)
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc

# Prove it is unreadable
cat record.enc

# Decrypt back and confirm the round-trip
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```
**Why:** AES-256-CBC with PBKDF2 key derivation and a random salt encrypts the file with a single shared key. One key does both encryption and decryption — fast, but the key must be protected.
**Result:** `record.enc` is unreadable ciphertext, and `diff` confirms the decrypted file matches the original (`MATCH: decryption successful`).

What is the key-distribution problem with symmetric encryption, and why does it matter for the cloud?

**Answer:** The key-distribution problem is that both the sender and receiver must possess the same secret key before any encrypted communication can occur, and that key must be transmitted through a separate, secure channel. If the key is intercepted during delivery, an attacker can decrypt all past and future messages.

This matters for the cloud because:
- **Scale:** A cloud environment may have thousands of instances, services and tenants — each needing keys. Sharing symmetric keys pairwise becomes unmanageable.
- **No physical security:** Cloud storage and networks are virtualized and multi-tenant; there is no single trusted courier to deliver keys.
- **Key lifecycle:** Keys must be rotated, revoked and destroyed — operations that are hard to coordinate when every party holds a copy.
- **Compliance:** Regulations (PDPA, PCI-DSS) require centralised key control; distributing keys manually violates audit requirements.

This is why cloud platforms use a Key Management Service (KMS) — it solves the distribution problem by keeping the master key in a hardware-protected module and letting services request encryption/decryption through API calls instead of sharing raw keys.

Evidence: <div align="left">
<img alt="AES-256 encryption and decryption with MATCH confirmation" src="evidence lab 3/lab3task1a.png">
<img alt="Decrypted output matching the original record" src="evidence lab 3/lab3task1b.png">
</div>

---

## Task 2 — Asymmetric Encryption & Digital Signatures

```bash
# Generate a 2048-bit RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# Encrypt with the PUBLIC key, decrypt with the PRIVATE key
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt

# Sign with the PRIVATE key; verify with the PUBLIC key
openssl dgst -sha256 -sign private.pem -out record.sig record.txt
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```
**Why:** Anyone can encrypt with the public key; only the private key can decrypt. Signatures prove origin and integrity — the private key signs, and anyone with the public key can verify.
**Result:** `record.rsa` decrypts back to the original, and the signature verifies with `Verified OK`.

**Note:** Notice how the roles reverse: encryption uses the public key, signing uses the private key. This is the basis of PKI and TLS.

Evidence: <div align="left">
<img alt="RSA key pair, public-key encryption and Verified OK signature output" src="evidence lab 3/lab3task2.png">
</div>

---

## Task 3 — Encryption in Transit (TLS)

```bash
# Generate a self-signed certificate for localhost
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
    -days 7 -nodes -subj '/CN=localhost'

# Serve HTTPS on port 8443 using a small container
docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx

# Connect over TLS (-k accepts the self-signed certificate)
curl -k https://localhost:8443/record.txt
```
**Why:** The self-signed certificate lets nginx serve the record over an encrypted HTTPS channel, and `-k` tells curl to accept the self-signed certificate for the local test.
**Result:** `curl -k https://localhost:8443/record.txt` returns the record over TLS.

**Security tip:** Compare mentally with plain HTTP: over HTTP the record would travel in clear text and any on-path attacker could read it (eavesdropping, Week 3). TLS makes intercepted traffic unreadable.

**Note:** End of Session A. Stop the TLS container (`docker stop tls`). Keep `record.enc`, the RSA keys, and all outputs for the report.

Evidence: <div align="left">
<img alt="curl -k over HTTPS returning the record" src="evidence lab 3/lab3task3a.png">
<img alt="TLS connection details" src="evidence lab 3/lab3task3b.png">
<img alt="nginx serving the record over HTTPS" src="evidence lab 3/lab3task3c.png">
<img alt="TLS verification output" src="evidence lab 3/lab3task3d.png">
</div>

*End of Session A. The AES ciphertext, RSA keys and TLS outputs from Tasks 1–3 are kept as evidence; Session B moves key management into a cloud KMS.*

---

## Session B (Week 6) — Key Management, Envelope Encryption & Erasure

Start LocalStack if it is not running (see Lab 1). All KMS commands below target it through the endpoint variable.

## Task 4 — Create and Use a KMS Master Key

```bash
EP='--endpoint-url=http://localhost:4566'

# Create a customer master key (CMK) and capture its KeyId
aws $EP kms create-key --description 'CCSE tenant-A master key'
# Copy the KeyId from the output into KEY_A below
KEY_A=<PASTE_KEYID>

# Encrypt a small secret directly with KMS
aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
    --query CiphertextBlob --output text
```
**Why:** The customer master key is the cloud analogue of the passphrase from Task 1, but it never leaves the KMS — encryption and decryption happen through KMS API calls.
**Result:** `kms create-key` returns a `KeyId` (captured into `KEY_A`), and `kms encrypt` returns a base64 ciphertext blob for `hello`.

Evidence: <div align="left">
<img alt="KMS create-key output with the captured KeyId" src="evidence lab 3/lab3task.png">
<img alt="KMS encrypt output" src="evidence lab 3/lab3task.png">
</div>

---

## Task 5 — Envelope Encryption

```bash
# 5.1 Ask KMS for a data key (returns plaintext + encrypted versions)
aws $EP kms generate-data-key --key-id $KEY_A --key-spec AES_256 \
    --query '[Plaintext,CiphertextBlob]' --output text
# Save column 1 as datakey.b64 (plaintext) and column 2 as datakey.enc (wrapped)

# 5.2 Encrypt the big file locally with the PLAINTEXT data key
base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc \
    -pass file:./datakey.bin

# 5.3 Destroy the plaintext data key from disk — keep only the wrapped copy
rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```
**Why:** For large data you do not encrypt with the master key directly. You generate a data key, encrypt the data locally with it, and store the data key wrapped by the master key. This is envelope encryption.
**Result:** `record.env.enc` is encrypted with the one-time data key, and only the KMS-wrapped `datakey.enc` remains on disk.

**Note:** To read the data later you send `datakey.enc` back to KMS (`kms decrypt`) to unwrap it, use it, then discard it. Only the small master key ever needs hardware-grade protection.

Evidence: <div align="left">
<img alt="Data key generation, local AES encryption and wrapped data key" src="evidence lab 3/task5.png">
</div>

# kena tambah confirmation +undo delete"
---

## Task 6 — Per-Tenant Keys & Cryptographic Erasure

```bash
# A separate key for tenant B
aws $EP kms create-key --description 'CCSE tenant-B master key'
KEY_B=<PASTE_KEYID>

# Schedule deletion of tenant A's key (minimum window)
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7

# Disable it immediately to simulate erasure
aws $EP kms disable-key --key-id $KEY_A

# Attempt to unwrap tenant A's data key now — it should FAIL
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```
**Why:** A second master key for tenant B shows that one tenant's key cannot unwrap another tenant's data. Deleting tenant-A's key demonstrates cryptographic erasure — deletion becomes provable because the key no longer exists.
**Result:** The `kms decrypt` attempt fails after the key is disabled and scheduled for deletion, proving `record.env.enc` is permanently unrecoverable.

**Caution:** Once the key that wrapped the data key is gone, `record.env.enc` is just noise — no one, not even the provider, can decrypt it. This is why per-object/per-tenant keys make deletion provable (Week 4).

Evidence: <div align="left">
<img alt="Failed kms decrypt after key erasure" src="evidence lab 3/task6.png">
</div>

---

## Task 7 — Integrity & Tamper-Evidence

```bash
# Fingerprint the file
sha256sum record.txt

# Tamper with a copy and show the hash changes
cp record.txt tampered.txt; echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt

# Hash chain: each entry includes the previous hash (tamper-evident log)
PREV=0
for line in 'login ok' 'file read' 'export data'; do \
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
  echo "$line | $PREV"; done
```
**Why:** Encryption protects confidentiality; hashing protects integrity. Any modification changes the SHA-256 fingerprint, and chaining each log entry to the previous entry's hash makes earlier tampering detectable.
**Result:** The two hashes differ after tampering, and the hash chain shows each entry embedding the previous entry's hash.

Evidence: <div align="left">
<img alt="Differing SHA-256 hashes and the tamper-evident hash chain" src="evidence lab 3/task7.png">
</div>

---

## Verification Command

```bash
aws --endpoint-url=http://localhost:4566 kms list-keys
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```
**Why:** `kms list-keys` confirms which master keys remain (tenant-A's key should be disabled or pending deletion), and re-verifying the RSA signature confirms the integrity chain from Task 2 still holds.
**Result:** `kms list-keys` shows the surviving keys, and the signature verifies with `Verified OK`.

Evidence: <div align="left">
<img alt="kms list-keys and RSA signature re-verification" src="evidence lab 3/lab3verify.png">
</div>

---

## Short-Answer Questions

**Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.**

| Aspect | Symmetric (e.g. AES) | Asymmetric (e.g. RSA) |
|---|---|---|
| **Speed** | Fast — suited to bulk data | Slow — suited to small payloads |
| **Keys** | One shared key | Public/private key pair |
| **Key distribution** | Key must be shared securely out of band | Public key can be distributed freely; only the private key is secret |
| **Typical use** | Encrypting data at rest and bulk data in transit | Key exchange, digital signatures, TLS handshakes |

**Q2. Why is key management described as the weakest link, not the algorithm?**

Encryption is only as strong as its key management. The algorithms are public and standard, but the key is the one secret that protects everything — if keys are shared poorly, stored insecurely, reused too long, or not rotated and destroyed, an attacker can obtain the key (or the wrapped copy) and the strongest algorithm becomes useless. In the cloud, where storage is virtualized and shared, where the keys live and who can use them is the real security control.

**Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.**

Envelope encryption encrypts the data with a random one-time data key, then encrypts (wraps) that data key with the master key held in the KMS. The wrapped data key is stored alongside the ciphertext. Only the small master key needs hardware-grade protection because it is the only key that can unwrap data keys — the data keys themselves are short-lived, used once, and discarded.

**Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?**

In the cloud, physical storage is virtualized, shared and replicated, so overwriting the underlying bytes is impractical and unreliable. Cryptographic erasure instead destroys the key that encrypted the data. Once the key is gone, every copy of the ciphertext — including backups — becomes unreadable, so deletion is provable even though the bytes still exist.

**Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)?**

Each log entry incorporates the hash of the previous entry, so every entry is cryptographically bound to the one before it. Modifying, inserting or deleting an earlier entry changes its hash, which breaks every subsequent link in the chain — tampering is immediately detected by recomputing the hashes.

---

## Cleanup & Teardown

```bash
# Stop the TLS container used in Task 3
docker stop tls 2>/dev/null
```
```bash
# Remove the temporary crypto artifacts
rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt
```
```bash
# Stop and remove LocalStack
docker stop localstack && docker rm localstack
```

## Security Best-Practices Checklist

- [X] Data encrypted at rest (AES) and decryption verified.
- [X] Asymmetric keys used correctly (encrypt with public, sign with private).
- [X] Data protected in transit with TLS.
- [X] Envelope encryption used; plaintext data key not left on disk.
- [X] Per-tenant keys used; cryptographic erasure demonstrated.
- [X] Integrity verified with hashing / hash chain.

---

## Conclusion

This lab demonstrated that data protection is the combination of cryptography and key management: symmetric and asymmetric algorithms protect confidentiality, TLS protects data in transit, and a KMS provides key management and provable deletion at cloud scale.

### Session A — Encryption Fundamentals
- AES-256 encrypted `record.txt` at rest, and decryption was verified with `MATCH: decryption successful`.
- RSA-2048 encrypted with the public key and signed with the private key; the signature verified with `Verified OK`.
- Serving the record over HTTPS proved that TLS keeps in-transit data unreadable to on-path attackers.

### Session B — Key Management, Envelope Encryption & Erasure
- LocalStack KMS created and used a customer master key instead of a local passphrase.
- Envelope encryption protected the record with a one-time data key, wrapped by the master key.
- Per-tenant keys and key deletion made the tenant-A data permanently unrecoverable (decrypt failed).
- SHA-256 hashing detected tampering, and the hash chain made the log tamper-evident.

### Key Takeaway
Encryption is only as strong as its key management — the algorithm is public, but the key is the real secret. Protect keys, rotate them, and destroy them provably when they are no longer needed.

---

