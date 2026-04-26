+++
author = "Anubhav Gain"
title = "Encryption Demystified: What Every Engineer Needs to Know"
date = "2026-03-23"
description = "You use TLS every day. You probably store passwords with bcrypt. But do you know why these work — and what breaks them? The concepts every engineer must understand."
tags = ["encryption", "cryptography", "security", "tls", "engineering"]
categories = ["security", "engineering"]
+++

Cryptography is infrastructure. Framework defaults, managed certificates, and library wrappers handle it invisibly — until the day you must make a deliberate choice with no default. That is when partial understanding becomes a real liability.

<!--more-->

## Symmetric vs Asymmetric: The Fundamental Tradeoff

Symmetric encryption uses one shared key for both directions. AES is the standard — fast, hardware-accelerated, ideal for bulk data. The problem is key distribution: both parties need to agree on that key, which an untrusted channel cannot facilitate.

Asymmetric encryption uses a linked key pair: data encrypted with the public key can only be decrypted by the private key. Anyone can encrypt; only you decrypt. The cost is speed — RSA and elliptic-curve operations are far slower than AES.

Protocols combine both. TLS encrypts sessions with a symmetric key; asymmetric cryptography negotiates that key at handshake time.

## AES in Practice: Mode Selection Is Everything

AES is a block cipher; the mode of operation determines actual security.

**ECB** encrypts each block independently. Identical plaintext produces identical ciphertext — patterns survive. Encrypt a bitmap in ECB and the shapes remain visible. Never use it.

**CBC** chains blocks via XOR with the previous ciphertext, eliminating patterns. Requires a random IV; provides confidentiality but not integrity. Padding oracle attacks (POODLE, BEAST) exploit CBC error responses. Use only with a separate MAC.

**GCM** is the standard: authenticated encryption in one pass with a 16-byte verification tag. Tamper with the ciphertext and decryption fails. Hard rule: the IV must never be reused under the same key. Use a random 96-bit IV per operation. Default to AES-256-GCM.

## RSA and Elliptic Curve: Why Smaller Keys Are Not Weaker

RSA security rests on integer factoring. That problem scales subexponentially: 2048-bit RSA provides about 112 bits of security; 3072 bits buys 128.

ECC uses the elliptic curve discrete logarithm problem — harder to attack, closer to exponential scaling. A 256-bit ECC key matches 3072-bit RSA security in a fraction of the size, with faster handshakes and lower CPU cost. Prefer X25519 for key agreement and Ed25519 for signatures; both have clean security proofs and constant-time implementations.

## The TLS Handshake: What Actually Happens

TLS 1.3 completes in one round trip. The client sends cipher suites and an ephemeral Diffie-Hellman public value. The server replies with its own key share and certificate; both sides derive session keys from the DH exchange.

The critical property is forward secrecy. Session keys come from ephemeral DH values generated fresh per connection and discarded after use. The server's private key only proves identity — it never touches key derivation. Record traffic today, steal the key later: past sessions remain safe. Without ephemeral DH, one stolen key decrypts everything ever captured. ECDHE is non-negotiable.

## Password Hashing: The Algorithm Is the Entire Answer

Passwords are secrets to verify, not encrypt. You should never recover a plaintext password from storage — only confirm a match.

MD5 and SHA-1 are fast, which is fatal here. A GPU computes billions of SHA-1 hashes per second; a leaked database cracks in hours against any wordlist.

**bcrypt** is intentionally slow with a cost factor you raise as hardware improves. It handles salting automatically. Mind its 72-byte input limit.

**Argon2id** is the current recommendation. Its memory cost is the key advance: more threads require more RAM, making GPU and ASIC cracking expensive. Use OWASP defaults (64 MB, 3 iterations, parallelism 4) and scale up.

## Common Crypto Mistakes in Production Code

**Hardcoded or reused IVs.** IVs must be unique per encryption under a given key. Hardcoding breaks every mode's security properties.

**ECB mode.** Appears when code specifies `AES/ECB/PKCS5Padding` without understanding modes. Use GCM.

**Rolling your own crypto.** Combining primitives incorrectly creates invisible vulnerabilities. Use libsodium, BoringSSL, or your language's standard library with established constructs: NaCl secretbox, TLS, JWE.

**MD5 or SHA-1 for passwords.** Still in legacy codebases. No justification makes a fast hash acceptable for password storage.

**Unauthenticated encryption.** CBC without a MAC allows silent tampering. GCM and ChaCha20-Poly1305 are required when integrity matters.

## Post-Quantum Cryptography: When to Pay Attention

Shor's algorithm on a large quantum computer would break RSA and ECC entirely. That hardware does not exist yet, but "harvest now, decrypt later" is real: adversaries record ciphertext today, intending to decrypt once quantum hardware arrives. Data requiring decade-long confidentiality is already at risk.

NIST finalized its first post-quantum standards in 2024: ML-KEM (Kyber) for key encapsulation and ML-DSA (Dilithium) for signatures.

Care now if you build long-lived or critical infrastructure. For most engineers, the first step is TLS 1.3 with ECDHE and watching your library vendor's PQC roadmap.

## Using Crypto Correctly Without Being a Cryptographer

You do not need to implement primitives — only choose the right ones and use them correctly:

- TLS 1.3 everywhere; never disable certificate verification.
- AES-256-GCM with a random 96-bit IV, never reused.
- Argon2id for passwords; bcrypt where Argon2 is unavailable.
- Ed25519 or X25519 for asymmetric operations.
- HKDF or PBKDF2 for key derivation; never derive keys directly from a raw password.
- Use a well-audited library; never combine raw primitives yourself.

The gap between using crypto and understanding it shows up in the IV you forgot to randomize, the ECB mode left in place, and the SHA-1 hash storing passwords for eight years. The primitives are well-understood; the mistakes are avoidable; getting this right is the baseline for systems that deserve trust.
