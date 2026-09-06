# Secure Network Architecture — Multi-site (Hub & Spoke)

Design of a **multi-site enterprise security architecture** applying
**defense in depth** and **Zero Trust** principles: layered segmentation, strict
IT / OT separation, end-to-end encryption (mTLS + IPSec), and centralized
supervision by a SOC (SIEM / SOAR).

**Hub-and-spoke** topology: a headquarters (Casablanca) and two remote sites
(Laâyoune, Tanger) connected by encrypted tunnels.

> Design project for my cybersecurity coursework. The editable source is
> included: [`architecture_reseau.drawio`](architecture_reseau.drawio)
> (open at [app.diagrams.net](https://app.diagrams.net)).

---

## Layered overview

Traffic enters through the **Cloud EDGE** (SASE + DDoS protection) then crosses
four security layers, each with its own role. Baseline at every boundary:
**default deny + mTLS**.

```mermaid
flowchart TB
    EDGE["Cloud EDGE<br/>Prisma SASE + Cloudflare DDoS"]

    subgraph HUB["HUB — CASABLANCA (HQ)"]
        direction TB

        subgraph L1["LAYER 1 — Perimeter firewall"]
            FW["FortiGate 6000F cluster<br/>SD-WAN · IPSec VPN inter-site"]
            FM["FortiManager<br/>centralized rule management"]
            FA["FortiAnalyzer<br/>log centralization"]
        end

        subgraph L2["LAYER 2 — Application DMZs · mTLS + default deny"]
            subgraph DMZ1["CRM DMZ"]
                CRM["CRM applications"]
                WAF["WAF<br/>HTTP/S filtering · OWASP Top 10"]
                EDR["EDR<br/>endpoint detection"]
            end
            subgraph DMZ2["Automation DMZ — M2M"]
                API["Akamai API Gateway<br/>flow filtering · rate-limiting"]
                PAM["HashiCorp Vault + CyberArk PAM<br/>secrets · privileged access"]
            end
        end

        subgraph L3["LAYER 3 — Internal LAN · IT / OT separation"]
            FWIT["IT/OT firewall<br/>+ Data Diode (one-way flow)"]
            subgraph LANIT["IT LAN"]
                ITP["200 workstations · 20 printers"]
                NAC["NAC · device enrollment"]
                MSEG["Micro-segmentation · East-West"]
                UEM["UEM · patching · inventory"]
            end
            subgraph LANOT["OT LAN — Industrial"]
                IFW["Industrial FW<br/>Modbus · OPC-UA"]
                SCADA["SCADA / PLC · embedded IoT"]
                OTV["OT Visibility & Asset Discovery"]
            end
        end

        subgraph L4["LAYER 4 — Global SOC"]
            SIEM["SIEM · log correlation"]
            SOAR["SOAR · automated response"]
            MFA["MFA — Entra ID Protect"]
        end
    end

    EDGE -->|"mTLS"| L1
    L1 -->|"mTLS + default deny"| L2
    L2 --> FWIT
    FWIT --> LANIT
    FWIT -->|"one-way flow via Data Diode"| LANOT
    L3 -->|"logs & alerts upstream"| L4
```

Key security point: the `internal` OT flow is protected by a **data diode** that
makes the path **physically one-way** — the industrial world can be observed but
never reached from IT.

---

## Multi-site topology

Remote sites (spokes) replicate a local security stack (FW, SIEM, SOAR, MFA,
IT/OT VLAN separation) and report to the hub over **mTLS + IPSec AES-256**
tunnels. Policy is pushed from the hub via **FortiManager**.

```mermaid
flowchart TB
    subgraph HUBC["HUB — CASABLANCA"]
        HSEC["Central SOC · SIEM · SOAR · MFA<br/>FortiManager (central policy)"]
    end

    subgraph SP1["SPOKE — Laâyoune"]
        L_FW["local FW cluster · filtering & routing"]
        L_SIEM["SIEM · SOAR · MFA (local + upstream)"]
        L_IT["IT VLAN · 20 workstations"]
        L_OT["OT VLAN · Industrial / IoT"]
        L_FW --- L_IT
        L_FW --- L_OT
    end

    subgraph SP2["SPOKE — Tanger"]
        T_FW["local FW cluster · filtering & routing"]
        T_SIEM["SIEM · SOAR · MFA (local + upstream)"]
        T_IT["IT VLAN · 15 workstations"]
        T_OT["OT VLAN · Industrial / IoT"]
        T_FW --- T_IT
        T_FW --- T_OT
    end

    SP1 -->|"mTLS + IPSec AES-256 tunnel"| HUBC
    SP2 -->|"mTLS + IPSec AES-256 tunnel"| HUBC
    HUBC -->|"NAC policy via FortiManager"| SP1
    HUBC -->|"NAC policy via FortiManager"| SP2
```

---

## Network Access Control (NAC) policy

On connection, each device is identified by its **MAC OUI** (first 6 bytes =
vendor). Devices with no supplicant (printers, VoIP, IoT) go to a dedicated VLAN;
workstations and servers undergo full compliance checks before being granted
access.

```mermaid
flowchart TB
    START["Device attempts to connect"] --> NAC{"NAC — MAC OUI identification"}

    NAC -->|"Printer · VoIP · IoT<br/>(ZTE · Huawei · Nokia)"| VLAN120["→ VLAN 120<br/>exception (no supplicant)"]

    NAC -->|"Workstation / Server"| CHECK{"Compliance checks"}
    CHECK -->|"OS compliant<br/>AD domain-joined<br/>AV active & updated<br/>valid license<br/>login / password<br/>vendor MAC OUI (DELL)"| GRANT["Access granted<br/>→ IT VLAN"]
    CHECK -->|"any check fails"| DENY["Access denied<br/>→ quarantine"]
```

---

## Design decisions & rationale

| Decision | Why |
|----------|-----|
| **Defense in depth (4 layers)** | One barrier isn't enough; each layer limits propagation if the previous one falls. |
| **Default deny + mTLS everywhere** | Nothing is allowed by default; mutual auth prevents service impersonation. |
| **IT / OT separation + Data Diode** | Industrial systems (SCADA/PLC) must never be reachable from IT; the diode makes the flow **physically** one-way. |
| **Encrypted hub & spoke (IPSec AES-256)** | Centralizes policy and monitoring while keeping local autonomy per site. |
| **Global SOC (SIEM + SOAR)** | Correlates logs across all sites/layers + automated playbook response. |
| **NAC with OUI exceptions** | Realistic access control: you can't require a supplicant from a printer, hence the exception VLAN. |
| **East-West micro-segmentation** | Blocks lateral movement of an attacker already inside. |

---

## Technologies by domain

| Domain | Solutions |
|--------|-----------|
| Edge / SASE | Prisma SASE, Cloudflare (DDoS) |
| Firewall / SD-WAN | FortiGate 6000F, FortiManager, FortiAnalyzer |
| Application security | WAF (OWASP Top 10), EDR, Akamai API Gateway |
| Secrets / PAM | HashiCorp Vault, CyberArk |
| Network access | NAC, micro-segmentation, IT/OT VLANs, ACLs |
| Endpoints | UEM (patching, inventory), EDR |
| OT / Industrial | Industrial FW (Modbus, OPC-UA), Data Diode, OT visibility |
| SOC | SIEM, SOAR, MFA (Entra ID Protect) |
| Encryption | mTLS, IPSec AES-256 |

---

## Source file

The full original diagram (with graphical layout and color legend) is editable
at [`architecture_reseau.drawio`](architecture_reseau.drawio). To export it as an
image into `docs/`: open at [app.diagrams.net](https://app.diagrams.net) →
*File → Export as → PNG*.
