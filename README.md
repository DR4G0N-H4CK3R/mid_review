# Cross-Layer Verification of an Untrusted IoT Gateway

> **Detecting a compromised smart-home hub with an independent radio witness**

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Tests: 18 Passed](https://img.shields.io/badge/pytest-18%20passed-brightgreen.svg)](tests/)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Windows%20%7C%20Raspberry%20Pi-orange.svg)]()

---

## 📌 Project Overview

In smart home deployments (Zigbee, Thread, Matter), peripheral end-devices and battery-powered sensors cannot route traffic directly to the IP network. Instead, every frame passes through an IoT gateway/coordinator (e.g., Home Assistant, Philips Hue Bridge, SmartThings) that translates IEEE 802.15.4 / Zigbee frames into IP-side telemetry, automations, and cloud reports.

Because the hub is the sole entity bridging both domains, existing security solutions (NIDS, TLS, secure boot, attestation) take the gateway's reported state on trust. However, real-world vulnerabilities (e.g., **CVE-2020-6007** Hue Bridge heap overflow, **CVE-2023-27482** Home Assistant auth bypass, SmartThings RCE chains) demonstrate that hubs can be compromised, allowing an attacker to manipulate, suppress, or forge state reports.

**Cross-Layer Gateway Verifier** treats the IoT gateway as an untrusted, Byzantine $\text{RF} \leftrightarrow \text{IP}$ translator. By deploying an independent, non-intrusive radio witness (e.g., nRF52840 / CC2531 sniffer) alongside a passive IP network tap, this engine validates the gateway's northbound IP reporting against physical/link-layer radio ground truth using a set of formal cross-layer invariants.

---

## 👥 Authors & Academic Context

- **Student Researchers:**
  - Bhaskar S L (`AM.SC.U4CYS23017`)
  - Arjun S (`AM.SC.U4CYS23012`)
- **Guide:** Dr. Kurunandan Jain
- **Mentor:** Pavan Raja
- **Institution:** Amrita Vishwa Vidyapeetham — Department of Cybersecurity (S7 Project Mid Review)

---

## 🔑 Key Features & Novelty

1. **New Trust Boundary:** Treats the hub as a potentially Byzantine translator verified by an out-of-band observer that the gateway cannot manipulate, compromise, or bypass.
2. **Metadata-Only Analysis:** Operates strictly on cleartext MAC/NWK headers, timing, lengths, and IP-flow metadata—no decryption of Zigbee network keys or TLS payloads required.
3. **Four Formal Cross-Layer Invariants:**
   - **Conservation:** Ensures reported state changes quantitatively correlate with physical sensor transmissions ($R_{\text{reports}} / R_{\text{received}}$).
   - **Provenance:** Flags hub IP events attributed to a device when no corresponding radio uplink occurred within a temporal tolerance band ($\Delta t \le 1.0\,\text{s}$).
   - **Translation:** Detects semantic tampering, payload length inflation, or abnormal delays against learned device commissioning signatures.
   - **Emitter Consistency:** Uses physical-layer features (RSSI/LQI) or link-layer fallback (MAC counter discontinuities and duplicate ACKs) to catch gateway radio spoofing/forgery.
4. **Loss-Aware Accounting:** Distinguishes genuine packet loss from gateway suppression using MAC sequence gap analysis and orphan ACK recovery, tracking capture completeness ($C$).
5. **Fail-Safe Degraded Reporting:** If radio visibility drops below $C < 0.70$ or retransmissions exceed $40\%$, windows transition to `DEGRADED` rather than generating false negatives or positives.
6. **$k$-of-$n$ Temporal Persistence:** Eliminates transient burst noise by requiring $\ge 3$ of the last 5 evaluable windows to concur before raising a confirmed alert.

---

## 🏗 System Architecture

```
+---------------------------------------------------------------------------------------+
|                                    LOCAL IOT SITE                                     |
|                                                                                       |
|   +---------------+            +--------------------------------------------------+   |
|   | Smart Devices |            |           DATA CAPTURE & PREPROCESSING           |   |
|   | (Bulb, Sensor,| --Zigbee-> |  - RF Sniffer (CC2531 / nRF52840, DLT 195/283)   |   |
|   | Socket, etc.) |            |  - IP Tap (Gateway TLS Records / WebSockets)     |   |
|   +---------------+            +--------------------------------------------------+   |
|           |                                             |                             |
|           v                                             v                             |
|    [Untrusted Hub]                             [Feature Extraction]                   |
|    (RF<->IP Bridge)                                     |                             |
|           |                                             v                             |
|           v                             [Cross-Layer Invariant Evaluation]            |
|       (IP Flow)                         - Conservation     - Provenance               |
|           |                             - Translation      - Emitter Consistency      |
|           v                                             |                             |
|   +---------------+                                     v                             |
|   | Northbound IP |                       [Temporal Consensus Engine]                 |
|   | Egress        |                            (3-of-5 Windows)                       |
|   +---------------+                                     |                             |
|           |                                             v                             |
|           +-----------------------------------> [Attributed Alerts]                   |
|                                             (SQLite, JSON, MQTT, UI)                  |
+---------------------------------------------------------------------------------------+
```

---

## 📊 Evaluation & Empirical Results

The verification engine was validated using **15.4 hours** of real 802.15.4 mesh traffic across 21 commercial devices from the **ZIOTP2025** benchmark dataset (252,896 frames, Topologies A & B), paired with Home Assistant OS tcpdumps.

### Attack Detection Performance (Real Captures)

| Attack Misbehaviour Class | Detection Rate (Top A) | Detection Rate (Top B) | Median Latency | Attribution Accuracy |
| :--- | :---: | :---: | :---: | :---: |
| **Leak (Exfiltration)** | **1.00** | **1.00** | **10 s** | 0.71 |
| **Change (Tamper)** | **1.00** | 0.44 | 42 s | 0.69 |
| **Forge (Spoofed Frames)** | **0.61** | **0.50** | **26 s** | 0.62 *(no RSSI)* |
| **Delay** | 0.67 | 0.33 | 50 s | 0.49 |
| **Swap (Misattribution)** | 0.67 | 0.33 | 50 s | 0.49 |
| **Invent (Fabrication)** | 0.58 | 0.33 | 72 s | 0.43 |
| **Hide (Suppression)** | 0.58 | 0.28 | 50 s | 0.47 |

*Note: Forgery detection was achieved entirely via link-layer counter continuity fallback, as public 802.15.4 dumps lack physical RSSI/LQI metadata.*

### Invariant vs. Fingerprint Transferability

Unlike state-of-the-art classifier fingerprints (e.g., ZIOTP2025 baseline type F1 dropping from $0.95 \to 0.74$, device ID F1 collapsing from $0.89 \to 0.41$ when transferring between topologies), **our behavioral conservation ratio transfers within $\approx 10\%$**:
- **Bulb:** 0.77 (Top A) vs. 0.74 (Top B)
- **Socket:** 0.59 (Top A) vs. 0.62 (Top B)
- **Motion Sensor:** 0.74 (Top A) vs. 0.83 (Top B)

---

## ⚙️ Installation & Setup

### Prerequisites

- Python 3.10+
- `pip` or virtual environment manager
- Wireshark / TShark (for live packet streaming / pcapng inspection)
- Optional: Docker (for containerized deployment on Raspberry Pi)

### Setup

```bash
# Clone the repository
git clone https://github.com/<your-username>/cross_layer_gateway_verifier.git
cd cross_layer_gateway_verifier

# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install required dependencies
pip install -r requirements.txt
```

---

## 🚀 Usage

### 1. Launching the Live Verification Dashboard
Replay pre-captured real paired streams (or simulated traces) through the local web dashboard:

```bash
python real_main.py --root ZIOTP2025 --topology A --scenario mixed
```
Open [http://127.0.0.1:8000/](http://127.0.0.1:8000/) in your browser to inspect:
- Real-time decision timeline (Normal, Candidate, Confirmed Anomaly, Degraded).
- Rolling capture completeness ($C$), packet counters, and orphan ACK recovery.
- Per-device invariant health matrices and live violation feeds.

### 2. Verifying an Arbitrary PCAP Pair
Verify an offline radio capture against a gateway IP capture:

```bash
python scripts/verify_capture.py verify \
  --radio path/to/radio_capture.pcapng \
  --ip path/to/hub_tcpdump.pcapng \
  --gateway-ip 192.168.8.116 \
  --server-ports 8123 \
  --state state/model \
  --out state
```

### 3. Running Benchmark Experiments
Reproduce the full 3-fold cross-validation experiment on the ZIOTP2025 dataset:

```bash
python scripts/ziotp_experiments.py --root ../ZIOTP2025 --out results/ziotp
```

### 4. Running Test Suite
Execute the unit and integration tests (including ground-truth validation checks):

```bash
python -m pytest tests
```

---

## 🗂 Project Structure

```
cross_layer_gateway_verifier/
├── data/                      # Local trace captures & injection scripts
├── scripts/
│   ├── verify_capture.py      # CLI for running verification on raw pcap/pcapng
│   ├── ziotp_experiments.py   # Benchmark evaluation runner for ZIOTP2025
│   ├── run_ablations.py       # Ablation studies (no radio, no RSSI, rules only)
│   └── run_detection_floor.py # Detection floor parameter sweeps
├── src/
│   ├── capture/               # Radio (DLT 195/283) and IP flow segmentation
│   ├── accounting/            # Loss accounting, orphan ACKs, completeness (C)
│   ├── invariants/            # Conservation, Provenance, Translation, Emitter
│   ├── models/                # Optional R-GCN relational residual model
│   ├── consensus/             # k-of-n window temporal voting logic
│   └── sinks/                 # Alert formatting (SQLite, JSON, MQTT)
├── tests/                     # 18 pytest test cases (4 validating real dataset GT)
├── real_main.py               # Live dashboard and playback engine
├── requirements.txt
└── README.md
```

---

## 🧭 Roadmap

- [x] **Phase 1 (Weeks 1–6):** Formal invariant design, loss accounting, synthetic simulation.
- [x] **Phase 2 (Weeks 7–12):** ZIOTP2025 integration, mesh-aware sequence gap engine, trace-driven misbehaviour injection, live UI dashboard.
- [ ] **Phase 3 (Weeks 13–14):** Dedicated hardware testbed deployment (Raspberry Pi 4/5 + nRF52840 witness + Home Assistant Matter/Thread testbed).
- [ ] **Phase 4 (Weeks 15–16):** Live lying translator implementation, real physical RSSI verification, and final paper preparation.

---

## 📚 References

1. S. Marti, T. Giuli, K. Lai, M. Baker, *"Mitigating routing misbehavior in mobile ad hoc networks,"* ACM MobiCom, 2000.
2. C. Karlof, D. Wagner, *"Secure routing in wireless sensor networks: attacks and countermeasures,"* Ad Hoc Networks, 2003.
3. D.-G. Akestoridis et al., *"Zigator: Analyzing the security of Zigbee-enabled smart homes,"* ACM WiSec, 2020.
4. A. Boiano et al., *"Analyzing Zigbee traffic: datasets, classification and storage trade-offs,"* arXiv:2602.03140, 2026 (ZIOTP2025).
5. C. Fu, Q. Zeng, X. Du, *"HAWatcher: Semantics-aware anomaly detection for appified smart homes,"* USENIX Security, 2021.
