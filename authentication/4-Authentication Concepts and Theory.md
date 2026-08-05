# Authentication concepts and theory

The theoretical foundation underneath the practical guides: what authentication factors actually are, how threat models and compliance regimes shape design decisions, and how authentication systems get architected at scale. Where a topic has a dedicated practical guide (JWT structure, OAuth flows, WebAuthn mechanics), this file gives the conceptual framing and links out rather than re-deriving the mechanics — see [1-authentication.md](<1-authentication.md>), [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>), [3-Oauth Guide.md](<3-Oauth Guide.md>), and [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>).

## TL;DR
- **Authentication** = "who are you?" (identity verification). **Authorization** = "what can you do?" (permission management). Authentication is always the prerequisite.
- Three classic factors: **knowledge** (something you know), **possession** (something you have), **inherence** (something you are). MFA combines factors from different categories — two passwords isn't MFA.
- Modern authentication design is driven as much by **threat modeling and compliance** (GDPR, HIPAA, PCI-DSS) as by the mechanism itself.
- Architecture shape matters: **centralized** (single IdP, SSO) vs. **distributed** (per-service auth, API gateways) vs. **hybrid** (multi-cloud) each trade off consistency, blast radius, and operational complexity differently.
- Emerging directions: passwordless/passkeys (see [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) for the full WebAuthn treatment), risk-based/continuous authentication, and — much further out — quantum-resistant cryptography.

## 1. Authentication fundamentals

### Authentication vs. authorization
- **Authentication (AuthN)**: "Who are you?" — identity verification, the security perimeter around a resource.
- **Authorization (AuthZ)**: "What are you allowed to do?" — permission management, evaluated *after* authentication succeeds.

They're often implemented together (a JWT can carry both an identity claim and a roles claim) but are conceptually and often architecturally separate — a system can authenticate someone correctly and still authorize them incorrectly (over-permissive roles), and vice versa (correct permission model, but weak identity verification lets an impostor in as a legitimate user).

### The three authentication factors

| Factor | Examples | Strength | Weakness |
|---|---|---|---|
| **Knowledge** (something you know) | Password, PIN, security question | Cheap, universally implementable | Guessable, phishable, reusable if leaked, forgettable |
| **Possession** (something you have) | Hardware token (YubiKey), authenticator app, SMS/email code | Meaningfully harder to remotely compromise than knowledge alone | Can be lost/stolen; SMS specifically is vulnerable to SIM-swapping |
| **Inherence** (something you are) | Fingerprint, face, voice, behavioral biometrics | Can't be forgotten or (easily) shared | Irrevocable if compromised — you can't rotate a fingerprint; privacy/legal weight; requires specialized hardware |

**Multi-factor authentication (MFA)** combines factors from *different* categories — a password plus a TOTP code is 2FA (knowledge + possession); a password plus a security question is not meaningfully stronger, since both are knowledge factors an attacker who phishes one can often phish the other. **Adaptive MFA** varies which/how many factors are required based on assessed risk (see risk-based authentication below) rather than applying the same static requirement to every login. See [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) for how TOTP-based MFA compares to phishing-resistant passkeys — the two are not equivalent despite both being "a second factor."

MFA's real security benefit is forcing an attacker to compromise two independent channels simultaneously — a leaked password database alone becomes far less useful. The costs are real too: added login friction, more complex account-recovery flows (what happens when someone loses their second factor?), and — for SMS-based OTP specifically — a genuinely weaker security profile than app-based TOTP or hardware keys, since SMS delivery is interceptable via SIM-swap social engineering attacks against the carrier, not just the target.

## 2. Protocols — conceptual roles, not mechanics

This repo has dedicated files for the mechanics of each protocol below. This section exists to explain *why* each protocol exists and where it fits, not to re-explain request/response formats.

