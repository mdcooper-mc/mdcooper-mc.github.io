+++
title = "Identity‑Anchored Encryption"
description = "An identity‑centric encryption model where every record is protected with its own key, and decryption requires verified identity, explicit user approval, and full auditability."
+++

This project demonstrates how encryption can be made **identity‑centric** — shifting control from purely key‑based
systems to ones anchored in user identity and consent. Each record is encrypted with its own random key, which is then
wrapped with the user’s public key. Even if one key is compromised, only that single record is exposed.

Decryption is never automatic. It requires:

1. A valid identity token (JWT) issued by the Identity Provider (Okta).
2. Explicit user approval if the actor requesting access is not the data owner.
3. A successful unwrap of the record’s encryption key by the Key Management Service.

Every decrypt event is logged with actor, subject, purpose, and key version, ensuring a defensible audit trail. This
makes the Identity Provider the **front door for decryption** — the arbiter of whether access is legitimate.

---

# Per-user Envelope Encryption with Okta Approval

## Plain-language Summary (for non-technical readers)

We are designing a system to keep each customer’s information safe.  
Here’s the simple idea:

- Every piece of data is locked with its own unique “padlock” (a random key).
- That padlock is then sealed inside a box that only the customer’s personal key can open.
- When someone (like support staff) needs to see the data, they must:  
  &nbsp;&nbsp;&nbsp;&nbsp;1. Prove who they are by signing in.  
  &nbsp;&nbsp;&nbsp;&nbsp;2. Get the customer’s approval if they are not the customer themselves.  
  &nbsp;&nbsp;&nbsp;&nbsp;3. Only then can the system unlock the padlock and show the information.
- Every time this happens, it is recorded so there’s a clear trail of who accessed what and why.

This means:

- Customers stay in control of their own information.
- Support staff can help, but only with permission.
- Even if one padlock is broken, it only affects that single piece of data, not everything.

---

## Overview

- Encrypt each record with a random **Data Encryption Key (DEK)**.
- Wrap the DEK with the user’s **public key**.
- Require a valid **JWT** and **user approval** (if impersonated) to unwrap the DEK.
- Audit every decrypt with both actor and subject identities.

---

## Write Path (Encrypt)

1. **Identify subject**  
   `user_sub` = stable ID from Okta.

2. **Generate DEK**  
   Random symmetric key (AES-GCM), used only for this record.

3. **Encrypt data**  
   `ciphertext = Encrypt(data, DEK)`

4. **Wrap DEK**  
   `wrappedDEK = Encrypt(DEK, user_public_key)`

5. **Store record**
   ```json
   {
   "user_sub": "00u123...",
   "ciphertext": "...",
   "wrappedDEK": "...",
   "kid": "k_v1"
   }
   ```

---

## Read Path (Decrypt)

1. **Authenticate actor**
    - Validate JWT from Okta.
    - If actor ≠ subject → trigger Okta approval flow.
    - Only proceed if approval granted.

2. **Fetch record**
    - Retrieve `ciphertext`, `wrappedDEK`, `kid`.

3. **Unwrap DEK**  
   `DEK = Decrypt(wrappedDEK, user_private_key)`  
   (private key held securely in KMS or user device).

4. **Decrypt ciphertext**  
   `plaintext = Decrypt(ciphertext, DEK)`

5. **Audit**
    - Log `actorSub`, `subjectSub`, `approval`, `kid`, `timestamp`.

---

## Key Concepts

- **DEK:** Short-lived, random key used to encrypt actual data; exists only in memory during operations.
- **Public/Private Key Pair (PPK):**  
  &nbsp;&nbsp;- Public key: stored in Okta or a key registry; used to wrap DEKs.  
  &nbsp;&nbsp;- Private key: never leaves secure storage; used to unwrap DEKs.
- **Approval Flow:** Okta push/email prompt ensures the subject consents when actor ≠ subject.
- **Audit Trail:** Every decrypt event records actor, subject, purpose, and key version.

---

## One-Liner Summaries

- **Encrypt:**  
  `(user_sub, data → [generate DEK → encrypt data → wrap DEK with user’s public key])`

- **Decrypt:**  
  `(JWT + Okta approval → unwrap DEK with user’s private key → decrypt ciphertext → return plaintext)`

---

## Your Shorthand Mnemonics

- **Encrypt:**  
  [user_sub, data (PPK > DEK >> data)]

- **Decrypt:**  
  JWT (approval flow via Okta [user/supportuser]) → DEK unwrap → decrypt key → decrypt(enc_text)

---

## Identity Provider as the Front Door

In this model, the **Identity Provider (Okta)** is not just authenticating users but also acting as the **policy
gatekeeper** for decryption:

- **Identity Provider (IdP) role:**  
  &nbsp;&nbsp;- Issues JWTs binding actor identity (`actor_sub`) and subject identity (`user_sub`).  
  &nbsp;&nbsp;- Enforces approval workflows (push/email consent).  
  &nbsp;&nbsp;- Provides claims that the KMS/decrypt service uses to decide whether to unwrap the DEK.

- **Key Management role:**  
  &nbsp;&nbsp;- Holds private/master keys.  
  &nbsp;&nbsp;- Only unwraps DEKs if IdP-backed claims say “yes.”  
  &nbsp;&nbsp;- Logs every unwrap with actor/subject context.

- **Application role:**  
  &nbsp;&nbsp;- Stores ciphertext + wrapped DEK.  
  &nbsp;&nbsp;- Calls out to IdP + KMS when a decrypt is requested.  
  &nbsp;&nbsp;- Returns plaintext only after both identity and approval checks pass.

**Key takeaway:** Okta becomes the **front door for decryption** — the arbiter of whether a decrypt request is
legitimate. This shifts the model from being purely key-centric to being **identity-centric**, with consent and
auditability built in.

---

## Skills

**Languages & Frameworks**  
Python | Go | Rust | Markdown Documentation | Okta SDKs

**Security & Identity**  
JWT | Okta Approval Flows | Consent‑Driven Access | Audit Logging

**Encryption & Key Management**  
Envelope Encryption | Per‑Record DEKs | AES‑GCM | KMS/HSM Integration | Public/Private Key Wrapping

**Architecture & Design**  
Identity‑Centric Security | Modular Microservices | Consent‑First Workflows | Regulatory Defensibility

**Documentation & Communication**  
Narrative‑Driven Markdown | Business‑Friendly Summaries | Shorthand Mnemonics | Example‑Rich Use Cases