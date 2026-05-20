# docs/installation.md — Installation Guide

**LSCI Platform v2.1 · TEBI Lab · IIT Bombay**

This guide walks through the complete installation of the platform from a bare Windows machine to a fully working GUI. Follow every section in order. Do not skip sections — each one is a prerequisite for the next.

---

## Table of Contents

1. [System Prerequisites](#1-system-prerequisites)
2. [Install Python 3.10](#2-install-python-310)
3. [Clone the Repository](#3-clone-the-repository)
4. [Create a Virtual Environment](#4-create-a-virtual-environment)
5. [Install pip Packages](#5-install-pip-packages)
6. [Install the Camera SDK (scTDC)](#6-install-the-camera-sdk-sctdc)
7. [Install NI-DAQmx Runtime and PyDAQmx](#7-install-ni-daqmx-runtime-and-pydaqmx)
8. [Install USB-Serial Drivers](#8-install-usb-serial-drivers)
9. [Assemble the Project Directory](#9-assemble-the-project-directory)
10. [Verify the Complete Install](#10-verify-the-complete-install)
11. [First Launch](#11-first-launch)
12. [Companion Scripts](#12-companion-scripts)
13. [Troubleshooting First-Run Failures](#13-troubleshooting-first-run-failures)
14. [Uninstall / Clean Rebuild](#14-uninstall--clean-rebuild)

---

## 1. System Prerequisites

Before installing any software, confirm your machine meets these requirements:

| Requirement | Minimum | Recommended |
|---|---|---|
| Operating System | Windows 10 64-bit (Build 1903+) | Windows 11 64-bit |
| RAM | 8 GB | 16 GB or more |
| Storage | 20 GB free | SSD with 50+ GB free |
| Screen Resolution | 1280 x 960 | 1920 x 1080 or higher |
| USB Ports | 3 free USB-A ports | USB 3.0 preferred |
| Internet | Required for initial install | Not needed at runtime |

Screen resolution note: The GUI window is sized to 95% of screen dimensions at startup. On a 1280x960 screen the window will be 1216x912 px, which is tight but usable. On anything smaller, panels may overlap. A Full HD (1920x1080) or wider screen is strongly recommended for the dual-panel live + speckle view.

Architecture: 64-bit Windows is mandatory. The scTDClib.dll camera SDK and PyDAQmx NI wrapper are both 64-bit only. A 32-bit Python interpreter will fail to load them.

---

## 2. Install Python 3.10

### 2.1 Download

Go to: https://www.python.org/downloads/release/python-31011/

Scroll to Files at the bottom and download:
Windows installer (64-bit) — the file named python-3.10.x-amd64.exe

Why Python 3.10 specifically?
The scTDC camera SDK Python wheel from the vendor is compiled against the CPython 3.10 ABI (cp310). Python 3.11+ may also work but has not been validated against the current camera SDK version. Python 3.9 and below are not compatible with customtkinter >= 5.2.

### 2.2 Install

Run the installer. On the first screen:

Check both boxes at the bottom:
- [x] Install launcher for all users (recommended)
- [x] Add Python 3.10 to PATH

Click Customize installation.
On Optional Features — ensure all are checked.
On Advanced Options:
- [x] Install for all users
- [x] Add Python to environment variables
- [x] Precompile standard library
- Installation path: C:\Python310 (avoid paths with spaces)

Click Install.

### 2.3 Verify

Open a new Command Prompt (Win+R -> cmd) and run:

```bat
python --version
```

Expected: Python 3.10.x

Also verify pip:

```bat
pip --version
```

Expected: pip 23.x.x from C:\Python310\Lib\site-packages\pip (python 3.10)

If python is not found, add C:\Python310 and C:\Python310\Scripts to your system PATH via:
Control Panel -> System -> Advanced System Settings -> Environment Variables -> Path -> Edit.

---

## 3. Clone the Repository

If you have Git installed:

```bat
git clone https://github.com/your-org/lsci-platform.git
cd lsci-platform
```

If you do not have Git:
- Download the repository as a ZIP from the GitHub page
- Extract to a folder such as C:\lsci-platform
- Open a Command Prompt and cd into that folder

Path recommendation: Keep the project in a short path with no spaces and no Unicode characters — e.g., C:\lsci-platform or D:\lab\lsci. Long paths or paths with spaces have been known to cause issues with Modbus COM port detection on some Windows configurations.

---

## 4. Create a Virtual Environment

A virtual environment isolates this project's packages from any other Python projects on the machine. This is especially important because pymodbus had breaking API changes between v2.x and v3.x.

```bat
cd C:\lsci-platform

:: Create the virtual environment
python -m venv lsci_env

:: Activate it
lsci_env\Scripts\activate
```

After activation, your prompt will change to show (lsci_env):
  (lsci_env) C:\lsci-platform>

Always activate this environment before running the platform. You can create a launcher script:

```bat
:: File: run_lsci.bat  (save in the project root)
@echo off
call C:\lsci-platform\lsci_env\Scripts\activate
python new_GUI_sc_55.py
pause
```

Double-clicking run_lsci.bat will activate the environment and launch the GUI automatically.

---

## 5. Install pip Packages

With the virtual environment active:

```bat
:: Upgrade pip first
python -m pip install --upgrade pip

:: Install all pip-available packages
pip install -r requirements.txt
```

This installs:
- Image processing: Pillow, opencv-python
- Scientific stack: numpy, scipy, scikit-learn
- Visualization: matplotlib
- GUI: customtkinter, tkdial
- Hardware comms: pymodbus, pyserial
- File I/O: python-docx
- Audio: pygame
- NI-DAQ wrapper: PyDAQmx (wrapper only — requires NI runtime, see Section 7)

Expected install time: 3-8 minutes depending on internet speed.

### 5.1 Post-install pip verification

```bat
python -c "import numpy, cv2, customtkinter, pymodbus, serial, matplotlib, scipy, sklearn, PIL, pygame, tkdial, docx; print('All pip packages OK')"
```

Expected: All pip packages OK

### 5.2 Known pip install warnings

pygame deprecation notice on install — safe to ignore.

pymodbus version note: the platform uses the v3.x API. If you see
"ImportError: cannot import name 'ModbusSerialClient'", you have pymodbus v2.x:

```bat
pip uninstall pymodbus
pip install "pymodbus>=3.5.0"
```

---

## 6. Install the Camera SDK (scTDC)

The scTDC module is not available on PyPI. It must be obtained from the camera vendor.

### 6.1 What to request from the vendor

Contact the camera vendor (TDC/GPX3 Camera System) and request the following for Python 3.10, Windows 64-bit:

| File | Description |
|---|---|
| scTDClib.dll | Core camera driver DLL |
| tdc_gpx3.ini | Camera initialization configuration |
| scTDC-x.y.z-cp310-cp310-win_amd64.whl | Python 3.10 wrapper wheel |
| Dependency DLLs | Additional DLLs that scTDClib.dll loads (vendor will specify) |

In some SDK packages the vendor also provides a USB3 Vision driver installer or FTDI driver. Install those before the Python wheel.

### 6.2 Place the DLL and INI file

Copy the following files into the project root directory — the same folder as new_GUI_sc_55.py:

```
C:\lsci-platform\
    new_GUI_sc_55.py       <- main script
    scTDClib.dll           <- MUST be here
    tdc_gpx3.ini           <- MUST be here
    requirements.txt
    README.md
    ...
```

Why must they be in the same directory?

The application uses this at module level:

```python
script_dir = os.path.dirname(os.path.abspath(__file__))
os.add_dll_directory(script_dir)
```

os.add_dll_directory() (Python 3.8+) adds the specified directory to the DLL search path for the current process only. It does not add to PATH. This means scTDClib.dll and any DLLs it depends on must all be present in script_dir — not in System32, not on PATH, not in a subdirectory.

### 6.3 Install the Python wheel

With the virtual environment active:

```bat
pip install path\to\scTDC-x.y.z-cp310-cp310-win_amd64.whl
```

Replace path\to\ with the actual path where you saved the wheel.

### 6.4 Verify camera SDK

```bat
python -c "import scTDC; print('scTDC OK')"
```

If this fails with "OSError: [WinError 126] The specified module could not be found", the DLL is either:
- Not in the project root directory, or
- Missing a dependency DLL

To diagnose missing dependencies, use the free tool Dependencies (available on GitHub). Open scTDClib.dll in that tool to see exactly which DLLs it requires and which are missing.

### 6.5 About tdc_gpx3.ini

The INI file contains hardware-specific timing parameters. The application loads it with:

```python
camera = scTDC.Camera(inifilepath="tdc_gpx3.ini", autoinit=False, lib=lib)
```

The path is relative to the working directory. Always launch the application from inside the project root directory, or use the run_lsci.bat launcher.

Do not edit tdc_gpx3.ini unless directed by the camera vendor.

---

## 7. Install NI-DAQmx Runtime and PyDAQmx

### 7.1 Install NI-DAQmx Runtime

1. Go to: https://www.ni.com/en/support/downloads/drivers/download.ni-daq-mx.html
2. Select the latest stable version (minimum: NI-DAQmx 19.0)
3. Download the full installer (~1.5-2 GB)
4. Run the installer — accept defaults
5. RESTART WINDOWS when prompted. This step is mandatory.

### 7.2 Verify DAQ card recognition

After reboot, open NI MAX (Start Menu -> search "NI MAX"):
- Expand Devices and Interfaces
- Your DAQ card should appear as Dev1 (or Dev2, Dev3, etc.)

The platform hardcodes the device name "Dev1" in two places in new_GUI_sc_55.py:
  - "Dev1/port0/line0" in CreateDOChan calls (lines ~1913 and ~5574)

If your card is listed as Dev2 or another name in NI MAX, edit both occurrences before the stimulator will work.

### 7.3 Verify PyDAQmx

```bat
python -c "import PyDAQmx; print('PyDAQmx OK')"
```

If this fails with "ImportError: DLL load failed", the NI-DAQmx runtime is not installed or the system needs a reboot.

### 7.4 Physical wiring

Connect DAQ card digital output terminal Dev1/port0/line0 to the stimulator hardware input. Ensure signal ground is connected to stimulator ground. Set the stimulator hardware to direct digital input mode — the software generates the full pulse waveform including duration and polarity.

---

## 8. Install USB-Serial Drivers

Three USB-to-serial adapters connect hardware peripherals to the PC.

### 8.1 Driver download table

| Chipset | Device Manager label | Driver download |
|---|---|---|
| CP2102 / CP2104 (Silicon Labs) | Silicon Labs CP210x USB to UART Bridge | silabs.com/developers/usb-to-uart-bridge-vcp-drivers |
| FT232R / FT232H (FTDI) | USB Serial Port (COMx) | ftdichip.com/drivers/vcp-drivers |
| CH340 / CH341 | USB-SERIAL CH340 (COMx) | wch.cn/download/CH341SER_EXE.html |
| Prolific PL2303 | Prolific USB-to-Serial Comm Port (COMx) | prolific.com.tw |

Install drivers for whichever chipsets your hardware uses. When in doubt, install all four — they do not conflict.

### 8.2 Verify COM port assignment

After installing drivers and connecting hardware:
1. Open Device Manager (Win+X -> Device Manager)
2. Expand Ports (COM & LPT)
3. Confirm three COM port entries appear — one for each device

The application auto-detects ports at startup. The console output will confirm:

```
[PORT DETECTION] Available COM ports: ['COM3', 'COM4', 'COM7']
[PORT DETECTION] RELAY_GALVO found on COM4
[PORT DETECTION] XYZ_STAGE found on COM3
[PORT DETECTION] LASER found on COM7
```

### 8.3 If auto-detection fails

Fallback port assignments if description matching fails:
- Relay / Galvo / Z-axis: first available COM port (fallback COM4)
- XYZ Stage: second available COM port (fallback COM3)
- Laser Monitor: third available COM port (fallback COM7)

To override with fixed port numbers, edit auto_detect_ports() in new_GUI_sc_55.py (lines 74-106) and add explicit assignments before the fallback section.

---

## 9. Assemble the Project Directory

After completing all installation steps, your project root must contain:

```
C:\lsci-platform\
    new_GUI_sc_55.py               Main application — run this file
    scTDClib.dll                   Camera vendor SDK DLL — REQUIRED
    tdc_gpx3.ini                   Camera configuration — REQUIRED
    logo.png                       Lab logo (optional — shown in About dialog)
    live_contrast_GUI_2.py         Companion: Live Speckle Tracker
    ST_GUI_3.py                    Companion: Spatial-Temporal Analysis
    requirements.txt
    README.md
    ENVIRONMENT.md
    CHANGELOG.md
    docs/
        architecture.md
        installation.md
        hardware_setup.md
        calibration_guide.md
        user_guide.md
        data_formats.md
        api_reference.md
    lsci_env\                      Virtual environment folder
    run_lsci.bat                   Optional launcher script
```

Files generated at runtime (do not commit to git):
- galvo_calibration.pkl
- galvo_calibration_config.json
- <name>_<timestamp>.pkl / .json  (named calibration backups)

---

## 10. Verify the Complete Install

Save and run this verification script:

```python
# verify_install.py
import sys, os

print("=" * 60)
print("LSCI Platform - Install Verification")
print("=" * 60)
print(f"Python:   {sys.version}")
print(f"Platform: {sys.platform}")
print(f"Arch:     {'64-bit' if sys.maxsize > 2**32 else '32-bit'}")
print(f"Workdir:  {os.getcwd()}")
print()

if sys.maxsize <= 2**32:
    print("FAIL  Python is 32-bit. Must use 64-bit Python 3.10.")
    sys.exit(1)
else:
    print("OK    64-bit Python")

required_files = ["scTDClib.dll", "tdc_gpx3.ini", "new_GUI_sc_55.py"]
for f in required_files:
    status = "OK  " if os.path.exists(f) else "MISS"
    print(f"{status}  {f}")

print()
packages = [
    ("numpy", "numpy"), ("Pillow", "PIL"), ("opencv-python", "cv2"),
    ("scipy", "scipy"), ("scikit-learn", "sklearn"),
    ("matplotlib", "matplotlib"), ("customtkinter", "customtkinter"),
    ("tkdial", "tkdial"), ("pymodbus", "pymodbus"),
    ("pyserial", "serial"), ("python-docx", "docx"),
    ("pygame", "pygame"), ("PyDAQmx", "PyDAQmx"), ("scTDC", "scTDC"),
]

all_ok = True
for pip_name, import_name in packages:
    try:
        mod = __import__(import_name)
        ver = getattr(mod, "__version__", "n/a")
        print(f"OK    {pip_name:<22}  v{ver}")
    except ImportError as e:
        print(f"FAIL  {pip_name:<22}  {e}")
        all_ok = False

print()
try:
    from PIL.Image import Resampling
    print("OK    PIL.Image.Resampling  (Pillow >= 10.0 confirmed)")
except ImportError:
    print("FAIL  PIL.Image.Resampling missing — pip install 'Pillow>=10.0'")
    all_ok = False

try:
    from pymodbus.client import ModbusSerialClient
    print("OK    pymodbus v3 API confirmed")
except ImportError:
    print("FAIL  pymodbus v3 API missing — pip install 'pymodbus>=3.5.0'")
    all_ok = False

print()
import serial.tools.list_ports
ports = [p.device for p in serial.tools.list_ports.comports()]
print(f"{'OK  ' if ports else 'WARN'}  COM ports: {ports if ports else 'none detected'}")

print()
try:
    import PyDAQmx
    t = PyDAQmx.Task()
    t.StopTask()
    print("OK    NI-DAQmx runtime responsive")
except Exception as e:
    print(f"WARN  NI-DAQmx: {e}")

print()
print("=" * 60)
print("Ready to launch." if all_ok else "Fix issues above before launching.")
print("=" * 60)
```

```bat
python verify_install.py
```

All lines should show OK. WARN lines for COM ports (no hardware connected yet) and NI-DAQmx (if not using stimulator) are acceptable.

---

## 11. First Launch

### 11.1 Launch

From within the activated virtual environment, in the project root:

```bat
python new_GUI_sc_55.py
```

Or double-click run_lsci.bat.

### 11.2 Console output on startup

Watch the console during startup. You should see:

```
[PORT DETECTION] Available COM ports: ['COM3', 'COM4', 'COM7']
[PORT DETECTION] RELAY_GALVO found on COM4
[PORT DETECTION] XYZ_STAGE found on COM3
[PORT DETECTION] LASER found on COM7
[PORT DETECTION] Final port assignment: {'relay_galvo': 'COM4', 'xyz_stage': 'COM3', 'laser': 'COM7'}
✅ Connected to COM4
DEBUG: Camera process started
```

If you see "Modbus Error: Failed to connect to COM4" — non-fatal. The GUI will open, but relay/galvo/z-axis controls will not work until hardware is connected.

### 11.3 First-use startup sequence

Perform these steps in order every session:

| Step | Action | What to check |
|---|---|---|
| 1 | Power on all hardware | Laser current appears in header bar |
| 2 | Click Initialize Camera | Dialog: "Camera initialized successfully" |
| 3 | Turn on relays 1-4 via the Relay button | Relay buttons turn green |
| 4 | Click Start Preview | Live image appears in left panel |
| 5 | Click Enable Galvo in Multi-Capture and Galvo window | Button turns green |
| 6 | Verify calibration loaded | Status bar: "Loaded 3x3 calibration" |
| 7 | Set Save Directory in Camera Controls | Browse to your data folder |
| 8 | (Optional) Click Calibrate Scale | Rulers update with mm ticks |

### 11.4 Laser current display

The laser input current is shown in the header bar (yellow text) at 1-second intervals. If it shows "--- mA", the laser serial port was not detected — check COM port assignment and cable connection.

---

## 12. Companion Scripts

Two additional scripts are launched by the main GUI via subprocess.Popen. Both must be present in the same directory as new_GUI_sc_55.py.

### 12.1 Live Speckle Tracker — live_contrast_GUI_2.py

Launched by: Quick Actions -> Launch Speckle Tracker

Standalone real-time speckle contrast monitoring tool. It runs as a completely separate process with its own camera connection.

CRITICAL: The Live Speckle Tracker must be launched BEFORE clicking Initialize Camera in the main GUI. Both tools cannot hold an active camera connection simultaneously. If the main GUI has already initialized the camera, restart it and launch the Speckle Tracker first.

### 12.2 Spatial-Temporal Analysis — ST_GUI_3.py

Launched by: Quick Actions -> Spatial-Temporal Contrast

Post-processing tool for spatiotemporal speckle contrast from saved data. Opens independently — camera does not need to be initialized.

### 12.3 If companion scripts are missing

The main GUI continues to work normally. Clicking their buttons shows a FileNotFoundError dialog. Only these two features are unavailable.

---

## 13. Troubleshooting First-Run Failures

### GUI window does not appear at all

Cause A — Import error on startup. Run:
```bat
python new_GUI_sc_55.py 2>&1 | more
```
Look for ImportError or ModuleNotFoundError at the top.

Cause B — scTDC DLL missing. Run:
```bat
python -c "import scTDC; print('OK')"
```
Address any OSError shown. Use the Dependencies tool to check for missing dependency DLLs.

Cause C — 32-bit Python. Run python --version — if it shows [32 bit], install the 64-bit version.

### "Modbus Error: Failed to connect" on launch

Hardware not connected or powered off. Connect hardware and restart the GUI.

### Camera initialization fails with error code

| Code | Meaning | Fix |
|---|---|---|
| -1 | SDK library not loaded | scTDClib.dll missing from project root |
| -2 | INI file not found | tdc_gpx3.ini missing or wrong working directory |
| -3 | Camera hardware not detected | USB cable not connected or USB driver not installed |
| -5 | Camera already in use | Live Speckle Tracker or another process holds camera |
| Other | Hardware-specific | Note code and message, contact camera vendor |

### Speckle video shows flat black image

1. Ensure Start Preview is running first
2. Check exposure time is not too low (minimum ~500 us in normal lighting)
3. Confirm laser is powered on and relay is ON
4. Increase C-axis slider value (at 0.000 the display shows nothing)

### NI-DAQmx stimulator fails

1. Confirm NI-DAQmx runtime is installed and system was rebooted afterward
2. Open NI MAX and confirm device name matches "Dev1"
3. Check digital output cable connection Dev1/port0/line0 to stimulator
4. Confirm stimulator hardware is in direct digital input mode

### Laser current shows "--- mA" continuously

1. Check laser Arduino is connected and powered
2. Confirm Arduino appears in Device Manager under Ports (COM & LPT)
3. Baud rate must be 19200; serial output format must be "Current: <float>\n"

---

## 14. Uninstall / Clean Rebuild

To completely remove the virtual environment and start fresh:

```bat
deactivate                   (if environment is currently active)
rmdir /S /Q lsci_env         (removes the entire virtual environment)
```

Then repeat Sections 4 and 5 to rebuild. The camera SDK wheel, NI-DAQmx runtime, and USB drivers do not need to be reinstalled — only the virtual environment is removed.

To uninstall NI-DAQmx:
  Control Panel -> Programs and Features -> NI-DAQmx -> Uninstall

To remove USB-serial drivers:
  Device Manager -> Ports (COM & LPT) -> right-click -> Uninstall device
  -> check "Delete the driver software for this device"

---

*Document version: January 2026 - Platform v2.1*
