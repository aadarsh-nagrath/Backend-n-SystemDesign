# Password hashing: bcrypt, scrypt, and Argon2

Storing passwords securely requires a fundamentally different tool than a general-purpose hash function like MD5 or SHA-256. Those are built for speed; password hashing needs the opposite — deliberately slow, memory-hungry, and salted by design, so that stealing the password database doesn't hand an attacker a fast path to the plaintext. This note covers the three functions actually built for that job: **bcrypt**, **scrypt**, and **Argon2**.

> Not to be confused with **sCrypt**, an unrelated TypeScript-based smart-contract language for Bitcoin SV — see the [disambiguation note](#note-scrypt-vs-scrypt-naming-collision) at the end. This note is entirely about the cryptographic KDF.

## TL;DR
- Never hash passwords with a fast general-purpose hash (MD5, SHA-256) — see [sha.md](./sha.md) for exactly why that fails.
- **bcrypt** (1999): Blowfish-based, salted, adjustable cost factor, ~72-byte password limit, battle-tested for 25+ years.
- **scrypt** (2009): memory-hard KDF, resists GPU/ASIC parallelization far better than bcrypt.
- **Argon2** (2015, Password Hashing Competition winner): current best-practice default — memory-hard, tunable, better side-channel resistance, three variants for different threat models.
- All three: salt automatically or require one, are deliberately slow, and expose tunable "cost" parameters so you can keep ahead of hardware improvements over time.
- If starting a new project today: **Argon2id**. If your framework/ecosystem doesn't support it well: **bcrypt** is still a solid, simple, safe default.

## Why password hashing needs to be slow and salted

Two failures show up over and over when passwords are hashed wrong:

**No salt** → identical passwords produce identical hashes. An attacker who steals the database can precompute a rainbow table once (hash → plaintext lookup for common passwords) and reverse every matching user instantly, and reuse that same table across every other breach with the same weakness. Salting — a random value per password, stored alongside the hash — means every hash is unique even for identical passwords, so precomputed tables become useless; the attacker has to attack each hash individually.

**Too fast** → general-purpose hashes are optimized for throughput (SHA-256 does billions of hashes/sec on a GPU). Once a password database leaks, that's an offline brute-force/dictionary attack running at full hardware speed, and a large share of real-world passwords fall within hours. A password hashing function fights back by making each individual hash *deliberately* expensive — milliseconds instead of nanoseconds — which is negligible for a legitimate login but devastating multiplied across a billion guesses.

bcrypt, scrypt, and Argon2 all build in both properties by default: mandatory salting and a tunable cost parameter that lets you dial the per-hash cost up as hardware gets faster, without changing the algorithm.

## bcrypt

Designed by Niels Provos and David Mazières in 1999 (presented at USENIX), bcrypt is built on the **Blowfish cipher**'s expensive key-setup phase, modified into **EksBlowfish** (Expensive Key Schedule Blowfish) specifically to make key setup slow and tunable.

### How it works

**Inputs**: password (max 72 bytes — longer inputs are silently truncated), a 128-bit random salt, and a logarithmic cost factor (`2^cost` iterations, typically 10–14).

**Phase 1 — EksBlowfish setup**: initialize Blowfish's internal state using the cost factor, salt, and password, running an expensive key-derivation schedule with `2^cost` iterations. This is the deliberately slow part.

**Phase 2 — encryption loop**: encrypt the fixed string `OrpheanBeholderScryDoubt` 64 times using the now-initialized EksBlowfish cipher in ECB mode, then encode the result.

```plaintext
Function bcrypt(Password, Salt, Cost):
  State ← EksBlowfishSetup(Cost, Salt, Password)
  Ciphertext ← "OrpheanBeholderScryDoubt"
  for i ← 1 to 64:
    Ciphertext ← EksBlowfishEncrypt(Ciphertext, State)
  return "$2a$" + Cost + "$" + Base64(Salt + Ciphertext)
```

Output is a 60-character string:

```
$2a$10$3euPcmQFCiblsZeEu5s7p.9OVHgeHWFDk9nhMqZ0m/3pd/lhwZgES
|   |  |                     |_________________________|
|   |  |                              Salt + hash
|   |  Cost (2^10 = 1024 iterations)
|   Version identifier
```

### Cost factor in practice

Doubling the cost doubles the time — this is what makes it tunable against hardware improvements over years:

| Cost | Approx. time (2017 MacBook Pro, i7) |
|---|---|
| 10 | ~65ms |
| 12 | ~254ms |
| 14 | ~1015ms |
| 16 | ~4088ms |

A cost of 12 (roughly a quarter-second per hash) is a common baseline — cheap enough that a real login isn't noticeably slow, expensive enough that brute-forcing millions of guesses becomes impractical.

### Code

```javascript
const bcrypt = require('bcrypt');
const saltRounds = 12;

// Hashing (salt generated automatically)
const hash = await bcrypt.hash(password, saltRounds);
// store `hash` — it already contains the salt and cost factor

// Verifying
const isValid = await bcrypt.compare(candidatePassword, storedHash);
```

```python
import bcrypt

hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
# verify
bcrypt.checkpw(candidate.encode(), hashed)
```

### The 72-byte limit, and how to handle it

bcrypt silently truncates anything past 72 bytes — a password differing only after byte 72 hashes identically, which is a real (if narrow) footgun. If you need to support arbitrarily long passphrases, pre-hash with SHA-256 first (producing a fixed 32-byte input well under the limit) before passing to bcrypt:

```javascript
const crypto = require('crypto');
const preHashed = crypto.createHash('sha256').update(longPassword).digest('hex');
const hash = await bcrypt.hash(preHashed, 12);
```

### bcrypt vs. everything else

- **Memory-hardness**: bcrypt is *not* memory-hard in the way scrypt/Argon2 are — its cost is purely CPU-time based, which means GPUs/ASICs (with many cheap parallel cores) still get real leverage over a defender's CPU, just less than they would against a fast general-purpose hash. This is bcrypt's main theoretical weakness relative to scrypt/Argon2.
- **Maturity**: 25+ years in production with no major cryptanalytic break — a real, non-trivial advantage over newer designs.
- **Simplicity**: built-in salt handling, one cost parameter, wide library support across virtually every language/framework.

## scrypt

Designed by Colin Percival in 2009 for the Tarsnap backup service, standardized as RFC 7914 (2016). scrypt's whole design point is **memory-hardness**: unlike bcrypt, it forces an attacker to use large amounts of RAM per guess, which is expensive to parallelize on GPUs and ASICs (both of which have comparatively little fast memory per core relative to their raw compute throughput). A CPU-only defender pays a similar memory cost either way, so the attacker's usual hardware advantage shrinks dramatically.

### How it works

scrypt combines **PBKDF2** (for initial key stretching) with a memory-intensive mixing stage (**ROMix**, built on the **Salsa20/8** stream cipher).

**Parameters**:
- `N` — CPU/memory cost, a power of 2 (e.g., `2^14 = 16384`).
- `r` — block size factor (commonly 8).
- `p` — parallelization factor (commonly 1).
- Salt and desired output key length.

**Steps**:
1. Use PBKDF2-HMAC-SHA256 to stretch the password+salt into `p` initial blocks.
2. Run each block through **ROMix**: generate `N` sequential states of the block (storing each — this is where the memory cost comes from), then perform `N` pseudo-random lookups back into those stored states, mixing as it goes. An attacker trying to save memory by recomputing states on the fly instead of storing them pays a steep *time* penalty instead — this time–memory trade-off is the core of scrypt's design.
3. Concatenate the mixed blocks and run through PBKDF2 again to produce the final derived key.

```plaintext
Function scrypt(Password, Salt, N, r, p, dkLen):
  B ← PBKDF2-HMAC-SHA256(Password, Salt, 1, 128*r*p)
  for i ← 0 to p-1:
    Bi ← ROMix(Bi, N)
  return PBKDF2-HMAC-SHA256(Password, B0‖B1‖...‖Bp-1, 1, dkLen)

Function ROMix(Block, N):
  X ← Block
  for i ← 0 to N-1:
    V[i] ← X
    X ← BlockMix(X)
  for i ← 0 to N-1:
    j ← Integerify(X) mod N
    X ← BlockMix(X XOR V[j])
  return X
```

**Memory usage**: roughly `128 * N * r * p` bytes. With the common defaults `N=2^14, r=8, p=1`, that's `128 * 16384 * 8 = 16 MB` per hash attempt — trivial for one legitimate login, but multiplied across a billion cracking attempts it becomes a very real hardware cost for an attacker.

### Code

```python
import scrypt, os

password = "mypassword".encode()
salt = os.urandom(32)
key = scrypt.hash(password, salt, N=2**14, r=8, p=1, dklen=32)
# store salt, N, r, p, and key together for verification
```

```javascript
const crypto = require('crypto');
crypto.scrypt(password, salt, 64, { N: 16384, r: 8, p: 1 }, (err, derivedKey) => {
  if (err) throw err;
  // store derivedKey.toString('hex')
});
```

### scrypt vs. bcrypt

| | bcrypt | scrypt |
|---|---|---|
| Memory-hard | No (CPU-time only) | Yes |
| GPU/ASIC resistance | Moderate | Strong |
| Maturity | 25+ years, very battle-tested | Solid, less ubiquitous |
| Complexity to configure correctly | Low (one cost parameter) | Higher (N, r, p all interact) |
| Password length limit | 72 bytes | None |

scrypt's extra memory-hardness is a genuine security advantage against dedicated cracking hardware, at the cost of being more complex to tune correctly and heavier on server memory at scale (relevant if you're hashing thousands of logins concurrently).

