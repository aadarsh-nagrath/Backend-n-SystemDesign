# The SHA family (SHA-1, SHA-2, SHA-3)

SHA (Secure Hash Algorithm) is a family of cryptographic hash functions developed by the NSA and standardized by NIST. It's the backbone of digital signatures, TLS certificates, blockchain, and file integrity checks. The family has three generations — SHA-1 (broken), SHA-2 (current standard), SHA-3 (future-proofing) — and understanding why one replaced the next is mostly a story of collision resistance breaking down under advancing cryptanalysis and hardware.

## TL;DR
- **SHA-1**: 160-bit, broken since 2017 (Google's SHAttered attack produced a real collision). Deprecated for anything security-relevant.
- **SHA-2** (SHA-256, SHA-384, SHA-512, etc.): current industry standard. No practical collision attacks. Used for TLS certificates, code signing, Bitcoin.
- **SHA-3** (Keccak): structurally different (sponge construction, not Merkle–Damgård), immune to length-extension attacks by design. Not more "broken-proof" than SHA-2 today, but a hedge against future Merkle–Damgård-specific attacks.
- **None of SHA-1/2/3 should ever be used alone for password hashing** — they're fast general-purpose hashes, and speed is the opposite of what you want for passwords. See below for exactly why, and [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md) for the actual right tool.

## What these functions have in common

All three generations share the baseline cryptographic hash properties: deterministic, fixed-length output, preimage-resistant, second-preimage-resistant, collision-resistant, and exhibit the avalanche effect (a one-bit input change flips ~half the output bits). Where they differ is *how* they achieve this internally, and what that internal structure does or doesn't protect against.

## SHA-1

- **Introduced**: 1993 (NSA), standardized as FIPS 180-1 in 1995.
- **Output**: 160 bits (40 hex characters).
- **Construction**: Merkle–Damgård, same family as MD5 — pads input to a block boundary, processes 512-bit blocks through 80 rounds using logical functions (`Ch`, `Parity`, `Maj`), rotations, and modular addition across five 32-bit state words (`H0`–`H4`).

```python
import hashlib
print(hashlib.sha1("hello".encode()).hexdigest())
# aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d
```

**Broken in practice**: the 2017 **SHAttered** attack (Google + CWI Amsterdam) produced two distinct PDF files with an identical SHA-1 hash, at a complexity of about 2^63.1 — computationally expensive but well within reach of a well-resourced attacker (Google reported roughly 6,500 CPU-years and 100 GPU-years, but the point was proving *feasibility*, and the cost has only dropped since). Like MD5, it also inherits length-extension weakness from the Merkle–Damgård construction, and its 160-bit output is short enough to worry about birthday-bound brute force (~2^80) on top of that.

NIST deprecated SHA-1 for security use in 2011 (FIPS 180-4); browsers and CAs stopped accepting SHA-1 certificates by 2017. It still shows up in non-adversarial contexts — Git's object hashing being the most visible example — where the property being relied on is content-addressing/deduplication, not collision-resistance against a malicious party (though Git's move toward SHA-256 for object hashing is itself a tacit acknowledgment that even that assumption has limits).

## SHA-2

- **Introduced**: 2001 (NSA), standardized as FIPS 180-2 (2002).
- **Variants**: SHA-224, SHA-256, SHA-384, SHA-512, plus truncated forms SHA-512/224 and SHA-512/256.
- **Construction**: still Merkle–Damgård, but with a larger state, more rounds (64 for the 256-bit variants, 80 for 512-bit), and a more complex message schedule than SHA-1 — the added complexity is precisely what's kept it collision-resistant so far.

```python
import hashlib
print(hashlib.sha256("hello".encode()).hexdigest())
# 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
```

```javascript
const crypto = require('crypto');
console.log(crypto.createHash('sha256').update('hello').digest('hex'));
// 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
```

**Security today**: no practical collision or preimage attacks against SHA-256 or SHA-512. Theoretical collision resistance sits at 2^128 for SHA-256 (birthday bound on a 256-bit output) — astronomically out of reach with current or foreseeable computing. It's the default choice for TLS certificates (required since 2016), code signing, Bitcoin's proof-of-work, and general file integrity.

**Residual weakness**: because it's still Merkle–Damgård, plain SHA-256/SHA-512 is theoretically vulnerable to length-extension attacks (the same class of issue MD5 has) unless you use a truncated variant like SHA-512/256, or protect a MAC construction with HMAC instead of naive `hash(secret + message)` concatenation.

