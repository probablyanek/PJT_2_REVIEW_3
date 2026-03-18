
## SLIDE 1 — Title Slide

**Title:** ML-Based Asynchronous Sensor Fusion for Indoor Localization

**Subtitle:** Edge-Deployed Neural Fusion on ESP32-S3 ISAC Nodes

[Your name, guide, department, university, Review stage, date]

---

## SLIDE 2 — The Problem

**Heading:** Why Sensor Fusion for Indoor Spaces?

**Left column — The gap:**
- GPS fails indoors — warehouses, factories, hospitals have zero satellite coverage
- 6G ISAC nodes combine communication + sensing on one device
- No single sensor works reliably alone

**Right column — Two sensors, opposite weaknesses:**

| | Radar (24 GHz FMCW) | Wi-Fi FTM (802.11mc) |
|---|---|---|
| Precision | ±0.15 m near-field | ±1.5 m typical |
| Angle | Yes (±60° FoV) | No (range only) |
| Through walls | No — blocked by concrete, metal | Yes — 2.4 GHz penetrates |
| Update rate | 10–15 Hz | ~1–2 Hz |
| Identity | No — sees all objects | Yes — knows which tag |

**Bottom line:** Radar is precise but blind behind walls. Wi-Fi sees through walls but is noisy. Fusing them gives the best of both.

---

## SLIDE 3 — Objectives

**Heading:** Project Objectives

1. Design a COTS hardware node that performs radar + Wi-Fi sensing simultaneously
2. Develop an ML model that learns to dynamically trust the better sensor
3. Deploy the trained model directly on the ESP32-S3 microcontroller (edge inference — no cloud)
4. Validate that fused accuracy beats either sensor alone under occlusion

**Scope:** Static indoor localization (fixed-position targets with micro-motion)

**Coordinate system:** Polar (r, θ) — harmonizes radar's 2D output with Wi-Fi's range-only output

---

## SLIDE 4 — Hardware Architecture

**Heading:** System Hardware (All COTS Components)

**Sensing Node:**
- ESP32-S3 — dual-core Xtensa LX7 @ 240 MHz
- HLK-LD2450 — 24 GHz FMCW radar, UART @ 256k baud, 10–15 Hz
- Roles: FTM initiator + radar parser + ML inference engine

**Active Tag:**
- Secondary ESP32 — FTM responder + ESP-NOW heartbeat
- Trihedral corner reflector — boosts radar cross-section, zero compute cost

**Connections:**

```
[HLK-LD2450] --UART 256k--> [ESP32-S3] <--Wi-Fi FTM--> [ESP32 Tag]
   GPIO16/17                  Core 1          Core 0        SoftAP
```



---

## SLIDE 5 — RTOS Task Architecture

**Heading:** Dual-Core FreeRTOS Design

**The problem solved:** Wi-Fi FTM bursts disable lower-priority interrupts for microsecond-precise timing. If UART shares the same core, the 128-byte hardware FIFO overflows in ~5 ms at 256k baud → corrupted radar frames.

**The solution:** Strict core isolation.

```
┌──────────── Core 0 ────────────┐  ┌──────────── Core 1 ────────────┐
│                                │  │                                │
│  wifi_ftm_task (priority 5)    │  │  uart_radar_task (priority 5)  │
│  • Wi-Fi STA connection        │  │  • UART ISR registered here    │
│  • FTM initiation every 500ms  │  │  • 8192-byte RX buffer         │
│  • HT40 bandwidth (40 MHz)     │  │  • Parse LD2450 → (r, θ)      │
│  • Min-RTT multipath filter    │  │                                │
│  • ESP-NOW heartbeat monitor   │  │  ml_inference_task (priority 3)│
│                                │  │  • TFLite Micro Float32        │
│  ── never touches UART ──      │  │  • Forward-fill + windowing    │
│                                │  │  • ~350 μs per inference       │
└────────────────────────────────┘  └────────────────────────────────┘
                    │                          ▲
                    └──── FreeRTOS Queue ───────┘
                         (64 × sensor_event_t)
```

**Key:** UART ISR on Core 1 cannot be preempted by Wi-Fi on Core 0 → zero dropped radar frames.

---

## SLIDE 6 — Polar Coordinate Design

**Heading:** Why Polar, Not Cartesian?

**Problem with Cartesian (x, y):**
- Radar gives (x_mm, y_mm) → negative x values for left-side targets
- Wi-Fi gives range only → no native (x, y) without angle info
- Projecting 1D Wi-Fi range into 2D Cartesian requires the radar's angle → circular dependency

**Polar solution (r, θ):**
- Radar → (r, θ) directly via `r = √(x² + y²)`, `θ = atan2(x, y)`
- Wi-Fi → (r, —) naturally: FTM measures range, has no angle
- Both sensors share the r-axis → direct comparison without geometric projection

```
        θ = 0° (boresight)
           │
   -60°    │    +60°
      \    │    /
       \   │   /
        \  │  /
         \ │ /         Target at (r=3.2m, θ=+25°)
          \│/              ●
     ──────●────── Sensor origin
```