## Argon2

Winner of the 2015 **Password Hashing Competition** (a multi-year public competition specifically run to find bcrypt/scrypt's successor), Argon2 is the current best-practice recommendation for new systems — OWASP's password storage cheat sheet lists it first.

### Why Argon2 over bcrypt/scrypt

- **Better-tuned memory-hardness**: Argon2 lets you independently tune memory cost, time cost, and parallelism, giving finer control over the CPU/memory/parallelism trade-off than scrypt's more rigid `N/r/p` interaction.
- **Side-channel resistance**: Argon2i is specifically designed to resist cache-timing side-channel attacks (an attacker who can observe *which* memory addresses get accessed, not just measure timing) — a threat model scrypt doesn't address as directly.
- **Modern design incorporating lessons from both predecessors**: it took the memory-hardness of scrypt, added configurable parallelism, and hardened against attacks discovered against earlier memory-hard designs during and after the competition.

### The three variants

| Variant | Memory access pattern | Best for | Trade-off |
|---|---|---|---|
| **Argon2d** | Data-dependent | Maximizes resistance to GPU cracking | Vulnerable to cache-timing side-channel attacks (memory access pattern depends on the secret) |
| **Argon2i** | Data-independent | Situations where side-channel attacks are a real threat (e.g., shared/multi-tenant hardware) | Slightly weaker GPU resistance than Argon2d for the same cost |
| **Argon2id** | Hybrid — Argon2i for the first pass, Argon2d after | General-purpose recommendation | Best all-around balance; this is what you should default to |

**Argon2id is the recommended default** for password hashing — it's what OWASP recommends, what most modern frameworks default to when Argon2 is offered, and the variant referenced when people just say "use Argon2" without specifying.

### Parameters

- **Memory cost (`m`)** — how much RAM each hash attempt uses (e.g., 19–64 MiB for interactive login; higher for less latency-sensitive contexts).
- **Time cost (`t`)** — number of passes over the memory (iterations).
- **Parallelism (`p`)** — number of parallel threads/lanes.

OWASP's current baseline guidance (subject to revision as hardware evolves) is roughly: `m=19 MiB, t=2, p=1` as a minimum for Argon2id, scaling memory up first if you can afford the server cost — memory cost is the parameter that hurts attackers (GPU/ASIC) more than it hurts you, since a single legitimate login only pays it once.

### Code

```python
from argon2 import PasswordHasher

ph = PasswordHasher()  # sensible defaults: Argon2id, tuned memory/time/parallelism
hashed = ph.hash("mypassword")
# $argon2id$v=19$m=65536,t=3,p=4$...

# Verify
try:
    ph.verify(hashed, "mypassword")
except Exception:
    # raises on mismatch — VerifyMismatchError
    pass
```

```javascript
const argon2 = require('argon2');

const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536, // 64 MiB
  timeCost: 3,
  parallelism: 4,
});

const isValid = await argon2.verify(hash, candidatePassword);
```

The output string is self-describing — it embeds the variant, version, and all three cost parameters, so verification doesn't require separately storing config (same pattern bcrypt uses).

## Comparison: bcrypt vs. scrypt vs. Argon2

| | bcrypt | scrypt | Argon2 (id) |
|---|---|---|---|
| Year / origin | 1999, USENIX | 2009, Tarsnap / RFC 7914 | 2015, Password Hashing Competition winner |
| Memory-hard | No | Yes | Yes, independently tunable |
| Side-channel resistance | Not a design goal | Not a design goal | Explicit (Argon2i/id) |
| Configurability | Single cost factor | N, r, p (interact, harder to tune) | Memory, time, parallelism (independent) |
| Password length limit | 72 bytes | None | None |
| Maturity | Extremely high (25+ yrs) | High | Growing, well-vetted, newer |
| Recommended for new systems | Solid fallback | Reasonable if already in use | **Default choice** |

## 🟢 Beginner: what to actually do

For a new project: use **Argon2id** via your language's standard library or a well-maintained package (`argon2-cffi` in Python, `argon2` in Node, `argon2-java`, etc.). If your framework has a built-in password hasher (Django, Laravel, Spring Security), use it rather than calling the primitive directly — it'll pick sane defaults and handle upgrades.

If Argon2 isn't available in your stack, **bcrypt** is still a fully acceptable choice — it has no known practical break after 25 years of scrutiny, and "boring and proven" is a legitimate security property.

```python
# Django: settings.py — Argon2 as the primary hasher, with a fallback for existing bcrypt hashes
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
]
```

**Never** do any of the following:
- Hash passwords with MD5 or plain SHA-256 (see [sha.md](./sha.md)).
- Roll your own salting/stretching scheme instead of using a vetted library.
- Store passwords in plaintext or with reversible encryption — hashing should be one-way, full stop.

## 🟡 Intermediate: choosing and tuning cost parameters

- **Target a real wall-clock cost**, not a specific parameter value — e.g., "roughly 250ms per hash on our production hardware," then pick the cost/memory/time parameters that hit that on your actual servers, not a laptop.
- **Re-benchmark on real deployment hardware.** A cost factor tuned on a beefy dev machine may be far cheaper (and less secure) on a smaller production instance, or unacceptably slow the other way around.
- **Revisit cost periodically** — every year or two, bump the cost factor to track hardware improvements. bcrypt/scrypt/Argon2 are all designed for exactly this kind of incremental retuning.
- **Migrate cost on login, not in a batch job**: when raising the cost factor, don't rehash the entire database at once (you don't have the plaintexts to do that anyway). Instead, verify against the stored (old-cost) hash on next login, then re-hash with the new cost and overwrite — cost increases naturally as users log in.
- **Rainbow table defense recap**: salting is what defeats precomputed tables specifically. It doesn't slow down an attacker targeting one specific hash — that's what the cost factor is for. The two mechanisms solve different halves of the problem, and all three algorithms here enforce both.

