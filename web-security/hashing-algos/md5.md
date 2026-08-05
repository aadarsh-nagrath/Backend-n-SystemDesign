# MD5 (Message-Digest Algorithm 5)

MD5 is a cryptographic hash function that produces a 128-bit digest. Designed by Ronald Rivest in 1991, it was once the default choice for checksums, password storage, and digital signatures — it is now broken for every security-relevant use case and should never be used where an attacker can supply the input. It's still fine, and still used, for pure integrity checks against non-adversarial corruption.

## TL;DR
- MD5 produces a 128-bit (32 hex char) hash. Fast, simple, and — critically — **not collision-resistant**.
- Practical collision attacks exist since 2004; two different files can be crafted to produce the same MD5 hash in seconds on commodity hardware.
- **Never use MD5 for**: passwords, digital signatures, SSL/TLS certificates, or anything where an adversary controls the input.
- **Still fine for**: non-adversarial checksums (accidental corruption detection), consistent-hashing partition keys, git object identifiers (content-addressing, not a security boundary).
- Officially declared broken in 2008 (CMU SEI); RFC 6151 (2011) formalized the deprecation for security use.

## What a cryptographic hash function needs

A cryptographic hash function is a one-way function: fixed-length output, from any-length input, that's supposed to have these properties:

1. **Deterministic** — same input always gives the same output.
2. **Preimage resistance** — given a hash `h`, it should be infeasible to find any `m` such that `hash(m) = h`.
3. **Second preimage resistance** — given `m1`, it should be infeasible to find a different `m2` with `hash(m1) = hash(m2)`.
4. **Collision resistance** — it should be infeasible to find *any* two inputs `m1 ≠ m2` with `hash(m1) = hash(m2)` (weaker requirement than second preimage resistance — the attacker gets to choose both inputs, not match a fixed one).
5. **Avalanche effect** — a one-bit input change flips roughly half the output bits, so outputs look uncorrelated with small input differences.
6. **Fast computation** — efficient to compute in both directions of use (hashing large files, verifying signatures at scale).

MD5 satisfies 1, 5, and 6. It fails 4 outright (practical collisions since 2004) and is theoretically weak on 2 and 3. That's the whole story of why it's deprecated — one broken property is enough to disqualify a hash function from security use, because attacks only need to exploit the property that's weak.

## How MD5 works

MD5 processes input in **512-bit (64-byte) blocks** using the **Merkle–Damgård construction** — a design shared with SHA-1 and (part of) SHA-2, which matters later because it's also the source of the length-extension weakness.

**1. Padding.** The message is padded so its length is congruent to 448 mod 512: append a single `1` bit, then `0` bits, then the original message length encoded in the final 64 bits.

**2. Initialization.** A 128-bit state is split into four 32-bit words with fixed initial constants:
```
A = 0x67452301
B = 0xEFCDAB89
C = 0x98BADCFE
D = 0x10325476
```

**3. Processing.** Each 512-bit block is split into sixteen 32-bit words and run through 4 rounds of 16 operations each (64 total), using round-specific non-linear functions:
```
F(B, C, D) = (B ∧ C) ∨ (¬B ∧ D)   // Round 1
G(B, C, D) = (B ∧ D) ∨ (C ∧ ¬D)   // Round 2
H(B, C, D) = B ⊕ C ⊕ D            // Round 3
I(B, C, D) = C ⊕ (B ∨ ¬D)         // Round 4
```
Each operation combines the current state, an input word, a round constant, and a left-rotation amount, then feeds the result back into the state.

**4. Output.** The final `A‖B‖C‖D` state, concatenated, is the 128-bit digest — usually shown as 32 hex characters.

```plaintext
Function md5(message):
  Pad message to 448 mod 512, append 64-bit length
  A, B, C, D = 0x67452301, 0xEFCDAB89, 0x98BADCFE, 0x10325476
  for each 512-bit block M:
    A', B', C', D' = A, B, C, D
    for i = 0 to 63:
      if 0 ≤ i ≤ 15:  F = (B∧C)∨(¬B∧D), g = i
      if 16 ≤ i ≤ 31: F = (B∧D)∨(C∧¬D), g = (5i+1) mod 16
      if 32 ≤ i ≤ 47: F = B⊕C⊕D,        g = (3i+5) mod 16
      if 48 ≤ i ≤ 63: F = C⊕(B∨¬D),     g = (7i) mod 16
      temp = F + A' + K[i] + M[g]
      A', D', C' = D', C', B'
      B' = B' + leftrotate(temp, s[i])
    A, B, C, D = A+A', B+B', C+C', D+D'
  return A ‖ B ‖ C ‖ D as 32 hex digits
```

