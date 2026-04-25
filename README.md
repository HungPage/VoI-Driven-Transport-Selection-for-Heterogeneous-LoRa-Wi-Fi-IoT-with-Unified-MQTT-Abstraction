# VoI-Driven-IoT-Transport-Selection

> **Project Version**: 1.0.0 (Research Prototype)
> **Status**: Ready for IEEE GCCE 2026 Submission
> **Author**: Jun-Sen Hung (TKU CSIE)

---

##  Project Abstract

Underground culvert flood monitoring faces a fundamental dilemma: **Wi-Fi enables fast transmission but drains batteries, while LoRa conserves energy but lacks bandwidth**. This project implements a transport selection mechanism driven by the semantic **Value of Information (VoI)** rather than RSSI alone.

We deploy five Raspberry Pi Pico 2W nodes with RYLR998 LoRa modules, classify water-level dynamics into four semantic states via deterministic hard rules, and unify both physical channels under a single MQTT abstraction layer. Results across four strategies show that VoI-Driven operation achieves **2.07× lower power than Always-WiFi** while uniquely employing all four link classes adaptively to traffic semantics.

---

##  System Architecture (Internal Logic)

The system is decoupled into three functional layers to ensure **channel transparency** between physical radios and upstream applications:

### Layer 0 — Sensing
Acquires water level (m) and computes the 10-minute delta (Δ10). A CatBoost student model (1 tree, depth 10, 8 features) classifies each reading into one of four semantic states.

### Layer 1 — VoI Hard Rule
Maps the semantic state to a **link class** via a deterministic table. No learning, no thresholds—just rules:

| State          | Trigger Condition          | Link Class            | TX Energy |
|----------------|---------------------------|-----------------------|-----------|
| S0 (Stable)    | wl < 6.5m, \|Δ10\| < 0.05 | B: LoRa-1×            | 100 mJ    |
| S1 (Rising)    | wl < 8.0m, Δ10 > 0        | C: Wi-Fi-ECO          | 400 mJ    |
| S2 (Alert)     | wl ≥ 8.0m, rising         | A: Wi-Fi-STD          | 600 mJ    |
| S3 (Receding)  | wl ≥ 6.5m, Δ10 < 0        | D: LoRa-2× (TimeDiv)  | 200 mJ    |

### Layer 2 — Routing & Resilience
- **DQN policy**: chooses DIRECT vs RELAY for the LoRa path
- **AODV mesh**: maintains routes among leaves for multi-hop forwarding
- **Bidirectional fallback**: LoRa↔Wi-Fi switch after 3 consecutive failures, recovery probed every 60s
- **Unified MQTT**: sink bridges both channels into one topic hierarchy (`sensor/node_XX/data`)

```
[Pico 2W Leaves N3-N7] ──LoRa 923MHz / Wi-Fi 2.4GHz──→ [Pico 2W Sink N1]
                                                            │
                                                            ↓ MQTT (port 1883)
                                                       [Laptop Broker]
                                                            ├── Mosquitto
                                                            ├── CSV Logger
                                                            └── Dashboard
```

---

##  Experimental Results

Five Pi Pico 2W nodes replay a typhoon water-level trace from the Nanshenjiao Bridge monitoring station. Four strategies evaluated, each ≥30 minutes:

| Strategy           | Avg Power | 24h Energy | Battery Life | Link Classes Used |
|--------------------|-----------|------------|--------------|-------------------|
| Always-LoRa        | 64.51 mW  | 5574 J     | 8.4 days     | 2 (B, D)          |
| Always-WiFi        | 177.85 mW | 15366 J    | 3.0 days     | 2 (A, C)          |
| Periodic           | 31.70 mW  | 2739 J     | 17.0 days    | 3 (A, B, D)       |
| **VoI-Driven**     | **86.08 mW** | **7437 J** | **6.3 days** | **4 (A, B, C, D)** |

**Key insight**: VoI-Driven is the **only** strategy that exercises all four link classes within a single run, dynamically routing 34% of traffic through Wi-Fi-ECO (S1 pre-alerts), 5% through Wi-Fi-STD (S2 critical alerts), and 61% through LoRa variants. Periodic appears most efficient in steady-state but is **event-blind**: alerts may be delayed up to 10 minutes by the fixed Wi-Fi window.

---

##  Repository Structure