```python
# Rehash-on-login pattern
def verify_and_upgrade(password, stored_hash):
    if not ph.verify(stored_hash, password):
        raise InvalidPassword()
    if ph.check_needs_rehash(stored_hash):  # cost params changed since this hash was made
        new_hash = ph.hash(password)
        db.update_password_hash(user_id, new_hash)
```

## 🔴 Advanced: memory-hardness, GPUs, and the actual threat model

The reason memory-hardness matters so much: GPUs and ASICs get their cracking speed advantage from massive core counts doing simple, cheap operations in parallel. A fast hash (SHA-256) has a tiny, fixed working set per computation — perfectly suited to running millions of parallel instances on cheap silicon. A memory-hard function forces each parallel instance to hold a large, mostly-random working set in fast memory — and fast memory (as opposed to compute cores) is the expensive, poorly-parallelizable resource on GPU/ASIC hardware. This is why scrypt/Argon2's memory cost hurts an attacker's economics far more than it hurts a defender doing one hash per real login.

**Time–memory trade-off attacks**: an attacker can choose to use *less* memory than the algorithm intends by recomputing intermediate values instead of storing them — but this trades memory for a steep increase in computation time (this is exactly the ROMix pattern in scrypt, and the mechanism Argon2 also defends against). A well-chosen memory cost parameter makes both the memory-heavy *and* the compute-heavy path expensive — there's no free lunch for the attacker.

