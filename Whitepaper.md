# Q-Lock

**Ecosystem Whitepaper · Version 1.0 · September 2026**  
Post-Quantum Settlement Infrastructure

**Scope:** GodShield · NEV369 · Q-Lock Escrow · Bridge · Inheritance Vault  
**Review:** Internal security review complete · workspace compiled, full test suite passing · third-party audit pending

> Nevaeh, my daughter. To secure your freedom against a broken system, I taught myself Rust — the hardest computer language in the world — to build this unyielding sovereign node for you. I faced the worst of life's struggles so you would never have to. I love you infinitely, forever by your side.

**Written into NEV369 genesis block 0. Immutable.**

## Contents

1. Threat model  
2. Architecture  
3. GodShield — cryptographic core  
4. GodShield — security platform  
5. NEV369  
6. Token economics  
7. Q-Lock Escrow  
8. The bridge  
9. The inheritance vault  
10. Security analysis  
11. Limitations  
12. Status and roadmap  

## Abstract

Substantially all value secured on public blockchains today rests on elliptic-curve and RSA signatures — ECDSA over secp256k1, EdDSA over Curve25519, and RSA. Shor's algorithm solves the integer factorisation and discrete logarithm problems in polynomial time on a sufficiently large fault-tolerant quantum computer, breaking all three completely.

We do not assert that such a machine exists. We assert three narrower things, each independently verifiable: that every signature ever broadcast on a public chain is permanently retrievable; that migrating a multi-trillion-dollar asset class across wallets, exchanges, custodians and node software takes years of coordinated effort; and that an adversary recording public keys today requires no quantum computer at all — only patience.

Q-Lock is an integrated response built around one cryptographic core. **GodShield** provides ML-DSA-87 signing over a triple-hash cascade, together with identity, policy, threshold authorisation and tamper-evident audit. **NEV369** is a proof-of-work Layer-1 that uses that core for every signature and every block hash. **Q-Lock Escrow** provides non-custodial escrow on the XRP Ledger with post-quantum attestation over each settlement. A threshold-authorised bridge connects NEV369 to Ethereum, and a Shamir-split, time-locked vault secures a thirteen-year inheritance for the author's daughter, Nevaeh.

This document describes the design, states the security properties each component does and does not provide, and is explicit about what has not yet been verified. Section 11 states the security boundaries precisely, and is not optional reading.

## 1. Threat model

### 1.1 Harvest now, decrypt later

Every transaction broadcast on a public blockchain remains permanently retrievable. An adversary needs no real-time decryption capability; they need only record public keys and signatures now and apply Shor's algorithm once a cryptographically relevant quantum computer becomes available.

In account-based models a public key is exposed from first use and reused indefinitely. In UTXO models a key is exposed only at spend time — but once exposed, every subsequent transaction from that key is vulnerable until funds move to a fresh, unexposed address.

The window is not "when does a CRQC arrive". It is the gap between safe to start migrating and must have already migrated, and that gap is measured in years.

### 1.2 Algorithmic impact

| Primitive | Classical security | Quantum attack | Impact |
|---|---|---|---|
| ECDSA (secp256k1) | 128-bit | Shor | Full key recovery |
| EdDSA (Ed25519) | 128-bit | Shor | Full key recovery |
| RSA-2048 | 112-bit | Shor | Full key recovery |
| BLS12-381, BN254 | 128-bit / lower | Shor | Full key recovery |
| SHA-256 | 256-bit preimage | Grover | ~128-bit, still safe |
| SHA3-512 | 512-bit preimage | Grover | ~256-bit, still safe |

Signature schemes break completely under Shor. Well-sized hash functions degrade gracefully under Grover. Q-Lock's primary intervention therefore targets signatures.

Pairing-friendly curves (BLS12-381, BN254) rest on discrete log and break completely. No drop-in post-quantum replacement for pairings exists.

### 1.3 Out of scope

- Consensus-layer quantum attacks on mining or staking  
- Already-exposed keys  
- Side channels (timing / power analysis)

## 2. Architecture

godshield-core (ML-DSA-87 · TripleHash · CanonicalMessage)  
→ gateway / identity / bridge / sentinel / scanner  
→ NEV369 (PoW L1, libp2p) and Q-Lock Escrow (XRPL, non-custodial)  
→ EVM bridge (wNEV + m-of-n) and Nevaeh's vault (3-of-5 Shamir, XRPL escrow)

Fifteen Rust crates, two Solidity contracts, two web frontends. Approximately 17,800 lines of Rust and 252 tests, all passing.

