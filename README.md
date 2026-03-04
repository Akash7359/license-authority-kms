# 🔐 lam-kms

> **License Authority & Key Management System (RSA Based)** — Centralized RSA cryptographic license enforcement, daily key rotation, kill-switch control, and tamper detection for fintech microservices.

[![Stack](https://img.shields.io/badge/stack-Laravel%20%7C%20Redis%20%7C%20RSA-1a56db?style=flat-square)](/)
[![Crypto](https://img.shields.io/badge/crypto-RSA%202048%2B%20%7C%20SHA--256-7c3aed?style=flat-square)](/)
[![Transport](https://img.shields.io/badge/transport-mTLS%20%2B%20TLS%201.2%2B-0e9f6e?style=flat-square)](/)
[![Compliance](https://img.shields.io/badge/compliance-RBI%20%7C%20PCI--DSS-e3a008?style=flat-square)](/)
[![Version](https://img.shields.io/badge/version-1.0-gray?style=flat-square)](/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

---

## 📑 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Data Flow](#data-flow)
- [License Validation Flow](#license-validation-flow)
- [License Assertion Token (LAT)](#license-assertion-token-lat)
- [Diagnostic & Tamper Detection](#diagnostic--tamper-detection)
- [Mandatory Microservice Integration](#mandatory-microservice-integration)
- [Kill Switch & Grace Mode](#kill-switch--grace-mode)
- [Data Model](#data-model)
- [API Reference](#api-reference)
- [Non-Functional Requirements](#non-functional-requirements)
- [Error & Exception Handling](#error--exception-handling)
- [Forbidden Developer Actions](#forbidden-developer-actions)
- [Deployment](#deployment)

---

## Overview

`lam-kms` is a **centralized License Authority & Key Management System** built for fintech microservice architectures. It enforces software licensing using **RSA public/private key cryptography**, providing daily key rotation, cryptographically bound license tokens, runtime tamper detection, and a remotely-triggered kill switch — all without ever exposing the private key to client environments.

**Key capabilities:** License enforcement for backend microservices · Daily RSA key rotation and distribution · Kill switch and grace-period handling · Code integrity validation · Host fingerprinting · Forensic audit logging

**Prepared by:** Akash Sonawane · v1.0 · 2026-01-14 · Reviewed by: Architecture Review Board · Approved by: CTO / CISO

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT LAYER                         │
│   Client / API Gateway                                  │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│              PROJECT MICROSERVICES (Laravel)            │
│   License Middleware — intercepts every request         │
│   Code Hash Validator · Host Fingerprint Check          │
│   Heartbeat Scheduler                                   │
└────────────┬────────────────────────────────────────────┘
             │ Cache hit               │ Cache miss / expired
             ▼                         ▼
┌────────────────────┐    ┌────────────────────────────────┐
│   Redis License    │    │   License Authority & KMS      │
│   Cache            │    │   (LAM)                        │
│   (signed entries) │    │   mTLS + RSA Key Management    │
└────────────────────┘    │   License Validation           │
                          │   Kill Switch Enforcement      │
                          └────────────────┬───────────────┘
                                           │
                          ┌────────────────▼───────────────┐
                          │   License / RSA Key Database   │
                          │   projects · project_keys      │
                          │   project_microservices        │
                          └────────────────────────────────┘
```

**Design principles:**
- Private RSA keys **never leave** LAM/KMS
- Microservices are **verification-only** — they cannot assert or modify license state
- Redis is a **performance cache**, never a trust source
- All service-to-service communication uses **mTLS**

---

## Data Flow

### Level 0 — Context Diagram

```
+-------------------+        +---------------------------+
|   Client / API    | -----> |   Project Microservice    |
+-------------------+        +---------------------------+
                                          |
                                          v
                             +----------------------------------+
                             |  License Authority & KMS (LAM)  |
                             |   (TLS + RSA Key Management)    |
                             +----------------------------------+
```

### Level 1 — Detailed DFD

```
+-----------+      +------------------+      +----------------+
|  Client   | ---> | Project Service  | ---> |  Redis Cache   |
+-----------+      +------------------+      +----------------+
                          |       ^
                          |       |
        License Expired   |       |  Valid License (signed)
                          v       |
          +----------------------------------------------+
          |     License Authority & KMS (LAM)            |
          |     - mTLS Authentication                    |
          |     - RSA Public/Private Key Management      |
          |     - License Validation & Issuance          |
          |     - Kill Switch Enforcement                |
          +----------------------------------------------+
                          |
                          v
               +-----------------------------+
               |   License / RSA Key DB      |
               +-----------------------------+
```

---

## License Validation Flow

```
Incoming API Request
        |
        v
Check Redis License Cache
        |
   +---------+-----------+
   |                     |
   v                     v
 Valid             Expired / Missing
   |                     |
   v                     v
Process             Grace Period Valid?
Request                  |
                  +------+------+
                  |             |
                  v             v
         Allow + Async      Block Request
         Renew (Non-         (Kill Switch
         blocking)           Active)
                  |
                  v
    Call LAM/KMS (mTLS + RSA Validation)
                  |
                  v
    Update Redis Cache
      - public_key
      - key_version
      - expiry / grace window
```

---

## License Assertion Token (LAT)

LAM issues a **License Assertion Token (LAT)** cryptographically signed using the LAM private RSA key (SHA-256). The LAT is the single authoritative license state artifact.

**LAT Properties:** RSA signed (SHA-256) · Short-lived and non-replayable · Verifiable but non-modifiable by clients

**LAT Payload Structure:**

```json
{
  "project_id":        "FINTECH_X",
  "service_name":      "payments-service",
  "service_hash":      "sha256_code_manifest",
  "host_fingerprint":  "hashed_machine_identity",
  "key_version":       "v7",
  "issued_at":         "2026-01-14T10:00:00Z",
  "expires_at":        "2026-01-15T10:00:00Z",
  "license_status":    "ACTIVE"
}
```

Microservices may **only verify** the LAT using the public key fetched from LAM. LAT modification, regeneration, or local override is strictly forbidden.

### LAT Verification Rules

Every microservice must validate all of the following on each request:

- LAT RSA signature is valid (using LAM public key)
- `project_id` and `service_name` match runtime service identity
- `key_version` is currently active
- `expires_at` is in the future

---

## Diagnostic & Tamper Detection

### Control Principles

| Principle | Description |
|---|---|
| Centralized Authority | LAM is the single source of truth for license validity |
| Zero Trust Execution | Microservices never self-assert license state |
| Cryptographic Binding | License is bound to service identity, code state, and host |
| Time-Bound Trust | All licenses expire and require revalidation |
| Detect Over Prevent | System prioritizes detection and controlled shutdown |

### Runtime Code Integrity

Each microservice generates a code manifest at build time and recalculates the hash at startup and at fixed runtime intervals. The hash is compared against `service_hash` inside the LAT.

```bash
# Build-time manifest generation
sha256sum app/ bootstrap/ vendor/ > service.manifest
```

| Integrity State | Action |
|---|---|
| Hash Match | Normal request processing |
| Temporary Mismatch | Grace Mode + Alert to LAM |
| Persistent Mismatch | License revocation + kill switch activation |

### Host Environment Fingerprinting

To prevent license reuse across unauthorized environments, each service derives a host fingerprint from CPU identifier, primary disk UUID, and MAC address. The combined fingerprint is hashed and embedded in the LAT. Any mismatch immediately invalidates the license.

### Heartbeat & Behavioral Diagnosis

Each microservice periodically sends a diagnostic heartbeat to LAM:

```json
{
  "service_name":           "payments-service",
  "lat_version":            "v7",
  "license_state":          "ACTIVE",
  "redis_hit_ratio":        98,
  "failed_license_checks":  0,
  "uptime_seconds":         86400
}
```

**Behavioral anomalies detected:** Missing or delayed heartbeats · Repeated grace-mode activations · Redis-only validation patterns · License refresh suppression attempts

### Redis Trust Degradation Model

Redis is treated strictly as a **performance cache**, never a trust source. All Redis license entries include RSA signatures, which are verified on every read. Unsigned or invalid entries are silently ignored. Redis availability issues must never disable license validation logic.

### Forensic Audit Logging

LAM maintains immutable audit logs for all diagnostic events including code hash mismatches, heartbeat anomalies, license issuance and revocation, and key rotation history. These logs provide regulatory, contractual, and legal evidence for compliance audits and client disputes.

---

## Mandatory Microservice Integration

### Startup Diagnostic Sequence

Every microservice must execute the following sequence **before accepting any traffic**. Failure at any step prevents the service from starting or forces immediate shutdown.

```
1. Load service identity (project_id, service_name)
2. Fetch active LAM public key via mTLS
3. Load last known License Assertion Token (LAT)
4. Verify LAT RSA signature
5. Validate LAT expiry and key_version
6. Compute runtime code hash
7. Validate host fingerprint
8. Load Redis cache (signed entries only)
9. Register heartbeat scheduler
```

### License Middleware Enforcement

All request handling layers must be protected by a centralized **License Middleware**. No controller, worker, job, or cron may bypass this middleware. If middleware is disabled or removed, LAT renewal fails and the service enters kill-switch state.

### Heartbeat Scheduler Responsibilities

Each microservice must run a background heartbeat scheduler that reports: license state · integrity status · grace-mode usage · Redis vs LAM validation ratio. Missing or delayed heartbeats are treated as behavioral anomalies.

### End-to-End Diagnostic Lifecycle

```
Service Startup
      |
Verify LAT Signature
      |
Validate Code Hash & Host Fingerprint
      |
Intercept All Requests via License Middleware
      |
Send Periodic Heartbeat to LAM
      |
LAM Decision → Allow | Grace Mode | Kill Switch
```

---

## Kill Switch & Grace Mode

| Scenario | Enforcement Action |
|---|---|
| LAM temporary unavailability | Grace Mode (time-bound) |
| License expired | Grace Mode + forced renewal |
| Code integrity failure | Immediate Kill Switch |
| Host fingerprint mismatch | Immediate Kill Switch |
| License revoked | Immediate Kill Switch |
| Project status = SUSPENDED / REVOKED | Immediate Kill Switch |

The kill switch is **data-driven** and enforced through license state, not application code. If enforcement logic is removed from the microservice code, LAT renewal fails and the service shuts down on the next validation cycle.

---

## Data Model

### ER Diagram

```
+--------------------+
|      projects      |
+--------------------+
| PK id              |
|    name            |
|    status          |
|    license_expiry  |
|    created_at      |
|    updated_at      |
+--------------------+
          | 1
          |
          | *
+---------------------------+
|      project_keys         |
+---------------------------+
| PK id                     |
| FK project_id             |
|    key_version            |
|    encrypted_key          |
|    valid_from             |
|    valid_to               |
|    is_active              |
+---------------------------+
          | 1
          |
          | *
+---------------------------+
|  project_microservices    |
+---------------------------+
| PK id                     |
| FK project_id             |
|    service_name           |
|    is_allowed             |
|    created_at             |
|    updated_at             |
+---------------------------+
```

### `projects`

| Column | Type | PK/FK | Nullable | Default | Description |
|---|---|---|---|---|---|
| `id` | BIGINT | PK | No | Auto | Unique project identifier |
| `name` | VARCHAR(100) | | No | | Project name |
| `status` | ENUM | | No | `ACTIVE` | `ACTIVE` / `SUSPENDED` / `REVOKED` |
| `license_expiry` | TIMESTAMP | | No | | License expiry datetime |
| `created_at` | TIMESTAMP | | No | | Record creation time |
| `updated_at` | TIMESTAMP | | No | | Record update time |

### `project_keys`

| Column | Type | PK/FK | Nullable | Default | Description |
|---|---|---|---|---|---|
| `id` | BIGINT | PK | No | Auto | Key record ID |
| `project_id` | BIGINT | FK | No | | Reference to `projects.id` |
| `key_version` | VARCHAR(10) | | No | | Version identifier (e.g. `v7`) |
| `encrypted_key` | TEXT | | No | | Encrypted RSA private key |
| `valid_from` | TIMESTAMP | | No | | Key activation time |
| `valid_to` | TIMESTAMP | | Yes | NULL | Key expiry time |
| `is_active` | BOOLEAN | | No | `FALSE` | Active key flag |

### `project_microservices`

| Column | Type | PK/FK | Nullable | Default | Description |
|---|---|---|---|---|---|
| `id` | BIGINT | PK | No | Auto | Record ID |
| `project_id` | BIGINT | FK | No | | Reference to `projects.id` |
| `service_name` | VARCHAR(100) | | No | | Microservice name |
| `is_allowed` | BOOLEAN | | No | `TRUE` | Access allowed flag |
| `created_at` | TIMESTAMP | | No | | Creation time |
| `updated_at` | TIMESTAMP | | No | | Update time |

---

## API Reference

### `GET /api/v1/license/public-key`

Fetch the active RSA public key for a project. Used by microservices to verify LAT signatures.

| Property | Value |
|---|---|
| Method | `GET` |
| Authentication | mTLS |
| Response | `public_key`, `key_version`, `valid_till` |

**Example response:**

```json
{
  "public_key":   "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----",
  "key_version":  "v7",
  "valid_till":   "2026-01-15T10:00:00Z"
}
```

---

## Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Redis license lookup < 10ms |
| Cryptography | RSA 2048+ · SHA-256 signing |
| Transport | TLS 1.2+ for all communication · mTLS for service-to-service |
| Availability | Zero-downtime key rotation |
| Compliance | RBI Cyber Security Framework · PCI-DSS aligned |
| Auditability | Immutable forensic logs for all license events |

---

## Error & Exception Handling

| Condition | Response |
|---|---|
| License expired | Enter Grace Mode + async renewal |
| Project SUSPENDED / REVOKED | Immediate Kill Switch |
| LAM temporarily unavailable | Use cached signed key within grace window |
| Code hash mismatch (single) | Grace Mode + alert to LAM |
| Code hash mismatch (repeated) | Immediate Kill Switch |
| Host fingerprint mismatch | Immediate Kill Switch + revocation |
| Unsigned Redis entry | Ignore entry, fall through to LAM |

---

## Forbidden Developer Actions

The following are strictly prohibited. Detection of any forbidden action results in **permanent license revocation**.

- Hardcoding license checks or flags in application code
- Mocking LAM responses in any non-development environment
- Bypassing or disabling the license middleware
- Extending LAT expiry locally without LAM issuance
- Suppressing heartbeat failures or altering heartbeat payloads
- Disabling runtime integrity hash checks
- Caching unsigned or self-generated license tokens in Redis

---

## Deployment

- Dockerized Laravel microservices with isolated environment secrets
- Private RSA keys stored in HSM or encrypted secrets manager — never in application code or environment files
- Redis protected using ACL and AUTH
- All inter-service routes restricted to mTLS-only listeners
- Rollback via Redis cache restore from last known valid signed state
- Key rotation is zero-downtime — new key version distributed before old key expires

---

## References

- Internal FRD — License Control Platform
- OWASP Cryptographic Storage Cheat Sheet
- RBI Cyber Security Framework
- PCI-DSS v4.0 Cryptographic Key Management Requirements

---

## Glossary

| Term | Definition |
|---|---|
| LAM | License Authority Microservice — the central issuing and enforcement authority |
| KMS | Key Management System — manages RSA key lifecycle and rotation |
| LAT | License Assertion Token — RSA-signed token representing current license state |
| mTLS | Mutual TLS — bidirectional certificate authentication between services |
| Grace Mode | Time-bound fallback state allowing continued operation during LAM unavailability |
| Kill Switch | Forced service shutdown triggered by license revocation or integrity failure |
| RSA | Rivest–Shamir–Adleman — asymmetric cryptographic algorithm used for signing |

---

## License

[MIT](LICENSE) © Your Organization

---

<p align="center">
  <code>Laravel · Redis · RSA 2048 · mTLS · SHA-256 · RBI · PCI-DSS</code><br/>
  <sub>License Authority & Key Management System v1.0 — Production Reference</sub>
</p>