**Model output:** Predicted (r̂, θ̂) in meters and degrees

---

## SLIDE 7 — Feature Engineering

**Heading:** Forward-Fill, Not Zero-Fill

**The problem with zero-filling missing radar data:**

When radar drops out (occlusion, FoV escape), setting r=0, θ=0 tells the neural network: *"the target is at the sensor origin."* The network learns to pull predictions toward (0, 0) during every dropout. This is geometrically wrong.

**Forward-fill solution:**

| Event | radar_r | radar_θ | radar_fresh | wifi_r | wifi_fresh |
|---|---|---|---|---|---|
| Radar valid | 2.08 | 30.5 | **1.0** | 2.34 (fwd) | 0.0 |
| Radar valid | 2.09 | 30.2 | **1.0** | 2.34 (fwd) | 0.0 |
| Radar dropout | 2.09 (fwd) | 30.2 (fwd) | **0.0** | 2.34 (fwd) | 0.0 |
| Radar dropout | 2.09 (fwd) | 30.2 (fwd) | **0.0** | 2.34 (fwd) | 0.0 |
| Wi-Fi arrives | 2.09 (fwd) | 30.2 (fwd) | 0.0 | **2.19** | **1.0** |
| Radar returns | **2.10** | **30.4** | **1.0** | 2.19 (fwd) | 0.0 |

- `fwd` = held from last valid reading
- `fresh` flag = tells the model whether to trust this value or treat it as stale
- Sliding window of 5 events → model sees short-term consensus

**5 features × 5 timesteps = 25 input values → MLP**

---

## SLIDE 8 — ML Model Architecture

**Heading:** Float32 MLP — Why Not INT8?

**Architecture:**

```
Input (25) → Dense(48, ReLU) → Dense(24, ReLU) → Dense(2, Linear)
                                                    ↓
                                              [r_norm, θ_norm]
```

| Property | Value |
|---|---|
| Parameters | 2,474 |
| Model size | 11.6 KB (.tflite) |
| Precision | Float32 (no quantization) |
| Output activation | Linear (not tanh) |
| Inference time | ~350 μs on ESP32-S3 |

**Why Float32 instead of INT8:**

INT8 quantization compresses all values into 256 discrete bins.
For a 6-meter room: `6.0m ÷ 255 = 2.35 cm` resolution floor — but error compounds through every hidden layer.
Tanh lookup tables in INT8 deviate 30%+ from true values.

At 2,474 parameters = 11.6 KB, the model fits trivially in 512 KB SRAM. Quantization saves nothing meaningful but destroys regression accuracy.

---

## SLIDE 9 — Training Pipeline

**Heading:** Data → Features → Model → Deployment

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│ Manual Data  │    │ Feature Eng  │    │  Training   │    │  Conversion  │
│ Collection   │───→│ (Python)     │───→│  (Keras)    │───→│  (TFLite)    │
└─────────────┘    └──────────────┘    └─────────────┘    └──────────────┘
                                                                │
 70 static poses     Forward-fill       EarlyStopping          Pure Float32
 30s each            Normalize [0,1]    ReduceLR               No quantization
 Micro-motion        Window = 5         MSE loss               xxd → C header
 Polar ground truth  Split by segment   200 epochs max         alignas(16)
                                                                │
                                                                ▼
                                                    ┌──────────────────┐
                                                    │  ESP32-S3        │
                                                    │  TFLite Micro    │
                                                    │  Float32 inference│
                                                    └──────────────────┘