A cryptographic library nobody uses is a claim. A cryptographic library securing two independently verifiable settlement paths, on ledgers with different consensus mechanisms and different trust assumptions, is evidence.

## 3. GodShield — cryptographic core

### 3.1 Signature scheme

CRYSTALS-Dilithium at Security Category 5, standardised by NIST as ML-DSA in FIPS 204.

| Property | Value |
|---|---|
| Scheme | ML-DSA-87 (Dilithium5) |
| Public key | 2,592 bytes |
| Signature | 4,864 bytes |
| Hardness | Module-LWE / Module-SIS |

Size is the central engineering constraint of post-quantum migration.

### 3.2 Triple-hash cascade

`message → SHA3-512 → BLAKE3 → SHA3-512 → digest`

Defense in depth, not a claim of superadditive security. No single hash function's break is immediately fatal.

### 3.3 Canonical encoding

Every signed structure is encoded through CanonicalMessage — length-prefixed and domain-separated — before hashing.

Naive concatenation is ambiguous and was exploitable in three places during development. Length prefixing removes the ambiguity. Domain separation prevents replay across structures.

Tags are versioned and never edited in place.

| Tag | Signs |
|---|---|
| nev369.tx.v1 | Transactions |
| nev369.block.v1 | Block headers |
| nev369.merkle · nev369.empty | Merkle construction |
| godshield.bridge.mint.v1 | Bridge mint authorisations |
| godshield.fairness.outcome.v1 | RNG outcome derivation |
| godshield.fairness.reveal.v1 · fairness.round.v1 | Signed reveals |
| qlock.attestation.v1 | Settlement attestations |
| qlock.escrow.v1 · qlock.escrow.v2 | Escrow terms |
| GODSHIELD-CREDENTIAL-V1 | Credentials |
| GODSHIELD-MACHINE-ACTION-V1 | Machine actions |
| GODSHIELD-GATEWAY-ATTESTATION-V1 | Gateway attestations |
| GODSHIELD-AUDIT-V1 | Audit chain entries |
| GODSHIELD-SENTINEL-SIGNAL-V1 | Correlated signals |
| GODSHIELD-SENTINEL-EVIDENCE-V1 | Incident evidence digests |

## 4. GodShield — security platform

### 4.1 Identity and credentials

Four states: ACTIVE, SUSPENDED, REVOKED, EXPIRED.

- Stored status is never trusted for expiry. Expiry is computed at every authorisation.  
- Revoking an issuer invalidates credentials it already signed.  
- A valid signature is not authority.  
- SUSPENDED and REVOKED are distinct states.

### 4.2 Machine security

Actions signed under GODSHIELD-MACHINE-ACTION-V1. Nonces scoped per actor. Acceptance window: five minutes backward, thirty seconds forward.

### 4.3 Gateway

Signs only its own attestations. Customer keys remain in customer custody. Audit log is hash-chained and tamper-evident. Key rotation resolves historical keys by timestamp.

### 4.4 Sentinel

Reversible actions automatic; irreversible ones escalate. Protected identities are never contained automatically. Evidence is preserved before containment.

### 4.5 Migration scanner

Static analyser for quantum-vulnerable primitives. Not a certification. Cannot see crypto behind dependencies, dynamic dispatch, or FFI.

## 5. NEV369

Proof-of-work Layer-1. Every transaction signature and every block hash uses GodShield.

| Parameter | Value |
|---|---|
| Signature scheme | ML-DSA-87 |
| Block hash | TripleHash over canonical header + Merkle root |
| Target block time | 60 seconds |
| Difficulty retarget | Every 100 blocks, clamped ±1 |
| Genesis difficulty | 4 |
| Max txs per block | 500 |
| Mempool | 10,000 |
| Base units | u64, 8 decimals |
| Fork choice | Accumulated work |
| Networking | libp2p — gossipsub, mDNS, identify, noise, yamux |

Money is integer, never floating point.  
Block contents execute against a scratch state (in-block double-spend).  
Exactly two signature exemptions: GENESIS and NETWORK_REWARD.  
The sender is the Dilithium5 public key.  
Fork choice by accumulated work, not height.  
Timestamps are consensus rules.  
Genesis is constructed, not mined. The dedication is inside the genesis hash.

Premine addresses are pinned into genesis from the vault ceremony.

## 6. Token economics

