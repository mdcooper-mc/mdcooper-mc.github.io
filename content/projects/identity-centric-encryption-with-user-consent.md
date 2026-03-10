+++
title = "Identity‑Anchored Encryption"
date = 2025-06-10
categories = ["Security", "Architecture"]
tags = ["Okta", "Envelope Encryption", "JWT", "AES-GCM", "Go", "Python"]
+++

---

## Identity-Centric Security Model

This project demonstrates how encryption can be made **identity‑centric** — shifting control from purely key‑based
systems to ones anchored in user identity and consent. Each record is encrypted with its own random key, which is then
wrapped with the user’s public key. Even if one key is compromised, only that single record is exposed.

Decryption is never an automatic process. Instead, it is governed by a multi‑step verification sequence that begins with a valid identity token (JWT) issued by Okta. If the actor requesting access is not the original data owner, the system triggers an explicit user approval workflow. Only after identity is verified and consent is granted does the Key Management Service attempt to unwrap the record’s unique encryption key.

Every decrypt event is logged with actor, subject, purpose, and key version, ensuring a defensible audit trail. This makes the Identity Provider the **front door for decryption** — the arbiter of whether access is legitimate.

---

## Envelope Encryption with Okta Approval

### Plain-language Summary

Our system is designed with a fundamental focus on keeping customer information secure through a multi‑layered approach. At its core, every individual piece of data is locked with its own unique digital padlock (a random key). To further enhance security, each padlock is then sealed inside a protective box that can only be opened with the customer’s personal key.

When access is needed — for example, if a support staff member needs to view the data to assist — they must first prove their identity by signing in. If the person requesting access is not the customer themselves, the system requires explicit approval from the customer before proceeding. Only once these conditions are met can the system unlock the padlock and reveal the information. Throughout this entire process, every action is meticulously recorded, creating a clear and auditable trail of who accessed what and for what purpose.

This architecture ensures that customers remain in full control of their own information at all times. Support staff are empowered to provide help, but their access is strictly limited by the customer’s permission. Furthermore, by using individual keys for every record, we ensure that even if one padlock were ever compromised, the impact would be confined to that single piece of data rather than the entire system.

---

### Architectural Overview

The model operates by encrypting each individual record with a random **Data Encryption Key (DEK)**, which is then wrapped using the user’s **public key**. Accessing the data requires a valid **JWT** and, if the request is being made through impersonation, explicit **user approval** to unwrap the DEK. To maintain accountability, every decryption event is **audited** with both the actor and subject identities.

---

## Write Path (Encrypt)

The write process begins by identifying the subject using their stable Okta ID. The system then generates a unique, one‑time **Data Encryption Key (DEK)** using the AES‑GCM algorithm to encrypt the record. Once the **ciphertext** is created, the DEK is itself wrapped with the subject’s **public key**. The resulting record, which includes the subject ID, ciphertext, wrapped DEK, and key version identifier, is then stored as shown in the following JSON example:

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

Decryption starts with the authentication of the actor, validating their Okta JWT. If the person attempting to access the data is not the subject, the system triggers an Okta approval flow and only proceeds once consent is explicitly granted. After these checks, the system retrieves the ciphertext and wrapped DEK. The DEK is then unwrapped using the subject’s private key — which is held securely within a Key Management Service or on a trusted device. With the original DEK restored, the system decrypts the ciphertext into its original plaintext. Finally, every step is recorded in an audit log that captures the actor, subject, approval status, and timestamp.

---

## Key Concepts

The system relies on several core components to maintain its identity‑centric security model. The **Data Encryption Key (DEK)** is a short‑lived, random key used only for the initial encryption of data, remaining in memory only for as long as necessary. To protect these keys, we use a **Public/Private Key Pair (PPK)**; the public key is used for wrapping DEKs, while the private key is kept securely within a KMS to unwrap them when needed. The **Approval Flow** is a critical security layer that uses Okta push or email prompts to ensure consent is granted whenever someone other than the data owner requests access. Finally, the **Audit Trail** ensures that every decryption event is logged with full context, including the actor, subject, purpose, and key version.

---

## Summary of Operations

In summary, the encryption process takes a subject ID and their data to generate a per‑record DEK, which is then used to encrypt the data before being wrapped by the user’s public key. The corresponding decryption process requires a valid JWT and any necessary Okta approvals to unwrap that DEK using the private key, ultimately revealing the original plaintext after a successful audit event.

---

## Identity Provider as the Front Door

In this model, the **Identity Provider (Okta)** serves as both the authentication mechanism and the primary policy gatekeeper for all decryption activities. The IdP issues JWTs that bind actor and subject identities, enforces necessary approval workflows such as push or email consent, and provides the claims that the Key Management Service (KMS) uses to decide whether to unwrap a DEK.

The Key Management Service holds the essential private keys and only performs unwrapping operations when the IdP‑backed claims confirm authorization. Each such operation is then logged with full context. On the application side, the system stores the ciphertext and wrapped DEKs and orchestrates calls to both the IdP and KMS when a decrypt is requested. Plaintext is only returned to the user after both identity and approval checks have successfully passed.

**Key takeaway:** By making Okta the front door for decryption, the model moves from a traditional key‑centric system to an **identity‑centric** one, where every access request is audited and governed by explicit user consent.

---

## Skills and Technologies

- **Identity & Access Management**: Okta SDKs, JWT, Okta Approval Flows, Identity-Centric Security.
- **Languages**: Python, Go, Rust.
- **Cryptography**: Envelope Encryption, AES-GCM, Per-record DEKs, Public/Private Key Wrapping, KMS/HSM Integration.
- **Architecture**: Modular Microservices, Consent-Driven Access, Regulatory Defensibility.
- **Documentation**: Narrative-driven Markdown, Technical and Business-friendly Documentation.