```python
import hashlib

def md5_hash(text):
    return hashlib.md5(text.encode()).hexdigest()

print(md5_hash("The quick brown fox jumps over the lazy dog"))
# 9e107d9d372bb6826bd81d3542a419d6

print(md5_hash("The quick brown fox jumps over the lazy dog."))
# e4d909c290d0fb1ca068ffaddf22cbd0   — one period changes the entire hash (avalanche effect)

print(md5_hash(""))
# d41d8cd98f00b204e9800998ecf8427e
```

## The collision attack, mechanically

A collision means two *different* inputs produce the *same* MD5 hash: `md5(m1) == md5(m2)` for `m1 ≠ m2`. The danger isn't abstract — it's that an attacker can produce two files that look/behave differently but hash identically, then substitute one for the other after the hash has been "verified."

**Timeline:**
- **1993** — Den Boer and Bosselaers find pseudo-collisions in MD5's compression function (a weaker, structural warning sign).
- **1996** — Hans Dobbertin finds actual collisions in the compression function; the community starts recommending SHA-1 instead.
- **2004** — Xiaoyun Wang et al. find full MD5 collisions in about an hour on commodity hardware. This is the moment MD5 went from "theoretically weak" to "practically broken."
- **2005** — Arjen Lenstra et al. construct two different X.509 certificates with the same MD5 hash — proving the attack works against real-world signed documents, not just abstract inputs.
- **2006** — Vlastimil Klima reduces collision computation to minutes on a laptop.
- **2008** — Researchers use a chosen-prefix collision to forge a rogue Certificate Authority certificate, making it possible to mint fake but "validly signed" SSL certificates for arbitrary domains.
- **2012** — The Flame malware (nation-state-grade espionage tooling) is found to have used an MD5 collision to forge a Microsoft code-signing certificate, letting it masquerade as a legitimately signed Windows Update component.
- **2013** — Collision resistance broken down to 2^18 complexity, running in under a second.

**Why this matters concretely**: a chosen-prefix collision lets an attacker pick two *meaningfully different* documents (e.g., "you're approved" and "you're not approved," or a benign binary and a malicious one) that happen to share an MD5 hash, given some freedom to add trailing padding bytes. This is exactly what the rogue-CA and Flame attacks exploited — the "signature" only verified a hash, and the hash didn't uniquely identify the content.

```plaintext
File 1: ...58712467eab4004583eb8fb7f89...
File 2: ...50712467eab4004583eb8fb7f89...
Both hash to: 79054025255fb1a26e4bc422aef54eb4
```

**Complexity summary:**

| Attack | Complexity | Practicality |
|---|---|---|
| Collision (identical prefix) | ~2^18–2^24 | Seconds on a laptop |
| Chosen-prefix collision | ~2^39 | Minutes to hours; used in real forgery attacks |
| Preimage | ~2^123.4 | Still theoretical, not practical as of writing |
| Brute-force via birthday attack | ~2^64 | Infeasible directly, but irrelevant — collision attacks are far cheaper |

Note the gap: preimage attacks (finding an input for a *given* hash) remain impractical. It's specifically collision resistance that's broken — the attacker needs to control both documents, which is exactly the scenario in certificate/signature forgery.

## Other weaknesses

- **Length extension attacks**: because of the Merkle–Damgård construction, given `md5(secret ‖ message)` and the length of `secret`, an attacker can compute `md5(secret ‖ message ‖ padding ‖ extra)` without knowing `secret` itself. This breaks naive MAC schemes built as `hash(secret + message)` — use HMAC instead, which is specifically designed to resist this.
- **Brute force / dictionary attacks on passwords**: MD5's speed — the same property that makes it a good checksum — is exactly what makes it a bad password hash. Modern GPUs compute billions of MD5 hashes per second, and precomputed rainbow tables (some spanning hundreds of billions of entries) make looking up common passwords near-instant. See [sha.md](./sha.md) and [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md) for why *any* fast general-purpose hash — MD5 or SHA-256 alike — is the wrong tool for passwords, and what to use instead.

## Where MD5 is still fine vs. actively dangerous

| Use case | Verdict | Why |
|---|---|---|
| Detecting accidental file corruption (e.g., download integrity) | ✅ Fine | No adversary is crafting the input; collisions don't happen by accident |
| Git object hashing (content addressing) | ✅ Fine (note: git actually uses SHA-1, historically) | Used for content-addressable lookup, not as a security boundary against a malicious contributor in most workflows |
| Consistent-hashing partition keys, cache keys, dedup keys | ✅ Fine | Only need uniform distribution and determinism, not attacker-resistance |
| Non-security checksums in internal tooling | ✅ Fine | No adversarial input model |
| Password hashing | ❌ Dangerous | Fast + no built-in salt = trivially brute-forced/rainbow-tabled |
| Digital signatures / SSL certificates | ❌ Dangerous | Chosen-prefix collisions let attackers forge signed content |
| File integrity where an adversary controls one side (e.g., verifying an untrusted download against a hash an attacker also influences) | ❌ Dangerous | Collision attacks defeat exactly this scenario |
| Any MAC built as `hash(secret + message)` | ❌ Dangerous | Length extension attack applies |