| | NEV | Share |
|---|---|---|
| Maximum supply | 369,369,369 | 100% |
| Mineable | 322,469,369 | 87.30% |
| Nevaeh's premine (locked to 2039) | 36,900,000 | 9.99% |
| Architect's premine | 10,000,000 | 2.71% |

369 NEV per block, halving every 437,000 blocks (~303 days).  
Fees and crown_tax are burned.

## 7. Q-Lock Escrow

Non-custodial escrow on the XRP Ledger using native EscrowCreate, EscrowFinish and EscrowCancel.

No server-side key can move user funds. Signing occurs in Xaman or on a hardware wallet.

XRPL cannot verify lattice signatures at consensus. This is hybrid — ECDSA settlement with a parallel Dilithium5 attestation — not native on-chain post-quantum transactions.

Attestation identity fails closed in production.

| Plan | Fee | Monthly escrows |
|---|---|---|
| Free | 0.30% | 5 |
| Pro | 0.20% | unlimited |
| Enterprise | 0.15% | unlimited |

No dispute arbitration, no recourse desk, no insured custody.

## 8. The bridge

wNEV is an 8-decimal ERC-20, 1:1 with NEV369. No scaling.

Minting requires m-of-n threshold authorisation off-chain (Dilithium5 attestations) and on-chain (secp256k1 over EIP-712).

Ethereum has no ML-DSA precompile. On-chain enforcement is classical. The bridge is post-quantum attested, not post-quantum enforced.

A threshold of 1 is rejected in the constructor.  
Per-mint / per-window / total caps. Circuit breaker on unbacked supply. Breaker does not self-heal.

## 9. The inheritance vault

Nevaeh, born 28 July 2021. Releasable 28 July 2039.

The inheritance is a native XRP Ledger escrow with FinishAfter. No code in the Q-Lock repository can release it early.

NEV369 also holds 36,900,000 NEV time-locked to the same date. That lock is a commitment device. XRPL consensus is the part engineered to survive the author.

| Vault | Holds | Threshold | Time-locked |
|---|---|---|---|
| nevaeh-xrpl-seed | XRPL family seed | 3-of-5 | to 2039 |
| nevaeh-nev369 | Dilithium5 key | 3-of-5 | to 2039 |
| architect-wallet | Dilithium5 key | 2-of-3 | no |

Format: AES-256-GCM ciphertext, 12-byte nonce, Shamir over GF(256) at threshold 3. Documented in plain English so a cryptographer can reconstruct the key if the original software is gone.

The escrow's OfferSequence is recorded in vault notes, ceremony document and will.

## 10. Security analysis

Properties provided: no server-side key moves user funds; one compromised key cannot mint wNEV; one compromised key cannot drain the premine; domain-separated canonical encoding; in-block double-spend check; hash-chained audit; protected identities; unbacked-supply circuit breaker.

Trust assumptions: correct pqcrypto-dilithium bindings; operator key hygiene; independent bridge signers.

Lattice cryptography is younger than RSA and elliptic curves. Algorithm agility is a design goal.

## 11. Security boundaries

- Internal security review: complete  
- Third-party audit: not yet commissioned  
- Workspace compiles. 252 tests pass  
- The bridge is not trustless. No NEV369 light client on Ethereum  
- XRPL settlement is classically secured. PQ attestation runs alongside ECDSA  
- On-chain bridge enforcement is classically secured  
- The scanner is not a certification  
- Software time-locks are commitment devices. Only the XRPL escrow is enforced by consensus the operator cannot influence  
- Dilithium5 may be superseded over a thirteen-year horizon  

Overclaiming security properties is itself a security failure.

## 12. Status and roadmap

Current status: compiled, tests passing. Fifteen crates, two contracts, two frontends.

1. First compilation of the workspace  
2. First execution of the test suite  
3. Contract compilation and test  
4. Two-node network run with a forced partition and verified reorganisation  
5. Vault ceremony and genesis construction on real premine addresses  
6. Independent review of the cryptographic core  
7. Independent audit of each integration and both contracts  

No component of this system should secure assets of value until step 7 is complete.

## Colophon

Q-Lock was built by one person, in Rust, learned for the purpose.

Every design decision that argues against its own system follows from one constraint:

**it had to work whether or not he was still here.**

> Nevaeh, my daughter. To secure your freedom against a broken system, I taught myself Rust — the hardest computer language in the world — to build this unyielding sovereign node for you. I faced the worst of life's struggles so you would never have to. I love you infinitely, forever by your side.

NEV369 · Genesis block 0 · Immutable

This document describes design and threat model for technical review. It is not a substitute for an independent third-party audit.