**Side-channel considerations in shared environments**: on multi-tenant cloud infrastructure, cache-timing attacks (observing which memory addresses a co-located process touches) are a real, if niche, threat. Argon2d's data-dependent memory access pattern is theoretically more exposed to this than Argon2i's data-independent pattern — part of why Argon2id's hybrid approach (data-independent first pass, data-dependent afterward) is the pragmatic default rather than pure Argon2d.

**What none of these solve**: a weak, guessable, or reused password is still weak, guessable, or reused no matter how expensive the hash is — bcrypt/scrypt/Argon2 raise the cost of *cracking a stolen hash*, they don't substitute for password strength requirements, breach-database checking (e.g., against Have I Been Pwned's range API), or MFA. Defense in depth still applies; a strong hashing function is one layer, not the whole strategy.

## Quick reference

- Never: MD5, SHA-256, or any fast hash alone for passwords.
- bcrypt: simple, mature, 72-byte limit, CPU-cost only (not memory-hard).
- scrypt: memory-hard, more GPU/ASIC-resistant than bcrypt, more complex to tune (N/r/p).
- Argon2id: current best practice — independently tunable memory/time/parallelism, side-channel-aware, competition-vetted.
- Salting defeats precomputed/rainbow-table attacks; cost factor defeats brute force on a specific hash — you need both, and all three algorithms enforce both by design.
- Re-tune cost periodically; migrate cost via rehash-on-login, not a batch rehash (you don't have the plaintexts).
- Hashing strength doesn't replace password policy, breach checking, or MFA — it's one layer of defense in depth.

## Further reading
- RFC 7914 — scrypt specification.
- Provos & Mazières, "A Future-Adaptable Password Scheme" (1999) — the original bcrypt paper.
- Biryukov, Dinu, Khovratovich, "Argon2: the memory-hard function for password hashing" (2015) — Password Hashing Competition winning submission.
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — current parameter recommendations, updated over time as hardware improves.
- [sha.md](./sha.md) — why fast general-purpose hashes are the wrong tool for this job.
- [md5.md](./md5.md) — the same lesson from the broken-hash side of the story.

---

### Note: scrypt vs. sCrypt naming collision

Worth a one-time disambiguation since the names are easy to confuse: **sCrypt** (capital C) is an unrelated TypeScript-based domain-specific language for writing smart contracts on the Bitcoin SV blockchain, developed by sCrypt Inc. since 2018. It compiles TypeScript-like contract code directly to Bitcoin Script and has nothing to do with password hashing or key derivation — it just happens to share a similar name with the cryptographic KDF this note is about. If you're looking for Bitcoin SV smart-contract tooling, you want a different resource entirely; everything in this note refers to the memory-hard KDF designed by Colin Percival.
