video link:- https://www.youtube.com/watch?v=iwbNeCkA8oM
# 🩺 ANEBILIN — Non-Invasive Biomarker & Multimodal ECG Screening System

[![React 19](https://img.shields.io/badge/Frontend-React%2019-61DAFB?logo=react)](dashboard)
[![Google Gemini / Gemma](https://img.shields.io/badge/Vision%20AI-Gemma%20%2F%20Gemini%203.8-4285F4?logo=google)](dashboard/src/utils/gemmaClient.ts)
[![ElevenLabs TTS](https://img.shields.io/badge/Voice%20AI-ElevenLabs%20TTS-black)](dashboard/src/utils/elevenLabsClient.ts)

**ANEBILIN** is an end-to-end, point-of-care medical screening platform engineered to democratize preventative diagnostic testing in rural and underserved healthcare environments. It pairs custom **ESP32-S3 on-device hardware** (multispectral photometry and single-lead ECG) with a high-performance **React 19 clinician web application**, **FastAPI multi-disease machine learning engine**, and **multimodal visual AI** (Google Gemini 3.8 / Gemma) for automated cardiac waveform analysis.

---

## 🔬 Hardware Architecture & Implementation

The hardware subsystem is a self-contained, battery-powered clinical screening device designed for immediate point-of-care vital acquisition. It executes low-latency signal filtering, calibration algorithms, and optical feature extraction directly at the edge on an **ESP32-S3** microcontroller before streaming structured telemetry via Bluetooth Low Energy (BLE).

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                        ESP32-S3 SCREENING DEVICE                       │
 │                                                                        │
 │  ┌───────────────────────┐         ┌────────────────────────────────┐  │
 │  │   AMS AS7343 Sensor   │         │    AD8232 ECG Analog Front     │  │
 │  │  (14-ch Multispectral)│         │     (Lead I Single-Lead)       │  │
 │  └───────────┬───────────┘         └───────────────┬────────────────┘  │
 │              │ I2C (400 kHz)                       │ ADC1 (GPIO8)      │
 │              ▼                                     ▼                   │
 │  ┌──────────────────────────────────────────────────────────────────┐  │
 │  │                         CORE 1 ENGINE                            │  │
 │  │   • 1st-Order IIR AC/DC Splitter      • 1000 Hz ADC Sampler      │  │
 │  │   • Peak Detection & HRV Engine       • Circular PSRAM Buffer    │  │
 │  │   • Multi-Wavelength OLS (Hb/Bili)    • SQI Baseline Tracking    │  │
 │  └─────────────────────────────┬────────────────────────────────────┘  │
 │                                │ Internal Event Queue                  │
 │                                ▼                                       │
 │  ┌──────────────────────────────────────────────────────────────────┐  │
 │  │                         CORE 0 HOST                              │  │
 │  │   • SSD1306 OLED (128x64)             • Capacitive Wake Touch    │  │
 │  │   • 5 kHz NIR LED PWM Driver          • NimBLE GATT Server (FFF0)│  │
 │  └─────────────────────────────┬────────────────────────────────────┘  │
 └────────────────────────────────┼───────────────────────────────────────┘
                                  │ Bluetooth Low Energy (32-byte Float Stream)
                                  ▼
                     CLINICIAN WEB DASHBOARD (React 19)
```

### 1. Core Hardware Components
* **ESP32-S3 Microcontroller (Xtensa Dual-Core @ 240 MHz)**:
  - Leverages dual-core FreeRTOS task isolation: Core 1 is dedicated entirely to real-time ADC conversion and photometric integration loops, while Core 0 handles NimBLE host operations, OLED display updates, and USB CLI.
  - Native USB CDC support for diagnostics, fast NVS calibration storage, and hardware power management with wake-from-deep-sleep support.
* **AMS AS7343 14-Channel Multispectral Optical Sensor**:
  - Covers 12 visible spectral bands (380 nm – 750 nm), NIR (855 nm), and Clear channel.
  - Custom SMUX configuration routes 6 parallel photodiode channels simultaneously per integration cycle for rapid one-shot acquisition.
  - High dynamic range with on-chip 16-bit ADCs, auto-zero drift compensation, programmable gain (`0.5x` to `2048x`), and integration time (`100 ms` window for optimal tissue SNR).
* **AD8232 Single-Lead ECG Analog Front-End**:
  - Integrated instrumentation amplifier with two-pole high-pass filter to reject motion artifacts and electrode half-cell potential.
  - Right-Leg Drive (RLD) amplifier for common-mode rejection.
  - Samples at up to **1000 Hz** with hardware shutdown (`SDN`) control when idle to preserve battery.
* **Dual-Source Optical Illumination Subsystem**:
  - Onboard programmable AS7343 LED (regulated up to 129 mA DAC current steps) for surface reflectance probing.
  - External **855 nm High-Power NIR LED** driven via 5 kHz hardware PWM (`LEDC`) for transillumination through fingertip tissue.
* **SSD1306 0.96" I2C OLED Display**:
  - Independent secondary I2C bus (`Wire1` @ 400 kHz) rendering live photoplethysmogram waveforms, computed vitals (SpO2, HR, Hb, Bilirubin), BLE connection status, and diagnostic flags.
* **Capacitive Touch Pad & Power Management**:
  - GPIO 1 capacitive sensing supporting single-tap measurement triggers and hold-to-sleep / wake-from-sleep power states.

### 2. Hardware Pinout & Wiring Specification

| Component | Signal | ESP32-S3 Pin | Mode / Configuration |
|---|---|---|---|
| **AMS AS7343** | `SDA` | `GPIO 41` | I2C Primary Bus (`Wire`), 400 kHz |
| **AMS AS7343** | `SCL` | `GPIO 40` | I2C Primary Bus (`Wire`), 400 kHz |
| **AMS AS7343** | `INT` | `GPIO 42` | Digital Input (Data Ready / SMUX Done) |
| **AD8232 ECG** | `OUTPUT` | `GPIO 8` | Analog In (`ADC1_CH7`), 12-bit (0–4095) |
| **AD8232 ECG** | `SDN` | `GPIO 17` | Digital Output (Active-Low Shutdown) |
| **NIR LED A** | `GATE` | `GPIO 6` | Hardware PWM (`LEDC`), 5 kHz, 8-bit |
| **NIR LED B** | `GATE` | `GPIO 48` | Hardware PWM (`LEDC`), 5 kHz, 8-bit |
| **SSD1306 OLED** | `SDA` | `GPIO 11` | I2C Secondary Bus (`Wire1`), 400 kHz |
| **SSD1306 OLED** | `SCL` | `GPIO 12` | I2C Secondary Bus (`Wire1`), 400 kHz |
| **Touch Button** | `TOUCH` | `GPIO 1` | Capacitive Touch Channel 1 |

### 3. On-Device Edge Signal Processing & Math Models
1. **1st-Order IIR DC/AC Signal Filter**: Separates raw spectral counts into a slow DC baseline (tissue/bone absorption) and high-frequency pulsatile AC component (arterial blood volume expansion).
2. **SpO2 & Heart Rate Engine**: Uses a calibrated optical ratio-of-ratios ($R = \frac{AC_{red}/DC_{red}}{AC_{nir}/DC_{nir}}$) coupled with adaptive temporal peak detection to compute arterial blood oxygenation and pulse interval HRV.
3. **Multi-Wavelength Hemoglobin (Hb) Calibration**: Employs multivariate linear regression across 4 visible and NIR bands sensitive to oxygenated/deoxygenated hemoglobin absorption peaks.
4. **Bilirubin Estimation**: Utilizes differential spectral absorbance between 450 nm (bilirubin absorption peak) and 500 nm baseline reference to non-invasively detect hyperbilirubinemia risk.
5. **High-Speed ECG Buffer**: Continuously samples AD8232 analog voltage at 1000 Hz into an internal ring buffer, enabling block retrieval of 30-to-60 second diagnostic strips for cloud and visual analysis.

---

## 💻 Feature Section: How Software Utilizes the Hardware Telemetry

The software ecosystem transforms raw on-device sensor telemetry into actionable, clinical-grade diagnostic intelligence. Every software feature directly ingests, analyzes, and contextualizes the data produced by the hardware.

```
                           ESP32-S3 HARDWARE TELEMETRY
                   [32-Byte Packed Float Stream + 1 kHz ECG]
                                       │
                                       ▼ (Web Bluetooth API)
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               CLINICIAN WEB SOFTWARE                                  │
│                                                                                        │
│   ┌─────────────────────┐    ┌─────────────────────┐    ┌──────────────────────────┐   │
│   │  60 FPS Telemetry   │    │  Multimodal Vision  │    │  4-Disease ML Prediction │   │
│   │      Visualizer     │    │   (Gemini 3.8 AI)   │    │      (FastAPI Engine)    │   │
│   │ • Live PPG & Lead I │    │ • Wave Morphology   │    │ • Anaemia, Heart Disease,│   │
│   │ • Dynamic Baseline  │    │ • P-QRS-T Detection │    │   Cholesterol, Jaundice  │   │
│   │ • Real-time SQI     │    │ • Ectopic Arrhythmia│    │ • 48 Clinical Features   │   │
│   └──────────┬──────────┘    └──────────┬──────────┘    └────────────┬─────────────┘   │
│              │                          │                            │                 │
│              ▼                          ▼                            ▼                 │
│   ┌────────────────────────────────────────────────────────────────────────────────┐   │
│   │                     SYNTHESIS, TRIAGE & CLINICAL READOUT                       │   │
│   │  • ElevenLabs Neural Text-to-Speech: Voice-directed diagnosis & urgent alerts  │   │
│   │  • OCR Camera Capture: Paper lab sheet & monitor vitals digitisation           │   │
│   │  • Offline-First PWA: IndexedDB persistent records with Supabase cloud sync    │   │
│   │  • FHIR / PDF Export: Standardized medical documentation for hospital referral  │   │
│   └────────────────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1. ⚡ Browser-Native Web Bluetooth Direct Pipeline
- **Zero-Driver Handshake**: The React application interfaces directly with the ESP32-S3 via the browser's native Web Bluetooth API without requiring proprietary drivers or bridges.
- **Binary Telemetry Unpacking**: Listens on custom GATT service `0xFFF0` (Characteristic `0xFFF3`), unpacking 32-byte binary Little-Endian float arrays (`SpO2`, `HR`, `Hb`, `Bilirubin`, `ECG sample`, `SQI`, `Timestamp`) at 40 Hz with sub-10ms latency.
- **Automated Reconnect & Session Preservation**: Seamlessly resumes active patient monitoring sessions during transient wireless dropouts.

### 2. 🫀 Multimodal Visual ECG Waveform Analysis (Google Gemini 3.8 Flash / Gemma)
- **Software Waveform Rendering to Vision AI**: The software buffers 1000 Hz Lead I ECG signals, removes DC baseline wandering, normalizes amplitude, and renders crisp diagnostic strip snapshots onto an offscreen canvas.
- **Deep Visual Morphological Inspection**: These visual waveform images are transmitted to Google Gemini 3.8 Flash to evaluate the morphology that pure scalar numbers cannot convey:
  - **Atrial P-Wave**: Evaluates presence, amplitude, and PR interval to check for Sinoatrial conduction defects.
  - **Ventricular QRS Complex**: Evaluates QRS sharpness, duration, and R-peak axis deviation to flag bundle branch blocks.
  - **ST Segment & T-Wave**: Inspects elevation, depression, and T-wave inversion for ischemia or electrolyte abnormalities.
  - **Isoelectric Baseline Stability**: Automatically filters and classifies signal quality versus patient motion artifact.
- **Pictorial Safety Cards**: Displays color-coded diagnostic component breakdowns with clinical rationale and actionable recommendations.

### 3. 🧪 Multi-Disease Risk Prediction Engine (FastAPI + Machine Learning)
- **Ingestion of Hardware Biomarkers**: The Python FastAPI inference backend ingests live biomarkers recorded by the hardware (SpO2, Heart Rate, Hemoglobin, Total Bilirubin) alongside patient demographics, reported symptoms, and lifestyle indicators across a **48-feature clinical space**.
- **Specialized Multi-Model Classification**: Runs four dedicated machine learning pipelines trained on clinical epidemiological datasets:
  - **Anaemia Model (`XGBClassifier`)**: Correlates non-invasive optical Hemoglobin levels with fatigue, pallor symptoms, and dietary metrics.
  - **Heart Disease Model (`LogisticRegression`)**: Evaluates resting heart rate, ECG rhythm findings, chest discomfort, and age.
  - **High Cholesterol Model (`LogisticRegression`)**: Synthesizes biometric ratios, BMI, blood pressure, and vascular risk factors.
  - **Jaundice Model (`LogisticRegression`)**: Analyzes optical Bilirubin reflectance along with liver risk indicators, yellowing symptoms, and abdominal markers.
- **Continuous Probabilistic Scoring**: Outputs continuous probability values ($0.0$ to $1.0$) mapped to intuitive risk categories (Low, Moderate, High, Very High) to prevent binary misclassification in borderline screening cases.

### 4. 🗣️ ElevenLabs Neural Audio & Conversational Assistant
- **Voice-Directed Clinical Triage**: Converts software-generated risk calculations and ECG visual analysis reports into lifelike synthesized audio readouts using ElevenLabs neural voice models.
- **Resilient Multi-Tier Fallback**: Automatically cascades to browser-native Web Speech Synthesis if cloud API quotas or offline conditions occur.
- **Multilingual Voice Assistant**: Enables community health workers to interact with patient records, query diagnosis guidelines, and receive spoken audio summaries in English, Hindi, and Tamil.

### 5. 📷 Camera-Based OCR Vitals Ingestion
- **Bridging Analog & Digital**: Frontline clinicians can photograph physical laboratory slips, paper printouts, or third-party patient monitor screens using their device camera.
- **Automated Field Extraction**: Vision AI parses optical characters, extracts blood pressure, glucose, and lab metrics, and auto-populates the patient intake form to augment hardware readings.

### 6. 🌐 Offline-First PWA & Synced Patient Storage
- **Offline Field Screening**: Built as a Progressive Web App (PWA) with client-side caching (IndexedDB). Health workers can screen patients in remote rural areas without internet access.
- **Bidirectional Cloud Sync**: When an internet connection is re-established, the software seamlessly syncs queued local records, ECG traces, and ML evaluations with Supabase.
- **FHIR & PDF Generation**: Generates compliant clinical diagnostic summaries ready for hospital handoff, referral dispatch, or email transmission.

### 7. ☁️ DigitalOcean & Docker Compose Ready
- **1-Click Cloud Deployment**: Configured with [`.do/app.yaml`](.do/app.yaml) for automated zero-downtime deployment on DigitalOcean App Platform with free SSL (mandatory for Web Bluetooth).
- **Turnkey Docker Compose**: Pre-packaged [`docker-compose.yml`](docker-compose.yml) coordinating production Nginx (React 19 SPA) and FastAPI ML server containers.

---

## 📁 Repository Structure

```
ANEBILIN/
├── .do/app.yaml                  # DigitalOcean App Platform deployment specification
├── docker-compose.yml            # Production Docker orchestration (Frontend + Backend)
├── Dockerfile                    # Multi-stage production Nginx container build
├── DEPLOY_DIGITALOCEAN.md        # Comprehensive DigitalOcean deployment manual
├── platformio.ini                # PlatformIO build definition for ESP32-S3
├── src/
│   └── main.cpp                  # Firmware entry: FreeRTOS dual-core tasks, OLED, BLE, CLI
├── lib/
│   ├── AS7343/                   # ams AS7343 spectral sensor low-level driver & SMUX logic
│   ├── ECG/                      # AD8232 single-lead ECG sampling & circular buffer
│   └── AnebilModel/              # IIR filter, AC/DC ratio engine, multi-regression, NimBLE
├── tools/                        # Python scripts for offline training and telemetry analysis
│   ├── analyze_ecg.py            # ECG waveform visualization and feature analysis
│   ├── ble_scan.py               # BLE GATT discovery and verification
│   └── train_offline.py          # On-device biomarker model calibration and training
└── dashboard/                    # React 19 Web Dashboard & FastAPI ML Server
    ├── src/                      # TypeScript frontend (pages, components, Web Bluetooth)
    │   ├── components/           # ECG visualizer, AI Assistant, OCR, visual status cards
    │   ├── contexts/             # BLE GATT context, patient data store, multilingual i18n
    │   ├── pages/                # Telemetry dashboard, live recording, ECG monitor, settings
    │   └── utils/                # Gemini 3.8 client, ElevenLabs TTS client, signal filters
    ├── ml_server/                # Python FastAPI inference backend
    │   ├── api_server.py         # REST API endpoints (/health, /api/v1/predict, /api/v1/ecg)
    │   ├── predict.py            # 4-disease ML pipeline runner
    │   ├── train_models.py       # Scikit-learn & XGBoost model training scripts
    │   └── *.joblib              # Serialized production model weights
    ├── Dockerfile                # Dashboard web service container
    └── nginx.conf                # Production reverse proxy and SPA router
```

---

## 🚀 Quick Start Guide

### 1. ESP32-S3 Firmware Compilation & Flash
Ensure [PlatformIO](https://platformio.org/) is installed:
```bash
# Clone the repository
git clone https://github.com/SayanGhanty09/CODEFESTTHON_2026_SAYAN_GHANTY.git
cd CODEFESTTHON_2026_SAYAN_GHANTY

# Build and upload firmware to ESP32-S3
pio run -t upload

# Open serial monitor (115200 baud)
pio device monitor
```

### 2. Launch Local Clinician Web Application & ML Backend

**Terminal 1 — Machine Learning Server:**
```bash
cd dashboard/ml_server
python -m venv .venv
source .venv/bin/activate       # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn api_server:app --host 0.0.0.0 --port 8000 --reload
```

**Terminal 2 — React 19 Frontend:**
```bash
cd dashboard
npm install
cp .env.example .env.local       # Configure your API keys (Supabase, Gemini, ElevenLabs)
npm run dev
```


## ⚖️ Research & Screening Disclaimer

> **IMPORTANT**: The ANEBILIN platform is an engineering and research screening prototype intended for preliminary risk stratification and point-of-care vital telemetry. It is not an FDA/CE cleared diagnostic medical device and should not replace standard laboratory blood work, 12-lead electrocardiography, or professional physician evaluation.