## SHA-3 (Keccak)

- **Introduced**: 2015 by NIST, based on Keccak, winner of the public SHA-3 competition (2007–2012) — the response to growing unease about a single construction family (Merkle–Damgård) underpinning MD5, SHA-1, *and* SHA-2.
- **Variants**: 224, 256, 384, 512-bit outputs, plus extendable-output functions SHAKE128/SHAKE256.
- **Construction**: the **sponge construction** — a fundamentally different design from Merkle–Damgård. Data is absorbed into a 1600-bit state via a permutation function (`Keccak-f`, 24 rounds of theta/rho/pi/chi/iota operations), then squeezed out to produce the digest.

```python
import hashlib
print(hashlib.sha3_256("hello".encode()).hexdigest())
# 3338be694f50c5f338814986cdf0686453a888b84f424d792af4b9202398f392
```

**Why it exists despite SHA-2 being fine**: not because SHA-2 is broken, but as *diversification* — if a structural weakness were ever found in Merkle–Damgård itself, SHA-2 and SHA-1 and MD5 would all be affected simultaneously, since they share the same construction. SHA-3's sponge design is immune to length-extension attacks by construction (no separate mitigation needed), and gives the ecosystem a fallback that doesn't share SHA-2's failure mode. Adoption has been slower than SHA-2 mainly because SHA-2 remains uncompromised and well-optimized in hardware — SHA-3 tends to be faster in dedicated hardware but slower in general-purpose software.

## Why plain SHA-256 (or any SHA) is the wrong tool for password hashing

This is the single most important practical takeaway from this whole family of algorithms, and it's worth being explicit about the mechanism, not just the rule.

**The property that makes SHA-256 good at its job — speed — is exactly the property that makes it bad for passwords.** SHA-256 is designed to hash gigabytes of data (file integrity, TLS handshakes, blockchain mining) as fast as possible. Modern CPUs and GPUs are *extremely* good at this: consumer GPUs can compute well over a billion SHA-256 hashes per second, and purpose-built ASICs (the same hardware used for Bitcoin mining) go vastly higher still.

Now put a password behind it:

```python
# Vulnerable: fast, unsalted hash used for password storage
import hashlib

def hash_password_wrong(password):
    return hashlib.sha256(password.encode()).hexdigest()
```

Two separate failures stack here:

1. **No salt** — identical passwords produce identical hashes. An attacker who steals the password database can precompute a **rainbow table** (a lookup table of hash → plaintext for common passwords) once, and instantly reverse every user whose password is in that table, across every breach that reuses the same table. Two users with the password `hunter2` get the identical stored hash, immediately revealing they share a password even before any cracking happens.

2. **No deliberate slowness** — because SHA-256 is optimized for raw throughput, an attacker with the stolen hash database runs an offline brute-force/dictionary attack at GPU/ASIC speed. At roughly a billion+ SHA-256 hashes per second on a single consumer GPU, an 8-character password drawn from a realistic character set can fall within hours to days; anything in a common wordlist (rockyou.txt-style dictionaries, which cover a huge share of real-world passwords) falls in seconds. Multiply by a GPU cluster or ASIC farm and the numbers get worse, not better.

Compare that to a purpose-built password hash: bcrypt/scrypt/Argon2 are deliberately *slow* (a tunable cost factor targeting, say, 100–500ms per hash) and *memory-hard* (scrypt/Argon2 specifically — deliberately expensive to parallelize on GPU/ASIC hardware, which have comparatively little fast memory per core). That turns an attacker's billion-hashes-per-second advantage into a few hashes per second per core — a difference of eight or nine orders of magnitude in effort for the exact same password. Salting is also mandatory by design in all three, not an opt-in add-on you might forget.

```python
# Fixed: use a purpose-built, slow, salted password hash instead
import bcrypt

def hash_password(password):
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
```

**The nuance worth keeping**: SHA-256 salted and iterated *can* be used as a password KDF if you apply enough iterations yourself (this is literally what PBKDF2-HMAC-SHA256 is) — but that's strictly worse than bcrypt/scrypt/Argon2 because it's not memory-hard, so GPUs and ASICs still get an outsized advantage over a defender's commodity CPU. If you see raw `hashlib.sha256(password)` with no salt and no iteration count, that's an unambiguous bug. If you see PBKDF2 with a high iteration count, that's *acceptable* (and still standard in some compliance contexts, e.g., FIPS-constrained environments) but not best practice — see [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md) for the full mechanics of why bcrypt/scrypt/Argon2 win, including Argon2 as the current best-practice default.