```

**Dataset:** ~35,000 events across 70 segments (50 clear + 20 occluded)

**Split:** 70% train / 15% val / 15% test — split by segment, not by row

---

## SLIDE 10 — Results: Model Accuracy

**Heading:** Fusion Accuracy — Test Set

| Metric | Train | Validation | Test |
|---|---|---|---|
| MSE (normalized) | 0.000588 | 0.000621 | 0.001958 |
| MAE range (m) | 0.107 | 0.078 | **0.118** |
| MAE angle (°) | 2.09 | 3.02 | **4.88** |
| Euclidean mean (m) | 0.168 | 0.144 | **0.256** |
| Euclidean 95th % (m) | 0.385 | 0.276 | **0.555** |

**[Image: Training loss curve showing convergence at epoch 23, EarlyStopping at epoch 38]**

---

## SLIDE 11 — Results: Error by Distance

**Heading:** Accuracy Across Distance Bands

| Distance Band | Samples | MAE range (m) | MAE angle (°) | Euclidean (m) |
|---|---|---|---|---|
| 2–3 m | 566 | 0.116 | 6.66 | 0.277 |
| 3–5 m | 794 | 0.119 | 3.62 | 0.242 |

**[Image: Scatter plot — predicted r vs ground truth r, showing tight diagonal clustering]**

**[Image: Scatter plot — predicted θ vs ground truth θ]**

---

## SLIDE 12 — Results: Sensor Comparison

**Heading:** Fused Output vs Raw Sensors

| Source | MAE range (m) | Angle info? | Works through walls? |
|---|---|---|---|
| Raw Radar | ±0.15 – 0.65 | Yes | No |
| Raw Wi-Fi FTM | ±1.1 – 4.8 | No | Yes |
| **ML Fusion** | **0.118** | **Yes** | **Yes** |

**Key result:** Fusion maintains sub-20 cm range accuracy while inheriting Wi-Fi's wall-penetration capability.

**During radar occlusion:**

| Condition | Samples | Euclidean MAE (m) |
|---|---|---|
| Radar available | 1,163 | 0.262 |
| Radar occluded | 197 | 0.226 |

Model seamlessly shifts trust to Wi-Fi when radar drops out — no manual switching logic needed.

---

## SLIDE 13 — Results: Edge Deployment

**Heading:** Inference Performance on ESP32-S3

| Metric | Value |
|---|---|
| Model file size | 11.6 KB |
| Tensor arena | 32 KB (internal SRAM) |
| Inference latency | ~350 μs per prediction |
| Real-time budget | 50,000 μs (50 ms) |
| Headroom | **99.3% budget unused** |
| Conversion accuracy loss | 1.79 × 10⁻⁷ (effectively zero) |
| Framework | TFLite Micro, Float32, no quantization |

**[Image: Screenshot of ESP32-S3 serial monitor showing FUSED output lines with inference times]**

---

## SLIDE 14 — Results: Multipath Mitigation

**Heading:** HT40 + Min-RTT Filtering

**Problem:** 2.4 GHz Wi-Fi at 20 MHz bandwidth → theoretical resolution floor of 7.5 m. Indoor multipath adds 1–3 m bias.

**Solution implemented:**
1. Force 40 MHz (HT40) bandwidth → resolution floor drops to 3.75 m
2. 16 frames per FTM burst → collect 16 RTT measurements
3. Take the minimum RTT → closest to the true line-of-sight path

```
Raw FTM ranges from one burst:  3.1, 2.9, 4.2, 2.8, 3.5, 2.7, 3.8, 2.9,
                                 3.0, 2.8, 4.5, 2.7, 3.2, 2.9, 3.6, 2.8
                                                        ↓
Min-RTT filter:                                        2.7 m
True distance:                                         2.5 m
Naive mean:                                            3.2 m ← 0.7m worse
```

---

## SLIDE 15 — Challenges & Solutions

**Heading:** Key Technical Challenges

| # | Challenge | Root Cause | Solution |
|---|---|---|---|
| 1 | Radar drops stationary targets | LD2450 MTI filter requires Doppler shift | Micro-motion protocol — subjects sway gently, feet stay planted |
| 2 | UART data corruption during FTM | Wi-Fi ISR preempts UART on same core, 128-byte HW FIFO overflows in 5 ms | UART driver initialized on Core 1, physically isolated from Wi-Fi on Core 0 |
| 3 | INT8 quantization destroys accuracy | 256 bins over 6 m = 2.35 cm floor, compounds through layers; tanh LUT deviates 30% | Stay Float32 — model is only 11.6 KB, quantization saves nothing |
| 4 | Zero-fill pulls predictions to origin | Missing radar r=0 means "target at sensor" not "unknown" | Forward-fill last valid reading + binary freshness flag |
| 5 | Wi-Fi multipath inflates range | 2.4 GHz reflects off walls, 20 MHz can't resolve reflections | HT40 (40 MHz) + 16-frame min-RTT filter |

---

## SLIDE 16 — Remaining Work

**Heading:** Remaining ~15%

| Task | Status | Details |
|---|---|---|
| Extended range data collection | Pending | Current dataset covers 2–5 m only. Need 0.5–2 m and 5–6 m segments to prevent blind spots. |
| Hardware packaging | Pending | 3D-print PLA enclosures, vertical stacking of radar + ESP32 to reduce EM coupling |
| Multi-tag testing | Pending | Validate with 2–3 simultaneous tags — ESP-NOW collision handling |
| Field validation | Pending | Walk-tests in actual warehouse/factory environment |

**Known limitation:** Wi-Fi FTM provides no angle information. During full radar blackout, range accuracy is maintained but angular estimate relies entirely on stale forward-fill. A second angle-capable sensor (UWB, camera) would resolve this.

---

## SLIDE 17 — References

1. Pegoraro et al. (2023) — SPARROW: mmWave radar occlusion characterization
2. Wang et al. (2022) — ISAC for 6G: principles and applications
3. Chen, Arambel & Mehra (2002) — Covariance Intersection for distributed fusion
4. Aggarwal et al. (2022) — IEEE 802.11mc FTM ranging evaluation
5. Zubow et al. (2023) — FTM-based indoor localization
6. Hoang et al. (2025) — Wi-Fi FTM survey
7. Gaiba et al. (2024) — ESP32 Wi-Fi sensing platform
8. Espressif (2024) — ESP-IDF v5.2 FTM programming guide
9. David et al. (2022) — TFLite Micro deployment on embedded MCUs
10. HiLink (2024) — HLK-LD2450 datasheet and UART protocol specification