- **OAuth 2.0** — delegated *authorization*: lets a client get scoped access to a resource without the user's password. See [3-Oauth Guide.md](<3-Oauth Guide.md>) for roles, flows, PKCE, and tokens.
- **OpenID Connect (OIDC)** — adds an identity layer on top of OAuth 2.0 (the `id_token`), turning OAuth's authorization into actual authentication. Detailed in both [3-Oauth Guide.md](<3-Oauth Guide.md>) and [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>).
- **SAML** — XML-based, enterprise-focused SSO/authentication standard, predates OAuth. Full flow and component breakdown in [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>).
- **WebAuthn / passkeys** — public-key-based passwordless authentication, phishing-resistant by design. Full conceptual and mechanical treatment (registration/authentication ceremonies, why it resists phishing, comparison to TOTP) in [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>).
- **JWT** — the token *format* most of the above protocols use to carry claims. Structure, revocation strategies, and algorithm attacks in [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>).

### Session vs. token-based — the conceptual trade-off
The fundamental split in how identity is remembered across requests:

| | Session-based | Token-based |
|---|---|---|
| Where state lives | Server (session store) | Client (self-contained token) |
| Revocation | Immediate — delete the server record | Delayed unless extra machinery added (see JWT file) |
| Scaling model | Needs a shared store once you have >1 server | Scales without shared state |
| Primary risk | CSRF, session hijacking | Token theft, replay |

