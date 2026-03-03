# license-authority-kms
RSA-based License Authority &amp; Key Management System (LAM-KMS) for secure microservice license enforcement. Features License Assertion Tokens (LAT), runtime integrity validation, host fingerprinting, Redis signed cache, heartbeat diagnostics, key rotation, and centralized kill-switch control for fintech-grade systems.

+----------------------------+
|     Incoming API Request   |
+----------------------------+
               |
               v
+----------------------------+
|     Check Redis License    |
+----------------------------+
               |
        +------+------+
        |             |
        v             v
+---------------+   +----------------------+
|     Valid     |   |  Expired / Missing   |
+---------------+   +----------------------+
        |                     |
        v                     v
+------------------+   +--------------------------+
| Process Request  |   |  Grace Period Valid ?    |
+------------------+   +--------------------------+
                              |
                      +-------+-------+
                      |               |
                      v               v
        +------------------------+   +-----------------------+
        | Allow + Async Renew    |   | Block Request         |
        | (Non-blocking)         |   | (Kill Switch Active)  |
        +------------------------+   +-----------------------+
                      |
                      v
        +--------------------------------------+
        | Call LAM/KMS (mTLS + RSA Validation) |
        +--------------------------------------+
                      |
                      v
        +------------------------------+
        | Update Redis Cache           |
        | - public_key                 |
        | - key_version                |
        | - expiry / grace window      |
        +------------------------------+