```
.
├── N234_node_student_model_integrated.py   # Leaf firmware (N3-N7)
├── N1sink_student_model_compatible.py      # Sink firmware (N1)
├── mqtt_csv_logger.py                      # Laptop MQTT→CSV recorder
├── compute_energy.py                       # Energy analysis (Table I)
├── Plot.py                                 # Figure generator (Fig 3, Fig 5)
├── index.html                              # Real-time dashboard
├── N1sink.txt                              # Sample sink console log
├── N234..node.txt                          # Sample leaf console log
├── README.md
└── experiments/
    ├── run1_voi/         # VOI_DRIVEN  — proposed
    ├── run2_lora/        # ALWAYS_LORA — baseline
    ├── run3_wifi/        # ALWAYS_WIFI — baseline
    ├── run4_periodic/    # PERIODIC    — baseline
    └── unknown/          # transient packets (ignorable)
```

---

##  Hardware Setup

| Component                  | Qty | Notes                                      |
|----------------------------|-----|--------------------------------------------|
| Raspberry Pi Pico 2W       | 6   | 5 leaves + 1 sink                          |
| RYLR998 LoRa module        | 6   | UART, 923 MHz, SF10, BW 500 kHz            |
| 18650 Li-ion + holder      | 1   | 3500 mAh × 3.7 V (battery test reference)  |
| Laptop (Linux/Mac/Windows) | 1   | Runs Mosquitto + Logger + Dashboard        |

---

##  Quick Start

### 1. Laptop dependencies
```bash
# MQTT broker
sudo apt install mosquitto mosquitto-clients     # Linux
brew install mosquitto && brew services start mosquitto   # macOS

# Python tools
pip install paho-mqtt matplotlib numpy
```

### 2. Configure firmware
Edit the EXP block at the top of `N234_node_student_model_integrated.py`:
```python
NODE_ID        = 4                  # 3, 4, 5, 6, or 7
SINK_IP        = "192.168.0.20"
WIFI_SSID      = "your_wifi"
WIFI_PASS      = "your_password"

EXP_MODE       = "VOI_DRIVEN"       # or ALWAYS_LORA / ALWAYS_WIFI / PERIODIC
EXP_DURATION_S = 1800               # 30-min auto-stop
EXP_LABEL      = "run1_voi"
```

Edit `N1sink_student_model_compatible.py`:
```python
MQTT_BROKER = "192.168.0.11"        # Your laptop's LAN IP
```

### 3. Flash and run
1. Use Thonny IDE to save firmware as `main.py` on each Pico 2W
2. Start the logger on the laptop:
   ```bash
   python mqtt_csv_logger.py --broker 127.0.0.1 --out ./experiments
   ```
3. Power on all six nodes; experiments auto-stop after 30 minutes

### 4. Reproduce all four strategies
Repeat steps 2–3 with `EXP_MODE` and `EXP_LABEL` set to each of: `VOI_DRIVEN/run1_voi`, `ALWAYS_LORA/run2_lora`, `ALWAYS_WIFI/run3_wifi`, `PERIODIC/run4_periodic`. Keep the logger running through all four.

### 5. Generate paper artifacts
```bash
python compute_energy.py --batch ./experiments    # Table I
python Plot.py                                     # Fig 3 + Fig 5
```

---

##  Energy Model

Total energy is estimated as:
```
E_total (mJ) = Σ E_tx + P_idle × duration_s
```
where `P_idle = 12 mW` (Pi Pico 2W LoRa-listen baseline) and `E_tx` per packet ∈ {A:600, B:100, C:400, D:200} mJ.

This is **model-based extrapolation**, not INA219 current measurement. Constants are documented in `compute_energy.py` and aligned with `LINK_CLASS` in the firmware.

---

##  Known Limitations

- Energy is model-based, not directly measured with INA219 instrumentation.
- CSV records only successful packets; transmission failures appear in node consoles but are not aggregated.
- N5 in the testbed has degraded CYW43 + LoRa hardware; this serves as a real-world demonstration of multi-hop resilience.
- N7's RYLR998 module developed transmit-side faults during testing; the firmware's Wi-Fi fallback maintained delivery.
- Experiment durations vary (30–74 min) due to manual restart timing; per-packet normalization is provided.

---

##  Citation

```bibtex
@inproceedings{hung2026voi,
  title     = {VoI-Driven Transport Selection for Heterogeneous LoRa/Wi-Fi IoT
               with Unified MQTT Abstraction},
  author    = {Hung, Jun-Sen},
  booktitle = {Proc. IEEE Global Conference on Consumer Electronics (GCCE)},
  year      = {2026}
}
```

---

##  License

MIT License. Water-level data derived from public records of the Nanshenjiao Bridge monitoring station (Taiwan).

---

Department of Computer Science and Information Engineering
Tamkang University, New Taipei, Taiwan
✉️ 412411240@o365.tku.edu.tw