## Comparison

| Function | Output size | Construction | Collision resistance | Status |
|---|---|---|---|---|
| **SHA-1** | 160 bits | Merkle–Damgård | Broken (SHAttered, 2017) | Deprecated for security use |
| **SHA-256** | 256 bits | Merkle–Damgård | Strong (2^128) | Current standard |
| **SHA-512** | 512 bits | Merkle–Damgård | Strong (2^256) | Current standard, higher margin |
| **SHA-3-256** | 256 bits | Sponge | Strong, no practical attacks | Future-proofing / diversification |

## 🟢 Beginner: picking a SHA variant

- Need a general-purpose hash for file integrity, signatures, or a Git-like content address? **SHA-256** — it's the default for a reason: well-supported, hardware-accelerated, no known weaknesses.
- Building something for the very long term where you want a construction independent of SHA-2's design? **SHA-3**.
- Maintaining something that already uses SHA-1 for non-security purposes (legacy checksums)? Leave it if there's truly no adversarial input; migrate if there's any doubt.
- **Never** reach for any of these — SHA-1, SHA-256, or SHA-3 — as a password hash on their own. That's a different job with different tools; see [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md).

## 🟡 Intermediate: length extension and HMAC

A subtle Merkle–Damgård gotcha: if you build a naive MAC as `tag = SHA256(secret ‖ message)`, an attacker who knows the tag and the length of `secret` (even without knowing `secret` itself) can compute a valid tag for `secret ‖ message ‖ padding ‖ attacker_data` — extending the message without ever learning the secret. This has broken real systems that rolled their own auth-tag scheme this way.

```python
# Vulnerable pattern — do not build a MAC like this
import hashlib
def naive_mac(secret, message):
    return hashlib.sha256((secret + message).encode()).hexdigest()
```

```python
# Fixed: use HMAC, which is specifically designed to resist length extension
import hmac, hashlib
def mac(secret, message):
    return hmac.new(secret.encode(), message.encode(), hashlib.sha256).hexdigest()
```

SHA-3 sidesteps this class of bug entirely by construction (the sponge design doesn't expose the internal state the way Merkle–Damgård does) — one of its real practical advantages, not just diversification for its own sake.

## 🔴 Advanced: migration history and what it teaches

The SHA-1 → SHA-2 transition (2011–2017) is a useful case study in how slow and expensive cryptographic migrations are at internet scale: certificate reissuance across every CA, software updates across every TLS stack, and years of dual-support before old clients could be dropped. It's the reason cryptographic agility (designing systems so the hash/cipher can be swapped without a full rewrite) is treated as a first-class design concern today, not an afterthought — you don't want to discover you're hard-coded to a broken primitive the week a practical attack against it ships.

Post-quantum considerations: symmetric primitives like SHA-2/SHA-3 are comparatively quantum-resistant (Grover's algorithm only gives a quadratic speedup against preimage search, so doubling the output size roughly restores the original security margin) — this is a fundamentally different risk profile from asymmetric crypto (RSA, ECC), which Shor's algorithm breaks outright on a sufficiently large quantum computer. SHA-256/SHA-3-256 aren't considered urgent migration targets for that reason, unlike RSA/ECC-based signatures and key exchange.

## Quick reference

- SHA-1: 160-bit, broken (2017), avoid for anything security-relevant.
- SHA-256/SHA-512: current standard, Merkle–Damgård, strong collision resistance, hardware-accelerated.
- SHA-3: sponge construction, immune to length-extension by design, diversification hedge.
- HMAC, not naive concatenation, for MACs built on any Merkle–Damgård hash.
- **Passwords**: none of the above, ever, used alone. Use bcrypt, scrypt, or Argon2 — see [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md).

## Further reading
- FIPS 180-4 — SHA-2 specification.
- FIPS 202 — SHA-3 specification.
- Stevens et al., "The First Collision for Full SHA-1" (2017) — the SHAttered paper.
- [md5.md](./md5.md) — the earlier, more thoroughly broken hash family member, and the mechanics of collision attacks in more detail.
- [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md) — correct password hashing (bcrypt, scrypt, Argon2) and why they beat plain SHA-256.
