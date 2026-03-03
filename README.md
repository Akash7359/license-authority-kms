# license-authority-kms
RSA-based License Authority &amp; Key Management System (LAM-KMS) for secure microservice license enforcement. Features License Assertion Tokens (LAT), runtime integrity validation, host fingerprinting, Redis signed cache, heartbeat diagnostics, key rotation, and centralized kill-switch control for fintech-grade systems.

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>LAM/KMS Flowchart</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/10.6.1/mermaid.min.js"></script>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background: #0d1117; color: #e6edf3; padding: 32px 24px;
  }
  h1 { font-size: 22px; font-weight: 700; color: #f0f6fc; margin-bottom: 6px; }
  .subtitle { font-size: 13px; color: #8b949e; margin-bottom: 32px; }
  .tabs { display: flex; gap: 4px; border-bottom: 1px solid #30363d; }
  .tab {
    padding: 8px 16px; font-size: 13px; font-weight: 500; cursor: pointer;
    border-radius: 6px 6px 0 0; border: 1px solid transparent;
    color: #8b949e; background: transparent; user-select: none;
  }
  .tab:hover { color: #e6edf3; background: #161b22; }
  .tab.active { color: #f0f6fc; background: #161b22; border-color: #30363d; border-bottom-color: #161b22; }
  .panel {
    display: none; background: #161b22; border: 1px solid #30363d;
    border-top: none; border-radius: 0 6px 6px 6px; padding: 28px 24px;
  }
  .panel.active { display: block; }
  .panel-title {
    font-size: 14px; font-weight: 600; color: #8b949e;
    text-transform: uppercase; letter-spacing: 0.06em; margin-bottom: 20px;
  }
  .mermaid { display: flex; justify-content: center; overflow-x: auto; }
</style>
</head>
<body>

<h1>🔐 License Authority &amp; KMS — System Flowcharts</h1>
<p class="subtitle">RSA-Based License Validation, Startup Diagnostics &amp; Enforcement Lifecycle</p>

<div class="tabs">
  <div class="tab active" onclick="showTab(0)">License Validation</div>
  <div class="tab" onclick="showTab(1)">Startup Diagnostic</div>
  <div class="tab" onclick="showTab(2)">High-Level Architecture</div>
  <div class="tab" onclick="showTab(3)">Kill Switch Logic</div>
</div>

<div class="panel active">
  <div class="panel-title">License Validation &amp; Refresh Flow</div>
  <div class="mermaid">
flowchart TD
    A([Incoming API Request]) --> B[Check Redis License Cache]
    B --> C{License State?}
    C -- Valid --> D([Process Request])
    C -- Expired / Missing --> E{Grace Period Active?}
    E -- Yes --> F[Allow + Async Renewal]
    E -- No / Kill Switch --> G([Block Request])
    F --> H[Call LAM/KMS via mTLS + RSA]
    H --> I{LAM Response?}
    I -- License Valid --> J[Update Redis Cache]
    I -- Revoked/Suspended --> G
    J --> D
    style A fill:#1f6feb,color:#fff,stroke:none
    style D fill:#238636,color:#fff,stroke:none
    style G fill:#da3633,color:#fff,stroke:none
    style H fill:#6e40c9,color:#fff,stroke:none
  </div>
</div>

<div class="panel">
  <div class="panel-title">Mandatory Startup Diagnostic Sequence</div>
  <div class="mermaid">
flowchart TD
    S([Service Start]) --> S1[Load Service Identity]
    S1 --> S2[Fetch LAM Public Key via mTLS]
    S2 --> S3[Load Last Known LAT]
    S3 --> S4{Verify LAT RSA Signature}
    S4 -- Invalid --> KILL([Service Shutdown])
    S4 -- Valid --> S5{Validate LAT Expiry and key_version}
    S5 -- Expired --> KILL
    S5 -- OK --> S6[Compute Runtime Code Hash]
    S6 --> S7{Validate Host Fingerprint}
    S7 -- Mismatch --> KILL
    S7 -- Match --> S8[Load Signed Redis Cache]
    S8 --> S9[Register Heartbeat Scheduler]
    S9 --> READY([Accept Traffic])
    style S fill:#1f6feb,color:#fff,stroke:none
    style READY fill:#238636,color:#fff,stroke:none
    style KILL fill:#da3633,color:#fff,stroke:none
  </div>
</div>

<div class="panel">
  <div class="panel-title">High-Level System Architecture</div>
  <div class="mermaid">
flowchart LR
    Client([Client]) --> GW[API Gateway]
    GW --> MS[Project Microservice Laravel]
    MS <--> RC[(Redis Cache)]
    MS <-->|mTLS| LAM[License Authority and KMS]
    LAM <--> DB[(License / RSA Key DB)]
    subgraph LAM_INT [LAM Internals]
      A1[mTLS Auth] --> A2[RSA Key Mgmt] --> A3[LAT Issuance]
    end
    LAM --- LAM_INT
    style Client fill:#1f6feb,color:#fff,stroke:none
    style MS fill:#6e40c9,color:#fff,stroke:none
    style LAM fill:#9e6a03,color:#fff,stroke:none
    style RC fill:#1f6feb,color:#fff,stroke:none
    style DB fill:#238636,color:#fff,stroke:none
  </div>
</div>

<div class="panel">
  <div class="panel-title">Kill Switch &amp; Enforcement Decision Tree</div>
  <div class="mermaid">
flowchart TD
    HB([Heartbeat / Request Cycle]) --> IC{Code Hash Integrity?}
    IC -- Persistent Mismatch --> KS
    IC -- Pass --> HF{Host Fingerprint Match?}
    HF -- Mismatch --> KS
    HF -- Match --> LS{License Status?}
    LS -- ACTIVE --> ALLOW([Allow])
    LS -- Expired --> GP{Grace Period Remaining?}
    GP -- Yes --> GRACE[Grace Mode + Force Renewal]
    GP -- No --> KS
    LS -- SUSPENDED or REVOKED --> KS
    GRACE --> RENEW[Renew LAT from LAM]
    RENEW --> ALLOW
    KS([Kill Switch - Shutdown])
    style ALLOW fill:#238636,color:#fff,stroke:none
    style GRACE fill:#9e6a03,color:#fff,stroke:none
    style KS fill:#da3633,color:#fff,stroke:none
    style HB fill:#1f6feb,color:#fff,stroke:none
  </div>
</div>

<script>
  mermaid.initialize({
    startOnLoad: true,
    theme: 'dark',
    themeVariables: {
      primaryColor: '#1f6feb', primaryTextColor: '#e6edf3',
      primaryBorderColor: '#30363d', lineColor: '#8b949e',
      secondaryColor: '#161b22', tertiaryColor: '#0d1117',
      fontSize: '14px'
    }
  });
  function showTab(index) {
    document.querySelectorAll('.tab').forEach((t, i) => t.classList.toggle('active', i === index));
    document.querySelectorAll('.panel').forEach((p, i) => p.classList.toggle('active', i === index));
  }
</script>
</body>
</html>