This is the single most consequential early architectural decision in an auth system, because it determines whether "log this user out everywhere, right now" is trivial (session-based) or requires deliberate engineering (token-based — see the JWT file's revocation section). Full mechanics of both: [1-authentication.md](<1-authentication.md>).

## 3. Threat models and security principles

### Common authentication threats
- **Credential theft** — passwords/tokens stolen via breach, phishing, malware, or interception.
- **Session/token hijacking** — an attacker obtains a valid session ID or token and rides it.
- **Man-in-the-middle** — interception of authentication traffic in transit (mitigated by TLS everywhere, HSTS, certificate pinning for high-value clients).
- **Brute force / credential stuffing** — systematic guessing, or replaying breached credentials from other sites against yours (this is why password reuse is dangerous even for "unimportant" accounts).
- **Social engineering / phishing** — manipulating a human into handing over credentials or approving a fraudulent MFA prompt ("MFA fatigue" attacks — spamming push approval requests until the user taps "approve" out of annoyance — are a live, common attack pattern against push-based MFA specifically).

### Defense in depth
No single control should be the only thing standing between an attacker and an account. Layer network security (TLS, firewalls), application security (input validation, output encoding), authentication security (strong hashing, MFA), session security (secure cookies, rotation), and monitoring (logging, alerting) — a failure in any one layer shouldn't be a total compromise.

### Other governing principles
- **Principle of least privilege** — grant the minimum access necessary; this is an authorization concern but authentication design should make it easy to enforce (fine-grained scopes/roles in the token or session, not one flat "authenticated" bit).
- **Fail-safe defaults** — a system should default to *denying* access; explicit action should be required to grant it, never the reverse.
- **Security by design** — threat-model authentication flows at design time, not as a retrofit; secure defaults are cheaper than defense-in-depth added after a breach.

### Common implementation vulnerabilities
Weak/reused passwords, insecure storage (plaintext or weak hashing — always bcrypt/scrypt/Argon2, never raw SHA-256 for passwords), insufficient rate limiting (enables brute force), predictable or non-expiring session IDs, and token security gaps (no expiration, no revocation path, no algorithm pinning — see the JWT file for the concrete attacks these gaps enable).

## 4. Advanced theoretical concepts

### Zero-knowledge proofs
A cryptographic technique letting one party (the prover) convince another (the verifier) that they know a secret — without revealing the secret itself. Three defining properties: **completeness** (a true statement is always accepted from an honest prover), **soundness** (a false statement can't be proven by a cheating prover, except with negligible probability), and **zero-knowledge** (the verifier learns nothing beyond "the statement is true").

Practical variants: **zk-SNARKs** (succinct, non-interactive — practical for blockchain use since proofs are small and verification is fast), **zk-STARKs** (similar goal, avoids the "trusted setup" requirement SNARKs have, at the cost of larger proof sizes), **Bulletproofs** (efficient for range proofs specifically, e.g. "this value is between 0 and 100 without revealing the value"). Authentication use cases remain mostly niche/blockchain-adjacent today (proving credential possession without revealing the credential, privacy-preserving identity verification, voting eligibility) rather than mainstream web auth — but the underlying idea (proving a fact without exposing the data behind it) is conceptually the same spirit as WebAuthn's "prove possession of a private key without ever transmitting it."

### Risk-based authentication (RBA)
Instead of applying the same static authentication requirement to every login, RBA scores each attempt against contextual signals and scales requirements to match:

- **Location** — geographic distance from the user's usual locations, IP reputation, VPN/Tor usage.
- **Device** — is this a recognized device fingerprint, or a first-time browser/OS combination?
- **Time** — login at 3am local time when the user has never done so before is a weaker signal alone, but compounds with others.
- **Behavior** — typing cadence, mouse movement patterns (see continuous authentication below).
- **Network** — known-malicious IP ranges, unexpected ASN changes mid-session.

Scoring approaches range from simple rule-based thresholds ("+0.3 risk if country changed since last login") to ML-based models trained on historical legitimate-vs-fraudulent session data, with hybrid approaches common in practice. The payoff is asymmetric friction: low-risk logins pass through with just a password, medium-risk logins get an MFA challenge, high-risk logins get MFA plus a CAPTCHA or get blocked outright pending manual review — reducing friction for the vast majority of legitimate logins while concentrating security cost where the risk actually is.

### Continuous authentication
Rather than authenticating once at login and trusting the session indefinitely, continuous authentication monitors behavioral signals *throughout* the session to detect if control has silently changed hands (a stolen unlocked laptop, a hijacked session token being used by someone other than the original user).

Signals: keystroke dynamics (timing/rhythm, not just content), mouse movement patterns, which applications/features are used and how, network behavior. A behavioral profile is built over time; live sessions are compared against it, and significant deviation triggers a step-up challenge (re-enter password, provide a second factor) rather than an immediate hard logout. Real challenges: privacy (this is inherently invasive monitoring, and needs to be disclosed and scoped carefully), false positives (a legitimately tired or injured user typing differently shouldn't get logged out mid-task), and performance overhead of continuous client-side monitoring.

### Passwordless authentication — landscape
"Passwordless" is an umbrella covering several distinct mechanisms with very different security properties — worth distinguishing rather than treating as one bucket:

- **Magic links** — a one-time link emailed/texted to a known address; security reduces to the security of that email/SMS channel (if the mailbox is compromised, so is the account).
- **Push notifications** — approve/deny a login on a trusted, already-authenticated device; vulnerable to MFA-fatigue attacks (see above) if not paired with number-matching.
- **Hardware tokens / passkeys (WebAuthn)** — public-key cryptography, phishing-resistant by construction. This is the strongest category and the one worth understanding in depth — see [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) for the full explanation of what a passkey actually is and why it resists phishing where OTP and magic links don't.

The common thread across all passwordless methods is eliminating the shared-secret-typed-into-a-form pattern that makes passwords phishable — but "passwordless" alone doesn't guarantee phishing resistance (magic links and non-number-matching push notifications can both still be socially engineered); only public-key-based methods like WebAuthn make phishing structurally impossible rather than just less convenient.

## 5. Architecture patterns

### Centralized authentication (Identity Provider pattern)
One system (an IdP — Okta, Auth0, an internal identity service) is the single source of truth for identity; other applications federate to it rather than each maintaining their own user store. This is what SSO is built on.

**Benefits**: consistent policy enforcement across every connected app, one place to disable a compromised or offboarded account, reduced administrative overhead. **Costs**: the IdP becomes a single point of failure (if it's down, nobody can log into anything), it must scale to handle every app's authentication load, and switching IdPs later is a significant migration.

### Distributed authentication (microservices pattern)
Each service authenticates independently, or authentication is centralized only at an API gateway that then propagates a validated identity (often a JWT) to downstream services — this is why stateless tokens are the dominant pattern in microservice architectures (see the session-vs-token trade-off above: no service needs to share a session store with any other).

**Benefits**: independent scaling and failure isolation per service, flexibility to use different auth methods per service if genuinely needed (rare in practice — consistency is usually worth more than flexibility here). **Costs**: larger attack surface (more services independently handling tokens/credentials), harder to guarantee consistent enforcement, and distributed logging/monitoring becomes necessary just to answer "who did what, where" across the system.

### Hybrid / multi-cloud authentication
Federating between cloud IdPs (AWS Cognito, Azure AD/Entra ID, Google Identity) and on-premises directories (Active Directory, LDAP) during a cloud migration, or permanently for regulatory/data-residency reasons. **Benefits**: flexibility to pick the best-fit method per use case, a gradual migration path off legacy systems. **Costs**: real operational complexity in keeping multiple systems' user data synchronized and consistently secured — this is where a lot of real-world auth incidents originate, in the seams between systems rather than within any single one.

## 6. Compliance and standards

Compliance requirements are a major real-world driver of authentication design decisions — often more binding in practice than pure security preference, since they're externally audited.

| Regime | Authentication-relevant requirements |
|---|---|
| **GDPR** | Explicit consent for data processing, data minimization (collect only what's needed for auth, not more), right to erasure, privacy-by-design |
| **HIPAA** | Strong authentication for health data access, comprehensive audit logs, encryption in transit and at rest, secure session handling |
| **PCI-DSS** | Multi-factor authentication is effectively mandatory for cardholder data access, strict access control, continuous monitoring |
| **NIST Cybersecurity Framework** | Identify → Protect → Detect → Respond → Recover — a lifecycle model, not just a design-time checklist |
| **OWASP Authentication Cheat Sheet** | Practical, implementation-level guidance: password policy, session management, MFA, account recovery, logging — see [Authentication Security Best Practices.md](<Authentication Security Best Practices.md>) for concrete implementations of most of this |

The practical takeaway: design for the strictest regime you're plausibly subject to from the start (MFA, audit logging, encryption at rest/in transit) rather than retrofitting compliance later — retrofitting audit logging into a system that wasn't designed to produce it is materially harder than building it in from day one.

## 7. Authentication across contexts

Different platforms create genuinely different constraints, not just different UI:

- **Web** — HTTP's statelessness is the root reason session/token mechanisms exist at all; browser-specific attack surface (CSRF, XSS) shapes cookie flag choices (`HttpOnly`, `SameSite`).
- **Mobile** — can lean on OS-level secure storage (Keychain/Keystore) and biometric APIs the OS already trusts, but must handle offline scenarios and app-store review requirements around how credentials are stored.
- **IoT** — often severely resource-constrained (can't run a full TLS stack or store large certificates), frequently deployed for years without physical access for updates, so the auth mechanism chosen at manufacture time needs to remain sound for that whole lifespan — certificate-based (X.509) device identity is the common answer, since it doesn't require ongoing interactive login.
- **APIs** — inherently stateless by convention even when the underlying mechanism (session cookies) technically supports state; scope-based permission models and rate limiting matter more here than in human-facing login flows, since API clients don't self-limit their request rate the way a human clicking a login button does.

## Key takeaways

1. Authentication should layer multiple defenses — no single control is sufficient (defense in depth).
2. Session-vs-token is the foundational early architectural choice; it determines how hard revocation is later.
3. Threat modeling and compliance requirements should shape the design from the start, not get bolted on afterward.
4. Passwordless/phishing-resistant methods (passkeys) are where the field is heading — see [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) for the deep dive.
5. Architecture shape (centralized/distributed/hybrid) has real, different trade-offs — pick deliberately based on the system's actual failure and scaling requirements, not by default.

## Further reading
- [1-authentication.md](<1-authentication.md>) — concrete mechanism catalog and comparison table.
- [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>) — JWT structure, revocation, algorithm attacks.
- [3-Oauth Guide.md](<3-Oauth Guide.md>) — OAuth 2.0 flows, PKCE, OIDC.
- [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) — WebAuthn/passkeys, SAML, ZKP implementations, risk-based auth implementations.
- [Authentication Security Best Practices.md](<Authentication Security Best Practices.md>) — concrete implementation of the OWASP/compliance guidance summarized above.
