# 🔬 Cerebral Blood Flow & Speckle Contrast Imaging Platform

**Small Animal Imaging Platform — Version 2.1**  
TEBI Lab · IIT Bombay  
Principal Investigator: Dr. Hari M. Varma  
Developer: Arghya Mondal

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Quick Start](#quick-start)
- [GUI Layout](#gui-layout)
- [Module Summary](#module-summary)
- [Data Outputs](#data-outputs)
- [Repository Structure](#repository-structure)
- [Documentation](#documentation)
- [License](#license)
- [Contact](#contact)

---

## Overview

This platform is a comprehensive Python-based GUI application for **Laser Speckle Contrast Imaging (LSCI)** of small animal (rodent) brain tissue. It integrates real-time camera acquisition, speckle contrast computation, galvo-mirror beam steering, XYZ stereotaxic stage control, electrical forepaw stimulation via NiDAQ, relay control, and laser current monitoring — all within a single Tkinter/CustomTkinter interface.

The system is designed for hemodynamic studies in neuroscience, specifically for imaging cerebral blood flow (CBF) responses to somatosensory stimulation.

### What is Laser Speckle Contrast Imaging?

When coherent (laser) light illuminates a biological tissue, back-scattered photons form a random interference pattern called a **speckle pattern**. In regions with moving scatterers (e.g., red blood cells), the speckle pattern fluctuates rapidly, reducing local contrast. The **Speckle Contrast (K)** is defined as:

```
K = σ / <I>
```

Where `σ` is the standard deviation of pixel intensities in a local region and `<I>` is the mean intensity. Lower K → faster flow → higher perfusion. This platform computes a **frame-stacked** version across N frames:

```
K² = (std(stack) / (mean(stack) + ε))²  ×  4095
```

This produces a 12-bit speckle contrast map displayed in false color (viridis colormap) in real time.

---

## Key Features

### 🎥 Camera & Imaging
- Real-time **live preview** from a scientific TDC (scTDC) camera at 12-bit depth (1920×1440 px)
- Real-time **Speckle Contrast Video** computed from rolling frame stacks at user-defined FPS
- ROI (Region of Interest) selection for cropped acquisition and analysis
- Dual display panels: raw live feed and false-color speckle contrast view
- Adjustable **contrast sliders** for both live feed and speckle C-axis (1/k²)
- On-image **pixel coordinate display** with mm-mapped readout
- Vertical **colorbar** for speckle contrast scale visualization

### 📏 Spatial Calibration
- Interactive **ruler calibration tool** — draw a line on the live image, enter real-world distance (mm) to set px/mm factor
- Mouse coordinate display with live px → mm conversion
- Calibration factor persists across sessions

### 🪞 Galvo Mirror Control & Calibration
- Two-axis galvo (Motor X, Motor Y) via **Modbus RTU** over a shared serial port
- Dial-based angle control (±40° range) with thread-safe Modbus locking
- **Four calibration modes:**
  1. **Manual** — enter all grid angles by hand
  2. **Dial-based (Auto)** — click image points while adjusting galvo dials
  3. **Hybrid** — mix manual pre-set angles with dial-based for remaining points
  4. **Auto Calibrate** — click-based calibration with automated regression grid generation
- Configurable grids: 2×2, 2×3, 3×2, 3×3
- **Linear regression** (scikit-learn) to map image pixel coordinates → galvo angles
- **Galvo Home** function returns beam to matrix[0,0] position
- Grid scan (CoordinateSelector) for automated point-to-point beam positioning

### 💾 Calibration Management
- Auto-save/load calibration on startup (JSON + pickle formats)
- Named **backup and restore** of calibration files
- Status display with grid dimensions and calibration type
- ROI parameter tracking within calibration metadata

### 📊 Intensity Profile Analysis
- **Dual-mode**: draw lines on either the live image or the speckle contrast view
- Extracts pixel intensity along the drawn line
- Displays a two-panel matplotlib plot (intensity profile + image with line)
- Export profile data: `.mat` (MATLAB), `.png` (plot), `.docx` (statistics report)

### 🔌 Hardware Integration (Auto-Detected COM Ports)
- **Unified Modbus client** for Relay, Galvo X/Y, and Z-axis on a single COM port
  - Thread-safe communication with `threading.Lock` and automatic reconnection
  - **Relay Control**: 4-channel peripheral device power management
  - **Galvo Motors**: Precision beam steering with regression calibration
  - **Z-axis**: Camera focus control
- **XYZ Stereotaxic Stage**: Auto-detected; supports multiple speed modes (Slow/Medium/Fast)
- **Laser Current Monitor**: Real-time display (0–1000 mA) via a separate serial thread

### ⚡ NiDAQ Forepaw Electrical Stimulator
- Integrated control panel for current-pulse stimulation
- Configurable parameters: number of sets, pulses per set, pulse duration (ms), inter-pulse interval (ms), inter-set interval (s)
- Real-time progress bar, time elapsed, and set counter
- Safety confirmation dialog before every stimulation run
- Synchronization hook with multi-capture acquisition

### 📁 Multi-Capture Acquisition
- Capture across multiple exposure times in a single automated run
- N-frame stacking per exposure (default 100 frames)
- **Data type classification**: Dark, Baseline, Stimulation (with dedicated subfolders)
- Optional stimulation sync: stimulator runs in parallel with capture
- Pause/abort controls during acquisition
- Comprehensive **timing analysis**: per-capture duration, galvo move time, save time, total elapsed

### 💽 Data Management & Export
- All captures **timestamped** and organized into typed subdirectories
- Output formats:
  - **16-bit PNG** (raw speckle contrast)
  - **8-bit PNG** (preview)
  - **MAT file** (MATLAB-compatible, via `scipy.io.savemat`)
  - **DOCX log** (metadata, capture parameters, timing stats via `python-docx`)
  - **Pickle** (calibration data)
  - **JSON** (calibration data, human-readable)

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     IntegratedGUI (Main Process)            │
│  ┌──────────────┐  ┌───────────────────┐  ┌──────────────┐  │
│  │  Live Feed   │  │  Speckle Contrast │  │  Control     │  │
│  │  (Tkinter    │  │  View (Tkinter    │  │  Panels      │  │
│  │   Canvas)    │  │   Canvas +        │  │  (Left       │  │
│  │              │  │   Colorbar)       │  │  Sidebar)    │  │
│  └──────────────┘  └───────────────────┘  └──────────────┘  │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Background Threads                        │ │
│  │  ┌───────────────┐  ┌──────────────┐  ┌─────────────┐  │ │
│  │  │ Frame Listener│  │ Speckle Video│  │ Laser Serial│  │ │
│  │  │ (data_queue)  │  │ (rolling K²) │  │ Monitor     │  │ │
│  │  └───────────────┘  └──────────────┘  └─────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  command_queue ──────────────────────────────────────────┐  │
└──────────────────────────────────────────────────────────┼──┘
                                                           │
                    ┌──────────────────────────────────────▼───┐
                    │     camera_process (Separate Process)    │
                    │  ┌──────────┐  ┌──────────┐  ┌────────┐  │
                    │  │INITIALIZE│  │ PREVIEW  │  │SPECKLE │  │
                    │  │  CAM     │  │ LOOP     │  │ FRAME  │  │
                    │  └──────────┘  └──────────┘  └────────┘  │
                    │       scTDClib / scTDC Camera            │
                    └──────────────────────────────────────────┘

Serial Hardware (Modbus RTU + Serial)
  └── COM port (auto-detected)
        ├── Relay Controller   (Slave ID: 2, registers 0-3)
        ├── Galvo Motor X/Y    (Slave ID: 2, registers 0-6, float32)
        └── Z-axis Camera      (Slave ID: 2)
  └── XYZ Stereotaxic Stage   (separate auto-detected port)
  └── Laser Current Monitor   (separate auto-detected port, ASCII serial)

NiDAQ
  └── Forepaw Stimulator      (PyDAQmx, digital pulse output)
```

---

## Hardware Requirements

| Component | Details |
|---|---|
| **Camera** | scTDC-compatible scientific camera (1920×1440, 12-bit) |
| **Galvo System** | 2-axis galvo mirror with Modbus RTU controller (±40° range) |
| **Relay Board** | 4-channel Modbus RTU relay controller |
| **Z-axis Stage** | Modbus-compatible motorized Z-axis for camera focus |
| **XYZ Stereotaxic Stage** | Serial-controlled 3-axis positioning stage |
| **Laser** | CW laser with serial-readable current feedback (0–1000 mA) |
| **NiDAQ Card** | National Instruments DAQ card (digital output for stimulation) |
| **PC** | Windows 10/11, 64-bit; ≥16 GB RAM recommended |

### COM Port Auto-Detection

The software automatically scans for COM ports and attempts to match devices by USB descriptor keywords:

| Device | Detection Keywords |
|---|---|
| Relay / Galvo / Z-axis | `CP2102`, `FT232R`, `USB Serial Device`, `USB-to-Serial` |
| XYZ Stage | `CH340`, `Prolific`, `USB Serial Port`, `SERIAL` |
| Laser Monitor | `Arduino`, `USB Serial`, `CP2102` |

If auto-detection fails, the software falls back to COM4 (Relay/Galvo) and assigns remaining ports in order.

---

## Software Requirements

Tested on: **Python 3.10, Windows 10/11 (64-bit)**

### Core Python Dependencies

```
tkinter          (stdlib)
customtkinter    >= 5.2
Pillow           >= 10.0
numpy            >= 1.24
opencv-python    >= 4.8
scipy            >= 1.11
scikit-learn     >= 1.3
matplotlib       >= 3.7
pymodbus         >= 3.5
pyserial         >= 3.5
python-docx      >= 1.0
pygame           >= 2.5
tkdial           >= 0.3
PyDAQmx          >= 1.4
scTDC            (proprietary — from camera vendor)
```

See [`requirements.txt`](requirements.txt) for the full pinned dependency list.

### Proprietary / Vendor Dependencies

| Library | Source |
|---|---|
| `scTDC` | Camera vendor SDK — place `scTDClib.dll` and `tdc_gpx3.ini` in the script directory |
| `PyDAQmx` | NI-DAQmx driver + Python wrapper — requires NI-DAQmx runtime installed |

---

## Quick Start

### 1. Install dependencies

```bash
git clone https://github.com/your-org/lsci-platform.git
cd lsci-platform
pip install -r requirements.txt
```

### 2. Place vendor files

Copy the following into the project root directory (same folder as `new_GUI_sc_55.py`):
- `scTDClib.dll` (camera SDK)
- `tdc_gpx3.ini` (camera initialization config)
- `logo.png` (optional — lab logo shown in About dialog)

### 3. Connect hardware

- Plug in the Modbus USB-serial adapter (Relay/Galvo/Z-axis)
- Plug in the XYZ stage USB cable
- Plug in the laser USB serial cable
- Ensure NI-DAQmx runtime is installed if using the stimulator

### 4. Launch

```bash
python new_GUI_sc_55.py
```

### 5. First-time startup sequence

1. The software auto-detects COM ports (check console output for confirmation)
2. Click **Initialize Camera** to connect to the scTDC camera
3. Click **Start Preview** to begin live imaging
4. Click **Enable Galvo** before any galvo operations
5. If no calibration file is found, run **Auto Calibrate** from the Galvo panel
6. Select a **Save Directory** before any capture

---

## GUI Layout

```
┌────────────────────────────────────────────────────────────────────┐
│  [Header Bar] — Platform title | Laser Current | Action Buttons    │
├──────────────────┬─────────────────────────────────────────────────┤
│  LEFT SIDEBAR    │  DISPLAY AREA                                   │
│  ─────────────── │  ┌──────────────────┐  ┌──────────────────────┐ │
│  Camera Controls │  │  Live Preview    │  │  Speckle Contrast    │ │
│  Exposure / FPS  │  │  (raw 12-bit)    │  │  View (viridis)      │ │
│  ─────────────── │  │  + contrast      │  │  + C-axis slider     │ │
│  Galvo Controls  │  │    slider        │  │  + colorbar          │ │
│  Dial X / Dial Y │  └──────────────────┘  └──────────────────────┘ │
│  Enable / Home   │                                                 │
│  ─────────────── │                                                 │
│  Calibration     │                                                 │
│  (Scale + Galvo) │                                                 │
│  ─────────────── │                                                 │
│  ROI Controls    │                                                 │
│  ─────────────── │                                                 │
│  Multi-Capture   │                                                 │
│  ─────────────── │                                                 │
│  XYZ Stage       │                                                 │
│  Z-axis Focus    │                                                 │
│  ─────────────── │                                                 │
│  Laser Monitor   │                                                 │
├──────────────────┴─────────────────────────────────────────────────┤
│  [Status Bar] — Current operation/state messages                   │
└────────────────────────────────────────────────────────────────────┘
```

---

## Module Summary

| Section | Class / Function | Responsibility |
|---|---|---|
| **1. Constants** | `Commands` | Enum-style command codes for the camera process IPC |
| **2. Serial Ports** | `detect_com_ports()`, `auto_detect_ports()` | Dynamic USB-serial port detection |
| **3. Camera Process** | `camera_process()` | Isolated multiprocessing camera loop (init, preview, speckle, exit) |
| **4. Config Management** | `ConfigurationManager` | Save/load calibration as JSON and pickle |
| **4. Calib Manager** | `CalibrationManager` | GUI dialog for backup/restore of calibration files |
| **5. Galvo Calibration** | `CoordinateSelector` | Image→galvo coordinate mapping via regression matrix |
| **5. Regression** | `combined_function()` | Fit linear regression, generate galvo angle grid |
| **6. Scale Calibration** | `CalibrationTool` | Interactive ruler for px/mm calibration |
| **7. Manual Input** | `ManualAngleInput` | Grid-based manual galvo angle entry window |
| **8. Intensity Profile** | `IntensityProfileTool` | Line-draw intensity analysis on live or speckle image |
| **9. Stimulator** | `ForepawStimulatorApp` | NiDAQ pulse stimulation control panel |
| **10. Main GUI** | `IntegratedGUI.__init__()` | All state initialization, hardware connections, layout |
| **11. Camera Methods** | `initialize_camera()`, `start_preview()`, `stop_preview()` | Camera IPC wrappers |
| **11. Speckle Video** | `capture_speckle_video()` | Rolling frame stack K² computation loop |
| **11. Galvo Methods** | `send_galvo_motor1/2()`, `send_galvo_motor1/2_direct()` | Modbus register writes for angle control |
| **11. Multi-Capture** | (multi-capture thread) | Automated exposure sweep, data type filing, stimulation sync |
| **11. Data Save** | `_save_png_16bit()` | 16-bit PNG export with PIL/cv2 fallback |
| **12-15. Hardware** | Relay, XYZ, Z-axis, Laser methods | Modbus and serial communication handlers |
| **16. Entry Point** | `__main__` | `multiprocessing.set_start_method("spawn")`, root Tk loop |

---

## Data Outputs

For each acquisition session, the platform creates a timestamped directory structure:

```
<save_directory>/
└── YYYYMMDD_HHMMSS/
    ├── dark_data/
    │   ├── dark_YYYYMMDD_HHMMSS_<exp>us.png       (16-bit speckle contrast)
    │   ├── dark_YYYYMMDD_HHMMSS_<exp>us_8bit.png  (8-bit preview)
    │   ├── dark_YYYYMMDD_HHMMSS_<exp>us.mat        (MATLAB struct)
    │   └── dark_YYYYMMDD_HHMMSS_<exp>us.docx       (metadata log)
    ├── base_data/
    │   └── base_*.png / *.mat / *.docx
    └── stimulation_data/
        └── stim_*.png / *.mat / *.docx
```

### MAT File Contents

Each `.mat` file contains a MATLAB struct with fields:

| Field | Type | Description |
|---|---|---|
| `speckle_contrast` | uint16 array | Raw speckle contrast map |
| `exposure_us` | scalar | Exposure time in microseconds |
| `num_frames` | scalar | Number of stacked frames |
| `timestamp` | string | ISO 8601 capture time |
| `roi_active` | bool | Whether ROI was applied |
| `roi_coords` | 1×4 array | [x1, y1, x2, y2] |
| `calibration_factor` | scalar | px/mm calibration |
| `capture_duration_s` | scalar | Capture timing |
| `save_duration_s` | scalar | Save timing |

---

## Repository Structure

```
lsci-platform/
├── new_GUI_sc_55.py          # Main application (single-file)
├── requirements.txt          # Python dependencies
├── tdc_gpx3.ini              # Camera config (from vendor — not tracked in git)
├── scTDClib.dll              # Camera SDK DLL (from vendor — not tracked in git)
├── logo.png                  # Optional lab logo
├── galvo_calibration.pkl     # Auto-saved calibration (generated at runtime)
├── galvo_calibration_config.json  # Auto-saved calibration JSON
├── docs/
│   ├── architecture.md       # Detailed module and data flow diagrams
│   ├── installation.md       # Step-by-step setup guide
│   ├── hardware_setup.md     # Wiring, Modbus IDs, NiDAQ connections
│   ├── calibration_guide.md  # Scale and galvo calibration walkthroughs
│   ├── user_guide.md         # Full GUI operation manual
│   ├── data_formats.md       # Output file format specifications
│   └── api_reference.md      # Class and method documentation
├── CHANGELOG.md              # Version history
└── .gitignore
```

---

## Documentation

| Document | Description |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | System design, class diagram, IPC flow |
| [`docs/installation.md`](docs/installation.md) | Dependency installation, driver setup |
| [`docs/hardware_setup.md`](docs/hardware_setup.md) | Serial wiring, Modbus register map, NiDAQ pinout |
| [`docs/calibration_guide.md`](docs/calibration_guide.md) | All four galvo calibration modes + scale calibration |
| [`docs/user_guide.md`](docs/user_guide.md) | GUI walkthrough, multi-capture, stimulation sync |
| [`docs/data_formats.md`](docs/data_formats.md) | MAT, PNG, DOCX, PKL, JSON format specs |
| [`docs/api_reference.md`](docs/api_reference.md) | All classes, methods, parameters |

---

## License

**Academic Use Only**

This software was developed as part of a research project at IIT Bombay. It is intended for use within the TEBI Lab and collaborating academic institutions. Redistribution, commercial use, or modification for non-academic purposes requires explicit written permission from the Principal Investigator.

---

## Contact

| Role | Name | Contact |
|---|---|---|
| Principal Investigator | Dr. Hari M. Varma | TEBI Lab, IIT Bombay |
| GUI Developer | Arghya Mondal | 30006166@iit.ac.in |

**For technical issues:** Refer to the Troubleshooting section in [`docs/user_guide.md`](docs/user_guide.md) before raising an issue.

---

*Release: January 2026 · Version 2.1*