The dividing line is simple: **is there an adversary who benefits from crafting a colliding input?** If no (pure accidental-corruption detection), MD5's speed is a feature. If yes (anything security- or trust-related), MD5's broken collision resistance is disqualifying.

## 🟢 Beginner: recognizing MD5 in the wild

If you see `hashlib.md5(password.encode())` or `md5(password)` anywhere near authentication code, that's a bug to fix — not a style choice. Same for MD5 used to verify a downloaded file where the hash itself comes from an untrusted or attacker-influenced source (e.g., a hash published alongside a user-uploaded file, rather than from the original publisher over a trusted channel).

```python
import hashlib

# Dangerous — do not do this for passwords
def hash_password_insecure(password):
    return hashlib.md5(password.encode()).hexdigest()
```

```python
# Fixed — use a purpose-built password hash instead (see scrypt-and-bcrypt.md)
import bcrypt

def hash_password(password):
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```

## 🟡 Intermediate: mitigating MD5 if you're stuck with it

Legacy systems sometimes can't drop MD5 overnight (a data format, a third-party integration, an on-disk format used by other tooling). If you must keep it around temporarily:

1. **Add a salt** to reduce rainbow-table effectiveness (does *not* fix collision resistance, only precomputation):
   ```python
   import hashlib
   password, salt = "mypassword", "randomsalt"
   hashed = hashlib.md5((password + salt).encode()).hexdigest()
   ```
2. **Key stretching** (repeated hashing) slows brute force somewhat, but is a weak substitute for a real password-hashing function — bcrypt/Argon2 do this properly with tunable cost.
3. **Plan a real migration**: store both the old MD5 hash and a new bcrypt/Argon2 hash; verify against MD5 on login, then re-hash with the new algorithm and drop the old one once the user has logged in post-migration.

None of this makes MD5 acceptable long-term — it buys time during a migration, nothing more.

## 🔴 Advanced: why the certificate forgery attacks worked

The 2008 rogue-CA attack is worth understanding in more detail because it shows *why* collision resistance specifically (not preimage resistance) is the property that breaks signature schemes.

A CA signs a certificate by hashing its contents and signing the hash. If an attacker can produce two certificates — one entirely benign (submitted for signing) and one malicious (with attacker-controlled fields like the subject name) — that hash to the *same* MD5 value, then the CA's signature on the benign certificate is *also* a valid signature on the malicious one, because the signature only ever attested to the hash, not the full content. The attacker gets the CA to sign the harmless-looking version, then swaps in the malicious one — the signature still verifies.

This requires a **chosen-prefix collision** (the attacker needs freedom to append different suffixes to two different chosen prefixes and still collide), which is exactly what the 2007–2008 research achieved against MD5, at a practical ~2^39 complexity using clusters of PlayStation 3s. It's a direct demonstration of why "the hash is hard to reverse" (preimage resistance, still true for MD5) is not sufficient for signature security — collision resistance is the property signatures actually depend on, and that's the one MD5 lost.

## Quick reference

- Output: 128 bits / 32 hex chars.
- Construction: Merkle–Damgård, 4 rounds × 16 ops.
- Broken since: 2004 (practical collisions), 2008 (chosen-prefix collisions used in real forgery).
- Official status: "cryptographically broken and unsuitable for further use" (CMU SEI, 2008); RFC 6151 (2011).
- Safe uses: non-adversarial checksums, cache/partition keys, dedup.
- Unsafe uses: passwords, signatures, certificates, any MAC as `hash(secret+msg)`.
- Replacement for security use: SHA-256/SHA-3 for general hashing (see [sha.md](./sha.md)); bcrypt/scrypt/Argon2 for passwords (see [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md)).

## Further reading
- RFC 1321 — original MD5 specification.
- RFC 6151 — security considerations update, formal deprecation guidance.
- Wang, Xiaoyun et al., "How to Break MD5 and Other Hash Functions" (2005) — the paper behind the 2004 practical collision.
- [sha.md](./sha.md) — the SHA family and why SHA-256 replaced MD5/SHA-1 for general-purpose hashing.
- [scrypt-and-bcrypt.md](./scrypt-and-bcrypt.md) — correct password hashing (bcrypt, scrypt, Argon2).
